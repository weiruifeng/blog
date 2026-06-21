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
  // 环形数组：固定长度 bucketCount，每个桶记录一个时间片的数据
  private buckets: Array<{ success: number; total: number; timestamp: number }>;

  constructor(private windowSizeMs: number, private bucketCount: number) {
    // 每个桶负责的时间长度，例如 10000ms / 10 = 每桶 1000ms
    this.bucketWidth = windowSizeMs / bucketCount;
    // 初始化所有桶，timestamp=0 表示尚未写入过数据
    this.buckets = Array.from({ length: bucketCount }, () => ({
      success: 0,
      total: 0,
      timestamp: 0,
    }));
  }

  // 计算当前时间落在哪个桶：先算全局桶编号，再取模映射到环形数组下标
  // 例如 now=15500ms, bucketWidth=1000ms → 全局桶号=15 → 下标=15%10=5
  private currentBucketIndex(now: number): number {
    return Math.floor(now / this.bucketWidth) % this.bucketCount;
  }

  record(success: boolean): void {
    const now = Date.now();
    const idx = this.currentBucketIndex(now);
    const bucket = this.buckets[idx];
    // bucketStartTime：当前桶的"起始时间戳"，用于判断桶是否已过期
    // 例如 now=15500, bucketWidth=1000 → bucketStartTime=15000
    const bucketStartTime = Math.floor(now / this.bucketWidth) * this.bucketWidth;
    // 如果桶上记录的时间戳不等于当前周期，说明这个下标已被上一轮数据占用，需要清空（惰性重置）
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
    // windowStart：窗口起点，早于此时间的桶数据应被丢弃
    const windowStart = now - this.windowSizeMs;
    let totalSuccess = 0;
    let totalCount = 0;

    for (const bucket of this.buckets) {
      // 过期桶（timestamp <= windowStart）或空桶跳过
      if (bucket.timestamp <= windowStart || bucket.total === 0) continue;
      totalSuccess += bucket.success;
      totalCount += bucket.total;
    }

    // 窗口内无任何请求时返回 null（避免返回误导性的 0）
    return totalCount === 0 ? null : totalSuccess / totalCount;
  }
}
```

**追问回答：**

1. **分布式共享滑动窗口：** 用 Redis 替换本地桶数组。每个桶对应两个 Redis Hash 字段（`bucket:{windowId}:{bucketIndex}:success` 和 `total`），用 `HINCRBY` 原子更新，`EXPIRE` 控制过期。读取时用 `HMGET` 批量拉取所有桶。可以用 Lua 脚本保证读写原子性。

2. **O(1) 优化：** 维护一个全局累计量 `{success, total}`，record 时：先减去被覆盖的旧桶数据，再加上新数据，更新全局量。getSuccessRate 直接读全局量，时间复杂度 O(1)。代价是每次 record 需要多一次减法操作。

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
- 某步骤失败后，标记其所有下游为"跳过"，但不影响其他无关路径

```ts
async function executeDAG(steps: Step[]): Promise<Map<string, any>> {
  const results = new Map<string, any>();
  const skipped = new Set<string>(); // 因上游失败而需要跳过的步骤

  // ── 第一步：构建图结构 ──────────────────────────────────────────
  const inDegree = new Map<string, number>();   // 每个节点还剩多少未完成的依赖
  const dependents = new Map<string, string[]>(); // 反向图：id → "依赖它的步骤"列表
  const stepMap = new Map<string, Step>();

  for (const step of steps) {
    stepMap.set(step.id, step);
    inDegree.set(step.id, step.dependsOn.length); // 入度 = 依赖数量
    dependents.set(step.id, []);
  }
  // 建立反向边：A 依赖 B，则 dependents[B] 包含 A（B 完成后要通知 A）
  for (const step of steps) {
    for (const dep of step.dependsOn) {
      if (!stepMap.has(dep)) throw new Error(`未知依赖: ${dep}`);
      dependents.get(dep)!.push(step.id);
    }
  }

  // ── 第二步：Kahn 算法检测环（用临时副本，不影响后续执行） ────────
  const tempQueue = [...steps.filter(s => s.dependsOn.length === 0).map(s => s.id)];
  const tempDeg = new Map(inDegree); // 深拷贝入度表，避免影响执行阶段
  const topoOrder: string[] = [];
  while (tempQueue.length > 0) {
    const id = tempQueue.shift()!;
    topoOrder.push(id);
    // 把该节点"移除"后，更新其所有下游节点的入度
    for (const dep of dependents.get(id)!) {
      tempDeg.set(dep, tempDeg.get(dep)! - 1);
      if (tempDeg.get(dep) === 0) tempQueue.push(dep); // 入度归零则加入队列
    }
  }
  // 若拓扑序列长度 < 总节点数，说明有节点永远无法入队 → 存在环
  if (topoOrder.length !== steps.length) {
    const cycleNodes = steps.filter(s => !topoOrder.includes(s.id)).map(s => s.id);
    throw new Error(`检测到循环依赖，涉及节点: ${cycleNodes.join(', ')}`);
  }

  // ── 第三步：并行执行（层序 BFS） ────────────────────────────────
  // completedDeg 跟踪执行过程中的入度（区别于检测用的 tempDeg）
  const completedDeg = new Map(inDegree);
  // 初始就绪队列：所有入度为 0 的节点（无依赖，可立即并行执行）
  let readyQueue = steps.filter(s => s.dependsOn.length === 0).map(s => s.id);

  const executeStep = async (id: string): Promise<void> => {
    const step = stepMap.get(id)!;
    // 上游失败导致跳过：记录错误结果但不抛出，不阻断其他无关路径
    if (skipped.has(id)) {
      results.set(id, new Error(`上游失败，跳过`));
      return;
    }
    try {
      const result = await step.execute();
      results.set(id, result);
    } catch (err) {
      results.set(id, err);
      // DFS 递归标记所有下游为跳过（失败具有传播性）
      const markSkipped = (nodeId: string) => {
        for (const dep of dependents.get(nodeId)!) {
          skipped.add(dep);
          markSkipped(dep); // 递归传播到更深层下游
        }
      };
      markSkipped(id);
    }
    // 无论成功失败，都要减少下游入度（下游需要知道这个上游"结束了"）
    for (const dep of dependents.get(id)!) {
      completedDeg.set(dep, completedDeg.get(dep)! - 1);
    }
  };

  // 每轮并行执行当前所有就绪节点，执行完后找出新的就绪节点
  while (readyQueue.length > 0) {
    await Promise.all(readyQueue.map(id => executeStep(id)));
    // 入度降为 0 且尚未执行的节点，构成下一批就绪队列
    const nextBatch: string[] = [];
    for (const [id, deg] of completedDeg) {
      if (deg === 0 && !results.has(id)) {
        nextBatch.push(id);
      }
    }
    readyQueue = nextBatch;
  }

  return results;
}
```

**追问回答：**

1. **超时取消：** 用 `Promise.race([step.execute(), timeout(ms)])` 竞争，超时时 resolve 一个特殊错误对象，将该步骤视为失败处理，其下游标记跳过。若 execute 内部支持 AbortController，传入 signal 触发真正取消。

2. **限制最大并发数（10000+ 步骤）：** 维护一个信号量（`Semaphore`），核心是一个计数器 + 等待队列。每次执行前 `acquire()`（超出上限则等待），执行后 `release()`。实现：
```ts
class Semaphore {
  private count: number; // 当前可用"许可证"数量（>0 时可直接 acquire）
  private queue: Array<() => void> = []; // 等待许可证的 resolve 函数队列（FIFO）
  constructor(limit: number) { this.count = limit; } // limit 即最大并发数
  async acquire() {
    if (this.count > 0) {
      this.count--; // 直接拿到许可证，不等待
      return;
    }
    // 没有空余许可证：挂起，把 resolve 放入队列
    // release() 被调用时会从队列取出 resolve，唤醒这个等待者
    await new Promise<void>(r => this.queue.push(r));
  }
  release() {
    this.count++; // 归还许可证
    this.queue.shift()?.(); // 唤醒队头等待者（如有）
  }
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
- 层 1：直接 JSON.parse，失败则继续
- 层 2：用正则提取 ` ```json ``` ` 或 ` ``` ``` ` 代码块，提取内容后 parse
- 层 3：扫描文本中最外层的 `{...}` 或 `[...]`，通过括号计数找到匹配的闭合位置

```ts
function extractJSON(text: string): object | null {
  // ── 层 1：整串直接解析（最快路径，LLM 直接输出合法 JSON 时命中）──
  try {
    return JSON.parse(text.trim());
  } catch {}

  // ── 层 2：提取 Markdown 代码块内容 ──────────────────────────────
  // 匹配 ```json...``` 或 ```...```，捕获组 [1] 为代码块内容
  const codeBlockRegex = /```(?:json)?\s*\n?([\s\S]*?)\n?```/g;
  let match: RegExpExecArray | null;
  while ((match = codeBlockRegex.exec(text)) !== null) {
    try {
      return JSON.parse(match[1].trim()); // 找到第一个能解析的代码块就返回
    } catch {}
  }

  // ── 层 3：括号计数扫描（不用贪婪正则，天然规避 ReDoS） ─────────
  // 原理：从每个 { 或 [ 出发，用计数器追踪嵌套深度，depth=0 时找到闭合位置
  const startChars = new Set(['{', '[']);
  for (let i = 0; i < text.length; i++) {
    if (!startChars.has(text[i])) continue;
    const openChar = text[i];
    const closeChar = openChar === '{' ? '}' : ']';
    let depth = 0;
    let inString = false; // 当前是否在字符串内（字符串内的括号不计入深度）
    let escaped = false;  // 上一个字符是否是反斜杠（处理 \" 转义）

    for (let j = i; j < text.length; j++) {
      const ch = text[j];
      // 处理转义字符：\\ 或 \" 后面的字符不改变状态
      if (escaped) { escaped = false; continue; }
      if (ch === '\\' && inString) { escaped = true; continue; }
      // 双引号切换字符串状态（进入/退出字符串）
      if (ch === '"') { inString = !inString; continue; }
      // 字符串内部的括号不参与深度计算
      if (inString) continue;
      if (ch === openChar) depth++;
      else if (ch === closeChar) {
        depth--;
        if (depth === 0) {
          // 找到与起始括号匹配的闭合括号，尝试解析这个子串
          try {
            return JSON.parse(text.slice(i, j + 1));
          } catch {}
          break; // 这个起始位置的结构无效，从下一个 { 或 [ 重试
        }
      }
    }
  }

  return null;
}
```

**追问回答：**

1. **多个 JSON 块返回哪个？** 当前实现返回第一个成功解析的。实际应视业务场景决定：如果 LLM 输出通常是"分析文字 + 结构化结果"格式，应返回**最后一个**（最终答案往往在末尾）；也可返回所有 JSON 数组，由调用方选择。

2. **ReDoS 风险：** 层 3 我已改用**括号计数扫描**而非正则，时间复杂度严格 O(n)，不存在回溯。层 2 的正则 ` /```(?:json)?\s*\n?([\s\S]*?)\n?```/g ` 中 `[\s\S]*?` 是惰性匹配，配合锚定分隔符 ` ``` `，回溯路径有限，实际风险极低。若超偏执，可在提取前加字符数限制：`if (text.length > 1_000_000) return null`。

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
  private cancelled = false; // 取消标志位，cancel() 设为 true 后下一个 chunk 前检查

  constructor(
    private items: T[],
    private processor: (item: T) => R,
    private options: {
      chunkSize: number;
      onProgress?: (processed: number, total: number) => void;
    }
  ) {}

  async run(): Promise<R[]> {
    const { chunkSize, onProgress } = this.options;
    const total = this.items.length;
    let i = 0;

    while (i < total && !this.cancelled) {
      // 处理一个 chunk（同步，不让出控制权）
      const end = Math.min(i + chunkSize, total);
      for (let j = i; j < end; j++) {
        this.results.push(this.processor(this.items[j]));
      }
      i = end;
      onProgress?.(i, total);
      // ★ 关键：每批结束后主动让出事件循环
      // 让浏览器/Node 有机会处理定时器、I/O、用户输入等其他事件
      await yieldControl();
    }

    // 取消时返回空数组（而非 reject），调用方用 getPartialResults() 拿部分结果
    return this.cancelled ? [] : this.results;
  }

  cancel(): void {
    this.cancelled = true;
  }

  // 返回拷贝，防止外部意外修改内部 results
  getPartialResults(): R[] {
    return [...this.results];
  }
}

// 跨平台让出事件循环的工具函数
// Node.js 用 setImmediate（在 check 阶段执行，比 setTimeout 更快更稳定）
// 浏览器用 setTimeout(0)（没有 setImmediate）
function yieldControl(): Promise<void> {
  return new Promise(resolve => {
    if (typeof setImmediate !== 'undefined') {
      setImmediate(resolve); // Node.js 环境：I/O 回调之后、下一帧之前执行
    } else {
      setTimeout(resolve, 0); // 浏览器环境：进入 task 队列，让渲染和交互事件先执行
    }
  });
}
```

**追问回答：**

1. **三者区别：**
   - `process.nextTick`：在当前事件循环末尾、I/O 回调之前执行，**不让出**到 I/O 事件，会阻塞 I/O 处理，不适合此场景。
   - `setTimeout(fn, 0)`：进入 timers 阶段，至少延迟 1ms（受系统调度影响），让出控制权彻底，浏览器中是唯一选择。
   - `setImmediate`：在 check 阶段执行（I/O 回调之后），比 `setTimeout` 更快更稳定，**最适合 Node.js 场景**。

2. **processor 是异步函数时：** 将 `processor: (item: T) => R` 改为 `(item: T) => Promise<R>`，批处理时用 `Promise.all` 并行执行一批：
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
- 状态机三个状态：`READ_TYPE`（读 1 字节类型）、`READ_LENGTH`（读 4 字节长度）、`READ_PAYLOAD`（读 N 字节载荷）
- 维护 `buffer` 积累未解析数据，每次 feed 后循环尝试解析直到数据不足

```ts
import { EventEmitter } from 'events';

// 状态机三个状态，对应 TLV 协议的三个字段
type ParserState = 'READ_TYPE' | 'READ_LENGTH' | 'READ_PAYLOAD';

class TLVParser extends EventEmitter {
  // buffer 积累所有已收到但未解析完的字节（处理半包的核心）
  private buffer: Buffer = Buffer.alloc(0);
  // 当前解析状态，初始等待读取包类型字段
  private state: ParserState = 'READ_TYPE';
  private currentType = 0;     // 暂存当前包的 type 字段（等到 payload 读完才一起 emit）
  private expectedLength = 0;  // 暂存当前包声明的 payload 长度

  feed(chunk: Buffer): void {
    // 将新到达的数据拼接到缓冲区末尾（处理跨包场景：包头在上个 chunk，包体在这个）
    this.buffer = Buffer.concat([this.buffer, chunk]);
    this.parse(); // 每次喂入数据后都尝试解析（可能已能凑出完整包）
  }

  private parse(): void {
    // 循环尝试解析，直到缓冲区数据不足以完成当前状态的读取
    while (true) {
      if (this.state === 'READ_TYPE') {
        // 需要 1 字节，不足则等待下次 feed
        if (this.buffer.length < 1) break;
        this.currentType = this.buffer[0]; // 读取第 1 个字节作为类型
        this.buffer = this.buffer.slice(1); // 消费掉这 1 个字节
        this.state = 'READ_LENGTH'; // 状态转移：去读长度字段

      } else if (this.state === 'READ_LENGTH') {
        // 需要 4 字节（big-endian uint32），不足则等待
        if (this.buffer.length < 4) break;
        this.expectedLength = this.buffer.readUInt32BE(0); // 大端序读取 4 字节整数
        // 防超大包攻击：恶意客户端可能发送 length=2^32-1 来耗尽服务器内存
        const MAX_PAYLOAD_SIZE = 64 * 1024 * 1024; // 64MB
        if (this.expectedLength > MAX_PAYLOAD_SIZE) {
          this.emit('error', new Error(`非法包长度: ${this.expectedLength}`));
          this.destroy();
          return;
        }
        this.buffer = this.buffer.slice(4); // 消费掉这 4 个字节
        this.state = 'READ_PAYLOAD'; // 状态转移：去读 payload

      } else if (this.state === 'READ_PAYLOAD') {
        // 需要 expectedLength 个字节，不足则等待（典型的半包场景）
        if (this.buffer.length < this.expectedLength) break;
        // 截取恰好 expectedLength 字节作为 payload
        const payload = this.buffer.slice(0, this.expectedLength);
        // 消费 payload 后，剩余字节可能是下一个包的开头（粘包场景）
        this.buffer = this.buffer.slice(this.expectedLength);
        this.emit('message', { type: this.currentType, payload }); // 一个完整包解析完毕
        this.state = 'READ_TYPE'; // 状态归位，准备解析下一个包
      }
    }
  }

  // 连接断开时调用，清理残留数据和监听器
  destroy(): void {
    // 有残留表示收到了一个不完整的包，上报错误供业务层感知
    if (this.buffer.length > 0) {
      this.emit('error', new Error(`连接断开，残留 ${this.buffer.length} 字节未完成包`));
    }
    this.buffer = Buffer.alloc(0);
    this.state = 'READ_TYPE';
    this.removeAllListeners(); // 避免内存泄漏
  }
}
```

**追问回答：**

1. **半包清理：** 连接断开时调用 `destroy()`，检查 buffer 是否有残留数据，有则 emit `error` 事件上报（包含残留字节数和当前解析状态），然后清空 buffer 和监听器，防止内存泄漏。

2. **防内存耗尽攻击：** 在 `READ_LENGTH` 状态解析到 `expectedLength` 后，立即校验是否超过最大值（如 64MB），超过则 emit error 并关闭连接（代码已内置此逻辑）。

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
interface SSEEvent {
  event?: string;
  data: string;
  id?: string;
}

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

