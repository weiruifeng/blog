---
layout: post
title: "MCP 工具链面试问答（二）：认证与安全"
date: 2025-01-18
categories: 面试
tags: [AI, MCP, CLI, TypeScript, Node.js]
---

> 基于简历「AI Agent 工具链与 MCP 生态建设」项目推导，考察 API Token 哈希存储、Device Flow 原子锁、Token 失效处理等问题。

---

## Q4. API Token 哈希存储与时序安全比较

**问题：** API Token 存储哈希值并通过"时序安全比较"防止时序攻击——你用的是 `crypto.timingSafeEqual`？时序攻击在 Token 验证场景下的实际可利用性有多高，这是理论安全加固还是应对真实威胁？Token 的哈希算法选用的是什么，为何？

---

### 一、实现：timingSafeEqual + SHA-256

代码在 `api-token.service.ts` 的 `verify` 方法：

```typescript
async verify(rawToken: string): Promise<SkillApiToken | null> {
  if (!rawToken || !rawToken.startsWith('sk-')) return null;
  
  const prefix = rawToken.slice(0, 10);  // 前 10 字符作为查找索引
  const hash = createHash('sha256').update(rawToken).digest('hex');
  
  const candidates = await this.tokenModel.findAll({
    where: { tokenPrefix: prefix },
  });
  
  for (const record of candidates) {
    if (!timingSafeEqual(
      Buffer.from(record.tokenHash, 'hex'),
      Buffer.from(hash, 'hex')
    )) continue;
    
    if (record.revokedAt) return null;
    await record.update({ lastUsedAt: new Date() });
    return record;
  }
  return null;
}
```

Token 格式：`sk-<randomBytes(32).toString('hex')>` — 前缀 `sk-` + 64 位十六进制随机数，共 67 字符。

存储时：
- `tokenPrefix`：原始 Token 的前 10 个字符（明文），用于数据库索引缩小候选集
- `tokenHash`：SHA-256 哈希值（64 位十六进制），用于实际比较

---

### 二、时序攻击在这个场景的实际可利用性

时序攻击的前提是：攻击者能够精确测量服务端响应时间，且响应时间与比较的字节数正相关。

**普通字符串比较（`===`）的问题：** JavaScript 的字符串比较会在第一个不匹配字符处短路，攻击者通过枚举第一个字节（256 种）观察响应时间差异，可以逐字节还原哈希值。

**这个场景的实际威胁等级：中等偏高，值得防御。**

- Token 验证接口是网络服务，延迟中有网络抖动，会淹没微秒级时序差异——这是时序攻击的干扰项
- 但攻击者可以通过大量请求统计平均值来消除抖动。现代网络环境下，几万次请求可以建立稳定的时序基线
- SHA-256 的 64 位十六进制哈希是 64 字节，逐字节枚举最多 64 × 256 次请求即可还原——在没有速率限制的情况下，这是可实现的攻击

结论：`timingSafeEqual` 是对真实威胁的防御，不是纯理论加固。但它的有效性依赖于：哈希比较本身恒定时间，网络层无法被消除的抖动提供额外保护。

---

### 三、为什么选 SHA-256 而不是 bcrypt/Argon2

API Token 和密码的安全模型不同：

| 对比维度 | 密码 | API Token |
|---------|------|---------|
| 原始值长度 | 通常 8-20 字符，低熵 | 64 位十六进制随机数，256 bit 熵 |
| 暴力破解可行性 | 可行，需要慢哈希防御 | 不可行，2^256 搜索空间 |
| 验证频率 | 低频（用户登录） | 高频（每次 API 请求） |

因为 Token 本身是高熵随机数（`randomBytes(32)`），不存在字典攻击或暴力破解的可行路径。bcrypt 的慢哈希设计是为了对抗低熵密码的暴力枚举，对高熵 Token 没有必要，还会引入不必要的 CPU 开销。

