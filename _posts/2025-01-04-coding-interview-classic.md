---
layout: post
title: "前端高频面试编程题（25 题）"
date: 2025-01-04
categories: 面试
tags: [算法, JavaScript, 高频题]
---

> 来源：前端面试指南（字节跳动等大厂 P6/P7 高频题）

---

## 1. 版本号比较

**题目：** 给定两个版本号字符串如 `"1.2.3"` 和 `"1.2.4"`，返回 `-1`（v1 < v2）、`0`（相等）、`1`（v1 > v2）。

**思路：** 按 `.` 分割逐段比较，缺失的段视为 `0`。

```javascript
function compareVersion(v1, v2) {
  const a = v1.split('.');
  const b = v2.split('.');
  const len = Math.max(a.length, b.length);
  for (let i = 0; i < len; i++) {
    const n1 = parseInt(a[i] || '0', 10);
    const n2 = parseInt(b[i] || '0', 10);
    if (n1 > n2) return 1;
    if (n1 < n2) return -1;
  }
  return 0;
}

console.log(compareVersion('1.2.3', '1.2.4')); // -1
console.log(compareVersion('1.10.0', '1.9.0')); // 1
console.log(compareVersion('1.0', '1.0.0'));    // 0
```

---

## 2. 模态框实现

**题目：** 实现原生 JS 模态框，点击按钮弹出，点击遮罩或关闭按钮关闭。

**思路：** 创建遮罩层 + 弹窗容器，CSS 控制显示，绑定点击事件。

```javascript
function createModal(content) {
  const overlay = document.createElement('div');
  overlay.style.cssText = `
    position:fixed;inset:0;background:rgba(0,0,0,.5);
    display:flex;align-items:center;justify-content:center;z-index:1000;
  `;
  const modal = document.createElement('div');
  modal.style.cssText = `background:#fff;padding:24px;border-radius:8px;min-width:300px;position:relative;`;
  const closeBtn = document.createElement('button');
  closeBtn.textContent = '×';
  closeBtn.style.cssText = 'position:absolute;top:8px;right:12px;border:none;background:none;font-size:20px;cursor:pointer;';
  modal.innerHTML = content;
  modal.appendChild(closeBtn);
  overlay.appendChild(modal);
  document.body.appendChild(overlay);
  const close = () => document.body.removeChild(overlay);
  closeBtn.addEventListener('click', close);
  overlay.addEventListener('click', (e) => { if (e.target === overlay) close(); });
  return { close };
}
```

---

## 3. 声明提升与原型链

**题目：** 分析以下代码输出：

```javascript
function Foo() {
  getName = function () { console.log(1); };
  return this;
}
Foo.getName = function () { console.log(2); };
Foo.prototype.getName = function () { console.log(3); };
var getName = function () { console.log(4); };
function getName() { console.log(5); }

Foo.getName();        // ?
getName();            // ?
Foo().getName();      // ?
getName();            // ?
new Foo.getName();    // ?
new Foo().getName();  // ?
```

**思路：**
1. 函数声明整体提升，`var` 声明提升但赋值不提升
2. 最终 `getName` 被赋值为输出 `4` 的函数（`var` 覆盖函数声明）
3. `Foo()` 执行后将全局 `getName` 改为输出 `1`
4. `new Foo().getName()` 沿原型链找到输出 `3`

**答案：**
```
2  // Foo.getName() - 静态方法
4  // getName() - var 赋值覆盖函数声明
1  // Foo().getName() - Foo() 修改了全局 getName，this 是 window
1  // getName() - 已被修改
2  // new Foo.getName() - 等价于 new (Foo.getName)()
3  // new Foo().getName() - 实例找原型
```

---

## 4. class 顺序与 CSS 定义顺序

**题目：** `class="b a"`，CSS 中先定义 `.a` 再定义 `.b`，最终应用哪个样式？

**结论：** CSS 优先级与 HTML 中 class 书写顺序**无关**，由 **CSS 文件中的定义顺序**决定。选择器优先级相同时，**后定义的覆盖先定义的**。

```html
<style>
  .a { color: red; }
  .b { color: blue; }  /* 后定义，优先级更高 */
