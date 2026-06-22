---
layout: post
title: "Agent 框架面试问答（一）：架构设计与规划层"
date: 2025-01-11
categories: 面试
tags: [AI, Agent, LLM, TypeScript]
---

> 基于简历「自主设计 Agent 框架与 AI 系统落地」项目推导，考察架构设计与规划层相关问题。

---

## Q1. 放弃 LangChain 的根因，以及自研架构解决了什么

**问题：** 你放弃 LangChain Agent 方案，理由是"流程确定性和步骤间数据传递不稳定"。能说说 LangChain ReAct Agent 在这方面的根因是什么？你自研的「规划 + 执行循环」架构在架构上具体解决了哪些 LangChain 没解决的问题？

---

### LangChain ReAct 的根因分析

**ReAct 的本质是一个"每步都由 LLM 自主决策"的循环**：

```
Thought → Action(tool, args) → Observation → Thought → ...
```

每一步 LLM 既要决定"做什么"，又要从上一步的 Observation 文本中"读取"数据传给下一步。这带来三个根本性问题：

**问题一：流程确定性缺失**

步骤顺序不是预定义的，是 LLM 每次推理时临时决定的。"是否继续"、"调用哪个 tool"、"何时输出 Final Answer"全部依赖 LLM 的 Thought，同一个问题不同推理结果可能不同。在"查询链接 → 补全参数 → 生成二维码"这类有严格顺序的多步任务中，LLM 可能跳步、重复调用同一个 tool、或者在参数还不完整时就提前输出 Final Answer。

**问题二：步骤间数据传递依赖 LLM 从文本中"理解"**

Tool 返回值作为字符串 Observation 追加到 prompt，下一步需要的参数要靠 LLM 从 Observation 文本里准确提取后填入下一次 tool call 的 args。这是额外的推理负担，链路越深错误率越高，且完全不可预测。

**问题三：没有暂停/恢复语义**

AgentExecutor 是阻塞执行到 Final Answer 或超出最大迭代数。如果执行到一半需要等用户输入参数，没有任何机制支持中途暂停然后从断点恢复。

---

### 自研架构如何解决这三个问题

**1. 流程确定性：一次性规划，严格顺序执行**

LangChain 是"规划和执行混在每步 LLM 推理里"，自研方案把两个阶段完全分开。Planner 单次 LLM 调用，输出固定的执行计划：

```typescript
// route-planner.service.ts
export interface RoutePlan {
  steps: RoutePlanStep[];  // 执行顺序固定
  confidence: number;      // 置信度不足时，LLM 自生成澄清问题
  clarificationNeeded?: string | null;
}
```

Executor 严格按 `plan.steps` 顺序执行：

```typescript
// route-execution-loop.service.ts
while (stepIndex < plan.steps.length) {
  const step = plan.steps[stepIndex];  // 顺序固定，不允许 LLM 跳步
  // ...
  stepIndex++;  // 只有成功才往前走
}
```

**2. 数据传递：显式数据链，代码级传递**

完全不依赖 LLM 从文本中"理解"上游产出。每个 capability 在注册表中声明 `produces`，执行成功后产出数据直接通过代码写入 `inheritedData`：

```typescript
// route-registry.ts — 声明式能力注册
{
  id: 'provide_link',
  produces: ['link', 'platform', 'linkType', 'linkName'],
},
{
  id: 'generate_qrcode',
  requires: ['link'],
}

// route-execution-loop.service.ts — 代码级数据传递
if (result.success) {
  Object.assign(inheritedData, result.produces);  // 产出直接写入，不经过 LLM
}

// 下一步执行时
const mergedSlots = { ...currentSlots, ...inheritedData };  // 上游产出直接可用
```

**3. 暂停/恢复：AsyncGenerator 协程**

用 `AsyncGenerator` 的 `yield` 语义实现两个暂停点：

```typescript
// route-execution-loop.service.ts
async *executeLoop(...): AsyncGenerator<LoopEvent> {
  while (stepIndex < plan.steps.length) {
    // 暂停点 1：参数缺失，等用户回复
    if (missingSlots.length > 0) {
      yield { type: 'slot_collection', stepIndex, missingSlots };
      return;  // 暂停，状态序列化到 Redis
    }
    // 暂停点 2：需要用户填结构化表单
    if (error.type === 'param_missing' && sd?.['fields']) {
      yield { type: 'interaction', stepIndex, interactionFields, executorState };
      return;  // 暂停，executorState 序列化到 Redis
    }
  }
  yield { type: 'done', inheritedData };
}
```

