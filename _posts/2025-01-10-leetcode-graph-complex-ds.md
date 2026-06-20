---
layout: post
title: "LeetCode 高频面试题（六）：图 · 复杂数据结构"
date: 2025-01-10
categories: 面试
tags: [算法, LeetCode, TypeScript, 高频题, 图, 数据结构]
---

> 本篇覆盖图算法（二分图、拓扑排序）和复杂数据结构（并查集、LRU Cache），含完整 TypeScript 实现。
> 难度标注：⭐ Easy ｜ ⭐⭐ Medium ｜ ⭐⭐⭐ Hard

---

## 图的表示方法

图通常有两种表示方法：
- **邻接矩阵**：建立 n×n 矩阵 G，若节点 i 连向节点 j，则 `G[i][j] = 1`
- **邻接链表**：大小为 n 的数组，每个位置储存该节点连向的其它节点

---

## 一、二分图

**二分图判定（染色法）**：若可以用两种颜色对图的节点进行着色，且保证相邻节点颜色不同，则图为二分图。使用广度优先搜索实现。

### 785. Is Graph Bipartite? ⭐⭐

**题目：** 给定一个图（邻接链表形式），判断其是否可以二分。

**思路：** 用 0 表示未检查节点，1 和 2 表示两种颜色。BFS 对未染色节点染色，若发现相邻节点颜色相同则不是二分图。

```typescript
function isBipartite(graph: number[][]): boolean {
  const n = graph.length;
  const color = new Array(n).fill(0);
  const queue: number[] = [];
  for (let i = 0; i < n; i++) {
    if (color[i] !== 0) continue;
    queue.push(i);
    color[i] = 1;
    while (queue.length > 0) {
      const node = queue.shift()!;
      for (const j of graph[node]) {
        if (color[j] === 0) {
          queue.push(j);
          color[j] = color[node] === 2 ? 1 : 2;
        } else if (color[node] === color[j]) {
          return false;
        }
      }
    }
  }
  return true;
}
```

---

## 二、拓扑排序

**拓扑排序**：对有向无环图（DAG）排序，使得若原图中节点 i 指向节点 j，则排序结果中 i 一定在 j 之前。

**算法步骤（BFS/Kahn 算法）：**
1. 统计所有节点的入度
2. 将入度为 0 的节点加入队列
3. 每次取出队首节点，将其指向的节点入度减 1
4. 若某节点入度变为 0，加入队列
5. 若所有节点都被处理，则存在拓扑排序；否则图中有环

### 210. Course Schedule II ⭐⭐

**题目：** 给定 N 个课程和课程的前置必修关系，求可以一次性上完所有课程的顺序。若不可能完成则返回空数组。

```typescript
function findOrder(numCourses: number, prerequisites: number[][]): number[] {
  const graph: number[][] = Array.from({ length: numCourses }, () => []);
  const indegree = new Array(numCourses).fill(0);
  for (const [a, b] of prerequisites) {
    graph[b].push(a);
    indegree[a]++;
  }
  const queue: number[] = [];
  for (let i = 0; i < numCourses; i++) {
    if (indegree[i] === 0) queue.push(i);
  }
  const res: number[] = [];
  while (queue.length > 0) {
    const u = queue.shift()!;
    res.push(u);
    for (const v of graph[u]) {
      indegree[v]--;
      if (indegree[v] === 0) queue.push(v);
    }
  }
  return res.length === numCourses ? res : [];
}
```

---

## 三、并查集

**并查集**（Union-Find）可以动态地连通两个点，并且非常快速地判断两个点是否连通。

**核心操作：**
- `find(p)`：查找 p 的根节点（使用路径压缩优化）
- `connect(p, q)`：连接 p 和 q（使用按秩合并优化）
- `isConnected(p, q)`：判断 p 和 q 是否连通

```typescript
class UnionFind {
  private id: number[];
  private size: number[];

  constructor(n: number) {
    this.id = Array.from({ length: n }, (_, i) => i);
    this.size = new Array(n).fill(1);
  }

  find(p: number): number {
    while (p !== this.id[p]) {
      this.id[p] = this.id[this.id[p]]; // 路径压缩
      p = this.id[p];
    }
    return p;
  }

  connect(p: number, q: number): void {
    const i = this.find(p), j = this.find(q);
    if (i === j) return;
    if (this.size[i] < this.size[j]) {
      this.id[i] = j;
      this.size[j] += this.size[i];
    } else {
      this.id[j] = i;
      this.size[i] += this.size[j];
    }
  }

  isConnected(p: number, q: number): boolean {
    return this.find(p) === this.find(q);
  }
}
```