</style>
<div class="b a">文字</div>
<!-- 最终颜色为 blue -->
```

---

## 5. 二叉树遍历

**题目：** 实现二叉树前序、中序、后序（递归）及层序遍历（BFS）。

```javascript
// 前序（根-左-右）
function preorder(root) {
  if (!root) return [];
  return [root.val, ...preorder(root.left), ...preorder(root.right)];
}

// 中序（左-根-右）
function inorder(root) {
  if (!root) return [];
  return [...inorder(root.left), root.val, ...inorder(root.right)];
}

// 后序（左-右-根）
function postorder(root) {
  if (!root) return [];
  return [...postorder(root.left), ...postorder(root.right), root.val];
}

// 层序（BFS）
function levelOrder(root) {
  if (!root) return [];
  const result = [];
  const queue = [root];
  while (queue.length) {
    const size = queue.length;
    const level = [];
    for (let i = 0; i < size; i++) {
      const node = queue.shift();
      level.push(node.val);
      if (node.left) queue.push(node.left);
      if (node.right) queue.push(node.right);
    }
    result.push(level);
  }
  return result;
}
```

---

## 6. 无重复最长子串

**题目：** 给定字符串 `s`，找出不含重复字符的最长子串的长度。例：`"abcabcbb"` → `3`。

**思路：** 滑动窗口 + Set，右指针右移，出现重复字符则左指针右移消除重复。时间复杂度 O(n)。

```javascript
function lengthOfLongestSubstring(s) {
  let left = 0, max = 0;
  const set = new Set();
  for (let right = 0; right < s.length; right++) {
    while (set.has(s[right])) { set.delete(s[left]); left++; }
    set.add(s[right]);
    max = Math.max(max, right - left + 1);
  }
  return max;
}

console.log(lengthOfLongestSubstring('abcabcbb')); // 3
console.log(lengthOfLongestSubstring('pwwkew'));   // 3
```

---

## 7. jQuery 链式调用实现

**题目：** 实现支持链式调用的 jQuery-like 工具：`$('div').css('color', 'red').on('click', fn)`

**思路：** 每个方法执行完后返回 `this`。

```javascript
class jQuery {
  constructor(selector) {
    this.elements = Array.from(document.querySelectorAll(selector));
  }
  css(prop, value) {
    this.elements.forEach(el => { el.style[prop] = value; });
    return this;
  }
  on(event, handler) {
    this.elements.forEach(el => { el.addEventListener(event, handler); });
    return this;
  }
  text(content) {
    if (content === undefined) return this.elements[0]?.textContent;
    this.elements.forEach(el => { el.textContent = content; });
    return this;
  }
}

function $(selector) { return new jQuery(selector); }
```

---

## 8. 回溯法：数字组合之和为目标值

**题目：** 给定无重复数字数组和目标值，找出所有和为目标值的子数组组合，每个数字可重复使用。`[2,3,6,7], 7` → `[[2,2,3],[7]]`

**思路：** 回溯 DFS，从当前索引选数，和超过目标则剪枝。

```javascript
function combinationSum(candidates, target) {
  const result = [];
  candidates.sort((a, b) => a - b);
  function backtrack(start, current, remain) {
    if (remain === 0) { result.push([...current]); return; }
    for (let i = start; i < candidates.length; i++) {
      if (candidates[i] > remain) break;
      current.push(candidates[i]);
      backtrack(i, current, remain - candidates[i]); // 传 i 允许重复使用
      current.pop();
    }
  }
  backtrack(0, [], target);
  return result;
}

