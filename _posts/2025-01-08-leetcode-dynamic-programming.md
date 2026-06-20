---
layout: post
title: "LeetCode 高频面试题（四）：动态规划"
date: 2025-01-08
categories: 面试
tags: [算法, LeetCode, TypeScript, 高频题, 动态规划]
---

> 本篇专注动态规划（DP），涵盖一维 DP、二维 DP、子序列、背包问题、股票交易等经典类型，含完整 TypeScript 实现。
> 难度标注：⭐ Easy ｜ ⭐⭐ Medium ｜ ⭐⭐⭐ Hard

---

## 动态规划概述

动态规划在查找有很多**重叠子问题**的情况的最优解时有效。解题关键是找到**状态转移方程**，通过计算和存储子问题的解来求解最终问题。

---

## 一、基本动态规划（一维）

### 70. Climbing Stairs ⭐

**题目：** 给定 n 节台阶，每次可以走一步或两步，求一共有多少种方式可以走完这些台阶。

**状态转移：** `dp[i] = dp[i-1] + dp[i-2]`（第 i 阶可以从第 i-1 或 i-2 阶到达）

空间可压缩到 O(1)，使用两个变量滚动计算。

```typescript
function climbStairs(n: number): number {
  if (n <= 2) return n;
  let pre2 = 1, pre1 = 2;
  for (let i = 2; i < n; i++) {
    const cur = pre1 + pre2;
    pre2 = pre1;
    pre1 = cur;
  }
  return pre1;
}
```

---

### 198. House Robber ⭐

**题目：** 一排房子，每个房子有一定钱财。如果抢了两栋相邻的房子则触发警报。求最多可以抢劫多少钱。

**状态转移：** `dp[i] = max(dp[i-1], nums[i-1] + dp[i-2])`（不抢当前房子 vs 抢当前房子）

```typescript
function rob(nums: number[]): number {
  if (!nums.length) return 0;
  if (nums.length === 1) return nums[0];
  let pre2 = 0, pre1 = 0;
  for (const num of nums) {
    const cur = Math.max(pre2 + num, pre1);
    pre2 = pre1;
    pre1 = cur;
  }
  return pre1;
}
```

---

### 413. Arithmetic Slices ⭐⭐

**题目：** 给定一个数组，求其中连续且等差的子数组个数。

**状态转移：** `dp[i]` 表示以 i 结尾的等差子数组个数。若 `nums[i]-nums[i-1] == nums[i-1]-nums[i-2]`，则 `dp[i] = dp[i-1] + 1`。

```typescript
function numberOfArithmeticSlices(nums: number[]): number {
  const n = nums.length;
  if (n < 3) return 0;
  const dp = new Array(n).fill(0);
  for (let i = 2; i < n; i++) {
    if (nums[i] - nums[i - 1] === nums[i - 1] - nums[i - 2]) {
      dp[i] = dp[i - 1] + 1;
    }
  }
  return dp.reduce((sum, val) => sum + val, 0);
}
```

---

## 二、基本动态规划（二维）

### 64. Minimum Path Sum ⭐⭐

**题目：** 给定 m×n 的非负整数矩阵，求从左上角到右下角、经过数字之和最小的路径（每次只能向右或向下移动）。

**状态转移：** `dp[i][j] = min(dp[i-1][j], dp[i][j-1]) + grid[i][j]`，可以压缩为一维数组。

```typescript
function minPathSum(grid: number[][]): number {
  const m = grid.length, n = grid[0].length;
  const dp = new Array(n).fill(0);
  for (let i = 0; i < m; i++) {
    for (let j = 0; j < n; j++) {
      if (i === 0 && j === 0) dp[j] = grid[0][0];
      else if (i === 0) dp[j] = dp[j - 1] + grid[i][j];
      else if (j === 0) dp[j] = dp[j] + grid[i][j];
      else dp[j] = Math.min(dp[j], dp[j - 1]) + grid[i][j];
    }
  }
  return dp[n - 1];
}
```

