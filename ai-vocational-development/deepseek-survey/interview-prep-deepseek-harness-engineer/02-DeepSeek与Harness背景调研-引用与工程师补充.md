# 02 · DeepSeek 与 Harness 背景调研（工程师视角补充）

> **注意**：DeepSeek 公司背景、Harness 团队时间线、对标产品对比 已在 PM 版包里写过完整版，本文档**只做引用 + 工程师视角补充**，避免重复。

---

## 一、必读的 PM 版章节

请先打开并读完：

> **[../interview-prep-deepseek-harness-pm/02-DeepSeek与Harness背景调研.md](../interview-prep-deepseek-harness-pm/02-DeepSeek与Harness背景调研.md)**

它已经覆盖了：
- DeepSeek 公司 60 秒速写
- Harness 团队组建时间线（2026-05-20 公开）
- 对标产品速读（Claude Code / Cursor / Codex / OpenCode / Gemini CLI / OpenClaw / Manus / Cowork / Hermes）
- Anthropic Harness 三篇博客的关键观点
- DeepSeek 可能的差异化路径（5 条）
- 团队组织结构推测

工程师面试同样需要知道这些。**先把那份读完再回来。**

---

## 二、工程师视角的补充——技术栈与源码层面的"insider 信号"

> 下面这些是 PM 版没展开、但**工程师面试可能被深问**的话题。

### 2.1 Claude Code 源码层的关键事实（背诵卡）

来自 DuoCode 2026-03-31 源码级分析：

| 维度 | 关键数字 / 事实 |
| --- | --- |
| 代码量 | ~513,237 行 TypeScript / 1,902 文件 |
| 工具数 | 40 个 tool / 101 个命令 / 20 个服务模块 / 8 个 task 执行类型 |
| **Runtime** | **Bun**（不是 Node）；用 `bun:bundle` feature gates 做 dead code elimination |
| UI 渲染 | Custom Fork of Ink（不是标准 Ink）+ React Reconciler + Yoga WASM layout + 60fps frame scheduler |
| 终端渲染 | DECSTBM 硬件滚动 + ScrollBox + viewport culling + CharPool / StylePool 字符串 interning |
| 最大文件 | REPL.tsx 896KB（含全屏 layout / VirtualMessageList / PromptInput / modal） |
| Query Engine | 单 QueryEngine 实例/会话，自带 maxTurns / maxBudgetUsd / fallbackModel / **diminishing-returns detector**（连续 3 次 <500 token 增量就停） |
| Tool 系统 | TOOL_DEFAULTS 默认**permissive**；安全靠 tool-specific override（与传统 deny-by-default 相反） |
| Bash 安全 | AST 解析 + bashClassifier（speculative 2s 超时）+ 23 类违规分类 + 18 个 zsh 危险命令黑名单 |
| 权限规则 | **deny rule 永远 > allow**（hook 也无法绕过） |
| Tool 编排 | toolOrchestration.ts: 按 concurrency-safety 分批；默认 max 10 并发；env var `CLAUDE_CODE_MAX_TOOL_USE_CONCURRENCY` |
| 上下文管理 | 三层 pipeline：getUserContext / getSystemContext / fetchSystemPromptParts；都 memoized |
| Compaction | 4 阶段：snip → micro-compact → context collapse → auto-compact trigger |
| Memory | MEMORY.md capped 200 行 / 25KB |
| ~90% 代码 | Anthropic 创始工程师自报：Claude Code 约 90% 代码是 Claude 自己写的（git blame attribution）|

**面试官如果对源码很熟，能背出 3~5 个具体数字就证明你真的看过**。

### 2.2 主流 Harness 的技术栈横向对比

| 产品 | 主语言 | Runtime | UI 框架 | Schema | Sandbox | 并发模型 |
| --- | --- | --- | --- | --- | --- | --- |
| **Claude Code** | TypeScript | **Bun** | Custom Ink fork + Yoga | Zod | OS-level sandbox_init / landlock | Promise.all + concurrency-safe partition |
| **Cursor** | TypeScript | Node (VSCode) | React + Electron / Native | Zod-ish | 较弱（GUI 自动确认多） | Worker / Extension Host |
| **Codex CLI** | TypeScript | Node | Ink | — | Sandbox + Policy + Cloud | Async / queue |
| **OpenCode** | Go / Bun | Bun | Ink fork | — | OS-level | Goroutines / async |
| **Gemini CLI** | TypeScript | Node | Ink | — | Policy Engine | Async |
| **OpenClaw / OpenHands** | Python | CPython | Web (React) + Browser + Container | Pydantic | Docker / Firecracker | Asyncio |
| **DeepSeek Harness（推测）** | **TBD（TS or Rust 可能性最高）** | TBD | TBD | TBD | TBD | TBD |

**关键观察**：
- **Node ⇒ Bun ⇒ Rust** 是 CLI Harness 这一年的迁移趋势（启动速度 + dead-code elimination + 内存）
- **Zod 是 TS 阵营 schema 事实标准**，Pydantic 是 Python 阵营事实标准
- **OS-level sandbox**（不是 Docker）正在成为主流——启动快、用户体感无感

### 2.3 工程师视角的"DeepSeek Harness 可能技术选型"猜想（用于聊技术选型题）