### 684. Redundant Connection ⭐⭐

**题目：** 在无向图找出一条边，移除它之后该图能够成为一棵树。如果有多个解，返回在原数组中位置最靠后的那条边。

**思路：** 用并查集判断添加每条边时是否会形成环。若两个端点已连通则该边为冗余边。

```typescript
function findRedundantConnection(edges: number[][]): number[] {
  const n = edges.length;
  const uf = new UnionFind(n + 1);
  for (const e of edges) {
    const [u, v] = e;
    if (uf.isConnected(u, v)) return e;
    uf.connect(u, v);
  }
  return [-1, -1];
}
```

---

## 四、LRU Cache

**LRU Cache（最近最少使用缓存）**：固定大小的缓存，查询或插入时将该信息标为最近使用，缓存满时删除最旧的信息。

**实现：** 双向链表（维护使用顺序）+ 哈希表（O(1) 寻址）。最新信息在链表头，最旧信息在链表尾。

### 146. LRU Cache ⭐⭐

```typescript
class DLinkedNode {
  key: number;
  val: number;
  prev: DLinkedNode | null = null;
  next: DLinkedNode | null = null;
  constructor(key = 0, val = 0) { this.key = key; this.val = val; }
}

class LRUCache {
  private capacity: number;
  private hash: Map<number, DLinkedNode>;
  private head: DLinkedNode;
  private tail: DLinkedNode;

  constructor(capacity: number) {
    this.capacity = capacity;
    this.hash = new Map();
    this.head = new DLinkedNode();
    this.tail = new DLinkedNode();
    this.head.next = this.tail;
    this.tail.prev = this.head;
  }

  private removeNode(node: DLinkedNode): void {
    node.prev!.next = node.next;
    node.next!.prev = node.prev;
  }

  private addToHead(node: DLinkedNode): void {
    node.prev = this.head;
    node.next = this.head.next;
    this.head.next!.prev = node;
    this.head.next = node;
  }

  get(key: number): number {
    const node = this.hash.get(key);
    if (!node) return -1;
    this.removeNode(node);
    this.addToHead(node);
    return node.val;
  }

  put(key: number, value: number): void {
    const node = this.hash.get(key);
    if (node) {
      node.val = value;
      this.removeNode(node);
      this.addToHead(node);
    } else {
      const newNode = new DLinkedNode(key, value);
      this.hash.set(key, newNode);
      this.addToHead(newNode);
      if (this.hash.size > this.capacity) {
        const tail = this.tail.prev!;
        this.removeNode(tail);
        this.hash.delete(tail.key);
      }
    }
  }
}
```

**使用示例：**
```typescript
const cache = new LRUCache(2);
cache.put(1, 1);
cache.put(2, 2);
cache.get(1);    // 返回 1
cache.put(3, 3); // 淘汰 key 2
cache.get(2);    // 返回 -1（未找到）
cache.put(4, 4); // 淘汰 key 1
cache.get(1);    // 返回 -1（未找到）
cache.get(3);    // 返回 3
cache.get(4);    // 返回 4
```

---

## 五、字符串算法

### 242. Valid Anagram ⭐

**题目：** 给定两个字符串，判断它们是否是同构的（字母种类和数量都相同，顺序可能不同）。

**思路：** 用长度为 26 的数组统计两个字符串每个字母出现次数的差值，最后全为 0 则同构。

```typescript
function isAnagram(s: string, t: string): boolean {
  if (s.length !== t.length) return false;
  const count = new Array(26).fill(0);
  const base = 'a'.charCodeAt(0);
  for (let i = 0; i < s.length; i++) {
    count[s.charCodeAt(i) - base]++;
    count[t.charCodeAt(i) - base]--;
  }
  return count.every(c => c === 0);
}
```

---

### 205. Isomorphic Strings ⭐

**题目：** 给定两个字符串，判断它们是否同构（可以对 s 中的字符进行替换使其变成 t，不同字符不可以映射到同一字符）。

**思路：** 记录每个位置的字符在 s 和 t 中第一次出现的位置，若两者始终相同则同构。

```typescript
function isIsomorphic(s: string, t: string): boolean {
  const mapS = new Map<string, number>();
  const mapT = new Map<string, number>();
  for (let i = 0; i < s.length; i++) {
    const cs = s[i], ct = t[i];
    if (!mapS.has(cs)) mapS.set(cs, i);
    if (!mapT.has(ct)) mapT.set(ct, i);
    if (mapS.get(cs) !== mapT.get(ct)) return false;
  }
  return true;
}
```

