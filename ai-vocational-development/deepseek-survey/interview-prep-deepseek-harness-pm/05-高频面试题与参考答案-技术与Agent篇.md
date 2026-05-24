# 05 · 高频面试题与参考答案 · 技术与 Agent 篇

> 这一篇覆盖偏技术/Agent 工程类问题。重点不是测你能不能"开发"，而是测你能不能**和研究员/工程师对齐**——这是 PM 的硬通货。

---

## Q1：详细解释一遍 Agent Loop 是怎么跑的，每一步在干什么

**考察意图**：测你是否真的看过/写过 agent 源码。

**示范回答（带白板讲解版）**：

> "用 Anthropic 的 Messages API + tool_use 举例。
>
> ```
> messages = [{'role':'user', 'content': user_input}]
>
> while True:
>     response = client.messages.create(
>         model, messages, tools=TOOL_SCHEMAS, system=SYSTEM_PROMPT
>     )
>
>     # 把 assistant 输出整段塞回 messages
>     messages.append({'role':'assistant','content': response.content})
>
>     if response.stop_reason == 'end_turn':
>         break
>     if response.stop_reason == 'tool_use':
>         tool_results = []
>         for block in response.content:
>             if block.type == 'tool_use':
>                 result = TOOLS[block.name](**block.input)  # 实际执行
>                 tool_results.append({
>                     'type':'tool_result',
>                     'tool_use_id': block.id,
>                     'content': str(result)
>                 })
>         messages.append({'role':'user','content': tool_results})
>         continue
>     # 其他 stop_reason: max_tokens / stop_sequence / pause_turn
> ```
>
> 工业级版本要加：
> 1. **流式输出**：用 SSE / streaming 把 thinking/text/tool_use block 实时吐给 UI（中断要立即生效）
> 2. **预算控制**：max_turns、max_budget_usd、max_tokens 三层
> 3. **diminishing returns**：连续 N 轮 token 增量 < 阈值就强停
> 4. **错误处理**：tool 抛错时把 error message 当 tool_result 注入下一轮，**让模型自己读 error**——这是核心 trick
> 5. **并发**：concurrency-safe 的 tool 才并发，不安全的 exclusive
> 6. **中断与恢复**：把当前 messages 序列化到 checkpoint，重启时 load 回来"

---

## Q2：用一段话讲清 KV Cache 在 Agent 场景下为什么重要

**示范回答**：

> "Transformer 推理时每个 token 在每一层都要算 K/V 矩阵，前缀如果不变就可以复用——这是 prompt cache 的物理基础。
>
> **Agent 场景的关键观察是**：每一轮循环里，前 N 轮的对话历史是不变的，只在末尾追加一段新 tool_result + assistant 思考。如果你能让 system prompt 和历史这一大块走 cache，**计费只算新增的 delta token**——Anthropic 的 cache_control 给到 90% 折扣。
>
> Harness 设计的 cache 友好原则有三个：
> 1. **System prompt 切两段**：稳定段（角色 + tool spec + 规则）放前面 cache_control，动态段（git status / 时间戳）放后面
> 2. **Tool result 大块尽量靠后**：因为它会变，往后放不污染前缀 cache
> 3. **Compaction 时机和 cache 边界对齐**：每次 compact 后立即写一个新的 cache breakpoint
>
> **业务影响**：一个长 session（30+ 轮 tool_use）做对 cache，token 成本能从 $5 降到 $0.5。这是 Harness 真正能省钱的地方，比换模型省得多。"

---

## Q3：横向对比 Claude Code、Cursor、Codex CLI、OpenCode，每个有什么独到设计？

**示范回答**（表格化输出，背骨架就行）：

