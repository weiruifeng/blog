---
layout: post
title: "非幂等操作的重试与数据一致性"
date: 2025-01-26
categories: 源码阅读
tags: [Thrift, RPC, Node.js, 美团, 重试, 幂等性, 分布式一致性]
---

> 前置：[导引与术语表]( {% post_url 2025-01-21-mtfe-thrift-overview %} ) 术语表、[生产级特性]( {% post_url 2025-01-23-mtfe-thrift-analsys %} ) 的"重试"一节。本篇聚焦一个具体问题：SDK 默认会对超时重试，这对非幂等操作（转账、下单、扣费）有重复执行的风险，怎么兜。

## 问题：SDK 的重试对非幂等操作是危险的

回顾 [生产级特性]( {% post_url 2025-01-23-mtfe-thrift-analsys %} ) 讲的重试机制：发生 `TIMEOUT`、`CONNECT_TIMEOUT`、`UNKNOWN` 三种异常时，SDK 自动重试，默认重试 2 次（共 3 次尝试）。

风险在于：**超时发生时，你无法确定服务端有没有处理完。**

```
客户端发请求 → 服务端开始处理 → 处理完成、正要回包 → 网络/超时丢了响应
   ↓ 客户端看到 TIMEOUT，重试
   ↓ 服务端又被调一次 → 重复扣费/重复下单
```

对幂等操作（查询）这无所谓，重试十次结果一样。但对非幂等操作（转账、下单、扣库存、扣费），重试会导致：

- 重复扣费——用户账户被多扣
- 重复下单——同一个订单建多次
- 状态混乱——业务状态被多次推进

这是这个包默认行为下的一个固有风险，不是 bug，是"默认偏向可用性"的设计选择——大多数调用是幂等的查询，宁可重试保可用。但用到非幂等操作时必须自己兜。

## 第一道防线：区分操作类型，非幂等少重试或不重试

最直接的办法：对非幂等操作把重试关掉或调小。

```javascript
// 查询类（幂等）——放心重试
thriftPool.exec(UserService, {
  methodName: 'getUserById',
  retry: 3,
  timeout: 5000,
}, userId);

// 转账类（非幂等）——不重试或只重试 1 次，超时拉长
thriftPool.exec(PaymentService, {
  methodName: 'transfer',
  retry: 0,          // 不重试
  timeout: 15000,    // 给足时间，靠超时拉长换取"一次成功"
}, fromUserId, toUserId, amount);
```

思路：非幂等操作宁可慢一点（超时拉长）、宁可失败一次让上层处理，也不要盲目重试。失败后由上层决定怎么补救（查状态、人工介入），而不是自动重发。

可以封装一个统一入口，按操作类型自动选策略：

```javascript
const OPERATION_CONFIGS = {
  QUERY:               { retries: 3, timeout: 5000 },   // 查询，随便重试
  IDEMPOTENT_WRITE:    { retries: 2, timeout: 8000 },   // 幂等写（带幂等键）
  NON_IDEMPOTENT_WRITE:{ retries: 0, timeout: 15000, requireIdempotencyKey: true },
};

async function call(service, method, params, opType) {
  const cfg = OPERATION_CONFIGS[opType];
  if (cfg.requireIdempotencyKey && !params.idempotencyKey) {
    throw new Error('非幂等操作必须提供 idempotencyKey');
  }
  return thriftPool.exec(service, {
    methodName: method,
    timeout: cfg.timeout,
    retry: cfg.retries,
    header: { 'X-Idempotency-Key': params.idempotencyKey },
  }, ...Object.values(params));
}
```

## 第二道防线：业务层做幂等（根本解）

关重试只是降低概率，不能根治——网络层任何一次重试（哪怕是 SDK 之外的网关重试、用户手动重试）都可能让非幂等操作被执行多次。**根本解是让操作本身幂等**。

核心思路：给每个操作一个**幂等键**（idempotencyKey），服务端处理前先查"这个键处理过没有"，处理过就直接返回上次的结果。

### 客户端：生成并携带幂等键

```javascript
const { v4: uuidv4 } = require('uuid');

async function createPayment(orderInfo) {
  const idempotencyKey = uuidv4();  // 客户端生成，本次重试/重发都用同一个
  return thriftPool.exec(PaymentService, {
    methodName: 'createPayment',
    header: { 'X-Idempotency-Key': idempotencyKey },
  }, idempotencyKey, orderInfo);
}
```