---

### 221. Maximal Square ⭐⭐

**题目：** 给定一个二维 0-1 矩阵，求全由 1 构成的最大正方形面积。

**状态转移：** `dp[i][j]` 表示以 (i,j) 为右下角的全 1 最大正方形边长。若当前为 '1'，则 `dp[i][j] = min(dp[i-1][j-1], dp[i][j-1], dp[i-1][j]) + 1`。

```typescript
function maximalSquare(matrix: string[][]): number {
  if (!matrix.length || !matrix[0].length) return 0;
  const m = matrix.length, n = matrix[0].length;
  const dp: number[][] = Array.from({ length: m + 1 }, () => new Array(n + 1).fill(0));
  let maxSide = 0;
  for (let i = 1; i <= m; i++) {
    for (let j = 1; j <= n; j++) {
      if (matrix[i - 1][j - 1] === '1') {
        dp[i][j] = Math.min(dp[i - 1][j - 1], Math.min(dp[i][j - 1], dp[i - 1][j])) + 1;
        maxSide = Math.max(maxSide, dp[i][j]);
      }
    }
  }
  return maxSide * maxSide;
}
```

---

## 三、子序列问题

### 300. Longest Increasing Subsequence ⭐⭐

**题目：** 给定未排序的整数数组，求最长的递增子序列长度（子序列不必连续）。

**O(n log n) 解法：** 维护一个始终递增的 dp 数组，对每个数字用二分查找确定其插入位置。

```typescript
function lengthOfLIS(nums: number[]): number {
  const dp: number[] = [];
  for (const num of nums) {
    let lo = 0, hi = dp.length;
    while (lo < hi) {
      const mid = (lo + hi) >> 1;
      if (dp[mid] < num) lo = mid + 1;
      else hi = mid;
    }
    dp[lo] = num;
  }
  return dp.length;
}
```

---

### 1143. Longest Common Subsequence ⭐⭐

**题目：** 给定两个字符串，求它们最长的公共子序列长度。

**状态转移：**
- 若 `text1[i-1] === text2[j-1]`，则 `dp[i][j] = dp[i-1][j-1] + 1`
- 否则 `dp[i][j] = max(dp[i-1][j], dp[i][j-1])`

```typescript
function longestCommonSubsequence(text1: string, text2: string): number {
  const m = text1.length, n = text2.length;
  const dp: number[][] = Array.from({ length: m + 1 }, () => new Array(n + 1).fill(0));
  for (let i = 1; i <= m; i++) {
    for (let j = 1; j <= n; j++) {
      if (text1[i - 1] === text2[j - 1]) dp[i][j] = dp[i - 1][j - 1] + 1;
      else dp[i][j] = Math.max(dp[i - 1][j], dp[i][j - 1]);
    }
  }
  return dp[m][n];
}
```

---

## 四、背包问题

### 416. Partition Equal Subset Sum ⭐⭐（0-1 背包）

**题目：** 给定正整数数组，求是否可以把数组分成和相等的两部分。

**思路：** 等价于 0-1 背包——选取一部分数字使其和为 `sum/2`。使用一维布尔数组，**逆向遍历**（0-1 背包特征，防止重复选取）。

```typescript
function canPartition(nums: number[]): boolean {
  const sum = nums.reduce((a, b) => a + b, 0);
  if (sum % 2 !== 0) return false;
  const target = sum / 2;
  const dp = new Array(target + 1).fill(false);
  dp[0] = true;
  for (const num of nums) {
    for (let j = target; j >= num; j--) {
      dp[j] = dp[j] || dp[j - num];
    }
  }
  return dp[target];
}
```

---

### 322. Coin Change ⭐⭐（完全背包）

**题目：** 给定一些硬币面额，求最少可以用多少枚硬币组成给定金额。若不存在解则返回 -1。

