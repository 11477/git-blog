---
title: "Melos Agent 设计总结：用角色拆分 + Spec 驱动构建多 Agent 研发流水线"
date: 2026-07-01
tags: ["架构", "AI", "Agent", "SDD", "Melos"]
categories: ["架构拆解"]
summary: "拆解 Melos 后端 Agent 工作流的设计全局：11 个角色 Agent 的拆分逻辑、7 阶段 Spec 驱动流水线、25 个 Skill 的分层体系、四层知识查询优先级、前后端镜像与分化策略，以及文件系统为真的协作哲学。"
draft: false
---

> 这是「架构拆解」系列的新一篇。前篇 [Multica 平台设计原理](../multica平台设计原理/) 从源码层面拆了 Multica（现 Melos）这个 managed agents 平台本身——控制面/数据面分离、Postgres 工作队列、异构 agent 统一接入。这篇补另一侧视角：**在这个平台上，怎么设计一套能真正跑起来的 Agent 协作系统。**

本文内容基于 Melos 后端 Agent 工作流的实际设计文档和 11 个 Agent 指令源文件，不是黑盒推断。

## 一句话定位

这套设计要解决的核心问题是：**不用一个大模型包办全流程，而是把研发流程拆成 11 个各有专长的 Agent，让它们通过 Issue 评论互相委派，像真实团队一样协作。** 每个 Agent 拿到的是明确的单一角色（"你是开发总负责人，只负责调度，别写代码"），而非"你是全栈工程师，啥都你干"。

技术底色：**多 Agent（Claude Code）+ Spec-Driven Development（SDD）+ Issue 驱动 + Mention 协作**。

## 一、最核心的设计决策：角色拆分，而非能力堆叠

这是理解整套设计的第一把钥匙。

直觉上，给一个 Agent 塞更多 Skill、更长的 system prompt，它应该更能干。但实际上正相反——Agent 能执行的指令密度有上限，prompt 越长，"注意力的分配"越稀释。一个同时被告知"你要写代码、写测试、检查规范、做 code review、写 commit message"的 Agent，往往哪个都做不精。

这套设计的答案是把研发流程**按角色拆成 11 个 Agent**，每个只拿一个小但明确的职责上下文：

```
pm-bd (产品经理，入口)
  ↓ 需求分析→PRD→分流
bd-common-API架构师 (接口设计枢纽，前后端均需时出场)
  ↓ 接口文档→跨端评审→创建开发子 Issue
bd-common-开发总负责人 (流程调度中枢，只调度不干事)
  ↓ 按 7 阶段流水线依次委派
① git管理员 → ② 技术探索专家(可选) → ③ Spec制品生产者 → ④ 开发工程师
  → ⑤ 质量工程师 → ⑥ 测试工程师 → ⑦ 代码归档员
独立: bd-common-AI套件管理员 (初始化 AI 配置)
```

拆分的粒度怎么定的？答案是**参考真实团队的岗位边界**——产品经理、API 架构师、开发工程师、测试工程师这些角色本来就是按职责隔离的，Agent 拆分直接套用这个边界。一个关键设计是：

- **"总负责人"只做调度**：它的 system prompt 里写的是"绝对禁止自己执行任何操作：不得编写代码、执行测试、调用 SDD skill"。所有工作必须通过 Mention 委派给执行 Agent。
- **每个 Agent 拿的上下文是"够用但不超"的**：比如开发工程师不查 Deep-Wiki（项目业务背景），因为 Spec 制品生产者已经把这些信息固化到 specs 里了；质量工程师不查 knowledge base（团队踩坑），因为编码规范已经写在 `.claude/rules/` 里了。

这背后是一个隐含假设：**前序 Agent 的产物 → 后序 Agent 的输入，Agent 之间不需要共享上下文，信息通过制品文件传递。** 即"文件系统为真"。

## 二、"文件系统为真"：Agent 间协作的状态传递哲学

Agent 没有共享内存，也不共享聊天历史。当开发总负责人把任务从 Spec 制品生产者手里转到开发工程师手里时，后者看不到前者跟总负责人聊了什么。

这套设计的答案是：**所有需要传递的信息，落成文件。** 上游 Agent 往仓库里写 markdown 制品，下游 Agent 启动时读取这些制品作为输入的"事实源"。

具体的信息流是这样的：

