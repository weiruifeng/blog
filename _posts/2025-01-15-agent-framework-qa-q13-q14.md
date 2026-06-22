---
layout: post
title: "Agent 框架面试问答（五）：SDK 设计与多轮会话管理"
date: 2025-01-15
categories: 面试
tags: [AI, Agent, SDK, TypeScript, React]
---

> 基于简历「自主设计 Agent 框架与 AI 系统落地」项目推导，考察零 UI 框架依赖的 SDK 生命周期管理与多轮会话截断策略。

---

## Q13. 零 UI 框架依赖的 SDK 生命周期管理

**问题：** 对话 SDK 设计为"零 UI 框架依赖"，它如何管理 SSE 连接的生命周期？业务组件卸载时，如果 SSE 连接还在推流，SDK 如何优雅清理而不造成内存泄漏？

---

### 为什么"零 UI 框架依赖"

SDK 的核心类 `MarketingDialogueClient` 是一个普通 TypeScript 类，不继承任何框架基类，不使用 React hooks、Vue reactivity 或任何框架的生命周期 API：

```typescript
// marketing-dialogue-client.ts
export class MarketingDialogueClient {
  private abortController: AbortController | null = null;
  private baseUrl: string;

  constructor(baseUrl: string = '') {
    this.baseUrl = baseUrl || (typeof window !== 'undefined' ? window.location.origin : '');
  }

  abort(): void {
    if (this.abortController) {
      this.abortController.abort();
      this.abortController = null;
    }
  }

  async streamMessage(req, cb): Promise<SSEResult> { ... }
  async resumeStream(sessionId, userId, cb): Promise<SSEResult> { ... }
  async submitInteraction(sessionId, userId, values): Promise<...> { ... }
}
```

"零依赖"的含义是：这个类可以在任何 JS 环境里实例化——React 组件、Vue setup、Node.js 脚本、甚至浏览器控制台——不需要任何框架上下文。

---

### SSE 连接生命周期的核心机制

用 `fetch` + `AbortController` 替代 `EventSource`，原因是 `EventSource` 不支持自定义 Header 且自动重连不可控：

```typescript
async streamMessage(req, cb): Promise<SSEResult> {
  this.abortController = new AbortController();  // 每次请求创建新实例

  const response = await fetch(url, {
    method: 'GET',
    headers: { Accept: 'text/event-stream', 'Cache-Control': 'no-cache' },
    signal: this.abortController.signal,  // 绑定中止信号
  });

  try {
    return await parseSSEStreamV2(response, cb);
  } catch (err) {
    if (err?.name === 'AbortError') throw err;
    throw new Error(`网络请求失败：${err?.message}`);
  } finally {
    this.abortController = null;  // 无论成功/失败/中止，清除引用
  }
}
```

`finally` 块保证了 `abortController` 引用在请求结束后一定被置 null，不会因为 `streamMessage` 返回后忘记清理而持有 `AbortController` 的强引用。

---

### 组件卸载时的清理路径

业务组件（React hook `use-chat.ts`）持有 `clientRef`，在卸载时调用 `abort()`：

```typescript
// use-chat.ts
const clientRef = useRef(new MarketingDialogueClient(baseUrl));

const abort = useCallback(() => {
  clientRef.current?.abort();  // 触发 fetch signal

  // 更新 UI 状态（组件还没卸载时执行）
  if (currentBotMessageIdRef.current) {
    updateMessage(currentBotMessageIdRef.current, {
      content: { phase: 'error', errorMessage: '已取消' },
      isStreaming: false,
    });
  }
  setIsLoading(false);
}, [updateMessage, setIsLoading]);

// 组件卸载时调用
useEffect(() => {
  return () => { abort(); };
}, [abort]);
```

清理链路：
1. `abort()` → `abortController.abort()` → `fetch` signal 触发
2. `parseSSEStreamV2` 的 `reader.read()` 抛 `AbortError`
3. `streamMessage` 的 catch 捕获 `AbortError`，重新抛出
4. `finally` 块执行，`this.abortController = null`
5. `use-chat` 的 Promise 收到 rejection，链路结束
6. 组件已卸载，后续没有 state 更新调用

---

### 内存泄漏防护点

