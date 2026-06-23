---
layout: post
title: "MCP 工具链面试问答（一）：Skill 设计与安装引擎"
date: 2025-01-17
categories: 面试
tags: [AI, MCP, CLI, TypeScript, Node.js]
---

> 基于简历「AI Agent 工具链与 MCP 生态建设」项目推导，考察 Skill 激活机制、安装引擎幂等性、权限合并策略等问题。

---

## Q1. Skill "激活"的具体机制，以及对用户现有配置的侵入性控制

**问题：** Skill 的三种形态中，"Agent 工作流型 Skill 安装后能直接激活完整的 AI 协作模式"——这个"激活"具体发生了什么？是向 Claude/Cursor 等工具注入 System Prompt，还是修改了工具的配置文件，还是注册了 MCP Server？对用户现有配置的侵入性如何控制？

---

### 一、Skill 的两种形态与激活机制

项目中 Skill 分两种形态，激活机制完全不同：

**形态一：单文件 Markdown Skill（skillType=1）**

纯文本文件，安装就是把 `.md` 文件写到 Agent 的 skills 目录：

```typescript
// skill-hub-installer.ts
if (result.skillType === 1) {
  await fs.writeFile(path.join(dir, 'SKILL.md'), result.buffer);
}
```

Claude Code 在启动时会扫描 `~/.claude/skills/` 目录，把其中的 `.md` 文件内容自动注入到 System Prompt，这一步由 Claude Code 自身完成，CLI 不直接操作 System Prompt。

**形态二：Archive Bundle Skill（skillType=2，带 INSTALL.json）**

这是"Agent 工作流型 Skill"，一个 ZIP 包，内含 `INSTALL.json` 声明需要安装的组件。激活过程由 `MergeInstaller` 完成，实际上是四步操作的组合：

```typescript
// skill-hub-installer.ts
const mergeInstaller = new MergeInstaller(
  zipForInstall.getEntries(),
  installManifestObj,
  claudeBase,
  { force: false }
);

await mergeInstaller.copyProvides();     // 1. 复制文件到 Agent 目录
await mergeInstaller.mergeSettings();    // 2. 幂等合并 settings.json
await mergeInstaller.appendClaudeMd();   // 3. 追加 CLAUDE.md（带 marker）
await mergeInstaller.registerMcp();      // 4. 注册 MCP Server
```

**`INSTALL.json` 声明示例：**

```json
{
  "runtime": "claude-code",
  "provides": {
    "skills": "skills/",
    "rules":  "rules/",
    "settings": "settings.json",
    "mcp": { "src": "mcp.json", "scope": "user" },
    "claude-md": { "src": "CLAUDE.md", "mode": "append", "marker": "my-skill" }
  }
}
```

各 `provides` 字段对应的激活行为：
- `skills/rules/agents/commands/hooks/scripts`：复制文件到 `~/.claude/` 对应目录，Claude Code 读取这些目录完成激活
- `settings`：向 `~/.claude/settings.json` 合并权限和 Hook 配置
- `mcp`：调用 `claude mcp add` 注册 MCP Server，Claude Code 重启后生效
- `claude-md`：向 `~/.claude/CLAUDE.md` 追加内容，Claude Code 启动时读取

---

### 二、侵入性控制机制

侵入性控制的核心是三个机制：

**1. 幂等追加，绝不覆盖**

`settings.json` 的 `permissions.allow` 使用 Set 去重追加，已有配置不会被删除：

```typescript
// merge-installer.ts
dst.permissions[key] = [...new Set([
  ...(dst.permissions[key] ?? []),  // 保留用户现有配置
  ...src.permissions[key]           // 追加 Skill 所需权限
])];
```

Hook 的合并也检查 command 是否已存在，相同 command 不重复注册：

```typescript
const exists = (dst.hooks[event] as any[]).some(
  (g: any) => (g.hooks ?? []).some((h: any) => h.command === srcHook.command)
);
if (!exists) { /* 才追加 */ }
```

**2. CLAUDE.md 带 marker 的幂等追加**

追加内容用 `<!-- INSTALL-BEGIN:marker -->` 和 `<!-- INSTALL-END:marker -->` 包裹，识别到 marker 存在时跳过（非 force 模式）：

