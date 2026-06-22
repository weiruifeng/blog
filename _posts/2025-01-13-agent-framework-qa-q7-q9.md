---
layout: post
title: "Agent 框架面试问答（三）：LLM 稳定性与容错"
date: 2025-01-13
categories: 面试
tags: [AI, Agent, LLM, TypeScript, 熔断器]
---

> 基于简历「自主设计 Agent 框架与 AI 系统落地」项目推导，考察 LLM 结构化输出稳定性、自纠错机制与熔断器设计相关问题。

---

## Q7. 三层 JSON 提取兜底

**问题：** 三层 JSON 提取兜底中，"正则提取"作为最后一层兜底，适用于什么场景？LLM 输出的 JSON 在哪些情况下会让前两层都失败？你的正则方案覆盖了哪些异常格式，准确率大概是多少？

---

### 三层策略的代码实现

三层逻辑集中在 `StructuredLlmCaller.tryParse()` 里：

```typescript
private tryParse(raw: string): ParseResult<T> {
  // 第一层：直接 JSON.parse
  try {
    const obj = JSON.parse(raw);
    const result = this.schema.safeParse(obj);
    if (result.success) return { success: true, data: result.data };
    return { success: false, error: `Schema validation failed: ${result.error.message}` };
  } catch {}

  // 第二层：从 Markdown 代码块提取
  const codeBlockMatch = raw.match(/```(?:json)?\s*([\s\S]*?)```/);
  if (codeBlockMatch) {
    try {
      const obj = JSON.parse(codeBlockMatch[1].trim());
      const result = this.schema.safeParse(obj);
      if (result.success) return { success: true, data: result.data };
      return { success: false, error: `Schema validation failed after code block extraction: ${result.error.message}` };
    } catch {}
  }

  // 第三层：正则提取 {...} 或 [...]
  const jsonMatch = raw.match(/(\{[\s\S]*\}|\[[\s\S]*\])/);
  if (jsonMatch) {
    try {
      const obj = JSON.parse(jsonMatch[1]);
      const result = this.schema.safeParse(obj);
      if (result.success) return { success: true, data: result.data };
      return { success: false, error: `Schema validation failed after bracket extraction: ${result.error.message}` };
    } catch {}
  }

  return { success: false, error: `Cannot parse JSON from output: ${raw.substring(0, 100)}` };
}
```

---

### 前两层失败的典型场景

**第一层（直接 `JSON.parse`）失败**：输出有任何非 JSON 前缀或后缀就会抛异常。

| 输出形态 | 第一层结果 |
|---------|-----------|
| `{"steps": [...]}` | ✅ 成功 |
| `这是执行计划：\n{"steps": [...]}` | ❌ 失败（前缀文字） |
| `{"steps": [...]}\n以上是计划。` | ❌ 失败（后缀文字） |
| `{"steps": [...], // 注释}` | ❌ 失败（含注释） |

**第二层（代码块提取）失败**：第一层失败且输出里没有三个反引号包裹的块时跳过。

两层同时失败的最常见场景：**LLM 输出了完整 JSON 但包裹在自然语言里，且没有使用代码块格式**，比如：

```
好的，根据您的请求，执行计划如下：{"steps": [{"routeId": "provide_link"}], "confidence": 0.9}
请确认是否继续。
```

---

### 正则层覆盖的格式和局限

正则是 `/(\{[\s\S]*\}|\[[\s\S]*\])/`，贪婪匹配从**第一个 `{` 到最后一个 `}`**。

**能覆盖的格式：**
- JSON 前后有任意自然语言
- JSON 内部有换行（`[\s\S]*` 匹配含换行的任意字符）
- JSON 嵌套任意深度（只要整体结构完整）

**不能覆盖的格式（正则提取出的字符串 `JSON.parse` 会失败）：**

如果输出里在 JSON 之前出现了孤立的 `}`，正则会从第一个 `{` 匹配到最后一个 `}`，导致截取范围超出 JSON 边界：

```
完成了{这个}任务：{"steps": [...]}  →  正则匹配 {这个}任务：{"steps": [...]}  →  parse 失败
```

---

### 准确率

没有生产环境的精确统计，从使用情况来看：

- **第一层覆盖率最高**：规划 Prompt 的 System Prompt 明确要求 LLM 只输出 JSON，大多数情况下第一层直接命中
- **第二层是最常见的兜底**：部分模型会习惯性地用代码块包 JSON，第二层能稳定捕获
- **第三层触发频率最低**：temperature 较低（`0.1`）且 Prompt 明确约束输出格式时，走到第三层的比例极小；走到第三层的情况多数能 parse 成功，因为 LLM 输出的自然语言里带花括号的词汇很少

---

### 追问

**Q：三层都失败后返回 `fromFallback: true` 和一个空计划，这个信号在业务层怎么处理，用户看到什么？**

`fromFallback: true` 会被 Dialogue Manager 识别。规划阶段的 fallback 是 `{ steps: [], confidence: 0 }`，`confidence: 0` 进入澄清追问流程，用户看到"我没有理解您的需求，能换个方式描述吗"之类的兜底文案。不会有 JSON 解析失败的技术错误暴露给用户，是 fail-safe 而非 fail-silent。

