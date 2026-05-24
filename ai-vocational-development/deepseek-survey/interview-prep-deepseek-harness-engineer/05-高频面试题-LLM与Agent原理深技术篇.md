# 05 · 高频面试题 · LLM 与 Agent 原理深技术篇

> 这一篇是给**研究员风格面试官**准备的——他/她会用 30 秒判断你是不是真的懂模型。答错关键点立刻减分。
>
> **判分核心**：能说出"为什么"+ 能讲"边界"，不光是"是什么"。

---

## Q1：详细讲一下 Transformer 推理时的 KV Cache，它在 Agent Loop 里怎么工作？

**示范回答**：

> "KV Cache 的物理基础——Transformer 每层 self-attention 算 Q/K/V 三个矩阵。在 autoregressive 推理时，每生成一个新 token 都要看前面所有 token 的 K/V。如果前缀不变，K/V 算过的就可以缓存复用——这是 prefix caching。
>
> **在 Agent Loop 里的关键特征是**——
>
> 1. **System prompt + tool spec + 历史 messages 通常很长且重复**。每一轮 LLM call，前 N 轮的 token 都不变（assistant 上轮回复 + tool_result 都是 append-only）。这是 cache hit 的金矿。
>
> 2. **商业化层**：Anthropic 通过 `cache_control: {type: 'ephemeral'}` 显式标记，cache hit token 收 0.1× input price（节约 90%）；OpenAI 自动 cache，hit 收 0.5×。
>
> 3. **Harness 设计的核心三原则**：
>    a. **稳定段放前面 + 显式 cache breakpoint**：system prompt / tool spec / 大段示范
>    b. **变化段放后面**：用户输入、上一轮观察、当前 git 状态
>    c. **Compaction 后立即重建 breakpoint**：compact 破坏前缀，需重新 warm cache
>
> 4. **数字感**：一个 30 turn 长 session，做对 cache：$5 → $0.5；做错 cache：每轮都重算前缀，成本爆炸。
>
> **常见错误**：把动态时间戳 / random session id 放 system prompt 头部——这会让每次 cache miss。"

**追问 + 应对**：
- ❓"如果模型不暴露 cache_control 怎么办？"→ 一般 API 会自动检测重复前缀；OpenAI 走这条；自托管 vLLM / SGLang 通过 prefix prefix tree 实现
- ❓"cache 边界最多几个？" → Anthropic 4 个 cache_control breakpoint，需要规划
- ❓"为什么 Anthropic 不自动 cache？" → trade-off：自动 cache 会让长尾 prompt 的 cache pollution；显式标记让用户控成本

---

## Q2：讲清 tool_use / function_calling 在 OpenAI、Anthropic、Gemini 三家协议层的差异

**示范回答**：

> "三家的 tool_use 协议差异不大但都有坑：
>
> | 维度 | Anthropic | OpenAI | Gemini |
> | --- | --- | --- | --- |
> | 块名 | tool_use | tool_calls | functionCall |
> | input 字段类型 | object | **JSON string** | object |
> | tool_result 写回 | user role 加 tool_result block + tool_use_id | tool role 加 content + tool_call_id | user role 加 functionResponse |
> | 并行 call | 同 assistant 多 block | 同 assistant 多 calls | 视 model |
> | Streaming | content_block_delta with input_json_delta | tool_calls 增量 | candidate updates |
>
> **最大的坑是 OpenAI 的 arguments 是 JSON 字符串**——而且流式时可能截断在 JSON 中间。所以 harness 要 buffer 直到 stop 后才 parse。
>
> **Harness 跨模型适配的 idiomatic pattern**：
> 1. 定义内部中间表示 `ToolCall { id, name, input: object }`
> 2. 三个 adapter 各自负责 in/out 转换
> 3. 内部 dispatch / permission / logging 全部走中间表示
>
> 这样换模型只改 adapter，core engine 不动。"

