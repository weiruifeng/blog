---
layout: post
title: "编程题：算法 & 前端（含答案）"
date: 2025-01-02
categories: 面试
tags: [算法, 前端, TypeScript]
---

> 题目均结合简历中的真实工程场景设计，考察算法基础、前端工程能力。
> 难度标注：⭐ 热身 ｜ ⭐⭐ 中等 ｜ ⭐⭐⭐ 进阶 ｜ ⭐⭐⭐⭐ 挑战

---

## 一、算法题

---

### A1. 滑动窗口成功率统计 ⭐⭐

**背景：** 你在简历中提到 RPC 链路降级"以滑动窗口成功率而非单次失败作为降级判断依据"。现在请你实现这个核心数据结构。

**题目：**

实现一个 `SlidingWindowCounter`，用于统计最近 N 秒内的请求成功率，支持以下操作：

```ts
class SlidingWindowCounter {
  constructor(windowSizeMs: number, bucketCount: number) {}
  record(success: boolean): void {}
  getSuccessRate(): number | null {}
}
```

**要求：**
- 时间窗口为 `windowSizeMs` 毫秒，内部分为 `bucketCount` 个桶
- 每个桶记录该时间片内的成功数和总数
- 调用 `getSuccessRate()` 时自动丢弃过期桶
- 不能使用数组存储所有历史请求（内存有界）

**示例：**
```
windowSizeMs=10000, bucketCount=10（每桶 1s）
t=0s:   record(true) × 10
t=5s:   record(false) × 5
t=11s:  getSuccessRate()
// t=0s 的桶已过期，只统计 t=1s-10s 的数据
// 返回 5/10 = 0.5
```

**追问：**
1. 如果要支持多实例共享同一个滑动窗口（分布式场景），数据结构需要如何改造？
2. 当 bucketCount 很大时，`getSuccessRate()` 的时间复杂度是多少？如何优化到 O(1)？

---

**思路：**
- 将时间窗口均分为 `bucketCount` 个桶，每个桶宽 `windowSizeMs / bucketCount` 毫秒
- 用桶索引 `Math.floor(now / bucketWidth) % bucketCount` 定位当前桶
- 每次 record 时，若当前桶的时间戳已过期则重置该桶
- getSuccessRate 时遍历所有桶，跳过过期桶累加统计

```ts
class SlidingWindowCounter {
  private bucketWidth: number;
  private buckets: Array<{ success: number; total: number; timestamp: number }>;

  constructor(private windowSizeMs: number, private bucketCount: number) {
    this.bucketWidth = windowSizeMs / bucketCount;
    this.buckets = Array.from({ length: bucketCount }, () => ({
      success: 0, total: 0, timestamp: 0,
    }));
  }

  private currentBucketIndex(now: number): number {
    return Math.floor(now / this.bucketWidth) % this.bucketCount;
  }

  record(success: boolean): void {
    const now = Date.now();
    const idx = this.currentBucketIndex(now);
    const bucket = this.buckets[idx];
    const bucketStartTime = Math.floor(now / this.bucketWidth) * this.bucketWidth;
    if (bucket.timestamp !== bucketStartTime) {
      bucket.success = 0;
      bucket.total = 0;
      bucket.timestamp = bucketStartTime;
    }
    bucket.total++;
    if (success) bucket.success++;
  }

  getSuccessRate(): number | null {
    const now = Date.now();
    const windowStart = now - this.windowSizeMs;
    let totalSuccess = 0, totalCount = 0;
    for (const bucket of this.buckets) {
      if (bucket.timestamp <= windowStart || bucket.total === 0) continue;
      totalSuccess += bucket.success;
      totalCount += bucket.total;
    }
    return totalCount === 0 ? null : totalSuccess / totalCount;
  }
}
```

**追问回答：**

1. **分布式共享：** 用 Redis 替换本地桶数组。每个桶对应两个 Redis Hash 字段，用 `HINCRBY` 原子更新，`EXPIRE` 控制过期。可以用 Lua 脚本保证读写原子性。

2. **O(1) 优化：** 维护全局累计量 `{success, total}`，record 时先减去被覆盖的旧桶数据再加上新数据，getSuccessRate 直接读全局量。代价是每次 record 多一次减法操作。

---

### A2. 有依赖关系的任务调度（拓扑排序变体） ⭐⭐⭐

**背景：** 你的 Agent 规划层"单次 LLM 调用输出含步骤依赖关系的完整执行计划"，执行层需要按依赖关系有序执行各步骤。

**题目：**

```ts
interface Step {
  id: string;
  dependsOn: string[];
  execute: () => Promise<any>;
}

async function executeDAG(steps: Step[]): Promise<Map<string, any>> {}
```