```typescript
// merge-installer.ts
const begin = `<!-- INSTALL-BEGIN:${provide.marker} -->`;
const end   = `<!-- INSTALL-END:${provide.marker} -->`;

if (beginIdx !== -1 && endIdx !== -1 && !this.opts.force) {
  return;  // 已安装，跳过，不修改用户内容
}
// 首次安装：appendFile，不替换用户已有内容
await fs.appendFile(dstPath, `\n${newSection}\n`);
```

**3. 精确卸载，不影响其他 Skill**

卸载时只删除 marker 包裹的内容，不动其他配置：

```typescript
// skill-hub-installer.ts uninstall
const existing = await fs.readFile(claudeMdPath, 'utf-8');
// 精确切割 BEGIN/END 区间，保留区间外所有内容
const cleaned = existing.slice(0, beginIdx) + existing.slice(endIdx + end.length);
```

---

### 三、多 Agent 支持与路径隔离

不同 Agent 有各自独立的安装路径，不互相干扰：

```typescript
// agents.ts
export const SUPPORTED_AGENTS = {
  'claude-code': {
    global: '~/.claude/skills',
    local:  '.claude/skills',
  },
  'openclaw': {
    global: '~/.openclaw/skills',
    local:  '.openclaw/skills',
  },
  'codex': {
    global: '~/.codex/skills',
    local:  '.codex/skills',
  },
};
```

用户选择安装到哪些 Agent（交互式多选），CLI 只修改对应 Agent 的配置文件，其他 Agent 不受影响。

---

### 追问

**追问 1：`INSTALL.json` 的 runtime 字段如果写错了，会安装到错误的 Agent 吗？**

不会。安装前会检查 `installManifestObj.runtime === agentId`，不匹配则退化为普通 ZIP 解压，不执行 `mergeSettings` 和 `registerMcp`：

```typescript
} else if (installManifestObj && installManifestObj.runtime === agentId) {
  // 仅 runtime 匹配时才走 MergeInstaller
  await mergeInstaller.copyProvides();
  // ...
} else {
  await this.extractBundle(result.buffer, dir);  // 普通解压，不注入配置
}
```

**追问 2：MCP Server 注册用的是 `claude mcp add`，这要求用户环境里有 `claude` 命令。如果用户没装 Claude Code CLI 怎么办？**

这是一个已知依赖。`registerMcp` 内部用 `execAsync` 调用 `claude mcp add`，如果失败会 catch 掉但不中断安装流程——其他文件已经复制完成，只有 MCP Server 注册失败，用户看到警告可以手动注册。从设计角度说，MCP 型 Skill 本身就以 Claude Code 为前提，这个依赖是合理的。

**追问 3：用户安装了多个 Skill，都往 CLAUDE.md 里追加内容，会不会导致 CLAUDE.md 非常长？**

会，这是设计上的权衡。每个 Skill 追加的内容有 marker 隔离，不互相干扰，但确实会累积变长。目前没有 CLAUDE.md 大小限制的自动管理。理论上可以按 marker 统计各 Skill 占用的字节数，在安装新 Skill 时给出提示，但目前优先保证安全性（幂等、可精确卸载）。

---

## Q2. 多 Skill 同名权限的去重合并策略

**问题：** 声明式安装引擎中，多个 Skill 声明了同名工具权限但参数配置不同（比如两个 Skill 都声明了 `bash` 权限但 allowedCommands 不同），你的去重合并策略是什么？是取并集、取交集还是报冲突？

---

### 一、核心策略：取并集，不报冲突

对于 `permissions.allow`/`ask`/`deny` 三个数组，合并策略是**取并集**：

```typescript
// merge-installer.ts
for (const key of ['allow', 'ask', 'deny'] as const) {
  if (src.permissions[key]) {
    dst.permissions[key] = [...new Set([
      ...(dst.permissions[key] ?? []),  // 现有权限
      ...src.permissions[key]           // 新 Skill 声明的权限
    ])];
  }
}
```

用 `Set` 去重，相同字符串的权限项只保留一条。两个 Skill 都声明 `"Bash(git:*)"` 时，最终 `allow` 里只有一条 `"Bash(git:*)"` 。

---

### 二、"allowedCommands 不同"的场景实际是什么

