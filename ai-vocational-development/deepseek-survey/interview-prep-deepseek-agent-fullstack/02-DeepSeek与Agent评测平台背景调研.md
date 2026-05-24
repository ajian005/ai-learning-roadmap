# 02 · DeepSeek 与 Agent 评测平台背景调研（全栈视角）

> DeepSeek 公司背景 / Harness 团队时间线 / 行业概览已在 PM 包写过完整版，本文档**只做引用 + 全栈岗位特化补充**——业界 eval platform 调研、轨迹回放工具、LangChain/OAI SDK 现状、RL Agent 训练 infra。

---

## 一、必读的 PM 版背景章节

> **[../interview-prep-deepseek-harness-pm/02-DeepSeek与Harness背景调研.md](../interview-prep-deepseek-harness-pm/02-DeepSeek与Harness背景调研.md)**

它已经覆盖了：DeepSeek 公司 60 秒速写、Harness 团队组建时间线（2026-05-20 公开）、对标 Agent 产品分析、Anthropic Harness 三篇博客的关键观点。

工程师全栈面试同样需要这些背景。先读完再回来。

---

## 二、全栈岗视角的 DeepSeek 背景补充

### 2.1 DeepSeek 内部 Agent 训练 infra 的"形状"（推测）

公开信息里 DeepSeek 没有详细 disclose 其 RL infra，但结合：
- DeepSeek-R1 paper 提到 GRPO（Group Relative Policy Optimization）
- DeepSeek-Coder 系列在 SWE-Bench 上的训练涉及 agent trajectory 数据
- 团队公开提"Harness 与模型共同进化"

**可推测的 RL Agent 训练 infra 拓扑**：

```
┌──────────────────────────────────────────────────────────┐
│  Agent 任务池 (内部 + 公开 benchmark)                     │
│   - SWE-Bench / AiderBench / 自有 100 task               │
│   - Adversarial probe                                    │
└───────────────────┬──────────────────────────────────────┘
                    │
        ┌───────────▼───────────┐
        │   Agent Scaffold 适配层 │  ← 本岗负责
        │  - 内部统一接口         │
        │  - LangChain adapter   │
        │  - OAI SDK adapter     │
        │  - 自研 harness adapter│
        └───────────┬───────────┘
                    │
        ┌───────────▼───────────┐
        │   Agent 容器集群        │  ← 本岗负责
        │  - K8s pod per task    │
        │  - sandbox + tool exec │
        │  - resource limit      │
        └───────────┬───────────┘
                    │
        ┌───────────▼───────────┐
        │   Trajectory 收集 + 存储 │
        │  - JSON / Protobuf     │
        │  - DuckDB / S3 / blob  │
        └───────────┬───────────┘
                    │
        ┌───────────▼────────────────┐
        │   Eval Platform             │  ← 本岗负责
        │  - 前端 trajectory viewer    │
        │  - LLM-as-judge scoring     │
        │  - leaderboard / dashboard  │
        └───────────┬────────────────┘
                    │
                    ▼
            ┌──────────────┐
            │ 算法 / 研究员 │
            └──────┬───────┘
                   │
                   ▼
          ┌────────────────┐
          │  RL Trainer    │  ← 训练侧 infra（不是本岗主要负责）
          │  GRPO/DPO/PPO  │
          └────────────────┘
```

> **本岗负责图中三个绿色框**：Agent Scaffold 适配层 / Agent 容器集群 / Eval Platform。**研究员 / 算法在你的 platform 里看 trajectory、debug、提反馈，然后由训练侧 infra 把数据喂回模型。**

### 2.2 业界对标产品

DeepSeek 内部肯定借鉴了下面这些产品，**面试时随时能引用**：

| 产品 | 定位 | 关键特性 | DeepSeek 借鉴点 |
| --- | --- | --- | --- |
| **LangSmith** | LangChain 官方 | trace + eval + monitoring + 反馈循环 | 评测平台 + trajectory viewer |
| **LangFuse** | 开源 LLM observability | trace + score + dataset + prompt mgmt | 开源 + self-host 友好 |
| **Weights & Biases (W&B)** | ML 实验跟踪 | run / sweep / table / report | 实验对比 + 可视化 |
| **MLflow** | ML 生命周期 | tracking / model registry / serve | 多模型管理 |
| **Phoenix (Arize)** | LLM eval 开源 | trace + eval + dataset 三件套 | OpenInference 标准 |
| **Helicone** | LLM proxy + obs | proxy + cache + logs + cost | 代理层观测 |
| **OpenTelemetry GenAI semantic conventions** | 行业标准 | LLM/agent span 规范 | 接 OTel 生态 |
| **Ragas / DeepEval** | RAG/LLM eval lib | LLM-as-judge metrics | scoring pipeline |

