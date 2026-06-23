---
layout: post
title: "MCP 工具链面试问答（三）：AI 安全扫描与版本管理"
date: 2025-01-19
categories: 面试
tags: [AI, MCP, CLI, TypeScript, Node.js]
---

> 基于简历「AI Agent 工具链与 MCP 生态建设」项目推导，考察 AI 安全扫描流水线、软下架机制、ETag 增量更新与分布式一致性等问题。

---

## Q7. 高风险 Skill 扫描完成前已被大量安装的应急处置

**问题：** AI 安全扫描流水线"不阻塞发布但标记高风险版本"——如果一个高风险 Skill 在扫描完成之前已经被大量用户安装，你的应急处置链路是什么？软下架机制能覆盖这个场景吗？

---

### 一、当前发布与扫描的时序关系

先说清楚现有的设计：**安全扫描与版本发布是并行的，不是串行的。**

```
用户 publish Skill
   ↓
版本创建（status=0 pending）
   ↓
运营审核通过 → updateStatus(id, 1) → 版本 published
   ↓（同时异步触发）
computeAndSaveDigest(id)     — 异步计算 SHA-256
triggerSecurityScan(id)      — 异步发起 AI 扫描
```

版本状态变为 `published` 之后，Skill 就可以被用户下载安装了，此时扫描可能还没完成。这意味着确实存在"扫描结论出来前已有用户安装"的时间窗口。

---

### 二、扫描结果落地后的标记机制

扫描完成后，结果存入 `skill_hub_security_scan` 表：

```typescript
// security-scan.service.ts
await securityScanModel.upsert({
  versionId,
  scanStatus: 2,           // DONE
  riskLevel: mergedRisk,   // 0=SAFE, 1=LOW, 2=MEDIUM, 3=HIGH
  findings: allFindings,   // 风险项详情
  scannedAt: new Date(),
  rawResponse: rawText,
});
```

高风险版本（`riskLevel=3`）会被标记，但**当前实现不会自动触发任何对用户的通知或下架动作**——标记只是数据库记录，需要人工介入才能触发后续处置。

---

### 三、软下架（Yank）机制

Yank 是运营手动触发的：

```typescript
// version.service.ts
async yank(versionId: number, yankReason: string): Promise<void> {
  await versionModel.update(
    { versionStatus: 3, yankReason },
    { where: { id: versionId } }
  );
}
```

版本 `versionStatus=3` 后，`registry.service.ts` 的 `findLatestApproved` 会过滤掉 yank 版本：

```typescript
where: {
  skillId,
  versionStatus: 1,  // 只返回 status=1（published）的版本
}
```

CLI 运行 `myfe skills update` 时，服务端返回当前最新可用版本，如果最新版被 yank，会降级到上一个 `status=1` 的版本。

---

### 四、软下架能覆盖这个场景吗？

**部分覆盖，但有盲区。**

软下架能做到的：
- 阻止新用户安装高风险版本（服务端不再下发该版本）
- 已安装的用户运行 `myfe skills update` 时，拉到的是降级后的安全版本

软下架不能做到的：
- **主动推送**：已安装用户不会收到任何通知，只有主动 update 才会感知
- **强制卸载**：CLI 没有服务端推送通道，无法主动触发用户侧的卸载或回滚
- **安装数统计**：当前没有精确的"高风险版本被安装了多少次"的实时数据，只有总下载数

**对比完整应急链路应有的能力：**

| 能力 | 当前实现 | 状态 |
|------|---------|------|
| 标记高风险版本 | scan 结果写入 DB | ✅ 已实现 |
| 阻止新安装 | Yank → 不在下发列表 | ✅ 已实现（手动触发） |
| 已安装用户更新时降级 | update 命令走 latest | ✅ 已实现 |
| 主动通知已安装用户 | 无 | ❌ 未实现 |
| 强制回滚 | 无 | ❌ 未实现 |
| 自动触发 Yank（扫描→自动下架） | 无，需人工 | ❌ 未实现 |

---

