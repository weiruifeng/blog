---
layout: post
title: "Agent 框架面试问答（二）：Generator 协程与数据链"
date: 2025-01-12
categories: 面试
tags: [AI, Agent, LLM, TypeScript, Redis]
---

> 基于简历「自主设计 Agent 框架与 AI 系统落地」项目推导，考察 Generator 协程、暂停/恢复机制与步骤间数据链相关问题。

---

## Q4. Generator 协程与异步 I/O，以及 Redis 中间态序列化

**问题：** 你用 Generator 实现"暂停/恢复语义"，但 Generator 本身是同步的，你如何配合异步 I/O（比如等待用户填写结构化表单）？中间态序列化到 Redis 时，Generator 的状态本质上是什么——你序列化的是哪些数据，不序列化哪些？

---

### Generator 是同步的——架构上如何拆解这个问题

关键在于：**AsyncGenerator 的暂停点不等于"等待异步 I/O 完成"**，两者是两个独立的机制。

执行层使用 `AsyncGenerator`，通过 `yield` 语义向上层抛出事件：

```typescript
// route-execution-loop.service.ts
private async *executeLoop(...): AsyncGenerator<LoopEvent> {
  while (stepIndex < plan.steps.length) {
    // 暂停点 1：槽位缺失，yield 后 return，生成器终止本次迭代
    if (missingSlots.length > 0) {
      yield { type: 'slot_collection', stepIndex, missingSlots };
      return;
    }

    // 异步 I/O 在生成器内部是正常 await，不是暂停点
    result = await this.callExecutor(step.routeId, mergedSlots, execCtx);

    // 暂停点 2：executor 要求用户填表单，yield 后 return
    if (error.type === 'param_missing' && sd?.['fields']) {
      yield { type: 'interaction', stepIndex, interactionFields, executorState };
      return;
    }

    Object.assign(inheritedData, result.produces);
    stepIndex++;
  }
  yield { type: 'done', inheritedData };
}
```

关键设计：**`yield` 之后紧跟 `return`，生成器本次执行结束，不再挂起等待**。这不是"暂停在中间"，而是"有序地终止，把状态交出去"。

---

### 异步 I/O 发生在哪——生成器之外

真正的"等待异步 I/O"发生在上层的 `dialogue-manager.service.ts`：

```typescript
for await (const event of this.executionLoop.execute(plan, execCtx, currentSlots, startStepIndex)) {
  switch (event.type) {
    case 'slot_collection': {
      // 生成器已经 return 了，这里是纯异步代码空间
      const extracted = await this.nluService.extractSlots(...);
      if (stillMissing.length === 0) {
        // 提取成功，直接从同一步骤重启一个新的生成器执行
        await this.runExecutionLoop(sessionId, plan, ..., event.stepIndex, ...);
        return;
      }
      // 提取失败，追问用户——把状态序列化到 Redis，HTTP 响应返回给前端
      await this.contextManager.setPausedExecution(sessionId, { plan, stepIndex: event.stepIndex, ... });
    }

    case 'interaction': {
      // 生成器已 return，等待用户提交表单
      // 这是跨 HTTP 请求的等待——当前请求结束，下一次请求才会恢复
      await this.contextManager.setPausedExecution(sessionId, { plan, stepIndex: event.stepIndex, ... });
    }
  }
}
```

**等待用户填写表单**的场景，等待跨越了两个 HTTP 请求：
- 第一个请求：识别到 `interaction` 事件，把状态存 Redis，返回表单给前端
- 用户填写表单、提交，触发第二个请求（`handleInteractionSubmit`）
- 第二个请求：从 Redis 取回状态，验证参数完整性，触发第三个请求恢复执行

**等待期间 generator 根本不存在**——它已经 `return` 了。

---

### Redis 序列化的是什么，不序列化什么

**序列化的数据**（`PausedExecution` 接口，全部可 `JSON.stringify`）：

```typescript
export interface PausedExecution {
  plan: RoutePlan;                           // 执行计划（步骤数组，纯数据）
  stepIndex: number;                         // 暂停在哪一步
  inheritedData: Record<string, SlotValue>;  // 已完成步骤的产出数据
  pauseReason: 'slot_collection' | 'interaction';
  originalMessage: string;
  waitingForSlots?: SlotDefinition[];
  interactionFields?: InteractionField[];
  executorState?: ExecutorState;             // executor 中间状态
}

// ExecutorState 是最典型的"中间状态"快照
export interface ExecutorState {
  linkKey: string;
  variantPlatform: string;
  targetPlatform: string | null;
  collectedValues: Record<string, string | null>;
  remainingRequired: string[];
  queryDefs: Array<{ key: string; required: boolean; ... }>;
  pathKeys: string[];
}
```

