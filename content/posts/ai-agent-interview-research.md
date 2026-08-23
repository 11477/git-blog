---
title: "AI Agent 研发工程师面试准备调研（2026 版）"
date: 2026-08-20
draft: false
---

> 面向中国大陆求职者；资料检索时间：2026-08-20。本文的“高频”是根据公开岗位 JD 中重复出现的能力项，以及主流 Agent 官方文档共同归纳，不代表任何公司的内部题库。优先引用公司招聘页、协议规范、官方技术文档和原始论文。

## 一页结论

AI Agent 研发工程师不是“Prompt 工程师”的新名字。当前岗位通常要求候选人把 **LLM、工具、知识、状态和运行时** 组合成一个可评测、可观测、可恢复、可控成本且安全的系统。

优先级建议：

- **P0（所有方向都要会）**：Python/后端与计算机基础；LLM/Transformer 基础；Prompt 与上下文工程；Tool Calling 与 Agent Loop；RAG；评测；系统设计；安全与权限。
- **P1（大多数岗位会追问）**：状态/记忆、工作流与多 Agent、MCP、异步任务与容错、可观测性、成本/延迟优化。
- **P2（按方向准备）**：SFT/RLHF/DPO/PPO/GRPO、分布式训练与推理、A2A、多模态、代码执行沙箱、推荐/搜索策略。

