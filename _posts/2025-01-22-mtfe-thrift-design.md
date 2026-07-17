---
layout: post
title: "@mtfe/thrift 整体架构与调用流程"
date: 2025-01-22
categories: 源码阅读
tags: [Thrift, RPC, Node.js, 美团, 架构]
---

> 前置：先看 [导引与术语表]( {% post_url 2025-01-21-mtfe-thrift-overview %} ) 的术语表。本篇讲包的全貌——有哪些组件、一次调用怎么走完、代码怎么组织。

## 这个包要解决什么问题

在美团内部，一个 Thrift 服务背后可能有几十上百个实例，散布在不同机房。调用方面对的痛点：

1. **不知道连谁**——只有一个 appkey，没有具体 IP。
2. **实例会变**——扩缩容、宕机、发布，实例列表随时变。
3. **要就近**——同机房快、跨机房慢，最好优先连近的。
4. **要稳**——某个实例挂了要自动换一个，超时要能重试。
5. **要可观测**——调用链要能追踪，耗时和成功率要能监控。
6. **要安全**——调用要能被鉴权，非法调用要被拦。

`@mtfe/thrift` 把这些全打包成一个 SDK。用户只给 appkey，剩下全包。

## 核心组件

```
用户代码
   │
   ▼
ThriftPool ── 编排一次调用的全流程
   ├─ SGAgentClient ── 服务发现（本地 SGAgent → 哨兵 SGAgent）
   │     └─ SentinelManager ── 拉哨兵 SGAgent 列表
   ├─ Thrift ── 单次 RPC 执行器（建连接、组装协议栈、收发一帧）
   │     └─ protocol/ transport/ ── 协议与传输层
   ├─ getSignature ── 鉴权（新旧两套并行）
   ├─ Mtrace + MtraceWorker ── 链路追踪（埋点 + 批量上报）
   └─ Cat ── 监控埋点
```

每个组件的职责，一句话：

