---
layout: post
title: "LeetCode 高频面试题（五）：数据结构（数组 · 字符串 · 链表 · 树）"
date: 2025-01-09
categories: 面试
tags: [算法, LeetCode, TypeScript, 高频题, 数据结构]
---

> 本篇覆盖数组与矩阵、字符串、链表、树四大数据结构类型，含完整 TypeScript 实现。
> 难度标注：⭐ Easy ｜ ⭐⭐ Medium ｜ ⭐⭐⭐ Hard

---

## 一、数组与矩阵

### 1. Two Sum ⭐（哈希表）

**题目：** 给定整数数组，已知有且只有两个数的和等于给定值，求这两个数的位置。

**思路：** 哈希表存储遍历过的值及位置，每次查找是否存在 `target - nums[i]`。

```typescript
function twoSum(nums: number[], target: number): number[] {
  const map = new Map<number, number>();
  for (let i = 0; i < nums.length; i++) {
    const complement = target - nums[i];
    if (map.has(complement)) return [map.get(complement)!, i];
    map.set(nums[i], i);
  }
  return [];
}
```

---

### 128. Longest Consecutive Sequence ⭐⭐⭐（哈希表）

**题目：** 给定整数数组，求数组中的数字可以组成的最长连续序列长度。

**思路：** 把所有数字放入 Set，只从序列起点（`num-1` 不在 Set 中）开始向后扩展，避免重复计算。O(n) 时间复杂度。

```typescript
function longestConsecutive(nums: number[]): number {
  const set = new Set(nums);
  let ans = 0;
  for (const num of set) {
    if (!set.has(num - 1)) {
      let cur = num, length = 1;
      while (set.has(cur + 1)) { cur++; length++; }
      ans = Math.max(ans, length);
    }
  }
  return ans;
}
```

---

### 48. Rotate Image ⭐⭐（矩阵）

**题目：** 给定 n×n 矩阵，顺时针旋转 90 度，要求 in-place 操作。

**思路：** 每次只考虑间隔 90 度的四个位置，进行 O(1) 额外空间的旋转。

```typescript
function rotate(matrix: number[][]): void {
  const n = matrix.length - 1;
  for (let i = 0; i <= Math.floor(n / 2); i++) {
    for (let j = i; j < n - i; j++) {
      const temp = matrix[j][n - i];
      matrix[j][n - i] = matrix[i][j];
      matrix[i][j] = matrix[n - j][i];
      matrix[n - j][i] = matrix[n - i][n - j];
      matrix[n - i][n - j] = temp;
    }
  }
}
```

---

### 560. Subarray Sum Equals K ⭐⭐（前缀和）

**题目：** 给定一个数组和整数 k，寻找和为 k 的连续区间个数。

**思路：** 前缀和 + 哈希表。遍历到位置 i 时，若当前前缀和为 `psum`，则 `map[psum-k]` 即为以当前位置结尾且满足条件的区间个数。

```typescript
function subarraySum(nums: number[], k: number): number {
  let count = 0, psum = 0;
  const map = new Map<number, number>();
  map.set(0, 1);
  for (const num of nums) {
    psum += num;
    count += map.get(psum - k) ?? 0;
    map.set(psum, (map.get(psum) ?? 0) + 1);
  }
  return count;
}
```

---

## 二、栈与队列

### 20. Valid Parentheses ⭐

**题目：** 给定只由括号组成的字符串，求是否合法（每种括号都一一对应）。

**思路：** 遍历字符串，遇到左括号压栈，遇到右括号判断与栈顶是否匹配。

```typescript
function isValid(s: string): boolean {
  const stack: string[] = [];
  const map: Record<string, string> = { '}': '{', ']': '[', ')': '(' };
  for (const c of s) {
    if (c === '{' || c === '[' || c === '(') {
      stack.push(c);
    } else {
      if (!stack.length || stack[stack.length - 1] !== map[c]) return false;
      stack.pop();
    }
  }
  return stack.length === 0;
}
```

---

### 739. Daily Temperatures ⭐⭐（单调栈）

**题目：** 给定每天的温度，求对于每天需要等几天才能等到更暖和的天气。

**思路：** 维护单调递减栈（存储位置）。每当新日期的温度高于栈顶位置的温度时，弹出栈顶并记录等待天数。