  constructor(url: string, options?: {
    headers?: Record<string, string>;
    maxRetries?: number;
    retryDelay?: number;
    onMessage: (event: SSEEvent) => void;
    onError?: (err: Error) => void;
    onReconnect?: (attempt: number) => void;
  }) {
    this.url = url;
    this.headers = options?.headers ?? {};
    this.maxRetries = options?.maxRetries ?? 3;
    this.retryDelay = options?.retryDelay ?? 1000;
    this.onMessage = options!.onMessage;
    this.onError = options?.onError;
    this.onReconnect = options?.onReconnect;
  }

  connect(): void {
    this.closed = false;
    this.retryCount = 0;
    this.doConnect();
  }

  close(): void {
    this.closed = true;
    this.abortController?.abort();
  }

  private async doConnect(): Promise<void> {
    this.abortController = new AbortController();
    const headers: Record<string, string> = {
      Accept: 'text/event-stream',
      ...this.headers,
    };
    if (this.lastEventId) {
      headers['Last-Event-ID'] = this.lastEventId;
    }

    try {
      const response = await fetch(this.url, {
        headers,
        signal: this.abortController.signal,
      });

      if (!response.ok || !response.body) {
        throw new Error(`HTTP ${response.status}`);
      }

      await this.readStream(response.body);
      // 流正常结束（服务端主动关闭），不重连
    } catch (err: any) {
      if (this.closed) return; // 手动关闭，不重连
      this.onError?.(err instanceof Error ? err : new Error(String(err)));
      await this.scheduleReconnect();
    }
  }