| 泄漏风险 | 防护手段 |
|---------|---------|
| AbortController 未释放 | `finally` 置 null，脱离强引用 |
| ReadableStream reader 未关闭 | `AbortError` 时 reader 随 fetch 连接一起关闭 |
| 回调引用持续存活 | 回调在闭包内，随 `parseSSEStreamV2` 调用栈结束而 GC |
| 流数据全量缓存 | `reader.read()` 增量处理，buffer 仅保留当前未解析行 |

---

### 追问

**Q：`use-chat.ts` 里 `clientRef` 用的是 `useRef`，而不是 `useMemo` 或 `useState`，为什么？**

`useRef` 的引用不触发重渲染，适合持有"不影响 UI 呈现但需要贯穿组件生命周期的对象"。`MarketingDialogueClient` 实例在组件整个生命周期里只需要一个，不需要随 props/state 变化重建，`useRef` 是最合适的容器。`useMemo` 依赖 deps 数组，有被误清除的风险；`useState` 每次 setter 调用都触发重渲染，不合适。

**Q：如果同时发起了两个 `streamMessage`（比如用户快速连发两条消息），两次请求的 `AbortController` 会互相覆盖吗？**

会。`this.abortController` 是实例属性，第二次调用 `streamMessage` 会创建新的 `AbortController` 覆盖旧的。调用 `abort()` 时只会中止最后一次创建的控制器，第一次请求的 `fetch` 不受影响，会继续推流。实际上 `use-chat.ts` 里通过 `isLoading` 状态锁避免了并发发消息，确保前一条请求完成前无法发送下一条。`MarketingDialogueClient` 本身不做并发控制，依赖上层约束。

---

## Q14. 多轮会话管理与上下文截断

**问题：** SDK 中的"多轮会话管理"是如何实现的？会话历史存储在客户端还是服务端？当会话历史非常长时（超过 LLM 上下文窗口），你的截断策略是什么，如何保证截断后 Agent 对早期步骤产出的引用仍然正确？

---

### 会话历史的存储位置

分三层，职责不同：

| 存储层 | 内容 | 持久性 | 用途 |
|--------|------|--------|------|
| 客户端 Zustand store | 当前会话消息（含气泡类型、流状态） | 仅当前页面 | UI 渲染 |
| 服务端 MySQL | 完整对话消息（role + content） | 永久 | 历史查询、多设备同步 |
| 服务端 Redis | 对话上下文（state、slots、intent、persistentEntities） | 30 天 | 当前任务状态 |

**客户端只是 UI 镜像**，不是真正的"存储"——刷新页面后 Zustand store 清空，历史从 MySQL 重新加载。LLM 推理用的对话历史从 MySQL 取，不从客户端取。

---

### 截断策略

每次处理用户消息时，服务端加载最近 N 轮对话作为 LLM 的上下文窗口：

```typescript
// chat-history.service.ts
async loadConversationMessages(
  sessionId: string,
  maxTurns = 10,
): Promise<Array<{ role: 'user' | 'assistant'; content: string }>> {
  const messages = await this.messageModel.findAll({
    where: { sessionId },
    order: [['createdAt', 'ASC'], ['id', 'ASC']],
  });

  return messages
    .slice(-maxTurns * 2)  // 取最后 maxTurns 轮，即最多 20 条消息
    .map((msg) => ({
      role: msg.msgRole as 'user' | 'assistant',
      content: this.extractTextFromContent(msg.msgContent),
    }));
}
```

默认 `maxTurns = 10`，在 agent 内存策略里被显式配置：

```typescript
// agent-memory-policies.ts
export const AGENT_MEMORY_POLICY_OVERRIDES = [
  {
    appkey: 'com.maoyan.movie.fe.aitools',
    promptId: 'MarketingAssistant',
    policy: { mode: 'fixed', config: { enableMemory: true, maxTurns: 10 } },
  },
];
```

`maxTurns` 在代码里做了硬性钳制，上限 50 轮（100 条消息），防止配置失误传入超大值：

```typescript
const turns = Math.max(1, Math.min(Number(maxTurns) || 10, 50));
```

截断策略是**保留最新 N 轮，丢弃最早的消息**——最简单的"滑动窗口"，没有基于 token 计数的动态截断，也没有"重要消息保留"逻辑。

---

### 截断后步骤产出的引用如何保持正确

这是设计里最关键的分离：**步骤产出（`inheritedData`）不存在对话历史里，而是存在 Redis 的 `persistentEntities` 里。**

任务完成时，执行层把关键产出持久化到 Redis context：

