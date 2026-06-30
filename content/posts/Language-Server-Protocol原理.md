---
title: "Language Server Protocol (LSP) 是什么，以及它的实现原理"
date: 2026-06-30
tags: ["技术", "协议", "IDE"]
categories: ["技术"]
summary: "从 N×M 问题到 JSON-RPC：拆解 LSP 的设计动机、协议机制与语言服务器的内部实现"
draft: false
---

## 一、它解决的问题：从 N×M 到 N+M

在 LSP 出现之前，编辑器要支持某门语言的智能功能（补全、跳转、报错），必须单独写一套实现。假设有 **M 个编辑器**和 **N 种语言**，理论上需要 **M × N** 套集成代码——组合爆炸。

微软在 2016 年提出 **Language Server Protocol**，把它变成了 **M + N**：

- 每种语言只实现一个 **Language Server**（语言服务器）
- 每个编辑器只实现一个 **LSP Client**（客户端）
- 两者通过统一协议通信

```
            没有 LSP                        有了 LSP
   VSCode ─┬─ Python                VSCode ─┐
   Vim   ─┼─ Go        →            Vim   ─┼─ LSP ─┬─ pyright (Python)
   Emacs ─┴─ Rust                   Emacs ─┘       ├─ gopls (Go)
   (每条连线都要单独实现)                            └─ rust-analyzer (Rust)
```

## 二、它提供什么能力

语言服务器把"理解代码"的能力通过协议暴露出来：

| 功能 | 说明 |
|------|------|
| 代码补全 | Autocomplete / IntelliSense |
| 跳转定义 | Go to Definition |
| 查找引用 | Find References |
| 悬停提示 | Hover（显示类型、文档） |
| 错误诊断 | Diagnostics（红/黄波浪线） |
| 重命名 | Rename Symbol（跨文件） |
| 代码格式化 | Formatting |
| 符号搜索 | 文件/工作区符号导航 |

## 三、实现原理

### 1. 底层：JSON-RPC 2.0 消息

LSP 的所有通信都是 **JSON-RPC 2.0** 消息，分三类：

```jsonc
// Request（需要响应，带 id）
{ "jsonrpc": "2.0", "id": 1, "method": "textDocument/definition", "params": {} }

// Response（对 request 的回复，id 对应）
{ "jsonrpc": "2.0", "id": 1, "result": {} }

// Notification（单向，无 id，不需回复）
{ "jsonrpc": "2.0", "method": "textDocument/didChange", "params": {} }
```

消息在传输时前面带一个类似 HTTP 的头部，用 `Content-Length` 标明字节数，便于接收方从字节流中切分消息：

```
Content-Length: 126\r\n
\r\n
{"jsonrpc":"2.0","id":1,"method":"textDocument/completion",...}
```

### 2. 传输层：两个独立进程

```
┌─────────────┐   stdin/stdout (或 socket/pipe)   ┌──────────────────┐
│   Editor    │ ──────────  JSON-RPC  ──────────→ │ Language Server  │
│ (LSP Client)│ ←──────────────────────────────── │  (独立进程)       │
└─────────────┘                                    └──────────────────┘
```

- 编辑器启动时**把语言服务器作为子进程拉起**（如 `gopls`、`rust-analyzer`）
- 默认通过 **stdio** 通信：client 写到 server 的 stdin，读 server 的 stdout
- 也支持 TCP socket、命名管道——这让服务器能用任意语言写、甚至跑在远程

关键点是**进程隔离**：语言服务器崩溃不会拖垮编辑器，语言服务器也不需要知道编辑器是谁。

### 3. 生命周期与文档同步

**握手协商**

```
Client → initialize        （我支持哪些能力？capabilities）
Server → initializeResult   （我能提供哪些能力？capabilities）
Client → initialized        （确认，开始干活）
```

双方交换 **capabilities**——比如 client 说"我支持增量同步、支持 markdown 悬停"，server 说"我能做补全、跳转、重命名"。后续行为按协商结果走。

**文档同步（核心难点）**

服务器必须在内存里维护一份和编辑器一致的文档副本，否则解析的是过期代码：

```
打开文件 → textDocument/didOpen   （发全文）
用户敲键 → textDocument/didChange （发增量 diff，含行列范围 + 新文本）
保存    → textDocument/didSave
关闭    → textDocument/didClose
```

`didChange` 用**增量更新**：只传"第 10 行第 5 列到第 10 行第 8 列，替换为 xxx"，而不是整个文件重传——这是性能关键。

### 4. 语言服务器内部怎么算出结果

这才是"智能"真正发生的地方。LSP 协议本身不规定算法，由各服务器实现，但典型流程是：

```
源码文本
  │  词法分析 (Lexer)
  ▼
Token 流
  │  语法分析 (Parser)
  ▼
AST（抽象语法树）
  │  语义分析：符号表、类型推导、作用域解析
  ▼
语义模型（哪个变量指向哪个定义、什么类型）
  │
  ├─ textDocument/definition → 查符号定义位置
  ├─ textDocument/completion → 按当前作用域算候选
  ├─ textDocument/references → 反向索引找所有引用
  └─ textDocument/publishDiagnostics → 主动推错误给编辑器
```

现代语言服务器为了响应快，普遍用两个技术：

- **增量编译 / 增量解析**：改一行不重新解析整个项目，只重算受影响的部分。`rust-analyzer` 的 salsa 框架就是干这个的——一套"查询 + 缓存 + 失效传播"的增量计算引擎。
- **惰性计算**：只在被请求时才算某个文件的语义，不预先全量编译。

> 注意 `publishDiagnostics` 是 **server → client 的 notification**——你打完字、服务器后台算完，主动把错误推给编辑器画波浪线，而不是编辑器轮询。

### 5. 一次"跳转定义"的完整时序

```
用户 Ctrl+点击 foo()
  │
Editor → textDocument/definition { uri, position: {line, char} }
  │
Server: 在 AST 里定位 position 落在哪个标识符
        → 查符号表得到 foo 的声明节点
        → 转成 {uri, range}
  │
Server → result: { uri: "bar.go", range: {start, end} }
  │
Editor 跳转并高亮到 bar.go 对应位置
```

## 四、常见的语言服务器

- `gopls` — Go
- `rust-analyzer` — Rust
- `pyright` / `pylsp` — Python
- `typescript-language-server` — TypeScript / JavaScript
- `clangd` — C/C++

## 五、一句话总结

LSP = **JSON-RPC 消息**（协议）+ **进程隔离的 client/server**（架构）+ **文档增量同步**（保证 server 内存副本与编辑器一致）+ **server 内部的 AST/语义分析 + 增量计算**（产生智能）。

协议只定义"问什么、答什么格式"，把"怎么算出来"完全留给语言服务器自己——这正是它能写一次、被所有编辑器复用的根本原因。