  private async readStream(body: ReadableStream<Uint8Array>): Promise<void> {
    const reader = body.getReader();
    const decoder = new TextDecoder();
    let buffer = ''; // 跨 chunk 的文本缓冲，处理一个 SSE 事件被拆分在多个网络包里的情况

    try {
      while (true) {
        const { done, value } = await reader.read();
        if (done) break; // 服务端主动关闭：done=true，不会抛异常（区别于网络错误）
        // { stream: true } 告诉 TextDecoder 当前解码可能不完整（多字节字符跨 chunk 时不出错）
        buffer += decoder.decode(value, { stream: true });
        // SSE 协议规定：事件之间用空行（\n\n）分隔
        const parts = buffer.split('\n\n');
        // pop() 取出最后一段：它可能是下一个事件的开头（尚未收到完整事件），留在 buffer 中等待
        buffer = parts.pop()!;
        for (const part of parts) {
          this.parseEvent(part); // 完整事件块才解析
        }
      }
    } finally {
      reader.releaseLock(); // 必须释放，否则后续无法重新读取该流
    }
  }

  private parseEvent(raw: string): void {
    const lines = raw.split('\n');
    let event: SSEEvent = { data: '' };
    const dataParts: string[] = []; // SSE 允许多行 data: 字段，最终拼接为一个字符串

    for (const line of lines) {
      if (line.startsWith('event:')) {
        event.event = line.slice(6).trim();
      } else if (line.startsWith('data:')) {
        dataParts.push(line.slice(5).trim()); // 去掉 "data:" 前缀和首个空格
      } else if (line.startsWith('id:')) {
        event.id = line.slice(3).trim();
        this.lastEventId = event.id; // 记录最后事件 ID，断线重连时通过 Last-Event-ID 头续传
      }
    }

    event.data = dataParts.join('\n'); // 多行 data: 按换行符拼接
    if (event.data) this.onMessage(event); // 没有 data 字段的事件（如心跳注释行）不触发回调
  }