```typescript
function dailyTemperatures(temperatures: number[]): number[] {
  const n = temperatures.length;
  const ans = new Array(n).fill(0);
  const stack: number[] = [];
  for (let i = 0; i < n; i++) {
    while (stack.length && temperatures[i] > temperatures[stack[stack.length - 1]]) {
      const preIndex = stack.pop()!;
      ans[preIndex] = i - preIndex;
    }
    stack.push(i);
  }
  return ans;
}
```

---

### 239. Sliding Window Maximum ⭐⭐⭐（单调双端队列）

**题目：** 给定整数数组和滑动窗口大小，求在窗口滑动过程中每个时刻包含的最大值。

**思路：** 双端队列（存储下标）维护单调递减序列。每次入队前删掉比当前值小的所有元素；每次出队时若队首已超出窗口范围则删除队首。

```typescript
function maxSlidingWindow(nums: number[], k: number): number[] {
  const dq: number[] = [];
  const ans: number[] = [];
  for (let i = 0; i < nums.length; i++) {
    if (dq.length && dq[0] === i - k) dq.shift();
    while (dq.length && nums[dq[dq.length - 1]] < nums[i]) dq.pop();
    dq.push(i);
    if (i >= k - 1) ans.push(nums[dq[0]]);
  }
  return ans;
}
```

---

## 三、链表

链表操作两个小技巧：
1. 尽量处理当前节点的**下一个节点**而非当前节点本身
2. 建立**虚拟节点（dummy node）**使其指向链表头节点，即使链表所有节点被删除也有 dummy 存在

### 206. Reverse Linked List ⭐

**题目：** 翻转一个链表。

```typescript
// 迭代写法
function reverseList(head: ListNode | null): ListNode | null {
  let prev: ListNode | null = null, curr = head;
  while (curr) {
    const next = curr.next;
    curr.next = prev;
    prev = curr;
    curr = next;
  }
  return prev;
}

// 递归写法
function reverseListRecursive(head: ListNode | null, prev: ListNode | null = null): ListNode | null {
  if (!head) return prev;
  const next = head.next;
  head.next = prev;
  return reverseListRecursive(next, head);
}
```

---

### 21. Merge Two Sorted Lists ⭐

**题目：** 给定两个增序链表，将其合并成一个增序链表。

```typescript
function mergeTwoLists(l1: ListNode | null, l2: ListNode | null): ListNode | null {
  const dummy = new ListNode(0);
  let node = dummy;
  while (l1 && l2) {
    if (l1.val <= l2.val) { node.next = l1; l1 = l1.next; }
    else { node.next = l2; l2 = l2.next; }
    node = node.next!;
  }
  node.next = l1 ? l1 : l2;
  return dummy.next;
}
```

---

### 160. Intersection of Two Linked Lists ⭐

**题目：** 给定两个链表，求相交的节点（若无则返回 null）。

**思路：** 假设链表 A 头到相交点距离为 a，链表 B 头到相交点距离为 b，相交点到末尾距离为 c。两个指针各从链表头出发，走完本链表后走另一链表，两者在 a+b+c 步后同时到达相交节点。

```typescript
function getIntersectionNode(headA: ListNode | null, headB: ListNode | null): ListNode | null {
  let l1 = headA, l2 = headB;
  while (l1 !== l2) {
    l1 = l1 ? l1.next : headB;
    l2 = l2 ? l2.next : headA;
  }
  return l1;
}
```

---

### 234. Palindrome Linked List ⭐（O(1) 空间）

**题目：** 以 O(1) 的空间复杂度，判断链表是否回文。

**思路：** 快慢指针找到链表中点，翻转后半段，再比较两半是否相等。

```typescript
function isPalindrome(head: ListNode | null): boolean {
  if (!head || !head.next) return true;
  let slow: ListNode = head, fast: ListNode = head;
  while (fast.next && fast.next.next) {
    slow = slow.next!;
    fast = fast.next.next;
  }
  slow.next = reverseList(slow.next);
  slow = slow.next!;
  let curr: ListNode | null = head;
  while (slow) {
    if (curr!.val !== slow.val) return false;
    curr = curr!.next;
    slow = slow.next!;
  }
  return true;
}
```

---

## 四、树

树的题目通常用**递归**（DFS）解决，对于层次遍历用**BFS**。

### 104. Maximum Depth of Binary Tree ⭐

```typescript
function maxDepth(root: TreeNode | null): number {
  return root ? 1 + Math.max(maxDepth(root.left), maxDepth(root.right)) : 0;
}
```

---

### 110. Balanced Binary Tree ⭐

