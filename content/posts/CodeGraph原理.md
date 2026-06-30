---
title: "CodeGraph 是怎么实现的：给 AI agent 的代码图谱"
date: 2026-06-30
tags: ["技术", "代码智能", "AI"]
categories: ["技术"]
summary: "读源码拆解 CodeGraph：web-tree-sitter 解析、SQLite+FTS5 存储、Random Walk with Restart 结构排序——一个 100% 本地、零 embedding 的代码图谱工具"
draft: false
---

> 这是「代码智能工具原理」系列第二篇。上一篇讲了 [LSP](../language-server-protocol原理/) 如何服务"人在编辑器里写代码"；这篇讲 CodeGraph 如何服务"AI 帮你读代码"。本文内容基于对其开源源码（`@colbymchenry/codegraph`，纯 TypeScript）的直接阅读，不是黑盒推断。

## 一、它和 LSP 走了相反的路

LSP 是"逐次问答"——你点一下，语言服务器实时算一次，返回一个精确的小结果，服务的是人在编辑器里逐字符的交互。

CodeGraph 的目标用户是 **AI agent**。AI 不想要 20 次往返，它要的是"把这个流程相关的几个函数源码 + 它们的调用关系**一次性**给我"。于是 CodeGraph 走了另一条路：**预先把整个仓库解析成一张图，存进本地数据库，查询时直接走图**，把结果聚合成一大块带结构的上下文。

| 维度 | LSP | CodeGraph |
|------|-----|-----------|
| 何时计算 | 逐次实时算 | 预建索引，查时只读图 |
| 怎么找相关 | 精确语义单点 | 全文检索 + 随机游走结构排序 |
| 服务对象 | 人（编辑器逐字符） | AI（一次性拿整个相关子图 + 调用路径） |
| 响应形态 | 单点结果 | 子图 + 源码 + 调用路径打包 |

## 二、技术选型

三个核心选择：

- **解析：web-tree-sitter（WASM）** —— 跨平台零编译，按需懒加载语法，只编译项目里实际出现的语言。
- **存储：SQLite（WAL 模式）** —— 用 Node 20 内置的 `node:sqlite`，不依赖原生模块，索引落在 `<project>/.codegraph/codegraph.db`。
- **检索：纯结构化，零 embedding** —— 这是最反直觉的一点。它不用任何向量/语义模型，而是 FTS5 全文检索 + 图遍历 + 随机游走算法。源码里明确写着 "no embeddings"。

## 三、索引流程

`codegraph init` 触发一条严格分阶段的管线：

```
scan → parse → store → resolve → 后处理 → 盖版本戳
```

1. **scan**：优先用 `git ls-files` 列文件——天然尊重各级 `.gitignore`、自动递归 submodule；非 git 项目才退回文件系统遍历。
2. **parse**：tree-sitter 在 **worker 线程池**里并行解析，主线程独占写 SQLite（因为 SQLite 非线程安全）。产出节点、边，以及**未解析引用（unresolved refs）**。
3. **store**：按文件入库，用内容 hash 去重。
4. **resolve**：把 unresolved refs 批量绑定成 `calls` / `imports` / `extends` 等边。
5. **后处理**：再跑两遍解析链式调用、继承来的 `this.member` 等复杂情况。
6. **盖版本戳**：把抽取器版本号写进 metadata 表，供后续判断索引是否过期。

为什么"抽取"和"解析引用"要分两步？因为单个文件解析时，`foo()` 这个调用指向谁可能在别的文件里——必须等所有文件的符号都入库后，才能跨文件把"名字"绑定到"定义节点"。

## 四、几个值得细看的设计

### 1. 声明式语言抽象层

核心引擎 `tree-sitter.ts`（近 6000 行）是**语言无关**的。每新增一门语言，不需要改引擎，只需写一个**配置对象** `LanguageExtractor`，声明：

- 哪些 AST 节点类型算 `function` / `class` / `method`
- name 字段叫什么、body 字段叫什么
- 少量可选 hook（如何取函数签名、如何判断接收者类型）