  private async scheduleReconnect(): Promise<void> {
    if (this.retryCount >= this.maxRetries) return; // 超过最大重试次数，放弃
    this.retryCount++;
    // 指数退避：第1次等 delay×1，第2次等 delay×2，第3次等 delay×4...
    const delay = this.retryDelay * Math.pow(2, this.retryCount - 1);
    this.onReconnect?.(this.retryCount);
    await new Promise(r => setTimeout(r, delay));
    if (!this.closed) this.doConnect(); // 等待期间可能被 close() 手动关闭，再次检查
  }
}
```

**追问回答：**

1. **区分服务端主动关闭 vs 网络中断：** 服务端主动关闭时，`reader.read()` 返回 `{ done: true, value: undefined }`，流正常结束，不抛异常。网络中断时，`reader.read()` 或 `fetch` 本身会抛出 `TypeError: Failed to fetch` 或 `NetworkError`。因此在 `readStream` 正常 return 时不重连，在 `catch` 块中才重连。

2. **二进制数据（base64）：** SSE 本身是文本协议，base64 编码的数据作为普通字符串在 `data:` 字段传输。解析时在 `onMessage` 回调中检测 `event.event === 'binary'`，对 `event.data` 做 `atob(event.data)` 解码，或进一步转为 `Uint8Array`：
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
  // WeakSet：不阻止 GC，元素被从 DOM 移除后自动释放，无需手动清理
  private observedElements = new WeakSet<Element>();

  constructor(options: {
    report: (event: object) => void;
    root?: Element;
    exposeThreshold?: number;
  }) {
    this.report = options.report;
    this.root = options.root ?? document.documentElement;
    this.exposeThreshold = options.exposeThreshold ?? 0.5;
  }

  init(): void {
    // ── 点击埋点：事件委托，只在根节点绑定一个监听器 ──────────────
    // 好处：无论多少个按钮，只有一个 listener，动态新增元素也自动生效
    this.clickHandler = (e: Event) => {
      // closest() 向上查找最近的带 data-track-click 属性的祖先（或自身）
      // 处理"点击图标但埋点在按钮上"的场景
      const target = (e.target as Element).closest('[data-track-click]');
      if (!target) return;
      const raw = target.getAttribute('data-track-click');
      try {
        this.report(JSON.parse(raw!));
      } catch {
        // 容错：单个元素 JSON 配置错误不影响其他元素
        console.warn('[TrackingSystem] 非法 data-track-click JSON:', raw);
      }
    };
    this.root.addEventListener('click', this.clickHandler);

    // ── 曝光埋点：IntersectionObserver 监听元素进入视口 ──────────
    this.observer = new IntersectionObserver(
      (entries) => {
        // entries 是批量变化的元素列表（一次回调可能有多个元素状态变化）
        for (const entry of entries) {
          if (!entry.isIntersecting) continue; // 只处理"进入视口"事件，忽略"离开"
          const el = entry.target;
          const raw = el.getAttribute('data-track-expose');
          // once 默认 true；只有明确写 data-track-expose-once="false" 时才允许重复上报
          const once = el.getAttribute('data-track-expose-once') !== 'false';
          try {
            this.report(JSON.parse(raw!));
          } catch {
            console.warn('[TrackingSystem] 非法 data-track-expose JSON:', raw);
          }
          if (once) {
            // 上报后停止监听，避免用户滚动回来时重复触发
            this.observer?.unobserve(el);
          }
        }
      },
      { threshold: this.exposeThreshold } // 元素可见比例达到阈值才触发
    );

    // 扫描 DOM 中已有的曝光元素并注册（对页面初始加载时已存在的元素生效）
    this.root.querySelectorAll('[data-track-expose]').forEach(el => {
      this.observe(el);
    });
  }

  observe(element: Element): void {
    if (!this.observer) return;
    // 防止重复注册：同一元素 observe 两次会导致回调触发两次
    if (this.observedElements.has(element)) return;
    this.observedElements.add(element);
    this.observer.observe(element);
  }

  destroy(): void {
    // 移除点击监听器（必须传入同一个函数引用，因此 clickHandler 需要保存为成员变量）
    if (this.clickHandler) {
      this.root.removeEventListener('click', this.clickHandler);
      this.clickHandler = null;
    }
    // disconnect() 停止所有已注册元素的监听，比逐一 unobserve 更彻底
    this.observer?.disconnect();
    this.observer = null;
  }
}
```

