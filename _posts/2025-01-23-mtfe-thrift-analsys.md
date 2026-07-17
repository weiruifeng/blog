---
layout: post
title: "@mtfe/thrift 生产级特性"
date: 2025-01-23
categories: 源码阅读
tags: [Thrift, RPC, Node.js, 美团, 容错, 负载均衡]
---

> 前置：[导引与术语表]( {% post_url 2025-01-21-mtfe-thrift-overview %} ) 术语表、[整体架构与调用流程]( {% post_url 2025-01-22-mtfe-thrift-design %} ) 调用流程。本篇讲 SDK 在生产环境赖以生存的几项能力：鉴权、重试、降级、缓存、负载均衡、监控。重试只讲机制，非幂等操作的风险与兜底单独在 [非幂等操作的重试]( {% post_url 2025-01-26-mtfe-thrift-retry %} ) 展开。

## 一、鉴权：新旧并行，任一成功即可

### 为什么这样设计

美团内部鉴权系统在从老方案（`@mtfe/octo-auth`）迁到新方案（`@dp/patriot-sdk`）。迁移期间服务端可能只认老签名、或只认新签名。SDK 的做法是**并行调两个，谁成功用谁**，保证迁移期两边都能通。

### 机制

`getSignature` 同时起两个 promise，用 `allSettled`（polyfill）等结果：

- 新鉴权：`new Auth(localAppKey).getTokenSignature(remoteAppKey)`
- 旧鉴权：`octoAuth(localAppKey)`

拿到结果后注入 header（统一协议走 `localContext`）：

- `localContext['auth-appkey']` = 本应用 appkey（两种都带）
- 新签名成功 → `localContext['INF_AUTH']` = 新签名
- 旧签名成功 → `localContext['auth-signature']` = 旧签名

关键点：**任一鉴权失败只记 Cat，不阻断调用**。也就是说鉴权是"尽力而为"，不是硬门槛——这跟服务端怎么验有关，SDK 只负责把签名送上去。

### 配置

```javascript
new ThriftPool({
  enableAuth: true,
  serviceName: 'UserService',  // 鉴权需要，见下
});
```

注意 `serviceName` 这里的"必须配"**只是约定，不是硬校验**：没配只打 error 日志（`[octoAuth] 要使用鉴权功能，请配置 serviceName`），不抛错、不阻断。实际会不会出问题取决于服务端校验严不严，但 SDK 这边不会拦你。这是源码核对时发现原文档"强制"措辞不准确的地方。

## 二、重试：只对特定异常，且会换实例

### 默认配置

```javascript
DEFAULT_RETRY_OPTIONS = {
  retries: 2,        // 重试次数（注意：总尝试 = 1 + 2 = 3 次）
  minTimeout: 300,   // 重试最小间隔
  maxTimeout: 1000,  // 重试最大间隔
  randomize: false,  // 是否随机化间隔
};
```

`retry` 参数支持数字（当 retries 用）或对象（全量配置）：

```javascript
retry: 3                                    // 重试 3 次
retry: { retries: 3, minTimeout: 500, maxTimeout: 2000, randomize: true }
```

### 只对三种异常重试

```javascript
if (e instanceof TThriftException &&
    (e.type === TIMEOUT ||
     e.type === CONNECT_TIMEOUT ||
     e.type === UNKNOWN)) {
  if (operation.retry(e)) return;  // 继续重试
}
reject(e);  // 其他异常直接抛
```

逻辑很清楚：**超时、建连超时、未知异常才重试；业务错误（方法不存在等）不重试**——重试也是同样的错，浪费请求。

### 重试会换实例

每次重试会重新走 `getServiceList` + 加权轮询，所以重试很可能打到不同实例上，天然实现故障转移。注意缓存：10 秒内的服务列表是缓存的，所以短时间内重试拿到的列表可能一样，但轮询选的实例会变。

> **Mesh 模式例外**：Mesh 下实例选择权交给 sidecar，SDK 重试只是对同一个 sidecar socket 重试，不换实例。这对非幂等操作的影响见 [非幂等操作的重试]( {% post_url 2025-01-26-mtfe-thrift-retry %} )。

### 雪崩防护

`randomize: true` 会随机化重试间隔，避免大量请求在同一时刻集中重试把下游打爆（雪崩）。生产环境对下游较敏感的服务建议开。

重试的潜在风险（对非幂等操作可能重复扣费/下单）是个重要话题，单独在 [非幂等操作的重试]( {% post_url 2025-01-26-mtfe-thrift-retry %} ) 展开。