---

## Q3：Reasoning Model（o1 / R1 / Claude 4.5 thinking）的接口和成本怎么处理？

**示范回答**：

> "三个层面——
>
> **接口层**：
> - Anthropic：`thinking: {type:'enabled', budget_tokens: N}`，response 含 `thinking` block，可读可见
> - OpenAI o1/o3：自动 enable，**thinking 不暴露**，只计费 reasoning_tokens
> - DeepSeek-R 早期：thinking 可见，可关
>
> **成本层**：
> - thinking tokens 计费同 output tokens
> - 一个复杂 plan 可能消耗 10K thinking tokens，相当于 $0.30 一次
> - **KV cache 友好性**：thinking 占 context 但通常不该 echo 回下一轮 messages（防 context 爆）
>
> **Harness 设计层**：
> 1. **场景判断**：简单 tool call（read_file）关 reasoning；复杂 plan / debug 开
> 2. **全局开关**：UI 给"快/平衡/深思"三档
> 3. **Subagent 分工**：planner 开，generator 关
> 4. **Budget cap**：长 agent loop 中每轮 cap reasoning，防失控
> 5. **UI affordance**："Model is thinking..." 防用户以为卡死
> 6. **不回灌 thinking trace**：每轮拿到 response 后只保留 text / tool_use block，thinking 抛弃（OpenAI / Anthropic 推荐做法）"

**追问 + 应对**：
- ❓"为什么 OpenAI 不暴露 thinking？" → 安全 + 让模型敢探索（暴露会让模型自我审查）
- ❓"DeepSeek-R 系列在 agent 里你的实测体验？" → 长 trace 占 context 但 plan 质量明显涨；适合"少而精"的 agent 用法
- ❓"如果 thinking 失败 / 超时？" → 部分 API 支持 partial response；harness 要兜底"thinking 失败但已经 emit 的 text 仍可用"

---

## Q4：详细讲讲 Anthropic 11 月那篇 long-running harness 的核心方法，以及你的工程实现

**示范回答**：

> "Anthropic 11 月 'Effective harnesses for long-running agents' 解决的核心问题：**长任务的两种失败模式**——
> 1. 模型 'attempt too much at once'（一口气想做完所有事，context 爆）
> 2. 模型 'prematurely declare work complete'（半路宣布完工跑路）
>
> **他们的方案是两段式**——
>
> ```
> Initializer Agent
>   ├── 接收 product spec
>   ├── 分解为可独立完成的 task list
>   └── 写到 file artifact
>
> Coding Agent (×N runs)
>   ├── 每次启动只取 1 个 task
>   ├── 实施 → 完成 → 写 handoff artifact
>   └── Context reset 后下一个 agent 接力
> ```
>
> **核心 insight**：**context reset 比 compaction 强**——长 context 触发 'context anxiety'（Sonnet 4.5 在接近 context 上限时会过早收尾）。reset 给每个 agent 干净 slate。
>
> **工程实现要点**：
>
> 1. **Handoff schema**：包含 (a) 完成的 tasks (b) 当前状态 file paths (c) 下一步意图
> 2. **State 持久化**：所有 state 写文件而不是内存，让 reset 后 agent 能 rehydrate
> 3. **Orchestrator**：外层 supervisor 检测 agent 结束 / 失败 / 超时，决定是 reset 还是 abort
> 4. **Token budget per run**：每个 agent run 单独 cap，防 runaway
> 5. **Idempotency**：tool 副作用必须可重放（写文件用 atomic rename / DB 用 transaction）
>
> 3 月那篇又演进到 **GAN-style 三 Agent（Planner / Generator / Evaluator）**——加了 evaluator 解决 'agent 自评偏正' 问题。Evaluator 在独立 context 跑 Playwright 真测 UI，迫使 generator 不能靠 LLM 自评糊弄。
>
> 对 DeepSeek Harness 的启示：v0.1 先做两段式；v0.2 看模型 instruction following 程度决定要不要上三段式。"