**追问回答：**

1. **1000 个元素用一个 IntersectionObserver：** 用**一个** `IntersectionObserver`，将所有元素通过 `observe()` 注册到它上面。多个 Observer 实例本身就是开销。`IntersectionObserver` 的设计目标就是批量监听，回调参数 `entries` 是数组，一次回调可以处理多个元素状态变化，性能远优于每个元素一个实例。

2. **被遮挡误触发问题：** `IntersectionObserver` 只计算目标元素与根视口的几何相交，**无法感知 z-index 遮挡**。解决方案：
   - 在回调中用 `document.elementFromPoint(entry.boundingClientRect.x + w/2, entry.boundingClientRect.y + h/2)` 检测命中元素是否是目标元素本身或其子元素
   - 若命中元素不是目标，则判定为被遮挡，不上报
   - 注意 `elementFromPoint` 有一定性能开销，只在 `isIntersecting` 为 true 时调用

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
  let chunkSize = options.chunkSize ?? 50; // 可变：运行中自适应调整
  const total = items.length;
  let rendered = 0;

  const renderChunk = (): Promise<void> =>
    // 包装成 Promise 是为了配合 await，让每批渲染都在 rAF 回调中执行
    new Promise(resolve => {
      // requestAnimationFrame：在浏览器下一次重绘前执行
      // 每帧约 16ms 预算，在帧开始时执行 DOM 操作可以避免 layout thrashing
      requestAnimationFrame(() => {
        const start = rendered;
        const end = Math.min(start + chunkSize, total);
        const startTime = performance.now(); // 记录开始时间，用于后续自适应调整

        // DocumentFragment：内存中的轻量级容器
        // 好处：批量 DOM 操作在 fragment 上完成，最后一次性 append 到真实 DOM
        // 只触发一次 reflow，而不是每个节点一次
        const fragment = document.createDocumentFragment();
        for (let i = start; i < end; i++) {
          fragment.appendChild(renderItem(items[i], i));
        }
        container.appendChild(fragment); // 单次 DOM 插入，触发一次 layout
        rendered = end;

        // ── 自适应分片大小：根据实际耗时动态调整 ──────────────────
        const elapsed = performance.now() - startTime;
        if (elapsed > 16) {
          // 超过一帧预算（16ms），减小 chunk，避免下一帧卡顿，最小保留 10 个
          chunkSize = Math.max(10, Math.floor(chunkSize * 0.7));
        } else if (elapsed < 8) {
          // 耗时很短（不到半帧），适当增大 chunk，加快渲染速度，最大 500
          chunkSize = Math.min(chunkSize * 1.3, 500);
        }

        onProgress?.(rendered, total);
        resolve(); // 通知外层 while 循环这一批已完成，可以继续下一批
      });
    });

  // 每轮 await renderChunk() 会暂停到下一帧，给浏览器渲染和用户交互留出空间
  while (rendered < total) {
    await renderChunk();
  }
}
```

**追问回答：**

1. **rAF vs rIC：**
   - `requestAnimationFrame`：在**每一帧绘制前**执行，保证 16ms 内不超时，适合需要视觉反馈的 DOM 操作（渲染列表项时用户能看到渐进填充效果）。
   - `requestIdleCallback`：在浏览器**空闲时**执行，有 `deadline.timeRemaining()` 控制预算，适合非紧急后台任务（预计算、数据预加载）。
   - 此场景优先用 **rAF**：DOM 插入需要与渲染帧同步，且用户希望列表尽快出现，rIC 的调度是"有空才做"，延迟不可预测。

2. **虚拟滚动核心思路：**
   - 核心数据：`itemHeight`（固定行高）或 `itemHeights[]`（可变行高），`scrollTop`，容器 `clientHeight`
   - 计算：`startIndex = Math.floor(scrollTop / itemHeight)`，`endIndex = startIndex + Math.ceil(clientHeight / itemHeight) + buffer`
   - 只渲染 `[startIndex, endIndex]` 范围内的节点，容器用一个占位 div 撑开总高度 `total * itemHeight`
   - 可见节点用 `position: absolute; top: index * itemHeight` 定位
   - 监听 `scroll` 事件，更新 startIndex/endIndex，复用或重建 DOM 节点

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

// 纯函数 reducer：所有状态转换集中于此，状态变化可预测、可测试
function reducer(state: SSEState, action: Action): SSEState {
  switch (action.type) {
    case 'CONNECTING':
      // 重置所有历史数据，准备新的流式请求
      return { status: 'connecting', messages: [], fullText: '', error: null };
    case 'APPEND': {
      const messages = [...state.messages, action.payload];
      // fullText 始终等于 messages.join('')，方便外部直接使用完整文本
      return { ...state, status: 'streaming', messages, fullText: messages.join('') };
    }
    case 'DONE':
      return { ...state, status: 'done' };
    case 'ERROR':
      return { ...state, status: 'error', error: action.payload };
    case 'RESET':
      return { status: 'idle', messages: [], fullText: '', error: null };
    default:
      return state;
  }
}

const initialState: SSEState = { status: 'idle', messages: [], fullText: '', error: null };

function useSSE(url: string | null, options?: {
  headers?: Record<string, string>;
  onDone?: (fullText: string) => void;
}) {
  const [state, dispatch] = useReducer(reducer, initialState);
  const abortRef = useRef<AbortController | null>(null); // 用于取消 fetch 请求
  // pendingRef：token 缓冲区，收集 16ms 内到达的所有 token，批量 dispatch 一次
  // 用 ref 而非 state 是因为不需要触发重渲染，只是暂存
  const pendingRef = useRef<string[]>([]);
  const flushTimerRef = useRef<ReturnType<typeof setTimeout> | null>(null); // 批量刷新定时器
  // stateRef：在异步回调中读取最新 state（直接闭包捕获的 state 会是旧值）
  const stateRef = useRef(state);
  stateRef.current = state; // 每次渲染时同步最新 state 到 ref

  // 每 16ms 把 pendingRef 中的所有 token 一次性 dispatch 出去
  // 解决"每秒 100 个 token 触发 100 次渲染"的性能问题
  const scheduleFlush = useCallback(() => {
    if (flushTimerRef.current) return; // 已有定时器在等待中，不重复设置
    flushTimerRef.current = setTimeout(() => {
      flushTimerRef.current = null;
      const tokens = pendingRef.current.splice(0); // 清空缓冲区并取出所有 token
      if (tokens.length > 0) {
        // 注意：这里每个 token 仍然 dispatch 一次，但集中在一个 task 中执行
        // React 18 会自动批处理同一事件循环中的多次 setState（automatic batching）
        tokens.forEach(t => dispatch({ type: 'APPEND', payload: t }));
      }
    }, 16); // 约一帧的时间窗口
  }, []);

  const stop = useCallback(() => {
    abortRef.current?.abort(); // 触发 fetch 的 AbortError，中断正在进行的流读取
    abortRef.current = null;
    if (flushTimerRef.current) {
      clearTimeout(flushTimerRef.current); // 清除未触发的批量刷新定时器
      flushTimerRef.current = null;
    }
  }, []);

  const start = useCallback(async () => {
    if (!url) return;
    stop(); // 确保同一时刻只有一个活跃连接
    dispatch({ type: 'CONNECTING' });
    const controller = new AbortController();
    abortRef.current = controller; // 保存引用，以便 stop() 时调用 abort()

    try {
      const response = await fetch(url, {
        headers: { Accept: 'text/event-stream', ...(options?.headers ?? {}) },
        signal: controller.signal, // 绑定 AbortSignal，abort() 后此 fetch 会抛 AbortError
      });
      if (!response.body) throw new Error('No response body');

      const reader = response.body.getReader();
      const decoder = new TextDecoder();
      let buffer = '';

      while (true) {
        const { done, value } = await reader.read();
        if (done) break; // 服务端正常结束，不触发 catch
        buffer += decoder.decode(value, { stream: true });
        const parts = buffer.split('\n\n'); // 按 SSE 事件分隔符拆分
        buffer = parts.pop()!; // 保留尾部不完整的事件片段
        for (const part of parts) {
          for (const line of part.split('\n')) {
            if (line.startsWith('data:')) {
              pendingRef.current.push(line.slice(5).trim()); // 存入缓冲区
              scheduleFlush(); // 启动批量刷新定时器（幂等：已有定时器则跳过）
            }
          }
        }
      }
      dispatch({ type: 'DONE' });
      // 通过 stateRef 读取最新 fullText（直接用 state.fullText 是旧的闭包值）
      options?.onDone?.(stateRef.current.fullText);
    } catch (err: any) {
      if (err.name === 'AbortError') return; // 手动 stop() 导致的，不是真正的错误
      dispatch({ type: 'ERROR', payload: err instanceof Error ? err : new Error(String(err)) });
    }
  }, [url, stop, scheduleFlush, options]);

  const reset = useCallback(() => {
    stop();
    dispatch({ type: 'RESET' });
  }, [stop]);

  // url 变化时自动关闭旧连接（cleanup 在下次 effect 运行前执行）
  useEffect(() => {
    return () => stop();
  }, [url, stop]);

  // 组件卸载时清理，防止 setState 在已卸载组件上调用
  useEffect(() => {
    return () => stop();
  }, [stop]);

  return { ...state, start, stop, reset };
}
```