## 三、降级：每一层都有兜底

这个包的容错设计核心思想是**多层降级**——任何一个环节挂了都有备选。

### 服务发现降级（本地 → 哨兵 → 缓存）

`SGAgentClient.getSGAgentThrift` 三级降级：

1. **本地 SGAgent**：探测 `127.0.0.1:5266`，通就用它，并缓存 `localClient` 复用。
2. **哨兵 SGAgent**：本地挂了，`SentinelManager.getSGAgentList` 用 MNS 拉哨兵列表 → `shuffle` 打乱 → `promiseSome(detectSocket)` 逐个探测，挑第一个能连的。
3. **缓存兜底**：哨兵也拿不到时，返回上次缓存的服务列表（SGAgentClient 内部有个无 TTL 的 cache 专门干这个）。

还有个细节：`exec` 一旦抛错，会把 `status` 置 `LOCAL_UNAVAILABLE`、清掉 `localClient`，下次用 `getSGAgentThrift(true)` 强制走哨兵重试一次。即本地 SGAgent 出问题后会主动切换，不会一直卡在坏的本地上。

### Mesh 降级（Mesh → 直连）

```javascript
return this.meshCheck(options)
  .then(() => this._exec(..., true))   // Mesh 模式
  .catch(err => {
    if (err instanceof FallbackException) {
      return this._exec(..., false);   // 降级到直连
    }
    throw err;
  });
```

meshCheck 门槛不满足、订阅失败、心跳失败，都会触发回退。详见 [Mesh 模式]( {% post_url 2025-01-24-mtfe-thrift-mesh %} )。

还有一个 `ForceUseMeshException`：`syncRegisterClient` 失败时会抛它，被外层 catch 后**强制切 Mesh 重跑**。这是反向的——某些场景下 Mesh 反而是兜底。

### 业务降级（自定义 fallback）

支持给调用配一个 `fallback`，调用失败时返回兜底数据而不是抛错：

```javascript
// 函数形式：按错误类型决定返回什么
fallback: (error) => {
  if (error.name === 'TApplicationException') {
    return { success: false, message: '应用层错误，请稍后重试' };
  }
  return { success: false, message: '服务暂时不可用' };
}

// 数据形式：直接返回固定值
fallback: { success: false, errorCode: 'SERVICE_UNAVAILABLE' }
```

也可以事后用 `thriftPool.setFallback(fn)` 动态设置全局降级策略。

## 四、缓存：三个不同用途的 cache

源码里有三个 cache，别混：

### 1. 服务列表缓存（ThriftPool 层，有 TTL）

```javascript
const cacheKey = ['NodeThrift','serviceList', localAppKey, remoteAppKey,
                  serviceName, ipWhiteList, ipBlackList, ports, cell, swimlane].join('.');
// 命中直接返回；未命中则查 SGAgent 并 put，默认 10s 过期
cache.put(cacheKey, list, this.sgagentCacheTimeout);  // 默认 10000ms
```

缓存键包含所有过滤维度，所以不同 cell/swimlane/黑白名单组合有各自的缓存项。`sgagentCacheTimeout` 可配。

### 2. SGAgent 请求降级缓存（SGAgentClient 层，无 TTL）

`getServiceListByProtocol` 成功时把结果写进这个 cache；失败时从这读，作为降级。无 TTL，只在成功时覆盖更新。

### 3. 哨兵列表缓存（SentinelManager 层，无 TTL）

按 `${isLocalHostOnline}.${env}` 缓存哨兵 SGAgent 列表。获取失败时返回缓存值兜底。

手动清缓存的 API：`thriftPool.clearCache()`（清第 1 个）。

## 五、负载均衡：加权轮询 + 就近路由

### 加权轮询（hashByWeightedRoundRobin）

算法很直接：把每个实例的 weight 累加成区间，生成一个随机数，落在哪个区间就选哪个实例。

```
实例 A(weight=3)、B(weight=1)
区间：A=[1,3]、B=[4,4]，sum=4
随机数 r ∈ [1,4]，落在 [1,3] 选 A，落在 [4,4] 选 B
→ A 被选中的概率是 3/4，B 是 1/4
```

weight 缺省按 1，所以不配 weight 时就是普通随机。这里的"轮询"严格说是"加权随机"，不是严格轮询——不保证均匀交替，只保证长期比例符合权重。

TS 版实现（忠实还原 `lib/utils.js` 逻辑）：