### 五、应急处置的实际链路（当前）

1. 运营在平台看到扫描结果（`riskLevel=3`）
2. 运营手动 Yank 该版本，填写 `yankReason`
3. 已安装用户下次运行 `myfe skills update` 时获得降级版本
4. 没有运行 update 的用户继续使用高风险版本，无任何提示

**更完整的方案（未实现）：** wellknown index 中加入版本摘要和 yank 状态，CLI 定期拉取 index（或每次命令执行前检查），发现已安装版本被 yank 时自动提示用户更新，甚至自动触发降级。

---

### 追问

**追问 1：扫描阶段失败（LLM 接口超时或返回格式异常），版本的 scanStatus 是什么，对用户有影响吗？**

失败时 `scanStatus=2`（DONE），但 `riskLevel=null`，`findings=[]`。当前逻辑里 `riskLevel=null` 的版本不会被标记为高风险，用户可以正常下载。这意味着扫描失败等价于"扫描结果未知"，不会阻断安装——优先保证用户体验，接受扫描不可靠时的安全风险。更严格的做法是扫描失败时标记为"未扫描"状态，需要重扫才能发布，但当前没有实现。

**追问 2：AI 安全扫描的准确率如何，有没有评估过误报和漏报？**

没有正式评估，这是当前实现的局限。扫描提示词（`SkillHubSecurityScanLite` 和 `SkillHubSecurityScanPro`）基于安全规则描述让 LLM 判断，结果依赖模型能力，没有做基准测试。已知的挑战：.md 格式的 Skill 主要是指令文本，没有可执行代码，传统的"命令注入"类检测效果好；但"Prompt 注入"（Skill 指令诱导 AI 执行恶意操作）的检测边界模糊，当前模型未必能可靠识别。

---

## Q8. 软下架（Yank）的回退触发时机与离线用户感知

**问题：** 软下架（Yank）"被下架版本自动回退到上一个可用版本"——这个回退是在 CLI 工具运行时触发的，还是服务端推送的？如果用户处于离线状态，他们如何感知到版本被下架？

---

### 一、回退触发时机：CLI pull，不是服务端 push

**回退是 CLI 主动 pull，没有服务端推送机制。**

用户运行 `myfe skills update` 时，CLI 请求 `GET /cli/skills/:name/latest`，服务端返回最新的 `versionStatus=1`（published）版本——Yank 版本（`versionStatus=3`）被过滤掉，自然降级到上一个可用版本：

```typescript
// registry.service.ts
async findLatestPublished(skillId: bigint): Promise<SkillHubVersion | null> {
  return this.versionModel.findOne({
    where: { skillId, versionStatus: 1 },  // 只返回 published 版本
    order: [['id', 'DESC']],
  });
}
```

CLI 侧的更新检查依赖 ETag：

```typescript
// skill-hub-client.ts
async download(ref: string, ifNoneMatch?: string): Promise<SkillDownloadResult | null> {
  const headers: Record<string, string> = {};
  if (ifNoneMatch) headers['if-none-match'] = ifNoneMatch;
  
  const response = await axios.get(`${base}/cli/skills/${ref}/latest`, { headers });
  
  if (response.status === 304) return null;  // etag 匹配，无需更新
  
  return {
    buffer: Buffer.from(response.data),
    etag: response.headers['etag'] as string,
    // ...
  };
}
```

`myfe skills update` 读取本地 manifest 中的 etag，用 `If-None-Match` 请求服务端，如果版本被 yank，服务端返回降级版本（不同 etag），CLI 收到 200 后更新到降级版本。

---

### 二、离线用户完全无感知

**离线状态下，用户不会知道版本被 yank。**

CLI 没有本地缓存的 yank 状态，也没有任何离线通知机制。已安装的 Skill 文件存在磁盘上，Agent（如 Claude Code）直接读取这些文件使用，不经过 CLI。用户不联网，也不运行 `myfe skills update`，就会持续使用 yank 版本。

这是当前设计的已知缺陷。wellknown index（`/.well-known/agent-skills/index.json`）提供了另一条可能的感知路径：