**要求：**
1. 检测循环依赖，有环时抛出错误并指出环路
2. 某个步骤执行失败时，其所有下游步骤不再执行，但不影响其他无关步骤
3. 无依赖的步骤尽可能并行执行

**示例：**
```
A: dependsOn=[]
B: dependsOn=[A]
C: dependsOn=[A]
D: dependsOn=[B, C]

执行顺序：A → (B‖C) → D
```

**追问：**
1. 如果某个步骤执行超时，如何实现超时取消且不影响整体调度？
2. 如果步骤数量达到 10000+，如何限制最大并发数？

---

**思路：**
- 用 Kahn 算法（入度表 + 队列）检测环和确定执行顺序
- 每当一批步骤入度为 0 时，并行执行它们
- 某步骤失败后，标记其所有下游为"跳过"

```ts
async function executeDAG(steps: Step[]): Promise<Map<string, any>> {
  const results = new Map<string, any>();
  const skipped = new Set<string>();
  const inDegree = new Map<string, number>();
  const dependents = new Map<string, string[]>();
  const stepMap = new Map<string, Step>();

  for (const step of steps) {
    stepMap.set(step.id, step);
    inDegree.set(step.id, step.dependsOn.length);
    dependents.set(step.id, []);
  }
  for (const step of steps) {
    for (const dep of step.dependsOn) {
      if (!stepMap.has(dep)) throw new Error(`未知依赖: ${dep}`);
      dependents.get(dep)!.push(step.id);
    }
  }

  // 检测循环依赖
  const tempDeg = new Map(inDegree);
  const tempQueue = [...inDegree.entries()].filter(([, d]) => d === 0).map(([id]) => id);
  const topoOrder: string[] = [];
  while (tempQueue.length > 0) {
    const id = tempQueue.shift()!;
    topoOrder.push(id);
    for (const dep of dependents.get(id)!) {
      tempDeg.set(dep, tempDeg.get(dep)! - 1);
      if (tempDeg.get(dep) === 0) tempQueue.push(dep);
    }
  }
  if (topoOrder.length !== steps.length) {
    const cycleNodes = steps.filter(s => !topoOrder.includes(s.id)).map(s => s.id);
    throw new Error(`检测到循环依赖，涉及节点: ${cycleNodes.join(', ')}`);
  }

  const completedDeg = new Map(inDegree);
  let readyQueue = steps.filter(s => s.dependsOn.length === 0).map(s => s.id);

  const executeStep = async (id: string) => {
    const step = stepMap.get(id)!;
    if (skipped.has(id)) { results.set(id, new Error('上游失败，跳过')); return; }
    try {
      results.set(id, await step.execute());
    } catch (err) {
      results.set(id, err);
      const markSkipped = (nodeId: string) => {
        for (const dep of dependents.get(nodeId)!) { skipped.add(dep); markSkipped(dep); }
      };
      markSkipped(id);
    }
    for (const dep of dependents.get(id)!) {
      completedDeg.set(dep, completedDeg.get(dep)! - 1);
    }
  };

  while (readyQueue.length > 0) {
    await Promise.all(readyQueue.map(id => executeStep(id)));
    const nextBatch: string[] = [];
    for (const [id, deg] of completedDeg) {
      if (deg === 0 && !results.has(id)) nextBatch.push(id);
    }
    readyQueue = nextBatch;
  }
  return results;
}
```

**追问回答：**

1. **超时取消：** 用 `Promise.race([step.execute(), timeout(ms)])` 竞争，超时时将该步骤视为失败，其下游标记跳过。若 execute 支持 AbortController，传入 signal 触发真正取消。

2. **限制最大并发数：** 维护一个 `Semaphore`，核心是计数器 + 等待队列：
```ts
class Semaphore {
  private count: number;
  private queue: Array<() => void> = [];
  constructor(limit: number) { this.count = limit; }
  async acquire() {
    if (this.count > 0) { this.count--; return; }
    await new Promise<void>(r => this.queue.push(r));
  }
  release() { this.count++; this.queue.shift()?.(); }
}
```

---

### A3. LLM 输出的 JSON 多层提取 ⭐⭐

**背景：** 简历中提到"三层 JSON 提取兜底：直接解析 → Markdown 代码块提取 → 正则提取"。

**题目：**

```ts
function extractJSON(text: string): object | null {
  // 层 1：整个字符串直接 JSON.parse
  // 层 2：提取 ```json ... ``` 代码块内容再解析
  // 层 3：正则扫描文本中最外层的 { } 或 [ ] 结构
}
```

**测试用例（必须全部通过）：**
```ts
extractJSON('{"name":"test","value":42}')                   // → {name:"test", value:42}
extractJSON('好的：\n```json\n{"code":0}\n```\n请查收。')    // → {code:0}
extractJSON('结果：\n```\n{"status":"ok"}\n```')            // → {status:"ok"}
extractJSON('答案是 {"result": true} 你觉得呢？')            // → {result:true}
extractJSON('{"a":{"b":{"c":1}}} 结束')                    // → {a:{b:{c:1}}}
extractJSON('这是普通文字')                                   // → null
extractJSON('{"name": "test"')                              // → null（残缺）
```