```typescript
interface ServiceInstance {
  ip: string;
  port: number;
  weight?: number; // 缺省按 1
}

// 加权随机：按 weight 累加成区间，随机数落点选实例
// 返回选中的实例在列表中的下标
function hashByWeightedRoundRobin(serviceList: ServiceInstance[]): number {
  let sum = 0;
  const weightRange: [number, number][] = [];

  // 1. 每个实例占一段连续整数区间，区间长度等于其 weight
  serviceList.forEach(service => {
    const weight = typeof service.weight === 'number' ? service.weight : 1;
    weightRange.push([sum + 1, sum + weight]); // [起点, 终点]，闭区间
    sum += weight;
  });

  // 2. 生成 [1, sum] 的随机整数，落在哪个区间就选哪个实例
  const r = Math.floor(Math.random() * sum + 1);
  let index = 0;
  weightRange.some((range, i) => {
    if (r >= range[0] && r <= range[1]) {
      index = i;
      return true; // 命中即停
    }
    return false;
  });
  return index;
}

// 用法
const list: ServiceInstance[] = [
  { ip: '10.1.1.1', port: 8080, weight: 3 }, // 区间 [1,3]
  { ip: '10.1.1.2', port: 8080, weight: 1 }, // 区间 [4,4]
];
const picked = list[hashByWeightedRoundRobin(list)]; // A 命中概率 3/4，B 1/4
```

几个细节值得记：

- **闭区间** `[sum+1, sum+weight]`：因为随机数 `r` 是整数且包含端点，所以用闭区间判断。weight=1 时区间退化为单点 `[n, n]`，照样能命中。
- **`Math.floor(Math.random() * sum + 1)`**：`Math.random()` 返回 `[0,1)`，乘以 sum 得 `[0, sum)`，+1 再 floor 得 `[1, sum]` 的整数。注意运算符优先级——是 `Math.floor((Math.random() * sum) + 1)`，不是 `Math.floor(Math.random() * (sum + 1))`，两者结果不同。
- **不处理空列表**：源码没做 `serviceList` 为空的判断，空列表时 `sum=0`、`Math.random()*0+1=1`、`r=1`，`some` 不执行，返回 `index=0`。实际调用方在更上层会先抛 `EMPTY_SERVISE_LIST`，所以这里不会收到空列表。
- **命中即停**：用 `some` + `return true` 提前跳出，找到第一个命中的区间就返回。

### 就近路由（fweight）

fweight 是服务实例的"地理权重"——由 OCTO 根据实例与调用方的相对位置算好下发的，值越大越近。SDK 拿到过滤后的实例列表后，**不做加权随机，而是按 fweight 分桶，优先返回最近的一桶**。

选择动作两步：

1. **分桶**：遍历列表，按 fweight 落到三个桶之一（同机房 / 同地区 / 跨地区）。不匹配任何区间的实例不进桶。
2. **按优先级返回第一个非空桶**：同机房有就用同机房，否则看同地区，再否则看跨地区，都没有就返回原列表兜底。

返回的那一桶之后会交给"加权轮询"做最终实例选择。所以 fweight 是**先筛范围、再轮询**——先把候选缩到最近的机房，再在机房内按 weight 轮询。

TS 版实现（还原 `lib/SGAgentClient.js` 的 fweight 分桶逻辑）：

```typescript
interface ServiceInstance {
  ip: string;
  port: number;
  fweight: number; // 地理权重，越大越近
}

// 按 fweight 就近分桶，返回最近的一桶
function routeByFweight(serviceList: ServiceInstance[]): ServiceInstance[] {
  const sameIDC: ServiceInstance[] = [];      // 同机房
  const sameRegion: ServiceInstance[] = [];   // 同地区
  const diffRegion: ServiceInstance[] = [];   // 跨地区

  // 1. 分桶：严格区间，等于边界值的不进任何桶
  serviceList.forEach(service => {
    if (service.fweight > 1 && service.fweight < 100) {
      sameIDC.push(service);
    } else if (service.fweight > 0.001 && service.fweight < 0.1) {
      sameRegion.push(service);
    } else if (service.fweight > 0.000001 && service.fweight < 0.0001) {
      diffRegion.push(service);
    }
    // 其余 fweight（含边界值、<=0、异常值）直接丢弃，不进桶
  });

  // 2. 按优先级返回第一个非空桶，都空则原列表兜底
  if (sameIDC.length > 0) return sameIDC;
  if (sameRegion.length > 0) return sameRegion;
  if (diffRegion.length > 0) return diffRegion;
  return serviceList;
}
```

几个细节值得记：

