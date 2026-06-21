---
layout: post
title: "编程题：系统设计（含答案）"
date: 2025-01-03
categories: 面试
tags: [系统设计, TypeScript, Node.js]
---

> 考察系统抽象能力、工程严谨性、对并发/原子性/幂等性的理解。
> 难度标注：⭐⭐⭐ 进阶 ｜ ⭐⭐⭐⭐ 挑战

---

### D1. 设计熔断器（Circuit Breaker） ⭐⭐⭐

**背景：** 简历中实现了"熔断层，熔断采用惰性状态转移（由下次请求触发而非定时轮询）"。

**题目：**

严格按照以下状态机实现熔断器：

```
CLOSED ──（错误率超阈值）──→ OPEN
OPEN   ──（下次请求触发）──→ HALF_OPEN
HALF_OPEN ──（探测成功）──→ CLOSED
HALF_OPEN ──（探测失败）──→ OPEN
```

```ts
class CircuitBreaker {
  constructor(options: {
    failureThreshold: number;    // 触发 OPEN 的失败次数阈值
    successThreshold: number;    // HALF_OPEN 中连续成功次数阈值
    timeout: number;             // OPEN 状态持续时长 ms
    onStateChange?: (from: CircuitState, to: CircuitState) => void;
  }) {}
  async execute<T>(fn: () => Promise<T>): Promise<T> {}
  getState(): CircuitState {}
}
```

**要求：**
1. OPEN 状态时，`execute()` 直接 reject，不调用 `fn`（快速失败）
2. 超过 `timeout` 后，第一个进入的请求触发惰性状态转移到 HALF_OPEN
3. HALF_OPEN 中只允许一个探测请求通过，其余快速失败
4. 探测成功 `successThreshold` 次后转回 CLOSED；任意失败立即回退 OPEN

**示例：**
```ts
const breaker = new CircuitBreaker({ failureThreshold: 3, successThreshold: 2, timeout: 5000 });

for (let i = 0; i < 3; i++) {
  await breaker.execute(() => Promise.reject(new Error('fail'))).catch(() => {});
}
console.log(breaker.getState()); // 'OPEN'

await sleep(5001);
await breaker.execute(() => Promise.resolve('ok'));
console.log(breaker.getState()); // 'HALF_OPEN'（还差一次成功）
```

**追问：**
1. 在高并发下，OPEN → HALF_OPEN 的"只允许一个探测请求"如何实现协程安全？
2. 如果要支持"双模型热备"（主模型熔断后切备用模型），外层如何封装？

---

