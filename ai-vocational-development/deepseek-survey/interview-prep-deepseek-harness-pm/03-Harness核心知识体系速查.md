# 03 · Harness 核心知识体系速查（30 秒电梯版 + 1 个例子）

> JD 任职要求第 6 条原文："理解 LLM 以及 Agent 基本机制及其技术原理，包括 **LLM API、KV Cache、Agent Loop、Tool Use、Reasoning、Planning、Skills、MCP、Memory、Subagent、Multi-Agent** 等相关知识。对 **Prompt Engineering、Context Engineering、Harness Engineering** 等课题有第一手实践。"
>
> 这一节把每个词压缩成 **30 秒能讲清的电梯版 + 1 个可证明你真用过的例子**。面试现场不要照背，但每个词你要能"无缝挂钩"。

---

## 一、技术词速查矩阵

| 关键词 | 30 秒电梯版 | 一个产品/工程例子 | 易混点 |
| --- | --- | --- | --- |
| **LLM API** | LLM 暴露的是 stateless 接口：messages in / completion out + tool_use；服务端不持有会话。所有"记忆"都靠每次 request 把 history 重传过去。 | OpenAI Chat Completion 和 Anthropic Messages API 差异：Anthropic 支持 `cache_control` 显式声明哪段 prompt 走 prompt cache | API 不等于"模型"，API 是模型 + serving + 计费的接口层 |
| **KV Cache** | Transformer 推理时每个 token 会算出 K/V；如果前 N 个 token 不变，K/V 就可复用——这是 prompt cache 的物理基础。**Agent 场景下，固定的 system prompt + 已经发生的对话历史就是 cache hit 的金矿。** | Claude prompt caching 收费降 90%、Anthropic 显式 cache breakpoint；Claude Code 把 system prompt 切成 cacheable 段 + 动态段 | KV Cache 是模型推理细节，"prompt cache" 是计费侧暴露；二者关联但概念不同 |
| **Agent Loop** | **感知 → 思考 → 行动 → 观察** 循环。最小实现：LLM 输出 tool_use → 执行 → 把 tool_result 写回 messages → 再 call LLM。直到模型返回 `stop_reason=end_turn`（或满足终止条件）。 | minimal_agent.py 50 行就可以跑通；Claude Code 里 QueryEngine 是这个 loop 的工业实现，自带 diminishing-returns detector（连续 3 次新增 <500 token 就停） | 不是 "ReAct = Agent"。ReAct 是一种 prompting pattern；Agent Loop 是工程实现 |
| **Tool Use / Function Calling** | 把"外部能力"打包成 JSON Schema，注入模型 context。模型决定何时 call、call 啥参数；harness 负责 dispatch + 把 result 塞回去。 | Anthropic tool_use block、OpenAI function_calling、Gemini function_declarations；Claude Code 40 个 tool 全部走同一个 ToolDef 接口 | tool 不只是 API 调用；read_file / write_file / bash 这种"环境感知 + 改变"才是 Agent 的关键 tool |
| **Reasoning** | 模型在输出最终答案前的 chain-of-thought / hidden scratchpad。o1/R1/Sonnet 4.5 把它做成显式 thinking block，开销大但能提升复杂任务表现。 | DeepSeek-R1 走 RL 训练显式 reasoning；Anthropic 的 `thinking` block 可控开关；面对 Harness：thinking token 也走 KV cache 但不计入历史 | reasoning ≠ planning。reasoning 是单次推理深度，planning 是多步任务分解 |
| **Planning** | 把"用户的高层需求"拆成"可执行子任务"。可由模型隐式做（CoT），也可显式做（生成 task list 文件）。**长任务必走显式 planning**（防止上下文中后期失焦）。 | Anthropic 11 月 long-running harness：Initializer Agent 写 task list；2026-03 的 Planner / Generator / Evaluator 三 Agent | planning 不等于 chain-of-thought；planning 输出是结构化 artifact（list / DAG），可被验证 |
| **Skills** | "把一段最佳实践 prompt + 一些示范 + 几个推荐 tool"打包成可复用单元。让 Agent 在合适场景被"自动启用"。 | Anthropic Skills（平台注册，强审计）vs OpenCode Skills（FS 挂载，强可移植）；Claude Code 的 frontend-design skill | Skills ≠ subagent。Skills 是 prompt+context bundle；subagent 是独立 context 的运行实例 |
| **MCP (Model Context Protocol)** | 模型 ↔ 工具的**标准协议**，解决 N×M 集成爆炸。任何工具供应方实现 MCP server，任何 Agent 客户端通过 MCP client 接入。 | Claude Code / Cursor / Codex 都支持；Playwright MCP 让 Agent 操控浏览器；Anthropic 在 evaluator agent 里用 Playwright MCP 实测 UI | MCP 不是 API gateway，是 stdio/SSE 协议；MCP server 通常在用户本机起进程 |
| **Memory** | 跨 session / 跨对话的持久化信息。常见三层：短期（context window 内的 messages）/ 中期（session 级 KV / file artifact）/ 长期（向量库 / `MEMORY.md` 注入） | Claude Code 的 CLAUDE.md / `MEMORY.md`（capped 200 行 / 25KB）；Cursor 的 .cursorrules；ChatGPT 的 memory feature | memory 不是 RAG，RAG 是检索增强；memory 是"写"端，RAG 是"读"端，但二者底层可共用向量库 |
| **Subagent** | 一个独立 context window 的子 Agent。父 Agent spawn 一个 worker，worker 在隔离上下文里跑完，只把 summary 汇报回父。**核心价值是上下文隔离 + 专注度。** | Claude Code 的 Task tool；OpenClaw ACP；Anthropic 3 月 generator/evaluator 互通过文件而非共享 context | subagent 不省 token（甚至更费），但能防"长 context 失焦" |
| **Multi-Agent** | 多个 Agent 协作（Planner/Worker/Evaluator/Critic 等），通信靠 messages / files / shared memory。模式：Controller-Worker、Planner-Executor、GAN-style（gen + eval） | Anthropic 三 Agent；OpenClaw 多 Agent；Cowork 国内代表 | 不是 "Agent 数量越多越好"。每加一个 Agent，错误传播 + 协调成本爆涨 |
| **Prompt Engineering** | 通过"措辞 / 格式 / few-shot"控制模型行为。**对 Agent 来说，system prompt 不仅告诉模型怎么说话，还告诉它怎么用 tool、什么时候停。** | Claude Code system prompt 走 section-based composition（54KB），支持 A/B 和 feature gate | 进入 2026 后，单纯 prompt engineering 已是基础功；当下竞争壁垒已转向 context engineering |
| **Context Engineering** | 决定模型"看到什么"——筛选、压缩、组装、refresh。**Anthropic 自评：context 是对输出质量影响最大的杠杆，超过模型选择。** | Claude Code 三层 context pipeline（user / system / system-prompt-parts），4 阶段 compression（snip / micro / collapse / auto-compact） | context engineering ≠ RAG；RAG 只是 retrieve 端，context engineering 包含 compress / cache / handoff / refresh 整链 |
| **Harness Engineering** | 围绕模型的所有工程：Agent Loop / Tool / Permission / Multi-Agent / Recovery / Verification / Lifecycle Hooks。**Model + Harness = Agent。** | 同级目录《主流 AI Agent Harness Engineering 深度分析》七大组件 | 不是"前端工程"或"backend 工程"；是一门独立工程学科，知识跨研究员 / 系统工程师 / 产品 |