暂停时将 `{ plan, stepIndex, inheritedData, executorState }` 写 Redis，恢复时从 `startStepIndex` 重建执行上下文继续跑：

```typescript
// dialogue-manager.service.ts
await this.runExecutionLoop(
  sessionId,
  paused.plan,           // 原计划不变
  paused.originalMessage,
  ...,
  paused.stepIndex,      // 从断点继续，跳过已完成步骤
  paused.inheritedData,  // 已完成步骤的产出数据恢复
);
```

---

### 一句话对比

| 问题 | LangChain ReAct | 自研方案 |
|------|----------------|---------|
| 流程确定性 | LLM 每步动态决策，顺序不固定 | 单次规划输出固定步骤数组，Executor 严格顺序执行 |
| 步骤间数据传递 | Tool 返回文本，下步靠 LLM 从 Observation 提取 | `produces` 声明式注册，代码级 `Object.assign` 传递，零 LLM 依赖 |
| 暂停/恢复 | 不支持，必须一次执行到底 | AsyncGenerator 两个 yield 暂停点，Redis 序列化中间态 |
| 交互表单 | 无原生支持 | `ExecutorState` 记录参数缺失，生成结构化表单，提交后从断点继续 |

**核心思路：把"LLM 擅长的"（理解意图、生成计划、填充槽位）和"LLM 不擅长的"（保证顺序、传递数据、管理状态）彻底拆开。**

---

### 追问

**Q：暂停状态序列化到 Redis，如果 Redis 在暂停期间挂了，用户的多步任务怎么处理？**

坦诚说：当前没有针对这个场景的专门处理。Redis 故障时 `setPausedExecution` 会抛异常，用户的多步任务状态丢失，需要重新开始对话。可以接受的理由是：营销助手的任务时长通常在数分钟内，Redis 在这个窗口内故障的概率很低；且任务本身是幂等的，用户重试成本低。

**Q：`produces` 声明在注册表里，Executor 执行时怎么保证一定把对应 key 写进 `inheritedData`？有类型约束吗？**

目前是靠约定，没有编译期类型约束。`RouteStepResult.produces` 的类型是 `Record<string, SlotValue>`，executor 返回什么 key 都合法，运行时不会校验它是否和注册表里声明的 `produces` 字段一致。这是一个真实的设计缺口——如果 executor 忘记返回某个 key，下游步骤拿到 `undefined` 时才会暴露问题。可以通过在 `executeLoop` 里加 post-execution 校验来检测，但目前没做。

**Q：用户在 `slot_collection` 暂停阶段改变了意图，框架如何处理？**

`interaction` 暂停有意图变更检测：恢复时会重新调用 Planner，如果新消息的意图和暂停时的步骤不一致（`isSameIntent === false`），清除暂停状态按新意图重新规划执行。`slot_collection` 暂停则没有意图检测——直接把用户消息作为槽位值提取，继续原计划。用户说"算了不要了"时，NLU 提取不到槽位值，仍然会追问。这是个已知的用户体验缺陷，理想做法是在 `slot_collection` 恢复路径上也做一次意图分类。

---

## Q2. 单次规划的 Token 上限风险，以及为何不采用流式规划

**问题：** Planner 阶段你说是"单次 LLM 调用输出含步骤依赖关系和置信度评估的完整执行计划"，这意味着你把所有步骤规划都压缩进一次推理。如果用户任务非常复杂，导致输出 Token 超出限制，你怎么处理？有没有考虑过流式规划（边规划边执行）的方案，为何没采用？

---

### 为什么 Token 超限在这个场景几乎不会发生

Token 超限有两种情况——**输入超限**和**输出超限**。

**输出超限几乎不可能。** RoutePlan 的结构极其紧凑：

```typescript
export interface RoutePlan {
  steps: RoutePlanStep[];
  confidence: number;
  clarificationNeeded?: string | null;
}
export interface RoutePlanStep {
  routeId: string;
  inheritFrom?: string | null;
  reason?: string;
}
```

能力注册表是有界的（当前 7 个顶层 capability），即使用户任务再复杂，执行计划最多也就 3-4 步，每步就是一个 routeId + 可选的 reason 短语，大约几百个 token，远低于输出限制。

**输入超限有一定压力**，但能力注册表是静态的，不会随任务复杂度增长，也是有界的。

---

### 如果 LLM 输出异常，实际的兜底链路

通过 `StructuredLlmCaller` 做了多层保护：

