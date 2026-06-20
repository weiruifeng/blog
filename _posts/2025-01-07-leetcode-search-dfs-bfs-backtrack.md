---
layout: post
title: "LeetCode 高频面试题（三）：搜索 · DFS · BFS · 回溯"
date: 2025-01-07
categories: 面试
tags: [算法, LeetCode, TypeScript, 高频题]
---

> 本篇覆盖深度优先搜索（DFS）、回溯法、广度优先搜索（BFS）三大搜索算法，含完整 TypeScript 实现。
> 难度标注：⭐ Easy ｜ ⭐⭐ Medium ｜ ⭐⭐⭐ Hard

---

## 一、深度优先搜索（DFS）

DFS 在搜索到新节点时立即对该节点进行遍历，用**先进后出的栈**（或等价的递归）实现。对于已搜索过的节点进行标记（状态记录/记忆化），防止重复搜索。

### 695. Max Area of Island ⭐

**题目：** 给定一个二维 0-1 矩阵，0 表示海洋，1 表示陆地，相邻的陆地形成岛屿。求最大的岛屿面积。

**思路：** 对每个值为 1 的格子启动 DFS，访问后将格子置为 0 防止重复，递归统计四个方向的面积之和。

```typescript
function maxAreaOfIsland(grid: number[][]): number {
  const m = grid.length, n = grid[0]?.length ?? 0;
  const dirs = [-1, 0, 1, 0, -1];

  const dfs = (r: number, c: number): number => {
    if (r < 0 || r >= m || c < 0 || c >= n || grid[r][c] === 0) return 0;
    grid[r][c] = 0;
    let area = 1;
    for (let k = 0; k < 4; k++) {
      area += dfs(r + dirs[k], c + dirs[k + 1]);
    }
    return area;
  };

  let maxArea = 0;
  for (let i = 0; i < m; i++) {
    for (let j = 0; j < n; j++) {
      if (grid[i][j] === 1) maxArea = Math.max(maxArea, dfs(i, j));
    }
  }
  return maxArea;
}
```

---

### 547. Friend Circles ⭐⭐

**题目：** 给定一个对称的 0-1 矩阵，`(i,j)=1` 表示第 i 和第 j 个人是朋友（朋友关系可传递）。求一共有多少个朋友圈。

**思路：** 本质是求连通分量数量。每行表示一个节点，用 DFS 搜索同一朋友圈内的所有人。

```typescript
function findCircleNum(friends: number[][]): number {
  const n = friends.length;
  const visited = new Array(n).fill(false);

  const dfs = (i: number): void => {
    visited[i] = true;
    for (let k = 0; k < n; k++) {
      if (friends[i][k] === 1 && !visited[k]) dfs(k);
    }
  };

  let count = 0;
  for (let i = 0; i < n; i++) {
    if (!visited[i]) { dfs(i); count++; }
  }
  return count;
}
```

---

### 417. Pacific Atlantic Water Flow ⭐⭐

**题目：** 给定海拔高度矩阵，左/上边界是太平洋，右/下边界是大西洋，水只能从高处流向低处或相同高度。求哪些位置的水可以同时流到两个大洋。

**思路：** 反向思考——从两个大洋边界出发，向上流（海拔不降低），用 DFS 标记哪些位置可以分别到达太平洋和大西洋，两个标记都为 true 的位置即为答案。

```typescript
function pacificAtlantic(matrix: number[][]): number[][] {
  if (!matrix.length || !matrix[0].length) return [];
  const m = matrix.length, n = matrix[0].length;
  const dirs = [-1, 0, 1, 0, -1];
  const canReachP = Array.from({ length: m }, () => new Array(n).fill(false));
  const canReachA = Array.from({ length: m }, () => new Array(n).fill(false));

  const dfs = (r: number, c: number, canReach: boolean[][]): void => {
    if (canReach[r][c]) return;
    canReach[r][c] = true;
    for (let k = 0; k < 4; k++) {
      const x = r + dirs[k], y = c + dirs[k + 1];
      if (x >= 0 && x < m && y >= 0 && y < n && matrix[r][c] <= matrix[x][y]) {
        dfs(x, y, canReach);
      }
    }
  };

  for (let i = 0; i < m; i++) { dfs(i, 0, canReachP); dfs(i, n - 1, canReachA); }
  for (let j = 0; j < n; j++) { dfs(0, j, canReachP); dfs(m - 1, j, canReachA); }

  const ans: number[][] = [];
  for (let i = 0; i < m; i++) {
    for (let j = 0; j < n; j++) {
      if (canReachP[i][j] && canReachA[i][j]) ans.push([i, j]);
    }
  }
  return ans;
}
```

---

## 二、回溯法

回溯法是优先搜索的一种特殊情况，常用于需要记录节点状态的 DFS。排列、组合、选择类问题使用回溯法比较方便。

**核心模式：**
1. 修改状态（进入当前分支）
2. 递归子节点
3. 回改状态（退出当前分支，恢复原状）

### 46. Permutations ⭐⭐

**题目：** 给定一个无重复数字的整数数组，求其所有的排列方式。

**思路：** 对于当前位置 i，将其之后任意位置交换，然后继续处理位置 i+1，直到处理到最后一位。递归完成后再交换回来（回溯）。

```typescript
function permute(nums: number[]): number[][] {
  const ans: number[][] = [];

  const backtrack = (level: number): void => {
    if (level === nums.length - 1) {
      ans.push([...nums]);
      return;
    }
    for (let i = level; i < nums.length; i++) {
      [nums[i], nums[level]] = [nums[level], nums[i]];  // 修改状态
      backtrack(level + 1);
      [nums[i], nums[level]] = [nums[level], nums[i]];  // 回改状态
    }
  };

  backtrack(0);
  return ans;
}
```

