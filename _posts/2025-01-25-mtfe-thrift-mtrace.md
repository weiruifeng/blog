---
layout: post
title: "@mtfe/thrift 的链路追踪 Mtrace"
date: 2025-01-25
categories: 源码阅读
tags: [Thrift, RPC, Node.js, 美团, 链路追踪, Mtrace]
---

> 前置：[导引与术语表]( {% post_url 2025-01-21-mtfe-thrift-overview %} ) 术语表、[整体架构与调用流程]( {% post_url 2025-01-22-mtfe-thrift-design %} ) 调用流程与协议两条路。本篇讲链路追踪怎么在 Thrift 调用里实现——traceId 怎么来、怎么跨服务串联、怎么塞进协议帧、怎么上报。

## 链路追踪要解决什么

分布式系统里，一次用户请求可能穿过好几个服务。出了问题（慢、错），怎么知道是哪一段的问题？链路追踪的做法：给每次调用打一个 span（记录谁调谁、耗时、状态），把同一次请求的所有 span 用同一个 traceId 串起来，最终在平台上看到一条完整调用链。

`@mtfe/thrift` 在每次 RPC 前后埋点，生成客户端 span 和服务端 span，通过协议头把 traceId 传给下游，让整条链路串起来。

## 一次调用产生两条 span

```
客户端                        服务端
  │                             │
  ├─ new Mtrace() + start()     │   ← 客户端 span 开始
  ├─ 把 traceId/spanId 塞进 header
  ├─ 发请求 ───────────────────→ ├─ 收到请求
  │                             ├─ new Mtrace() + start()  ← 服务端 span 开始
  │                             ├─ 从 header 取 traceId/spanId（复用同一个 traceId）
  │                             ├─ 处理业务
  │                             ├─ end() + record()        ← 服务端 span 结束、上报
  │                             └─ 回响应
  ├─ 收响应                     │
  └─ end() + record()           │   ← 客户端 span 结束、上报
```

关键：**客户端和服务端用同一个 traceId**，但各自记各自的 span（客户端记端到端耗时，服务端记处理耗时）。traceId 由客户端生成、经协议头传给服务端，服务端复用。这样平台拿到两条 span 就能拼成"客户端发起 → 服务端处理 → 客户端收到"的完整链路。

## traceId 怎么生成

`Mtrace.js` 的 `generateTraceId`：

1. 用 `uuid.v1()` 生成（基于时间戳 + MAC，有序）
2. 去掉横线，取前 16 位和后 16 位 hex
3. 逐字节异或
4. 转成十进制字符串

结果是一个十进制大数字符串，作为本次链路的唯一 ID。

如果上游已经传了 traceId（`options.traceId`），就继承用，不新生成——这是跨服务串联的基础。

## spanId 与全量采样

- `spanId`：标识当前 span。上游传来的 `options.spanId` 缺省为 `'0'`。
- `debug = sample = (spanId !== '0')`：**上游传来的 spanId 非 '0' 时，全量采样**。这是个隐式约定——上游想让某次调用被完整追踪，就传一个非 '0' 的 spanId，下游就会上报而不被采样丢弃。

## traceId 怎么塞进协议帧

请求头注入有两条互斥路径（详见 [整体架构与调用流程]( {% post_url 2025-01-22-mtfe-thrift-design %} ) 的"协议两条路"），两套都支持链路头：

### 老协议（非 unified）

在 `writeMessageBegin` 之后，追加一个 field id = 32767、type = STRUCT 的 `RequestHeader`，里面装链路字段：

```thrift
struct RequestHeader {
    10001: string traceId,       // 链路 ID
    10002: string spanId,
    10003: string clientAppkey,  // 调用方 appkey
    10004: string clientIp,
    10005: string spanName,      // 服务名.方法名
    10006: string serverIpPort,
    10007: bool   debug,
    10008: bool   sample,
    10009: string version,
}
```

### 统一协议（unified）

由 `getHeader` 组装结构化的 `traceInfo`（含 traceId / spanId / clientIp / clientAppkey），经 `transport.unify()` 注入到帧里。鉴权头、cell/swimlane 也走同一条路。

无论哪套协议，traceId 都能透传，所以 Mtrace 不依赖具体协议选型。

## 服务端怎么收链路

`ThriftServer.js` 这边：