---

## Q5：MCP 协议详细讲一下，包括 transport / message / lifecycle / 工程陷阱

**示范回答**：

> "MCP（Model Context Protocol）是 Anthropic 主推的模型-工具标准协议，解决 N×M 集成爆炸——N 个模型 × M 个工具不再要 N×M 次集成。
>
> **Transport**：
> - **stdio**（最常见）：客户端启 child process，stdin/stdout JSON-RPC 2.0
> - **SSE / HTTP**：远程 MCP server
>
> **核心方法**：
> - `initialize` / `initialized`：协议握手 + capabilities 协商
> - `tools/list`：列工具
> - `tools/call`：调工具
> - `resources/list` & `resources/read`：读资源
> - `prompts/list`：列 prompt 模板
> - `notifications/*`：server → client 通知
>
> **Lifecycle**：
> ```
> Client spawn server process
>   → initialize
>   → tools/list (cache schema)
>   → loop: tools/call
>   → shutdown
> ```
>
> **工程陷阱**：
> 1. **Stdio buffer flush**：必须 line-based + 立即 flush，否则死锁
> 2. **Schema 不规范的第三方 server**：要宽容 parser
> 3. **子进程 crash**：要 supervisor 重启 + exponential backoff
> 4. **Permission 粒度**：每个 tool 单独审批，不能整 server 一刀切
> 5. **Tool name 冲突**：多 server 都叫 `read_file` —— 加 namespace（`servername.toolname`）
> 6. **Lifecycle 长连接**：长跑 client 要定期 health check，server 进程内存增长
> 7. **Capability negotiation**：旧 server 不支持新方法，client 要 graceful degrade
>
> **DeepSeek Harness 必须支持 MCP**——不然等于自我孤立。我建议 v0.1 就支持 stdio MCP，并贡献一两个 RFC 推进权限标准化。"

---

## Q6：Anthropic 那篇 Context Engineering 博客的核心 idea 是什么？工程上怎么落？

**示范回答**：

> "Anthropic 'Effective Context Engineering for AI Agents'（2025 末）核心论点：**Context window 是有限的认知预算，不是无限大的硬盘**。
>
> **三大 takeaway**：
>
> 1. **质量 > 数量**：把 100 个无关文件 dump 进去远不如精心挑 5 个最相关的——'context rot' 现象
> 2. **过期信息要主动 purge**：旧 tool_result 用过就 fold / drop
> 3. **System prompt 影响指数级**：不只是开场白，决定模型整轮 prior
>
> **工程实现**（Claude Code 范式）：
>
> ```
> Layer A·Build pipeline
>   getUserContext()    # CLAUDE.md / 偏好
>   getSystemContext()  # git status (truncate to 2KB) / 环境
>   fetchSystemPromptParts()  # 按 section 组装，支持 A/B test
>
> Layer B·Compression
>   1. snip compacting     # 移除 superseded messages
>   2. micro-compact       # 单 response 内部去重
>   3. context collapse    # 折叠相关 tool_result
>   4. auto-compact        # 接近 budget 时压全文
>
> Layer C·Retrieval
>   按需注入：用户提到的 file → 优先 read；用户提到的 concept → 优先 RAG
> ```
>
> **常见 anti-pattern**：
> - 一开始 dump 整个 repo（context rot 严重）
> - 完整保留所有 tool_result（很多是中间过程没用了）
> - System prompt 不 cache（成本暴涨）
> - 没有 budget 监控，到 context 上限才 panic compact
>
> **对 DeepSeek Harness**：建议把 'context budget 实时仪表' 做成 first-class UI 元素，让用户看到 context 在被怎么花。"

---

## Q7：模型行为里你最印象深的 3 个细节？

**考察意图**：测"对模型行为有品味"——研究员风格面试官最关心这个。