**追问：**
1. 如果 LLM 输出了多个 JSON 块，返回哪个？
2. 如何防止正则提取层产生 ReDoS 风险？

---

**思路：**
- 层 1：直接 JSON.parse
- 层 2：正则提取代码块，parse 内容
- 层 3：括号计数扫描（不用贪婪正则，避免 ReDoS）

```ts
function extractJSON(text: string): object | null {
  // 层 1
  try { return JSON.parse(text.trim()); } catch {}

  // 层 2
  const codeBlockRegex = /```(?:json)?\s*\n?([\s\S]*?)\n?```/g;
  let match: RegExpExecArray | null;
  while ((match = codeBlockRegex.exec(text)) !== null) {
    try { return JSON.parse(match[1].trim()); } catch {}
  }

  // 层 3：括号计数扫描
  const startChars = new Set(['{', '[']);
  for (let i = 0; i < text.length; i++) {
    if (!startChars.has(text[i])) continue;
    const closeChar = text[i] === '{' ? '}' : ']';
    const openChar = text[i];
    let depth = 0, inString = false, escaped = false;
    for (let j = i; j < text.length; j++) {
      const ch = text[j];
      if (escaped) { escaped = false; continue; }
      if (ch === '\\' && inString) { escaped = true; continue; }
      if (ch === '"') { inString = !inString; continue; }
      if (inString) continue;
      if (ch === openChar) depth++;
      else if (ch === closeChar) {
        depth--;
        if (depth === 0) {
          try { return JSON.parse(text.slice(i, j + 1)); } catch {}
          break;
        }
      }
    }
  }
  return null;
}
```

**追问回答：**

1. **多个 JSON 块：** 当前返回第一个成功解析的。实际应视业务场景：如果 LLM 输出通常是"分析文字 + 结构化结果"，应返回**最后一个**；也可返回所有 JSON 数组由调用方选择。

2. **ReDoS 风险：** 层 3 已改用括号计数扫描，时间复杂度严格 O(n)，无回溯。层 2 的 `[\s\S]*?` 是惰性匹配，配合锚定分隔符 ` ``` `，回溯路径有限。若超偏执，可加字符数限制：`if (text.length > 1_000_000) return null`。

---

### A4. 异步任务队列（防止事件循环阻塞） ⭐⭐⭐

**背景：** 简历中提到"大批量数据处理通过异步化调度防止阻塞事件循环"。

**题目：**

```ts
class AsyncChunkProcessor<T, R> {
  constructor(
    items: T[],
    processor: (item: T) => R,
    options: {
      chunkSize: number;
      onProgress?: (processed: number, total: number) => void;
    }
  ) {}
  async run(): Promise<R[]> {}
  cancel(): void {}
  getPartialResults(): R[] {}
}
```

**要求：**
- 每个 chunk 处理完后必须让出事件循环
- 取消后 `run()` 的 Promise resolve（不 reject），返回空数组
- 可通过 `getPartialResults()` 获取取消前的进度

**追问：**
1. `setImmediate`、`setTimeout(fn, 0)`、`process.nextTick` 有什么区别？哪个最适合？
2. 如果 processor 是异步函数，如何修改？

---

```ts
class AsyncChunkProcessor<T, R> {
  private results: R[] = [];
  private cancelled = false;

  constructor(
    private items: T[],
    private processor: (item: T) => R,
    private options: { chunkSize: number; onProgress?: (processed: number, total: number) => void }
  ) {}

  async run(): Promise<R[]> {
    const { chunkSize, onProgress } = this.options;
    const total = this.items.length;
    let i = 0;
    while (i < total && !this.cancelled) {
      const end = Math.min(i + chunkSize, total);
      for (let j = i; j < end; j++) {
        this.results.push(this.processor(this.items[j]));
      }
      i = end;
      onProgress?.(i, total);
      await yieldControl();
    }
    return this.cancelled ? [] : this.results;
  }

  cancel(): void { this.cancelled = true; }
  getPartialResults(): R[] { return [...this.results]; }
}

