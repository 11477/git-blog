---
title: "Multica 平台设计原理：把编码 Agent 当团队成员管理"
date: 2026-06-30
tags: ["架构", "AI", "Go", "Agent"]
categories: ["架构拆解"]
summary: "读源码拆解 Multica——一个开源 managed agents 平台：控制面/数据面分离、Postgres 工作队列、异构 agent 统一接入、Squad 的去中心化路由，以及 core/ui/views 三层前端复用"
draft: false
---

> 这是「架构拆解」系列第一篇。前两篇 [LSP](../language-server-protocol原理/) 和 [CodeGraph](../codegraph原理/) 拆的是单个工具的原理；这个系列拆的是完整平台的架构。本文内容基于对 Multica 开源源码的直接阅读，不是黑盒推断。

## 一句话定位

Multica 是个开源的 **managed agents 平台**：把编码 agent（Claude Code、Codex、Copilot CLI、Cursor、Gemini 等十几种）当作团队成员来管理——像给同事派活一样给 agent 分配 issue，它自己领任务、写代码、报阻塞、更新状态。

名字致敬了 1960 年代的分时操作系统 **Multics**：当年是多个用户分时共享一台机器，现在是**人类和 AI agent 分时共享一个工作系统**。

技术骨架：**Go 单体后端 + 用户本地 daemon + pnpm/turbo 前端 monorepo（web/desktop/mobile）+ PostgreSQL(pgvector)**。

## 一、最核心的架构决策：控制面 / 数据面分离

这是理解整个平台的钥匙。

```
   ┌─────────────────────────────┐         ┌──────────────────────────────┐
   │   服务器（控制面）            │  HTTP   │  用户本机 daemon（数据面）      │
   │   Go 后端 + Postgres         │◄───────►│  真正拉起 agent 进程的地方       │
   │   - 任务状态权威             │  轮询    │  - 检测本机装了哪些 agent CLI   │
   │   - 编排/调度/广播           │  claim   │  - 领任务 → 起子进程 → 流式回传  │
   └─────────────────────────────┘         └──────────────────────────────┘
```

**关键：服务器从不直接跑 agent。** 真正执行编码 agent 的进程跑在**用户自己机器上的 daemon** 里。服务器只是状态权威和协调者。

为什么这样设计？因为 Claude Code、Codex 这些 CLI 本来就装在开发者机器上、用的是开发者本地的登录态和代码仓库。Multica 不去复制这套环境，而是**统一驱动本机已有的 CLI**——这是它能"一套接入十几种 agent"的物理基础。

## 二、任务生命周期：一个 Postgres 工作队列

README 说的 `enqueue → claim → start → complete/fail` 是一套真实的状态机，落在 `agent_task_queue` 表：

```
∅ → queued → dispatched → running → completed
                  ↘ waiting_local_directory ↗   ↘ failed
        任何活跃态 → cancelled
```

**并发控制是整个系统的精华。** 多个 daemon 同时来抢任务，怎么保证不会抢到同一个？答案是经典的 Postgres 工作队列模式：

```sql
... FOR UPDATE SKIP LOCKED       -- 行级锁 + 跳过已锁行，并发认领永不撞车
    AND NOT EXISTS (... status IN ('dispatched','running'))  -- 同一 agent 对同一 issue 只能有一个活跃任务
ORDER BY priority DESC, created_at ASC                        -- 优先级 + FIFO
```

两个值得记住的设计：

- **prepare lease（准备租约）**：任务被认领后、agent 真正开跑前，daemon 要解析仓库、物化输入，这段时间靠短租约续命证明自己活着。一旦不续约，任务被快速回收，不必等全局长超时窗口。
- **双通道：唤醒提示 vs 正确性分离**。WebSocket 推送只发 "best-effort 唤醒提示" 让 daemon 低延迟反应；真正的正确性由 `FOR UPDATE SKIP LOCKED` + HTTP 轮询兜底。**任一通道挂掉系统仍正确，只是慢一点**——非常稳健的工程哲学。

## 三、统一接入异构 Agent（最有看点的部分）

怎么用一套接口接入 Claude Code、Codex、Cursor、Kimi 等命令行调用方式、输出格式、认证全都不同的 agent？

统一接口只有一个方法：

```go
type Backend interface {
    Execute(ctx, prompt string, opts ExecOptions) (*Session, error)
}
```

输入用 `ExecOptions` 归一（cwd / model / systemPrompt / mcpConfig…），输出归一成统一的 `Message` 流（text / thinking / tool-use / tool-result / error）加最终的 `Result`。

**真正的秘密不是一个巨型 adapter，而是把十几种 agent 收敛成三种通信范式：**

| 范式 | 代表 agent | 机制 |
|------|-----------|------|
| **stream-json** over stdio | Claude、CodeBuddy、Cursor | 逐行 scan stdout 的 SDK 事件 |
| **app-server** JSON-RPC | Codex | 标准 JSON-RPC 调用 |
| **ACP**（Agent Communication Protocol，JSON-RPC 2.0） | Hermes、Kimi、Kiro、Qoder | 共享一套 client + helper |

新增一个 ACP agent 几乎只是写一个薄 backend——参考实现里成片的 helper 都标注着 "shared by all ACP backends"。

**适配层反复出现的主题是"headless 自治"**：daemon 没有交互 UI，所以所有 agent 的权限请求 / 审批 / 提问都被自动批准或禁用（Claude 的 `control_request` auto-approve、禁用 `AskUserQuestion`、ACP 的 `approve_for_session`、Hermes 的 `HERMES_YOLO_MODE=1`）。