**第一层：三层 JSON 提取**（直接解析 → Markdown 代码块提取 → 正则提取），覆盖 LLM 输出不规范的情况。

**第二层：Schema 校验错误注入重试**（`maxRetries: 1`），把 Zod 校验失败的详情注回 Prompt，让 LLM 自纠错。

**第三层：FALLBACK_PLAN 降级**，所有重试都失败时：

```typescript
const FALLBACK_PLAN: RoutePlan = { steps: [], confidence: 0 };
```

`confidence: 0` + `steps: []` 会触发澄清追问流程，向用户确认意图，而不是带着错误的计划继续执行。这是"fail safe"而非"fail silent"的设计。

---

### 为什么没有采用流式规划

**问题一：全局依赖关系无法分析**

`RoutePlanStep` 有 `inheritFrom` 字段，需要 Planner 在规划时"看到全局"才能正确标注：

```typescript
steps: [
  { routeId: 'provide_link' },
  { routeId: 'generate_qrcode', inheritFrom: 'provide_link' }  // 依赖关系在规划时确定
]
```

流式规划每次只看到"当前步骤完成了什么"，无法提前标注后续步骤的 `inheritFrom`，要么退化为每步都依赖 LLM 重新推断数据来源。

**问题二：每步多一次 LLM 调用，延迟和成本都升高**

一次性规划是 1 次 LLM 调用；流式规划是 N 次。对于 3 步的任务，这意味着 3 倍的规划延迟和 token 消耗，但收益几乎没有——因为步骤数量本来就有界。

**问题三：暂停/恢复状态管理会显著复杂化**

现在的暂停状态快照非常干净：`{ plan, stepIndex, inheritedData, executorState }`。如果改成流式规划，恢复时要区分"已规划已执行"、"已规划未执行"、"未规划"三种状态，逻辑复杂度大幅上升。

**已有的折中方案：流式推送思考过程**

规划阶段用 `streamChat`，通过 `onDelta` 回调把思考 token 实时推送到前端，满足"让用户看到进展"的需求，同时规划结果本身仍然是一次性完整输出，兼顾了用户体验和架构简洁性。

---

### 追问

**Q：规划阶段开启了流式推送思考过程（thinking token），这部分 token 和最终输出的计划 JSON 共享 `max_tokens: 700` 限制吗？**

这是个真实的隐患。`max_tokens: 700` 是传给模型的 completion 上限，thinking token 和输出 JSON 共享这个预算。deepseek-v3.2 的 thinking 内容通常在 200-400 token 之间，留给 JSON 计划的空间就压缩到了 300-500 token——按当前 2-3 步的计划量还够用，但余量不大。如果 thinking 内容特别长，JSON 可能被截断。

**Q：Schema 校验错误注入重试只有 `maxRetries: 1`，只重试一次，理由是什么？**

只重试一次是在响应延迟和可靠性之间的权衡。实践中这一次重试能覆盖大多数格式问题（LLM 偶发性输出了注释、多余字段等），成功率很高。如果第二次仍然失败，触发 FALLBACK_PLAN，用户看到澄清追问。相比加到 2-3 次重试，延迟多 1-2 秒，但成功率提升有限。

**Q：规划层选用 deepseek-v3.2，选型依据是什么？如果换成 Claude 或 GPT-4，架构上需要改什么？**

选 deepseek-v3.2 主要是成本。规划任务是高频的，对推理深度要求不高，主要是结构化 JSON 输出，deepseek 在这类任务上性价比很高。架构上换模型几乎不需要改动——模型配置在 Prompt seed 的 `modelConfigJson` 字段里，LLM Service 已经有模型适配层。唯一需要关注的是 thinking token 的格式：`consumeLlmStream` 里的 `thinkingMode` 解析逻辑是针对 deepseek 格式写的，换 Claude 的 extended thinking 需要调整这部分解析。

---

## Q3. 能力注册表序列化为 System Prompt 的 Token 控制

**问题：** 声明式能力注册表最终序列化为 System Prompt，如果注册了 20+ 条能力，System Prompt 会非常长。你是如何控制 Token 消耗的？动态注册的能力项在 LLM 推理时有没有出现过被截断或混淆的问题？

---

### 当前状态和实际约束

**约束一：规划 Prompt 的输出 token 有硬性上限**

```typescript
modelConfigJson: JSON.stringify({
  model: 'deepseek-v3.2',
  temperature: 0.1,
  max_tokens: 700,
}),
```

**约束二：序列化只保留 LLM 规划所需的字段，Slots 细节不进 Prompt**