function yieldControl(): Promise<void> {
  return new Promise(resolve => {
    if (typeof setImmediate !== 'undefined') setImmediate(resolve);
    else setTimeout(resolve, 0);
  });
}
```

**追问回答：**

1. **三者区别：**
   - `process.nextTick`：当前事件循环末尾、I/O 回调之前，**不让出** I/O 事件，不适合此场景。
   - `setTimeout(fn, 0)`：进入 timers 阶段，至少延迟 1ms，让出控制权彻底，浏览器中唯一选择。
   - `setImmediate`：check 阶段执行（I/O 回调之后），比 setTimeout 更稳定，**最适合 Node.js 场景**。

2. **processor 是异步函数时：** 改为 `(item: T) => Promise<R>`，批处理时用 `Promise.all` 并行执行一批：
```ts
const chunkResults = await Promise.all(
  this.items.slice(i, end).map(item => this.processor(item))
);
this.results.push(...chunkResults);
```

---

### A5. TCP 粘包拆包解析器 ⭐⭐⭐⭐

**背景：** 简历中提到"状态机解析 TCP 流完成粘包拆包"。

**题目：**

协议格式为 TLV（Type-Length-Value）：
```
| 1 byte: type | 4 bytes: length (big-endian uint32) | N bytes: payload |
```

```ts
class TLVParser extends EventEmitter {
  feed(chunk: Buffer): void {}
  // 解析完整包后 emit('message', { type: number, payload: Buffer })
}
```

**要求：** 正确处理半包、粘包、跨包三种场景，使用状态机，时间复杂度 O(chunk 大小)。

**追问：**
1. 连接断开时内部 buffer 中还有未完成的半包，如何清理和上报？
2. 如何防止恶意客户端发送超大 length 字段耗尽服务器内存？

---

**思路：**
- 状态机三个状态：`READ_TYPE`（读 1 字节）、`READ_LENGTH`（读 4 字节）、`READ_PAYLOAD`（读 N 字节）
- 维护 `buffer` 积累未解析数据，每次 feed 后循环尝试解析直到数据不足

```ts
import { EventEmitter } from 'events';

type ParserState = 'READ_TYPE' | 'READ_LENGTH' | 'READ_PAYLOAD';

class TLVParser extends EventEmitter {
  private buffer: Buffer = Buffer.alloc(0);
  private state: ParserState = 'READ_TYPE';
  private currentType = 0;
  private expectedLength = 0;

  feed(chunk: Buffer): void {
    this.buffer = Buffer.concat([this.buffer, chunk]);
    this.parse();
  }

  private parse(): void {
    while (true) {
      if (this.state === 'READ_TYPE') {
        if (this.buffer.length < 1) break;
        this.currentType = this.buffer[0];
        this.buffer = this.buffer.slice(1);
        this.state = 'READ_LENGTH';
      } else if (this.state === 'READ_LENGTH') {
        if (this.buffer.length < 4) break;
        this.expectedLength = this.buffer.readUInt32BE(0);
        // 防超大包
        const MAX_PAYLOAD_SIZE = 64 * 1024 * 1024; // 64MB
        if (this.expectedLength > MAX_PAYLOAD_SIZE) {
          this.emit('error', new Error(`非法包长度: ${this.expectedLength}`));
          this.destroy(); return;
        }
        this.buffer = this.buffer.slice(4);
        this.state = 'READ_PAYLOAD';
      } else if (this.state === 'READ_PAYLOAD') {
        if (this.buffer.length < this.expectedLength) break;
        const payload = this.buffer.slice(0, this.expectedLength);
        this.buffer = this.buffer.slice(this.expectedLength);
        this.emit('message', { type: this.currentType, payload });
        this.state = 'READ_TYPE';
      }
    }
  }

  destroy(): void {
    if (this.buffer.length > 0) {
      this.emit('error', new Error(`连接断开，残留 ${this.buffer.length} 字节未完成包`));
    }
    this.buffer = Buffer.alloc(0);
    this.state = 'READ_TYPE';
    this.removeAllListeners();
  }
}
```

**追问回答：**

1. **半包清理：** 调用 `destroy()`，检查 buffer 是否有残留数据，有则 emit `error` 上报（包含残留字节数和当前状态），然后清空 buffer 和监听器，防止内存泄漏。

2. **防内存耗尽：** 解析到 `expectedLength` 后立即校验是否超过最大值（如 64MB），超过则 emit error 并关闭连接（代码已内置此逻辑）。

---

## 二、前端编程题

---

### F1. SSE 流式客户端解析器（含断线重连） ⭐⭐⭐

**背景：** 简历中设计了"8 种对称 SSE 事件"，前端需要一个健壮的 SSE 解析器。

**题目：**

```ts
class SSEClient {
  constructor(url: string, options?: {
    headers?: Record<string, string>;
    maxRetries?: number;       // 默认 3
    retryDelay?: number;       // 默认 1000ms
    onMessage: (event: SSEEvent) => void;
    onError?: (err: Error) => void;
    onReconnect?: (attempt: number) => void;
  }) {}
  connect(): void {}
  close(): void {}
}
```

**要求：**
1. 使用 `fetch` + `ReadableStream`（不能用浏览器原生 `EventSource`，不支持自定义请求头）
2. 正确解析 SSE 格式（多行 `data:` 拼接、空行分隔事件）
3. 断线后用 `Last-Event-ID` 请求头续传
4. 服务端主动关闭时不触发重连
5. 网络错误时指数退避重连

**追问：**
1. 如何区分"服务端主动关闭"和"网络中断"？
2. 如果 SSE 流携带二进制数据（base64 编码），如何修改解析逻辑？

---

```ts
interface SSEEvent { event?: string; data: string; id?: string; }