公开 JD 也支持这一判断。例如百度的智能体算法岗同时列出规划、工具调用、反思、RAG、多 Agent、评测、Prompt/Fine-tuning；Agent 引擎岗则重点列出状态持久化、限流重试、trace、日志、资源隔离、OAuth/JWT 和 prompt injection 防护；后端岗仍要求操作系统、网络、数据库、Go 与工程质量。参见[百度智能体算法工程师 JD](https://talent.baidu.com/jobs/detail/GRADUATE/4f1cbc80-8332-4a92-b8fa-c0132b17d47e)、[百度 Agent 引擎开发工程师 JD](https://talent.baidu.com/jobs/detail/SOCIAL/d56ce9b0-296b-4615-9497-115968d4fc14)、[百度 Agent 后端研发工程师 JD](https://talent.baidu.com/jobs/detail/SOCIAL/d1876313-6604-4e8d-bbcc-0dca5be4e51b)。

---

## 一、岗位能力画像与方向差异

| 方向 | 典型职责 | 面试权重最高 | 常见加分项 |
|---|---|---|---|
| Agent 应用/全栈研发 | 把模型接进业务，做工作流、工具、RAG、前后端交付 | Tool Calling、RAG、Prompt、状态、评测、业务抽象 | 上线作品、真实用户、数据闭环、前端交互 |
| Agent 平台/引擎/Infra | Agent runtime、网关、插件、任务调度、存储、监控、安全 | 后端与分布式系统、异步与幂等、持久化、沙箱、权限、可观测性 | Go/Rust/TS，Redis/PostgreSQL/Kafka，K8s |
| Agent 算法/策略 | 规划、路由、工具选择、反思、模型训练和策略优化 | Transformer、后训练、强化学习、数据构造、离线/在线评测 | 论文、开源、PyTorch/DeepSpeed、竞赛 |
| RAG/知识 Agent | 企业知识检索、Deep Research、引用与权限 | 检索链路、Embedding、切分、重排、混合检索、评测 | 搜索/推荐经验、知识图谱、多模态文档 |
| 多模态/具身/浏览器/代码 Agent | 在网页、电脑、代码库或物理环境中执行长任务 | 环境建模、动作空间、工具接口、长程规划、沙箱、安全 | VLM、RL、SWE-bench/BrowserGym 等实践 |

岗位边界并不严格。百度的办公 Agent 算法岗就同时要求模型训练微调和完整智能体系统搭建；其 Agent 引擎岗同时要求 TS/Node、Redis/PostgreSQL/Kafka、分布式架构与 Agent 攻击面防护，说明“算法 + 工程”或“Agent + 后端”的交叉是常态。[办公 AI Agent 算法岗](https://talent.baidu.com/jobs/detail/SOCIAL/c8821b51-c061-4652-8fad-61d8a49b9f52)、[Agent 引擎岗](https://talent.baidu.com/jobs/detail/SOCIAL/d56ce9b0-296b-4615-9497-115968d4fc14)。

---

## 二、必备八股分类：高频问题与追问

### 1. 编程、后端和计算机基础（P0）

至少准备到能手写、能讲复杂度、能结合 Agent 场景做设计。

**高频问题**

1. Python 的 GIL 是什么？线程、进程、协程分别适合什么任务？Agent 并发调用多个 I/O 工具为什么常用 `asyncio`？
2. `async/await` 的事件循环如何工作？如何设置 timeout、取消任务、限制并发量？
3. HTTP、SSE、WebSocket 的区别；Agent 流式输出、双向交互、长任务进度分别如何选？
4. TCP 重传与 HTTP 重试有什么区别？为什么业务层重试会造成重复副作用？
5. Redis 的缓存、分布式锁、Stream/队列用法；缓存穿透、击穿、雪崩怎么处理？
6. MySQL/PostgreSQL 的事务隔离、索引、MVCC；如何保存 run、step、message、tool call 和 checkpoint？
7. Kafka/RabbitMQ 的至少一次、至多一次、恰好一次分别意味着什么？
8. 常见限流算法：固定窗口、滑动窗口、漏桶、令牌桶。
9. 一致性、可用性、分区容错；什么时候接受最终一致？
10. 常见算法题：哈希、堆、LRU、二叉树、图遍历、Top-K、滑动窗口、并查集；规划岗还应会 A*、Beam Search、MCTS。

**典型追问**

- 工具调用超时后，重试前如何判断它是否已经成功？答案应谈到 **幂等键、请求状态查询、事务/Outbox、补偿动作**，而不是只说指数退避。
- 1000 个 Agent run 同时等待外部 API，如何做背压？应覆盖连接池、Semaphore、队列、租户限额、熔断、deadline 和取消传播。
- 用户刷新页面后如何继续看到运行状态？应覆盖持久化 run ID、事件流 offset、断线重连和 checkpoint。

现实依据：公开引擎 JD 明确要求 Redis、PostgreSQL、Kafka、事件驱动、负载均衡、熔断降级、限流、认证授权；后端 JD 要求操作系统、网络、数据库和 Go。[百度 Agent 引擎岗](https://talent.baidu.com/jobs/detail/SOCIAL/d56ce9b0-296b-4615-9497-115968d4fc14)、[百度 Agent 后端岗](https://talent.baidu.com/jobs/detail/SOCIAL/d1876313-6604-4e8d-bbcc-0dca5be4e51b)。

### 2. LLM 与 Transformer 基础（P0；算法岗需深入）

**高频问题**

1. Transformer 为什么用 Self-Attention？Q/K/V 分别是什么？缩放点积为什么除以 `sqrt(d_k)`？
2. Multi-Head Attention 的作用；MHA、MQA、GQA 的区别与推理性能影响。
3. 位置编码：绝对位置、RoPE；长上下文外推为什么困难？
4. 训练与推理的区别；自回归解码、KV Cache 的作用、显存如何随 batch/序列长度变化。
5. Tokenization（BPE/SentencePiece）如何影响中文、代码、成本与工具 schema？
6. temperature、top-p、top-k、greedy、beam search 的差异；为什么结构化工具调用通常降低随机性？
7. 幻觉来自哪里？“模型知道”与“系统有依据”为什么不同？
8. Pretrain、SFT、RLHF、DPO 的目标分别是什么？
9. LoRA/QLoRA 为什么节省显存？哪些参数在训练？
10. Prompt caching、continuous batching、quantization、speculative decoding 各解决什么问题？

**典型追问**

- Attention 复杂度为何通常是 `O(n²)`？长上下文还有哪些瓶颈（KV Cache、首 token 延迟、有效信息利用）？
- DPO 和 PPO 型 RLHF 有何差别？DPO 把偏好优化化为分类式目标，不显式训练奖励模型和在线采样；应能解释 reference policy、chosen/rejected 与 beta 的意义。原始论文见[DPO](https://arxiv.org/abs/2305.18290)。
- Fine-tuning、RAG、Prompt 三者如何选？行为/风格和稳定格式可考虑微调；频繁变化、需引用的知识优先 RAG；快速试验和规则约束先 Prompt。三者也可组合。

算法岗的现实门槛更高：公开 JD 直接列出 Transformer、PyTorch、SFT、PPO/DPO/GRPO、DeepSpeed/Hugging Face 等。[百度 Agent 策略算法实习岗](https://talent.baidu.com/jobs/detail/INTERN/6f85641f-2a8e-4806-bbb5-4bbbf4705741)、[大模型应用/Agent 算法岗](https://talent.baidu.com/jobs/detail/INTERN/1a0bfe96-f59c-4384-9525-79fdf324c67f)、[大模型算法实习岗](https://talent.baidu.com/jobs/detail/INTERN/f7141fef-2692-4eee-ac31-cc7b5f473ccf)。

### 3. Prompt 与上下文工程（P0）

**高频问题**

1. System/developer/user/tool 信息如何分层？指令冲突怎么处理？
2. Zero-shot、few-shot、CoT、ReAct 各适合什么？
3. 如何写稳定的系统提示：角色、目标、边界、流程、输出 schema、失败与升级策略。
4. 上下文窗口满了怎么办：截断、滑动窗口、摘要、检索、状态外置各有什么信息损失？
5. 为什么“把所有信息都塞进 prompt”会变差？如何做 context selection、compression、ordering？
6. Prompt 版本怎么管理与 A/B 测试？如何防止只对几个 demo 过拟合？
7. 结构化输出与纯 JSON prompt 的区别；schema 校验失败如何恢复？

**典型追问**

- 工具描述为何也是 Prompt Engineering？Anthropic 的生产经验是工具定义和规格需要与总提示同等重视，甚至在 SWE-bench Agent 上优化工具花的时间更多。[Anthropic《Building effective agents》](https://www.anthropic.com/engineering/building-effective-agents)
- “上下文工程”具体控制什么？可以按模型上下文、工具上下文、生命周期上下文拆解；LangChain 官方把缺少正确上下文视为 Agent 不可靠的主要来源之一。[LangChain Context Engineering](https://docs.langchain.com/oss/python/langchain/context-engineering)
- 如何让输出严格符合 schema？应谈 JSON Schema、原生 structured output、校验器、有限重试和失败降级。OpenAI 的 Structured Outputs 在函数定义中使用 `strict: true` 约束参数 schema。[OpenAI Structured Outputs](https://openai.com/index/introducing-structured-outputs-in-the-api/)

### 4. Tool Calling 与 Agent Loop（P0，核心中的核心）

先能白板写出基本循环：

`用户目标 → 组装上下文 → 模型选择动作/工具 → 参数校验 → 权限检查 → 执行 → 记录 observation → 继续或结束`

**高频问题**

1. 什么是 Agent？它和 chatbot、workflow 有何区别？Anthropic 的区分是：workflow 走预定义代码路径，Agent 由模型动态决定过程和工具；OpenAI 则强调 LLM 管理工作流执行并能动态选择工具。[Anthropic](https://www.anthropic.com/engineering/building-effective-agents)、[OpenAI 实践指南](https://openai.com/business/guides-and-resources/a-practical-guide-to-building-ai-agents/)
2. Function/Tool Calling 的完整协议是什么？模型生成调用意图，不等于模型自己执行函数；应用需要校验、执行并回传结果。
3. 工具 schema 如何设计：粒度、名称、描述、必填字段、枚举、错误模型、返回值大小。
4. ReAct 是什么？它把推理轨迹与动作交替生成，使 Agent 能通过外部环境取得新信息并修正计划。[ReAct 原始论文](https://arxiv.org/abs/2210.03629)
5. ReAct、Plan-and-Execute、Router、Evaluator-Optimizer、并行 Fan-out 分别适合什么任务？
6. 停止条件如何设计：最终答案、结构化结束动作、最大步数、预算、deadline、重复检测、人工接管。
7. 工具失败分类：参数错误、权限错误、瞬态错误、业务错误、结果不确定；每类怎么恢复？
8. 并行工具调用什么时候安全？有依赖或副作用时为什么不能盲目并行？
9. 动态工具选择/工具检索的价值是什么？工具过多会占用上下文并降低选择质量。
10. Agent 为什么会死循环？如何检测相同动作、无进展状态、振荡和 token 预算耗尽？

**典型追问**

- “工具返回 500 怎么办？”优秀答案先区分可重试性，再谈 exponential backoff + jitter、最大次数、幂等、熔断、fallback、人工接管和 trace。
- “为什么不一开始就多 Agent？”Anthropic 建议先选最简单可用方案，因为 Agent 往往以延迟和成本换取性能；OpenAI 也建议先最大化单 Agent，复杂度需要时再拆。[Anthropic](https://www.anthropic.com/engineering/building-effective-agents)、[OpenAI 实践指南](https://openai.com/business/guides-and-resources/a-practical-guide-to-building-ai-agents/)
- “如何避免模型伪造已执行动作？”执行结果必须来自受信任 runtime，UI 和最终答案应基于 tool result/事件状态，而不是模型自述。

### 5. RAG 与检索系统（P0）

**高频问题**

1. RAG 是什么？原始工作把参数化生成模型与可检索的非参数记忆结合。[RAG 原始论文](https://arxiv.org/abs/2005.11401)
2. 完整链路：采集/解析 → 清洗 → 切分 → embedding → 索引 → query 改写 → 召回 → 重排 → 上下文构造 → 生成与引用。
3. Chunk size/overlap 如何选？按字符切分有什么问题？表格、代码、PDF、多层标题如何处理？
4. 稀疏检索 BM25 与稠密向量检索的差别；为什么常做 hybrid search？
5. Embedding 的余弦、点积、L2；ANN、HNSW 的基本思想和召回/延迟权衡。
6. Reranker 与 retriever 的区别；Cross-Encoder 为什么通常更准但更慢？
7. Query rewriting、multi-query、HyDE、metadata filter 各解决什么问题？
8. 如何做引用与可验证回答？找不到证据时如何拒答或降级？
9. RAG 如何更新、删除和权限隔离？多租户索引怎么做？
10. 如何评测：Recall@K/MRR/nDCG、context precision/recall、faithfulness、answer correctness；离线集如何构造？

**典型追问**

- “召回结果相关但答案仍错”如何定位？分层看解析、召回、排序、上下文拼装、生成、引用绑定，分别建立指标，不要只调 prompt。
- “Agentic RAG 与普通 RAG 区别？”前者让模型决定是否检索、检索什么、是否改写/多跳/调用不同数据源；灵活性上升，同时延迟、成本和不可控性也上升。
- “文档权限怎么保证？”检索前按用户身份做 ACL filter，返回后仍做授权校验；不要指望 prompt 让模型保密。

公开 JD 把 RAG、知识库、Long-context RAG、Deep Research 与 Agent 融合作为常见能力项。[百度智能体算法岗](https://talent.baidu.com/jobs/detail/GRADUATE/4f1cbc80-8332-4a92-b8fa-c0132b17d47e)、[百度 Agent 策略算法岗](https://talent.baidu.com/jobs/detail/INTERN/6f85641f-2a8e-4806-bbb5-4bbbf4705741)。

### 6. 状态、记忆与长任务（P1）

**高频问题**

1. Context、state、memory、checkpoint 有何区别？
2. 短期记忆（单 thread）与长期记忆（跨 thread）如何划分？
3. 长期记忆可分哪些类型：语义事实、情景经历、程序性规则；何时写、由谁写、如何遗忘？
4. 为什么聊天记录不等于记忆？全量历史会带来成本、噪声和隐私问题。
5. Checkpoint 需要保存什么：消息、计划、工具调用状态、版本、预算、租户/用户、重放信息。
6. 如何恢复中断任务？如何防止恢复后重复执行副作用？
7. 多个并发 run 更新同一记忆如何处理冲突？
8. 记忆污染和跨用户泄漏如何防？

**典型追问**

- LangChain 把短期记忆定义为 thread 范围的状态，长期记忆则跨会话持久化；LangGraph checkpoint 支持暂停恢复、HITL、故障恢复和调试。[LangChain Memory Overview](https://docs.langchain.com/oss/python/concepts/memory)、[LangGraph Persistence](https://docs.langchain.com/oss/python/langgraph/persistence)
- “Agent 能不能自动写共享记忆？”需要默认用户隔离，共享政策只读，敏感写入加审批；官方文档明确提醒共享可写记忆可能被 prompt injection 污染。[LangChain Long-term Memory](https://docs.langchain.com/oss/python/langchain/long-term-memory)

### 7. 多 Agent、MCP 与 A2A（P1；平台岗偏 P0）

**高频问题**

1. 多 Agent 的收益和代价是什么？如何按领域、权限、上下文或执行环境拆分？
2. Supervisor/Manager、handoff、agents-as-tools、黑板/共享状态模式的区别。
3. 多 Agent 如何避免信息丢失、循环委派、重复工作与成本爆炸？
4. MCP 解决什么问题？Host、Client、Server 各自职责是什么？
5. MCP 的 tools、resources、prompts 有何区别？谁控制它们？
6. MCP capability negotiation、传输、JSON-RPC、授权与生命周期的基本概念。
7. MCP 与普通 REST API 的关系：MCP 是给模型/Agent 暴露上下文与能力的标准协议，不替代业务 API 本身。
8. MCP 与 A2A 的区别：前者主要让 Agent 连接工具/数据；A2A 面向独立、可能不透明的 Agent 之间发现能力、协作和长任务。
9. A2A 的 Agent Card、Task、Message、Artifact 等概念（协议更新快，面试前看最新 spec）。
10. 跨 Agent 身份、授权、审计、数据最小披露怎么做？

**可引用的标准答案依据**

- MCP 架构由 host 协调多个 client，每个 client 与特定 server 维持隔离连接；server 暴露 tools/resources/prompts，host 负责权限与安全边界。[MCP Architecture](https://modelcontextprotocol.io/specification/2025-06-18/architecture)、[MCP Server Primitives](https://modelcontextprotocol.io/specification/2025-06-18/server/index)
- MCP HTTP 授权涉及 OAuth 体系、PKCE、token audience validation；规范明确禁止把收到的 token 直接透传给下游。[MCP Authorization](https://modelcontextprotocol.io/specification/2025-06-18/basic/authorization)
- A2A 官方定位是异构、独立 Agent 的通信与互操作，可发现能力、协商交互模态、协作长任务，同时不暴露内部状态和工具。[A2A 官方项目](https://github.com/a2aproject/A2A)、[A2A 官方站](https://a2a-protocol.org/)

### 8. 评测、可观测性与数据飞轮（P0）

这是最容易把“做过 demo”和“做过生产”区分开的部分。

**高频问题**

1. Agent 为什么不能只评最终答案？还要评路径、工具选择、参数、证据、步数、安全和成本。
2. 指标如何分层：任务成功率、工具成功率、检索指标、事实性、格式、延迟、token/费用、用户反馈、安全违规率。
3. 离线 eval set 如何构造：真实流量抽样、失败案例、边界案例、对抗案例、合成扩充；如何避免数据泄漏？
4. Exact match、规则评分、LLM-as-a-judge、人工评分各有什么偏差？
5. Pointwise、pairwise、reference-based、rubric grading 怎么选？
6. Trace 里至少记录什么：run/step、prompt/model 版本、tool call、参数/结果摘要、耗时、token、错误、用户/租户、决策与审批。
7. 如何做回归测试、A/B、canary、shadow traffic？
8. 线上反馈如何进入数据飞轮？如何避免把噪声和攻击样本直接学进去？
9. 如何定位成功率下降是模型、prompt、retrieval、tool、数据还是基础设施造成？
10. 如何设 SLO：p50/p95/p99 延迟、成功率、超时率、人工接管率和单位成功任务成本。

**典型追问**

- “LLM judge 能代替人工吗？”不能完全代替；需校准、抽查、固定 rubric、测试位置/长度/自偏好等偏差，并保留关键任务的人工 gold set。
- “什么是 trace grading？”它对 Agent 的端到端决策和工具调用轨迹赋结构化分数，更容易解释为什么成功或失败，而非只看黑盒最终输出。[OpenAI Agents SDK tracing 入口](https://github.com/openai/openai-agents-python)、[OpenAI Agent 实践指南](https://openai.com/business/guides-and-resources/a-practical-guide-to-building-ai-agents/)
- “指标提高但用户体验下降怎么办？”先检查代理指标与业务目标是否错位，再做分群、人工复核、线上实验，避免 reward hacking。

公开 JD 已直接要求建设评测体系，优化成功率、稳定性、成本、延迟和用户体验，并用“评估—优化—验证”形成闭环。[百度智能体算法岗](https://talent.baidu.com/jobs/detail/GRADUATE/4f1cbc80-8332-4a92-b8fa-c0132b17d47e)、[大模型应用/Agent 算法岗](https://talent.baidu.com/jobs/detail/INTERN/1a0bfe96-f59c-4384-9525-79fdf324c67f)。

### 9. 生产化系统设计（P0）

**高频问题**

1. 如何设计一个支持多租户的 Agent 平台？
2. 同步请求与异步长任务如何统一建模？
3. Agent run 如何做到可取消、可恢复、可重放和可审计？
4. 模型路由怎么做：能力、价格、延迟、上下文、地域与可用性。
5. 如何做 timeout、retry、fallback、circuit breaker、bulkhead 和 backpressure？
6. 如何控制 token 成本：上下文裁剪、缓存、工具结果摘要、小模型路由、最大步数、预算控制。
7. 流式 UI 如何展示计划、进度、工具审批、部分结果和错误？
8. 配置、prompt、tool schema、模型版本怎么发布和回滚？
9. 多租户的数据、计算、凭证、日志如何隔离？
10. 如何做容量估算：QPS、并发 run、平均步数、模型 RPM/TPM、工具依赖、存储增长。

**系统设计题答题骨架**

1. 先问清业务目标、成功标准、用户规模、风险和延迟预算。
2. 定义 Agent 状态机与停止条件，而不是先报框架名。
3. 画出 API/Gateway、Orchestrator、Model Gateway、Tool Registry/Executor、RAG、State/Checkpoint、Queue/Worker、Observability、Policy/HITL。
4. 讲清主链路与一次失败链路。
5. 补充幂等、一致性、权限、隔离、降级、容量和成本。
6. 最后给 eval、灰度、数据飞轮和演进计划。

### 10. 安全、权限与 Human-in-the-loop（P0）

**高频问题**

1. Prompt injection 与 jailbreak 有何区别？直接注入和间接注入是什么？
2. 为什么“在 system prompt 里说不要泄密”不是安全边界？
3. 工具最小权限、read/write 分离、allowlist、参数校验、审批门如何设计？
4. 如何隔离代码执行：容器/VM、文件系统、网络 egress、CPU/内存/时间限制、凭证不入沙箱。
5. 如何防止 SSRF、SQL injection、命令注入、路径穿越、敏感信息进入日志？
6. OAuth/JWT、用户委托授权、service identity、短期 token 的区别。
7. 哪些动作必须 HITL：付款、删除、发信、生产变更、外部发布等不可逆或高风险动作。
8. HITL 怎么暂停和恢复？审批对象应该包含动作、参数、影响、证据与风险，不只是“同意/拒绝”。
9. 如何处理不可信网页、邮件、文档和 MCP tool output？应视为数据而非指令，并限制其影响范围。
10. 如何审计：谁在何时以谁的权限让哪个 Agent 调了什么工具、参数和结果是什么。

**典型追问**

- Anthropic 将网页中的间接 prompt injection 视为浏览器 Agent 的重大安全挑战，并明确说这仍未解决；因此要依靠模型、环境、外部内容、权限和人工确认的重叠防线。[Prompt Injection Defenses](https://www.anthropic.com/research/prompt-injection-defenses)、[Trustworthy Agents](https://www.anthropic.com/research/trustworthy-agents)
- LangChain 的 HITL 模式会在工具执行前暂停，持久化图状态，并支持 approve/edit/reject；这解释了为什么 HITL 与 checkpoint 是一体的。[LangChain Human-in-the-loop](https://docs.langchain.com/oss/python/langchain/human-in-the-loop)
- OpenAI 同样建议用多层 guardrails，并对失败阈值和高风险动作设置人工接管。[OpenAI 实践指南](https://openai.com/business/guides-and-resources/a-practical-guide-to-building-ai-agents/)

---

## 三、算法/训练方向加考清单（P1/P2）

若 JD 出现“策略算法、模型训练、后训练、对齐、Agent RL”，至少补齐：

1. Cross Entropy、KL divergence、最大似然、teacher forcing。
2. SFT 数据构造、packing、mask、数据去重与质量控制。
3. RLHF 流程：SFT → preference/reward model → policy optimization；PPO 的 policy/value、advantage、clip、KL 约束。
4. DPO 推导直觉、reference model、beta、chosen/rejected 数据偏差。
5. RLAIF、online/offline preference learning；PPO、DPO、GRPO 的训练成本和适用性。
6. Tool-use/trajectory 数据如何采集；step-level 与 outcome reward 的差异；credit assignment。
7. Agent 训练的环境、动作空间、奖励稀疏、探索与安全约束。
8. 数据飞轮：失败聚类 → 标注/合成 → 训练或 prompt/tool 修复 → 回归评测 → 灰度。
9. DeepSpeed ZeRO、数据/张量/流水线并行的基本概念；混合精度、梯度累积与 checkpoint。
10. 评测污染、benchmark overfitting、reward hacking。

算法面试切忌只会说名词。至少准备一个“某类失败是如何通过数据、模型或策略优化改善”的完整案例，并说明为什么没有只改 Prompt。

---

## 四、项目实战：简历上最好有怎样的 Agent 项目

### 推荐项目 A：企业知识与任务执行 Agent

能力覆盖最广，适合应用、平台、RAG 岗。

- 数据：PDF/网页/数据库，包含增量更新、ACL 和引用。
- 检索：hybrid + rerank + query rewrite，建立检索与答案两层 eval。
- Agent：ReAct 或 planner-executor，至少 3 个真实工具，严格 schema。
- Runtime：异步 run、checkpoint、取消、timeout/retry、幂等。
- 安全：read/write 工具分级，高风险动作 HITL；租户隔离。
- 可观测：完整 trace、p95 延迟、token/任务成本、失败分类看板。
- 交付：容器化、README、架构图、20—100 条固定评测集和一次优化报告。

### 推荐项目 B：Deep Research / 浏览器 Agent

重点展示任务拆解、搜索、来源质量、证据合并、去重、引用、预算和抗注入。不要把“能搜网页并总结”包装成完成品；要有检索质量、来源可信度、结论可追溯和中断恢复。

### 推荐项目 C：代码 Agent / 数据分析 Agent

重点展示沙箱、文件 diff、命令 allowlist、测试闭环、长任务状态、产物验证。数据 Agent 要把只读查询与写操作分开，对 SQL 做解析/限制/审批。

### 面试讲项目的结构

用 5—8 分钟回答：

1. **问题**：用户是谁，原流程为什么不够好，成功标准是什么。
2. **约束**：数据、权限、延迟、成本、准确性、并发。
3. **架构选择**：为什么用 workflow/单 Agent/多 Agent；为何选该检索和状态方案。
4. **最难故障**：展示一条真实 trace，根因是什么。
5. **量化结果**：成功率、Recall@K、p95、token 成本、人工接管率的前后对比。
6. **安全与边界**：什么不能自动做，如何授权和审计。
7. **复盘**：如果重做会删什么、加什么。

面试官常追问：

- “不用 Agent、只用确定性 workflow 能不能做？”
- “为什么用这个框架，自己写 loop 有多难？”
- “你的评测集从哪来？有没有过拟合？”
- “最差 case 是什么？线上失败后用户看到什么？”
- “真正由你完成的模块是什么？”
- “去掉 LLM 后系统还有哪些确定性保证？”

---

## 五、两道系统设计题的准备重点

### 题 1：设计一个企业客服 Agent

应覆盖：意图/路由、知识检索、订单只读查询、退款写操作、HITL、会话状态、权限、引用、升级人工、敏感信息处理、eval 和 SLO。

关键 trade-off：常见稳定流程用确定性 workflow；边界情况和非结构化问题再交给 Agent。退款等高风险动作需策略引擎与审批，不让模型直接拥有最终权限。

### 题 2：设计一个支持百万用户的通用 Agent 平台

应覆盖：

- API Gateway + auth + tenant quota；
- Orchestrator/状态机与 Queue/Worker；
- Model Gateway（路由、限流、fallback、缓存、成本）；
- Tool Registry/Executor（schema、版本、权限、沙箱、幂等）；
- State/Checkpoint/Artifact store；
- RAG service 与租户 ACL；
- Event/streaming；
- trace、metrics、eval、审计；
- Policy/Guardrail/HITL。

追问重点通常不是“用了几个微服务”，而是长任务如何恢复、副作用如何幂等、多租户如何隔离、模型和外部工具限流如何传播、成本如何按任务核算。

---

## 六、面试前的准备顺序

### 第一阶段：打牢 P0

1. 能不依赖框架写一个 100—200 行的 tool-calling loop。
2. 完整复习 Transformer、推理、采样、KV Cache、Prompt/RAG。
3. 复习 Python 异步、网络、数据库、缓存、队列、幂等、限流。
4. 给现有项目补 eval、trace、重试/超时、权限和失败处理。

### 第二阶段：工程化

1. 加 checkpoint、取消/恢复、流式事件。
2. 做一套不少于 20 条的回归集，覆盖正常、边界、工具故障和注入攻击。
3. 量化成功率、p95、平均步骤、token/任务成本。
4. 写一页系统设计与一页事故复盘。

### 第三阶段：按 JD 定向

- 应用岗：多练业务建模、Prompt/RAG、前后端交付。
- 平台岗：深入异步 runtime、分布式系统、可观测、安全、MCP。
- 算法岗：深入 Transformer、SFT/DPO/PPO/GRPO、数据和实验设计。
- RAG 岗：深入 IR、ANN、rerank、数据治理和检索评测。
- 代码/浏览器 Agent：深入沙箱、权限、prompt injection 和长任务。

---

## 七、面试前必读的一手资料

### Agent 设计与框架

- [Anthropic：Building effective agents](https://www.anthropic.com/engineering/building-effective-agents)——workflow 与 Agent 的边界、路由/并行/编排/评估器模式、工具设计。
- [OpenAI：A practical guide to building agents](https://openai.com/business/guides-and-resources/a-practical-guide-to-building-ai-agents/)——模型、工具、指令、单/多 Agent 与 guardrails。
- [OpenAI Agents SDK 官方文档](https://openai.github.io/openai-agents-python/agents/)——agent、runner、handoff、guardrail、session、structured output。
- [LangChain Agents](https://docs.langchain.com/oss/python/langchain/agents)——工具循环、状态、流式与 middleware。
- [LangGraph Persistence](https://docs.langchain.com/oss/python/langgraph/persistence)——checkpoint、HITL、故障恢复、time travel。

### 协议与安全

- [MCP 规范：Architecture](https://modelcontextprotocol.io/specification/2025-06-18/architecture)
- [MCP 规范：Authorization](https://modelcontextprotocol.io/specification/2025-06-18/basic/authorization)
- [A2A 官方协议项目](https://github.com/a2aproject/A2A)
- [Anthropic：Prompt injection defenses](https://www.anthropic.com/research/prompt-injection-defenses)
- [LangChain：Human-in-the-loop](https://docs.langchain.com/oss/python/langchain/human-in-the-loop)

### 原始论文

- [Attention Is All You Need](https://arxiv.org/abs/1706.03762)
- [Retrieval-Augmented Generation](https://arxiv.org/abs/2005.11401)
- [Chain-of-Thought Prompting](https://arxiv.org/abs/2201.11903)
- [ReAct](https://arxiv.org/abs/2210.03629)
- [InstructGPT / RLHF](https://arxiv.org/abs/2203.02155)
- [DPO](https://arxiv.org/abs/2305.18290)

### 岗位需求样本

- [百度 2027 AIDU 智能体算法工程师](https://talent.baidu.com/jobs/detail/GRADUATE/4f1cbc80-8332-4a92-b8fa-c0132b17d47e)
- [百度 AI Agent 算法工程师（社招）](https://talent.baidu.com/jobs/detail/SOCIAL/c8821b51-c061-4652-8fad-61d8a49b9f52)
- [百度 Agent 引擎开发工程师](https://talent.baidu.com/jobs/detail/SOCIAL/d56ce9b0-296b-4615-9497-115968d4fc14)
- [百度 Agent 后端研发工程师](https://talent.baidu.com/jobs/detail/SOCIAL/d1876313-6604-4e8d-bbcc-0dca5be4e51b)
- [百度 Agent 策略算法实习生](https://talent.baidu.com/jobs/detail/INTERN/6f85641f-2a8e-4806-bbb5-4bbbf4705741)
- [百度大模型应用/Agent 算法工程师](https://talent.baidu.com/jobs/detail/INTERN/1a0bfe96-f59c-4384-9525-79fdf324c67f)
- [百度大模型应用研发工程师](https://talent.baidu.com/jobs/detail/SOCIAL/a5ff8d15-b547-4a87-ba55-a128dae953cd)

---

## 最后提醒

真正有区分度的不是背出多少框架 API，而是能否对每个主题回答四层：**原理是什么、为什么这样设计、失败时怎样定位、生产上如何权衡**。如果时间有限，优先把一个项目做到“有固定评测集 + 有 trace + 有真实失败案例 + 有量化优化 + 有权限与恢复机制”；这比再做三个只会聊天的 Agent demo 更有说服力。