console.log(combinationSum([2, 3, 6, 7], 7)); // [[2,2,3],[7]]
```

---

## 9. React 评论回复列表（递归渲染）

**题目：** 实现抖音评论回复列表组件，支持嵌套评论渲染。

**思路：** 设计树形数据结构，递归渲染评论组件。

{% raw %}
```jsx
const comments = [
  {
    id: 1, author: '用户A', content: '这是一条评论',
    children: [
      { id: 2, author: '用户B', content: '回复评论', children: [] },
      { id: 3, author: '用户C', content: '也回复了', children: [] },
    ],
  },
];

function CommentItem({ comment, depth = 0 }) {
  return (
    <div style={{ paddingLeft: depth * 20 }}>
      <div><strong>{comment.author}</strong>：{comment.content}</div>
      {comment.children?.map(child => (
        <CommentItem key={child.id} comment={child} depth={depth + 1} />
      ))}
    </div>
  );
}

function CommentList({ comments }) {
  return (
    <div>
      {comments.map(comment => <CommentItem key={comment.id} comment={comment} />)}
    </div>
  );
}
```
{% endraw %}

---

## 10. makeRandList：生成不重复随机数列表

**题目：** 实现 `makeRandList(a, b, c)`：生成长度为 `c` 的随机数列表，范围 `[a, b]`，不重复。

**思路：** 用 Set 存储已生成的数，循环生成直到数量满足。需满足 `b - a + 1 >= c`。

```javascript
function makeRandList(a, b, c) {
  if (b - a + 1 < c) throw new Error('范围不足以生成不重复的列表');
  const set = new Set();
  while (set.size < c) {
    set.add(Math.floor(Math.random() * (b - a + 1)) + a);
  }
  return Array.from(set);
}

console.log(makeRandList(1, 10, 5)); // 如 [3, 7, 1, 9, 5]
```

---

## 11. 二叉树路径：根节点到叶子的路径

**题目：** 生成从根节点到每个叶子节点的路径，用 `->` 连接。`[1, 2, 5]` → `"1->2->5"`

**思路：** DFS 维护当前路径数组，到达叶子节点时 join 后加入结果。

```javascript
function binaryTreePaths(root) {
  if (!root) return [];
  const result = [];
  function dfs(node, path) {
    path.push(node.val);
    if (!node.left && !node.right) {
      result.push(path.join('->'));
    } else {
      if (node.left) dfs(node.left, [...path]);
      if (node.right) dfs(node.right, [...path]);
    }
  }
  dfs(root, []);
  return result;
}

// 树 1->2->5, 1->3
const tree = { val: 1, left: { val: 2, left: null, right: { val: 5, left: null, right: null } }, right: { val: 3, left: null, right: null } };
console.log(binaryTreePaths(tree)); // ["1->2->5", "1->3"]
```

---

## 12. 任务管理器：分批并发执行

**题目：** 实现任务管理器，限制同时执行的最大并发数。

**思路：** 维护运行中任务计数器，有任务完成就从队列取下一个，始终保持并发数不超上限。

```javascript
class TaskManager {
  constructor(limit) {
    this.limit = limit;
    this.queue = [];
    this.running = 0;
  }
  add(task) {
    return new Promise((resolve, reject) => {
      this.queue.push({ task, resolve, reject });
      this._run();
    });
  }
  _run() {
    while (this.running < this.limit && this.queue.length > 0) {
      const { task, resolve, reject } = this.queue.shift();
      this.running++;
      Promise.resolve(task())
        .then(resolve, reject)
        .finally(() => { this.running--; this._run(); });
    }
  }
}

const manager = new TaskManager(2);
const delay = (ms, val) => () => new Promise(r => setTimeout(() => r(val), ms));
manager.add(delay(1000, 'A')).then(console.log);
manager.add(delay(500,  'B')).then(console.log);
manager.add(delay(300,  'C')).then(console.log);
// 输出顺序：B -> C -> A
```

---

## 13. 异步加法：Promise 链式调用

**题目：** 只能通过异步接口 `addRemote(a, b)` 做加法，实现支持任意个参数的 `add` 函数。

**思路：** 用 `reduce` 串行调用 `addRemote`，将多个数两两相加。

```javascript
const addRemote = async (a, b) => new Promise(resolve => {
  setTimeout(() => resolve(a + b), 1000);
});