`buildDescriptionsFromCapabilities` 只序列化 `id`、`name`、`description`、`requires`、`produces`，不包含 `slots` 的具体定义：

```typescript
export function buildDescriptionsFromCapabilities(capabilities: RouteCapability[]): string {
  return capabilities
    .map(
      (r, i) =>
        `${i + 1}. **${r.id}** - ${r.name}\n   ${r.description}` +
        (r.requires.length ? `\n   需要前置数据：${r.requires.join(', ')}` : '') +
        (r.produces.length ? `\n   产出数据：${r.produces.join(', ')}` : ''),
    )
    .join('\n\n');
}
```

Slots 的完整定义（参数名、类型、枚举值、prompt 话术）只在执行阶段 Executor 内使用，不进入规划 Prompt。这是有意的关注点分离：Planner 只需要知道"能做什么、需要什么、产出什么"，不需要知道"每个参数怎么填"。

---

### 如果真的扩展到 20+ 能力，架构上预留了什么

**预留一：subRoutes 嵌套结构，支持两阶段规划**

类型定义已经支持 dispatcher route + subRoutes 的层级设计：

```typescript
export interface RouteCapability {
  subRoutes?: RouteCapability[];  // 非空则为 dispatcher，不是叶子路由
}

export function findRouteCapabilityDeep(id: string): RouteCapability | undefined {
  function search(list: RouteCapability[]): RouteCapability | undefined {
    for (const cap of list) {
      if (cap.id === id) return cap;
      if (cap.subRoutes) {
        const found = search(cap.subRoutes);
        if (found) return found;
      }
    }
    return undefined;
  }
  return search(ROUTE_CAPABILITIES);
}
```

如果能力增长到 20+，可以将其组织为大类（如 `link_operations`、`content_operations`），Planner 第一步只看顶层大类，选中大类后再做第二次规划，每次 LLM 看到的能力列表始终保持在 10 条以内。

**预留二：意图分类作为前置过滤层**

代码中已经有独立的意图分类 Prompt，在 Planner 之前运行。如果能力扩展，可以在意图分类阶段先收窄候选集，只把相关能力传给 Planner。

---

### 截断和混淆问题有没有出现过

**截断：没有出现过。**

**混淆：出现过，根因不是能力列表太长。** 早期曾出现 LLM 把 `convert_direct_url` 和 `provide_link` 混用——两者描述都涉及"链接"，语义相似。解决方式是精化能力的 `description` 字段，用更具体的触发场景区分两者：

```
// provide_link：用户说"给我XXX链接"、"我要XXX的链接"，从零获取链接
// convert_direct_url：用户已有一个原始链接，需要转换成平台专属格式
```

混淆的根因是**描述不够精确**，而非能力列表太长。能力数量增加时，每条 description 的质量比控制总量更重要。

---

### 追问

**Q：description 字段是工程师凭经验写的，还是有验证流程？出现混淆后怎么知道是 description 不够精准导致的？**

目前是工程师凭经验写，没有自动化验证流程。判断是 description 问题还是模型推理问题的依据：可观测日志里记录了每次规划的 `steps[].routeId`，对比用户的原始消息和实际选中的 routeId，如果选错是**系统性的**（同类表述总是选错同一个 capability），大概率是 description 区分度不够；如果是随机的，才可能是模型抖动。

**Q：能力注册表现在是代码常量，如果运营想通过管理平台动态新增能力，现有架构支持吗？**

描述类字段（id、name、description、requires、produces、slots 定义）可以动态化，改造成数据库表 + 缓存热加载不复杂，`buildRouteDescriptionsForPrompt` 改成异步读库就行。难点在 executor：每个 capability 需要一段业务逻辑代码（`callExecutor` 的 switch 分支），这部分没法从管理平台配置出来。所以动态化只能覆盖"纯信息查询型"能力，对需要调用外部 API、有复杂参数处理逻辑的能力，仍然需要代码发布。

**Q：意图分类（`MarketingDialogueIntentClassification`）和规划（Planner）是两次 LLM 调用，有没有考虑过合并成一次？为什么分开？**

考虑过，最终分开的原因是职责差异大。意图分类输出的是一个简单枚举（`new` / `continue` / `follow_up` / `chitchat` 等），temperature 可以调很低，模型用轻量的就够；规划输出的是复杂 JSON，需要理解所有能力描述，对模型能力要求高。合并成一次要么用大模型做分类（浪费），要么用小模型做规划（不稳），分开可以对每段任务独立选模型和参数。另外分开后意图分类失败不影响规划日志，更容易排查问题。
