# 如何给自建 AI 聊天站加上类似于官端的树状分支机制

对话树（Conversation tree）是 Claude 等AI软件的Chatting界面用来支撑用户消息编辑（Edit）、AI回复重试（Retry）与分支切换的底层数据结构。在对话树结构之上每条消息都是树上的一个节点，对某一消息的编辑或重试操作不会覆盖任何原有的消息内容，同时从同一位置长出一簇新的分支。

这篇笔记记录了我们在自建对话站内从零实现这套机制的过程：怎么设计节点结构、怎么让服务端管理对话树的状态、怎么在前端渲染当前分支和导航箭头。格式为Node.js + 原生 JS。

> *关于 VPS* 对话树本身的数据结构（第1、2章）和前端渲染逻辑（第4章）不依赖服务端——纯前端把树存在 localStorage 里也能跑。但如果你想要跨设备访问、防浏览器崩溃丢数据、多标签页不打架，就需要一个后端来持有对话树的权威副本。本篇的第3、5、6章讲的就是前后端配合的部分。整体核心思路：**服务端是对话树的主人，前端是对话树的画布。**

## 目录

1. [单条消息的样子](#1-单条消息的样子)
2. [编辑（Edit）、重试（Retry）与继续对话](#2-编辑edit重试retry与继续对话)
3. [为什么让服务端做决策](#3-为什么让服务端做决策)
4. [前端怎样渲染与导航对话树](#4-前端怎样渲染与导航对话树)
5. [把分支状态同步到服务端](#5-把分支状态同步到服务端)
6. [怎么安全地存到磁盘](#6-怎么安全地存到磁盘)
7. [重申前文注意事项（摘要）](#7-重申前文注意事项摘要)
- [附：不用服务端也能跑吗](#附不用服务端也能跑吗)

## 1 单条消息的样子

对话树的每个节点就是一条消息。整棵对话树是一个**扁平的哈希表**（非嵌套结构）+一个指针。

```
     __root__
        │
      user₁
        │
       ai₁
        │
    ┌───┴───┐
  user₂   user₂'          ← user₂ 被编辑，产生了 user₂'
    │        │
   ai₂     ai₂'           ← 两条路各自往下走
    │
  user₃
```

```json
{
  "nodes": {
    "__root__": { "id": "__root__", "role": "system", "parentId": null },
    "m_a1": { "id": "m_a1", "role": "user",      "content": "...", "parentId": "__root__", "ts": 1720300000 },
    "m_a2": { "id": "m_a2", "role": "assistant", "content": "...", "parentId": "m_a1",     "ts": 1720300001 },
    "m_b1": { "id": "m_b1", "role": "user",      "content": "...", "parentId": "m_a2",     "ts": 1720300002 }
  },
  "currentLeaf": "m_b1"
}
```

注意事项：

**`1. __root__`** 是一个不可见的哨兵节点，它永远存在且永远不渲染。对话的第一条消息挂在它的下面。它的存在让"第一条消息也可以有兄弟"成为可能——如果用户编辑了开头的第一句话，新旧两条消息都挂在 `__root__` 下形成分支。

**`2. parentId`** 是节点间唯一的关系纽带。每个节点只知道自己的父亲是谁而不知道自己有哪些孩子。`children` 数组是运行时从 `parentId` 反向推出来的衍生数据。后文会细讲。

**`3. currentLeaf`** 指向用户当前正在看的那条分支的最深处。整棵树只有这一个导航状态，不管你的对话树有多少分支、多少层，"我现在在哪"永远只需要这一个指针来回答。

> **为什么结构扁平？** 嵌套的对话树（children 套 children）查找是递归、更新要深拷贝、序列化的时候还要处理循环引用。扁平哈希表的一切操作都是 `nodes[id]`，O(1)。简单是一个用户与Ta的ai维护一个系统正常运行时最重要的品质，每多一个概念就多一份面对更多风险的可能。

## 2 编辑（Edit）、重试（Retry）与继续对话

编辑用户输入的消息（Edit）、重试 AI 输出的回复（Retry）与继续进行对话看起来是三件不同的事，不过在对话树里它们是**同一个操作**：

```js
addNode(parentId, role, content)
```

区别仅仅是 `parentId` 指向谁：

```
继续对话:    addNode(当前叶子节点, 'user', '新消息')
             → 在对话末尾往下长一层

编辑旧消息:  addNode(原消息的父节点, 'user', '修改后的内容')
             → 在原消息旁边长出一个兄弟节点

重试 AI:     addNode(AI回复的父节点, 'assistant', '')
             → 在原回复旁边长出一个兄弟节点，然后流式填入新回复
```

代码完全一样，我们不需要为此写多套逻辑。

```js
// 编辑
async function submitEdit(originalNodeId, newText) {
  const original = tree.nodes[originalNodeId];
  const node = await addNode(original.parentId, 'user', newText);
  streamAIReply(node.id);
}

// 重试
async function retryReply(aiNodeId) {
  const aiNode = tree.nodes[aiNodeId];
  const node = await addNode(aiNode.parentId, 'assistant', '');
  streamAIReply(null, node.id);
}
```

编辑（Edit）与重试（Retry）两个函数的核心都是一句 `addNode(original.parentId, ...)`。对话树在这一刻自动分叉了——因为同一个父节点此刻有了两个子节点。旧的子节点（原消息/原回复）和新的子节点并排挂着，各自延伸出自己的分支。

`submitEdit` 在创建新节点后立刻调了 `streamAIReply`——编辑完毕即触发重新生成。这是一个设计选择。也可以把编辑和重新生成拆成两步（编辑后不自动请求新回复，用户手动点重试）。对话树的结构对两种做法都成立。

> *永久留存旧的消息* 编辑和重试只新增节点。旧的分支完整保留，随时可以切回去。

## 3 为什么让服务端做决策

前面的 `addNode` 不是一个前端函数，它是一个发给服务端的请求。前端不自己造节点、生成 ID与写磁盘。前端只负责描述意图（"我要在这个父节点下加一条消息"），让服务端执行。

换个说法：**前端是投影仪，服务端是胶片。**投影仪不改胶片的内容——它只负责把胶片上的画面投出来。所有对树的修改（加节点、切分支、删消息）都发生在胶片上，投影仪收到新画面就重新投。

```js
// 服务端
app.post('/api/conversations/:id/add-node', (req, res) => {
  const conv = loadConv(req.params.id);
  const { parentId, role, content } = req.body;

  const node = {
    id: 'm_' + Date.now().toString(36) + '_' + Math.random().toString(36).slice(2,7),
    role, content, parentId,
    children: [],
    ts: Date.now()
  };

  conv.tree.nodes[node.id] = node;
  conv.tree.nodes[parentId].children.push(node.id);
  conv.tree.currentLeaf = node.id;
  saveConv(req.params.id, conv);
  res.json({ node });
});

// 切换分支也走服务端——currentLeaf 是持久化状态
app.post('/api/conversations/:id/switch-branch', (req, res) => {
  const conv = loadConv(req.params.id);
  const leafId = walkToLeaf(conv.tree.nodes, req.body.siblingId);
  conv.tree.currentLeaf = leafId;
  saveConv(req.params.id, conv);
  res.json({ currentLeaf: leafId });
});
```

```js
// 前端 — 发请求，用服务端返回的节点
async function addNode(parentId, role, content) {
  const res = await fetch(`/api/conversations/${convId}/add-node`, {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({ parentId, role, content })
  });
  const { node } = await res.json();
  tree.nodes[node.id] = node;
  tree.nodes[parentId].children.push(node.id);
  tree.currentLeaf = node.id;
  return node;
}
```

> **为什么服务端生成 ID** 防止前后端打架撞车（特别是在诸多边界条件下），于是让服务端拥有对话树的形状完全的控制权。前端为建议者，服务端为决定者。

> **为什么分支切换也走服务端？** `currentLeaf` 需要持久化。例如，就算用户刷新浏览器或是关闭浏览器再打开，页面仍显示离开前停留的那条分支路径。

## 4 前端怎样渲染与导航对话树

渲染当前对话：调用 `getPath()`，拿到一个数组，从头到尾画。**跟画一个线性对话完全一样**——对话树的复杂度被 `getPath()` 吸收了，渲染层看到的就是一个数组。

```js
function getPath(tree) {
  const path = [];
  let cur = tree.nodes[tree.currentLeaf];
  while (cur && cur.id !== '__root__') {
    path.unshift(cur);
    cur = cur.parentId ? tree.nodes[cur.parentId] : null;
  }
  return path;
}
```

分支箭头：画每条消息的时候，查一下它的父节点有几个子节点。如果超过一个，说明这里有分支，画一个 `< 2/3 >` 的导航控件。

```js
function renderBranchNav(node) {
  const parent = tree.nodes[node.parentId];
  if (!parent || parent.children.length <= 1) return '';

  const siblings = parent.children;
  const idx = siblings.indexOf(node.id);

  return `
    <span class="branch-nav">
      <button onclick="switchBranch('${node.id}',-1)" ${idx===0?'disabled':''}>‹</button>
      ${idx+1}/${siblings.length}
      <button onclick="switchBranch('${node.id}',1)" ${idx===siblings.length-1?'disabled':''}>›</button>
    </span>`;
}
```

### 分支切换——纯前端就能完成

导航箭头画出来了，接下来让它能点。用户点击箭头时，需要从目标兄弟节点开始一路往下走到最深的叶子——这条路上每遇到分支就取最新的（最后一个子节点）：

```js
function walkToLeaf(nodes, startId) {
  let cur = nodes[startId];
  while (cur?.children?.length > 0) {
    cur = nodes[cur.children[cur.children.length - 1]];
  }
  return cur ? cur.id : startId;
}
```

切换分支 = 找到目标兄弟 → `walkToLeaf` 走到底 → 把结果写入 `currentLeaf` → 用新的 `getPath()` 重新渲染。这个操作**不需要服务端参与**——它操作的全是内存里已有的树结构：

```js
function switchBranch(nodeId, delta) {
  const parent = tree.nodes[tree.nodes[nodeId].parentId];
  const newSibling = parent.children[parent.children.indexOf(nodeId) + delta];
  tree.currentLeaf = walkToLeaf(tree.nodes, newSibling);
  renderConversation();
}
```

到这里为止，你已经有了一个**完整可用的分支导航器**：能渲染当前路径、能画出分支箭头、能点击切换到不同分支。全部逻辑跑在浏览器里，零网络请求。如果你的对话树存在 localStorage 里（见附录），这就够了——拖一个 HTML 文件进浏览器就能体验完整的分支切换。

## 5 把分支状态同步到服务端

上一章你已经有了一个能在本地切换分支的导航器。那为什么我们的实现里还要把 `switchBranch` 的结果发给服务端？

因为 `currentLeaf` 需要**持久化**。纯本地版的切换改的是内存里的变量——刷新页面就丢了。如果你需要：刷新后仍停在上次看的分支、换一台设备接着看、多标签页不互相踩——你就需要服务端来持有 `currentLeaf` 的权威副本。

把上一章的纯本地版升级成走服务端只需要一步改动——把本地赋值换成一次 API 请求：

```js
async function switchBranch(nodeId, delta) {
  const parent = tree.nodes[tree.nodes[nodeId].parentId];
  const newSibling = parent.children[parent.children.indexOf(nodeId) + delta];
  const res = await fetch(`/api/conversations/${convId}/switch-branch`, {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({ siblingId: newSibling })
  });
  tree.currentLeaf = (await res.json()).currentLeaf;
  renderConversation();
}
```

逻辑完全一样——找兄弟、走到叶子、更新指针、重新渲染。唯一的区别是 `walkToLeaf` 跑在服务端而不是浏览器里，结果通过 API 返回。这样 `currentLeaf` 的变更被写进了磁盘，刷新不丢、跨设备同步。

> **为什么分支切换也走服务端？** 不是因为切换逻辑需要服务端——上一章证明了它不需要。是因为 `currentLeaf` 作为持久化状态，它的权威副本应该和对话树的其他数据放在一起。前端可以自己算出正确的 leafId（它确实也在第4章这样做了），但最终要把这个结果"报备"给服务端，让服务端的磁盘副本保持同步。

## 6 怎么安全地存到磁盘

### parentId 是唯一真相，children 是衍生数据。

运行时每个节点都有一个 `children` 数组，方便查询和渲染。但**写磁盘的时候把 children 剥掉**，只存 `parentId`。读回来的时候从所有节点的 `parentId` 反向重建 `children`。

记得为 `children` 排序。如果从 parentId 重建 children 时不排序，分支导航会有概率乱序（取决于 `Object.values` 的遍历顺序）。建议按时间戳排列消息顺序，旧的在前新的在后。

```js
// 写盘前剥掉 children
function stripForDisk(nodes) {
  const out = {};
  for (const [id, node] of Object.entries(nodes)) {
    const { children, ...rest } = node;
    out[id] = rest;
  }
  return out;
}

// 读盘后重建 children
function rebuildChildren(nodes) {
  for (const n of Object.values(nodes)) n.children = [];
  for (const n of Object.values(nodes)) {
    if (n.parentId && nodes[n.parentId])
      nodes[n.parentId].children.push(n.id);
  }
  // 按时间排序，分支顺序稳定
  for (const n of Object.values(nodes)) {
    if (n.children.length > 1)
      n.children.sort((a, b) => (nodes[a]?.ts || 0) - (nodes[b]?.ts || 0));
  }
}
```

> *为什么不两个都存？* 如果 `parentId` 和 `children` 同时存盘，它们可能在某次异常写入后不一致——节点 A 的 parentId 指向 B，但 B 的 children 里没有 A。`parentId` 是单向指针，自身不可能矛盾。`children` 每次读盘重建，永远跟 `parentId` 一致。仅一份真相，零可能出错。

### 原子写入

先写 `.tmp`，再 `rename`。断电不会留下一个写了一半的坏文件。

```js
function saveConv(id, conv) {
  const data = { ...conv, tree: { ...conv.tree, nodes: stripForDisk(conv.tree.nodes) } };
  const file = `./data/${id}.json`;
  fs.writeFileSync(file + '.tmp', JSON.stringify(data));
  fs.renameSync(file + '.tmp', file);
}
```

## 7 重申前文注意事项（摘要）

1. **`parentId` 是唯一的亲缘关系。** `children` 写盘时剥掉，读盘时重建。
2. **`currentLeaf` 是唯一的导航状态。** 一个指针指向叶子，往上走就是当前路径。不在每个分支点存各自的"当前选中"。
3. **编辑、重试、继续——都是 `addNode(parentId)`。** 差别只在 parentId 指向谁。永远不覆盖、不删除旧节点。
4. **服务端生成 ID，服务端持久化。** 前端描述意图，服务端决定结果。
5. **原子写入。** 先写 tmp，再 rename。你的对话数据比你的代码更不能丢。

## 附：不用服务端也能跑吗

能。第4章已经证明了——`getPath()`、`renderBranchNav()`、`walkToLeaf()`、`switchBranch()` 全部是纯前端逻辑，不需要任何网络请求就能完整运行。对话树本身是一个**纯逻辑的数据结构**——节点、parentId、currentLeaf——这些代码不关心自己住在哪里。

第3、5、6章围绕"服务端权威"展开，但那是一个**架构选择**，不是对话树的前置条件。本篇选择它是因为实际场景需要跨设备同步、崩溃恢复和多标签页隔离。如果你的需求更简单——只想在本地浏览器里体验对话分叉——把第3章的 `addNode` 从 fetch 换成 localStorage 读写就行：

```js
// 纯本地版的 addNode — 逻辑和服务端版完全一样，只是存储换了位置
function addNode(parentId, role, content) {
  const tree = JSON.parse(localStorage.getItem('tree'));
  const node = {
    id: 'm_' + Date.now().toString(36) + '_' + Math.random().toString(36).slice(2,7),
    role, content, parentId, ts: Date.now()
  };
  tree.nodes[node.id] = node;
  tree.currentLeaf = node.id;
  localStorage.setItem('tree', JSON.stringify(tree));
  // 别忘了在渲染前调 rebuildChildren() 重建 children 数组
  return node;
}
```

第4章的 `switchBranch()` 本来就不走网络——直接拿来用。`rebuildChildren()` 同理。纯本地版需要替换的只有涉及 `fetch` 的部分（addNode、第5章版的 switchBranch），其余函数一个字都不用改。

纯本地版丢掉的东西：换浏览器就看不到（没有跨设备）、清缓存就没了（没有持久备份）、两个标签页同时操作可能互相踩（没有并发控制）。得到的东西：零部署门槛，一个 HTML 文件拖进浏览器就能用。

---
.177...

<sub>Architecture & documentation: Opus 4.6</sub>