async function add(...nums) {
  return nums.reduce(
    (promiseAcc, cur) => promiseAcc.then(acc => addRemote(acc, cur)),
    Promise.resolve(0)
  );
}

add(1, 2).then(result => console.log(result));    // 3
add(3, 5, 2).then(result => console.log(result)); // 10
```

---

## 14. LRU 缓存

**题目：** 实现 LRU 缓存，支持 `get(key)` 和 `put(key, value)`，容量满时淘汰最久未使用的项。时间复杂度 O(1)。

**思路：** 用 `Map`（按插入顺序迭代）模拟 LRU：`get` 命中则删除再重新插入（移到最新位置）；`put` 超出容量则删除 `map.keys().next().value`（最旧的键）。

```javascript
class LRUCache {
  constructor(capacity) {
    this.capacity = capacity;
    this.map = new Map();
  }
  get(key) {
    if (!this.map.has(key)) return -1;
    const val = this.map.get(key);
    this.map.delete(key);
    this.map.set(key, val); // 移到最新
    return val;
  }
  put(key, value) {
    if (this.map.has(key)) {
      this.map.delete(key);
    } else if (this.map.size >= this.capacity) {
      this.map.delete(this.map.keys().next().value); // 删最旧
    }
    this.map.set(key, value);
  }
}

const cache = new LRUCache(2);
cache.put(1, 1); cache.put(2, 2);
console.log(cache.get(1)); // 1
cache.put(3, 3);           // 淘汰 key=2
console.log(cache.get(2)); // -1
```

---

## 15. 全排列

**题目：** 给定不含重复数字的数组，返回所有可能的全排列。`[1,2,3]` → `[[1,2,3],[1,3,2],...]`

**思路：** 回溯法，每次从未使用的数字中选一个加入路径，直到路径长度等于数组长度。

```javascript
function permute(nums) {
  const result = [];
  const used = new Array(nums.length).fill(false);
  function backtrack(path) {
    if (path.length === nums.length) { result.push([...path]); return; }
    for (let i = 0; i < nums.length; i++) {
      if (used[i]) continue;
      used[i] = true;
      path.push(nums[i]);
      backtrack(path);
      path.pop();
      used[i] = false;
    }
  }
  backtrack([]);
  return result;
}
```

---

## 16. 数字千分位格式化

**题目：** 将 `1234567890` 格式化为 `"1,234,567,890"`。

```javascript
// 方法一：正则
function formatNumber(num) {
  return String(num).replace(/\B(?=(\d{3})+(?!\d))/g, ',');
}

// 方法二：原生 API
function formatNumber2(num) {
  return num.toLocaleString('en-US');
}

// 方法三：手动处理（支持小数）
function formatNumber3(num) {
  const [integer, decimal] = String(num).split('.');
  const formatted = integer.split('').reverse()
    .reduce((acc, digit, i) => i > 0 && i % 3 === 0 ? [digit, ',', ...acc] : [digit, ...acc], [])
    .join('');
  return decimal ? `${formatted}.${decimal}` : formatted;
}

console.log(formatNumber(1234567890));  // "1,234,567,890"
```

---

## 17. 迷宫最短路径

**题目：** 二维数组表示迷宫，`0` 通路 `1` 墙，求从 `[0,0]` 到 `[m-1,n-1]` 的最短路径步数，不可达返回 `-1`。

**思路：** BFS 逐层扩展，第一次到达终点即为最短路径。时间复杂度 O(m×n)。

```javascript
function shortestPath(maze) {
  const m = maze.length, n = maze[0].length;
  if (maze[0][0] === 1 || maze[m-1][n-1] === 1) return -1;
  const dirs = [[0,1],[0,-1],[1,0],[-1,0]];
  const visited = Array.from({ length: m }, () => new Array(n).fill(false));
  const queue = [[0, 0, 1]]; // [row, col, steps]
  visited[0][0] = true;
  while (queue.length) {
    const [r, c, steps] = queue.shift();
    if (r === m - 1 && c === n - 1) return steps;
    for (const [dr, dc] of dirs) {
      const nr = r + dr, nc = c + dc;
      if (nr >= 0 && nr < m && nc >= 0 && nc < n && !visited[nr][nc] && maze[nr][nc] === 0) {
        visited[nr][nc] = true;
        queue.push([nr, nc, steps + 1]);
      }
    }
  }
  return -1;
}