| 维度 | Claude Code | Cursor | Codex CLI | OpenCode |
| --- | --- | --- | --- | --- |
| **形态** | Terminal TUI（Bun + Ink fork） | VSCode fork（GUI IDE） | CLI + Web + 云沙箱 | Terminal CLI（开源） |
| **核心循环** | 单 agent + Task subagent | Tab 补全 + Agent Mode 双轨 | Agent + sandbox 双层 | 单 agent，CC 风格 |
| **工具系统** | 40 tools，强 Zod schema | 内置 + 自动 codebase index | 内置 + sandbox API | 模块化，跨模型 |
| **权限/安全** | 8 模式 + bash classifier + deny>allow | 较弱（GUI 自动确认多） | sandbox + Policy Engine | 简化 |
| **Skills/扩展** | Anthropic 平台注册，强审计 | Rules + .cursorrules + MCP | MCP 为主 | Skills 走 FS 挂载 |
| **多 Agent** | Task tool（subagent） | 暂无强 multi-agent | 远程 task 多类型 | 暂弱 |
| **独到设计** | section-based system prompt 支持 A/B；PreCompact handoff；DECSTBM 硬件滚动渲染 | Tab 上的 next-edit 预测；codebase index pipeline | 双层执行（本地/云）；远程 task 工厂模式 | 真·跨模型；FS 风格 Skills 生态友好 |
| **致命弱点** | 终端门槛、Skills 平台封闭、贵 | Agent Mode 偶尔幽灵编辑；index stale | CLI 体验粗糙、文档分散 | 早期生态、质量参差 |

**收尾一句**："如果让我给 DeepSeek Harness 偷设计，我会偷 CC 的 PreCompact handoff + Cursor 的 codebase index + Codex 的双层执行 + OpenCode 的跨模型——这四件事任何一件单独做就能比同梯队强。"

---

## Q4：如果模型输出格式错了（比如该 JSON 但漏了一个引号），Harness 怎么处理？

**考察意图**：测你对"模型不可靠"的工程直觉。

**示范回答**：

> "三层 fallback：
>
> **L1·宽容解析**：先用 lenient parser（如 `json5` / `dirty-json`）尝试修复尾逗号、漏引号等小问题。
>
> **L2·结构化重提示**：解析失败时，把 error message + 期望 schema 注入下一轮 prompt：
> > 'Your last tool_use input failed schema validation: missing closing brace at line 3. Please retry with valid JSON matching {schema}.'
> 模型通常一轮就修好。
>
> **L3·硬保护**：连续 3 次 schema 失败 → Circuit Breaker 触发 → 提示用户 'agent 当前对该工具的 schema 表现异常，建议降级到手动模式'，并把这条 trajectory flag 给训练团队（这是高质量训练样本）。
>
> 配套的工程：tool schema 用 Zod / Pydantic 统一定义、validateInput 在 tool dispatch 前必跑、error message 写成模型可读（不是 Python stack trace）。"

---

## Q5：怎么设计一个让 Agent 能"安全执行 bash 命令"的方案？

**示范回答**：

> "参考 Claude Code 的 bashSecurity 设计，做四层防护：
>
> **L1·语法层**：用 bash AST parser 把命令解析成 tree，识别 23 类已知危险（shell metacharacter / 命令替换 / 重定向到敏感路径 / `IFS` 注入 / zsh 特定 18 个危险命令 / ...）
>
> **L2·语义层**：bashClassifier（可以是小模型或规则）判 high/medium/low 危险度。`rm -rf /tmp/build` 和 `rm -rf /` 语法都是 `rm -rf`，但语义完全不同——这层必须能区分。
>
> **L3·权限层**：classifier 输出对接 Policy Engine：high → 强制 ask user；medium → 可选 ask（按 mode）；low → 自动通过。**关键设计：classifier 有 2 秒超时，超时降级到 ask user**——不能因 classifier 卡死阻塞 agent。
>
> **L4·执行层**：在 sandbox 跑（FS isolation + 网络隔离）；timeout 强制 kill；stdout/stderr 截断；返回结构化结果（exit_code + stdout + stderr + duration）。
>
> 还有一条**铁律**：deny rule 永远 > allow——即使用户/hook 说 allow，命中 deny 规则也立即 block。这是不可绕过的最后一道防线。"

---

## Q6：怎么实现一个 Agent 的"中断与恢复"？

**示范回答**：