```typescript
// wellknown.service.ts
// index 里每条 skill 带有 digest 字段（SHA-256）
{
  name: 'react-best-practices',
  type: 'skill-md',
  url: '/cli/skills/react-best-practices/latest',
  digest: 'sha256:abc123...',
  installType: 'claude-code',
}
```

如果 index 里的 digest 发生变化（yank 后降级版本的 digest 不同），CLI 可以检测到并自动更新。但**当前 CLI 没有定期检查 wellknown index 的主动轮询逻辑**，这条路径只在用户主动 `myfe skills update` 时走到。

---

### 三、为什么没做服务端推送

主要原因是 CLI 是命令行工具，不是常驻进程。用户不执行命令时，没有连接可以接受推送。即使做 WebSocket 或 SSE，也需要 CLI 作为后台守护进程运行，这对 CLI 工具来说过于重量级。

实际可行的替代方案（未实现）：

1. **每次命令执行前的后台检查**：`myfe skills install/update/list` 等命令执行时，非阻塞地后台请求一次 yank 状态，发现已安装版本被 yank 时打印警告
2. **wellknown index 轮询**：CLI 本地缓存 index，每次执行命令时 If-None-Match 拉取增量更新，digest 变化时提示

---

### 追问

**追问 1：Yank 后"上一个可用版本"是怎么确定的？如果历史上所有版本都被 yank 了，返回什么？**

"上一个可用版本"是 `versionStatus=1` 且 `id` 最大的记录（按创建时间倒序）。如果所有历史版本都被 yank，`findLatestPublished` 返回 `null`，服务端返回 404，CLI 的 `download` 方法返回 `null`，`update` 命令提示"无可用版本"。这个场景实际等价于 Skill 完全不可用，应当同时把 Skill 本身的 `skillStatus` 设为 `3`（deprecated）。

**追问 2：版本降级时，CLI 是否会提示用户"因安全原因降级到 X.X.X"？**

当前不会。降级对用户透明——只是 etag 不匹配后下载了一个新版本，用户看到的是"更新成功"，不知道是因为原版本被 yank。更好的体验应该在响应头或响应体里携带 `X-Yanked: true` 和 `X-Yank-Reason` 字段，CLI 检测后提示用户原因。目前没有实现。

---

## Q9. 条件请求增量更新的 ETag 设计与分布式一致性

**问题：** "条件请求增量更新"——你用的是 ETag 还是 Last-Modified？在分布式部署（多台服务器）场景下，如何保证不同节点返回相同的 ETag，避免客户端被迫全量更新？

---

### 一、使用 ETag，值是版本号字符串

```typescript
// registry.controller.ts
@Get('skills/latest')
async downloadLatest(@Query() query, @Res() res, @Headers('if-none-match') ifNoneMatch) {
  const version = await registryService.findLatestPublished(skill.id);
  const versionNo = version.versionNo;
  const etag = `"${versionNo}"`;  // ETag = 版本号，如 "1.0.3"
  
  if (ifNoneMatch && ifNoneMatch === etag) {
    res.status(304).end();
    return;
  }
  
  res.setHeader('ETag', etag);
  res.setHeader('Cache-Control', 'max-age=300');
  // 返回文件内容...
}
```

ETag 值直接用版本号字符串（如 `"1.0.3"`），不是文件内容的哈希。

---

### 二、为什么不用 Last-Modified

`Last-Modified` 是时间戳，精度只到秒。如果版本在同一秒内发布（测试环境或批量操作），可能出现时间戳相同但内容不同的情况。版本号（语义化版本）天然是内容的唯一标识，比时间戳更可靠。

此外，版本号对用户友好——调试时直接能看出是哪个版本，不需要解析时间戳。

---

### 三、分布式部署下的 ETag 一致性

**关键问题：** 如果多台服务器部署，ETag 值基于什么计算，不同节点返回的 ETag 会一样吗？

**答案：ETag = 版本号，来自数据库，天然一致。**