const maze = [[0,0,1,0],[1,0,0,0],[1,1,0,1],[0,0,0,0]];
console.log(shortestPath(maze)); // 7
```

---

## 18. HTML 字符串转 VDOM

**题目：** 将 HTML 字符串转成 VDOM 对象（类 JS 对象表示）。

**思路：** 利用 `DOMParser` 解析 HTML，递归将 DOM 节点转为 JS 对象。

```javascript
function htmlToVdom(htmlStr) {
  const parser = new DOMParser();
  const doc = parser.parseFromString(htmlStr, 'text/html');
  return nodeToVdom(doc.body.firstChild);
}

function nodeToVdom(node) {
  if (node.nodeType === Node.TEXT_NODE) {
    return { type: 'text', content: node.textContent };
  }
  const vnode = { type: node.tagName.toLowerCase(), props: {}, children: [] };
  for (const attr of node.attributes) vnode.props[attr.name] = attr.value;
  for (const child of node.childNodes) vnode.children.push(nodeToVdom(child));
  return vnode;
}

// htmlToVdom('<div class="app"><p>Hello</p></div>')
// { type: 'div', props: { class: 'app' }, children: [{ type: 'p', ... }] }
```

---

## 19. 原型链输出题

**题目：** 分析以下代码输出：

```javascript
function Animal(name) { this.name = name; }
Animal.prototype.speak = function () { console.log(this.name + ' makes a sound.'); };

function Dog(name) { Animal.call(this, name); }
Dog.prototype = Object.create(Animal.prototype);
Dog.prototype.constructor = Dog;
Dog.prototype.speak = function () { console.log(this.name + ' barks.'); };

const d = new Dog('Rex');
d.speak();                          // ?
console.log(d instanceof Dog);      // ?
console.log(d instanceof Animal);   // ?
```

**思路：** `Dog.prototype.speak` 覆盖了父级方法；`instanceof` 沿原型链检查。

**答案：**
```
Rex barks.   // Dog.prototype.speak 覆盖了 Animal.prototype.speak
true          // d 的原型链上有 Dog.prototype
true          // d 的原型链上也有 Animal.prototype
```

---

## 20. 请求并发队列（限并发数）

**题目：** 实现 `requestWithLimit(urls, limit)`：并发请求多个 URL，限制最大并发数，返回按 URL 顺序排列的所有结果。

**思路：** 启动 `limit` 个 worker 协程，每个 worker 循环从共享索引取 URL 执行，用 `index` 追踪结果位置保证顺序。

```javascript
async function requestWithLimit(urls, limit) {
  const results = new Array(urls.length);
  let index = 0;
  async function worker() {
    while (index < urls.length) {
      const i = index++;
      try {
        results[i] = await fetch(urls[i]).then(r => r.json());
      } catch (e) {
        results[i] = { error: e.message };
      }
    }
  }
  const workers = Array.from({ length: Math.min(limit, urls.length) }, worker);
  await Promise.all(workers);
  return results;
}
```

---

## 21. 最长子数组和

**题目：** 给定整数数组，找出和最大的连续子数组，返回最大和。`[-2,1,-3,4,-1,2,1,-5,4]` → `6`

**思路：** 动态规划（Kadane 算法）：`dp[i] = max(nums[i], dp[i-1] + nums[i])`。时间 O(n)，空间 O(1)。

```javascript
function maxSubArray(nums) {
  let maxSum = nums[0];
  let current = nums[0];
  for (let i = 1; i < nums.length; i++) {
    current = Math.max(nums[i], current + nums[i]);
    maxSum = Math.max(maxSum, current);
  }
  return maxSum;
}