关键：**幂等键由客户端生成，同一次业务意图（哪怕被重试多次）用同一个键**。这样无论服务端收到几次，只真正执行一次。

### 服务端：幂等键 + 分布式锁

```javascript
async function createPayment(idempotencyKey, orderInfo) {
  const cacheKey = `payment:idempotency:${idempotencyKey}`;

  // 1. 先查：处理过就直接返回上次结果
  const cached = await redis.get(cacheKey);
  if (cached) return JSON.parse(cached);

  // 2. 加分布式锁，防并发重复处理
  const lockKey = `lock:payment:${idempotencyKey}`;
  const locked = await redis.set(lockKey, '1', 'PX', 5000, 'NX');
  if (!locked) throw new Error('操作正在处理中，请稍后重试');

  try {
    // 3. double-check（拿到锁后再查一次，防并发窗口）
    const recheck = await redis.get(cacheKey);
    if (recheck) return JSON.parse(recheck);

    // 4. 真正执行业务（用事务保证原子）
    const result = await db.transaction(async (trx) => {
      const balance = await getBalance(orderInfo.userId, trx);
      if (balance < orderInfo.amount) throw new Error('余额不足');
      await deductBalance(orderInfo.userId, orderInfo.amount, trx);
      return await createPaymentRecord({ ...orderInfo, idempotencyKey, status: 'SUCCESS' }, trx);
    });

    // 5. 缓存结果（设过期，比如 1 小时）
    await redis.setex(cacheKey, 3600, JSON.stringify(result));
    return result;
  } finally {
    // 6. 释放锁
    await redis.del(lockKey);
  }
}
```

几个要点：

- **double-check**：拿到锁后再查一次缓存。因为"查缓存没命中 → 加锁"之间有并发窗口，两个请求可能都查到没命中、都拿到锁（第二个其实拿不到，但 double-check 是双保险）。
- **业务用事务**：扣款 + 记录要么全成功要么全失败，不能扣了款没记下。
- **缓存结果**：让后续重复请求直接返回，不再执行业务。
- **锁要设过期**：防进程崩溃导致锁不释放。

## Mesh 模式下的额外注意

[Mesh 模式]( {% post_url 2025-01-24-mtfe-thrift-mesh %} ) 讲过：Mesh 模式下 SDK 重试只是对同一个 sidecar socket 重试，**不换实例**，而且 sidecar 自己也可能重试。这意味着非幂等操作的重试行为更不可控——你关了 SDK 的 retry，sidecar 那层可能还在重。

所以非幂等操作在 Mesh 环境下**更要靠业务幂等**，不能依赖关 SDK retry。header 里有个 `retryTime` 字段（`(options.retry||{}).retries || 1`）是告诉 sidecar 期望重试次数，非幂等操作可以把它设成 0/1 暗示 sidecar 少重试，但这只是建议，不能当硬保障。

## 监控：发现潜在的重复操作

即便做了幂等，也该有监控来发现"重复尝试"的信号——频繁的重复尝试说明链路有问题（超时多、重试多）。

简单做法：在客户端按"用户+金额+时间窗口"记最近一次操作，短时间内重复就告警。这只是发现信号，真正的去重还是靠上面的幂等键。

```javascript
const lastOp = new Map();
function detectDuplicate(userId, amount, windowMs = 60000) {
  const key = `${userId}:${amount}`;
  const now = Date.now();
  const last = lastOp.get(key);
  if (last && now - last < windowMs) {
    alert('可能的重复支付尝试', { userId, amount, gap: now - last });
  }
  lastOp.set(key, now);
}
```

## 小结

非幂等操作的一致性保障要分层做，不能指望 SDK：

1. **SDK 层**：非幂等操作关重试或重试 1 次，超时拉长——降低风险，非根治。
2. **业务层**：幂等键 + 服务端去重（查缓存 → 加锁 → double-check → 事务执行 → 缓存结果）——根本解。
3. **Mesh 层**：sidecar 也可能重试，更要靠业务幂等，不能依赖关 SDK retry。
4. **监控层**：发现重复尝试的信号，定位链路问题。

一句话：**SDK 的重试是为幂等场景设计的可用性保障；非幂等场景必须自己上幂等设计，把"是否已处理"的判断权拿到业务侧。** 别把一致性寄托在"关掉 retry 就安全"上——网络层任何一跳都可能重发。