**思路：** 完全背包——每枚硬币可以使用无限次。**正向遍历**（完全背包特征，允许重复选取）。

```typescript
function coinChange(coins: number[], amount: number): number {
  const dp = new Array(amount + 1).fill(amount + 1);
  dp[0] = 0;
  for (let i = 1; i <= amount; i++) {
    for (const coin of coins) {
      if (i >= coin) dp[i] = Math.min(dp[i], dp[i - coin] + 1);
    }
  }
  return dp[amount] === amount + 1 ? -1 : dp[amount];
}
```

---

## 五、字符串编辑

### 72. Edit Distance ⭐⭐⭐

**题目：** 给定两个字符串，可以删除、替换和插入任意字符，求最少编辑几步可以将两个字符串变成相同。

**状态转移：** `dp[i][j]` 表示将 `word1[0..i-1]` 变成 `word2[0..j-1]` 的最少步骤：
- 相同：`dp[i][j] = dp[i-1][j-1]`
- 不同：`dp[i][j] = min(替换, 插入, 删除) + 1 = min(dp[i-1][j-1], dp[i-1][j], dp[i][j-1]) + 1`

```typescript
function minDistance(word1: string, word2: string): number {
  const m = word1.length, n = word2.length;
  const dp: number[][] = Array.from({ length: m + 1 }, () => new Array(n + 1).fill(0));
  for (let i = 0; i <= m; i++) dp[i][0] = i;
  for (let j = 0; j <= n; j++) dp[0][j] = j;
  for (let i = 1; i <= m; i++) {
    for (let j = 1; j <= n; j++) {
      if (word1[i - 1] === word2[j - 1]) {
        dp[i][j] = dp[i - 1][j - 1];
      } else {
        dp[i][j] = Math.min(dp[i - 1][j - 1], Math.min(dp[i - 1][j], dp[i][j - 1])) + 1;
      }
    }
  }
  return dp[m][n];
}
```

---

## 六、股票交易

### 121. Best Time to Buy and Sell Stock ⭐（只买卖一次）

**思路：** 遍历数组，维护历史最低价格，每天计算若今天卖出的收益是否最大。

```typescript
function maxProfit(prices: number[]): number {
  let sell = 0, buy = -Infinity;
  for (const price of prices) {
    buy = Math.max(buy, -price);
    sell = Math.max(sell, buy + price);
  }
  return sell;
}
```

---

### 188. Best Time to Buy and Sell Stock IV ⭐⭐⭐（最多买卖 k 次）

**思路：** 若 k 大于总天数的一半则可以无限交易。否则建立 buy[j]（第 j 次买入最大收益）和 sell[j]（第 j 次卖出最大收益）两个数组。

```typescript
function maxProfitK(k: number, prices: number[]): number {
  const days = prices.length;
  if (days < 2) return 0;
  if (k * 2 >= days) {
    let maxProfit = 0;
    for (let i = 1; i < days; i++) {
      if (prices[i] > prices[i - 1]) maxProfit += prices[i] - prices[i - 1];
    }
    return maxProfit;
  }
  const buy = new Array(k + 1).fill(-Infinity);
  const sell = new Array(k + 1).fill(0);
  for (const price of prices) {
    for (let j = 1; j <= k; j++) {
      buy[j] = Math.max(buy[j], sell[j - 1] - price);
      sell[j] = Math.max(sell[j], buy[j] + price);
    }
  }
  return sell[k];
}
```

---

### 139. Word Break ⭐⭐

**题目：** 给定一个字符串和字符串集合，求是否存在分割方式使得所有子字符串都在集合中。

**状态转移：** `dp[i]` 表示到位置 i 为止是否可以成功分割。

```typescript
function wordBreak(s: string, wordDict: string[]): boolean {
  const n = s.length;
  const dp = new Array(n + 1).fill(false);
  dp[0] = true;
  for (let i = 1; i <= n; i++) {
    for (const word of wordDict) {
      const len = word.length;
      if (i >= len && s.substring(i - len, i) === word) {
        dp[i] = dp[i] || dp[i - len];
      }
    }
  }
  return dp[n];
}
```