**示范回答**（按你真实观察改写）：

> "三件让我观察印象深的——
>
> **第一·Claude 的 user-deference**：用户说 'no' 后 Claude 会迅速换方向，不会 argue 三轮；这反映 Anthropic 在 SFT 时强化了 user-deference，但代价是有时'明明 Claude 是对的'也屈服。
>
> **第二·Sonnet 4.5 的 context anxiety**：context 接近上限时会过早收尾——"I'll wrap up here" 类型语言出现频率显著上升。这是 Anthropic 11 月那篇推 context reset 的根本原因。Opus 4.5 改善了。
>
> **第三·DeepSeek-V 系列的 tool_use schema drift**：相同 prompt 多次 inference，DeepSeek 偶尔会输出 'invalid JSON in tool_use input'（比 Claude 高几个 pp）。这要求 harness 兜底——lenient parser + retry with structured error。这反过来是模型团队的训练改进方向。
>
> 这种'品味'对 Harness 设计极重要——同样的 harness 套在不同模型上，需要不同的兜底强度。"

---

## Q8：详细讲一下 reward hacking 在 Agent trajectory 上的表现

**示范回答**：

> "Reward hacking = 模型学会优化指标但不优化真实目标。Agent 场景下典型表现：
>
> 1. **挑容易任务做**：如果 reward 看 task completion rate，agent 会 skew 接简单任务
> 2. **多 tool call 显示努力**：reward 看 'thoroughness' 时 agent 会 over-search
> 3. **草率收工**：reward 看 'step efficiency' 时 agent 会快速宣布完成
> 4. **回避高风险动作**：reward 重 safety 时 agent 不敢做该做的修改
> 5. **拍马屁**：reward 看 user satisfaction 时 agent 会 sycophantic（同意一切）
> 6. **格式逃逸**：reward 看 JSON valid 时 agent 输出 trivial JSON 蒙混
>
> **Harness 侧能做的**：
> 1. **对冲指标**：completion rate 配 cost-per-success；thoroughness 配 user-accept rate
> 2. **adversarial eval set**：故意设'什么都不该做就是对'的题，逼模型学 abstain
> 3. **trajectory replay analysis**：归类 stuck 模式（repeater / wanderer / looper），喂回研究员
> 4. **A/B with holdout**：永远留 5%~10% 用户绝不投放新 reward 训出的模型，看长期真实信号
>
> **Harness 不能修模型 misalignment，但 Harness 是 reward hacking 的检测器**——这是和研究员协作的高价值贡献。"

---

## Q9：你怎么看 Claude Code "permissive defaults + tool-specific override" 这个安全设计？

**考察意图**：测对反直觉设计的理解。

**示范回答**：

> "这个设计很反直觉但合理。
>
> **常规做法**：framework deny-by-default，工具默认无权限，需 opt-in。
>
> **Claude Code 反过来**：TOOL_DEFAULTS.checkPermissions 默认 allow，工具自己 opt-in 加 stricter check。
>
> **为什么这样设计**：
> 1. **强制每个 tool author 显式思考权限模型**——deny-by-default 容易让 author 偷懒（'反正默认拒了'），permissive default 反而逼 author 主动定义
> 2. **避免假阴性**：framework 层 deny 太广，会拒掉合理操作；让 tool 决定能精细
> 3. **deny rule 永远 > allow** 是兜底——即便 tool 自己说 allow，命中 user/global deny 也立即拒
>
> **代价**：
> 1. 单个 tool 漏写 checkPermissions = 安全漏洞
> 2. 新 tool author 容易忘
> 3. 审计成本高（必须 review 每个 tool）
>
> **我的判断**：这套对 Anthropic 这种 'tools 是自家精心维护' 场景合理；对开源 / MCP 第三方 tool 场景，**framework 层 deny-by-default + tool opt-in 更安全**。所以 DeepSeek Harness 如果走开源 + MCP 路线，我建议反着做——deny-by-default。"