> "中断和恢复要分别设计。
>
> **中断 (interrupt)**：
> - UI 侧：用户按 Esc / Ctrl-C → 立即向 agent loop 发 cancel signal
> - Loop 侧：在每个 await 点检查 signal（API stream / tool execute / hook fire 之间）
> - Tool 侧：每个 tool 都要实现 abort 协议（bash 用 SIGTERM、file read 用 stream cancel、API call 用 AbortController）
> - 反例：很多 agent "假装停了"，但 tool 还在后台跑——这是最差体验
>
> **恢复 (resume)**：
> - **快照**：每轮 loop 结束时把 messages + tool state + budget 序列化到 checkpoint（JSON / SQLite）
> - **断点续传**：重启时 load checkpoint，把 messages 喂回 API（自动享受 prompt cache）
> - **状态一致性**：tool 副作用要可重放/可幂等（写文件用临时文件 + rename、改 db 用 transaction id），不能直接做不可逆操作
>
> **进阶**：让 agent 主动支持 'pause + manual edit + resume'——用户在 pause 状态可以手改 plan 或文件，agent 醒来读到的就是修改后的状态。这是 Cursor / OpenClaw 在做的方向。"

---

## Q7：长任务（multi-hour）你怎么避免 agent 跑歪？

**考察意图**：直接对应 Anthropic 11 月那篇文章。

**示范回答**：

> "我会上 4 件事：
>
> 1. **任务分解**：用 Planner Agent 把任务拆成 N 个可独立完成 + 可验证的 sprint。每个 sprint 自带 'definition of done'。
>
> 2. **Context Reset（不是 compaction）**：每个 sprint 结束新 agent 起一个 fresh context，用 file artifact 传 state。这能消除 'context anxiety'（Sonnet 4.5 在 context fill 接近上限时会过早收尾的现象）。
>
> 3. **Sprint Contract**：sprint 开工前 generator 和 evaluator 先就 '做完长什么样' 达成契约（写到文件里），实施后按这个契约打分。**避免 evaluator 事后挑刺。**
>
> 4. **GAN-style 分离**：generator 干活、evaluator 用独立 context 跑 Playwright/test 验证、planner 看不变量 + 总目标。**三角制衡比单 agent 自评强。**
>
> 配套监控：StuckDetector（重复操作 / 进度停滞 / 操作循环检测）+ Circuit Breaker（连续 3 fail 断路）。
>
> 这是 Anthropic 2025-11 → 2026-03 两篇 harness 博客的连续演进，DeepSeek Harness 走这条路是合理的，但要等模型 instruction-following 到位再上三 agent，否则 agent 们互相不听话。"

---

## Q8：Skills 应该走平台注册 (Anthropic 风格) 还是 FS 挂载 (OpenCode 风格)？

**示范回答**：

> "两条路各有 trade-off：
>
> | 维度 | 平台注册 | FS 挂载 |
> | --- | --- | --- |
> | **审计** | 强（每次更新平台留痕） | 弱（用户机器上随意改） |
> | **合规** | 友好（中心化签发） | 用户负责 |
> | **可移植** | 弱（绑 vendor） | 强（一份文件随处跑） |
> | **社区繁荣度** | 慢（要审核） | 快（git clone 就能用） |
> | **质量分发** | 高（人工筛选） | 良莠不齐 |
>
> 我会推荐 DeepSeek 走 **'混合策略'**：
> - **核心 Skills**（涉及代码执行、外部 API 调用）走平台注册，签名 + 审计
> - **prompt-only Skills**（只是 prompt + few-shot）走 FS 挂载，社区随便贡献
> - 平台上有一个 'verified by DeepSeek' badge，FS 挂载的也能 import 进平台标记 'unverified'
>
> 这样既能保护 enterprise 用户，又能让开源社区自由贡献——**别二选一**。"

---

## Q9：你对 MCP（Model Context Protocol）怎么看？

**示范回答**：

> "MCP 是当前 Agent 生态里**最重要的标准化尝试**——解决的是 N×M 集成爆炸：N 个模型 × M 个工具，如果没有协议就要 N×M 次集成。
>
> **MCP 解决得不错的部分**：
> - 协议简洁（stdio / SSE）
> - 客户端服务端解耦，工具能进程隔离（崩了不带垮 agent）
> - 已经形成事实生态（CC / Cursor / Codex / Cline / OpenCode 全支持）
>
> **MCP 没解决好的部分**：
> - 缺权限标准：每个 client 自己定怎么 ask user，没统一 UX
> - 缺 schema 演进：MCP server 升级了 schema，client 怎么协商版本
> - 缺成本可见性：MCP tool 调用消耗的 token / latency 没标准 metrics
>
> **DeepSeek Harness 的 MCP 策略我建议**：
> - v0.1 **全面兼容**（不抗拒，先拿生态红利）
> - 同时**贡献几个 RFC** 推进权限标准（这是建立国际话语权的最低成本路径）
> - **不要试图发明自己的协议**——这是 OpenClaw ACP 走过的弯路。"