`version.versionNo` 是从 MySQL 查出来的，所有节点共享同一个 MySQL 实例（或主从集群）。同一个 Skill 的 latest 版本记录只有一条，所有节点读取到的 `versionNo` 完全相同，因此 ETag 一致，不存在客户端被迫全量更新的问题。

这是选择版本号作为 ETag 而非文件内容哈希（MD5/SHA-256）的重要原因：

| ETag 方案 | 分布式一致性 | 计算开销 |
|---------|------------|---------|
| 版本号字符串 | 来自 DB，天然一致 | 零计算 |
| 文件内容 MD5/SHA-256 | 需每次读文件计算，或缓存 | 读文件 I/O |
| Last-Modified | 来自 DB 的 updatedAt，天然一致 | 零计算，但精度低 |

---

### 四、版本号 ETag 的缺陷与 `fileDigest` 的补充

版本号 ETag 有一个边缘情况：如果服务端代码更新导致同一版本的响应内容发生变化（比如 ZIP 包重新打包但版本号不变），ETag 不会更新，客户端缓存的是旧内容。

这个问题通过 `fileDigest` 字段补充：版本发布审核通过时，异步计算文件的 SHA-256 并存入数据库：

```typescript
// version.service.ts
private async computeAndSaveDigest(versionId: number): Promise<void> {
  const version = await this.versionModel.findByPk(versionId);
  const fileBuffer = await this.storageService.download(version.filePath);
  const hash = createHash('sha256').update(fileBuffer).digest('hex');
  await version.update({ fileDigest: `sha256:${hash}` });
}
```

wellknown index 中暴露 `digest` 字段，CLI 可以校验本地已安装文件的完整性：

```json
{
  "name": "react-best-practices",
  "digest": "sha256:abc123...",
  "url": "/cli/skills/react-best-practices/latest"
}
```

ETag（版本号）负责增量更新的判断，`fileDigest`（SHA-256）负责内容完整性验证，两者互补。

---

### 五、Cache-Control 的配合

```typescript
res.setHeader('Cache-Control', 'max-age=300');  // 5 分钟
```

CDN 或代理层可以缓存 5 分钟，减少服务端压力。5 分钟后 CLI 重新请求，携带 `If-None-Match`，服务端查库返回 ETag 判断是否有更新。对版本更新不频繁的 Skill（通常几天到几周），命中率很高。

---

### 追问

**追问 1：wellknown index 里的 `digest` 和版本表里的 `fileDigest` 是同步更新的吗？如果 `fileDigest` 还没计算完，wellknown index 就被请求了怎么办？**

wellknown index 里只包含 `fileDigest` 不为空的 Skill：

```typescript
// wellknown.service.ts
const version = await this.versionService.findLatestApproved(skill.id);
if (!version?.fileDigest) return null;  // digest 未计算完的版本不进入 index
```

`fileDigest` 是异步计算的（版本审核通过后触发），在计算完成前，该版本不出现在 wellknown index 里——对 CLI 来说相当于该 Skill 暂时不可通过 index 发现。这个时间窗口通常在几秒到几十秒（取决于文件大小），对正常使用无影响。

**追问 2：版本号 ETag 如果服务端做了 hotfix（同版本号但不同内容），用户会持续使用错误内容吗？**

会，直到 `max-age=300`（5 分钟）过期后重新验证，或用户主动运行 `myfe skills update`。这是版本号 ETag 的固有缺陷。正确的处理方式是 hotfix 必须发布新版本号，不允许同版本号重新发布不同内容——这是语义化版本的约定，当前平台没有强制校验，但作为操作规范应当遵守。

**追问 3：不同下载端点（`/latest`、`/:name/latest`、`/:teamSlug/:skillName/latest`）的 ETag 值是相同的吗？**

是相同的，因为 ETag 来源都是同一个 `version.versionNo`，无论通过哪个 URL 访问同一个 Skill 的最新版本，ETag 值一致。这意味着 CLI 缓存的 etag 在切换 URL 格式时仍然有效，304 可以正确触发。