**Q：第三层正则是贪婪匹配，如果 LLM 输出里有两个 JSON 对象，会截取到哪个？**

会截取从第一个 `{` 到最后一个 `}` 的整段，把两个 JSON 和中间的文字全包进去，然后 `JSON.parse` 失败，第三层报 parse 错误，最终走 fallback。对多 JSON 场景处理不正确，但规划 Prompt 的约束很强，LLM 输出两个独立 JSON 对象的情况在实践中没有遇到过。

**Q：Schema 校验失败（JSON 结构合法但字段不符合 schema）和 JSON 格式错误是两种不同的失败，你区分处理了吗？**

没有区分——两种失败都走同一套"记录 `lastError`，注入重试 Prompt"的路径。区分有意义的地方在于：JSON 格式错误适合提示"请只输出 JSON"，schema 校验失败适合提示"字段 X 不正确"。目前两者都走同一个模板变量 `_retryError`，提示内容的精确度依赖 Zod 错误信息的质量。从实际效果看够用，没有因为不区分而出现重试策略明显错误的情况。

---

## Q8. Schema 验证错误注入自纠错机制

**问题：** "将 Schema 验证错误详情注入重试 Prompt 的自纠错机制"——这个错误详情是原始 Zod/JSON Schema 的错误信息，还是你经过处理的自然语言描述？注入错误信息是否曾经导致 LLM 在重试时陷入新的错误循环？

---

### 注入的是原始 Zod 错误，未做转换

错误信息直接取自 Zod 的 `result.error.message`，重试时作为模板变量注入：

```typescript
const callVars = attempt === 0
  ? { ...vars, _retryError: '', _retryAttempt: '0' }
  : { ...vars, _retryError: lastError, _retryAttempt: String(attempt) };
```

LLM 在重试时看到的错误形如：

```
Schema validation failed: Required at "steps"; Expected array, received undefined
```

或者 JSON 格式错误时看到：

```
Cannot parse JSON from output: 好的，执行计划如下...
```

没有任何自然语言转换——Zod 的错误格式是面向开发者的技术描述，直接暴露给了 LLM。

---

### 为什么没有做转换

一个务实的原因：**这批规划任务的 Schema 非常简单**，Zod 错误信息里出现频率最高的是字段缺失（`Required at "steps"`）和类型不匹配（`Expected number, received string`），这两种错误即使是技术格式，LLM 也能读懂并在下一次输出时修正。

如果 Schema 里有复杂的 union、enum、refine 条件，Zod 错误会变得更难理解（比如 `Invalid union`），这时候原始 Zod 错误的效果就会变差，需要转换成"字段 X 的值必须是 Y 或 Z"这类描述。目前没有遇到这种情况，所以没有做转换。

---

### 有没有出现过错误循环

**没有出现过。** 有两个结构性原因：

**原因一：重试次数上限很低**

路由规划阶段显式设置了 `maxRetries: 1`：

```typescript
new StructuredLlmCaller(..., { promptId: NLU_PROMPT_IDS.ROUTE_PLANNING, maxRetries: 1 })
```

`maxRetries: 1` 意味着循环只跑一次，**根本没有第二次重试**——第一次 LLM 调用失败后，注入错误做第二次调用，如果第二次也失败，直接走 fallback，不存在多轮循环的空间。

**原因二：规划 Prompt 的格式约束非常强**

规划 Prompt 的 System Prompt 明确告诉 LLM 只输出 JSON，配合 `temperature: 0.1` 的低随机性，重试时 LLM 通常能修正格式问题。

---

### 追问

**Q：`maxRetries: 1` 只允许一次重试，如果提高到 3，理论上可能出现什么问题？**

有两个风险。第一：如果 LLM 每次都输出格式正确但字段值错误的 JSON（比如 `confidence` 总是字符串而非数字），Zod 每次都报相同的类型错误，重试只是把同一个错误反复注入，LLM 大概率仍然输出同样的格式，3 次都失败。第二：如果注入的错误信息干扰了 LLM 对正确格式的判断，LLM 可能在正确格式和错误格式之间摇摆，每次输出不同的错误。目前 `maxRetries: 1` 是合理的上限。

**Q：错误信息里含有字段路径（如 `"steps[0].routeId"`），LLM 是否真的能根据这个定位到问题并修正？**

没有系统性验证，只有经验性的观察。从日志里看，第一次调用输出了 `confidence` 是字符串类型（比如 `"0.9"`）的情况下，注入 `Expected number, received string at "confidence"` 后，第二次输出的 `confidence` 改成了数字——说明 LLM 能理解这类简单的类型错误。但如果错误是 `Invalid union` 这种抽象错误，LLM 大概率看不懂，重试成功率就低。这块没有统计数据，是推断。

---

## Q9. 熔断器并发安全