---

## 二、Harness 七大核心组件速记（背诵卡）

> 这是从同级目录 [`ai-agent-harness-engineering-analysis.md`](../ai-agent-harness-engineering-analysis.md) 提炼。面试时被问"Harness 包含哪些组件"，按这 7 个讲一定够。

```
1. Agent 循环（核心引擎）
   感知 → 思考 → 行动 → 观察；ReAct / 流式 / 中断与恢复

2. 工具系统（能力层）
   工具注册 / Schema / 执行引擎 / MCP；并行 / 超时 / 结果处理

3. 权限与安全（防护层）
   分级权限（safe/medium/high）+ Policy Engine + Sandbox + 审批工作流

4. 多 Agent 协调
   Controller-Worker / Planner-Executor / GAN-style；上下文隔离 + 深度限制 + 预算

5. 错误恢复
   StuckDetector（Repeater/Wanderer/Looper）+ 分类恢复策略 + Circuit Breaker

6. 验证闭环
   Build → Verify (type/lint/build/test) → Self-Fix；最多 N 次

7. 生命周期钩子
   SessionStart/End、UserSubmit、PreToolUse/PostToolUse、PreCompact、Error
```

**口诀**：**「环、具、权、协、恢、验、钩」**——记不住的话画一张图，循环最中间，外面 6 个组件围一圈。

---

## 三、容易被深挖的"概念辨析"题（每个都有"陷阱版回答" vs "高分回答"）

