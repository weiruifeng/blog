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
  private state: CircuitState = 'CLOSED';
  private failureCount = 0;
  private successCount = 0;
  private openedAt = 0;
  private probing = false; // HALF_OPEN 中是否已有探测请求在进行

  constructor(private options: {
    failureThreshold: number;
    successThreshold: number;
    timeout: number;
    onStateChange?: (from: CircuitState, to: CircuitState) => void;
  }) {}

  private transition(to: CircuitState): void {
    const from = this.state;
    this.state = to;
    this.options.onStateChange?.(from, to);
  }

  async execute<T>(fn: () => Promise<T>): Promise<T> {
    // 惰性状态转移：OPEN → HALF_OPEN
    if (this.state === 'OPEN') {
      if (Date.now() - this.openedAt >= this.options.timeout) {
        this.transition('HALF_OPEN');
        this.successCount = 0;
        this.probing = false;
      } else {
        throw new Error('CircuitBreaker: OPEN，快速失败');
      }
    }

    if (this.state === 'HALF_OPEN') {
      if (this.probing) throw new Error('CircuitBreaker: HALF_OPEN 探测中，快速失败');
      this.probing = true;
      try {
        const result = await fn();
        this.successCount++;
        if (this.successCount >= this.options.successThreshold) {
          this.failureCount = 0;
          this.transition('CLOSED');
        } else {
          this.probing = false; // 还未达成功阈值，允许下次探测
        }
        return result;
      } catch (err) {
        this.openedAt = Date.now();
        this.transition('OPEN');
        throw err;
      }
    }

    // CLOSED 正常执行
    try {
      const result = await fn();
      this.failureCount = 0;
      return result;
    } catch (err) {
      this.failureCount++;
      if (this.failureCount >= this.options.failureThreshold) {
        this.openedAt = Date.now();
        this.transition('OPEN');
      }
      throw err;
    }
  }

  getState(): CircuitState { return this.state; }
}
```

**追问回答：**

1. **协程安全：** Node.js 是单线程事件循环，`this.probing = true` 的赋值是同步操作，检查和赋值之间不会有其他协程插入（因为没有 `await`），所以此实现已经是安全的。若在多线程环境（如 Java），需用 `AtomicBoolean` 的 `compareAndSet(false, true)` 实现 CAS 操作。

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
      try { return await this.primary.execute(this.primaryFn); } catch {}
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

interface ConfigMeta {
  contributions: {
    [jsonPath: string]: {
      arrays: { [key: string]: { value: any; skillId: string } };
      objects: { [skillId: string]: object };
    };
  };
}

class IdempotentInstaller {
  private configPath: string;
  private metaPath: string;

  constructor(configPath: string) {
    this.configPath = configPath;
    this.metaPath = configPath + '.meta.json';
  }

  private async readJSON(filePath: string): Promise<any> {
    try { return JSON.parse(await fs.readFile(filePath, 'utf-8')); }
    catch { return {}; }
  }

  private async atomicWrite(filePath: string, data: any): Promise<void> {
    const tmp = filePath + '.tmp.' + Date.now();
    await fs.writeFile(tmp, JSON.stringify(data, null, 2), 'utf-8');
    await fs.rename(tmp, filePath); // 原子重命名
  }

  private getByPath(obj: any, jsonPath: string): any {
    return jsonPath.split('.').reduce((o, k) => o?.[k], obj);
  }

  private setByPath(obj: any, jsonPath: string, value: any): void {
    const keys = jsonPath.split('.');
    let cur = obj;
    for (let i = 0; i < keys.length - 1; i++) {
      if (!cur[keys[i]]) cur[keys[i]] = {};
      cur = cur[keys[i]];
    }
    cur[keys[keys.length - 1]] = value;
  }

  private getDedupeKey(item: any): string {
    return item?.id ?? item?.name ?? JSON.stringify(item);
  }

  async install(patch: ConfigPatch): Promise<void> {
    const [config, meta] = await Promise.all([
      this.readJSON(this.configPath),
      this.readJSON(this.metaPath) as Promise<ConfigMeta>,
    ]);
    if (!meta.contributions) meta.contributions = {};

    // 先卸载旧版本（幂等：重复安装先清理再重新贡献）
    this.removeContributions(config, meta, patch.skillId);

    if (patch.arrayAppend) {
      for (const [jsonPath, items] of Object.entries(patch.arrayAppend)) {
        if (!meta.contributions[jsonPath]) meta.contributions[jsonPath] = { arrays: {}, objects: {} };
        const pathMeta = meta.contributions[jsonPath];
        let arr: any[] = this.getByPath(config, jsonPath) ?? [];
        for (const item of items) {
          const key = this.getDedupeKey(item);
          if (pathMeta.arrays[key] && pathMeta.arrays[key].skillId !== patch.skillId) {
            // 冲突：已有其他 skill 贡献了相同 key，不覆盖，记录冲突可在此上报
            console.warn(`[IdempotentInstaller] 冲突: ${jsonPath}[${key}] 已由 ${pathMeta.arrays[key].skillId} 贡献`);
            continue;
          }
          pathMeta.arrays[key] = { value: item, skillId: patch.skillId };
          if (!arr.find(a => this.getDedupeKey(a) === key)) arr.push(item);
        }
        this.setByPath(config, jsonPath, arr);
      }
    }

    if (patch.objectMerge) {
      for (const [jsonPath, obj] of Object.entries(patch.objectMerge)) {
        if (!meta.contributions[jsonPath]) meta.contributions[jsonPath] = { arrays: {}, objects: {} };
        meta.contributions[jsonPath].objects[patch.skillId] = obj;
        const existing = this.getByPath(config, jsonPath) ?? {};
        this.setByPath(config, jsonPath, { ...existing, ...obj });
      }
    }

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
    if (!meta.contributions) return;
    this.removeContributions(config, meta, skillId);
    await Promise.all([
      this.atomicWrite(this.configPath, config),
      this.atomicWrite(this.metaPath, meta),
    ]);
  }

  private removeContributions(config: any, meta: ConfigMeta, skillId: string): void {
    for (const [jsonPath, pathMeta] of Object.entries(meta.contributions ?? {})) {
      for (const [key, contribution] of Object.entries(pathMeta.arrays)) {
        if (contribution.skillId === skillId) {
          delete pathMeta.arrays[key];
          const arr: any[] = this.getByPath(config, jsonPath) ?? [];
          this.setByPath(config, jsonPath, arr.filter(a => this.getDedupeKey(a) !== key));
        }
      }
      if (pathMeta.objects[skillId]) {
        delete pathMeta.objects[skillId];
        // 重建合并结果（只保留其他 skill 的贡献）
        const merged = Object.values(pathMeta.objects).reduce((a, b) => ({ ...a, ...b }), {});
        this.setByPath(config, jsonPath, merged);
      }
    }
  }
}
```

