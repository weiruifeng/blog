---
layout: post
title: "@mtfe/thrift 的 Mesh 模式"
date: 2025-01-24
categories: 源码阅读
tags: [Thrift, RPC, Node.js, 美团, Service Mesh, Sidecar]
---

> 前置：[导引与术语表]( {% post_url 2025-01-21-mtfe-thrift-overview %} ) 术语表、[整体架构与调用流程]( {% post_url 2025-01-22-mtfe-thrift-design %} ) 调用流程。本篇专讲 Mesh（服务网格）模式——它是什么、跟直连有什么区别、怎么切换、怎么降级。

## Mesh 模式想解决什么

传统直连模式下，SDK 自己干所有网络活：服务发现、负载均衡、重试、熔断、监控埋点。问题是这些逻辑每种语言、每个 SDK 各实现一套，行为难统一，升级要跟着应用重启。

Mesh 模式换了个思路：**把所有网络治理逻辑从 SDK 挪到一个本地代理进程（sidecar）里**。SDK 只管把请求扔给 sidecar，剩下的由 sidecar 统一处理。

好处：
- **语言无关**：任何语言只要能把请求发给 sidecar 就行，sidecar 是统一的。
- **统一治理**：路由、限流、熔断策略在 sidecar 层统一生效，不用每个 SDK 各搞一套。
- **独立升级**：sidecar 升级不影响应用进程。
- **连接复用**：sidecar 到后端是长连接池，绕开了 SDK"一调用一连接"的开销（见 [整体架构与调用流程]( {% post_url 2025-01-22-mtfe-thrift-design %} ) 的"连接池真相"）。

代价：多一跳本地代理，多一层运维。

## 核心机制

### 应用和 sidecar 之间走 Unix Domain Socket

```
应用进程 → ThriftPool → Unix Domain Socket → octo_proxy(sidecar) → 目标服务实例
```

关键路径与端口（核对源码 `ThriftPool.js`）：

- **Unix Domain Socket 路径**：`/opt/meituan/apps/octo_proxy/socket/octo_proxy_outbound_unix_socket`——这是应用真正连接 sidecar 的通道。
- **订阅端口** `127.0.0.1:5318`：应用向 sidecar 声明"我要订阅哪个 remoteAppKey"，POST `/inc_subscribe`。
- **9101 是占位值，不是连接端口**：Mesh 模式下 SDK 把 `service` 对象填成 `{ip:'127.0.0.1', port:9101}`，但这只用于 mtrace/cat 上报和 header 里的 remoteIp/remotePort。真正建连走的是上面的 Unix socket，不是 TCP 9101。原文档把 9101 当成"TCP 代理端口"是误导。

为什么用 Unix Domain Socket 而不是 TCP 回环？性能更高（不走网络栈）、更安全（靠文件系统权限）、是内核级通信。

### 启动时订阅，运行时心跳

```
new ThriftPool({enableMesh: true})
   │
   ├─ meshSubscribe(remoteAppKey)
   │     POST 127.0.0.1:5318/inc_subscribe
   │     body: { localAppkey, remoteAppkeys: [remoteAppKey] }
   │     超时 1s
   │     成功 → meshSubscribeStatus = 'fullfilled'
   │     失败 → useMesh = false，回退直连
   │
   └─ 订阅成功后启动 meshHeartbeat
         每 1 秒：meshSocketDetect() 探 Unix socket 连通性
           通 → 继续下一轮
           不通 → 重新订阅 → 还失败 → useMesh = false，回退直连
```

订阅是告诉 sidecar"我要调这个服务"，之后 sidecar 负责去控制面拿实例、做路由。心跳是确认 sidecar 还活着，挂了就回退。

### Mesh 模式下 exec 的行为变了

`meshCheck` 有三道门槛，**任一不满足就抛 `FallbackException` 回退直连**：

1. `enableMesh` 开关为 true
2. `appenv` 存在（环境信息）
3. 有 `serviceName` 且**没有**自带 `serviceList`