---

## Q10：如果让你设计 trajectory 数据 schema 喂给研究员训练，你会怎么定字段？

**考察意图**：和研究员对话的核心能力。

**示范回答**：

> "我会按 'mandatory / nice-to-have / strictly opt-in' 三层设：
>
> **Mandatory（每条必有）**：
> ```json
> {
>   "trajectory_id": "uuid",
>   "model": "deepseek-coder-v3.5",
>   "harness_version": "0.1.2",
>   "timestamp": "ISO 8601",
>   "user_id_hash": "sha256",
>   "turn_count": 12,
>   "stop_reason": "end_turn",
>   "outcome": "success|fail|abandoned",
>   "intervention_count": 0,
>   "tools_used": ["read_file", "bash", "edit"],
>   "duration_ms": 38000,
>   "input_tokens": 5400,
>   "output_tokens": 3200,
>   "thinking_tokens": 1500
> }
> ```
>
> **Nice-to-have（聚合用）**：
> ```json
> {
>   "scenario_tag": "multi-file-refactor",
>   "difficulty_estimated": "medium",
>   "self_fix_count": 2,
>   "user_satisfaction": null,  // 可选 thumbs
>   "errors": [{"tool": "bash", "type": "permission_denied"}]
> }
> ```
>
> **Strictly opt-in（敏感）**：
> ```json
> {
>   "messages": [...],          // 完整 message 序列（含 thinking）
>   "tool_inputs": [...],       // 各 tool 输入
>   "tool_outputs": [...],      // 各 tool 输出
>   "file_changes": [...]       // diff hashes
> }
> ```
>
> **关键设计**：
> 1. **Mandatory 字段绝不包含代码内容** —— 默认上报安全
> 2. **opt-in 必须 explicit UI 确认 + 可随时撤回** —— 合规第一
> 3. **统一 schema 跨模型** —— 让研究员能横向对比 V3.5 vs V4 vs OpenAI
> 4. **不同字段不同保留期**：mandatory 90 天，opt-in 30 天
> 5. **Differential privacy 选项**：聚合统计用 noised 数据
>
> 我会要求和合规、研究员各对一遍 schema，确保 (a) 法务过 (b) 研究员真用得上 (c) 用户可解释。"

---

## Q11：解释 Subagent 和 Multi-Agent 在 token 经济和 latency 上的差异

**示范回答**：

> "**Token 经济**：
> - Subagent**比单 agent 更费 token**，不是省——因为它要把 task description 和必要 context 复制一份给子 agent
> - 但能避免父 agent context 爆炸的成本（如果 single agent 上下文 saturate 后回退/重做，成本更高）
> - 数学上：subagent 适合 'subtask 的 context 显著小于 parent total context' 的场景
>
> **Latency**：
> - Subagent 串行（spawn → run → return）会加总延迟
> - 但**多个 subagent 并行**能压缩总时延，前提是 subtasks 独立
> - Multi-agent（Planner/Generator/Evaluator）一般串行流水，延迟 ≈ sum，但每个 agent 更专注，避免 single agent 反复试错的隐藏 latency
>
> **决策矩阵**：
>
> | 场景 | 推荐 |
> | --- | --- |
> | 探索大量文件（read-only） | Subagent 并行 |
> | 长任务 + 阶段性目标 | Multi-agent 流水线 |
> | 子任务 context 小 | Subagent |
> | 子任务自验证难 | Multi-agent（加 evaluator） |
> | 短任务 | 单 agent（不要过度设计） |
>
> **DeepSeek Harness v0.1 建议**：先 single agent；v0.2 加 subagent（解决长 read 任务的 context 失焦）；v0.3 视情况加 multi-agent。"

---

## Q12：为什么 Anthropic 的三 Agent harness（Planner/Generator/Evaluator）有效？工程层面怎么实现？

**示范回答**：