SHA-256 的选择理由：
1. 高熵 Token 的安全性由随机数本身保证，SHA-256 只是"不存储明文"的要求
2. 验证是高频操作，SHA-256 的速度（微秒级）远优于 bcrypt（百毫秒级）
3. Node.js 内置 `crypto` 模块，无额外依赖

---

### 追问

**追问 1：tokenPrefix 是明文存储的，攻击者拿到数据库后能用 prefix 加速暴力破解吗？**

不能。`tokenPrefix` 的作用是"查找候选行"，不是"辅助暴力破解"。攻击者拿到数据库后只有 `tokenPrefix`（10 字符）和 `tokenHash`（SHA-256）。由于 Token 是 `randomBytes(32)` 生成的高熵随机数，即使知道前 10 字符，剩余部分的搜索空间仍然是 2^(67-10)×8 bit，计算上不可行。

**追问 2：Token 生成后"只返回一次"，如果用户丢失了怎么办？**

只能重新创建。`create` 方法只返回 `raw` token 一次，之后不再能恢复明文（数据库只有哈希）。这是有意为之——与 GitHub PAT 的设计相同，"生成后立刻复制，丢失则重新生成"。系统提供 token 管理界面，用户可以随时撤销旧 token 并创建新的。

---

## Q5. Device Flow 的原子锁防并发兑换

**问题：** Device Flow 中"原子锁防止并发请求重复兑换凭证"——这个原子锁是基于 Redis `SET NX EX` 实现的吗？如果加锁服务端在持有锁期间崩溃，TTL 过期后锁释放，但凭证已经被部分兑换，如何处理这种半提交状态？

---

### 一、原子锁实现：Redis SETNX

锁实现在 `device-flow.service.ts` 的 `pollToken` 方法：

```typescript
async pollToken(deviceCode: string) {
  // 验证 device code 状态...
  if (data.status === 'AUTHORIZED') {
    // 原子锁：SETNX，只允许一个请求成功
    const claimed = await this.redis.setnx(CLAIM_CATEGORY, deviceCode, '1');
    if (!claimed) throw new BadRequestException('device code 已被使用');
    
    try {
      const { token } = await this.apiTokenService.create(data.userId!, {
        tokenName: 'CLI Device Flow',
        scopes: ['read', 'publish'],
      });
      // 兑换成功：清理 code 和 claim lock
      await this.redis.del(CODE_CATEGORY, deviceCode);
      await this.redis.del(CLAIM_CATEGORY, deviceCode);
      return { status: 'success', token };
    } catch (err) {
      // 兑换失败：释放 claim lock，允许重试
      await this.redis.del(CLAIM_CATEGORY, deviceCode);
      throw err;
    }
  }
}
```

Redis `setnx` 是原子操作（SET if Not eXists），并发请求中只有一个能返回 `true`，其余都被拒绝。

---

### 二、Device Code 的 TTL 设计

Device code 存入 Redis 时设置了 15 分钟 TTL（900 秒），CLAIM lock 本身没有单独设置 TTL，依赖 device code 的清理逻辑：

```typescript
// 设置 device code，带 TTL
await this.redis.set(CODE_CATEGORY, code, JSON.stringify({ status: 'PENDING', ... }), 900);

// CLAIM lock 的 key 是 CLAIM_CATEGORY + deviceCode
const CLAIM_CATEGORY = 'device-flow:claim:';
const claimed = await this.redis.setnx(CLAIM_CATEGORY, deviceCode, '1');
```

---

### 三、半提交状态的处理

**崩溃场景：** 服务端在 `apiTokenService.create` 成功后、`redis.del(CLAIM_CATEGORY, ...)` 之前崩溃。

这时的磁盘状态：
- MySQL 里：Token 已创建，状态正常
- Redis 里：CLAIM lock 仍然存在（value='1'），device code 也仍然存在

**后果分析：**