class SSEClient {
  private abortController: AbortController | null = null;
  private lastEventId = '';
  private retryCount = 0;
  private closed = false;
  private url: string;
  private headers: Record<string, string>;
  private maxRetries: number;
  private retryDelay: number;
  private onMessage: (event: SSEEvent) => void;
  private onError?: (err: Error) => void;
  private onReconnect?: (attempt: number) => void;

  constructor(url: string, options?: any) {
    this.url = url;
    this.headers = options?.headers ?? {};
    this.maxRetries = options?.maxRetries ?? 3;
    this.retryDelay = options?.retryDelay ?? 1000;
    this.onMessage = options!.onMessage;
    this.onError = options?.onError;
    this.onReconnect = options?.onReconnect;
  }

  connect(): void { this.closed = false; this.retryCount = 0; this.doConnect(); }
  close(): void { this.closed = true; this.abortController?.abort(); }

  private async doConnect(): Promise<void> {
    this.abortController = new AbortController();
    const headers: Record<string, string> = { Accept: 'text/event-stream', ...this.headers };
    if (this.lastEventId) headers['Last-Event-ID'] = this.lastEventId;
    try {
      const response = await fetch(this.url, { headers, signal: this.abortController.signal });
      if (!response.ok || !response.body) throw new Error(`HTTP ${response.status}`);
      await this.readStream(response.body);
      // 流正常结束，不重连
    } catch (err: any) {
      if (this.closed) return;
      this.onError?.(err instanceof Error ? err : new Error(String(err)));
      await this.scheduleReconnect();
    }
  }

  private async readStream(body: ReadableStream<Uint8Array>): Promise<void> {
    const reader = body.getReader();
    const decoder = new TextDecoder();
    let buffer = '';
    try {
      while (true) {
        const { done, value } = await reader.read();
        if (done) break;
        buffer += decoder.decode(value, { stream: true });
        const parts = buffer.split('\n\n');
        buffer = parts.pop()!;
        for (const part of parts) this.parseEvent(part);
      }
    } finally { reader.releaseLock(); }
  }

  private parseEvent(raw: string): void {
    const lines = raw.split('\n');
    const event: SSEEvent = { data: '' };
    const dataParts: string[] = [];
    for (const line of lines) {
      if (line.startsWith('event:')) event.event = line.slice(6).trim();
      else if (line.startsWith('data:')) dataParts.push(line.slice(5).trim());
      else if (line.startsWith('id:')) { event.id = line.slice(3).trim(); this.lastEventId = event.id; }
    }
    event.data = dataParts.join('\n');
    if (event.data) this.onMessage(event);
  }

  private async scheduleReconnect(): Promise<void> {
    if (this.retryCount >= this.maxRetries) return;
    this.retryCount++;
    const delay = this.retryDelay * Math.pow(2, this.retryCount - 1);
    this.onReconnect?.(this.retryCount);
    await new Promise(r => setTimeout(r, delay));
    if (!this.closed) this.doConnect();
  }
}
```

**追问回答：**

1. **区分主动关闭 vs 网络中断：** 服务端主动关闭时，`reader.read()` 返回 `{ done: true }`，流正常结束不抛异常。网络中断时，`reader.read()` 或 `fetch` 会抛出 `TypeError: Failed to fetch`。`readStream` 正常 return 时不重连，在 `catch` 块中才重连。

2. **二进制数据（base64）：** SSE 是文本协议，base64 数据作为普通字符串在 `data:` 字段传输。在 `onMessage` 回调中检测 `event.event === 'binary'`，对 `event.data` 做解码：
```ts
const binary = Uint8Array.from(atob(event.data), c => c.charCodeAt(0));
```

---

### F2. 声明式埋点系统 ⭐⭐⭐

**背景：** 简历中"通过事件委托和自动曝光组件实现声明式埋点，覆盖自动 PV、声明式点击、可见性曝光三类场景"。

**题目：**

```html
<button data-track-click='{"event":"buy_click","item_id":"123"}'>立即购买</button>
<div data-track-expose='{"event":"card_expose","card_id":"456"}'>商品卡片</div>
<div data-track-expose='{"event":"banner_expose"}' data-track-expose-once="false">Banner</div>
```

```ts
class TrackingSystem {
  constructor(options: {
    report: (event: object) => void;
    root?: Element;
    exposeThreshold?: number;  // 默认 0.5
  }) {}
  init(): void {}
  destroy(): void {}
  observe(element: Element): void {}
}
```

**要求：**
1. 点击埋点用事件委托，不直接绑定到每个元素
2. 曝光埋点用 `IntersectionObserver`
3. `data-track-expose-once="false"` 的元素每次进入视口都上报
4. `destroy()` 彻底清理所有监听器
5. 对非法 JSON 属性值容错，不崩溃

---

```ts
class TrackingSystem {
  private root: Element;
  private report: (event: object) => void;
  private exposeThreshold: number;
  private observer: IntersectionObserver | null = null;
  private clickHandler: ((e: Event) => void) | null = null;
  private observedElements = new WeakSet<Element>();