**题目：** 判断一个二叉树是否平衡（任意节点两侧深度差 ≤ 1）。

**思路：** 在求深度的同时检测平衡性。若发现不平衡则返回 -1，祖先节点可以直接跳过判断。

```typescript
function isBalanced(root: TreeNode | null): boolean {
  return helper(root) !== -1;
}

function helper(root: TreeNode | null): number {
  if (!root) return 0;
  const left = helper(root.left), right = helper(root.right);
  if (left === -1 || right === -1 || Math.abs(left - right) > 1) return -1;
  return 1 + Math.max(left, right);
}
```

---

### 543. Diameter of Binary Tree ⭐

**题目：** 求二叉树的最长直径（任意两节点间的无向距离）。

**思路：** 递归时区分两个值：更新的最长直径（经过该子树根节点的两侧之和）vs 递归返回值（以该子树根节点为端点的单侧长度）。

```typescript
function diameterOfBinaryTree(root: TreeNode | null): number {
  let diameter = 0;
  function helper(node: TreeNode | null): number {
    if (!node) return 0;
    const l = helper(node.left), r = helper(node.right);
    diameter = Math.max(diameter, l + r);
    return Math.max(l, r) + 1;
  }
  helper(root);
  return diameter;
}
```

---

### 101. Symmetric Tree ⭐

**题目：** 判断一个二叉树是否对称。

**四步法：** ① 两个子树都为空 → 相等；② 只有一个为空 → 不相等；③ 根节点值不相等 → 不相等；④ 根据对称要求递归处理。

```typescript
function isSymmetric(root: TreeNode | null): boolean {
  return root ? check(root.left, root.right) : true;
}

function check(left: TreeNode | null, right: TreeNode | null): boolean {
  if (!left && !right) return true;
  if (!left || !right) return false;
  if (left.val !== right.val) return false;
  return check(left.left, right.right) && check(left.right, right.left);
}
```

---

### 105. Construct Binary Tree from Preorder and Inorder ⭐⭐

**题目：** 给定前序遍历和中序遍历结果，复原二叉树（树中无重复值）。

**思路：** 前序遍历的第一个节点是根节点，在中序遍历中找到根节点位置，左侧为左子树，右侧为右子树，递归复原。哈希表预处理中序遍历位置加速查找。

```typescript
function buildTree(preorder: number[], inorder: number[]): TreeNode | null {
  if (!preorder.length) return null;
  const hash = new Map<number, number>();
  for (let i = 0; i < inorder.length; i++) hash.set(inorder[i], i);
  return build(hash, preorder, 0, preorder.length - 1, 0);
}

function build(hash: Map<number, number>, preorder: number[], s0: number, e0: number, s1: number): TreeNode | null {
  if (s0 > e0) return null;
  const mid = preorder[s1];
  const index = hash.get(mid)!;
  const leftSize = index - s0;
  const node = new TreeNode(mid);
  node.left = build(hash, preorder, s0, index - 1, s1 + 1);
  node.right = build(hash, preorder, index + 1, e0, s1 + 1 + leftSize);
  return node;
}
```

---

### 208. Implement Trie (Prefix Tree) ⭐⭐（字典树）

**题目：** 建立字典树，支持快速插入单词、查找单词、查找单词前缀。

```typescript
class TrieNode {
  children = new Map<string, TrieNode>();
  isEnd = false;
}

class Trie {
  private root = new TrieNode();

  insert(word: string): void {
    let node = this.root;
    for (const c of word) {
      if (!node.children.has(c)) node.children.set(c, new TrieNode());
      node = node.children.get(c)!;
    }
    node.isEnd = true;
  }

  search(word: string): boolean {
    let node = this.root;
    for (const c of word) {
      if (!node.children.has(c)) return false;
      node = node.children.get(c)!;
    }
    return node.isEnd;
  }

  startsWith(prefix: string): boolean {
    let node = this.root;
    for (const c of prefix) {
      if (!node.children.has(c)) return false;
      node = node.children.get(c)!;
    }
    return true;
  }
}
```

---

---

### 448. Find All Numbers Disappeared in an Array ⭐

**题目：** 给定长度为 n、范围为 1~n 的整数数组（有些整数重复），求 1~n 中没有出现过的整数。

**思路：** 原地标记——把重复出现的数字在原数组对应位置设置为负数，最后仍为正数的位置即为缺失的数。