Claude Code 的 `settings.json` 里 `permissions.allow` 是一个字符串数组，格式是 `"ToolName(pattern)"` 这样的字符串。所以两个 Skill 分别声明 `"Bash(git:*)"` 和 `"Bash(npm:*)"` 时，它们是两条不同的字符串——Set 去重后都会保留，最终用户获得的是两条权限的并集：

```json
{
  "permissions": {
    "allow": ["Bash(git:*)", "Bash(npm:*)"]
  }
}
```

这两条规则同时生效，用户可以执行 `git` 和 `npm` 命令。

---

### 三、Hooks 的合并策略

Hooks 的合并逻辑更细：按 command 去重，相同 command 不重复追加：

```typescript
const exists = (dst.hooks[event] as any[]).some(
  (g: any) => (g.hooks ?? []).some((h: any) => h.command === srcHook.command)
);
if (!exists) {
  target.hooks.push(srcHook);  // 不同 command 才追加
}
```

两个 Skill 都注册 `PostToolUse` hook 但 command 不同，两个 hook 都会保留。如果 command 完全相同，后安装的 Skill 的 hook 不会重复追加。

---

### 四、不报冲突的设计理由

不报冲突是有意为之：

1. **用户不需要感知 Skill 内部实现细节**。权限是"授予"性质的，合并出的并集等价于"所有 Skill 所需权限的最小超集"，对用户是安全的（用户可以手动删除多余权限）。

2. **CLI 是安装工具，不是仲裁者**。两个 Skill 声明的权限有重叠，不代表它们冲突，权限粒度本来就允许多条规则并存。

3. **拒绝安装成本太高**。如果报冲突，用户要手动解决才能继续安装，体验很差，而实际上大多数"冲突"都可以安全合并。

---

### 追问

**追问 1：如果一个 Skill 声明 `allow: ["Bash(*:*)"]`（允许所有 bash 命令），另一个声明 `deny: ["Bash(rm:*)"]`，合并后 allow 里有 `Bash(*:*)` 而 deny 里有 `Bash(rm:*)`，Claude Code 实际行为是什么？**

这取决于 Claude Code 本身的权限优先级规则——CLI 只负责合并写入，不解释权限语义。Claude Code 通常是 deny 优先于 allow，所以即使 allow 有通配，deny 里的 `Bash(rm:*)` 也会生效。但这个行为是 Claude Code 的，不在 CLI 的控制范围内，也不在当前实现中做验证。这是设计上的边界划分：CLI 管安装，Claude Code 管执行时权限解释。

**追问 2：有没有 deny 与 allow 同时存在同一个条目的检测？**

没有。当前实现对 `allow`、`ask`、`deny` 三个数组独立处理，不做跨数组的语义冲突检测。如果某个 Skill 写入了自相矛盾的配置（同时 allow 和 deny 同一个条目），合并后也会原样保留，行为由 Claude Code 决定。这是当前实现的已知局限，有意选择了"简单合并"而非"语义验证"。

---

## Q3. 幂等追加与进程崩溃场景的处理

**问题：** "配置文件幂等追加"在安装一半时进程崩溃的场景下，如何保证下次安装不会产生重复配置或脏配置？你有没有实现类似数据库事务的回滚机制？

---

### 一、没有事务回滚，幂等性是防线

直接说实情：**没有实现类似数据库事务的回滚机制**。没有 WAL 日志，没有两阶段提交，没有原子重命名替换。安装流程是顺序执行的多个文件操作，如果中途崩溃，磁盘状态可能是"一半写完"的。

防止下次安装产生重复或脏配置的手段是**幂等性本身**：

**1. `CLAUDE.md` 的 marker 检查**

崩溃场景一：写完 `SKILL.md` 后崩溃，`CLAUDE.md` 还没追加。下次安装：重新追加，无 marker 所以正常写入，结果正确。

崩溃场景二：`CLAUDE.md` 已追加（marker 已写入）但后续步骤（`mergeSettings`）没完成。下次安装：`appendClaudeMd` 检测到 marker 存在，跳过；`mergeSettings` 执行幂等合并（Set 去重），补完缺失配置。

```typescript
// 检测到 marker 存在就跳过，不重复追加
if (beginIdx !== -1 && endIdx !== -1 && !this.opts.force) {
  return;
}
```

**2. `settings.json` 的 Set 去重**

即使 `mergeSettings` 重复执行，`Set` 去重保证同一条权限不会出现两次：