  constructor(options: { report: (event: object) => void; root?: Element; exposeThreshold?: number }) {
    this.report = options.report;
    this.root = options.root ?? document.documentElement;
    this.exposeThreshold = options.exposeThreshold ?? 0.5;
  }

  init(): void {
    this.clickHandler = (e: Event) => {
      const target = (e.target as Element).closest('[data-track-click]');
      if (!target) return;
      const raw = target.getAttribute('data-track-click');
      try { this.report(JSON.parse(raw!)); } catch {
        console.warn('[TrackingSystem] 非法 data-track-click JSON:', raw);
      }
    };
    this.root.addEventListener('click', this.clickHandler);

    this.observer = new IntersectionObserver((entries) => {
      for (const entry of entries) {
        if (!entry.isIntersecting) continue;
        const el = entry.target;
        const raw = el.getAttribute('data-track-expose');
        const once = el.getAttribute('data-track-expose-once') !== 'false';
        try { this.report(JSON.parse(raw!)); } catch {
          console.warn('[TrackingSystem] 非法 data-track-expose JSON:', raw);
        }
        if (once) this.observer?.unobserve(el);
      }
    }, { threshold: this.exposeThreshold });

    this.root.querySelectorAll('[data-track-expose]').forEach(el => this.observe(el));
  }

  observe(element: Element): void {
    if (!this.observer || this.observedElements.has(element)) return;
    this.observedElements.add(element);
    this.observer.observe(element);
  }

  destroy(): void {
    if (this.clickHandler) {
      this.root.removeEventListener('click', this.clickHandler);
      this.clickHandler = null;
    }
    this.observer?.disconnect();
    this.observer = null;
  }
}
```

**追问回答：**

1. **1000 个元素用一个 IntersectionObserver：** 用**一个**，将所有元素通过 `observe()` 注册进去。`IntersectionObserver` 设计目标就是批量监听，回调参数 `entries` 是数组，一次回调处理多个元素状态变化。多个实例实例反而是开销。

2. **被遮挡误触发：** `IntersectionObserver` 只计算几何相交，无法感知 z-index 遮挡。解决方案：在回调中用 `document.elementFromPoint` 检测命中元素是否是目标本身或其子元素，若不是则判定被遮挡，不上报。注意只在 `isIntersecting` 为 true 时调用以降低性能开销。

---

### F3. 分片渲染长列表 ⭐⭐

**背景：** 简历性能优化中提到"长列表分片渲染"。

**题目：**

```ts
async function renderChunked(options: {
  container: HTMLElement;
  items: any[];
  renderItem: (item: any, index: number) => HTMLElement;
  chunkSize?: number;           // 默认 50
  onProgress?: (rendered: number, total: number) => void;
}): Promise<void> {}
```

**要求：**
- 每批渲染后让出控制权给浏览器（优先使用 `requestAnimationFrame`）
- 单批 `renderItem` 耗时超过 16ms 时自动减小 `chunkSize`（自适应分片）
- 渲染过程中用户可以正常交互

---

```ts
async function renderChunked(options: {
  container: HTMLElement;
  items: any[];
  renderItem: (item: any, index: number) => HTMLElement;
  chunkSize?: number;
  onProgress?: (rendered: number, total: number) => void;
}): Promise<void> {
  const { container, items, renderItem, onProgress } = options;
  let chunkSize = options.chunkSize ?? 50;
  const total = items.length;
  let rendered = 0;

  const renderChunk = (): Promise<void> =>
    new Promise(resolve => {
      requestAnimationFrame(() => {
        const start = rendered;
        const end = Math.min(start + chunkSize, total);
        const startTime = performance.now();
        const fragment = document.createDocumentFragment();
        for (let i = start; i < end; i++) fragment.appendChild(renderItem(items[i], i));
        container.appendChild(fragment);
        rendered = end;

        const elapsed = performance.now() - startTime;
        if (elapsed > 16) chunkSize = Math.max(10, Math.floor(chunkSize * 0.7));
        else if (elapsed < 8) chunkSize = Math.min(Math.floor(chunkSize * 1.3), 500);

        onProgress?.(rendered, total);
        resolve();
      });
    });

  while (rendered < total) await renderChunk();
}
```

**追问回答：**

1. **rAF vs rIC：**
   - `requestAnimationFrame`：每帧绘制前执行，适合需要视觉反馈的 DOM 操作（用户能看到渐进填充效果）。
   - `requestIdleCallback`：浏览器空闲时执行，适合非紧急后台任务。此场景用 **rAF**，DOM 插入需与渲染帧同步，用户希望列表尽快出现。

2. **虚拟滚动核心思路：** 核心是 `startIndex = Math.floor(scrollTop / itemHeight)`，`endIndex = startIndex + Math.ceil(clientHeight / itemHeight) + buffer`，只渲染可见范围节点，容器用占位 div 撑开总高度，节点用 `position: absolute; top: index * itemHeight` 定位，监听 `scroll` 更新范围。

---

### F4. 实现一个 React useSSE Hook ⭐⭐⭐

**背景：** 简历将"SSE 流式解析、多轮会话管理封装为内部 SDK"。

**题目：**

```ts
function useSSE(url: string | null, options?: {
  headers?: Record<string, string>;
  onDone?: (fullText: string) => void;
}): SSEState & { start: () => void; stop: () => void; reset: () => void }
```

**要求：**
1. 组件卸载时自动停止 SSE，不产生内存泄漏
2. `url` 变化时，旧连接先关闭再建立新连接
3. 同一时刻只允许一个活跃连接
4. 流式 token 到达时批量更新（避免每个 token 都触发一次渲染）
5. `status` 状态转换：`idle → connecting → streaming → done/error`

---

```ts
import { useCallback, useEffect, useReducer, useRef } from 'react';

