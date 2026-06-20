---
layout: post
title: "LeetCode 高频面试题（一）：贪心 · 双指针 · 二分查找"
date: 2025-01-05
categories: 面试
tags: [算法, LeetCode, TypeScript, 高频题]
---

> 本系列整理 LeetCode 高频面试算法题，含完整 TypeScript 实现。本篇覆盖贪心算法、双指针与滑动窗口、二分查找三大主题。
> 难度标注：⭐ Easy ｜ ⭐⭐ Medium ｜ ⭐⭐⭐ Hard

---

## 一、贪心算法

贪心算法采用贪心的策略，保证每次操作都是**局部最优的**，从而使最后得到的结果是**全局最优的**。

### 455. Assign Cookies ⭐

**题目：** 有一群孩子和一堆饼干，每个孩子有一个饥饿度，每个饼干有一个大小。只有饼干的大小 ≥ 孩子的饥饿度时，这个孩子才能吃饱。求最多有多少孩子可以吃饱。

**思路：** 贪心策略——给剩余孩子里最小饥饿度的孩子分配最小的能饱腹的饼干。将孩子和饼干分别排序，从最小值出发，统计可以满足的对数。

```typescript
function findContentChildren(children: number[], cookies: number[]): number {
  children.sort((a, b) => a - b);
  cookies.sort((a, b) => a - b);
  let child = 0, cookie = 0;
  while (child < children.length && cookie < cookies.length) {
    if (cookies[cookie] >= children[child]) child++;
    cookie++;
  }
  return child;
}
```

---

### 135. Candy ⭐⭐⭐

**题目：** 一群孩子站成一排，每个孩子有评分。规则：评分更高的孩子必须比身旁的孩子获得更多糖果；所有孩子至少有一个糖果。求最少需要多少糖果。

**思路：** 两次遍历。先从左往右：右边比左边高则右边 = 左边 + 1；再从右往左：左边比右边高且当前数量不够则更新。

```typescript
function candy(ratings: number[]): number {
  const size = ratings.length;
  if (size < 2) return size;
  const num = new Array(size).fill(1);
  for (let i = 1; i < size; i++) {
    if (ratings[i] > ratings[i - 1]) num[i] = num[i - 1] + 1;
  }
  for (let i = size - 2; i >= 0; i--) {
    if (ratings[i] > ratings[i + 1]) {
      num[i] = Math.max(num[i], num[i + 1] + 1);
    }
  }
  return num.reduce((sum, val) => sum + val, 0);
}
```

---

### 435. Non-overlapping Intervals ⭐⭐

**题目：** 给定多个区间，计算让这些区间互不重叠所需要移除区间的最少个数。起止相连不算重叠。

**思路：** 等价于尽量多保留不重叠区间。区间结尾越小，留给其它区间的空间越大。按结尾升序排序，贪心选择结尾最小且不重叠的区间。

```typescript
function eraseOverlapIntervals(intervals: number[][]): number {
  if (intervals.length === 0) return 0;
  intervals.sort((a, b) => a[1] - b[1]);
  let removed = 0, prevEnd = intervals[0][1];
  for (let i = 1; i < intervals.length; i++) {
    if (intervals[i][0] < prevEnd) {
      removed++;
    } else {
      prevEnd = intervals[i][1];
    }
  }
  return removed;
}
```

---

## 二、双指针

双指针主要用于遍历数组，两个指针指向不同元素，协同完成任务。

- **方向相反**：常用于排好序的数组搜索
- **方向相同（滑动窗口）**：两指针包围的区域即为当前窗口，常用于区间搜索

### 167. Two Sum II ⭐

**题目：** 在增序整数数组里找到两个数，使它们的和为给定值。输出两个数的位置（从 1 开始计数）。

**思路：** 左指针指向最小值，右指针指向最大值，和小于目标则左移右指，和大于目标则左移右指。

```typescript
function twoSum(numbers: number[], target: number): number[] {
  let l = 0, r = numbers.length - 1;
  while (l < r) {
    const sum = numbers[l] + numbers[r];
    if (sum === target) return [l + 1, r + 1];
    else if (sum < target) l++;
    else r--;
  }
  return [];
}
```

---

### 88. Merge Sorted Array ⭐

**题目：** 给定两个有序数组，把两个数组合并为一个，要求 in-place（不开辟额外空间）。

**思路：** 从两个数组末尾开始，将较大的数字填入 nums1 末尾，三指针同步移动。

```typescript
function merge(nums1: number[], m: number, nums2: number[], n: number): void {
  let pos = m + n - 1;
  m--; n--;
  while (m >= 0 && n >= 0) {
    nums1[pos--] = nums1[m] > nums2[n] ? nums1[m--] : nums2[n--];
  }
  while (n >= 0) {
    nums1[pos--] = nums2[n--];
  }
}
```

---

### 142. Linked List Cycle II ⭐⭐

**题目：** 给定一个链表，如果有环路，找出环路的开始节点。

**思路：** Floyd 判圈法（快慢指针）。slow 每次走一步，fast 每次走两步。相遇后将 fast 重置到链表头，再让两者以同样速度前进，再次相遇的节点即为环路起点。