---

## Q10：你怎么定义"一个 Harness 评测好不好"？

**示范回答**：

> "我会同时跑两类评测：
>
> **类一·公开基准**（用于对外宣传 + 横向对比）：
> - SWE-Bench Verified（500 题，人工 verify）
> - SWE-Bench Pro（1865 题，long-horizon，多文件）
> - LiveCodeBench / AiderBench
> - 自己出的中文 / 国产模型友好的 SWE benchmark（这是 DeepSeek 的话语权）
>
> **类二·内部 trajectory eval**（用于真实迭代）：
> - DeepSeek 内部 100 个真实任务（research / infra / 文档），人工 + LLM 双评打分
> - **关键**：分难度分层（easy/medium/hard）+ 分场景分层（debug / feature / refactor / docs）
> - 每次 release 跑一遍 regression
>
> **指标**：除了 Pass@1，还要看 **Cost-per-success**（每完成一个任务花多少 token）、**Steps-per-success**（多少 turn 完成）、**Intervention-per-task**（用户需要干预多少次）。
>
> **Anti-pattern**：只看 Pass@1 会让 agent 学会'挑容易的做'（reward hacking）；只看步数会让 agent '草率收工'。所以**必须组合指标**。"

---

## Q11：DeepSeek 模型和 Claude/GPT 的差异，对 Harness 设计有什么影响？

**考察意图**：测你是否真的在和模型 specifics 打交道。

**示范回答**（请按你的真实使用经验调整）：

> "我观察到几个有意思的差异：
>
> 1. **DeepSeek-V 系列**：tool_use 输出格式比 Claude 稳定度略低（偶有 schema drift），所以 harness 要做更宽容的 parser + retry policy
> 2. **DeepSeek-R 系列（推理）**：thinking trace 很长，对长 agent loop 上下文消耗大——harness 要慎用 reasoning，只在'必须 plan'的关头开
> 3. **instruction following**：DeepSeek 模型对 'system instruction must always X' 这种硬指令，遵循度稍弱于 Claude——这对 deny rule 实现是个挑战，需要 harness 自己兜底（schema validator / classifier）
> 4. **多语言**：中文 instruction 表现强于很多西方模型——这是中文用户场景的优势
>
> 我对 DeepSeek Harness 的建议：**不要假装自己是 Anthropic**——CC 那一套深度依赖 Claude 的 alignment 特性，DeepSeek Harness 要在 harness 侧做更多 robust 设计来兜底模型行为。这反过来也是和模型训练团队'共同进化'的入口——'我兜不住的，下一代你训出来'。"

---

## Q12：Sandbox 你怎么实现？

**示范回答**：

> "三层 sandbox：
>
> **L1·进程隔离**：用 OS-level 的 sandbox 库（macOS sandbox_init / Linux landlock / Windows AppContainer），限定 FS + 网络 + 进程能力。**这是 Claude Code 的方案。**
>
> **L2·容器隔离**：对高风险任务（执行陌生代码、跑测试套件）起一个 Docker / Firecracker microVM，跑完销毁。**这是 OpenClaw 的方案，更安全但启动慢。**
>
> **L3·远程隔离**：跑在云端 sandbox（OpenAI Codex CLI 模式），本地零执行，所有副作用都在云端。**最安全但要求网络好 + 用户接受。**
>
> **DeepSeek Harness v0.1 建议**：先做 L1（成本低、覆盖 90% 场景）；v0.2 加 L2（用 Docker 跑陌生 PR 的 verify）；v0.3 看用户需求再决定要不要 L3。
>
> **配套**：sandbox 失败要 fail-closed（默认拒绝，不是默认放行）；UI 要能让用户看到 sandbox 状态（避免'我以为有 sandbox 其实没有'）。"

---

## Q13：如果你要选 1 个最重要的 lifecycle hook 给 v0.1 开放，选哪个？

**示范回答**：