- **ThriftPool**：用户入口。把"服务发现 → 选实例 → 鉴权 → 建连 → 调用 → 重试 → 埋点"串起来。持有所有默认配置。
- **SGAgentClient**：问本地 SGAgent 拿服务实例列表，本地挂了就问哨兵。还负责列表过滤（cell/swimlane/IP 黑白名单）和就近路由（fweight）。
- **SentinelManager**：本地 SGAgent 挂了时，用 MNS 拉一组哨兵 SGAgent 列表并缓存。
- **Thrift**（小写 lib/Thrift.js）：干一次真正的 RPC。建 TCP 或 Unix Domain Socket，组装协议栈，发请求收响应。**一次调用建一条连接，用完即毁**。
- **protocol/**：自实现的二进制协议。老协议通过 field id 32767 的 `RequestHeader` 挂链路头；统一协议通过 transport 层注入。
- **transport/**：帧传输。统一协议的魔数解析、adler32 校验、gzip/snappy 压缩、统一头注入都在这。
- **Mtrace / MtraceWorker**：前者是单次调用的 span 对象，后者攒一批 span gzip 压缩后批量上报。
- **Cat**：记事务耗时和错误。
- **info-uploader/**：构造连接池时往本地 raptor-log-agent 推一条 SDK 版本信息，纯埋点，与调用链无关。

还有几个边角文件：`utils.js`（加权轮询、socket 探测、shuffle、co 转换）、`extend.js`（往 thrift 注入 `TThriftException` 异常类型）、`binary.js`（二进制读写原语，从 thrift 拷的）、`ThriftRouter.js` + `ThriftServer.js`（服务端用的路由与处理框架，本系列主要讲客户端，服务端只在 [链路追踪 Mtrace]( {% post_url 2025-01-25-mtfe-thrift-mtrace %} ) 里顺带提它怎么收链路）。

## 一次调用的完整流程

从 `thriftPool.exec(Sample, 'getOrderById', 1)` 入口开始：

```
exec
 │
 1. meshCheck ──── 如果开了 Mesh 且条件满足，走 Mesh 分支（见 Mesh 模式篇）
 │                  否则抛 FallbackException，落回普通直连
 │
 2. getServiceList ── 拿服务实例列表
 │     ├─ 先查缓存（默认 10 秒 TTL）
 │     └─ 缓存没有 → SGAgentClient.getProperServiceList
 │           ├─ 本地 SGAgent (127.0.0.1:5266) 探测
 │           ├─ 本地挂了 → 哨兵 SGAgent（MNS 拉列表 → shuffle → 逐个探测挑能用的）
 │           ├─ 拿到原始列表后过滤：serviceName / cell / swimlane / ports / IP 黑白名单
 │           └─ fweight 就近路由：同机房 > 同地区 > 跨地区
 │
 3. 负载均衡 ──── hashByWeightedRoundRobin：按 weight 累加成区间，随机数落点选一个实例
 │
 4. getSignature ── 如果开鉴权，并行起新旧两个鉴权，allSettled 等结果（见生产级特性篇）
 │
 5. getMtrace ──── 建链路 span：继承上游 traceId 或新生成，start() 记开始时间
 │
 6. getHeader ──── 组装请求头：链路头、鉴权头、cell/swimlane 等塞进 header
 │
 7. 重试循环 ──── 用 retry 库包一层，最多 retries 次
 │     └─ Thrift.exec（单次 RPC）
 │           ├─ net.createConnection 建一条新 socket
 │           ├─ 组装 Transport + Protocol，序列化请求（含 header）
 │           ├─ 发帧 → 收响应帧 → 反序列化
 │           └─ 调用结束 conn.destroy() 关连接
 │
 8. 埋点 ──── Mtrace.end() 记 span，Cat 记事务（耗时/成功失败）
 │
 9. 返回结果（或抛异常）
```

几个关键细节：

- **缓存命中就直接用**，10 秒内不会重复问 SGAgent，避免每次调用都打服务发现。
- **重试只对特定异常**：`TIMEOUT`、`CONNECT_TIMEOUT`、`UNKNOWN` 三种才重试，业务异常（如 `UNKNOWN_METHOD`）不重试——重试也没用。
- **重试换实例**：每次重试会重新走一遍 getServiceList + 负载均衡，所以重试很可能打到不同实例上（故障转移）。但 Mesh 模式下重试只是对同一个 sidecar socket 重试，因为实例选择权交给了 sidecar。

## 连接池真相（重要，别被名字骗了）

类名叫 `ThriftPool`，但它**没有 TCP 连接池**。

核对 `lib/Thrift.js`：每次 `Thrift.prototype.exec` 都 `net.createConnection` 新建一条 socket，调用结束（无论成功失败）在 `terminate` 里 `conn.destroy()` 关掉。所以是"一调用一连接"——无连接复用、无保活、无最大连接数管理。

那"Pool"指什么？指**服务实例池 + 负载均衡 + 重试编排**。它管的是"有一池子服务实例，挑哪个、挂了怎么办"，不管 TCP 连接的复用。

`lib/Connection.js` 里有一套断线重连逻辑（`max_attempts`、`retry_backoff=1.7`、`retry_delay=150`、`connect_timeout`），看着像连接池管理的样子。但 `Thrift.js` 构造 Connection 时根本没传这些 option，**这段是死代码**，从没生效过。读源码时别被它带偏。

这点的实际含义：在高 QPS 场景下，每次调用都新建 TCP 连接是有开销的（三次握手 + Thrift 握手）。这是这个包的已知局限，也是为什么 Mesh 模式（连接由 sidecar 长连接复用）是一种性能改进方向。

## 代码组织

```
@mtfe/thrift/
├── lib/
│   ├── ThriftPool.js      # 客户端核心：编排 exec、默认常量、鉴权、mesh、缓存
│   ├── Thrift.js          # 单次 RPC 执行器（建连、协议栈、收发一帧）
│   ├── SGAgentClient.js   # 服务发现 + 列表过滤 + fweight 路由
│   ├── SentinelManager.js # 哨兵 SGAgent 列表获取与缓存
│   ├── Connection.js      # 从 thrift 拷的连接封装（断线重连部分是死代码）
│   ├── Mtrace.js          # 单次调用的链路 span 对象
│   ├── MtraceWorker.js    # 单例上报器：攒 span、动态频率、gzip 批量上报
│   ├── ThriftServer.js    # 服务端框架（读协议头、respond、上报服务端 span）
│   ├── ThriftRouter.js    # 服务端中间件路由（支持 generator 中间件）
│   ├── utils.js           # 加权轮询、socket 探测、shuffle、co 转换
│   ├── extend.js          # 往 thrift 注入 TThriftException 异常类型
│   ├── binary.js          # 二进制读写原语（从 thrift 拷的）
│   ├── protocol/          # 自实现二进制协议 + 老版 RequestHeader 包装
│   ├── transport/         # 帧传输 + 统一协议（魔数/校验/压缩/头注入）
│   └── info-uploader/     # SDK 版本信息埋点上报
├── polyfill/              # 兼容性补丁
└── tests/                 # 测试
```

## 异常类型

`extend.js` 往 thrift 注入的 `TThriftException`，type 有这几种，理解它们对看懂重试和降级很关键：

| type | 含义 | 会重试吗 |
|------|------|---------|
| `TIMEOUT` | 调用超时 | 是 |
| `CONNECT_TIMEOUT` | 建连超时 | 是 |
| `UNKNOWN` | 未知异常 | 是 |
| `UNKNOWN_METHOD` | 方法不存在 | 否（业务错误，重试无用） |
| `EMPTY_SERVISE_LIST` | 服务列表为空 | 否 |
| `CLOSE` | 连接关闭 | 否 |

（注意 `EMPTY_SERVISE_LIST` 是源码里的拼写，不是笔误。）

## 协议两条路

请求头怎么塞进 Thrift 帧，取决于用哪套协议，两套互斥：

**老协议（非 unified）**：在 `writeMessageBegin` 之后，追加一个 field id = 32767、type = STRUCT 的 `RequestHeader`。里面装 traceId / spanId / clientAppkey / clientIp / spanName / serverIpPort / debug / sample / version。

**统一协议（unified）**：由 `getHeader` 组装成结构化的 `requestInfo` / `traceInfo` / `routeInfo` / `globalContext` / `localContext`，经 `transport.unify()` 注入到帧里。鉴权头（`INF_AUTH` / `auth-signature`）、cell/swimlane（`INF_CELL` / `INF_SWIMLANE`）、远程 appkey（`OCTO_REMOTE_APPKEY`）都走这条。出帧以魔数 `0xab 0xba 0x01 protocol` 开头，protocol 字节里 0x80=校验、0x40=gzip、0x20=snappy、低位=序列化类型。

两套协议都支持链路头注入，所以 Mtrace 链路追踪不依赖具体协议（详见 [链路追踪 Mtrace]( {% post_url 2025-01-25-mtfe-thrift-mtrace %} )）。

## 默认配置一览

从 `ThriftPool.js` 顶部常量直接抄，方便查阅：

| 配置 | 默认值 | 说明 |
|------|--------|------|
| `timeout` | 1000ms | 单次 RPC 超时 |
| `sgagentTimeout` | 1000ms | 访问 SGAgent 超时 |
| `retry.retries` | 2 | 重试次数（总尝试 3 次） |
| `retry.minTimeout` | 300ms | 重试最小间隔 |
| `retry.maxTimeout` | 1000ms | 重试最大间隔 |
| `retry.randomize` | false | 是否随机化重试间隔 |
| `sgagentCacheTimeout` | 10000ms | 服务列表缓存 TTL |

生产环境用基本都要调大 `timeout`、按业务决定 `retry`、开 `unified` + `serviceName`。

## 设计上值得记的点

1. **appkey 解耦网络细节**：用户从头到尾不碰 IP，全靠 appkey。实例增减、机房迁移对调用方透明。
2. **多层降级**：服务发现有"本地 → 哨兵 → 缓存"三级；Mesh 有"Mesh → 直连"降级；鉴权有"新旧并行、任一成功即可"。每层都有兜底。详见 [生产级特性]( {% post_url 2025-01-23-mtfe-thrift-analsys %} )。
3. **就近路由**：fweight 把"同机房优先"做到 SDK 里，省去人工配机房策略。
4. **协议可演进**：老协议和统一协议共存，统一协议承载压缩/校验/鉴权等增强能力，老协议保兼容。
5. **Mesh 平滑迁移**：一个 `enableMesh` 开关切换，Mesh 挂了自动回退直连。详见 [Mesh 模式]( {% post_url 2025-01-24-mtfe-thrift-mesh %} )。
6. **局限**：无连接复用（一调用一连接），高 QPS 下有建连开销；`serviceName` 非硬校验；默认超时偏短。这些是用的时候要心里有数的。
