---
layout: post
title: "LeetCode 高频面试题（二）：排序 · 分治 · 数学 · 位运算"
date: 2025-01-06
categories: 面试
tags: [算法, LeetCode, TypeScript, 高频题]
---

> 本篇覆盖排序算法、分治法、数学题和位运算四大主题，含完整 TypeScript 实现。
> 难度标注：⭐ Easy ｜ ⭐⭐ Medium ｜ ⭐⭐⭐ Hard

---

## 一、排序算法

实际刷题时很少需要手写排序（直接用 `Array.prototype.sort()`），但熟悉各种排序算法可以解决由此引申的题目。

### 基础排序实现

```typescript
// 快速排序（左闭右开）
function quickSort(nums: number[], l: number, r: number): void {
  if (l + 1 >= r) return;
  let first = l, last = r - 1, key = nums[first];
  while (first < last) {
    while (first < last && nums[last] >= key) last--;
    nums[first] = nums[last];
    while (first < last && nums[first] <= key) first++;
    nums[last] = nums[first];
  }
  nums[first] = key;
  quickSort(nums, l, first);
  quickSort(nums, first + 1, r);
}

// 归并排序
function mergeSort(nums: number[], l: number, r: number, temp: number[]): void {
  if (l + 1 >= r) return;
  const m = l + Math.floor((r - l) / 2);
  mergeSort(nums, l, m, temp);
  mergeSort(nums, m, r, temp);
  let p = l, q = m, i = l;
  while (p < m || q < r) {
    if (q >= r || (p < m && nums[p] <= nums[q])) temp[i++] = nums[p++];
    else temp[i++] = nums[q++];
  }
  for (let i = l; i < r; i++) nums[i] = temp[i];
}
```

**各排序算法时间复杂度对比：**

| 算法 | 平均 | 最差 | 空间 | 稳定性 |
|------|------|------|------|--------|
| 快速排序 | O(n log n) | O(n²) | O(log n) | 不稳定 |
| 归并排序 | O(n log n) | O(n log n) | O(n) | 稳定 |
| 插入排序 | O(n²) | O(n²) | O(1) | 稳定 |
| 冒泡排序 | O(n²) | O(n²) | O(1) | 稳定 |
| 选择排序 | O(n²) | O(n²) | O(1) | 不稳定 |

---

### 215. Kth Largest Element in an Array ⭐⭐（快速选择）

**题目：** 在未排序数组中，找到第 k 大的数字。

**思路：** 快速选择——类似快速排序，但只需找到第 k 大的枢（pivot），不需要对两侧继续排序。平均时间复杂度 O(n)，空间复杂度 O(1)。

```typescript
function findKthLargest(nums: number[], k: number): number {
  const target = nums.length - k;
  let l = 0, r = nums.length - 1;
  while (l < r) {
    const mid = quickSelection(nums, l, r);
    if (mid === target) return nums[mid];
    else if (mid < target) l = mid + 1;
    else r = mid - 1;
  }
  return nums[l];
}

function quickSelection(nums: number[], l: number, r: number): number {
  let i = l + 1, j = r;
  while (true) {
    while (i < r && nums[i] <= nums[l]) i++;
    while (l < j && nums[j] >= nums[l]) j--;
    if (i >= j) break;
    [nums[i], nums[j]] = [nums[j], nums[i]];
  }
  [nums[l], nums[j]] = [nums[j], nums[l]];
  return j;
}
```

---

### 347. Top K Frequent Elements ⭐⭐（桶排序）

**题目：** 给定一个数组，求前 k 个最频繁的数字。

**思路：** 桶排序。先统计每个数字出现频次，再以频次为索引建立桶，最后从高频桶向低频桶收集 k 个元素。

```typescript
function topKFrequent(nums: number[], k: number): number[] {
  const counts = new Map<number, number>();
  let maxCount = 0;
  for (const num of nums) {
    const cnt = (counts.get(num) ?? 0) + 1;
    counts.set(num, cnt);
    maxCount = Math.max(maxCount, cnt);
  }
  const buckets: number[][] = Array.from({ length: maxCount + 1 }, () => []);
  for (const [num, cnt] of counts) {
    buckets[cnt].push(num);
  }
  const ans: number[] = [];
  for (let i = maxCount; i >= 0 && ans.length < k; i--) {
    for (const num of buckets[i]) {
      ans.push(num);
      if (ans.length === k) break;
    }
  }
  return ans;
}
```

---

## 二、分治法

分治问题由"分"（divide）和"治"（conquer）两部分组成，通过把原问题分为子问题，再将子问题合并，从而求解原问题。

**主定理**（分析分治时间复杂度）：对于 `T(n) = aT(n/b) + f(n)`，定义 `k = log_b(a)`：
- `f(n) = O(n^p)` 且 `p < k`：`T(n) = O(n^k)`
- `f(n) = O(n^k log^c n)`：`T(n) = O(n^k log^(c+1) n)`
- `f(n) = O(n^p)` 且 `p > k`：`T(n) = O(f(n))`

---

### 241. Different Ways to Add Parentheses ⭐⭐