1. `createContext` 时从 `request.transport.getUnifiedInfo()` 取出 `info.header`。
2. `respond` 时读 `info.header.traceInfo`（traceId / spanId / clientIp / clientAppkey）。
3. `ctx.trace.init({ id: traceInfo.traceId, spanId: traceInfo.spanId, ..., clientSide: false })`——**复用客户端传来的 traceId**，标记为服务端 span。
4. `MtraceWorker.record(ctx.trace.end(0))` 上报服务端 span。
5. 同时回写 `responseInfo = { sequenceId, status, message }`。

`clientSide` 字段区分这是客户端 span 还是服务端 span，平台据此画调用方向。

## 链路对象序列化

`Mtrace.serialize()` 把 span 标准化成上报结构：

```javascript
{
  traceId, spanId, spanName,
  start: startTime,
  duration: endTime - startTime,
  clientSide,
  infraName: 'node-thrift',
  infraVersion: '2.5.3',
  ip: localIp,
  appkey: localAppKey,
  status,
  local: { /* 本地信息 */ },
  remote: { /* 远程信息 */ },
}
```

`infraName: 'node-thrift'` 让平台知道这条 span 来自 Node 版 SDK。

## MtraceWorker：攒一批再上报

`MtraceWorker` 是单例，负责把 span 攒起来批量上报，不阻塞业务调用。

### 动态上报频率

上报周期 `period` 会根据积压的 span 数量动态调整：

- span 少（`<= 1024`）且 period 还没到上限 → `period *= 2`（降频，少打扰）
- span 多（`> 1024`）且 period 没到下限 → `period /= 2`（提频，赶紧上报）

边界：`DEFAULT_MIN_FLUSH_PERIOD = 2000ms`（最快 2 秒一次），`DEFAULT_MAX_FLUSH_PERIOD = 64000ms`（最慢 64 秒一次），`MAX_TRACES_LENGTH = 1024`。

这是个很实用的自适应设计：流量小时不频繁上报省资源，流量大时提高频率避免积压丢失。

### 批量上报

`flush` 时：

1. 把所有积压的 span 打包成 `ThriftSpanList`。
2. `zlib.gzipSync(Buffer.concat(trans.outBuffers))` 压缩。
3. 经 `sgAgentClient.uploadCommonLog({ cmd: 7 /*TRACE_COMPRESS_DATA_LIST*/, content })` 上报。

gzip 压缩 + 批量，把上报的网络开销压到最低。上报走的是 SGAgent 的 `uploadCommonLog` 通道（跟服务发现同一个 SGAgent，复用连接通道）。

## 链路串联的几个关键点

1. **traceId 全链路共享**：客户端生成，经协议头透传给所有下游服务，是串联的钥匙。
2. **spanId 形成层次**：每跳生成新 spanId，形成调用树。
3. **双向记录**：客户端和服务端各记一条 span，拼出完整耗时。
4. **异步批量上报**：不影响业务性能，靠 MtraceWorker 攒批 + gzip。
5. **采样隐式约定**：spanId 非 '0' 触发全量采样，让需要追踪的调用不被丢弃。

## 客户端怎么主动传链路信息

业务可以在 exec 时显式传 traceId/spanId，把自己的请求和下游 Thrift 调用串起来：

```javascript
thriftPool.exec(Service, {
  methodName: 'getUserInfo',
  traceId: req.traceId,      // 继承上游请求的 traceId
  spanId: req.spanId,
  mtraceParams: {            // 附加业务标签，进 span
    userId: req.userId,
    action: 'getUserInfo',
  },
}, userId);
```

`mtraceParams` 会被合并进 span，平台上看链路时能看到这些业务上下文，便于排查。

## 设计上值得记的点

1. **traceId 透传是串联核心**：客户端生成、协议头携带、服务端复用。这套机制和业界 OpenTelemetry 的 traceContext 思路一致，只是用了美团自己的协议头格式。
2. **双协议都支持**：链路追踪不绑死在统一协议上，老协议通过 RequestHeader 也能传，平滑兼容老服务。
3. **自适应上报频率**：按积压量动态调 period，是个简洁有效的资源/延迟权衡，值得在其他批量上报场景借鉴。
4. **gzip + 批量 + 复用 SGAgent 通道**：把上报开销压到最低，对业务近乎无感。
5. **采样靠 spanId 隐式触发**：没有独立的采样配置，靠 `spanId !== '0'` 这个约定。简单但有效——需要追踪时上游传非零 spanId 即可。