**关键观察**：业界没有一个完美的"agent eval platform"——LangSmith 偏 trace、W&B 偏 ML 训练、Phoenix 偏 RAG。**DeepSeek 内部很可能要自研一个兼具 trajectory 可视化 + 训练对齐的 eval platform**——这正是本岗的核心交付。

### 2.3 业界 Agent Scaffold（外部 framework）现状

| Framework | 主语言 | Agent 抽象 | Tool 协议 | 适合 | 集成难度 |
| --- | --- | --- | --- | --- | --- |
| **LangChain** | Python | AgentExecutor + LCEL | OpenAI function calling | 通用 agent | 中（API 变化多） |
| **LangGraph** | Python | StateGraph + Node | 自定义 | 复杂工作流 | 中 |
| **OpenAI Agents SDK** | Python / TS | Agent + Handoff + Guardrail | Function tool | 多 agent 编排 | 低（设计简洁） |
| **Pydantic AI** | Python | Agent + Tool | type-safe | 强类型场景 | 低 |
| **LlamaIndex Agents** | Python | ReActAgent / Worker | tool spec | RAG + Agent | 中 |
| **CrewAI** | Python | Crew + Agent + Task | role-based | 多 agent 协作 | 中 |
| **AutoGen (Microsoft)** | Python | ConversableAgent | function map | 多 agent 对话 | 中 |
| **DeepSeek 自研 Harness** | TBD | 自定义 | 自定义 | 自家产品 | N/A |

**集成层关键问题**：
- **统一 trajectory schema**：每家输出格式都不同，必须做归一
- **Tool 协议适配**：OpenAI / Anthropic / Gemini / MCP 4 套要 normalize
- **Streaming 协议**：每家 chunk 格式不同
- **State checkpoint**：LangGraph 有 checkpoint，LangChain 不直接有
- **Async vs Sync**：要兼容两种

### 2.4 DeepSeek 用本岗位"提升模型迭代效率"的具体含义

> 这是 JD 第 3 条职责的关键词。具体含义是：
>
> | 当前痛点 | 本岗解决 |
> | --- | --- |
> | 算法上传新 ckpt 后要手写 eval 脚本，1~2 天才出结果 | 1 小时自动 trigger + 看 dashboard |
> | 看 100 个 fail trajectory 要 grep JSON 文件 | UI 一键过滤 + diff |
> | 不知道为什么模型 reward 低 | trajectory viewer + 步骤回放 + LLM 解释 |
> | 多 ckpt 横向对比靠手工 Excel | leaderboard + 横向 chart |
> | 算法和工程沟通靠口头 / Slack | 评测平台直接评论 / 标记 + 自动 tag failure mode |

**面试金句**："Eval platform 的 ROI = (节省的算法时间 × 算法人数) ÷ 平台开发成本——这是我做工具时的指北星。"

---

## 三、必读的工程参考资料

### 3.1 业界开源代码（必看其中 3 个）