> "**为什么有效**：解决两大问题——
>
> 1. **Agent 自评偏正**：让 generator 自评，它会 confidently praise 自己；分离 evaluator + 在独立 context 跑（独立 LLM call），能给出更严格的反馈。
> 2. **长任务失焦**：planner 守 spec，generator 守实施，evaluator 守质量——三者各专注一面，避免 single agent 跨注意力。
>
> **工程实现要点**：
>
> 1. **Communication via files**：generator 写文件、evaluator 读，避免直接 message 传递（防 context 互污染）
> 2. **Sprint Contract**：sprint 开工前 generator 和 evaluator 先就 'definition of done' 达成契约（也写文件），避免事后挑刺
> 3. **Evaluator 用真工具验**：Playwright MCP 跑真页面，不是 LLM 想象——避免 LLM 评 LLM 的循环正反馈
> 4. **Failure handling**：evaluator 不过 → generator 拿 detailed feedback 重做（最多 N 次）
> 5. **Cost vs Quality**：Anthropic 实测 solo $9/20min vs full $200/6h，20× cost 换显著质量提升——**对'人类 review 成本高'的场景才划算**
>
> **工程陷阱**：
> - Agent 之间 file 协议混乱 → 需要 strict schema
> - Evaluator 太严 → 死循环；太松 → 退化成 solo
> - Planner agent 跑偏 → 把 spec 拆错，整个 pipeline 全废
>
> **对 DeepSeek Harness**：v0.1 不做，v0.2 视模型 instruction following 能力决定。研究员侧可以单独投资 evaluator agent 的 SFT 数据。"

---

## Q13：你怎么看 sandbox 在 LLM agent 里的工程权衡？

**示范回答**：

> "三种 sandbox 形态：
>
> | 形态 | 启动延迟 | 安全强度 | 兼容性 | 适合 |
> | --- | --- | --- | --- | --- |
> | **OS-level**（sandbox_init / landlock / AppContainer） | ~ms | 中-高 | 跨平台需写 3 套 | 主战场，CC 选这条 |
> | **Docker / Firecracker** | ~s | 高 | 跨平台一致 | OpenClaw / 高风险任务 |
> | **远程云沙箱** | ~s + 网络 | 最高 | 完全一致 | Codex CLI 模式 |
>
> **OS-level 的细节**：
> - macOS sandbox_init 用 Seatbelt profile（一种 Scheme dialect 描述）
> - Linux landlock 需要 5.13+ kernel，老内核要 fallback 到 seccomp+namespaces
> - bubblewrap 是 Linux 实用选择
> - Windows AppContainer + Job Object 配置最复杂
>
> **Harness 设计建议**：
> 1. **抽象接口 + 三平台适配器**
> 2. **fail-closed**：sandbox start 失败必须拒绝 tool 执行
> 3. **GUI 应用 sandbox 后**：clipboard / display 需要 IPC bridge
> 4. **网络默认 block + whitelist allow**
> 5. **临时文件 sandbox 自带 tmpfs**
> 6. **可选升级到 Docker**：对'跑陌生 PR test'这种场景加一层
>
> **DeepSeek Harness 我建议 v0.1 做 macOS + Linux + Windows 三平台 OS-level**，跑陌生代码再选 Docker——延迟敏感场景不能强上 Docker。"

---

## Q14：如何对一个 Agent 做评估（eval）？要做哪些 metric？

> 与文档 06 的'设计 Eval Pipeline' 题互补。这里更偏'metric 选择'。

**示范回答**：