interface SSEState {
  status: 'idle' | 'connecting' | 'streaming' | 'done' | 'error';
  messages: string[];
  fullText: string;
  error: Error | null;
}

type Action =
  | { type: 'CONNECTING' }
  | { type: 'APPEND'; payload: string }
  | { type: 'DONE' }
  | { type: 'ERROR'; payload: Error }
  | { type: 'RESET' };

function reducer(state: SSEState, action: Action): SSEState {
  switch (action.type) {
    case 'CONNECTING':
      return { status: 'connecting', messages: [], fullText: '', error: null };
    case 'APPEND': {
      const messages = [...state.messages, action.payload];
      return { ...state, status: 'streaming', messages, fullText: messages.join('') };
    }
    case 'DONE': return { ...state, status: 'done' };
    case 'ERROR': return { ...state, status: 'error', error: action.payload };
    case 'RESET': return { status: 'idle', messages: [], fullText: '', error: null };
    default: return state;
  }
}

function useSSE(url: string | null, options?: { headers?: Record<string, string>; onDone?: (fullText: string) => void }) {
  const [state, dispatch] = useReducer(reducer, { status: 'idle', messages: [], fullText: '', error: null });
  const abortRef = useRef<AbortController | null>(null);
  const pendingRef = useRef<string[]>([]);
  const flushTimerRef = useRef<ReturnType<typeof setTimeout> | null>(null);
  const stateRef = useRef(state);
  stateRef.current = state;

  const scheduleFlush = useCallback(() => {
    if (flushTimerRef.current) return;
    flushTimerRef.current = setTimeout(() => {
      flushTimerRef.current = null;
      const tokens = pendingRef.current.splice(0);
      tokens.forEach(t => dispatch({ type: 'APPEND', payload: t }));
    }, 16);
  }, []);

  const stop = useCallback(() => {
    abortRef.current?.abort();
    abortRef.current = null;
    if (flushTimerRef.current) { clearTimeout(flushTimerRef.current); flushTimerRef.current = null; }
  }, []);

  const start = useCallback(async () => {
    if (!url) return;
    stop();
    dispatch({ type: 'CONNECTING' });
    const controller = new AbortController();
    abortRef.current = controller;
    try {
      const response = await fetch(url, {
        headers: { Accept: 'text/event-stream', ...(options?.headers ?? {}) },
        signal: controller.signal,
      });
      if (!response.body) throw new Error('No response body');
      const reader = response.body.getReader();
      const decoder = new TextDecoder();
      let buffer = '';
      while (true) {
        const { done, value } = await reader.read();
        if (done) break;
        buffer += decoder.decode(value, { stream: true });
        const parts = buffer.split('\n\n');
        buffer = parts.pop()!;
        for (const part of parts) {
          for (const line of part.split('\n')) {
            if (line.startsWith('data:')) { pendingRef.current.push(line.slice(5).trim()); scheduleFlush(); }
          }
        }
      }
      dispatch({ type: 'DONE' });
      options?.onDone?.(stateRef.current.fullText);
    } catch (err: any) {
      if (err.name === 'AbortError') return;
      dispatch({ type: 'ERROR', payload: err instanceof Error ? err : new Error(String(err)) });
    }
  }, [url, stop, scheduleFlush, options]);

  const reset = useCallback(() => { stop(); dispatch({ type: 'RESET' }); }, [stop]);

  useEffect(() => () => stop(), [url, stop]);
  useEffect(() => () => stop(), [stop]);

  return { ...state, start, stop, reset };
}
```

**追问回答：**

1. **每秒 100 token 都触发 setState 的问题：** 每秒 100 次渲染会卡顿。用 `pendingRef` 缓冲 token，用 `setTimeout(16ms)` 批量合并，一帧内多个 token 只触发一次 `dispatch`（即代码中 `scheduleFlush` 的作用）。

2. **useReducer 重构优势：** 状态转换集中在 reducer 中，非法转换可在 reducer 内拦截；测试时只需测纯函数，不需要渲染组件；并发模式下 React 可能多次调用 reducer，纯函数保证幂等。

---

### F5. 实现带超时的 Promise 连接池 ⭐⭐⭐

**背景：** 简历中实现了"连接池，超时后销毁而非归还连接"。

**题目：**

```ts
class ConnectionPool<T> {
  constructor(options: {
    create: () => Promise<T>;
    destroy: (conn: T) => Promise<void>;
    validate?: (conn: T) => boolean;
    maxSize: number;
    acquireTimeout: number;
  }) {}
  async acquire(): Promise<T> {}
  async release(conn: T): Promise<void> {}
  async destroy(conn: T): Promise<void> {}
  getStats(): { total: number; idle: number; waiting: number } {}
}
```

**要求：**
- 空闲连接优先复用，无空闲且未达上限则新建
- 达到上限时排队等待，超时则 reject
- 归还时调用 `validate`，验证失败则销毁
- 排队请求按 FIFO 顺序分配

---

```ts
class ConnectionPool<T> {
  private idle: T[] = [];
  private waiting: Array<{ resolve: (conn: T) => void; reject: (err: Error) => void; timer: ReturnType<typeof setTimeout> }> = [];
  private totalCount = 0;