```
① git管理员：仓库检出 + 切分支
        ↓ (交付: 干净的代码仓库)
② 技术探索专家：方案调研
        ↓ (交付: design.md)
③ Spec制品生产者：查代码 → 查 wiki → 生成制品
        ↓ (交付: proposal.md + specs.md + db-spec.md + tasks.md)
④ 开发工程师：按 tasks.md 逐条实施
        ↓ (交付: diff + 打钩的 tasks.md)
⑤ 质量工程师：编译 + 静态分析 + DB Migration 校验
        ↓ (交付: 验证报告)
⑥ 测试工程师：契约测试 + 集成测试 + DB 校验
        ↓ (交付: 测试报告)
⑦ 代码归档员：commit + push + 知识沉淀 + wiki 增量更新
```

制品序列本身就是一个信息压缩链——每一层的 Agent 只需要理解上一层的产物，不需要理解再往上的原始上下文。Spec 制品生产者帮开发工程师"消化"了代码结构和技术背景，开发工程师只面对精炼过的 specs 和 tasks。

**但这里有一个精心设计的信息断层：两个"关键阶段"(③ Spec 完成、⑥ 测试完成)会停下来，mention issue 负责人（真人），等确认了再继续。** 这相当于在自动化流水线中打入了两个人机协作的断点，防止 AI 在关键决策点上一条道走到黑。

## 三、四层知识查询体系：Agent "遇到不知道的事"该问谁

Agent 不是知识的终点——它在编码过程中一定会遇到"不知道"的情况。这套设计给出的答案是**一个从代码到文档、从局部到全局的查询优先级链**：

```
第零层：CodeGraph（代码符号 + 调用链）
  ↓ 找不到/需要理解意图
第零点五层：Deep-Wiki（项目业务背景 + 架构约定 + 术语）
  ↓ 找不到/需要团队经验
第一层：Knowledge（团队踩坑）+ RDS Spec（公司 DB 规范）
  ↓ 找不到/需要官方文档
第二层：Context7（开源框架 API 文档）
```

每一层查不到才往下沉，避免"框架 API 不知道怎么用直接去问 Deep-Wiki"这种信息源错配。

但这套体系最精巧的设计不是分层本身，而是**不给所有 Agent 开放全部层级**——哪个 Agent 能访问哪层知识，是严格收敛的：

| 知识层级 | 能查的 Agent | 不能查的 Agent（及理由） |
|---------|-------------|----------------------|
| CodeGraph | API 架构师 / 技术探索 / Spec 制品 / 开发 / 质量 / 测试（6 家） | pm-bd（不读代码）/ 总负责人（只调度）/ git 管理员（只检出）/ 归档员（不读新代码） |
| Deep-Wiki | API 架构师 / 技术探索 / Spec 制品（3 家查询 + wiki 缺失检测） | 开发/质量/测试工程师（业务上下文从 specs 获取，避免重复查询和 specs 不一致） |
| Knowledge | 技术探索 / 开发 / 质量 / 归档（4 家） | Spec 制品（信息应在 specs 中落实，不是模糊经验）/ 测试工程师（测试逻辑从 specs 驱动） |
| Context7 | 6 家（条件触发） | 同上，但不是每次必查，只在遇到"开源框架 API/配置/版本差异"时触发 |

**收敛的理由是成本和可靠性**：Deep-Wiki 的 `/deep-wiki:generate` 多轮迭代"很贵"（token 消耗大），开发阶段如果每个 Agent 都去查一遍，不仅浪费，更危险的是——如果 Deep-Wiki 中的描述已经滞后于 specs（specs 里说了要改 A 表，wiki 还在描述旧的 B 表结构），开发工程师自己查 wiki 会得到矛盾信号。

所以设计的策略是：**让前置的"信息整合者"（API 架构师 / Spec 制品生产者）把 wiki 信息消化后固化到制品里，下游执行者只吃精炼过的制品。** 这是一种信息消费分层，和软件工程里"不要让每个函数都去读配置文件"是一样的道理。

## 四、SDD（Spec-Driven Development）：把"该写什么代码"标准化成制品序列

SDD 是这套系统的方法论骨架。核心思想是：**在写代码之前，先写完一套完整的规格制品，让规格制品的质量决定了代码的质量。**

整个 SDD 体系由 11 个 Skill 串成一条链：