```ts
type CircuitState = 'CLOSED' | 'OPEN' | 'HALF_OPEN';

class CircuitBreaker {
  private state: CircuitState = 'CLOSED'; // 初始为正常状态
  private failureCount = 0;  // CLOSED 中累计失败次数
  private successCount = 0;  // HALF_OPEN 中累计成功次数
  private openedAt = 0;      // 记录进入 OPEN 的时间，用于计算是否到了探测时机
  // HALF_OPEN 中是否已有探测请求在飞行中
  // Node.js 单线程：检查和赋值之间没有其他代码执行，天然无竞争
  private probing = false;

  constructor(private options: {
    failureThreshold: number;
    successThreshold: number;
    timeout: number;
    onStateChange?: (from: CircuitState, to: CircuitState) => void;
  }) {}

  private transition(to: CircuitState): void {
    const from = this.state;
    this.state = to;
    this.options.onStateChange?.(from, to); // 通知外部观察者（用于监控、日志）
  }

  async execute<T>(fn: () => Promise<T>): Promise<T> {
    // ── OPEN 状态：快速失败，但检查是否到了探测时机 ─────────────
    // "惰性转移"：不用定时器，而是在下一次请求到来时检查时间差
    if (this.state === 'OPEN') {
      if (Date.now() - this.openedAt >= this.options.timeout) {
        // 超过冷却时间，允许一次探测请求通过（转移到 HALF_OPEN）
        this.transition('HALF_OPEN');
        this.successCount = 0;
        this.probing = false;
      } else {
        // 还在冷却期，直接拒绝（不调用 fn，避免雪崩）
        throw new Error('CircuitBreaker: OPEN，快速失败');
      }
    }

    // ── HALF_OPEN 状态：只允许一个探测请求，其余快速失败 ────────
    if (this.state === 'HALF_OPEN') {
      if (this.probing) {
        // 已有一个探测请求在进行中，其他请求继续快速失败
        throw new Error('CircuitBreaker: HALF_OPEN 探测中，快速失败');
      }
      this.probing = true; // 标记探测已开始（同步操作，Node.js 单线程安全）
      try {
        const result = await fn();
        this.successCount++;
        if (this.successCount >= this.options.successThreshold) {
          // 连续成功达阈值，服务已恢复，回到正常状态
          this.failureCount = 0;
          this.transition('CLOSED');
        } else {
          // 还差几次成功才能恢复，允许下次探测继续
          this.probing = false;
        }
        return result;
      } catch (err) {
        // 探测失败：服务还没好，立刻回退到 OPEN 重新计时
        this.openedAt = Date.now();
        this.transition('OPEN');
        throw err;
      }
    }

    // ── CLOSED 状态：正常执行，统计失败次数 ─────────────────────
    try {
      const result = await fn();
      this.failureCount = 0; // 成功后重置，避免历史失败影响后续判断
      return result;
    } catch (err) {
      this.failureCount++;
      if (this.failureCount >= this.options.failureThreshold) {
        // 失败次数超阈值，触发熔断
        this.openedAt = Date.now();
        this.transition('OPEN');
      }
      throw err; // 无论是否触发熔断，错误都要继续向上抛
    }
  }

  getState(): CircuitState {
    return this.state;
  }
}
```

**追问回答：**

1. **高并发下 HALF_OPEN 只允许一个探测请求的协程安全：** Node.js 是单线程事件循环，`this.probing = true` 的赋值是同步操作，在检查和赋值之间不会有其他协程插入（因为没有 `await`），所以上述实现已经是安全的。若在多线程环境（如 Java），需要用 `AtomicBoolean` 的 `compareAndSet(false, true)` 实现 CAS 操作。

2. **双模型热备封装：**

```ts
class DualModelExecutor {
  constructor(
    private primary: CircuitBreaker,
    private fallback: CircuitBreaker,
    private primaryFn: () => Promise<any>,
    private fallbackFn: () => Promise<any>
  ) {}

  async execute(): Promise<any> {
    if (this.primary.getState() !== 'OPEN') {
      try {
        return await this.primary.execute(this.primaryFn);
      } catch {}
    }
    return this.fallback.execute(this.fallbackFn);
  }
}
```
主模型熔断时自动切到备用，对上层业务完全透明。

---

### D2. 实现幂等安装引擎 ⭐⭐⭐⭐

**背景：** 简历中实现了"声明式安装引擎：工具权限配置去重合并、配置文件幂等追加"。

**题目：**

```ts
interface ConfigPatch {
  skillId: string;
  arrayAppend?: { [jsonPath: string]: any[] };    // 追加到 JSON 数组的项
  objectMerge?: { [jsonPath: string]: object };   // 深度合并的对象
}

class IdempotentInstaller {
  constructor(configPath: string) {}
  async install(patch: ConfigPatch): Promise<void> {}
  async uninstall(skillId: string): Promise<void> {}
}
```

**要求：**
1. **幂等性**：同一个 `skillId` 的 patch 多次 install，配置文件内容保持不变
2. **去重合并**：多个 skill 的 `arrayAppend` 合并时去重，以对象的 `id` 或 `name` 字段为 key
3. **可追溯**：需要记录每个配置项由哪个 skillId 贡献（以便 uninstall）
4. **原子写入**：先写临时文件再重命名，防止写入过程中崩溃导致配置损坏