| fweight 区间 | 归类 | 量级 |
|-------------|------|------|
| `> 1 && < 100` | 同机房 | 个位~百位 |
| `> 0.001 && < 0.1` | 同地区 | 千分位~百分位 |
| `> 0.000001 && < 0.0001` | 跨地区 | 百万分位~万分位 |

- **严格开区间**：`> 下界 && < 上界`，等于边界值（如恰好 100、0.1、0.0001）的实例不进任何桶，会随兜底的原列表返回。fweight 由 OCTO 下发，正常不会卡在边界，但写代码时这个边界行为要心里有数。
- **不进桶 ≠ 丢弃**：没进桶的实例仍在原 `serviceList` 里，只有三个桶全空时才会作为兜底返回。三个桶只要有一个非空，没进桶的实例就被忽略。
- **只分桶、不排序**：桶内实例的顺序不变，后续交给加权轮询按 weight 选，所以 fweight 只负责"机房级别"的粗筛，不参与桶内细粒度选择。
- **降级语义**：同机房全挂（桶空）自动退到同地区，再退到跨地区，最后原列表——保证总有实例可调，宁可跨机房慢也不报空列表。

### IP 黑白名单

白名单：只保留列表里 IP 在白名单中的实例（过滤后为空则保持原列表，不强制清空）。
黑名单：剔除 IP 在黑名单中的实例。

### cell 灰度回退

开了 cell 过滤时，按 `status===ALIVE && cell===指定cell && serviceName 匹配` 筛选。有个细节：如果筛完是空的、且 cell 名以 `gray-release-` 开头，会自动关掉 cell 过滤重拉默认列表——灰度 cell 没实例时回退到默认，避免灰度配置把请求卡死。

## 六、监控与可观测

### Cat 事务埋点

每次调用记一个 Cat 事务，事务名按模式区分：

- 直连：`OctoCall`
- Mesh：`OctoMeshCall`

埋的数据包括 clientIp、timeout、swimlane、是否用鉴权等。成功记 SUCCESS，最终失败（重试用尽）记 FAIL。

### Mtrace 链路追踪

每次调用建一个 span，记录两端 IP/appkey、方法名、耗时、状态。traceId 跨服务串联，详见 [链路追踪 Mtrace]( {% post_url 2025-01-25-mtfe-thrift-mtrace %} )。

### 自动采集的指标

- 请求耗时（端到端）
- 成功率
- 重试次数
- 选中的实例（负载均衡分布）
- 鉴权使用情况（新旧）
- Mesh 使用率

### SDK 信息上报（info-uploader）

`new ThriftPool` 构造时会同步往本地 `127.0.0.1:5051`（raptor-log-agent）推一条 SDK 版本信息（LogType=101），失败落盘到 `/opt/logs/raptor-log-agent/sdk-info`。这是 SDK 自身的埋点，跟业务调用链无关，知道有这回事就行。

## 配置速查

生产环境典型配置（参数含义见上面各节）：

```javascript
new ThriftPool({
  localAppKey: 'com.meituan.consumer',
  remoteAppKey: 'com.meituan.service.provider',

  timeout: 3000,              // 默认 1s 偏短，生产调大
  retry: {
    retries: 2,
    minTimeout: 300,
    maxTimeout: 1000,
    randomize: true,          // 防雪崩
  },
  sgagentTimeout: 1000,
  sgagentCacheTimeout: 10000,

  unified: true,              // 统一协议
  serviceName: 'UserService',
  enableAuth: true,

  compress: 1,                // Snappy 压缩
  checksum: true,

  enableMesh: true,

  ipWhiteList: ['10.1.1.100'],   // 可选
  // ipBlackList: ['10.1.1.200'],

  fallback: (error) => ({ success: false, message: '服务暂时不可用' }),
});
```

## 设计上值得记的点

1. **每层都有兜底**：服务发现三级降级、Mesh 回退直连、鉴权新旧并行、SGAgent 失败读缓存。这是大规模分布式系统稳态运行的核心理念——别假设任何依赖永远可用。
2. **重试只对可恢复错误**：超时/未知才重试，业务错误不重试，避免无意义请求。
3. **就近路由内置**：fweight 让同机房优先成为 SDK 默认行为，不用人工配。
4. **缓存分级**：有 TTL 的（服务列表）和无 TTL 的（降级兜底）各司其职。
5. **鉴权与 serviceName 是约定非强制**：SDK 不拦你，但服务端可能会拒。配错了不报错只告警，排查时要注意看 error 日志。
