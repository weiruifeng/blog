---
layout: post
title: "Agent 框架面试问答（六）：LLM 测试、复现与模型评估"
date: 2025-01-16
categories: 面试
tags: [AI, Agent, LLM, 测试, 模型评估]
---

> 基于简历「自主设计 Agent 框架与 AI 系统落地」项目推导，考察线上错误复现、多模型对比评估、Prompt 回归保障与 LLM 不确定性测试策略。

---

## Q15. 线上错误场景复现与 Prompt 快照

**问题：** 你的 Agent 出现一次线上错误（比如 Planner 在某个特定用户输入下输出了错误的步骤依赖），你如何完整复现这个场景？你是否保存了完整的 Prompt 快照（包括 System Prompt、注入的数据链内容、上下文历史）？如果当时的 LLM 版本已被模型厂商静默更新，复现结果是否还有参考价值？

---

### Prompt 快照的完整性

`LlmLog` 表是复现的核心工具，保存了每次 LLM 调用的完整上下文：

```typescript
// llm-log.entity.ts
export class LlmLog extends Model {
  promptMessages: Array<{ role: string; content: string }>; // 完整消息链（含 System Prompt + 历史 + 当前消息）
  llmResponse: string;     // LLM 原始输出（未经任何处理）
  modelConfig: Record<string, any>; // { model, temperature, max_tokens, ... }
  llmModel: string;        // 使用的模型名
  llmProvider: string;     // 使用的 provider（volces / deepseek）
  tokenUsage: { promptTokens, completionTokens, totalTokens };
  durationMs: number;
  traceId: string;         // 关联同一次请求的所有日志
  sessionId: string;
  promptId: string;        // 是哪个 Prompt 模板
  isReplay: number;        // 标记这次是否是回放调用
}
```

`promptMessages` 存的是**最终发给 LLM 的完整消息数组**，包含：
- 第一条：System Prompt（含能力注册表序列化后的内容）
- 中间条：对话历史（已截断到 maxTurns 条）
- 最后一条：当前用户消息（含注入的变量，如 `_retryError`、`routeDescriptions`）

这意味着拿到一条 `LlmLog` 记录，就能完整还原当时 LLM 看到的输入。

---

### 有 Replay 接口支持本地重跑

系统提供了 `/llm-log/replay` 接口，直接用保存的 `promptMessages` 重发给 LLM：

```typescript
// llm-log.service.ts
async replay(id: number): Promise<{ content: string }> {
  const log = await this.llmLogModel.findByPk(id);
  const messages = log.promptMessages;

  const systemMsg = messages.find((m, i) => i === 0 && m.role === 'system');
  const lastUserMsg = messages[lastUserIndex];
  const historyMsgs = messages.filter((_, i) => /* 中间历史 */);

  return this.llmService.chat({
    appkey: log.appKey,
    promptId: log.promptId,
    systemPrompt: systemMsg?.content ?? '',
    userPrompt: lastUserMsg?.content ?? '',
    messages: historyMsgs,
    model: model || log.llmModel,  // 可指定不同模型对比
    llmCallContext: { traceId: log.traceId, isReplay: true },
  });
}
```

复现流程：找到对应 traceId 的 `LlmLog` 记录 → 调用 `/replay` → 对比原始输出和复现输出。

---

### LLM 版本静默更新后复现的参考价值

**有一定参考价值，但不等于完全复现。**

`LlmLog` 保存的是模型名（如 `deepseek-v3.2`），不保存模型的权重哈希或版本戳。如果 deepseek 在后台静默更新了 `v3.2` 的权重，用相同的 `promptMessages` 调用同名模型，输出的 token 概率分布可能已经变化。

对于 **Planner 输出了错误步骤依赖**这类问题：
- 如果根因是 **Prompt 描述歧义**（能力 description 写得不够区分），模型版本不影响复现——旧版本和新版本都会在同样歧义下犯同样的错，复现是有意义的
- 如果根因是**模型本身的某个特定 bug**（某类 token 序列触发了错误输出），模型更新后 bug 可能已经不存在，replay 结果和当时不同，无法复现

实践中的做法：replay 结果用于**验证 Prompt 修复方向是否正确**，而不是精确还原当时的错误。如果 replay 结果正常但当时确实出错了，说明错误是模型偶发的或已随版本消失，关注修复已有的 Prompt 歧义即可。

---

### 追问

**Q：traceId 能把哪些日志关联起来？一次 Planner 出错，能通过 traceId 找到所有相关日志吗？**

一次用户消息处理产生的所有 LLM 调用（意图分类、槽位提取、路由规划）共享同一个 `traceId`，都写在 `LlmLog` 表里。路由步骤的执行结果写在 `RouteCallLog` 表里，也带同样的 `traceId`。通过 `traceId` 可以找到：当时的意图分类结果是什么、槽位提取了哪些值、Planner 看到的完整 Prompt 和输出是什么、每个 executor 的入参和产出。基本能还原完整的处理链路，唯一缺的是 Redis context 的快照（Redis 只保留最新状态，不保留历史快照）。