**追问：**
1. 如果两个 Skill 都向同一个 `jsonPath` 贡献了 key 相同但 value 不同的对象，冲突策略是什么？如何让用户感知并介入？
2. `uninstall` 时如何确保只移除该 skill 贡献的内容，不误删其他 skill 的相同内容？

---

```ts
import * as fs from 'fs/promises';
import * as path from 'path';

interface ConfigPatch {
  skillId: string;
  arrayAppend?: { [jsonPath: string]: any[] };
  objectMerge?: { [jsonPath: string]: object };
}

// meta.json 结构：跟踪每个配置路径下，哪些 key 是由哪个 skill 贡献的
// 这是 uninstall 时精准移除的依据，避免误删其他 skill 的配置
interface ConfigMeta {
  contributions: {
    [jsonPath: string]: {
      arrays: { [dedupeKey: string]: { value: any; skillId: string } }; // 数组项贡献者
      objects: { [skillId: string]: object }; // 对象合并贡献者
    };
  };
}

class IdempotentInstaller {
  private configPath: string;
  private metaPath: string; // 伴随配置文件的元数据文件，记录各 skill 的贡献

  constructor(configPath: string) {
    this.configPath = configPath;
    this.metaPath = configPath + '.meta.json'; // 例如 config.json → config.json.meta.json
  }

  private async readJSON(filePath: string): Promise<any> {
    try {
      return JSON.parse(await fs.readFile(filePath, 'utf-8'));
    } catch {
      return {}; // 文件不存在时返回空对象（首次安装时正常）
    }
  }

  // 原子写入：先写临时文件，再用 rename 替换目标文件
  // rename 在同一文件系统上是原子操作，崩溃后不会留下半写状态
  private async atomicWrite(filePath: string, data: any): Promise<void> {
    const tmp = filePath + '.tmp.' + Date.now(); // 临时文件名加时间戳避免冲突
    await fs.writeFile(tmp, JSON.stringify(data, null, 2), 'utf-8');
    await fs.rename(tmp, filePath); // 原子替换，失败则 tmp 留在磁盘，不损坏原文件
  }

  // 按点分路径读取嵌套对象，例如 "permissions.allowed" → obj.permissions.allowed
  private getByPath(obj: any, jsonPath: string): any {
    return jsonPath.split('.').reduce((o, k) => o?.[k], obj);
  }

  // 按点分路径写入嵌套对象（中间层不存在时自动创建）
  private setByPath(obj: any, jsonPath: string, value: any): void {
    const keys = jsonPath.split('.');
    let cur = obj;
    for (let i = 0; i < keys.length - 1; i++) {
      if (!cur[keys[i]]) cur[keys[i]] = {};
      cur = cur[keys[i]];
    }
    cur[keys[keys.length - 1]] = value;
  }

  // 去重 key：优先取 id 字段，其次 name，最后降级为整体序列化（兜底）
  private getDedupeKey(item: any): string {
    return item?.id ?? item?.name ?? JSON.stringify(item);
  }

  async install(patch: ConfigPatch): Promise<void> {
    const [config, meta] = await Promise.all([
      this.readJSON(this.configPath),
      this.readJSON(this.metaPath) as Promise<ConfigMeta>,
    ]);
    if (!meta.contributions) meta.contributions = {};

    // 幂等性保证：重复安装时先清除旧的贡献，再重新写入
    // 等价于 uninstall 后再 install，确保不会产生重复条目
    await this.removeContributions(config, meta, patch.skillId);

    // ── arrayAppend：去重合并到目标数组 ────────────────────────────
    if (patch.arrayAppend) {
      for (const [jsonPath, items] of Object.entries(patch.arrayAppend)) {
        if (!meta.contributions[jsonPath]) {
          meta.contributions[jsonPath] = { arrays: {}, objects: {} };
        }
        const pathMeta = meta.contributions[jsonPath];
        let arr: any[] = this.getByPath(config, jsonPath) ?? [];

        for (const item of items) {
          const key = this.getDedupeKey(item);
          // 冲突检测：该 key 已被其他 skill 贡献 → 跳过（先到先得策略）
          if (pathMeta.arrays[key] && pathMeta.arrays[key].skillId !== patch.skillId) {
            continue;
          }
          // 记录贡献者（供 uninstall 时精准移除）
          pathMeta.arrays[key] = { value: item, skillId: patch.skillId };
          // 数组中去重（key 相同则不重复添加）
          if (!arr.find(a => this.getDedupeKey(a) === key)) {
            arr.push(item);
          }
        }
        this.setByPath(config, jsonPath, arr);
      }
    }

    // ── objectMerge：浅合并到目标对象 ──────────────────────────────
    if (patch.objectMerge) {
      for (const [jsonPath, obj] of Object.entries(patch.objectMerge)) {
        if (!meta.contributions[jsonPath]) {
          meta.contributions[jsonPath] = { arrays: {}, objects: {} };
        }
        meta.contributions[jsonPath].objects[patch.skillId] = obj; // 记录贡献者
        const existing = this.getByPath(config, jsonPath) ?? {};
        // 多个 skill 的对象按安装顺序叠加合并（后者覆盖前者的同名 key）
        this.setByPath(config, jsonPath, { ...existing, ...obj });
      }
    }

    // 同时原子写入 config 和 meta（两个文件独立原子，非跨文件事务）
    await Promise.all([
      this.atomicWrite(this.configPath, config),
      this.atomicWrite(this.metaPath, meta),
    ]);
  }

  async uninstall(skillId: string): Promise<void> {
    const [config, meta] = await Promise.all([
      this.readJSON(this.configPath),
      this.readJSON(this.metaPath) as Promise<ConfigMeta>,
    ]);
    if (!meta.contributions) return; // 没有 meta 说明从未安装过，无需操作

    await this.removeContributions(config, meta, skillId);

    await Promise.all([
      this.atomicWrite(this.configPath, config),
      this.atomicWrite(this.metaPath, meta),
    ]);
  }

  private async removeContributions(config: any, meta: ConfigMeta, skillId: string): Promise<void> {
    for (const [jsonPath, pathMeta] of Object.entries(meta.contributions ?? {})) {
      // ── 移除该 skill 贡献的数组项 ──────────────────────────────
      for (const [key, contribution] of Object.entries(pathMeta.arrays)) {
        if (contribution.skillId === skillId) {
          delete pathMeta.arrays[key]; // 从 meta 中移除贡献记录
          const arr: any[] = this.getByPath(config, jsonPath) ?? [];
          // 从实际数组中过滤掉该 key 对应的项
          this.setByPath(config, jsonPath, arr.filter(a => this.getDedupeKey(a) !== key));
        }
      }
      // ── 移除该 skill 贡献的对象合并 ────────────────────────────
      if (pathMeta.objects[skillId]) {
        delete pathMeta.objects[skillId]; // 从 meta 中移除
        // 用剩余 skill 的贡献重建合并结果（移除某一层后，重新叠加剩余层）
        const merged = Object.values(pathMeta.objects).reduce((a, b) => ({ ...a, ...b }), {});
        this.setByPath(config, jsonPath, merged);
      }
    }
  }
}
```