### 辨析 1：Agent vs Workflow vs Chatbot
- **陷阱**："Agent 就是会用工具的 Chatbot。"
- **高分**："Workflow 是**编排时确定**的（人决定步骤），Chatbot 是**单轮回复**。Agent 的核心是**运行时由模型自主决策**——下一步做什么、是否要 tool、是否要 subagent、何时停。Anthropic 的定义：'an LLM in a loop with tools'。"

### 辨析 2：Prompt Engineering vs Context Engineering vs Harness Engineering
- **高分**："三者关注的对象不同——Prompt 告诉模型'怎么做'（说什么话），Context 控制模型'看到什么'（输入），Harness 围绕模型'怎么跑'（基础设施）。我个人观察 2024 关键词是 prompt、2025 是 context、2026 是 harness——产业焦点在外移。"

### 辨析 3：Memory vs Context vs RAG
- **高分**："Context 是模型当下能看到的 tokens（窗口内）；Memory 是跨 session 的持久信息（要不要塞进 context 是另一个决策）；RAG 是把外部知识检索回来再塞 context 的方法。三者底层经常共用向量库，但语义层级不同。"

### 辨析 4：KV Cache vs Prompt Cache
- **高分**："KV Cache 是 Transformer 推理过程中的内部缓存（K/V matrices），任何重复前缀都自动复用——这是模型本身的事。Prompt Cache 是 API 层把这个能力**显式商品化**（如 Anthropic 的 cache_control），允许你声明哪段必须缓存、能享受多少折扣。Harness 设计要刻意把高频不变的部分（system prompt、稳定的 tool schema）放前面，把变动的（用户输入、上一步观察）放后面，最大化 cache hit。"

### 辨析 5：Subagent vs Multi-Agent
- **高分**："Subagent 是**父 Agent 主动 spawn** 出来的隔离上下文的执行单元，目的是上下文隔离和专注度（典型：Claude Code 的 Task tool）。Multi-Agent 通常指**预先设计好的多 Agent 协作架构**（Planner/Generator/Evaluator）。Subagent 是机制，Multi-Agent 是模式。"

### 辨析 6：Plan-Act-Validate vs ReAct
- **高分**："ReAct 是单循环里交替 thought/action/observation 的 prompting 模式；Plan-Act-Validate 是把这三步**显式拆开为不同阶段甚至不同 Agent**——典型 Gemini CLI。后者更适合长任务和需要验证的代码任务。"

---

## 四、典型工程"小技巧"清单（用 1 个就能让面试官记住你）

> 不需要全用，挑 3~5 个你真的能讲细节的。

1. **PreCompact handoff 模式**：在上下文 compact 之前，让模型主动把关键状态写到文件，next session 用 file 重建。**比 in-place compaction 更不掉信息。**（Anthropic 11 月帖）
2. **Sprint Contract**：generator 和 evaluator 在开工前先就"做完长什么样"达成契约。**减少 evaluator 事后挑刺。**（Anthropic 3 月帖）
3. **deny > allow 永远成立**：hook 即便返回 allow，命中 deny rule 也立即拒。**安全设计的不可绕过原则。**（Claude Code 源码）
4. **Diminishing-returns detector**：连续 3 次新增 <500 tokens 就停 loop。**防止模型在"已无新信息"时空转。**（Claude Code 源码）
5. **Speculative classifier with 2s timeout**：bash 命令先扔 classifier 判危险度，2 秒没回就降级到交互式审批。**安全 + 体验的折中。**（Claude Code 源码）
6. **Cache breakpoint engineering**：把 system prompt 拆成"稳定段（cacheable） + 动态段（不缓存）"，让 cache hit 率从 30% 拉到 80%+。**省成本最快的杠杆。**
7. **Tool partition by concurrency-safety**：只有所有运行中工具都是 concurrency-safe 才开并发，否则 exclusive。**简单但有效的并发安全。**（Claude Code 源码）
8. **Context resets vs Compaction 选择**：context anxiety 强（Sonnet 4.5 那种）选 reset；模型够稳（Opus 4.5）可纯靠 auto-compact。**模型行为决定 harness 设计。**
9. **GAN-style 三 Agent**：subjective 任务给 generator+evaluator+iteration，能突破单 agent 自评偏正问题。
10. **Skills 走平台 vs FS 的取舍**：平台注册=审计强、合规友好；FS 挂载=可移植、社区繁荣。**DeepSeek 极可能要做混合策略。**

---

## 五、面试官最爱"挖坑"的 5 个边界问题

