---
layout: post
title: "Agent 框架面试问答（四）：SSE 流式协议与可观测"
date: 2025-01-14
categories: 面试
tags: [AI, Agent, SSE, TypeScript, 可观测]
---

> 基于简历「自主设计 Agent 框架与 AI 系统落地」项目推导，考察 SSE 事件对称性、断线重连与日志积压防控相关问题。

---

## Q10. SSE 事件对称性与 finally 语义保证

**问题：** 你设计了 8 种对称 SSE 事件，"通过 finally 语义保证开始/结束事件成对出现"——如果服务端进程在 LLM 流式响应过程中崩溃（比如 OOM），finally 语义还能保证吗？你的前端状态机如何处理"收到 start 但永远等不到 end"的场景？

---

### 事件类型与对称结构

代码里定义了 9 个 SSE 事件，按功能分三组对称对：

```typescript
// stream-events.ts
export const STREAM_EVENTS = {
  CHAIN_START:        'on_chain_start',      // 整个链路开始
  CHAIN_END:          'on_chain_end',        // 整个链路结束（成功或错误都会发）
  CHAIN_ERROR:        'on_chain_error',      // 链路级错误通知
  CHAIN_STREAM:       'on_chain_stream',     // 流式文本 token
  CHAIN_SUGGESTIONS:  'on_chain_suggestions',

  CHAT_MODEL_START:   'on_chat_model_start', // LLM 推理开始（含 thinking 阶段）
  CHAT_MODEL_END:     'on_chat_model_end',   // LLM 推理结束

  TOOL_START:         'on_tool_start',       // 工具步骤开始
  TOOL_END:           'on_tool_end',         // 工具步骤结束
};
```

---

### "finally 语义"的实际实现

代码里用的是 **try-catch**，`CHAIN_END` 在成功路径和 catch 路径里各发一次，保证"不管正常还是异常，前端都能收到结束信号"：

```typescript
// dialogue-manager.service.ts
async handleMessageStream(input, subscriber, traceId): Promise<void> {
  try {
    this.streamResponseService.sendEventV2(subscriber, STREAM_EVENTS.CHAIN_START, {
      traceId, phase: 'init', sessionId, userId, startedAt,
    });

    // ... 规划、执行 ...

    // 正常结束：发 CHAIN_END
    this.streamResponseService.sendEventV2(subscriber, STREAM_EVENTS.CHAIN_END, {
      traceId, answer: finalAnswer, result: { type: 'text' },
    });

  } catch (error) {
    // 异常结束：catch 里也发 CHAIN_END，携带兜底文案
    const errMsg = '抱歉，处理您的请求时出现了错误，请稍后重试';
    this.streamResponseService.sendEventV2(subscriber, STREAM_EVENTS.CHAIN_END, {
      traceId, answer: errMsg, result: { type: 'error' },
    });
  }
}
```

---

### OOM 崩溃时 finally 语义能否保证

**不能保证。**

try-catch 的 catch 块能捕获 JavaScript 层面的异常，但**进程级 OOM 崩溃不是 JS 异常**——Node.js 进程被操作系统直接 SIGKILL，V8 堆直接回收，catch 块根本没有机会执行，`CHAIN_END` 永远发不出去。

这是架构上已知的可靠性缺口：没有持久化的"发件箱"，事件只在内存里流转，进程崩溃等于事件消失。

---

### 前端如何处理"收到 start 但永远等不到 end"

前端在 `parseSSEStreamV2` 里检测流是否以 `on_chain_end` 结束：

```typescript
// marketing-dialogue-client.ts
async function parseSSEStreamV2(response, cb): Promise<SSEResult> {
  const state = { accumulated: '', isCompleted: false };

  try {
    while (true) {
      const { done, value } = await reader.read();
      if (done) break;  // 流结束

      if (eventType === 'on_chain_end') {
        state.isCompleted = true;
      }
    }
  } catch (err) {
    if (err?.name === 'AbortError') throw err;
    throw err;
  }

  // 流结束但没有收到 on_chain_end
  if (!state.isCompleted) {
    throw new Error('SSE 流意外结束');  // 抛出，触发上层错误处理
  }

  return { answerText: state.accumulated, ... };
}
```