```typescript
function findDisappearedNumbers(nums: number[]): number[] {
  const ans: number[] = [];
  for (const num of nums) {
    const pos = Math.abs(num) - 1;
    if (nums[pos] > 0) nums[pos] = -nums[pos];
  }
  for (let i = 0; i < nums.length; i++) {
    if (nums[i] > 0) ans.push(i + 1);
  }
  return ans;
}
```

---

### 240. Search a 2D Matrix II ⭐⭐

**题目：** 给定每行每列都是增序的二维矩阵，设计快速搜索算法判断某个数字是否存在。

**思路：** 从右上角开始查找——当前值大于目标则左移，小于目标则下移。O(m+n) 时间复杂度。

```typescript
function searchMatrix(matrix: number[][], target: number): boolean {
  const m = matrix.length;
  if (!m) return false;
  const n = matrix[0].length;
  let i = 0, j = n - 1;
  while (i < m && j >= 0) {
    if (matrix[i][j] === target) return true;
    else if (matrix[i][j] > target) j--;
    else i++;
  }
  return false;
}
```

---

### 232. Implement Queue using Stacks ⭐

**题目：** 使用栈实现队列，支持 push、pop、peek、empty 操作。

**思路：** 两个栈。inStack 负责入队，outStack 负责出队。当 outStack 为空时，将 inStack 全部转移进去。

```typescript
class MyQueue {
  private inStack: number[] = [];
  private outStack: number[] = [];
  push(x: number): void { this.inStack.push(x); }
  private in2out(): void {
    if (!this.outStack.length) {
      while (this.inStack.length) this.outStack.push(this.inStack.pop()!);
    }
  }
  pop(): number { this.in2out(); return this.outStack.pop()!; }
  peek(): number { this.in2out(); return this.outStack[this.outStack.length - 1]; }
  empty(): boolean { return !this.inStack.length && !this.outStack.length; }
}
```

---

### 155. Min Stack ⭐

**题目：** 设计支持 O(1) 时间查询栈内最小值的最小栈。

**思路：** 额外维护一个辅助栈，栈顶始终是当前原栈的最小值。入栈时若新值 ≤ 辅助栈顶则同步压入辅助栈；出栈时若弹出值等于辅助栈顶则同步弹出。

```typescript
class MinStack {
  private s: number[] = [];
  private minS: number[] = [];
  push(x: number): void {
    this.s.push(x);
    if (!this.minS.length || this.minS[this.minS.length - 1] >= x) this.minS.push(x);
  }
  pop(): void {
    const top = this.s.pop()!;
    if (top === this.minS[this.minS.length - 1]) this.minS.pop();
  }
  top(): number { return this.s[this.s.length - 1]; }
  getMin(): number { return this.minS[this.minS.length - 1]; }
}
```

---

### 303. Range Sum Query - Immutable ⭐（前缀和）

**题目：** 设计数据结构，快速查询数组任意两个位置间所有数字的和。

**思路：** 前缀和 `psum[i]` 表示前 i 个数字之和，区间 `[i,j]` 的和为 `psum[j+1] - psum[i]`。

```typescript
class NumArray {
  private psum: number[];
  constructor(nums: number[]) {
    this.psum = new Array(nums.length + 1).fill(0);
    for (let i = 0; i < nums.length; i++) this.psum[i + 1] = this.psum[i] + nums[i];
  }
  sumRange(i: number, j: number): number { return this.psum[j + 1] - this.psum[i]; }
}
```

---

### 304. Range Sum Query 2D - Immutable ⭐⭐（积分图）

**题目：** 设计数据结构，快速查询矩阵中任意矩形区域所有数字的和。

**思路：** 积分图（二维前缀和）。`integral[i][j]` 表示以 (0,0) 为左上角、(i-1,j-1) 为右下角的矩形内数字之和。查询时用四个位置的积分图值做加减运算。

```typescript
class NumMatrix {
  private integral: number[][];
  constructor(matrix: number[][]) {
    const m = matrix.length, n = m > 0 ? matrix[0].length : 0;
    this.integral = Array.from({ length: m + 1 }, () => new Array(n + 1).fill(0));
    for (let i = 1; i <= m; i++) {
      for (let j = 1; j <= n; j++) {
        this.integral[i][j] = matrix[i-1][j-1]
          + this.integral[i-1][j] + this.integral[i][j-1] - this.integral[i-1][j-1];
      }
    }
  }
  sumRegion(r1: number, c1: number, r2: number, c2: number): number {
    return this.integral[r2+1][c2+1] - this.integral[r2+1][c1]
      - this.integral[r1][c2+1] + this.integral[r1][c1];
  }
}
```