```typescript
// context-manager.service.ts
async completedTask(sessionId: string, inheritedData: Record<string, any>): Promise<void> {
  const context = await this.getContext(sessionId);

  if (!context.persistentEntities) context.persistentEntities = {};
  const entityKeys = ['link', 'linkType', 'linkName'];
  for (const key of entityKeys) {
    if (inheritedData[key]) {
      context.persistentEntities[key] = {
        value: String(inheritedData[key]),
        source: context.currentTask?.plan?.steps?.[0]?.routeId || 'unknown',
        timestamp: Date.now(),
      };
    }
  }

  delete context.currentTask;
  await this.saveContext(context);
}
```

后续对话里用户说"再生成一个二维码"（`continue` 意图），系统从 `persistentEntities` 恢复上游数据，不从对话历史里提取：

```typescript
// dialogue-manager.service.ts
const inheritedData = intentType === 'continue'
  ? this.extractLastTaskContext(context)  // 从 Redis 取，不从历史消息取
  : {};

private extractLastTaskContext(context: any): Record<string, any> {
  const entities = context?.persistentEntities ?? {};
  const result: Record<string, any> = {};
  for (const key of ['link', 'linkType', 'linkName']) {
    if (entities[key]?.value) result[key] = entities[key].value;
  }
  return result;
}
```

**结论：对话历史截断和步骤产出引用是两个独立的维度。** 历史消息多少条、截不截断，不影响 Agent 能否正确引用早期步骤的产出——两者存在不同的存储里，生命周期独立。

---

### `persistentEntities` 的边界

`persistentEntities` 目前只保存三个 key：`link`、`linkType`、`linkName`。这是有意的最小化设计——只保存"跨任务引用最频繁"的实体，不把所有步骤产出都往里塞。

如果业务扩展后需要引用更多上游产出（比如"上次生成的短链接"、"上次选的目标平台"），需要在 `completedTask` 里手动扩展 `entityKeys` 数组。目前没有自动发现机制——不是所有 `produces` 里的 key 都会被持久化，只有显式声明在 `entityKeys` 里的才会。

---

### 追问

**Q：截断策略是"保留最新 N 轮"，如果第 1 轮的用户消息里包含了重要的约束条件（比如"所有链接都要用微信小程序格式"），截断后 LLM 还能记住这个约束吗？**

不能。一旦这条消息被滑出窗口，LLM 在后续推理时就看不到它了。目前的应对方式有两个：第一，Planner 每次推理时会重新读取用户的当前消息，用户如果在新消息里重申约束，Planner 能感知；第二，关键约束如果转化成了槽位值（比如 `platform: 'wxapp'`），会通过 `slots` 字段保存在 Redis context 里，不依赖对话历史。但如果约束只是自然语言说过一次、没有被提取成结构化槽位，就会随消息历史一起丢失。这是当前截断策略的已知缺陷，更精细的做法是在截断时识别"约束类消息"并单独保留，目前没实现。

**Q：MySQL 存的是完整消息历史，Redis 存的是最近状态，两者可能不一致吗？**

可能。Redis context 里的 `slots`、`intent` 是最近一次请求的状态快照，可能和 MySQL 历史消息里隐含的语义不完全对应。比如用户说了 10 轮之后改变了意图，`intent` 已经更新，但 `slots` 里可能还有 5 轮前被提取的旧槽位值残留。这种不一致在实践中通常不影响结果，因为每次请求都会重新提取当前消息里的槽位并合并，新值会覆盖旧值。如果出现异常，traceId 可以把 LLM 调用日志和 Redis 状态对应起来排查。

**Q：`extractTextFromContent` 把消息内容提取成纯文本注入 Prompt，如果消息内容是结构化数据（比如二维码卡片），LLM 看到的是什么？**

看到的是 `extractTextFromContent` 处理后的字符串表示。结构化消息（表单卡片、二维码卡片）存入 MySQL 时是 JSON 格式，`extractTextFromContent` 会把它提取成可读的文本摘要（比如"[二维码]已生成链接的二维码"），而不是把完整 JSON 塞进 Prompt。这样做一方面控制了 Prompt 长度，另一方面避免了 LLM 看到大量 UI 状态字段而产生困惑。具体的提取规则按消息类型分 case 处理，文本消息直接取 `content`，结构化消息取 `displayContent` 字段。