**Q：`LlmLog` 存的是完整的 `promptMessages`，用户的原始消息也被存到了日志里。有没有做脱敏处理？**

目前没有脱敏，用户消息以明文存入 `LlmLog`。这是一个已知的数据合规问题。营销助手的用户消息大多是营销任务指令（"给我领券链接的二维码"），不含敏感个人信息，目前接受了这个风险。如果接入的场景涉及个人信息，需要在写入前做内容过滤或字段级脱敏。

---

## Q16. 多 LLM 对比评估

**问题：** 你如何对比不同 LLM（比如 GPT-4o 和 Claude 3.5）在你的 Agent 框架上的实际效果？评估维度是什么——是人工打分、规则校验 Schema 合规率，还是基于 LLM 的自动评判（LLM as Judge）？当两个模型的评分在大多数样本上接近，只有少数样本差异显著时，你如何决策选型？

---

### 当前的对比方式

系统支持多个 provider 和模型，可以在 Prompt seed 的 `modelConfigJson` 里切换：

```typescript
// marketing-dialogue.seeds.ts
modelConfigJson: JSON.stringify({
  model: 'deepseek-v3.2',  // 改这里切换模型
  temperature: 0.1,
  max_tokens: 700,
}),
```

切换模型后，`LlmLog` 会记录每次调用使用的 `llmModel` 和 `llmProvider`，可以按模型名分组查询历史数据对比。但**没有系统化的评估流水线**，目前的模型对比是经验性的：换模型跑几十个典型任务，人工看 Planner 的输出计划是否合理，没有自动化打分。

---

### 评估维度与选型依据

实际做过的对比集中在两个维度：

**维度一：Schema 合规率**（规则校验，可自动化）

Planner 输出的 `RoutePlan` 经过 Zod schema 校验，`fromFallback` 标记记录了是否因为解析失败走了降级。查询 `LlmLog` 里 `fromFallback=true` 的比例可以衡量不同模型的结构化输出稳定性。这个指标简单客观，是选型的一票否决项——如果某个模型的 `fromFallback` 率超过 5%，直接排除。

**维度二：路由准确率**（人工评估）

抽取 50-100 条典型用户输入，对比模型选出的 `routeId` 是否符合预期。这一步是人工打分的，没有自动化。

当前选 deepseek-v3.2 的主要原因是在这两个维度上表现稳定，且成本显著低于 GPT-4 系列（规划是高频调用，成本权重高）。

---

### 两模型大多数接近、少数差异显著时如何决策

这种情况出现过，做法是**按差异样本的类型决策**：

- 如果差异集中在**边界 case**（极端输入、多义表达）→ 倾向于选在这些 case 上表现更好的模型，因为正常 case 两者差不多，边界 case 决定系统的"下限"
- 如果差异集中在**高频 case**（常见用户请求）→ 高频 case 的准确率直接影响绝大多数用户体验，即使低频 case 表现差一些也可以接受
- 如果差异分布无规律 → 看成本和延迟的差异，相近质量下选更便宜或更快的

坦诚说：这套决策过程是经验判断，没有统计显著性检验，样本量（50-100 条）也不足以做严格的 A/B 对比。如果要做更严格的选型，应该收集生产流量做在线评估，但目前规模不支持这么做。

---

### 追问

**Q：Schema 合规率是一票否决项，但合规不等于路由正确——LLM 可能输出了合法 JSON 但选了错误的 routeId。这个问题怎么处理？**

是的，Schema 合规只是必要条件不是充分条件。合规率用于筛掉"根本不能稳定输出结构化结果"的模型，路由准确率才是真正的质量指标。两个指标分层使用：合规率不达标直接淘汰，合规率都达标的再比路由准确率。路由准确率目前是人工标注，50 条样本里按 routeId 是否符合预期打分，准确率差距超过 10 个百分点才会影响选型决定。

**Q：有没有考虑过 LLM as Judge，让另一个 LLM 来评估规划结果的合理性？**

考虑过，没有实现。LLM as Judge 在这个场景下的问题是：评判标准是"routeId 是否符合用户意图"，这个标准本身就需要 Judge LLM 理解业务语义（每个 capability 的适用场景），相当于 Judge LLM 要重新学习一遍能力注册表。不如直接用人工标注，成本差不多，结论更可靠。LLM as Judge 更适合开放式生成任务（比如评估回答的质量），不太适合这种有明确正误的选择题。

---

## Q17. Prompt 改动后的回归保障