**不序列化的数据**：

- **服务依赖（`ExecutorDeps`）**：LLM 服务、数据库 handler——这些是 NestJS 注入的 class 实例，不可序列化。恢复时由框架重新注入。
- **AsyncGenerator 实例本身**：JavaScript 的 Generator 对象包含执行上下文（词法环境、调用栈），无法序列化。这也是为什么暂停不是"挂起 generator"，而是"return 后用序列化数据重建"。
- **EventEmitter / Stream 对象**：SSE 推流的 subscriber 是跨 HTTP 请求的连接对象，恢复时由新请求重新建立。

**本质上序列化的是"重启一个等价执行器所需的最小数据集"**，不是"保存执行器的运行时状态"。恢复时不是"唤醒"原来的 generator，而是用快照数据重建一个新的 generator 从 `startStepIndex` 开始跑。

---

### 两种暂停场景对比

| 场景 | 等待什么 | 跨越几个 HTTP 请求 | 恢复触发点 |
|------|---------|-----------------|-----------|
| `slot_collection` | 用户在对话框输入参数 | 2 | 下次对话消息到达 |
| `interaction` | 用户填写结构化表单 | 3 | `handleInteractionSubmit` + 后续 AI 请求 |

---

### 追问

**Q：为什么用 AsyncGenerator 而不是普通的状态机 + async/await？Generator 带来了什么额外收益？**

用 AsyncGenerator 的核心收益是**代码里 `yield` 的位置就是暂停点**，不需要额外维护一个状态枚举来描述"执行到哪了"。用状态机的话，每次新增暂停点都要同步修改状态枚举、转移条件、恢复逻辑三处；用 Generator 只需在代码里加一个 `yield`，暂停语义就是自文档的。

**Q：`PausedExecution` 在 Redis 里没有显式 TTL，用户如果永远不回来填表，这个 key 会一直存在吗？**

会，但上限是平台级别的 30 天 TTL，由 Redis 基础设施统一管理。30 天后 key 自动过期，下次用户请求时走重新开始的路径。如果业务上需要更短的超时（比如 10 分钟不填就取消），需要在写 Redis 时显式带 `EX` 参数，目前没做。

**Q：`ExecutorState` 里保存了 `queryDefs`，如果 catalog 的参数定义在暂停期间变了，用保存的旧 `queryDefs` 构建表单，会和新 catalog 产生不一致吗？**

会，这是一个已知盲区。`resumeFromExecutorState` 重新拉取 catalog 是为了拿最新的 `item`（校验 linkKey 存在），但后续构建表单用的还是 `state.queryDefs`（暂停时序列化的旧定义）。如果参数定义变了，用户填的表单字段可能和新模板不匹配，最终 URL 可能带错误参数。这个场景在营销配置的实际运营中极少发生，所以接受了这个风险。

---

## Q5. 暂停期间上游数据失效的感知与处理

**问题：** 恢复执行时"跳过已完成的 LLM 推理阶段，节省 token 且避免识别结果漂移"——如果上游能力的产出数据在暂停期间发生了外部变化，你的架构如何感知并处理这种失效？

---

### 直接说结论：架构当前没有通用的失效感知机制

坦诚说：**`inheritedData` 里保存的上游产出在恢复时会被原样复用，不会重新验证**。恢复入口：

```typescript
// dialogue-manager.service.ts
await this.runExecutionLoop(
  sessionId,
  paused.plan,
  paused.originalMessage,
  subscriber,
  traceId,
  paused.stepIndex,
  { ...paused.inheritedData },  // 快照数据，无失效期检查
  ...
);
```

如果 `inheritedData.link` 在暂停期间失效了，恢复后的下游步骤会拿着这个失效的 URL 继续执行，直到外部 API 报错才暴露问题。

---

### 有一个局部防护：catalog 在恢复时会重新拉取

`interaction` 暂停恢复后，executor 会通过 `resumeFromExecutorState` 重新拉取 catalog：

```typescript
async function resumeFromExecutorState(state, slots, deps) {
  const allItems = await deps.catalog.getAll();  // 重新拉取 catalog
  const item = allItems.find((i) => i.key === state.linkKey);
  if (!item) {
    return {
      success: false,
      displayContent: `抱歉，没有找到链接配置：${state.linkKey}`,
      error: { type: 'not_found', recoverable: false },
    };
  }
  // 后续用 state.collectedValues 继续
}
```