1. **[LangFuse 开源 repo](https://github.com/langfuse/langfuse)** —— self-host LLM observability，前端 Next.js + 后端 Node + Postgres
2. **[Phoenix (Arize) 开源 repo](https://github.com/Arize-ai/phoenix)** —— LLM trace 可视化，OpenInference 标准
3. **[OpenLLMetry](https://github.com/traceloop/openllmetry)** —— LLM 的 OTel SDK，看 instrumentation 思路
4. **[LangChain Agent Executor](https://github.com/langchain-ai/langchain/blob/master/libs/langchain/langchain/agents/agent.py)** —— 看 agent loop 源码
5. **[OpenAI Agents SDK](https://github.com/openai/openai-agents-python)** —— 看 Agent + Handoff 设计
6. **[OpenTelemetry GenAI Semantic Conventions](https://github.com/open-telemetry/semantic-conventions/tree/main/docs/gen-ai)** —— LLM span 标准

### 3.2 必读博客 / paper

1. [LangSmith 设计 blog](https://blog.langchain.dev/announcing-langsmith/) —— LangChain 官方 trace 工具设计哲学
2. [Anthropic Effective context engineering](https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents) —— 评测需要懂 context
3. [Eugene Yan: LLM patterns](https://eugeneyan.com/writing/llm-patterns/) —— Evals / RAG / Guardrails 系统综述
4. [Hamel Husain - LLM Evals](https://hamel.dev/blog/posts/evals/) —— 实战导向的 eval 方法论
5. [BrainTrust LLM Eval](https://www.braintrust.dev/blog) —— 商用 eval platform 设计思路

### 3.3 容器 / K8s 关键知识

- **multi-stage Dockerfile**：减小镜像 size
- **distroless images**：production base image 标准
- **K8s probe**：liveness vs readiness vs startup
- **resource request/limit**：OOM 防范
- **Helm chart structure**：template + values + dependencies
- **Init container + sidecar**：常用模式
- **HPA / VPA**：自动扩缩

### 3.4 RL Agent 训练相关

- **GRPO (Group Relative Policy Optimization)** —— DeepSeek-R1 用的算法
- **PPO / DPO** —— 主流 LLM RL 方法
- **Trajectory replay 在 RL 里的作用** —— off-policy data reuse
- **Reward shaping** —— 怎么用 trajectory 标 reward

---

## 四、面试可能被深问的"工程挑战"清单

> 主动 surface 出来的工程问题——证明你想过怎么落地。

| 工程挑战 | 难点 | 我的初步思路 |
| --- | --- | --- |
| **统一 trajectory schema 跨 5 个 framework** | 每家 message 格式 / tool call / streaming 都不同 | Protobuf schema + adapter 模式 + 版本号 |
| **K8s pod per agent task 的资源管理** | 100 个 agent 并发跑，CPU/Memory/Disk 抖动 | Job + resource quota + priority class + spot 节点 |
| **trajectory 存储**：JSON 太大 / SQL 不灵活 | 1 个 trace 几 MB，每天 100K，查询要快 | Parquet on S3 + DuckDB query + 按 trajectory_id sharding |
| **前端渲染 10K 步 trajectory** | DOM 爆 / 卡顿 / scroll 慢 | Virtual list + react-window + 按需 lazy load |
| **diff 2 个 trajectory** | 一步对一步的对比难做 | 树结构 diff + 颜色高亮 + step alignment 算法 |
| **LLM-as-judge 的 latency** | judge LLM 慢 / 贵 / 不稳定 | batch + cache + 异步队列 + small judge model fallback |
| **多模型横评 dashboard** | 大表格性能 + filter 复杂 | server-side filter + 增量 fetch + 列虚拟化 |
| **容器化 LangChain agent 跑 sandbox 内 bash** | sandbox 逃逸 / 资源滥用 | privileged=false + readOnlyRootFilesystem + seccomp + 网络 policy |
| **CI/CD 同时支持前端 + Python + Rust + Docker build** | 不同 toolchain 在 CI 上分散 | monorepo + nx/turborepo + cached layer + matrix builds |
| **依赖版本同步**：LangChain 一周 3 个新版 | 升级容易破内部接口 | snapshot test + version pin + canary 通道 |

**面试中怎么用**：被问"你怎么看 Agent 全栈最难的工程挑战"时，从上表挑 3 个，给出**初步思路 + trade-off + 你想跟团队对齐的边界**。

---

## 五、可能的"看源码"考题预演

DeepSeek 文化 research-driven，面试官可能给你 LangChain / OAI SDK 一段代码让你点评。**3 分钟内说出：**

1. **这段代码做什么**
2. **为什么这么设计**（推测意图）
3. **它的 trade-off / 隐患**
4. **如果让我改我会怎么改**

### 推荐预演的 5 个源码片段

| 来源 | 文件 | 看什么 |
| --- | --- | --- |
| LangChain | `agents/agent.py` AgentExecutor._call | agent loop 实现 + max_iter |
| OpenAI Agents SDK | `agents/_run_impl.py` | run loop + Handoff 实现 |
| LangGraph | `graph/state.py` | state checkpoint 设计 |
| MCP Python SDK | `stdio.py` | stdio transport + JSON-RPC 处理 |
| LangFuse | `web/src/server/api/routers/traces.ts` | trace 查询 API 设计 |

---

## 六、一句应试钥匙（全栈特化）

> **当面试官提到 evaluation / trajectory / framework / 容器，"我知道有这个东西" 是初级答法。全栈工程师的答法是 "我用过 [产品 X] 的 [feature]，它做得好的地方是 [Y]，做得不好的地方是 [Z]，我自己复刻过的版本是 [GitHub URL]——可以一起 review。"**