**题目：** 给定一个只包含加、减和乘法的表达式，求通过加括号可以得到多少种不同的结果。

**思路：** 对于每个运算符，先递归处理两侧的表达式，再处理此运算符。使用 Memoization 避免重复计算。

```typescript
function diffWaysToCompute(input: string): number[] {
  const memo = new Map<string, number[]>();

  function compute(s: string): number[] {
    if (memo.has(s)) return memo.get(s)!;
    const ways: number[] = [];
    for (let i = 0; i < s.length; i++) {
      const c = s[i];
      if (c === '+' || c === '-' || c === '*') {
        const left = compute(s.substring(0, i));
        const right = compute(s.substring(i + 1));
        for (const l of left) {
          for (const r of right) {
            if (c === '+') ways.push(l + r);
            else if (c === '-') ways.push(l - r);
            else ways.push(l * r);
          }
        }
      }
    }
    if (ways.length === 0) ways.push(parseInt(s));
    memo.set(s, ways);
    return ways;
  }

  return compute(input);
}
```

---

## 三、数学问题

### 公倍数与公因数

```typescript
// 辗转相除法求最大公因数
function gcd(a: number, b: number): number {
  return b === 0 ? a : gcd(b, a % b);
}

// 最小公倍数
function lcm(a: number, b: number): number {
  return Math.floor(a * b / gcd(a, b));
}
```

### 204. Count Primes ⭐（埃氏筛法）

**题目：** 给定一个数字 n，求小于 n 的质数的个数。

**思路：** 埃拉托斯特尼筛法——从 2 开始遍历，把所有是当前数字的倍数的整数标为合数；遍历完成后，未被标为合数的数字即为质数。只需遍历到 √n 即可。

```typescript
function countPrimes(n: number): number {
  if (n <= 2) return 0;
  const prime = new Array(n).fill(true);
  prime[0] = prime[1] = false;
  let count = Math.floor(n / 2);
  let i = 3;
  const sqrtn = Math.sqrt(n);
  while (i <= sqrtn) {
    if (prime[i]) {
      for (let j = i * i; j < n; j += 2 * i) {
        if (prime[j]) { prime[j] = false; count--; }
      }
    }
    i += 2;
  }
  return count;
}
```

---

### 415. Add Strings ⭐

**题目：** 给定两个由数字组成的字符串，求它们相加的结果（不能转 number 类型）。

**思路：** 从末尾开始逐位相加，处理进位，注意位数差。

```typescript
function addStrings(num1: string, num2: string): string {
  let i = num1.length - 1, j = num2.length - 1;
  let carry = 0, ans = '';
  while (i >= 0 || j >= 0 || carry > 0) {
    const a = i >= 0 ? parseInt(num1[i--]) : 0;
    const b = j >= 0 ? parseInt(num2[j--]) : 0;
    const sum = a + b + carry;
    ans = (sum % 10).toString() + ans;
    carry = Math.floor(sum / 10);
  }
  return ans;
}
```

---

### 384. Shuffle an Array ⭐⭐（Fisher-Yates 洗牌算法）

**思路：** 经典洗牌算法——从后往前，每次随机交换当前位置与其前面某个位置的元素，保证每种排列概率相等。

```typescript
class Solution {
  private origin: number[];
  constructor(nums: number[]) { this.origin = [...nums]; }
  reset(): number[] { return [...this.origin]; }
  shuffle(): number[] {
    const shuffled = [...this.origin];
    for (let i = shuffled.length - 1; i >= 0; i--) {
      const j = Math.floor(Math.random() * (i + 1));
      [shuffled[i], shuffled[j]] = [shuffled[j], shuffled[i]];
    }
    return shuffled;
  }
}
```

---

## 四、位运算

常用位运算技巧：

```
x ^ x = 0       // 异或自身为 0
x ^ 0 = x       // 异或 0 为自身
n & (n-1)       // 去除最低位的 1
n & (-n)        // 获得最低位的 1
```

### 461. Hamming Distance ⭐

**题目：** 给定两个十进制数字，求它们二进制表示的汉明距离（不同位的个数）。

**思路：** 对两数按位异或，统计结果中 1 的个数。

```typescript
function hammingDistance(x: number, y: number): number {
  let diff = x ^ y, ans = 0;
  while (diff) {
    ans += diff & 1;
    diff >>= 1;
  }
  return ans;
}
```

---

### 136. Single Number ⭐

**题目：** 数组中只有一个数字出现了一次，其余都出现了两次，求这个只出现一次的数字。

**思路：** 利用 `x ^ x = 0`，将所有数字按位异或，出现两次的互相抵消，剩下的即为答案。

```typescript
function singleNumber(nums: number[]): number {
  let ans = 0;
  for (const num of nums) ans ^= num;
  return ans;
}
```

---

### 338. Counting Bits ⭐⭐

**题目：** 给定非负整数 n，求从 0 到 n 的每个数字的二进制表达中分别有多少个 1。

**思路：** 动态规划 + 位运算。若最后一位为 1，则 `dp[i] = dp[i-1] + 1`；若最后一位为 0，则 `dp[i] = dp[i >> 1]`（算术右移等于除以 2）。