  constructor(private options: {
    create: () => Promise<T>;
    destroy: (conn: T) => Promise<void>;
    validate?: (conn: T) => boolean;
    maxSize: number;
    acquireTimeout: number;
  }) {}

  async acquire(): Promise<T> {
    while (this.idle.length > 0) {
      const conn = this.idle.pop()!;
      if (!this.options.validate || this.options.validate(conn)) return conn;
      this.totalCount--;
      await this.options.destroy(conn);
    }
    if (this.totalCount < this.options.maxSize) {
      this.totalCount++;
      try { return await this.options.create(); }
      catch (err) { this.totalCount--; throw err; }
    }
    return new Promise<T>((resolve, reject) => {
      const timer = setTimeout(() => {
        const idx = this.waiting.findIndex(w => w.timer === timer);
        if (idx !== -1) this.waiting.splice(idx, 1);
        reject(new Error(`获取连接超时（${this.options.acquireTimeout}ms）`));
      }, this.options.acquireTimeout);
      this.waiting.push({ resolve, reject, timer });
    });
  }

  async release(conn: T): Promise<void> {
    if (this.waiting.length > 0) {
      const waiter = this.waiting.shift()!;
      clearTimeout(waiter.timer);
      if (!this.options.validate || this.options.validate(conn)) {
        waiter.resolve(conn); return;
      }
      this.totalCount--;
      await this.options.destroy(conn);
      this.totalCount++;
      try { waiter.resolve(await this.options.create()); }
      catch (err) { this.totalCount--; waiter.reject(err as Error); }
      return;
    }
    if (!this.options.validate || this.options.validate(conn)) {
      this.idle.push(conn);
    } else {
      this.totalCount--;
      await this.options.destroy(conn);
    }
  }

  async destroy(conn: T): Promise<void> {
    this.totalCount--;
    await this.options.destroy(conn);
  }

  getStats() {
    return { total: this.totalCount, idle: this.idle.length, waiting: this.waiting.length };
  }
}
```

**追问回答：**

1. **连接预热：** 提供 `warmup(minSize)` 方法，并发创建 `minSize` 个连接：
```ts
async warmup(minSize: number): Promise<void> {
  const tasks = Array.from({ length: minSize }, () =>
    this.acquire().then(conn => this.release(conn))
  );
  await Promise.allSettled(tasks);
}
```

2. **create 过程中连接池被销毁：** 在 `create()` 前检查 `destroyed` 标志，create 完成后若已 destroyed 立即销毁连接：
```ts
if (this.destroyed) {
  await this.options.destroy(conn);
  this.totalCount--;
  throw new Error('连接池已销毁');
}
```