```typescript
function detectCycle(head: ListNode | null): ListNode | null {
  let slow = head, fast = head;
  while (true) {
    if (!fast || !fast.next) return null;
    fast = fast.next.next;
    slow = slow!.next;
    if (fast === slow) break;
  }
  fast = head;
  while (fast !== slow) {
    slow = slow!.next;
    fast = fast!.next;
  }
  return fast;
}
```

---

### 76. Minimum Window Substring ⭐⭐⭐

**题目：** 给定两个字符串 S 和 T，求 S 中包含 T 所有字符的最短连续子字符串，时间复杂度 O(n)。

**思路：** 滑动窗口。`l` 和 `r` 指针都从左向右移动。维护每个字符的缺少数量（chars）和标记数组（flag）。当窗口已包含 T 全部字符时，尝试右移 `l` 缩小窗口。

```typescript
function minWindow(s: string, t: string): string {
  const chars = new Array(128).fill(0);
  const flag = new Array(128).fill(false);
  for (const c of t) {
    flag[c.charCodeAt(0)] = true;
    chars[c.charCodeAt(0)]++;
  }
  let cnt = 0, l = 0, minL = 0, minSize = s.length + 1;
  for (let r = 0; r < s.length; r++) {
    const rc = s.charCodeAt(r);
    if (flag[rc]) {
      if (--chars[rc] >= 0) cnt++;
    }
    while (cnt === t.length) {
      if (r - l + 1 < minSize) { minL = l; minSize = r - l + 1; }
      const lc = s.charCodeAt(l);
      if (flag[lc] && ++chars[lc] > 0) cnt--;
      l++;
    }
  }
  return minSize > s.length ? '' : s.substring(minL, minL + minSize);
}
```

---

## 三、二分查找

二分查找每次将查找区间分成两部分，只取一部分继续查找，时间复杂度 O(log n)。

**注意**：使用 `mid = l + Math.floor((r - l) / 2)` 而非 `(l + r) / 2`，防止整数溢出。

### 69. Sqrt(x) ⭐

**题目：** 给定一个非负整数，求它的开方，向下取整。

**二分法：**

```typescript
function mySqrt(x: number): number {
  if (x === 0) return 0;
  let l = 1, r = x;
  while (l <= r) {
    const mid = l + Math.floor((r - l) / 2);
    const sqrt = Math.floor(x / mid);
    if (sqrt === mid) return mid;
    else if (mid > sqrt) r = mid - 1;
    else l = mid + 1;
  }
  return r;
}
```

**牛顿迭代法（更快）**，迭代公式：`x_{n+1} = (x_n + a/x_n) / 2`：

```typescript
function mySqrt(x: number): number {
  let r = x;
  while (r * r > x) {
    r = Math.floor((r + Math.floor(x / r)) / 2);
  }
  return r;
}
```

---

### 34. Find First and Last Position of Element ⭐⭐

**题目：** 给定增序整数数组和一个值，查找该值第一次和最后一次出现的位置，不存在则返回 `[-1, -1]`。

**思路：** 实现 `lowerBound`（第一个 ≥ target 的位置）和 `upperBound`（第一个 > target 的位置），使用左闭右开写法。

```typescript
function searchRange(nums: number[], target: number): number[] {
  if (nums.length === 0) return [-1, -1];
  const lower = lowerBound(nums, target);
  const upper = upperBound(nums, target) - 1;
  if (lower === nums.length || nums[lower] !== target) return [-1, -1];
  return [lower, upper];
}

function lowerBound(nums: number[], target: number): number {
  let l = 0, r = nums.length;
  while (l < r) {
    const mid = l + Math.floor((r - l) / 2);
    if (nums[mid] >= target) r = mid;
    else l = mid + 1;
  }
  return l;
}

function upperBound(nums: number[], target: number): number {
  let l = 0, r = nums.length;
  while (l < r) {
    const mid = l + Math.floor((r - l) / 2);
    if (nums[mid] > target) r = mid;
    else l = mid + 1;
  }
  return l;
}
```

---

### 81. Search in Rotated Sorted Array II ⭐⭐

**题目：** 一个增序数组被首尾相连后从某个位置断开（旋转数组），给定一个值，判断它是否在此数组中。数组中存在重复数字。

**思路：** 即使旋转后仍可利用递增性进行二分。关键在于判断哪半边是有序的。若中点与左端数字相同则无法判断，左端右移一位继续二分。

```typescript
function search(nums: number[], target: number): boolean {
  let l = 0, r = nums.length - 1;
  while (l <= r) {
    const mid = l + Math.floor((r - l) / 2);
    if (nums[mid] === target) return true;
    if (nums[l] === nums[mid]) {
      l++; // 无法判断哪个区间有序
    } else if (nums[mid] <= nums[r]) {
      // 右区间有序
      if (target > nums[mid] && target <= nums[r]) l = mid + 1;
      else r = mid - 1;
    } else {
      // 左区间有序
      if (target >= nums[l] && target < nums[mid]) r = mid - 1;
      else l = mid + 1;
    }
  }
  return false;
}
```

---

## 练习推荐

**贪心：** 452（最少箭数射爆气球）、763（分区间）、122（买卖股票 II）

**双指针：** 633（平方数之和）、680（验证回文串 II）、340（最多 K 个不同字符的最长子串）

**二分查找：** 154（旋转数组最小值 II）、540（有序数组中的单一元素）、4（两个有序数组的中位数）