```typescript
function countBits(num: number): number[] {
  const dp = new Array(num + 1).fill(0);
  for (let i = 1; i <= num; i++) {
    dp[i] = i & 1 ? dp[i - 1] + 1 : dp[i >> 1];
  }
  return dp;
}
```

---

### 342. Power of Four ⭐

**题目：** 给定一个整数，判断它是否是 4 的次方。

**思路：** 首先判断是否是 2 的次方（`n & (n-1) === 0`），然后判断 1 是否在奇数位（与 `0x55555555 = 1431655765` 按位与不为 0）。

```typescript
function isPowerOfFour(n: number): boolean {
  return n > 0 && (n & (n - 1)) === 0 && (n & 1431655765) !== 0;
}
```

---

### 504. Base 7 ⭐

**题目：** 给定一个十进制整数，求它在七进制下的表示（注意负数和零的处理）。

```typescript
function convertToBase7(num: number): string {
  if (num === 0) return '0';
  const isNegative = num < 0;
  if (isNegative) num = -num;
  let ans = '';
  while (num > 0) {
    ans = (num % 7).toString() + ans;
    num = Math.floor(num / 7);
  }
  return isNegative ? '-' + ans : ans;
}
```

---

### 172. Factorial Trailing Zeroes ⭐⭐

**题目：** 给定一个非负整数，判断它的阶乘结果的结尾有几个 0。

**思路：** 每个尾零来自 `2×5=10`。质因子 2 远多于 5，只需递归统计阶乘结果里有多少个质因子 5。

```typescript
function trailingZeroes(n: number): number {
  return n === 0 ? 0 : Math.floor(n / 5) + trailingZeroes(Math.floor(n / 5));
}
```

---

### 326. Power of Three ⭐

**题目：** 判断一个数字是否是 3 的次方。

**方法（整除）：** int 范围内 3 的最大次方是 `3¹⁹ = 1162261467`，若 `1162261467 % n === 0` 则是。

```typescript
function isPowerOfThree(n: number): boolean {
  return n > 0 && 1162261467 % n === 0;
}
```

---

### 528. Random Pick with Weight ⭐⭐

**题目：** 给定权重数组，按权重概率随机采样位置。

**思路：** 前缀和 + 二分查找。先建立前缀和（单调递增），每次随机产生一个数后二分查找其在前缀和中的位置。

```typescript
class WeightedRandom {
  private sums: number[];
  constructor(weights: number[]) {
    this.sums = [];
    let total = 0;
    for (const w of weights) { total += w; this.sums.push(total); }
  }
  pickIndex(): number {
    const total = this.sums[this.sums.length - 1];
    const pos = Math.floor(Math.random() * total) + 1;
    let l = 0, r = this.sums.length;
    while (l < r) {
      const mid = l + Math.floor((r - l) / 2);
      if (this.sums[mid] >= pos) r = mid;
      else l = mid + 1;
    }
    return l;
  }
}
```

---

### 190. Reverse Bits ⭐

**题目：** 给定一个十进制整数，输出它在二进制下的翻转结果。

```typescript
function reverseBits(n: number): number {
  let ans = 0;
  for (let i = 0; i < 32; i++) {
    ans = (ans * 2 + (n & 1)) >>> 0;
    n >>= 1;
  }
  return ans >>> 0;
}
```

---

### 318. Maximum Product of Word Lengths ⭐⭐

**题目：** 给定多个字符串，求任意两个不含相同字母的字符串的长度乘积的最大值。

**思路：** 为每个字符串建立长度为 26 的二进制掩码，两字符串含相同字母则按位与不为 0。

```typescript
function maxProduct(words: string[]): number {
  const masks = new Map<number, number>();
  let ans = 0;
  for (const word of words) {
    let mask = 0;
    for (const c of word) mask |= 1 << (c.charCodeAt(0) - 97);
    masks.set(mask, Math.max(masks.get(mask) ?? 0, word.length));
    for (const [hMask, hLen] of masks) {
      if (!(mask & hMask)) ans = Math.max(ans, word.length * hLen);
    }
  }
  return ans;
}
```

---

### 382. Linked List Random Node ⭐⭐（水库采样）

**题目：** 给定一个单向链表，要求设计算法可以随机取得其中的一个数字，链表长度未知。

**思路：** 水库采样——遍历到第 m 个节点时，有 `1/m` 的概率选择这个节点覆盖之前的选择。可以证明最终每个节点被选中的概率相等。

```typescript
class LinkedListRandom {
  private head: ListNode | null;
  constructor(head: ListNode | null) { this.head = head; }
  getRandom(): number {
    let ans = this.head!.val;
    let node = this.head!.next;
    let i = 2;
    while (node) {
      if (Math.floor(Math.random() * i) === 0) ans = node.val;
      i++;
      node = node.next;
    }
    return ans;
  }
}
```

---

## 练习推荐

**排序：** 451（按字符出现频率排序）、75（颜色分类/荷兰国旗问题）

**数学：** 168（Excel 表列名称）、238（除自身以外数组的乘积）

**位运算：** 268（丢失的数字）、260（只出现一次的数字 III）