**问题：** 当你修改了 Planner 的 Prompt 或调整了某个 Executor 的 Schema，你如何保证这次改动不会破坏已有的 LLM 链路？你有没有维护一套"黄金用例集（Golden Set）"？如果有，这批用例是如何录入的、覆盖率如何衡量、多久更新一次？如果 Prompt 改动后黄金用例的通过率从 95% 降到 88%，你的发布决策标准是什么？

---

### 当前的回归保障方式

**没有正式的 Golden Set。** 当前的回归保障分两层：

**第一层：单元测试（mock LLM）**

测试文件里对每个关键组件写了 mock 测试，LLM 调用返回预设的 JSON fixture：

```typescript
// route-planner.service.spec.ts
it('calls LLM and returns parsed plan for complex request', async () => {
  mockLlm.chat.mockResolvedValue(JSON.stringify({
    steps: [
      { routeId: 'provide_link' },
      { routeId: 'generate_qrcode', inheritFrom: 'provide_link' },
    ],
    confidence: 0.95,
  }));

  const plan = await service.plan('给我领券页面的微信小程序二维码', []);
  expect(plan.steps).toHaveLength(2);
  expect(plan.steps[0].routeId).toBe('provide_link');
});
```

这层测试验证的是"当 LLM 返回预设结果时，代码逻辑是否正确"，**不验证 LLM 在真实调用时是否返回预期结果**。Prompt 文本改了，mock 结果不变，测试依然通过——这是 mock 测试天然的盲区。

**第二层：人工冒烟测试**

改完 Prompt 后，手动跑 10-20 个典型 case，看输出是否符合预期。这是主要的回归手段，但覆盖率和一致性完全依赖个人经验，没有量化指标。

---

### Golden Set 如果要建，录入方式是什么

目前最贴近 Golden Set 的是 `LlmLog` 里的历史记录——可以从生产日志里选取典型的"正确案例"作为 baseline：

1. 从 `LlmLog` 里按 `promptId` 筛出路由规划的历史调用
2. 人工标注哪些调用的输出是"正确的"（routeId 选对、confidence 合理）
3. 把这批 `(promptMessages, expectedOutput)` 对存成测试用例文件
4. Prompt 改动后，用 `/replay` 接口批量重放，对比输出

**没有实现的原因**：这套流程需要额外的工具支撑（用例管理、批量 replay、自动对比），开发成本约 2-3 天，目前迭代速度不快（Prompt 改动不频繁），人工冒烟足够覆盖。

---

### 通过率从 95% 降到 88%，发布决策标准

这个假设场景没有在真实项目里出现过（因为没有 Golden Set），但判断框架是：

**看 7% 的失败集中在哪类 case：**

- 失败都集中在**边界 case**（奇怪输入、多义表达）→ 可以接受，这类 case 在生产里占比低
- 失败分布在**核心高频 case**（常见的"帮我找链接"）→ 不能发布，7% 的失败意味着大量用户受影响
- 失败是因为 Prompt 改动引入了新能力、旧 case 的预期本身过时了 → 先更新 Golden Set 的预期，再重新评估通过率

**没有固定的数字门槛**，因为 88% 在什么场景下可接受完全取决于失败分布。一刀切的"通过率 ≥ 90% 才发布"是懒惰的，应该看失败在哪里、影响哪些用户、可不可以快速 hotfix。

---

### 追问

**Q：单元测试里 mock 了 LLM 返回，意味着 Prompt 文本改了测试也照常通过。有没有办法在 CI 里检测 Prompt 有没有被修改？**

有，但没做。最简单的方式：把每个 Prompt seed 的文本内容哈希写进 CI 的 snapshot 文件，每次 CI 跑时重新计算哈希对比，不一致就告警"Prompt 已修改，请手动验证"。这相当于强制 Prompt 改动不被静默合并，必须经过人工确认。不需要自动跑真实 LLM，只是一个"变更感知"机制。

**Q：如果 Prompt 改动是为了修复一个 bug，但同时导致其他 case 通过率下降，这是典型的权衡场景，你怎么处理？**

先量化两边的影响：bug 修复影响了多少用户（从 `LlmLog` 里查这个 bug pattern 的出现频率），通过率下降影响了多少用户（从 Golden Set 的失败分布里估算）。影响用户数多的一边赢。如果无法量化，优先修复 bug（因为有明确的用户反馈），同时把因为 Prompt 改动而失败的 case 记录下来作为下一次迭代的优化目标，而不是阻塞这次发布。

---

## Q18. LLM 输出不确定性与测试策略

**问题：** LLM 输出具有概率性，同一个 Prompt 多次调用结果不完全相同。你在单元测试和集成测试中如何处理这个不确定性？是固定 `temperature=0` 强制确定性、还是多次采样后投票、还是对输出做语义等价判断而非精确字符串匹配？`temperature=0` 真的能保证可重复性吗——你踩过模型版本升级导致 `temperature=0` 输出漂移的坑吗？