**追问回答：**

1. **key 相同但 value 不同的冲突策略：** 策略可以是"先安装者优先"（first-write-wins）或"后安装者优先"（last-write-wins）。推荐：记录冲突到 `.meta.json` 的 `conflicts` 字段，安装时输出警告提示用户（`console.warn` 或 CLI 高亮提示），不静默覆盖。用户可以通过 `--force` 参数明确指定覆盖策略。

2. **uninstall 只移除自己贡献的内容：** 正是 `meta.json` 的作用——每个 arrayAppend 的 item 都记录了是哪个 `skillId` 贡献的，uninstall 时只删除 `skillId` 匹配的 item，其他 skill 贡献的相同 key 的 item 保持不变（通过 `pathMeta.arrays[key].skillId !== skillId` 判断跳过）。

---

### D3. 设计 Generator 协程任务调度器 ⭐⭐⭐⭐

**背景：** 简历中"采用 Generator 协程模型实现执行循环的暂停/恢复语义，暂停时将执行中间态序列化到 Redis"。

**题目：**

```ts
type StepResult =
  | { type: 'continue'; data: any }
  | { type: 'pause'; reason: string }
  | { type: 'done'; result: any }

type WorkflowStep = (input: any) => Generator<StepResult, any, any>;

class WorkflowEngine {
  async create(steps: WorkflowStep[], initialInput: any): Promise<string> {}
  async run(workflowId: string): Promise<
    | { status: 'paused'; reason: string; stepIndex: number }
    | { status: 'done'; result: any }
  > {}
  async resume(workflowId: string, userInput: any): Promise<...> {}
  serialize(workflowId: string): string {}
  deserialize(snapshot: string): string {}
}
```