Go 的抽取器只有约 380 行配置。正是这个抽象层让它能支持 20 多种语言（TS/JS、Python、Go、Rust、Java、C/C++、C#、PHP、Ruby、Swift、Kotlin 等），外加 Vue/Svelte、MyBatis XML、Spring properties 这类无标准语法的自定义抽取器。

### 2. 存储 schema

四张核心表：

| 表 | 作用 |
|----|------|
| `nodes` | 符号节点：函数/类/方法等，含位置、签名、可见性、是否导出 |
| `edges` | 关系边：`calls`/`imports`/`extends`/`implements`，带 `provenance` 标记来源 |
| `files` | 文件元信息，`content_hash` 用于增量检测 |
| `unresolved_refs` | 抽取期产出、待 resolve 消费的引用 |

外加一张 **FTS5 虚拟表 `nodes_fts`**，靠 SQLite 触发器自动和 `nodes` 同步——这是文本检索的核心。`edges` 表上有个 `provenance` 字段尤其有意思，它标记一条边是 tree-sitter 静态解析出来的，还是后面**启发式合成**的（比如 observer 模式、EventEmitter、React setState 这类静态分析看不到的动态派发，会被专门的逻辑补出来并打上 `heuristic` 标签）。

### 3. 杀手锏：Random Walk with Restart 排序

这是整个项目最精彩的设计。`explore` 命令排序相关性用的不是文本得分，而是**个性化 PageRank（带重启的随机游走）**：

- 从查询命中的种子节点出发，在调用/引用图上做无向随机游走
- 重启概率 α = 0.25，幂迭代直到收敛

效果是给出文本检索给不出的「**结构相关性**」：和命中簇调用相连的文件，会累积游走质量而排到前面；纯属文本巧合命中的孤立文件，只拿到自己那点 restart 概率，排到接近 0。这套算法**确定性、无需 embedding、还绕开了分词陷阱**。

### 4. explore 的完整检索管线

```
从查询抽符号名
  → 精确名查找 + FTS5 全文检索 + CamelCase 边界匹配（多通道 merge）
  → 从每个入口做 BFS 遍历子图（优先级 contains > calls > 其它边）
  → Random Walk with Restart 重排
  → 按文件聚合源码 + 生成 "Call paths" 段（对 calls 边做有界 DFS）
```

还有个工程细节：**自适应输出预算**。它按项目文件数分档调整输出上限（小项目约 13K 字符，大项目封顶约 24K），刻意压在 AI agent 的约 25K 内联结果上限之下——否则结果会被 host 落盘再 Read 回来，反而更慢、更费 token。

### 5. 三模式 daemon 架构

MCP 层有三种运行模式：

- **direct**：单进程 stdio，opt-out 或无索引时用
- **proxy**：host 实际连接的进程，本地秒答握手，把调用转发给 daemon
- **daemon**：detached 后台进程，通过 Unix socket 服务多个 proxy，**共享同一份图 + SQLite 句柄 + 文件 watcher**

文件 watcher 用原生 OS 事件（macOS FSEvents、Linux inotify）加去抖做增量更新，约 1 秒写入延迟。多个编辑器窗口连同一个项目时，共享一个 daemon，不重复建索引。

### 6. WASM 内存管理的小聪明

WASM 的线性内存只增不减，长时间解析大仓库会内存膨胀。它的对策很直接：worker **每解析 250 个文件就销毁整个线程重建**来回收堆；单文件解析超时 10 秒就重启 worker；解析失败的文件还有"干净 worker 重试 → 去掉注释再重试"的两级 fallback。

## 五、一句话总结

CodeGraph = **web-tree-sitter(WASM) 声明式抽取符号与关系** → **存进 SQLite + FTS5** → **查询时用全文检索定位种子、BFS 取子图、Random Walk with Restart 算结构相关性排序**，最后把相关源码 + 调用路径聚合成一大块上下文喂给 AI。

它和 LSP 共享了"解析 AST、理解符号关系"的底层思路，但优化目标截然不同：LSP 优化"人在编辑器里的逐次精确交互"，CodeGraph 优化"AI 一次性获取结构化的代码上下文"。而且全程在本地完成，不依赖任何远程模型或 embedding 服务。