```
sdd-brainstorm         独立探索工具，可随时调用
    ↓
sdd-new-change         创建变更 + 需求完整性检查（6 项：背景/功能/接口/数据字典/DB Migration/非功能需求）
    ↓
sdd-continue (标准)    逐个创建制品：proposal → specs → db-spec → design → tasks
sdd-ff (快速)          一次性生成全部制品（并行 dispatch subagent）
    ↓
sdd-review-spec        审查制品质量：接口完整性/DB 变更合理性/数据模型一致性/非功能量化
    ↓
sdd-code               按 tasks.md 批次编码（含 DB Migration 专项步骤）
    ↓
sdd-verify             验证：Tasks 完成度 + 编译 + 静态分析 + DB Migration 校验
    ↓
sdd-fix                自动修复（最多两轮，仍失败则人工介入）
    ↓
sdd-ship               归档 + 合并 + 飞轮（知识沉淀：DB 踩坑/中间件选型/性能调优经验）
```

后端版和前端的核心差异在于制品序列——后端多了一个 `db-spec`：

| 前端制品序列 | 后端制品序列 |
|-------------|-------------|
| proposal → specs → ui-spec → design → tasks | proposal → specs → **db-spec** → design → tasks |

`db-spec` 不是简单"创建一个 DDL 文件"，它是一个结构化的 DB 变更规格模板，包含：DDL 语句、Migration 脚本（版本化 + 幂等性）、索引策略、数据迁移方案（全量/增量/双写）、回滚方案。质量工程师在校验阶段会逐项检查 DDL 语法、字段类型与 spec 一致性、索引覆盖查询字段、NOT NULL/DEFAULT 约束。

**验证-修复闭环**是一个体现工程成熟度的细节设计：

```
sdd-verify
    ├─ Tasks 未完成 → 返回 sdd-code（重新编码）
    ├─ 编译/静态分析/DB Migration 有问题
    │     ├─ fix-r1.md 不存在 → sdd-fix（第一轮自动修复）
    │     │     └─ 修复后 → sdd-verify（重新验证）
    │     ├─ fix-r2.md 不存在 → sdd-fix（第二轮自动修复）
    │     │     └─ 修复后 → sdd-verify（重新验证）
    │     └─ fix-r2.md 存在 → 停止盲修，列出错误，人工介入
    └─ 全部通过 → 测试工程师
```

最多两轮自动修复，不是无限循环。`fix-r<N>.md` 文件本身就是"修复轮次计数器"——第二轮修完仍未通过时，不给你第三轮机会，而是把问题摊给用户。这在实践中避免了 Agent 在修复循环里来回拉锯烧 token 的常见死亡螺旋。

## 五、前后端的镜像与分化：同构但不等价

整套设计是**先有前端工作流（fd-common），再镜像出后端（bd-common）**。但镜像不是复制粘贴——每个层面的"后端差异化"是刻意的：

| 维度 | 前端 (fd-common) | 后端 (bd-common) | 差异原因 |
|------|-----------------|-----------------|---------|
| ③制品序列 | proposal → specs → ui-spec → tasks | proposal → specs → **db-spec** → tasks | 后端有 DB 变更，前端有 UI 规格 |
| ④编码 | React 19 + TS + Antd 6（固定技术栈） | **多技术栈分支**（Java/Go/Python） | 前端团队技术栈统一，后端多栈并存 |
| ⑤质量验证 | tsc + ESLint（固定工具链） | 编译 + 静态分析 + **DB Migration 校验**（按栈分支） | 后端多了 DB Migration 这一整类风险 |
| ⑥测试 | Mock 接口 + E2E Playwright | **契约测试 + 集成测试 + DB 校验** | 后端测试是 API 契约验证 + 业务逻辑 + 数据层校验三层 |
| AI 套件管理员 | fetk-ai（Front End Tech Kit） | betk-ai（Back End Tech Kit） | 两套独立配置文件 + CLI 命令 |
| Skill 数量 | 24 个前端 Skill | 25 个后端 Skill（含 2 个跨端共用的 cooper + melos-agent-lookup） | DB 相关 Skill（db-spec/contract-test/integration-test/db-verify/rds-spec）为后端独有 |

**但 Agent 总负责人、git 管理员、技术探索专家、代码归档员这四个角色的职责模型是完全同构的**——只是内嵌的 Skill 不同（比如技术探索专家在后端侧重 DB 选型/中间件/分库分表，前端侧重 UI 组件/交互方案）。

这种"同构骨架 + 差异化脏器"的设计让前后端流水线在概念上统一（都是一个总负责人按 7 阶段委派），但在执行层各自适配了不同的技术现实。

**跨前后端的 Agent（API 架构师、pm-bd/pm-fd）是一个仍未完全解决的物理边界问题。** 详见下文"待决设计"。

## 六、Skill 的分层体系：不只是"Agent 的小抄"