**追问：**
1. Generator 的执行上下文（局部变量）无法直接序列化，你如何设计数据流使得序列化只需要保存"哪一步的输入"而不需要保存调用栈？
2. 如果某个步骤内部调用了 LLM（异步操作），如何在不重新调用 LLM 的前提下恢复到暂停点之后的状态？

---

```ts
type StepResult =
  | { type: 'continue'; data: any }   // 步骤内部继续（中间状态）
  | { type: 'pause'; reason: string } // 需要用户输入，挂起工作流
  | { type: 'done'; result: any };    // 整个工作流结束

// 每个步骤是一个 Generator 函数：可以 yield 多次（中间状态），最终 return（输出给下一步）
type WorkflowStep = (input: any) => Generator<StepResult, any, any>;

interface WorkflowInstance {
  steps: WorkflowStep[];
  stepIndex: number; // 当前执行到第几步（也是序列化的核心信息）
  stepInput: any;    // 当前步骤的输入（每步完成后，上一步的 return 值成为它的输入）
  status: 'running' | 'paused' | 'done';
  result?: any;
  pauseReason?: string;
}

class WorkflowEngine {
  private instances = new Map<string, WorkflowInstance>(); // 内存中的工作流实例注册表
  private idCounter = 0;

  async create(steps: WorkflowStep[], initialInput: any): Promise<string> {
    const id = `wf-${++this.idCounter}-${Date.now()}`; // 简单的唯一 ID，生产中用 UUID
    this.instances.set(id, {
      steps,
      stepIndex: 0,       // 从第一步开始
      stepInput: initialInput,
      status: 'running',
    });
    return id;
  }

  async run(workflowId: string): Promise<
    | { status: 'paused'; reason: string; stepIndex: number }
    | { status: 'done'; result: any }
  > {
    const instance = this.instances.get(workflowId);
    if (!instance) throw new Error(`工作流不存在: ${workflowId}`);

    // 外层循环：逐步执行（每次从 stepIndex 对应的步骤开始）
    while (instance.stepIndex < instance.steps.length) {
      const stepFn = instance.steps[instance.stepIndex];
      // 每次运行步骤时重新调用生成器函数（暂停后重建 Generator，步骤内部状态从头来）
      // 这也是为什么步骤设计上要通过 input 参数传递状态，而非依赖 Generator 局部变量
      const gen = stepFn(instance.stepInput);

      // 内层循环：驱动单个步骤的 Generator 直到它暂停或完成
      let input: any = undefined; // gen.next(input) 的参数会作为上一个 yield 表达式的值
      while (true) {
        const { value, done } = gen.next(input);
        if (done) {
          // Generator 函数 return 了（步骤完成）
          // value 是 return 语句的值，作为下一步的 input
          instance.stepInput = value;
          instance.stepIndex++;
          break; // 跳出内层循环，外层继续执行下一步
        }

        const stepResult = value as StepResult;
        if (stepResult.type === 'continue') {
          input = stepResult.data; // 把 data 传回给下一次 gen.next()
        } else if (stepResult.type === 'pause') {
          // 工作流挂起：保存当前 stepIndex 和 stepInput 到 instance
          // 稍后 resume() 时从这里继续（但 Generator 本身不保存，重新执行到 pause 点）
          instance.status = 'paused';
          instance.pauseReason = stepResult.reason;
          return { status: 'paused', reason: stepResult.reason, stepIndex: instance.stepIndex };
        } else if (stepResult.type === 'done') {
          instance.status = 'done';
          instance.result = stepResult.result;
          return { status: 'done', result: stepResult.result };
        }
      }
    }

    // 所有步骤执行完毕，最后一步的输出作为整体结果
    instance.status = 'done';
    return { status: 'done', result: instance.stepInput };
  }

  async resume(workflowId: string, userInput: any) {
    const instance = this.instances.get(workflowId);
    if (!instance || instance.status !== 'paused') throw new Error('工作流未处于暂停状态');
    // 把用户输入注入到 stepInput 中，步骤重新执行时可以从 input 参数读取到 userInput
    instance.stepInput = { ...instance.stepInput, userInput };
    instance.status = 'running';
    return this.run(workflowId); // 从 stepIndex 对应的步骤重新执行（Generator 重建）
  }

  serialize(workflowId: string): string {
    const instance = this.instances.get(workflowId);
    if (!instance) throw new Error('工作流不存在');
    // 关键设计：只需序列化"第几步"和"当前输入"，不需要序列化 Generator 调用栈
    // 这是因为步骤间通过显式 stepInput 传数据，不依赖 Generator 内部局部变量
    return JSON.stringify({
      stepIndex: instance.stepIndex,
      stepInput: instance.stepInput,
      status: instance.status,
      pauseReason: instance.pauseReason,
      result: instance.result,
    });
    // 注意：steps（函数数组）是代码，无法 JSON 序列化，恢复时需要调用方重新提供
  }

  deserialize(snapshot: string): string {
    const data = JSON.parse(snapshot);
    const id = `wf-${++this.idCounter}-${Date.now()}`;
    this.instances.set(id, {
      steps: [], // 函数无法序列化，调用方恢复后需要调用 instance.steps = registeredSteps[type] 注入
      ...data,
    });
    return id;
  }
}
```