---

### 542. 01 Matrix ⭐⭐

**题目：** 给定一个由 0 和 1 组成的二维矩阵，求每个位置到最近的 0 的距离。

**思路：** 两次动态搜索——先从左上到右下，再从右下到左上，两次搜索完成四个方向上的查找。

```typescript
function updateMatrix(matrix: number[][]): number[][] {
  const n = matrix.length, m = matrix[0].length;
  const dp: number[][] = Array.from({ length: n }, () => new Array(m).fill(Infinity));
  for (let i = 0; i < n; i++) {
    for (let j = 0; j < m; j++) {
      if (matrix[i][j] === 0) dp[i][j] = 0;
      else {
        if (j > 0) dp[i][j] = Math.min(dp[i][j], dp[i][j - 1] + 1);
        if (i > 0) dp[i][j] = Math.min(dp[i][j], dp[i - 1][j] + 1);
      }
    }
  }
  for (let i = n - 1; i >= 0; i--) {
    for (let j = m - 1; j >= 0; j--) {
      if (matrix[i][j] !== 0) {
        if (j < m - 1) dp[i][j] = Math.min(dp[i][j], dp[i][j + 1] + 1);
        if (i < n - 1) dp[i][j] = Math.min(dp[i][j], dp[i + 1][j] + 1);
      }
    }
  }
  return dp;
}
```

---

### 279. Perfect Squares ⭐⭐

**题目：** 给定一个正整数，求其最少可以由几个完全平方数相加构成。

**状态转移：** `dp[i] = 1 + min(dp[i-1], dp[i-4], dp[i-9], ...)`

```typescript
function numSquares(n: number): number {
  const dp = new Array(n + 1).fill(Infinity);
  dp[0] = 0;
  for (let i = 1; i <= n; i++) {
    for (let j = 1; j * j <= i; j++) {
      dp[i] = Math.min(dp[i], dp[i - j * j] + 1);
    }
  }
  return dp[n];
}
```

---

### 91. Decode Ways ⭐⭐

**题目：** 字母 A-Z 可以表示成数字 1-26。给定一个数字串，求有多少种不同的解码方式。

**思路：** 注意特殊情况——数字 0 无法单独解码，相邻两数字超过 26 时无法组合解码。

```typescript
function numDecodings(s: string): number {
  const n = s.length;
  if (n === 0 || s[0] === '0') return 0;
  const dp = new Array(n + 1).fill(0);
  dp[0] = 1; dp[1] = 1;
  for (let i = 2; i <= n; i++) {
    const one = parseInt(s[i - 1]);
    const two = parseInt(s.substring(i - 2, i));
    if (one >= 1) dp[i] += dp[i - 1];
    if (two >= 10 && two <= 26) dp[i] += dp[i - 2];
  }
  return dp[n];
}
```

---

### 474. Ones and Zeroes ⭐⭐（多维背包）

**题目：** 给定 m 个 0 和 n 个 1，以及一些由 0-1 构成的字符串，求最多可以构成多少个给定字符串（每串只能构成一次）。

**思路：** 多维费用的 0-1 背包，有两个背包大小（0 的数量和 1 的数量）。

```typescript
function findMaxForm(strs: string[], m: number, n: number): number {
  const dp: number[][] = Array.from({ length: m + 1 }, () => new Array(n + 1).fill(0));
  for (const str of strs) {
    let count0 = 0, count1 = 0;
    for (const c of str) { if (c === '1') count1++; else count0++; }
    for (let i = m; i >= count0; i--) {
      for (let j = n; j >= count1; j--) {
        dp[i][j] = Math.max(dp[i][j], 1 + dp[i - count0][j - count1]);
      }
    }
  }
  return dp[m][n];
}
```

---

### 650. 2 Keys Keyboard ⭐⭐