25 个后端 Skill 分成四大类：

| 类别 | Skill | 数量 |
|------|------|:--:|
| **SDD 流程类** | sdd-brainstorm / sdd-new-change / sdd-continue / sdd-ff / sdd-sync / sdd-review-spec / sdd-code / sdd-verify / sdd-fix / sdd-ship / db-spec | 11 |
| **测试类**（后端独有） | contract-test / integration-test / db-verify | 3 |
| **基础设施类** | knowledge / codegraph / rds-spec / context7 / deep-wiki / nuwa-api / git-commit / create-subissue / material-flywheel / sync-specs / init-ai | 11 |
| **共用** | cooper / melos-agent-lookup | 2 |

Skill 不是简单的"一段提示词模板"——每个 Skill 目录是一个完整的包：`SKILL.md`（核心指令加上 YAML frontmatter）允许额外辅助文件。Skill 的 frontmatter 至少含 `name` 和 `description`，因为 Melos 平台通过 frontmatter 来发现和注册 Skill。

**Skill 在 Agent 中的体现方式有两种**：

1. **关联 Skills 表**：Agent 的固定职能所需 Skill，列在指令头部的表格里
2. **铁律/条件触发**：像 `rds-spec`（"编写 DDL 时必查 DB 规范"）、`context7`（"遇到开源框架 API 不确定时必查"）、`deep-wiki`、`nuwa-api` 这些不是每次必走的，而是写在 Agent 的"铁律"段落里，满足特定条件才触发。在 `bd-skills-summary.md` 的映射表中用 **※** 标记

这套体系还有一个有意思的闭环——**"物料飞轮"（Material Flywheel）**：代码归档员在归档阶段把这次开发中的踩坑经验、中间件选型结论、性能调优结果沉淀到 knowledge base 和样板间（CRUD 服务模板 / DB Migration 模板），下一次新需求的技术探索阶段就能直接复用。Skill 本身随时间演进，不是静态的。

## 七、入口分流机制：一个需求来了怎么走到正确流水线

整套系统不是只有一个入口——用户提交 Issue 后的路由决策是这样的：

```
pm-bd (或 pm-fd，目前两个 PM 功能等价)
    │
    ├─ 用户明确说"纯前端" → fd-common-create-subissue → 前端流水线
    ├─ 用户明确说"纯后端" → bd-common-create-subissue → 后端流水线
    └─ 未明确（默认前后端都要改）
            ↓
        bd-common-API架构师
            │
            ├─ 读前后端代码 → 生成 OpenAPI 接口文档
            ├─ 协调评审：mention 前后端负责人 review
            ├─ 评审通过 → 创建前端子 Issue + 后端子 Issue
            │              ↓                      ↓
            │         前端流水线                 后端流水线
            └─ 评审未通过 → 修改接口文档 → 重新评审
```