`reader.read()` 返回 `done: true` 是 TCP 连接正常关闭的信号。如果进程崩溃导致 TCP RST，浏览器的 fetch 会抛网络错误，同样进入 catch 路径。两种情况下，`state.isCompleted` 都是 `false`，前端都会抛出 `'SSE 流意外结束'`，上层 hook 把对话气泡状态设成 `error` 阶段，展示错误提示。

---

### 追问

**Q：CHAIN_ERROR 和 CHAIN_END（type=error）都可以表示错误，两者的区别是什么？**

`CHAIN_ERROR` 是"链路级通知"，用于在还没到结束时提前告知前端有错误发生（比如某个步骤失败，但整个链路还在处理中），前端可以用它更新 UI 中间状态。`CHAIN_END` 是"终结信号"，表示这次请求彻底结束，不管是成功还是失败。目前代码里 catch 里直接发 `CHAIN_END(type=error)` 跳过了 `CHAIN_ERROR`，前端只看 `CHAIN_END` 来决定最终状态，`CHAIN_ERROR` 更多用于调试日志。

**Q：`CHAT_MODEL_START` 和 `CHAT_MODEL_END` 成对出现，如果 LLM 流式推理中途失败，END 会发出吗？**

不一定会。`CHAT_MODEL_START` 发出后，LLM 推理进入 `consumeLlmStream`，如果推理失败会抛异常，被外层 catch 捕获，然后发 `CHAIN_END(type=error)`。但 `CHAT_MODEL_END` 本身可能不会发——catch 路径直接跳到 `CHAIN_END`，`CHAT_MODEL_START` 和 `CHAT_MODEL_END` 这对在异常路径下可能不对称。前端状态机需要在收到 `CHAIN_END` 时无条件清除所有未完成的 "thinking" 状态，不能依赖 `CHAT_MODEL_END` 来清除。

**Q：如果要真正保证 OOM 下也能发出 CHAIN_END，改造方向是什么？**

需要"写前日志"（write-ahead log）：在发送 `CHAIN_START` 之前，先往持久存储（Redis 或 DB）写一条"此 sessionId 有未完成的 chain"记录；进程重启后，守护进程扫描这些未完成记录，向客户端发送错误推送（或等待客户端重连时告知）。代价是每个请求多一次持久化写，且需要额外的"守护进程"机制。目前接受了崩溃时前端靠超时自愈的代价。

---

## Q11. 移动端断线重连与状态恢复

**问题：** SSE 连接在移动端网络切换（WiFi 切 4G）场景下，前端如何感知断线并重建连接？重建后如何恢复到中断前的执行状态？断线期间已推流的 token 如何处理——是重放还是丢弃？

---

### 断线感知机制

SDK 用 `fetch` + `ReadableStream` 实现 SSE，不用 `EventSource`：

```typescript
// marketing-dialogue-client.ts
async streamMessage(req, cb): Promise<SSEResult> {
  this.abortController = new AbortController();

  const response = await fetch(url, {
    method: 'GET',
    headers: { Accept: 'text/event-stream', 'Cache-Control': 'no-cache' },
    signal: this.abortController.signal,
  });

  return await parseSSEStreamV2(response, cb);
}
```

WiFi 切 4G 时，操作系统关闭旧网卡，TCP 连接断开。浏览器行为取决于断开方式：
- **TCP RST**：`reader.read()` 立即抛网络错误，catch 路径捕获
- **TCP FIN 正常关闭**：`reader.read()` 返回 `{ done: true }`，但 `isCompleted` 为 false，抛 `'SSE 流意外结束'`