---

### 77. Combinations ⭐⭐

**题目：** 给定整数 n 和 k，求在 1 到 n 中选取 k 个数字的所有组合方法。

**思路：** 回溯是否把当前数字加入结果。

```typescript
function combine(n: number, k: number): number[][] {
  const ans: number[][] = [];
  const comb: number[] = [];

  const backtrack = (pos: number): void => {
    if (comb.length === k) {
      ans.push([...comb]);
      return;
    }
    for (let i = pos; i <= n; i++) {
      comb.push(i);       // 修改状态
      backtrack(i + 1);
      comb.pop();         // 回改状态
    }
  };

  backtrack(1);
  return ans;
}
```

---

### 79. Word Search ⭐⭐

**题目：** 给定一个字母矩阵，所有格子上下左右相连。给定一个字符串，求字符串能否在字母矩阵中找到。

**思路：** 修改访问标记来实现回溯（而非修改输出方式）。在 DFS 时标记当前位置为已访问，所有可能搜索完成后回改为未访问。

```typescript
function exist(board: string[][], word: string): boolean {
  const m = board.length, n = board[0].length;
  const visited = Array.from({ length: m }, () => new Array(n).fill(false));
  const dirs = [-1, 0, 1, 0, -1];

  const backtrack = (i: number, j: number, pos: number): boolean => {
    if (i < 0 || i >= m || j < 0 || j >= n) return false;
    if (visited[i][j] || board[i][j] !== word[pos]) return false;
    if (pos === word.length - 1) return true;
    visited[i][j] = true;
    for (let k = 0; k < 4; k++) {
      if (backtrack(i + dirs[k], j + dirs[k + 1], pos + 1)) {
        visited[i][j] = false;
        return true;
      }
    }
    visited[i][j] = false;
    return false;
  };

  for (let i = 0; i < m; i++) {
    for (let j = 0; j < n; j++) {
      if (backtrack(i, j, 0)) return true;
    }
  }
  return false;
}
```

---

### 51. N-Queens ⭐⭐⭐

**题目：** 给定 n×n 棋盘，求有多少种方式可以放置 n 个皇后使得它们互不攻击（每行、列、斜线最多一个皇后）。

**思路：** 对每一行逐列尝试放置皇后，建立列（col）、左斜（ldiag）、右斜（rdiag）三个访问数组，通过回溯枚举所有合法方案。

```typescript
function solveNQueens(n: number): string[][] {
  const ans: string[][] = [];
  const board: string[] = Array(n).fill('.'.repeat(n));
  const col = new Array(n).fill(false);
  const ldiag = new Array(2 * n - 1).fill(false);
  const rdiag = new Array(2 * n - 1).fill(false);

  const backtrack = (row: number): void => {
    if (row === n) { ans.push([...board]); return; }
    for (let i = 0; i < n; i++) {
      if (col[i] || ldiag[n - row + i - 1] || rdiag[row + i]) continue;
      col[i] = ldiag[n - row + i - 1] = rdiag[row + i] = true;
      board[row] = '.'.repeat(i) + 'Q' + '.'.repeat(n - i - 1);
      backtrack(row + 1);
      col[i] = ldiag[n - row + i - 1] = rdiag[row + i] = false;
      board[row] = '.'.repeat(n);
    }
  };

  backtrack(0);
  return ans;
}
```

---

## 三、广度优先搜索（BFS）

BFS 是一层层进行遍历的，使用**先入先出的队列**实现。常用于处理**最短路径**问题。

在遍历一层时，当前队列中的节点数就是当前层的节点数，只需控制遍历这么多节点，就能保证遍历的都是当前层的节点。

### 934. Shortest Bridge ⭐⭐

**题目：** 给定一个二维 0-1 矩阵，只有两个岛屿，求最少要填海造陆多少个位置才可以将两个岛屿相连。

**思路：** 先通过 DFS 找到其中一个岛屿（标记为 2，加入队列）；再利用 BFS 从该岛屿向外扩展，直到碰到另一个岛屿，扩展的层数即为答案。

```typescript
function shortestBridge(grid: number[][]): number {
  const m = grid.length, n = grid[0].length;
  const dirs = [-1, 0, 1, 0, -1];
  const queue: [number, number][] = [];

  const dfs = (i: number, j: number): void => {
    if (i < 0 || i >= m || j < 0 || j >= n || grid[i][j] !== 1) return;
    grid[i][j] = 2;
    queue.push([i, j]);
    for (let k = 0; k < 4; k++) dfs(i + dirs[k], j + dirs[k + 1]);
  };

  let found = false;
  for (let i = 0; i < m && !found; i++) {
    for (let j = 0; j < n && !found; j++) {
      if (grid[i][j] === 1) { dfs(i, j); found = true; }
    }
  }

  let level = 0;
  while (queue.length > 0) {
    level++;
    const size = queue.length;
    for (let s = 0; s < size; s++) {
      const [r, c] = queue.shift()!;
      for (let k = 0; k < 4; k++) {
        const x = r + dirs[k], y = c + dirs[k + 1];
        if (x < 0 || x >= m || y < 0 || y >= n || grid[x][y] === 2) continue;
        if (grid[x][y] === 1) return level;
        grid[x][y] = 2;
        queue.push([x, y]);
      }
    }
  }
  return level;
}
```

---

## 练习推荐

**DFS：** 130（被围绕的区域）、257（二叉树的所有路径）

**回溯：** 47（全排列 II，处理重复元素）、40（组合总和 II）、37（解数独）

**BFS：** 310（最小高度树）、126（单词接龙 II，双向 BFS）