**追问回答：**

1. **每秒 100 token 都触发 setState 的问题：** 会导致每秒 100 次渲染，React 并发模式下可能还好，但旧版同步渲染会明显卡顿。解决方案：用 `pendingRef` 缓冲 token，用 `setTimeout(16ms)` 批量合并，一帧内多个 token 只触发一次 `dispatch`（如代码中 `scheduleFlush` 所示）。

2. **useReducer 重构优势：** 状态转换集中在 `reducer` 中，状态不合法的转换（如从 `done` 直接到 `streaming`）在 reducer 内部可以拦截；测试时只需测 reducer 纯函数，不需要渲染组件；并发模式下 React 可能多次调用 reducer，纯函数保证幂等。

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
  private idle: T[] = [];             // 空闲连接栈（pop 取最近归还的，利用 OS 缓存）
  // FIFO 等待队列：每个等待者携带自己的 resolve/reject 和超时定时器
  private waiting: Array<{
    resolve: (conn: T) => void;
    reject: (err: Error) => void;
    timer: ReturnType<typeof setTimeout>;
  }> = [];
  private totalCount = 0; // 池中所有连接数（idle + 正在使用中的）

  constructor(private options: {
    create: () => Promise<T>;
    destroy: (conn: T) => Promise<void>;
    validate?: (conn: T) => boolean;
    maxSize: number;
    acquireTimeout: number;
  }) {}

  async acquire(): Promise<T> {
    // ── 优先复用空闲连接（pop 而非 shift，O(1) 且利用连接的"热度"）──
    while (this.idle.length > 0) {
      const conn = this.idle.pop()!;
      if (!this.options.validate || this.options.validate(conn)) {
        return conn; // 验证通过，直接返回
      }
      // 验证失败（连接已断开/过期），销毁并继续找下一个空闲连接
      this.totalCount--;
      await this.options.destroy(conn);
    }

    // ── 未达到上限：新建连接 ────────────────────────────────────────
    if (this.totalCount < this.options.maxSize) {
      this.totalCount++; // 先占位，防止并发时超出 maxSize
      try {
        return await this.options.create();
      } catch (err) {
        this.totalCount--; // 创建失败，归还占位
        throw err;
      }
    }

    // ── 已达上限：进入 FIFO 等待队列，带超时 ──────────────────────
    return new Promise<T>((resolve, reject) => {
      const timer = setTimeout(() => {
        // 超时触发：从等待队列中移除自己（通过 timer 引用找到自己的位置）
        const idx = this.waiting.findIndex(w => w.timer === timer);
        if (idx !== -1) this.waiting.splice(idx, 1);
        reject(new Error(`获取连接超时（${this.options.acquireTimeout}ms）`));
      }, this.options.acquireTimeout);

      this.waiting.push({ resolve, reject, timer });
    });
  }

  async release(conn: T): Promise<void> {
    // ── 有等待者：跳过空闲池，直接转交给队头（FIFO） ─────────────
    if (this.waiting.length > 0) {
      const waiter = this.waiting.shift()!; // 取队头等待者
      clearTimeout(waiter.timer); // 取消超时，连接已就位

      if (!this.options.validate || this.options.validate(conn)) {
        waiter.resolve(conn); // 连接可用，直接给它
        return;
      }
      // 连接已损坏：销毁旧连接，为等待者新建一个
      this.totalCount--;
      await this.options.destroy(conn);
      this.totalCount++;
      try {
        waiter.resolve(await this.options.create());
      } catch (err) {
        this.totalCount--;
        waiter.reject(err as Error);
      }
      return;
    }

    // ── 无等待者：验证后放回空闲池 ──────────────────────────────────
    if (!this.options.validate || this.options.validate(conn)) {
      this.idle.push(conn);
    } else {
      // 验证失败，销毁（不放回池中），totalCount 减少
      this.totalCount--;
      await this.options.destroy(conn);
    }
  }

  // 主动销毁损坏连接（超时后连接不归还，调用此方法直接销毁）
  async destroy(conn: T): Promise<void> {
    this.totalCount--;
    await this.options.destroy(conn);
  }

  getStats() {
    return {
      total: this.totalCount,
      idle: this.idle.length,
      waiting: this.waiting.length,
    };
  }
}
```

**追问回答：**

1. **连接预热：** 在构造函数末尾（或提供 `warmup(minSize: number)` 方法）并发创建 `minSize` 个连接：
```ts
async warmup(minSize: number): Promise<void> {
  const tasks = Array.from({ length: minSize }, () => this.acquire().then(conn => this.release(conn)));
  await Promise.allSettled(tasks);
}
```
这样服务启动时连接池已有可用连接，首批请求无需等待 TCP 握手。

2. **create 过程中连接池被销毁：** 在 `create()` 前记录一个 `destroyed` 标志，create 完成后检查若已 destroyed，立即调用 `destroy(conn)` 释放掉这个刚建的连接，不放入池中：
```ts
if (this.destroyed) {
  await this.options.destroy(conn);
  this.totalCount--;
  throw new Error('连接池已销毁');
}
```