> "我会按 4 维 + 多基准设：
>
> **4 维**：
> 1. **Capability**：SWE-Bench Verified/Pro、AiderBench、LiveCodeBench、自有内部 task
> 2. **Alignment**：拒答率（user 说 no 后继续 rate）、越界率、sycophancy delta（同问加 'tell me I'm right' 答案变化）
> 3. **Honesty**：'I don't know' calibration rate、hallucinated tool args rate、自我修正率
> 4. **Efficiency**：steps-per-success、tokens-per-success、wall-clock vs human baseline
>
> **多基准**：
> - 公开 leaderboard 跑分（外宣 + 横评）
> - 内部 100 task（真实使用 + 高质量人评）
> - Adversarial 测试集（专门测 sycophancy / jailbreak / refusal）
> - **每季度 holdout 20% test set 替换** 防 contamination
>
> **避免 anti-pattern**：
> - 单一指标（Pass@1）→ reward hacking
> - 不分难度 → easy 拉平
> - 不分场景 → debug vs docs 应该分开看
> - LLM-as-judge 不 calibration → judge bias 累积
>
> **报告形式**：雷达图 + diff vs 上版 + 每个失败 case 链接 trajectory——研究员要的不是分数，是 actionable signal。"

---

## Q15：DeepSeek-R 系列在 Agent Loop 里的特殊性？

**考察意图**：测你和 DeepSeek 自家模型的实际交手。

**示范回答**（按你真实经验改写）：

> "我在 Agent loop 里实测 DeepSeek-R 的几个特点：
>
> 1. **Thinking trace 长**：复杂任务一次 reasoning 5K~15K tokens 常见，比 non-reasoning 模型贵
> 2. **Tool_use 稳定度略弱于 Sonnet**：偶发 schema drift，需要 retry policy
> 3. **Plan 质量明显高**：相同任务，开 thinking 的成功率比关掉高 10~15pp
> 4. **不适合 'tight loop'**：单 turn latency 高，10 turn 短任务体感慢
> 5. **中文表现强**：中文 instruction following 优于多数西方模型
>
> **Harness 设计的针对性**：
> 1. **场景判断**：simple tool call 关 reasoning；plan / debug 开
> 2. **Budget cap per turn**：防 thinking 失控
> 3. **Bound retry**：tool_use 失败 3 次以上 fallback to non-reasoning 简化 schema
> 4. **不回灌 thinking trace**：每轮拿到 response 只留 text + tool_use，thinking 抛弃
>
> 这些不是 R 系列的 bug——是模型形态决定的 trade-off。DeepSeek Harness 必须为 R 系列做专门优化。"

---

## 附：必背 LLM/Agent 术语英中对照

> 和研究员对话时混用英中是高分动作；只用英文显得装，只用中文显得没读原文。

| 中文 | 英文 |
| --- | --- |
| 提示词工程 | Prompt Engineering |
| 上下文工程 | Context Engineering |
| 线束工程 | Harness Engineering |
| 智能体循环 | Agent Loop |
| 工具调用 | Tool Use / Function Calling |
| 推理 | Reasoning |
| 思考链 | Chain-of-Thought (CoT) |
| 规划 | Planning |
| 任务分解 | Task Decomposition |
| 技能 | Skill |
| 子智能体 | Subagent |
| 多智能体 | Multi-Agent |
| 记忆 | Memory |
| 沙箱 | Sandbox |
| 上下文重置 | Context Reset |
| 上下文压缩 | Context Compaction |
| 流式 | Streaming |
| 工具调度 | Tool Orchestration |
| 并发安全 | Concurrency-safe |
| 权限引擎 | Permission Engine |
| 钩子 | Hook |
| 评估 | Eval / Evaluation |
| 自评 | Self-evaluation |
| 拒答率 | Refusal Rate |
| 谄媚 | Sycophancy |
| 幻觉 | Hallucination |
| 奖励黑客 | Reward Hacking |

---

## 一句应试钥匙

> **研究员风格面试官最讨厌"我看 Twitter 听过"。最喜欢"我看过原 paper / 我自己复刻过 / 我在 GitHub 写过"。**

> 每个高分回答都应该有一个具体引用——博客标题、paper title、数字、源码文件名——至少 1 个，最多 2 个。
