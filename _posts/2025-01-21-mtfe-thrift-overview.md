---
layout: post
title: "@mtfe/thrift 阅读导引与术语表"
date: 2025-01-21
categories: 源码阅读
tags: [Thrift, RPC, Node.js, 美团, 服务治理]
---

> 这一组文档是对美团 `@mtfe/thrift`（Node.js Thrift 客户端，当前分析版本 v4.3.2）源码的阅读笔记，用于个人技术沉淀与复习。源码位于 `node_modules/@mtfe/thrift/lib/`。

## 这是什么包

一句话：**让 Node 应用只用一个 appkey 就能调到美团内部的 Thrift 服务，顺带把服务发现、负载均衡、重试、鉴权、链路追踪、监控都包了。**

使用者只关心两件事——我是谁（`localAppKey`）、我要调谁（`remoteAppKey`）。剩下的（对方有哪些实例、连哪个、怎么连、失败了怎么办、调用怎么被追踪）全由 SDK 处理。

```javascript
const thriftPool = new ThriftPool({
  localAppKey: 'com.meituan.consumer',   // 我
  remoteAppKey: 'com.meituan.provider',  // 要调的服务
});
const order = await thriftPool.exec(Sample, 'getOrderById', 1);
```

## 怎么读这 5 篇

建议按顺序，每篇聚焦一个层面，尽量不重复：

| 篇目 | 主题 | 回答什么问题 |
|------|------|-------------|
| [导引与术语表]( {% post_url 2025-01-21-mtfe-thrift-overview %} )（本篇） | 导引 + 术语表 | 概念底座，后面各篇直接用术语 |
| [整体架构与调用流程]( {% post_url 2025-01-22-mtfe-thrift-design %} ) | 整体架构与调用流程 | 包有哪些组件、一次 `exec` 从入口到结果经过哪些环节、代码怎么组织 |
| [生产级特性]( {% post_url 2025-01-23-mtfe-thrift-analsys %} ) | 生产级特性 | 鉴权、重试、降级、缓存、负载均衡、监控分别怎么做 |
| [Mesh 模式]( {% post_url 2025-01-24-mtfe-thrift-mesh %} ) | Mesh 服务网格模式 | sidecar 是什么、Mesh 模式跟直连有什么区别、怎么平滑切换 |
| [链路追踪 Mtrace]( {% post_url 2025-01-25-mtfe-thrift-mtrace %} ) | 链路追踪 | traceId 怎么生成、怎么跨服务串联、怎么上报 |
| [非幂等操作的重试]( {% post_url 2025-01-26-mtfe-thrift-retry %} ) | 非幂等操作的重试与一致性 | SDK 默认重试对非幂等操作有什么风险、业务侧怎么兜 |

如果只看一篇，看 [整体架构与调用流程]( {% post_url 2025-01-22-mtfe-thrift-design %} )。想理解容错设计，看 [生产级特性]( {% post_url 2025-01-23-mtfe-thrift-analsys %} )。

## 术语表

下面这些词在各篇里反复出现，先在这里讲清楚。

### 美团基础设施

- **OCTO**：美团的服务治理平台。管服务注册、发现、鉴权、路由策略。本包几乎所有"服务发现"能力最终都落到 OCTO。
- **SGAgent**：装在每台机器本地的 agent，是访问 OCTO 的本地代理。应用不直连 OCTO 注册中心，而是问本地 SGAgent 拿服务列表。默认地址 `127.0.0.1:5266`。
- **哨兵 SGAgent（Sentinel）**：本地 SGAgent 挂了时的备用。一组集中部署的 SGAgent，通过 MNS 拉到列表后逐个探测挑一个能用的。
- **MNS（Meituan Naming Service）**：命名服务。能按 appkey 查到一个服务有哪些实例。本包用它查"哨兵 SGAgent 列表"本身（appkey = `com.sankuai.inf.sg_sentinel`）。
- **Mtrace**：美团的分布式链路追踪系统。本包在每次调用前后埋点，把 span 上报上去，最终在 Mtrace 平台拼出完整调用链。
- **Cat**：美团的监控告警系统。本包用它记事务（耗时/成功失败）和错误。
- **octo_proxy / Sidecar**：Mesh 模式下装在本地的代理进程。应用把请求交给它，由它负责服务发现和路由。应用与 sidecar 之间走 Unix Domain Socket。

### 协议与字段

- **appkey**：服务唯一标识，形如 `com.meituan.xxx`。调用方叫 `localAppKey`，被调方叫 `remoteAppKey`。
- **统一协议（Unified Protocol）**：美团在标准 Thrift 协议上扩展的协议。支持压缩、校验、鉴权头、链路头等。出帧以魔数 `0xab 0xba` 开头。对应配置项 `unified` / `serviceName`。
- **传统协议**：标准 Apache Thrift 二进制协议。链路头通过一个 field id = 32767 的 `RequestHeader` struct 挂在消息头里。
- **fweight**：服务实例的"地理权重"。SDK 用它判断实例与调用方的远近，做就近路由。值越大越近。
- **cell**：部署单元。可用于灰度隔离，让请求只打到指定 cell 的实例。
- **swimlane**：泳道。流量隔离机制，常见于压测/灰度，让带某 swimlane 标记的请求只走对应泳道的实例。

### SDK 内部对象

- **ThriftPool**：用户直接打交道的核心类。名字叫 Pool，但**并不是 TCP 连接池**——它管的是"服务实例池 + 负载均衡 + 重试编排"。详见 [整体架构与调用流程]( {% post_url 2025-01-22-mtfe-thrift-design %} ) 的"连接池真相"一节。
- **SGAgentClient**：负责服务发现的内部类。本地 SGAgent → 哨兵 SGAgent 的降级逻辑在这里。
- **SentinelManager**：负责拉取并缓存哨兵 SGAgent 列表。
- **Thrift**（注意大小写，区别于外层包名）：单次 RPC 的执行器。建 socket、组装协议栈、收发一帧数据。一次调用建一条连接，用完即毁。
- **Mtrace / MtraceWorker**：前者是单次调用的链路对象（一个 span），后者是单例上报器，攒一批 span 批量上报。

## 几个容易被源码误导的点（先记着）

写这组笔记时反复核对源码，发现原文档有几处不准确或容易误读，这里先点出来：

1. **ThriftPool 没有 TCP 连接池。** 每次 `exec` 都 `net.createConnection` 新建 socket，调用结束就 `destroy`。无复用、无保活。`Connection.js` 里那一套断线重连参数（`max_attempts`、`retry_backoff`）是死代码，从没被启用过。
2. **Mesh 模式下 `9101` 不是客户端实际连接的端口。** 它只是塞进 `service` 对象里给 mtrace/cat 上报和 header 用的占位值。真正的连接走 Unix Domain Socket。
3. **`serviceName` 在统一协议/鉴权下"必须配"只是约定，不是硬校验。** 没配只打 error 日志，不抛错、不阻断调用。
4. **重试次数的含义。** `retries: 2` 意味着 1 次初始尝试 + 2 次重试 = 共 3 次请求。别误以为是"总共 2 次"。
5. **默认超时是 1 秒。** `DEFAULT_TIMEOUT = 1000`、`DEFAULT_SGAGENT_TIMEOUT = 1000`。生产环境基本都要调大。

后面各篇会在用到这些点的地方展开。