**追问回答：**

1. **冲突策略：** 推荐"先安装者优先"（first-write-wins），并将冲突记录到 `.meta.json` 的 `conflicts` 字段，安装时输出警告（`console.warn` 或 CLI 高亮提示），不静默覆盖。用户可以通过 `--force` 参数明确指定覆盖策略。

2. **uninstall 只移除自己贡献：** 正是 `.meta.json` 的作用——每个 arrayAppend 的 item 都记录了是哪个 `skillId` 贡献的，uninstall 时只删除 `skillId` 匹配的 item（通过 `contribution.skillId === skillId` 判断），其他 skill 贡献的相同 key 的 item 保持不变。

---

### D3. 设计一个 Generator 协程任务调度器 ⭐⭐⭐⭐

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
interface WorkflowInstance {
  steps: WorkflowStep[];
  stepIndex: number;
  stepInput: any;
  status: 'running' | 'paused' | 'done';
  result?: any;
  pauseReason?: string;
}

class WorkflowEngine {
  private instances = new Map<string, WorkflowInstance>();
  private idCounter = 0;

  async create(steps: WorkflowStep[], initialInput: any): Promise<string> {
    const id = `wf-${++this.idCounter}-${Date.now()}`;
    this.instances.set(id, { steps, stepIndex: 0, stepInput: initialInput, status: 'running' });
    return id;
  }

  async run(workflowId: string): Promise<
    | { status: 'paused'; reason: string; stepIndex: number }
    | { status: 'done'; result: any }
  > {
    const instance = this.instances.get(workflowId);
    if (!instance) throw new Error(`工作流不存在: ${workflowId}`);

    while (instance.stepIndex < instance.steps.length) {
      const stepFn = instance.steps[instance.stepIndex];
      const gen = stepFn(instance.stepInput);
      let input: any = undefined;

      while (true) {
        const { value, done } = gen.next(input);
        if (done) {
          // 步骤完成，value 是返回值，作为下一步输入
          instance.stepInput = value;
          instance.stepIndex++;
          break;
        }
        const stepResult = value as StepResult;
        if (stepResult.type === 'continue') {
          input = stepResult.data;
        } else if (stepResult.type === 'pause') {
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

    instance.status = 'done';
    return { status: 'done', result: instance.stepInput };
  }

  async resume(workflowId: string, userInput: any) {
    const instance = this.instances.get(workflowId);
    if (!instance || instance.status !== 'paused') throw new Error('工作流未处于暂停状态');
    instance.stepInput = { ...instance.stepInput, userInput };
    instance.status = 'running';
    return this.run(workflowId);
  }

  serialize(workflowId: string): string {
    const instance = this.instances.get(workflowId);
    if (!instance) throw new Error('工作流不存在');
    // 只序列化"第几步 + 当前输入"，不序列化调用栈
    return JSON.stringify({
      stepIndex: instance.stepIndex,
      stepInput: instance.stepInput,
      status: instance.status,
      pauseReason: instance.pauseReason,
      result: instance.result,
    });
  }

  deserialize(snapshot: string): string {
    const data = JSON.parse(snapshot);
    const id = `wf-${++this.idCounter}-${Date.now()}`;
    this.instances.set(id, {
      steps: [], // 调用方恢复时需手动注入（函数无法序列化，通过类型注册表查找）
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

1. **为什么只需序列化"哪一步的输入"：** 关键设计是**步骤间通过显式数据链传递**——每一步的输入完全来自上一步的返回值（`stepInput`），不依赖 Generator 的局部变量或闭包状态。序列化时只需保存 `{ stepIndex, stepInput }`，恢复时从该步骤重新执行即可，局部变量会在重新运行时自然重建。这正是简历中所说的"步骤间通过显式数据链传递上游产出，不依赖 LLM 上下文记忆"的工程含义。

2. **步骤内调用 LLM 后不重复调用：** 将 LLM 的调用结果保存到 `stepInput` 的快照中（在 `yield pause` 之前把 LLM 结果存到上下文）。恢复时，步骤逻辑先检查快照中是否已有该 LLM 调用的结果，有则直接使用，不重新调用。设计上让每个可能暂停的步骤在 yield 前完成所有 LLM 推理，将结果写入状态，yield 后只做用户交互和后续处理，确保恢复点之前的推理结果已持久化。