> 这些题不是想难住你，是想看你"知不知道自己不知道什么"。**回答的关键不是给标准答案，而是说出 trade-off。**

### Q1：你怎么理解"模型与 Harness 共同进化"？
> 高分回答：
> "我把它理解成三件事——(1) Harness 给模型提供**训练信号**（用户 trajectory、tool 调用错误、补救成本），让模型在下一代 SFT/RL 里学会"跟这个 harness 更好地配合"；(2) Harness 给模型提供**能力 elicit 接口**（thinking、tool spec、skills），新模型一旦放出新能力（如 nested tool call），harness 第一时间能消费；(3) Harness 给模型提供**回退安全网**（permission、sandbox、verification），让研究员敢于放出更激进的模型能力——共同进化的副产品就是研究员越来越敢做大胆训练。"

### Q2：上下文超长怎么办？
> 高分回答：
> "三个层次：(1) **压缩**——summarize 已观察的 tool 结果（最简单但易掉信息）；(2) **重置**——清空 context 起新 agent，用 file artifact 传 state（Anthropic 11 月推荐）；(3) **重构**——把"上下文太长"识别为"任务分解粒度不对"，把长任务砍成多 sprint，每个 sprint 自带 contract 和验证（Anthropic 3 月推荐）。哪个用哪个取决于模型对'context anxiety'的敏感度——Sonnet 4.5 需要 reset，Opus 4.5 用 auto-compact 就够。"

### Q3：怎么评估 Agent 的好坏？
> 高分回答（详见文档 13）：
> "我会同时设三类指标——(1) **任务级**：task success rate、resolved rate（如 SWE-bench Verified/Pro）；(2) **轨迹级**：trajectory efficiency（步数 / token / 工具调用次数 vs 最优解）、tool call accuracy、自我修复率；(3) **体验级**：interrupt 率（用户打断 agent 的频率，反映信任度）、retention、NPS。**对'更多场景帮到更多人'，最易被忽略的是 retention by use case——按场景切分留存。**"

### Q4：Tool 出错 / 模型乱调 tool 怎么办？
> 高分回答：
> "分两层。**模型侧**：tool spec 写清楚 input schema 和典型用法，error message 要 model-friendly（直接告诉模型"你少了 param X"，而不是 stack trace）。**Harness 侧**：validateInput 守边界、permission 守安全、retry with backoff、StuckDetector（重复操作 / 进度停滞 / 操作循环）+ Circuit Breaker（连续 3 次 fail 断路）。最关键的是**把 error 当成 context 注入下一轮**——而不是吞掉。"

### Q5：DeepSeek Harness 应该做开源吗？
> 高分回答：
> "我会推荐**核心 harness 开源 + skills 生态开源 + 评测集开源**，但**对 DeepSeek 模型做 reference 优化**保留商业空间。理由：(1) 开源是 DeepSeek 历史心法，已有用户认知；(2) Harness 工程门槛低（Claude Code 51 万行很多是 UI 渲染），靠开源拿不到护城河——护城河是模型 + 评测 + 数据飞轮；(3) 开源能让 DeepSeek-Coder 模型的市场份额涨——Harness 是 wrapper，wrapper 越广，底层模型越赚。但**第一年别开**——先内部 ship，跑通 PMF，第二年再开。"

---

## 六、相关一句话工程引用（用于显示你读过原文）

| 出处 | 引用 |
| --- | --- |
| Anthropic 2026-03 | "Agents tend to respond by confidently praising the work even when quality is obviously mediocre." |
| Anthropic 2025-11 | "Agents either attempt too much at once or prematurely declare work complete." |
| LangChain 实证 | "仅改进 Harness（不换模型）准确率从 52.8% 提升到 66.5%。" |
| Anthropic best-practice | "Claude's context window fills up fast, and performance degrades as it fills." |
| Claude Code 源码注释 | "deny rules always win, even if a hook returns allow." |
| Cursor PM JD | "Titles matter less than the quality of what ships." |

> **使用方法**：面试中**最多引用 2 句**，多了像在背书。挑最贴当下话题的 1 句"杀招"用即可。

---

## 七、自我考核：每个词 30 秒讲清

把第一节表格遮住，逐个词朗读你的 30 秒版本。**做不到的词就回去重看**。考核标准：

- ✅ 能说出"它是什么"
- ✅ 能说出"为什么需要它"
- ✅ 能给出"1 个具体产品 / 工程例子"
- ✅ 能说出"和它最容易混的概念是什么"

四件事做不齐的，全部回去补。