console.log(maxSubArray([-2, 1, -3, 4, -1, 2, 1, -5, 4])); // 6
```

---

## 22. 二叉树最大深度

**题目：** 求二叉树最大深度（根到最远叶子的最长路径上的节点数）。

**思路：** 递归：`depth = 1 + max(left_depth, right_depth)`；空节点返回 0。

```javascript
// 递归
function maxDepth(root) {
  if (!root) return 0;
  return 1 + Math.max(maxDepth(root.left), maxDepth(root.right));
}

// BFS 迭代
function maxDepthBFS(root) {
  if (!root) return 0;
  let depth = 0;
  const queue = [root];
  while (queue.length) {
    depth++;
    const size = queue.length;
    for (let i = 0; i < size; i++) {
      const node = queue.shift();
      if (node.left) queue.push(node.left);
      if (node.right) queue.push(node.right);
    }
  }
  return depth;
}
```

---

## 23. 反转链表

**题目：** 反转单链表。`1->2->3->4->5` → `5->4->3->2->1`

**思路：** 迭代：维护 `prev`、`curr` 两个指针，逐步翻转 `next` 指向。

```javascript
// 迭代
function reverseList(head) {
  let prev = null, curr = head;
  while (curr) {
    const next = curr.next;
    curr.next = prev;
    prev = curr;
    curr = next;
  }
  return prev;
}

// 递归
function reverseListRecursive(head) {
  if (!head || !head.next) return head;
  const newHead = reverseListRecursive(head.next);
  head.next.next = head;
  head.next = null;
  return newHead;
}
```

---

## 24. lodash \_.get 实现

**题目：** 实现 `_.get(object, path, defaultValue)`，安全访问对象深层属性，失败返回默认值。

**思路：** 将路径字符串拆分后逐层访问，中途遇到 `null/undefined` 则返回默认值。

```javascript
function get(obj, path, defaultValue = undefined) {
  const keys = Array.isArray(path)
    ? path
    : path.replace(/\[(\d+)\]/g, '.$1').split('.').filter(Boolean);
  let result = obj;
  for (const key of keys) {
    if (result == null) return defaultValue;
    result = result[key];
  }
  return result === undefined ? defaultValue : result;
}

const obj = { a: { b: { c: 42 } }, arr: [1, 2, 3] };
console.log(get(obj, 'a.b.c'));        // 42
console.log(get(obj, 'arr[1]'));       // 2
console.log(get(obj, 'a.x.y', 'NA')); // "NA"
```

---

## 25. EventEmitter 实现

**题目：** 实现事件发布/订阅系统，支持 `on`、`off`、`emit`、`once`。

**思路：** 用 Map 存储事件名到监听函数数组的映射。`once` 通过包装函数实现，触发一次后自动 `off`。

```javascript
class EventEmitter {
  constructor() { this.events = new Map(); }

  on(event, listener) {
    if (!this.events.has(event)) this.events.set(event, []);
    this.events.get(event).push(listener);
    return this;
  }

  off(event, listener) {
    if (!this.events.has(event)) return this;
    this.events.set(event, this.events.get(event).filter(l => l !== listener));
    return this;
  }

  emit(event, ...args) {
    if (!this.events.has(event)) return false;
    this.events.get(event).forEach(listener => listener(...args));
    return true;
  }

  once(event, listener) {
    const wrapper = (...args) => { listener(...args); this.off(event, wrapper); };
    return this.on(event, wrapper);
  }
}

const emitter = new EventEmitter();
emitter.on('data', (msg) => console.log('收到:', msg));
emitter.once('connect', () => console.log('已连接'));
emitter.emit('data', 'hello'); // 收到: hello
emitter.emit('connect');       // 已连接
emitter.emit('connect');       // （无输出）
```