CLAIM lock 没有独立 TTL，它的"过期"依赖 device code 本身的 TTL（最多 15 分钟）。在 device code 的 TTL 过期后，Redis 自动删除 `CODE_CATEGORY:code`，但 `CLAIM_CATEGORY:code` 如果没有 TTL，会永久残留（这是当前实现的一个 bug）。

崩溃期间 CLI 如果重新 poll，setnx 失败（lock 存在），返回"device code 已被使用"。但实际上 Token 已经生成，用户无法获取。

**处理方式：** 坦诚说，当前没有专门的半提交恢复机制。用户遇到这个场景需要重新走 Device Flow（重新授权）。实际可行的改进：

1. 给 CLAIM lock 设置一个较短的 TTL（如 30 秒），锁过期后 CLI 可以重试 poll，服务端检查 Token 是否已创建（幂等性），而不是直接再创建一个
2. 在 `create` Token 时先检查是否已有 `CLI Device Flow` 命名的有效 Token——但这样会引入"重名 Token"的语义问题

目前接受这个风险的理由：Device Flow 崩溃场景的概率极低，用户重新授权的成本不高（整个流程 30 秒内）。

---

### 四、并发 Poll 的保护效果

正常并发场景（无崩溃）：

```
CLI 进程 A: setnx → true  → 创建 Token → 删除 code + lock → 返回 token
CLI 进程 B: setnx → false → throw "device code 已被使用"（400）
CLI 进程 C: setnx → false → 同上
```

`setnx` 的原子性由 Redis 单线程模型保证，不存在两个进程同时获得 `claimed=true` 的情况。

---

### 追问

**追问 1：CLAIM lock 没有 TTL 是个 bug，为什么没有修复？**

是已知问题。实际上触发概率非常低：需要服务端在"已创建 Token、未删除 lock"这个极短的时间窗口内崩溃。在服务正常的情况下，lock 在成功路径上删除，在失败路径上也在 catch 里删除。遗留的 lock key 会占用 Redis 内存，但值是 '1'（1 字节），在 device code TTL 过期后 `CODE_CATEGORY:code` 被 Redis 自动清理，但 `CLAIM_CATEGORY:code` 没有 TTL，确实会残留。正确修复是给 CLAIM lock 也设 TTL（比如 60 秒），兼顾崩溃恢复和内存泄漏。

**追问 2：Device code 轮询期间，CLI 会以什么频率 poll？服务端有没有做 poll 频率限制？**

CLI 侧目前是每 3 秒 poll 一次（参考 OAuth Device Flow 标准推荐的 interval）。服务端没有实现针对 device code 的频率限制，理论上可以无限制 poll——这是一个安全薄弱点，生产场景应该加上每个 device code 的 poll 速率限制（如每 10 秒最多 1 次），避免 device code 枚举攻击。

---

## Q6. Token 失效时 CLI 命令执行状态的处理

**问题：** "Token 失效时内部自动触发重新授权，用户无感知"——在 CLI 命令执行中间发生 Token 失效，重新授权需要打开浏览器，这个过程用户真的无感知吗？你如何处理 CLI 当前命令的执行状态——是重试还是回滚？

---

### 一、实现现状：有感知，非无感知

先说实情：**并非真正的"用户无感知"**，更准确的描述是"Token 失效不会让命令直接报错崩溃"，而是会暂停当前操作、引导用户重新授权。

Token 失效的处理在 `SkillHubClient` 的请求拦截层：

```typescript
// skill-hub-client.ts
private async request<T>(method: string, path: string, ...): Promise<T> {
  const token = await this.credentialsStore.getToken();
  
  try {
    const response = await axios({ headers: { Authorization: `Bearer ${token}` }, ... });
    return response.data;
  } catch (err) {
    if (axios.isAxiosError(err) && err.response?.status === 401) {
      // Token 失效：清除本地凭证，抛出特定错误
      await this.credentialsStore.clearToken();
      throw new TokenExpiredError('Token 已失效，请重新登录');
    }
    throw err;
  }
}
```