两种情况前端都能感知，上层 hook 把气泡状态标记为 error。**没有自动重连**，断线后需要用户主动重新发起请求。

---

### 重建后如何恢复执行状态

服务端有专门的恢复接口：

```typescript
// marketing-dialogue.controller.ts
@Sse('resume/stream')
resumeStream(
  @Query('sessionId') sessionId: string,
  @Query('userId') userId: string,
): Observable<MessageEvent> {
  // 从 Redis 读取 pausedExecution 快照，从断点步骤继续执行
  this.dialogueManager
    .handleResumeStream(sessionId, userId, subscriber, traceId, llmCallContext)
    .then(() => subscriber.complete());
}
```

`handleResumeStream` 从 Redis 里取出 `pausedExecution`（包含 plan、stepIndex、inheritedData），重建一个新的 AsyncGenerator 从 `startStepIndex` 继续执行，跳过已完成步骤。

**但这个恢复接口是为"主动暂停"设计的**（用户填表单、缺少槽位），不是为断线重连设计的。如果执行过程中没有触发暂停点（执行到一半网络断了），`pausedExecution` 为空，`resumeStream` 拿不到断点，只能重新开始。

---

### 断线期间已推流的 token：丢弃不重放

**丢弃**。恢复执行从断点步骤重新跑，而不是从流的某个字节偏移重放。用户看到的是"新一轮执行"的输出，之前流过来的 token 不会补发。

这个设计的合理性：营销助手的响应是结构化的步骤结果（"已为您找到链接"、表单、二维码），不是纯文本流。重放部分 token 反而会让 UI 处于中间状态（比如半个二维码卡片），不如从步骤开始处重新展示完整结果。

---

### 追问

**Q：用 `fetch` 而不用原生 `EventSource` 的具体原因是什么？**

`EventSource` 有三个限制：只支持 GET、不能自定义 header、自动重连逻辑不可控。营销助手的请求需要携带认证信息（通过 header 传 token），且断线后不应该无条件重连（需要判断是否有 pausedExecution 再决定走 resume 还是重新发起）。`fetch` + `AbortController` 在这两点上都更灵活，代价是需要自己实现 SSE 行解析（`parseSSELine`），但解析逻辑很简单，不构成维护负担。

**Q：resumeStream 接口是为主动暂停设计的，如果要支持断线重连，需要做什么改造？**

需要两个改动：第一，在执行过程中更高频地写入"检查点"到 Redis，不只在暂停点写，比如每完成一个步骤就更新 `stepIndex` 和 `inheritedData`；第二，前端断线重连时先查询 `pausedExecution` 是否存在，存在则走 resume，不存在则提示用户重新发起。目前每次步骤完成后会更新 `lastResult`，但 `pausedExecution` 只在主动暂停时写入，不覆盖"执行中"的场景。

---

## Q12. 三层可观测日志与积压防控

**问题：** 三层可观测日志中，"日志写入采用事件驱动异步模式，与 SSE 推流主流程解耦"——如果日志写入长期积压（比如 Kafka 消费延迟），积压的日志事件存在哪里？如何防止内存溢出？

---

### 三层日志的架构

```
SSE 推流主流程
  │
  ├─ LLM 推理完成 → emit(LLM_CALL_COMPLETED)  ──→ LlmLogListener → DB 写入
  │
  ├─ 路由步骤完成 → emit(ROUTE_CALL_COMPLETED) ──→ RouteCallLogListener → DB 写入
  │
  └─ HTTP 请求进出 → @LogOperation 装饰器      ──→ OperationLogInterceptor → DB 写入
```

事件发射是 fire-and-forget，不 await：

```typescript
// llm.service.ts — LLM 推理完成后
stream.on('end', () => {
  this.eventEmitter.emit(LLM_CALL_COMPLETED, {
    traceId, sessionId, promptId, llmResponse: fullText,
    tokenUsage: { promptTokens, completionTokens, totalTokens },
    durationMs: Date.now() - startTime,
  });
  // 不 await，主流程立即继续
});
```