还有个体现工程成熟度的细节——**治理"假成功"**：ACP agent 即使底层 LLM 报 429/400 错误，仍会返回 `stopReason=end_turn` 假装成功。Multica 用一个 stderr 嗅探器抓 provider 错误，把"假完成"纠正为真正的 failed。这种"假设上游 CLI 会骗你"的防御性归一，很能说明问题。

## 四、Squads：把路由决策交给 LLM 而非中心化调度器

README 说 Squads 是"把 agent 编成小队，leader 把任务委派给合适的成员"。直觉上后端应该有个能力匹配的路由算法——**结果没有**。

它的做法很聪明：**用 prompt 工程取代中心化路由器**。

1. 任务指派给 squad 时，入队给 **leader agent**，打上 `is_leader_task=true`
2. daemon 认领时注入一段 **squad-leader briefing**：操作协议（"只协调不干活、按成员技能匹配、用 @mention 委派、委派后停止"）+ **花名册**（每个成员的名字、角色、技能、可直接粘贴的 `[@Name](mention://agent/UUID)` mention 链接）
3. leader 发一条 @mention 评论 → 触发评论 trigger → 给被点名的成员入一个普通任务

于是**路由 = "leader 写评论 @ 谁"**，复用标准的入队 / 认领管线。leader 的"智能"来自 LLM 读花名册里的技能自己选人，后端只负责把这个决策物化成评论触发。这是把决策从代码挪进 prompt 的典型去中心化设计。

## 五、Skill 复用与"compound（累积）"

skill 是带 YAML frontmatter 的 `SKILL.md`，存在 workspace 级数据库表（`skill` / `skill_file` / `agent_skill`），是**团队共享资产**。

**复用靠"写进每种 CLI 的原生发现目录"，而不是自造加载器**：

| Agent | skill 注入路径 |
|-------|---------------|
| Claude | `.claude/skills/` |
| Copilot | `.github/skills/` |
| Cursor | `.cursor/skills/` |
| Codex | `CODEX_HOME/skills/`（特殊，因为它重定向 HOME） |

同一份 workspace skill 因此能在十几种 agent 上即插即用。

**"compound" 是一个闭环**：agent 运行时用 `multica skill create` 把一次解决方案沉淀成 workspace skill → 关联给 agent → 下次 dispatch 时注入 → 被复用。因为 skill 是团队范围共享的，能力随时间累积。前后端还各有一套 frontmatter 解析器（Go + TS），刻意保持一致，保证同一份 SKILL.md 两端解析结果相同。

## 六、前端：core / ui / views 三层 + 依赖注入

monorepo 的复用边界**按"能不能碰 DOM"切**，非常清晰：

```
@multica/core   ← headless 逻辑（API client、zod、WS、react-query、zustand）  三端通吃
@multica/ui     ← DOM 基础组件（shadcn 风格）
@multica/views  ← 业务整页（issues / chat / settings…）  web + desktop 复用
```

- **web（Next.js）和 desktop（Electron）复用到整页业务组件**——desktop 几乎是"Electron 套了一层 web 的 views"
- **mobile（Expo / RN）因为 DOM 不可用，只复用 core 的"大脑"，视图用 RN 原语重写**
- 三端差异通过一个 **`CoreProvider`** 收敛：注入各自的 storage 适配器（web 用 cookie、desktop 用 Electron、mobile 用 secure-store）、identity、wsUrl——同一个 Provider 注入不同适配器，跑出三个端

两个值得写的工程细节：

1. **zod 作为"API 漂移防御边界"**：`parseWithFallback` 用宽松 schema（枚举用 `z.string()` 而非 `z.enum`）解析响应，失败不抛异常而是返回 fallback + 告警日志。把"已安装的桌面客户端连到更新后端导致白屏"的事故，降级成"退化渲染但仍可用"。代码注释里直接挂着真实事故编号。

2. **实时进度流统一沉淀到 react-query 缓存**：WS 收帧 → `useRealtimeSync` 集中翻译 → 各领域 `ws-updaters` 精确 patch 缓存或选择性 invalidate。UI 组件对 WebSocket 完全无感，只是订阅 query——这是看板实时进度"不闪烁"的关键。

## 七、部署形态

- **`make selfhost` 一键起**：自动生成密钥 → 拉 GHCR 镜像 → 健康检查，起 3 个容器：`pgvector/pg17`（用了向量扩展，说明有语义搜索）+ Go 后端（启动时先跑 migrate 再起 server）+ Next.js standalone 前端
- 安全默认**只绑 `127.0.0.1`**，强制前置反代终止 TLS
- 额外用 **goreleaser** 发布跨平台 CLI 二进制 + Homebrew tap，把本机变成 agent 运行节点
- 另有 Helm chart 支持 K8s 部署

## 总结：三个设计哲学

1. **控制面 / 数据面分离**——服务器管状态，本地 daemon 管执行。这让它天然支持任意本地 agent，且代码和登录态不出用户机器。
2. **用 LLM prompt 取代中心化逻辑**——Squad 路由不是算法，是给 leader agent 注入花名册 + 操作手册让它自己决策。
3. **对不可信的上游做防御性归一**——无论是 agent CLI 的"假成功"输出（后端 stderr 嗅探），还是后端 API 的契约漂移（前端 zod fallback），都体现了"假设上游会变、会骗你"的成熟工程思维。