上层命令（如 `skills install`）捕获 `TokenExpiredError` 后，提示用户运行 `myfe skills login` 重新授权：

```typescript
// skills.ts
} catch (err) {
  if (err instanceof TokenExpiredError) {
    logger.error('认证已过期，请运行 myfe skills login 重新登录');
    process.exit(1);
  }
  throw err;
}
```

---

### 二、当前命令的执行状态处理

**结论：命令中止，不重试，不回滚，用户需要重新执行命令。**

这背后的设计取舍：

**不自动重试（不打开浏览器）的理由：**

CLI 命令是同步交互模式，在命令执行中途自动打开浏览器会让用户困惑（"我在装一个 Skill，为什么突然弹出登录页？"）。重新授权需要打开浏览器 + 用户在网页上点击确认，这是不可能"无感知"的操作。

**不回滚的理由：**

Token 失效通常发生在 HTTP 请求阶段（下载 Skill 前），此时本地文件系统还未修改，没有需要回滚的状态。如果 Token 在安装进行到一半时失效（比如下载完成、写文件时 API 返回 401），已复制的文件会留在磁盘——下次重新执行安装时，因为文件复制是幂等的（覆盖写），结果仍然正确。

**实际上"无感知"的场景：**

系统设计里更接近"无感知"的场景是 Token 刷新（不是 Token 失效）。如果引入了 refresh token 机制，访问 token 过期时 SDK 自动用 refresh token 换新的访问 token，用户确实无感知。但当前实现使用的是长期有效的 Personal Access Token（无 refresh 机制），一旦 revoke 或过期，必须用户手动重新授权。

---

### 三、Token 生命周期管理

Token 存储在 `~/.myfe-cli/credentials.json`（权限 0o600）：

```typescript
// credentials-store.ts
export class CredentialsStore {
  private credentialsPath = path.join(os.homedir(), '.myfe-cli', 'credentials.json');
  
  async getToken(): Promise<string | null> {
    try {
      const creds = await fs.readJSON(this.credentialsPath);
      return creds.token ?? null;
    } catch { return null; }
  }
  
  async saveToken(token: string): Promise<void> {
    await fs.ensureDir(path.dirname(this.credentialsPath));
    await fs.writeJSON(this.credentialsPath, { token, savedAt: Date.now() });
    await fs.chmod(this.credentialsPath, 0o600);  // 仅当前用户可读写
  }
  
  async clearToken(): Promise<void> {
    try { await fs.remove(this.credentialsPath); } catch {}
  }
}
```

Token 没有本地过期机制，有效性完全由服务端判断。

---

### 追问

**追问 1：用户运行 `myfe skills install` 失败后重新登录，再运行 install 命令，会重复安装吗？**

不会重复产生副作用，但会重新执行安装流程。文件复制是幂等的（覆盖写），settings.json 合并有 Set 去重，CLAUDE.md 追加有 marker 检查——重新执行结果和第一次成功执行完全相同。唯一有差异的是 download count：每次下载会在服务端 `+1`，但加 `?nocount=true` 参数可以跳过。更新检查（`myfe skills update`）用 etag 判断是否需要重下，不会重复下载相同版本。

**追问 2：为什么用 Personal Access Token 而不是短期 Access Token + Refresh Token？**

主要是简化实现。PAT（Personal Access Token）模式是 CLI 工具的成熟方案（GitHub CLI、npm token 等都是这个模式），生命周期由用户手动管理（可以随时 revoke），服务端无需维护 refresh token 表和刷新逻辑。对于 Skill Hub 这个内部工具，安全要求没有到需要引入 OAuth Access Token + Refresh Token 的程度。如果未来需要更细粒度的 token 管理（如按机器/会话颁发短期 token），Device Flow 的流程本身已经支持，只需要在后端把 token 生命周期改成短期 + 引入 refresh 机制即可。