**题目：** 给定一个字母 A，每次可以复制全部字符或粘贴已复制的字符，求最少需要几次操作把字符串延展到指定长度。

**思路：** `dp[i]` 表示延展到长度 i 的最少操作次数。对每个能整除 i 的因子 j：`dp[i] = dp[j] + dp[i/j]`。

```typescript
function minSteps(n: number): number {
  const dp = new Array(n + 1).fill(0);
  for (let i = 2; i <= n; i++) {
    dp[i] = i;
    for (let j = 2; j * j <= i; j++) {
      if (i % j === 0) { dp[i] = dp[j] + dp[i / j]; break; }
    }
  }
  return dp[n];
}
```

---

### 10. Regular Expression Matching ⭐⭐⭐

**题目：** 给定字符串和正则表达式（只含 `.` 和 `*`），求字符串是否可以被匹配。

**思路：** `dp[i][j]` 表示字符串前 i 位是否能被正则前 j 位匹配。根据 `*`、`.`、普通字符三种情况分别处理。

```typescript
function isMatch(s: string, p: string): boolean {
  const m = s.length, n = p.length;
  const dp: boolean[][] = Array.from({ length: m + 1 }, () => new Array(n + 1).fill(false));
  dp[0][0] = true;
  for (let i = 1; i < n + 1; i++) {
    if (p[i - 1] === '*') dp[0][i] = dp[0][i - 2];
  }
  for (let i = 1; i <= m; i++) {
    for (let j = 1; j <= n; j++) {
      if (p[j - 1] === '.') {
        dp[i][j] = dp[i - 1][j - 1];
      } else if (p[j - 1] !== '*') {
        dp[i][j] = dp[i - 1][j - 1] && p[j - 1] === s[i - 1];
      } else if (p[j - 2] !== s[i - 1] && p[j - 2] !== '.') {
        dp[i][j] = dp[i][j - 2];
      } else {
        dp[i][j] = dp[i][j - 1] || dp[i - 1][j] || dp[i][j - 2];
      }
    }
  }
  return dp[m][n];
}
```

---

### 309. Best Time to Buy and Sell Stock with Cooldown ⭐⭐

**题目：** 每次卖出后必须冷却一天，求最大收益。

**思路：** 状态机——建立四个状态 buy、sell、s1（冷却中）、s2（冷却后可买）及其转移方式。

```typescript
function maxProfitCooldown(prices: number[]): number {
  const n = prices.length;
  if (n === 0) return 0;
  const buy = new Array(n).fill(0);
  const sell = new Array(n).fill(0);
  const s1 = new Array(n).fill(0);
  const s2 = new Array(n).fill(0);
  s1[0] = buy[0] = -prices[0];
  sell[0] = s2[0] = 0;
  for (let i = 1; i < n; i++) {
    buy[i] = s2[i - 1] - prices[i];
    s1[i] = Math.max(buy[i - 1], s1[i - 1]);
    sell[i] = Math.max(buy[i - 1], s1[i - 1]) + prices[i];
    s2[i] = Math.max(s2[i - 1], sell[i - 1]);
  }
  return Math.max(sell[n - 1], s2[n - 1]);
}
```

---

## 总结：背包问题选择指南

| 题型 | 遍历顺序 | 典型题目 |
|------|---------|---------|
| 0-1 背包（每种物品只能选一次）| 逆向遍历 | 416 分割等和子集 |
| 完全背包（每种物品可选无限次）| 正向遍历 | 322 零钱兑换 |
| 多维背包（有多个限制条件）| 逆向多维遍历 | 474 一和零 |

## 练习推荐

**基础：** 53（最大子数组和）、213（打家劫舍 II）、343（整数拆分）

**子序列：** 583（两个字符串的删除操作）、646（最长数对链）

**背包：** 494（目标和）、完全背包变体

**股票：** 309（含冷冻期买卖股票）、714（含手续费买卖股票）