这个机制能覆盖**最极端的失效场景**：如果整个 catalog 条目被运营删掉了，恢复时会拿到 `not_found` 错误，用户看到明确的错误提示。

---

### 为什么这个设计缺口在当前场景可以接受

营销助手处理的是"链接配置"，这些数据由运营人员在后台维护，变更频率以天为单位，不是实时波动的票价或库存。暂停窗口是用户填表单的时间，通常是秒到分钟级别。**在这个时间窗口内，一个已确认存在的 catalog 条目内容发生变化的概率极低。**

此外，`persistentEntities` 里保存了 `timestamp`，这是后续加入 TTL 检查的基础设施：

```typescript
context.persistentEntities[key] = {
  value: String(inheritedData[key]),
  source: '...',
  timestamp: Date.now(),  // 记录了产出时间，后续可加有效期校验
};
```

---

### 如果真的需要感知失效，扩展路径

**方案一：catalog 版本号校验**

在 `ExecutorState` 里保存 catalog 条目的版本号或内容哈希，恢复时比较，不一致则清除 `executorState`，让 executor 从头执行。

**方案二：`inheritedData` 的 TTL 标记**

给 `persistentEntities` 里的每个 key 设置 maxAge：

```typescript
if (entities[key]?.value && Date.now() - entities[key].timestamp < MAX_ENTITY_AGE_MS) {
  result[key] = entities[key].value;  // 在有效期内才继承
}
```

**方案三：executor 失败时降级重试上游**

如果下游 executor 调用外部 API 失败，且失败原因可能是上游数据失效，可以在错误处理里清除 `inheritedData` 里的相关 key，并把 `stepIndex` 回退到上游步骤重新执行。

---

### 失效场景总结

| 失效场景 | 当前能否感知 | 暴露时机 |
|---------|------------|---------|
| catalog 条目被完全删除 | ✅ 能感知 | 恢复时 `not_found` 错误 |
| catalog 条目内容变更 | ❌ 感知不到 | 生成 URL 后业务侧报错 |
| 上游产出 URL 本身失效 | ❌ 感知不到 | 下游 API 调用时报错 |
| 跨任务继承的历史数据过期 | ❌ 感知不到 | 静默继承 |

---

### 追问

**Q：`persistentEntities` 设计时就带了 `timestamp` 字段，但从来没有检查过，为什么当时加了却没用上？**

加 `timestamp` 是预留给后续 TTL 校验用的，但一直没有出现具体的业务诉求来触发实现。这是"先埋基础设施、按需实装"的惯用模式，但确实也有变成永远不实装的风险。

**Q：如果下游 executor 调外部 API 失败，重试是用同一份 `inheritedData` 重试的——如果失败根因是上游数据失效，重试也一定失败。有没有考虑过重试时清除 `inheritedData` 里的相关 key？**

考虑过，但没有实现，原因是区分成本高。`api_error` 既可能是网络抖动（重试同样数据就能成功），也可能是数据失效（需要清除重跑上游）。要区分这两种场景，要么依赖外部 API 返回特定错误码，要么给每种 error 类型单独标注"是否需要重拉上游"。当前的 `retryable: true` 只表示"可以重试"，没有携带"重试策略"的语义。

---

## Q6. 数据链拼装复杂度管理

**问题：** "步骤间通过显式数据链传递上游产出，不依赖 LLM 上下文记忆"——这意味着每一步的 Prompt 都需要手动注入上游结果。当依赖链很深时，Prompt 拼装逻辑复杂度如何管理？有没有出现过数据链拼装错误导致 LLM 误判的情况？

---

### 数据链的核心机制：一个 `Object.assign`

复杂度控制的关键在于**数据链传递是代码层面的，不是 Prompt 层面的**：

```typescript
// route-execution-loop.service.ts
// 步骤执行前：合并当前任务槽位 + 所有上游产出
const mergedSlots: SlotValues = { ...currentSlots, ...inheritedData };

// 步骤执行成功后：上游产出写入 inheritedData
if (result.success) {
  Object.assign(inheritedData, result.produces);  // 直接 merge，不经过 LLM
  stepIndex++;
}
```

每个 executor 接收到的 `mergedSlots` 是一个**平铺的 key-value 字典**，executor 只需按 key 取值，不需要知道数据来自哪个步骤：

```typescript
// route-executors.ts — generate_qrcode executor
export async function generateQrcodeExecutor(slots, ctx, deps): Promise<RouteStepResult> {
  // 三层取值：当前槽位 → 备用字段 → 继承数据（兜底）
  const link =
    (slots.link as string) ||
    (slots.directPath as string) ||
    (ctx.inheritedData.link as string);

  const result = await deps.qrcodeService.generate(link);
  return { success: true, produces: { qrcodeUrl: result.qrcodeUrl, link } };
}
```