---

### 单元测试：完全 mock，不触碰真实 LLM

单元测试里 LLM 调用被完全 mock，`jest.fn()` 返回预设字符串：

```typescript
// structured-llm-caller.spec.ts
it('returns parsed data on first attempt when LLM returns valid JSON', async () => {
  const mockLlm = {
    chat: jest.fn().mockResolvedValue('{"intent":"get_link","confidence":0.9}'),
  };
  const caller = new StructuredLlmCaller(mockLlm, TestSchema, fallback, { promptId: 'test' });
  const result = await caller.call({});
  expect(result.data).toEqual({ intent: 'get_link', confidence: 0.9 });
});
```

测试验证的是**代码逻辑**（三层解析、重试机制、fallback 触发），不验证 LLM 行为。这层测试和 temperature 无关，输出是完全确定的。

---

### 集成测试与生产环境：用低 temperature 降低方差

生产 Prompt seed 里的温度配置：

```typescript
// marketing-dialogue.seeds.ts（规划和槽位提取类 Prompt）
modelConfigJson: JSON.stringify({
  model: 'deepseek-v3.2',
  temperature: 0.1,  // 接近 0 但不是 0
  max_tokens: 700,
}),
```

用 `0.1` 而不是 `0` 的原因：完全确定性（temperature=0）在某些 LLM API 实现里会让模型过于保守，对歧义输入总是选同一个次优答案；`0.1` 保留了少量随机性，在歧义输入时有小概率输出不同结果，但大多数情况下稳定。

没有实现"多次采样投票"——代价是每次请求 N 倍的 token 消耗和延迟，对路由规划这种每个用户消息都要触发的高频操作不可接受。

---

### temperature=0 能否保证可重复性

**不能完全保证。**

`temperature=0` 在理论上让 softmax 退化为 argmax，每次选最高概率 token，应该是确定性的。但实际上有几个干扰因素：

**GPU 浮点运算的非确定性**：分布式推理（多 GPU 并行）下，浮点加法顺序不同导致微小精度差异，极少情况下会影响 argmax 的结果，尤其是两个 token 概率非常接近时。

**模型版本静默更新**：LLM API 服务商会在不改变模型名的情况下更新权重（bug 修复、安全对齐调整），更新后 `temperature=0` 的输出可能和之前不同。

**没有踩过这个坑**，因为测试层完全 mock LLM，没有依赖真实 LLM 输出做精确匹配的自动化测试。如果存在这种测试，模型版本更新后会出现"测试通过但生产行为变了"的隐患。不用 `temperature=0` 做精确匹配测试，反而避免了这个问题——因为从一开始就没有把 LLM 的特定输出当成测试的基准。

---

### 如果要做语义等价判断，改造方向

对路由规划结果做语义等价判断，需要定义"等价"：
- `steps[*].routeId` 序列相同 → 等价（不管 `reason` 文字是否不同）
- 关键字段（`confidence` 高低、是否有 `clarificationNeeded`）对应 → 等价

这比精确字符串匹配宽松，可以容忍 `reason` 字段的表述变化。实现上就是在 replay 测试里不做 `toEqual(expectedOutput)`，而是做字段级提取后的结构对比。目前没有实现，因为没有 Golden Set 测试用例，语义等价判断没有用武之地。

---

### 追问

**Q：完全 mock LLM 的单元测试只验证了代码逻辑，如果 Prompt 模板里有语法错误（比如变量名拼错），测试能发现吗？**

不能，这是 mock 测试的根本盲区。Prompt 模板的语法（变量替换、格式）要到真实调用时才能验证，mock 返回的是预设字符串，不会触发模板渲染错误。目前的保障是：Prompt 模板在 `marketing-dialogue.seeds.ts` 里是明文字符串，变量名通过 `{variableName}` 插值替换，如果变量名拼错，LLM 会收到 `{varName}` 这个字面量而不是预期值——这会在人工冒烟测试时暴露（LLM 输出明显异常），不会被单元测试捕获。

**Q：你说用 `temperature=0.1` 而不是 `0`，但如果两次调用之间用户输入完全相同，`0.1` 的温度会导致不同的路由决策吗？**

理论上会，但概率极低。`temperature=0.1` 下，只有当两个候选 token 的原始概率非常接近时，少量温度才会翻转选择。Planner 的 Prompt 约束很强（能力描述明确、用户输入大多有明确意图），在这种情况下"最优 routeId"的概率通常远高于其他选项，`0.1` 的温度不足以改变结果。边界 case（用户输入高度歧义，多个 routeId 概率接近）才会出现不稳定，而这类 case 本来就应该触发 `clarificationNeeded`（置信度不足追问用户），不依赖 Planner 做出唯一确定的选择。