**问题：** 熔断采用"惰性状态转移（由下次请求触发）"——在极高并发下，OPEN 状态熔断后大量请求并发到达，如何防止它们同时触发状态转移到 HALF-OPEN 并全部落到备用模型？你有没有实现"只允许单请求探测"的机制？

---

### 惰性转移的代码实现

`isAvailable()` 是唯一的状态转移入口：

```typescript
// llm-circuit-breaker.service.ts
isAvailable(provider: string): boolean {
  const circuit = this.circuits.get(provider);
  if (!circuit) return true;
  if (circuit.state === ProviderState.CLOSED) return true;
  if (circuit.state === ProviderState.HALF_OPEN) return true;  // HALF_OPEN 直接放行
  // OPEN：检查冷却时间
  if (circuit.openAt && Date.now() - circuit.openAt >= this.cooldownMs) {
    circuit.state = ProviderState.HALF_OPEN;  // 惰性转移
    return true;
  }
  return false;
}
```

---

### 并发场景下的实际行为

**Node.js 单线程保证了 OPEN→HALF_OPEN 转移本身的原子性。** `isAvailable()` 是同步函数，一个请求的调用执行完之前，不会有其他 JS 代码插入运行。所以"多个请求同时把 state 从 OPEN 改成 HALF_OPEN"这个竞态条件不会发生——第一个请求改完后，后续请求进来看到的已经是 `HALF_OPEN`。

**但 HALF_OPEN 状态下没有"单请求探测"机制。** 当 state 是 `HALF_OPEN` 时，`isAvailable()` 直接返回 `true`，不计数、不加锁、不区分是否是探测请求。所以：

```
Request A: isAvailable() → OPEN，冷却到期 → 设 HALF_OPEN，返回 true → 发起 LLM 调用
Request B: isAvailable() → HALF_OPEN → 返回 true → 也发起 LLM 调用
Request C: isAvailable() → HALF_OPEN → 返回 true → 也发起 LLM 调用
```

A、B、C 全部成为"探测请求"，同时打到了原本熔断的 provider。

---

### 这个设计缺口在实际场景下的影响

**影响比较有限，原因是系统规模：**

当前只有两个 provider（`volces` 和 `deepseek`）。某个 provider 触发熔断时，所有请求会落到备用 provider。**并发打到熔断 provider 的场景只在 HALF_OPEN 恢复期间发生**，而营销助手的实际并发量不高（单用户对话场景），同时处于 HALF_OPEN 并发起探测的请求数量通常是个位数。

多探测请求的结果是幂等的：
- 探测请求都失败 → `recordFailure()` 把 state 改回 OPEN，重复调用幂等
- 探测请求都成功 → `recordSuccess()` 把 state 改成 CLOSED，重复调用幂等
- 一部分成功、一部分失败 → 可能出现"恢复了又熔断"的抖动，但不会出现状态损坏

---

### 如果要实现单请求探测，改动点在哪

改动很小，在 `isAvailable()` 里加一个标记位：

```typescript
// 修改后的设计（未实装）
if (circuit.openAt && Date.now() - circuit.openAt >= this.cooldownMs) {
  if (circuit.state !== ProviderState.HALF_OPEN) {
    circuit.state = ProviderState.HALF_OPEN;
    circuit.probeGranted = true;  // 探测权只给第一个请求
    return true;
  }
  // 已经有探测请求了，后续请求拒绝，让它们走备用 provider
  return false;
}
```

`probeGranted` 在 `recordSuccess()` 或 `recordFailure()` 时清除，下一个冷却周期重新竞争探测权。这个改动大约 10 行代码，没有实装的理由是当前并发量下影响不显著，且多探测对恢复速度反而更友好（多个成功探测能更快确认 provider 恢复）。

---

### 追问

**Q：熔断的触发条件是什么——单次失败就熔断，还是有失败率阈值？**

当前是**单次失败直接 OPEN**，没有失败率窗口。`recordFailure()` 被调用一次就把 state 设成 OPEN。

这是有意的强保守策略：LLM 服务的失败通常是批量性的（模型服务宕机），不是随机偶发的，单次失败就可以认为当前 provider 不可用，立即切备用比等到失败率达到阈值更快。代价是对偶发超时过于敏感，一次网络抖动就会触发 5 分钟冷却。

**Q：冷却时间 5 分钟（`cooldownMs = 300_000`）是怎么确定的？**

凭经验拍的，没有精确测量过 LLM 服务故障后的平均恢复时间。5 分钟的来源是：主流 LLM API 服务的故障通常是过载导致的，过载恢复一般在 1-5 分钟内，5 分钟作为保守上界。这个参数在 service 里是 `public` 的可以在测试里覆盖，但没有做成可运行时配置的参数。

**Q：只有两个 provider，如果两个都熔断了，`getAvailableProviders()` 返回空数组，请求怎么处理？**

`withProviderFallback()` 里 `for` 循环没有可用 provider 可以遍历，直接到循环结束，会抛出一个"所有 provider 不可用"的错误，前端收到 5xx，用户看到系统错误提示。当前没有"双熔断兜底"的处理——两个 provider 同时熔断在实践中还没有发生过，这是一个已知的可靠性下限。