| 维度 | 候选 | 我的猜想 | 理由 |
| --- | --- | --- | --- |
| 语言 | TypeScript / Rust / Go | **TS 优先** | 工程师人才池 + UI 渲染 + 跨平台 |
| Runtime | Bun / Node / Deno | **Bun 优先** | 跟随 Claude Code 已验证 |
| 模型接入 | DeepSeek 原生 / OpenAI-compat / 自定义 | **OpenAI-compat 先行**（兼容 DeepSeek 自家 API）| 生态友好 |
| Tool schema | Zod / JSON Schema 直写 | **Zod** | TS 阵营标配 |
| Sandbox | OS-level / Docker / WASM | **OS-level + 可选 Docker** | 启动速度优先 |
| MCP | 必有 / 不做 | **必有**（兼容生态） | 没有 MCP 等于自我孤立 |
| UI | TUI / GUI / Both | **TUI 先 + 未来 GUI** | 跟随 CC，但 GUI 是差异化牌 |
| 持久化 | SQLite / JSON / DuckDB | **SQLite** | 嵌入式 + 跨平台 + 老牌 |

**如果面试官问你"如果你来做技术选型"**，照上表+你的理由答，并主动列出 trade-off。

### 2.4 必读的 Anthropic 工程博客（不只是 harness 系列）

PM 版只列了 harness 三篇。工程师还要读：

1. **[Context engineering for agents](https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents)** —— context window 工程化方法
2. **[Building agents with Claude Agent SDK](https://www.anthropic.com/engineering/building-effective-agents)** —— SDK 设计思路（可直接对照 DeepSeek Harness 怎么设计）
3. **[Claude Code sandboxing](https://www.anthropic.com/engineering/claude-code-sandboxing)** —— sandbox 工程细节
4. **[Prompt caching](https://www.anthropic.com/news/prompt-caching)** —— KV cache 商业化
5. **[Voyager / Skill library](https://arxiv.org/abs/2305.16291)** —— skill 概念来源
6. **[OpenAI o1 reasoning](https://openai.com/o1)** —— reasoning model 的接口设计含义

工程师面试时引用一篇能让面试官立刻知道你读过原文。

---

## 三、工程师视角的"DeepSeek Harness 工程挑战"清单

> 这是你聊技术架构时可以**主动 surface 出来的工程难题**——证明你不只是"听过名词"，而是想过怎么落地。

| 工程挑战 | 难点 | 我的初步思路 |
| --- | --- | --- |
| **跨平台 sandbox** | macOS sandbox_init / Linux landlock / Windows AppContainer 三套 API 完全不同 | 抽象 `Sandbox` 接口 + 三个适配器；首选用 [bubblewrap](https://github.com/containers/bubblewrap)（Linux）+ macOS 原生 + Windows AppContainer |
| **Tool 并发安全** | 不是所有 tool 可并发（write 同一文件 race）；如何静态/动态判断 | concurrency-safe boolean + 分批调度（Claude Code 方案）；动态：基于 affected resource 推断 |
| **KV cache 友好的 prompt 装配** | 动态拼接 + 末尾追加的同时不破坏 prefix cache | section-based prompt + cache_control breakpoint 标注 + 末尾追加 tool_result |
| **流式输出 + interruptibility** | SSE / WebSocket 双向 + 实时 abort | AbortController + 在每个 await point 检查 signal |
| **长任务 context handoff** | compact / reset / fork 三选一的工程实现 | file-based handoff + resumable state |
| **bash 命令安全分类** | AST 解析 + classifier，2 秒超时降级 | tree-sitter-bash + LLM-as-classifier + timeout pattern |
| **跨模型 tool_use 适配** | OpenAI / Anthropic / Gemini 三套 tool_use 协议 schema 不同 | adapter pattern + 统一中间表示 + format converter |
| **MCP server lifecycle** | 子进程崩溃 / 协议版本协商 / stdin/stdout 协议解析 | child process supervisor + reconnect logic + schema versioning |
| **TUI 高性能渲染** | 60fps + 不闪 + 大文本滚动 | virtual list + diff render（Ink 已解决）|
| **telemetry 合规** | opt-in + 脱敏 + 跨境 + 用户可视审计 | local-first + 显式 export + 数据出境分类 |

**面试中怎么用**：被问"你怎么看 DeepSeek Harness 最难的工程挑战"时，从上表挑 3 个，给出**初步思路 + trade-off + 你想跟团队对齐的边界**。

---

## 四、可能的"看源码"考题预演

DeepSeek 是 research-driven 文化，研究员风格面试官可能给你一段开源代码（Claude Code / OpenCode / liteagent / aider）让你点评。**你需要能在 3 分钟内说出：**

1. **这段代码做什么**
2. **为什么这么设计**（推测意图）
3. **它的 trade-off / 隐患**
4. **如果让你改你会怎么改**

> **练习方式**：T-5 ~ T-3 每天打开一个开源 harness 的 1 个核心文件（如 OpenCode 的 agent loop、liteagent 的 ToolResult、Claude Code 反编译的 QueryEngine），用上面四步自问自答。

---

## 五、一句应试钥匙（工程师特化）

> **当面试官提到 DeepSeek 或行业，"我看到这个新闻"是 PM 答法。工程师答法是 "我看了一下 Claude Code 在这块的源码 / 我自己实现过这个 / 这是我的开源 repo——可以一起 review。"**

例：
- ❌ "我知道 Claude Code 用 Bun。"
- ✅ "我注意到 Claude Code 用 Bun 而不是 Node，主要是为了 build-time feature gates（COORDINATOR_MODE / BASH_CLASSIFIER）和 startup latency。我自己在 minimal harness 里也试过 Bun，cold start ~30ms 比 Node ~150ms 快 5 倍。但 Bun 的 ecosystem 还有些库不支持，DeepSeek 如果选 Bun 要做兼容性测试 list。"

这种程度的回答会让工程师面试官立刻把你归入"会写"那个候选池。