---

### 332. Reconstruct Itinerary ⭐⭐

**题目：** 给定一些飞机的起止机场，已知从 JFK 起飞，求字典序最小的飞行顺序。

**思路：** 哈希表记录起止关系（目标机场排序），用栈模拟 DFS（Hierholzer 算法），将终点逆序收集为路径。

```typescript
function findItinerary(tickets: string[][]): string[] {
  const map = new Map<string, string[]>();
  for (const [from, to] of tickets) {
    if (!map.has(from)) map.set(from, []);
    map.get(from)!.push(to);
  }
  for (const dests of map.values()) dests.sort();
  const ans: string[] = [];
  const stack: string[] = ['JFK'];
  while (stack.length) {
    const next = stack[stack.length - 1];
    const dests = map.get(next);
    if (!dests || !dests.length) ans.push(stack.pop()!);
    else stack.push(dests.shift()!);
  }
  return ans.reverse();
}
```

---

## 链表补充

### 24. Swap Nodes in Pairs ⭐⭐

**题目：** 给定一个链表，交换每个相邻的一对节点。

**思路：** 用虚拟节点 dummy，每次处理 p.next 和 p.next.next 两个节点的交换。

```typescript
function swapPairs(head: ListNode | null): ListNode | null {
  const dummy = new ListNode(0);
  dummy.next = head;
  let p: ListNode = dummy;
  while (p.next && p.next.next) {
    const s = p.next.next;
    const t = p.next;
    t.next = s.next;
    s.next = t;
    p.next = s;
    p = t;
  }
  return dummy.next;
}
```

---

## 树补充

### 437. Path Sum III ⭐⭐

**题目：** 给定整数二叉树，求有多少条路径节点值之和等于给定值（路径不需从根节点出发）。

**思路：** 双重递归——外层遍历每个节点，内层从该节点出发计算连续加入节点的路径数。

```typescript
function pathSum(root: TreeNode | null, sum: number): number {
  if (!root) return 0;
  return pathSumStart(root, sum) + pathSum(root.left, sum) + pathSum(root.right, sum);
}

function pathSumStart(root: TreeNode | null, sum: number): number {
  if (!root) return 0;
  let count = root.val === sum ? 1 : 0;
  count += pathSumStart(root.left, sum - root.val);
  count += pathSumStart(root.right, sum - root.val);
  return count;
}
```

---

### 637. Average of Levels in Binary Tree ⭐（层次遍历）

**题目：** 给定一个二叉树，求每层节点值的平均数。

**思路：** BFS 层次遍历，开始遍历一层时队列中的节点数即为该层节点数。

```typescript
function averageOfLevels(root: TreeNode | null): number[] {
  const ans: number[] = [];
  if (!root) return ans;
  const queue: TreeNode[] = [root];
  while (queue.length) {
    const count = queue.length;
    let sum = 0;
    for (let i = 0; i < count; i++) {
      const node = queue.shift()!;
      sum += node.val;
      if (node.left) queue.push(node.left);
      if (node.right) queue.push(node.right);
    }
    ans.push(sum / count);
  }
  return ans;
}
```

---

### 144. Binary Tree Preorder Traversal ⭐⭐（非递归）

**题目：** 不使用递归，实现二叉树的前序遍历。

**思路：** 递归本质是栈调用，用显式栈模拟。先压右子节点，再压左子节点，保证左子树先遍历。

```typescript
function preorderTraversal(root: TreeNode | null): number[] {
  const ret: number[] = [];
  if (!root) return ret;
  const stack: TreeNode[] = [root];
  while (stack.length) {
    const node = stack.pop()!;
    ret.push(node.val);
    if (node.right) stack.push(node.right);
    if (node.left) stack.push(node.left);
  }
  return ret;
}
```

---

### 99. Recover Binary Search Tree ⭐⭐⭐

**题目：** 给定一个二叉搜索树，已知有两个节点被不小心交换了，试复原此树。

**思路：** 中序遍历 BST 的结果是有序数组。用 prev 指针追踪前一个节点，若出现 `node.val < prev.val` 则记为错误节点。只出现一次错误说明是相邻节点交换，出现两次则交换两个节点。