**没有 Prompt 层面的数据注入**——`generate_qrcode` 的执行不需要把"第一步获取了什么链接"写进 Prompt，因为 `link` 已经通过代码传进来了。

---

### Prompt 层面的数据注入只发生在两个地方

**场景 1：NLU 槽位提取**（需要 LLM 从用户消息中理解并填充参数）

注入的是**平铺 JSON**，格式固定，不随链路深度变化：

```typescript
const vars = {
  message,
  slotDefinitions: this.formatSlotDefinitions(slotDefinitions),
  conversationHistory: conversationHistory || '',
  currentSlots: currentSlots ? JSON.stringify(currentSlots, null, 2) : '',
};
```

**场景 2：意图分类和路由规划**（需要 LLM 理解用户意图并选择能力）

规划 Prompt 注入的是**静态的能力注册表**，和执行链路深度无关——无论执行了几步，规划 Prompt 的格式始终一致。

**结论：Prompt 拼装没有随链路深度线性增长的部分。**

---

### 有没有出现过数据链拼装错误

**出现过一次**，根因在 executor 的取值优先级设计。取值逻辑是：

```typescript
const link =
  (slots.link as string) ||
  (slots.directPath as string) ||
  (ctx.inheritedData.link as string);
```

问题出在 `||` 的短路求值：**如果 `slots.link` 是一个空字符串 `''`，JavaScript 会认为它是 falsy，跳到第三层去取 `inheritedData.link`**。在一次测试中，用户输入了空字符串，`slots.link` 被设为 `''`，导致 executor 错误地取了上一次任务遗留在 `inheritedData` 里的历史链接，生成了错误的二维码。

修复方式是改为显式 null/undefined 判断：

```typescript
const link =
  (slots.link != null && slots.link !== '' ? slots.link as string : null) ||
  (slots.directPath != null && slots.directPath !== '' ? slots.directPath as string : null) ||
  (ctx.inheritedData.link as string);
```

这次问题的根因是**取值逻辑没有正确区分"用户主动设空"和"用户没有提供"**，不是数据链本身的问题。

---

### 复杂度管理的两个关键约束

**约束一：`produces` 声明式注册，key 扁平化**

所有步骤的产出都平铺进同一个 `inheritedData` 字典，没有嵌套结构。下游步骤按 key 取值，不需要了解数据的来源步骤。当前最长链路是 3 步，`inheritedData` 最多也就 4-6 个 key，完全没有管理压力。

**约束二：`inheritFrom` 显式声明依赖**

规划阶段 LLM 输出的计划里，每一步的 `inheritFrom` 字段标明它依赖哪个前置步骤的产出，依赖关系在规划时就定下来，不靠运行时猜测。

---

### 追问

**Q：`Object.assign(inheritedData, result.produces)` 是后者覆盖前者的。如果两个步骤都产出同名 key，后者会覆盖前者，这是预期行为吗？**

是预期行为，并且是有意设计的。`generate_qrcode` 产出的 `link` 和 `provide_link` 产出的 `link` 是同一个值（executor 里把入参的 `link` 原样写进 `produces` 往下传），覆盖后内容不变。这样设计的好处是：第三步只需从 `inheritedData` 取 `link`，不需要知道这个值是第一步直接产出的还是中间步骤透传的。

**Q：空字符串导致 `||` 短路的 bug 修复后，有没有补单元测试？**

修复后补了。对 `generateQrcodeExecutor` 的取值逻辑加了三个边界用例：`slots.link` 为空字符串时走继承数据、`slots.link` 为有效值时优先使用、`inheritedData.link` 为兜底时的行为。这类"容易被 `||` 短路坑的取值逻辑"在测试里专门作为边界用例记录，防止将来重构时引入同类问题。

**Q：`produces`/`requires` 是人工在注册表里维护的，和 executor 实际实现之间没有编译期约束。有没有更早的检测手段？**

目前没有，这是一个设计缺口。能做但没做的最简单方案：在 `executeLoop` 里加 post-execution 断言，对比 `result.produces` 的实际 key 集合和 capability 声明的 `produces` 数组，不一致时打 warn 日志。这样至少在测试环境跑一次就能暴露不一致，不需要等到下游步骤取到 `undefined` 才发现。没有实现的理由是规模还小——7 个 executor 每次改动都有人工 review，注册表和实现不一致很难漏进去。