> "选 **PreToolUse**——理由有三：
>
> 1. **安全价值最大**：PreToolUse 能在 tool 执行前拦截、改 input、注入额外 context、要求审批；这是用户最容易构建自己 policy 的钩点
> 2. **可观测价值最大**：PreToolUse 能把所有 tool 调用 mirror 到外部日志/审计系统，对企业用户是 must-have
> 3. **扩展价值最大**：用户能用 PreToolUse 写自己的 lint / governance / cost tracking——这是 plugin 生态的起点
>
> 如果允许我选 2 个，我加 **PreCompact**——让用户能自定义 handoff 文件格式，长任务跑长链路的核心扩展点。
>
> Claude Code 用了 12 个 hooks，但 v0.1 不需要这么多——先 PreToolUse + PreCompact + Error 三个起步，根据用户需求渐进式加。"

---

## Q14：DeepSeek-R1 这种"显式推理"模型，对 Harness 有什么挑战？

**示范回答**：

> "三个挑战：
>
> 1. **Token 经济**：thinking block 占大量 token，对 KV cache 友好度比 non-reasoning 模型差——harness 需要在 thinking 不必要的场景**自动关掉**（比如简单 file read 没必要 think）
> 2. **延迟**：用户感知的 first-token latency 拉长，UI 必须给"模型在思考"的 affordance，否则用户以为卡死
> 3. **可解释性的双刃**：thinking 暴露了模型推理路径，对 debug 是金矿，对 trust 是双刃——用户看到模型"我想骗用户"会立刻爆炸（即使最终输出没做）
>
> **Harness 设计建议**：
> - 给一个**全局 reasoning budget 控制**，UI 上能切'快/平衡/深思'三档
> - thinking 默认折叠，用户主动展开
> - 长链路 agent 任务里 reasoning 只在 Planner agent 开，Generator / Evaluator 关掉以省 token
>
> 这也是 DeepSeek Harness 的差异化机会——R 系列是 DeepSeek 招牌，Harness 必须把 R 用好用对。"

---

## Q15：模型出"hallucinated tool args"怎么办？

**示范回答**：

> "Hallucinated tool args 有两类：
>
> **第一类·参数错（schema valid，semantic invalid）**：比如调 `delete_file('important.txt')` 但用户没让删——这是 alignment 问题，harness 兜底：
> - 高风险 tool 强制 ask（不能纯靠模型判断）
> - 把'最近用户意图摘要'注入 PreToolUse hook，让 hook 二次确认
> - 给模型看 'recent denied actions' 作为 negative few-shot
>
> **第二类·参数对、tool 不该 call**：典型 reward hacking——agent 学会'多 call tool 显示努力'。harness 兜底：
> - eval set 里加'什么都不做就是对'的样本，逼模型学会 abstain
> - 工具调用花了多少 token / 时间，归到任务总成本——和模型训练 reward 挂钩
> - StuckDetector 检测出'连续 5 次 ls 同一个目录'就 inject 'you are repeating, reconsider'
>
> 最根本的解法还是把这些 trajectory 反馈给训练团队——**Harness 不能修模型的 misalignment，但 Harness 能拿到证据让训练团队修。**"

---

## 附：技术口语化小练习

把下面 10 个词，每个用 1 句中文 + 1 个英文术语 解释（这是和研究员/英文社区对话时常用）。

| 词 | 1 句中文 | 1 个英文术语 |
| --- | --- | --- |
| Agent Loop | LLM + 工具 + 上下文的循环执行 | "an LLM in a loop with tools" |
| KV Cache | Transformer 推理时复用的 attention 缓存 | "prefix caching" |
| Tool Use | 让模型调外部能力的函数调用 | "function calling" |
| Reasoning | 模型在答案前的显式思考 | "chain-of-thought" |
| Planning | 把高层目标拆成子任务 | "task decomposition" |
| Skills | 可复用的 prompt + tool bundle | "agent skill / capability" |
| MCP | 模型 ↔ 工具的标准协议 | "Model Context Protocol" |
| Memory | 跨 session 持久信息 | "agent memory / persistent state" |
| Subagent | 独立上下文的子执行单元 | "spawned worker / sub-agent" |
| Multi-Agent | 多个 Agent 协作 | "multi-agent orchestration" |

练到能脱口而出，面试现场用一遍英文术语会让对面研究员立刻知道'这个人和我说同一种话'。