```typescript
dst.permissions[key] = [...new Set([
  ...(dst.permissions[key] ?? []),
  ...src.permissions[key]
])];
```

**3. MCP Server 的先 remove 再 add**

```typescript
try {
  await execAsync(`claude mcp remove -s ${provide.scope} ${name}`);
} catch { /* not installed yet */ }
await execAsync(`claude mcp add -s ${provide.scope} ${name} ${def.command}${argStr}`);
```

先无条件 remove（失败也 catch），再 add。重复执行结果幂等。

**4. 文件复制的覆盖语义**

`copyProvides` 用 `fs.copy` 复制文件，`overwrite: true`（fs-extra 默认），同名文件直接覆盖，不产生重复。

---

### 二、崩溃后真正有风险的场景

幂等性覆盖了大多数崩溃场景，但有一个真实风险：

**`CLAUDE.md` 写入到一半时崩溃**。`appendFile` 是一次系统调用，理论上很难写到一半，但如果文件系统 flush 不完整，可能出现 marker 只写了 BEGIN 没写 END 的情况。下次安装时，`indexOf(begin) !== -1` 但 `indexOf(end) === -1`，既不匹配"已安装"的判断，也不匹配"未安装"的判断，会走到 `appendFile` 追加一份新的完整内容——导致 BEGIN 出现两次。

这个场景的概率极低（需要文件系统半写 + 进程恰好在 appendFile 中崩溃），目前没有专门处理。更健壮的做法是写临时文件再 atomic rename，但考虑到配置文件本身的重要性不如数据库，接受这个风险。

---

### 三、Skill Manifest 的防重复作用

`~/.myfe-cli/skill-hub-manifest.json` 记录了已安装的 Skill 及其 etag：

```typescript
// skill-hub-installer.ts
manifest[ref] = {
  etag: result.etag,
  skillType: result.skillType,
  scope,
  agents,
};
await this.writeManifest(manifest, ...);
```

这个 manifest 是在所有安装步骤完成后才写入的。崩溃时 manifest 没更新，下次用户运行 `myfe skills install` 时，`isInstalled` 检查 manifest 不含该 Skill，会重新安装——所有步骤重新执行，幂等性保证最终状态正确。

| 崩溃时机 | manifest 状态 | 下次安装结果 |
|---------|------------|------------|
| 复制文件前 | 无记录 | 重新安装，结果正确 |
| 复制文件后、mergeSettings 前 | 无记录 | 重新安装，文件覆盖 + 配置幂等合并，正确 |
| mergeSettings 后、manifest 写入前 | 无记录 | 重新安装，所有步骤幂等重执行，正确 |
| manifest 写入后 | 有记录 | update 检查 etag，不变则跳过 |

---

### 追问

**追问 1：manifest 写入本身崩溃了怎么办？manifest 文件是不是也可能写到一半？**

是的，manifest 写入同样没有事务保护。`fs.writeJSON` 底层是 `JSON.stringify` + `writeFile`，如果进程在写入途中崩溃，manifest 文件可能损坏（非合法 JSON）。下次读取时 `JSON.parse` 会抛异常，目前的处理是：

```typescript
// 读取 manifest 失败时返回空对象，相当于"没有任何已安装记录"
try {
  return await fs.readJSON(this.manifestPath(projectPath));
} catch {
  return {};
}
```

这意味着即使 manifest 损坏，下次安装会重新安装所有 Skill，幂等性保证最终正确，但会有重复安装的性能开销。可以通过 write-temp-then-rename 的原子写入来避免，目前没有实现。

**追问 2：用户手动编辑了 CLAUDE.md，把 marker 之间的内容改了，下次强制重装（force=true）会发生什么？**

`force=true` 时，`appendClaudeMd` 检测到 marker 存在，会用 Skill 原始内容替换 marker 区间：

```typescript
if (!this.opts.force) return;
await fs.writeFile(
  dstPath,
  existing.slice(0, beginIdx) + newSection + existing.slice(endIdx + end.length),
);
```

用户的手动修改会丢失。这是 `force` 模式的语义——用 Skill 原始版本覆盖，适合升级场景。普通安装（`force=false`）不会覆盖用户修改。这个行为在 `--force` 标志的文档里应该说清楚，是已知的 UX 权衡。