---

### 647. Palindromic Substrings ⭐⭐

**题目：** 给定一个字符串，求它有多少个回文子字符串。

**思路：** 中心扩展法——从每个位置向外扩展，分别以当前位置为中心（奇数长度）或以当前位置和下一位置之间为中心（偶数长度），统计回文子串数量。

```typescript
function countSubstrings(s: string): number {
  let count = 0;
  const extend = (l: number, r: number) => {
    while (l >= 0 && r < s.length && s[l] === s[r]) { count++; l--; r++; }
  };
  for (let i = 0; i < s.length; i++) {
    extend(i, i);
    extend(i, i + 1);
  }
  return count;
}
```

---

### 696. Count Binary Substrings ⭐

**题目：** 给定一个 0-1 字符串，求有多少子字符串中 0 和 1 数量相同且都是连续出现的。

**思路：** 记录当前连续相同字符长度（cur）和上一段连续相同字符长度（pre）。若 `pre >= cur` 则存在一个满足条件的子字符串。

```typescript
function countBinarySubstrings(s: string): number {
  let pre = 0, cur = 1, count = 0;
  for (let i = 1; i < s.length; i++) {
    if (s[i] === s[i - 1]) cur++;
    else { pre = cur; cur = 1; }
    if (pre >= cur) count++;
  }
  return count;
}
```

---

### 其他字符串算法

### 227. Basic Calculator II ⭐⭐

**题目：** 给定只包含非负整数和 `+`、`-`、`*`、`/` 的字符串，求计算结果。

**思路：** 栈处理运算优先级。遇到 `+/-` 直接入栈（负数取负），遇到 `*//` 弹出栈顶计算后再入栈，最后对栈求和。

```typescript
function calculate(s: string): number {
  const stack: number[] = [];
  let num = 0, op = '+';
  for (let i = 0; i < s.length; i++) {
    const c = s[i];
    if (c >= '0' && c <= '9') num = num * 10 + parseInt(c);
    if ((c === '+' || c === '-' || c === '*' || c === '/') || i === s.length - 1) {
      if (op === '+') stack.push(num);
      else if (op === '-') stack.push(-num);
      else if (op === '*') stack.push(stack.pop()! * num);
      else if (op === '/') stack.push(Math.trunc(stack.pop()! / num));
      op = c; num = 0;
    }
  }
  return stack.reduce((a, b) => a + b, 0);
}
```

---

### 28. Find the Index of the First Occurrence in a String ⭐（KMP 算法）

**题目：** 在 haystack 中找出 needle 第一次出现的位置，不存在返回 -1。

**思路：** KMP 算法——先计算 needle 的 `next` 数组（部分匹配表），再利用 next 数组避免重复比较已匹配字符。时间复杂度 O(m+n)。

```typescript
function strStr(haystack: string, needle: string): number {
  const n = haystack.length, m = needle.length;
  if (m === 0) return 0;
  const next = new Array(m).fill(0);
  for (let i = 1, j = 0; i < m; i++) {
    while (j > 0 && needle[i] !== needle[j]) j = next[j - 1];
    if (needle[i] === needle[j]) j++;
    next[i] = j;
  }
  for (let i = 0, j = 0; i < n; i++) {
    while (j > 0 && haystack[i] !== needle[j]) j = next[j - 1];
    if (haystack[i] === needle[j]) j++;
    if (j === m) return i - m + 1;
  }
  return -1;
}
```

---

## 复杂数据结构选型总结

| 场景 | 数据结构 | 典型题目 |
|------|---------|---------|
| O(1) 查找/插入 | 哈希表（Map/Set）| Two Sum、最长连续序列 |
| 动态连通性 | 并查集 | 冗余连接、朋友圈 |
| 最近使用缓存 | 双向链表 + 哈希表 | LRU Cache |
| 有序集合/多重集合 | 排序数组或自实现 | 天际线问题 |
| 优先访问最大/小值 | 堆（MinHeap/MaxHeap）| 合并 K 个链表 |
| 快速访问前缀 | 前缀和 / 积分图 | 区间和查询 |

## 练习推荐

**图：** 1059（所有路径通向目的地）、1135（最低成本联通所有城市，最小生成树）

**并查集：** 380（O(1) 时间插入删除获取随机元素）

**复杂 DS：** 432（全 O(1) 数据结构）、307（区间和查询（可变），线段树）