```typescript
function recoverTree(root: TreeNode | null): void {
  let mistake1: TreeNode | null = null;
  let mistake2: TreeNode | null = null;
  let prev: TreeNode | null = null;
  function inorder(node: TreeNode | null): void {
    if (!node) return;
    inorder(node.left);
    if (prev && node.val < prev.val) {
      if (!mistake1) mistake1 = prev;
      mistake2 = node;
    }
    prev = node;
    inorder(node.right);
  }
  inorder(root);
  if (mistake1 && mistake2) {
    const temp = mistake1.val;
    mistake1.val = mistake2.val;
    mistake2.val = temp;
  }
}
```

---

### 669. Trim a Binary Search Tree ⭐（BST）

**题目：** 给定一个 BST 和范围 [low, high]，修剪此树使得所有节点值都在范围内。

**思路：** 利用 BST 大小关系：若当前值 > high 则修剪后的树在左子树；若 < low 则在右子树。

```typescript
function trimBST(root: TreeNode | null, low: number, high: number): TreeNode | null {
  if (!root) return null;
  if (root.val > high) return trimBST(root.left, low, high);
  if (root.val < low) return trimBST(root.right, low, high);
  root.left = trimBST(root.left, low, high);
  root.right = trimBST(root.right, low, high);
  return root;
}
```

---

### 1110. Delete Nodes And Return Forest ⭐⭐

**题目：** 给定整数二叉树和一些整数，删掉这些整数对应的节点后，返回剩余的子树列表。

**思路：** 递归后序处理——先递归处理子节点，再判断当前节点是否需要删除。删除时将其子节点（若存在）加入结果集。

```typescript
function delNodes(root: TreeNode | null, to_delete: number[]): Array<TreeNode | null> {
  const forest: Array<TreeNode | null> = [];
  const dict = new Set(to_delete);
  function helper(node: TreeNode | null): TreeNode | null {
    if (!node) return null;
    node.left = helper(node.left);
    node.right = helper(node.right);
    if (dict.has(node.val)) {
      if (node.left) forest.push(node.left);
      if (node.right) forest.push(node.right);
      return null;
    }
    return node;
  }
  const newRoot = helper(root);
  if (newRoot) forest.push(newRoot);
  return forest;
}
```

---

## 优先队列（堆）

优先队列可以在 O(1) 时间内获得最大/最小值，并且可以在 O(log n) 时间内取出或插入任意值。通常用完全二叉树（堆）实现，位置 i 的父节点在 `i/2`，子节点在 `2i` 和 `2i+1`。

### 23. Merge k Sorted Lists ⭐⭐⭐

**题目：** 给定 k 个增序链表，将它们合并成一条增序链表。

**思路：** 将所有节点值收集后排序，重建链表（或用最小堆每次提取所有链表头部的最小节点）。

```typescript
function mergeKLists(lists: Array<ListNode | null>): ListNode | null {
  const vals: number[] = [];
  const collect = (node: ListNode | null) => {
    while (node) { vals.push(node.val); node = node.next; }
  };
  lists.forEach(collect);
  vals.sort((a, b) => a - b);
  const dummy = new ListNode(0);
  let cur = dummy;
  for (const v of vals) { cur.next = new ListNode(v); cur = cur.next; }
  return dummy.next;
}
```

---

### 218. The Skyline Problem ⭐⭐⭐

**题目：** 给定建筑物的起止位置和高度，返回建筑物轮廓（天际线）的拐点坐标。

**思路：** 将所有建筑物的左右端点拆成事件点（左端用负高度表示）排序后扫描。维护当前活跃建筑高度的有序结构，每次高度发生变化时记录拐点。

```typescript
function getSkyline(buildings: number[][]): number[][] {
  const events: [number, number][] = [];
  for (const [l, r, h] of buildings) {
    events.push([l, -h]);
    events.push([r, h]);
  }
  events.sort((a, b) => a[0] !== b[0] ? a[0] - b[0] : a[1] - b[1]);
  const ans: number[][] = [];
  const heights = [0];
  for (const [x, h] of events) {
    if (h < 0) {
      heights.push(-h);
      heights.sort((a, b) => b - a);
    } else {
      heights.splice(heights.indexOf(h), 1);
    }
    const curMax = heights[0];
    if (!ans.length || ans[ans.length - 1][1] !== curMax) ans.push([x, curMax]);
  }
  return ans;
}
```

---

## 练习推荐

**数组：** 769（最多能完成排序的块）、149（直线上最多的点数）

**链表：** 83（删除排序链表中的重复元素）、148（排序链表）

**树：** 226（翻转二叉树）、572（另一棵树的子树）、235（BST 的最近公共祖先）