API 架构师是这个分流体系里最关键也最脆弱的节点。它的职责天然跨前后端——要读前端代码（API 调用、TS 类型定义）也要读后端代码（路由、Controller、Model），要协调两边的评审，评审通过后再为两边各开一个子 Issue。但一个 Agent 进程物理上只能在一个容器里跑（后端容器或前端容器），这意味着它**必然有一侧的工具链和代码仓库访问不完全**。这是一个仍在讨论中的架构级问题（见 [`待讨论事项.md` 第 1 条](https://github.com/11477/melos-agent-workflow/blob/main/待讨论事项.md)）。

分流判定优先级很简洁：**用户明确说了按用户说的走，没说的默认就当"前后端都要改"。** 做多不如做少——不给 PM Agent 太多"猜测意图"的空间，减少误判。

## 八、几个值得写的设计细节

### 1. "断点保存"——容器重启后工作不丢

开发总负责人在**每个阶段完成后、发交接评论之前**先执行一次 `git add -A && git commit && git push`。这不是提交代码，是保存进度——"制品生成完成"、"质量验证完成"都可以是一笔 commit。Melos 的 Agent 跑在容器里，随时可能重启，没有这个断点容器重启后所有未提交的产物全丢。

### 2. Agent ID 不做硬编码——统一通过 Skill 动态查询

所有 Agent 在构造 `mention://agent/<id>` 链接时，必须先通过 `melos-agent-lookup` Skill 动态查询 Agent ID。不硬编码的原因是 Agent 可能在不同 workspace 中重新注册，ID 会变。这是一个"把静态配置变成动态查询"的工程化细节，在 Agent 数量的 scale 下尤其重要。

### 3. "用纯文字 mention，别用 mention 链接"——人机协作的正确姿势

关键阶段（③Spec 完成后、⑥测试完成后）总负责人要 mention issue 负责人（真人），等确认后再推进。但**交接通知中的 agent 名称必须用纯文字**，严禁出现 `mention://agent/...` 链接——因为这个链接会**立即自动触发**下一个 Agent，跳过人工确认这个步骤。

### 4. `.codegraph/` 不入仓库——"索引在每个 Agent 手里本地重建" vs "统一提交"

CodeGraph 索引（`.codegraph/` 目录）不提交到 git——由 git 管理员检出后加到 `.gitignore`。每个需要读代码的 Agent 在自己的阶段开始时独立执行 `codegraph init`（耗时短、不消耗 LLM token）。代码归档员不再重建索引。

这个设计的精妙之处在于：索引过期是常态（代码一直有人改），但每个 Agent 在自己阶段开始时重建，保证了拿到的是**自己当时的最新索引**，而不是依赖一个在归档后就不再更新的遥远快照。

### 5. 抽象概念的具体落地

| 抽象的设计概念 | 具体怎么落地的 |
|-------------|-------------|
| "Agent 不共享上下文" | 每个 Agent 独立启动 Claude Code 对话，不持久化历史，通过制品文件 + Issue 评论传递信息 |
| "多技术栈支持" | 不是每个 Agent 同时懂所有技术栈，而是**开发/质量/测试工程师在启动时检测项目标识文件**（pom.xml → Java、go.mod → Go、requirements.txt → Python），按技术栈分支选择对应的编译/测试/分析命令 |
| "知识随时间累积" | 代码归档员通过物料飞轮把经验沉淀成 knowledge base 条目，新需求的技术探索阶段直接命中 |
| "条件触发 ≠ 每次必查" | `rds-spec` 只在写 DDL 时触发，`context7` 只在遇到框架 API 不确定时触发，`nuwa-api` 只在 nuwa.json 存在时触发——避免把 Skill 执行做成"每次都来一遍"的 checklist hell |

## 九、待决设计与限制

### 1. 跨前后端 Agent 的运行时边界

`bd-common-API架构师` 要读两端代码，但进程物理上只能在一个容器（前端或后端）里跑。这意味着它永远有一侧的工具链不完整。目前候选方案有：拆成前后端两个 API 架构师靠 issue 评论交换契约、给 API 架构师单独部署一个装齐两侧工具链的容器、贴前端运行时等。详见 [`待讨论事项.md` 第 1 条](https://github.com/11477/melos-agent-workflow/blob/main/待讨论事项.md)。

### 2. 两个 PM 的分工

`pm-bd` 和 `pm-fd` 目前完全等价——都能处理纯前端 / 纯后端 / 前后端均需三种 issue。新需求来了用户该 @ 哪个？连前端 PM 都有已知 bug（处理纯后端需求时调的错误子 issue 创建工具）。详见 [`待讨论事项.md` 第 2 条](https://github.com/11477/melos-agent-workflow/blob/main/待讨论事项.md)。

### 3. 整套设计目前只是"纸上体系"

这些 Agent 指令和 Skill 设计已被导入 Melos 平台，但**尚未在大规模生产环境中跑过完整闭环**。Agent 间协作的可靠性、制品质量的稳定性、验证-修复闭环的收敛性，都需要真实 Issue 流量来检验。

## 总结：四个层次的设计思想

这套 Melos Agent 工作流设计可以提炼成四个层层递进的思想：

1. **角色拆分** — 不堆一个全能 Agent，按真实团队岗位拆成 11 个窄职责 Agent，每个只拿自己够用的上下文。角色拆分的粒度直接决定了后续所有设计的可行性。

2. **文件系统为真** — Agent 之间不共享内存、不共享聊天历史。信息通过制品文件（specs/tasks/db-spec/design）传递。上游产出 → 下游消费，像 Unix 管道一样串起来。

3. **先写规格，再写代码** — SDD 方法论保证"做什么"被充分定义后再启动"怎么做"。`db-spec` 这个后端独有制品说明：不是所有代码都适合 AI 生成，但**所有代码都应该有可验证的规格约束**。

4. **信息消费分层** — 不是所有 Agent 都能查所有知识源。CodeGraph 6 家查、Deep-Wiki 3 家查、Knowledge 4 家查。前置 Agent 消化原始信息并固化到制品，后置 Agent 只消耗精炼过的制品。信息消费的成本和可靠性由此得到控制。