通过门槛后，Mesh 模式的 exec 跟直连有几个关键不同：

- `getServiceList` 直接返回 `[]`——不取实例、不做加权轮询（实例选择交给 sidecar）。
- `service` 固定为 `{ip:'127.0.0.1', port:9101}`（占位，如上所述）。
- 真正建连：`new Thrift(UNIX_DOMAIN_SOCKET_PATH, null, options, true)`，`isMesh=true` 时 `connectType = UNIX_DOMAIN_SOCKET`。
- 重试：只是对同一个 sidecar socket 重试，不会换实例（实例选择权不在 SDK 手里了）。
- header 多塞几个字段给 sidecar 用（见下）。

### Mesh 模式的 header 扩展

开 Mesh 时，`getHeader` 会额外往 header 里塞：

- `signature`——鉴权签名
- `localAppKey`——本应用标识
- `retryTime`——期望重试次数（`(options.retry||{}).retries || 1`），让 sidecar 知道该重试几次
- `cell`——部署单元
- `swimlane`——泳道

这些是 sidecar 做路由和治理需要的上下文，由 SDK 透传过去。

## Mesh vs 直连对比

| 维度 | 直连模式 | Mesh 模式 |
|------|---------|-----------|
| 服务发现 | SDK 问 SGAgent | sidecar 负责 |
| 负载均衡 | SDK 加权轮询 | sidecar 负责 |
| 连接 | TCP 直连目标实例（一调用一连接） | Unix socket 到 sidecar（sidecar 到后端长连接复用） |
| 故障转移 | SDK 换实例重试 | sidecar 处理 |
| 监控埋点 | SDK 埋 | sidecar 统一埋（SDK 仍埋自己这一段） |
| 策略升级 | 改 SDK，跟应用发版 | sidecar 独立升级 |
| 实例选择权 | SDK | sidecar（SDK 重试也不换实例） |

## 怎么平滑切换

一个开关，零代码改动：

```javascript
const thriftPool = new ThriftPool({
  localAppKey: 'com.meituan.consumer',
  remoteAppKey: 'com.meituan.provider',
  enableMesh: true,   // 开 Mesh
  serviceName: 'UserService',  // Mesh 需要
});
```

关键是**自动降级**：订阅失败、心跳失败、或 meshCheck 门槛不满足，都会让 `useMesh = false`，无缝回退到直连。应用感知不到切换。这意味着可以灰度开 Mesh——即便 sidecar 没装好，应用也能正常跑。

监听降级事件可以做告警：

```javascript
thriftPool.on('meshFallback', () => {
  // 侧信道通知监控系统：Mesh 降级了
});
```

## 调试

开 debug 日志看 Mesh 相关行为：

```javascript
process.env.DEBUG = 'mtsdk:NodeThrift*';
```

Mesh 相关的关键日志标记：`Mesh: subscribe`、`Mesh(Detect): Fallback`、`ThriftPool(Mesh): subscribe error`、`ThriftMesh: FALLBACK to normal`。

## 设计上值得记的点

1. **关注点分离**：业务逻辑留在应用，网络治理交给 sidecar。应用代码不用再操心服务发现、负载均衡这些。
2. **渐进式**：开关切换 + 自动降级，让 Mesh 可以灰度上线、随时回退，不破坏现有应用。
3. **Unix Domain Socket 选型**：本地通信用 UDS 而非 TCP 回环，性能和安全性都更好。
4. **9101 的真实角色**：占位值，别误以为是连接端口。读源码时看到 `port: 9101` 要知道它只服务于埋点和 header。
5. **实例选择权转移**：Mesh 下 SDK 不再选实例，所以重试语义变了——只是对 sidecar 重试，不保证换实例。这对非幂等操作的判断有影响（见 [非幂等操作的重试]( {% post_url 2025-01-26-mtfe-thrift-retry %} )）。