**示例用法：**
```ts
const engine = new WorkflowEngine();

const steps: WorkflowStep[] = [
  function* step1(input) {
    yield { type: 'continue', data: `处理: ${input}` };
    return '步骤1完成';
  },
  function* step2(input) {
    yield { type: 'pause', reason: '请确认操作' };
    return `步骤2完成，用户确认了`;
  },
];

const wfId = await engine.create(steps, '初始数据');
const result1 = await engine.run(wfId);
// { status: 'paused', reason: '请确认操作', stepIndex: 1 }

const result2 = await engine.resume(wfId, '用户确认');
// { status: 'done', result: '步骤2完成，用户确认了' }
```

**追问回答：**

1. **为什么只需序列化"哪一步的输入"：** 关键设计是**步骤间通过显式数据链传递**，每一步的输入完全来自上一步的返回值（`stepInput`），不依赖 Generator 的局部变量或闭包状态。所以序列化时只需要保存 `{ stepIndex, stepInput }`，恢复时从该步骤重新执行即可，局部变量会在重新运行时自然重建。这就是为什么 Agent 执行框架要"步骤间通过显式数据链传递上游产出，不依赖 LLM 上下文记忆"。

2. **步骤内调用 LLM 后不重复调用：** 将 LLM 的调用结果保存到 `stepInput` 的快照中（在 yield pause 之前把 LLM 结果存到上下文）。恢复时，步骤逻辑先检查快照中是否已有该 LLM 调用的结果，有则直接使用，不重新调用。设计上让每个可能暂停的步骤在 yield 前完成所有 LLM 推理，将结果写入状态，yield 后只做用户交互和后续处理，确保恢复点之前的推理结果已持久化。