监听器用 NestJS `@OnEvent` 异步处理：

```typescript
// llm-log.listener.ts
@OnEvent(LLM_CALL_COMPLETED)
async handleLlmCallCompleted(payload: LlmCallCompletedPayload): Promise<void> {
  try {
    await this.llmLogService.createLog(payload);  // DB 写入
  } catch (err) {
    this.logger.error('LLM 日志事件处理失败', err?.stack);
    // 失败只记录日志，事件丢弃
  }
}
```

---

### 积压的日志事件存在哪里

**当前实现没有引入 Kafka**，日志事件走的是 NestJS EventEmitter2（基于 Node.js 原生 EventEmitter）。

积压的位置是 **Node.js 微任务队列**。`@OnEvent` 的异步 handler 是 Promise，emit 触发后 Promise 进入微任务队列等待执行。如果 DB 写入速度慢于事件发射速度，队列里会积累大量未完成的 Promise，每个 Promise 持有 payload 对象的引用，payload 不会被 GC。

---

### 当前没有积压防控机制

坦诚说：**没有显式的背压（backpressure）控制，也没有队列大小上限。**

EventEmitter2 本身不做流控：emit 多少次就调用多少次 handler，没有任何"积压超过 N 个就丢弃或降级"的逻辑。

如果 DB 写入长期延迟，积压的 Promise 数量会线性增长，每个 Promise 持有约 1-2KB 的 payload（包含完整的 LLM 响应文本），100 个积压事件约 100-200KB，1000 个约 1-2MB，一般不会直接触发 OOM，但会拖慢 GC 速度，间接影响主流程。

**唯一的保护是 DB 写失败时的 catch 块**：写入失败的事件被记录 warn 日志后丢弃，不会无限重试。这防止了"写入失败→无限重试→内存爆炸"，但代价是日志丢失。

---

### 合理的防控方案（当前未实现）

**方案一：事件批处理**——不是每个事件立即写 DB，而是先收集进内存 buffer，每 N 条或每 T 秒批量写一次，降低 DB 写入频率。

**方案二：有界队列 + 丢弃策略**——自己维护一个固定大小的队列（比如 `maxSize: 500`），超出后丢弃最旧的事件，发告警。

**方案三：引入 Bull/BullMQ**——把事件写入 Redis Queue，消费者从 Queue 取事件写 DB，天然解耦且有持久化保障。

目前没做的理由是：营销助手的并发量不高，单用户对话场景下每次请求产生 2-5 个日志事件，DB 写入延迟在百毫秒级，积压窗口极短，实际没有触发过明显积压。如果未来接入高并发场景，批处理是最轻量的改造路径。

---

### 追问

**Q：三层日志里，哪一层信息最全，出了问题首先查哪个？**

LLM 调用日志（第一层）最全，存了完整的 `promptMessages`（含 System Prompt 和所有消息历史）、LLM 原始响应、模型配置、token 用量、耗时和 traceId。出了 Planner 相关的问题（比如规划结果不对、LLM 输出了错误 JSON）首先查这层，可以直接复现当时的 Prompt 上下文。路由步骤日志（第二层）记录了每个 executor 的入参和产出，适合排查"链路哪一步出错"。Operation 日志（第三层）更宏观，适合统计接口成功率。

**Q：`LLM_CALL_COMPLETED` 事件里存了完整的 LLM 响应文本，如果响应很长（比如 2000 token），payload 对象会比较大，有没有考虑过只存摘要或引用？**

没有考虑过，当前直接存全文。2000 token 的响应文本约 4-8KB，写 DB 时是字符串字段。目前没有出现过单条 payload 过大导致问题的情况，因为规划 Prompt 的 `max_tokens: 700` 限制了响应长度，实际存储的响应都很短。如果接入长文本生成场景（比如营销文案写作），需要考虑截断或只存 token 用量不存全文。
