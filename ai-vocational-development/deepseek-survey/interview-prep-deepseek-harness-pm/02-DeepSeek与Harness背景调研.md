# 02 · DeepSeek 与 Harness 背景调研

> 目标：在面试官提到 DeepSeek 任何近期动向、Harness 团队、对标产品、行业脉络时，你都能接得住、甚至带一句"insider 才知道"的细节。

---

## 一、DeepSeek 公司 60 秒速写（更新至 2026-05）

| 维度 | 信息 |
| --- | --- |
| **创始人** | 梁文锋（幻方量化创始人之一） |
| **总部** | 杭州 + 北京（Harness 团队在北京海淀融科） |
| **代表模型** | DeepSeek-V 系列（通用）、DeepSeek-R 系列（推理）、DeepSeek-Coder 系列（代码） |
| **开源策略** | 长期高强度开源（V2/V3 权重、Coder 权重、训练论文均公开），是中国"Anthropic 路线"的旗手 |
| **组织文化** | 不设 KPI、不赛马、极度扁平、150 人左右、技术理想主义、研究优先于商业化 |
| **招聘标准** | 顶尖高校、年轻（< 30）、竞赛金奖以上优先、工作 ≤ 5 年优先 |
| **面试风格** | 高强度（3 小时连面）、深挖项目、不看包装、压力面 |

> **应试用一句话浓缩**："DeepSeek 是中国唯一一支以 research lab 形态做 frontier model 的团队，文化最接近 Anthropic + 早期 OpenAI。"

---

## 二、Harness 团队是怎么出现的（时间线）

```
2024 Q4 ──► DeepSeek-V2.5 发布，coder 能力快速逼近 GPT-4o
2025 Q1 ──► 出圈：V3 + R1 推理模型发布，全球关注
2025 H1 ──► Anthropic Claude Code（2025-02 研究预览，05-22 GA）成为 coding agent 事实标杆
2025 H2 ──► Cursor 估值飙升，Anysphere 招"Product Manager, Agent Harness"
2025 Q4 ──► Anthropic 发 Effective harnesses for long-running agents（2025-11）
2026 Q1 ──► OpenAI Codex CLI、Google Gemini CLI、OpenCode 相继成熟
2026-03 ──► Anthropic 发 Harness design for long-running application development（三 Agent 架构）
2026-03 ──► DuoCode 发 Claude Code 源码级分析（51 万行 / 40 工具 / 101 命令）
2026-05-20 ► DeepSeek 资深研究员 陈德里 在社交媒体公开发帖："来 DeepSeek 从零做 Code Harness"
              同日：DeepSeek 官网上线 Agent Harness 产品经理 / 研发工程师两岗
              地点：北京海淀融科资讯中心
              对标：Anthropic Claude Code
2026-05-21 ► （你看到的 JD）
```

### 关键洞察

1. **DeepSeek 出手的时机非常晚**：Claude Code 已经 GA 一年；Cursor 估值已起；OpenCode/Codex/Gemini CLI 都已成型。**DeepSeek 这是"后发追赶"，要做的不是抄，而是用国产模型 + 开源 + 桌面端的组合做差异化。**
2. **核心信号是"桌面端"**：JD 明写"桌面端 Agent 产品"，这是 DeepSeek 第一次公开承认要做 client。意味着不再只做 API + 网页 Chat，而是直接打 Cursor / Claude Code 的主战场。
3. **陈德里是研究员牵头**：意味着这个团队是 **research-driven product**，PM 必须懂研究员。

---

## 三、对标对象速读：Claude Code / Cursor / Codex / OpenCode / Cowork / Hermes

> JD 里点名的所有产品都必须能说出 3 句话以上。下表是面试可以直接背的版本。

### 3.1 Claude Code (Anthropic)

| 维度 | 关键信息 |
| --- | --- |
| **形态** | 终端 TUI（Bun runtime，custom Ink fork，~60fps 渲染） |
| **核心理念** | "Model + Harness = Agent"——Anthropic 官方提法的发源 |
| **架构** | 5 层：entry points / state / query engine / tool & permission / coordinator |
| **工具数** | 40 个工具，101 个命令，20 个服务 |
| **典型 trick** | PreCompact handoff、context resets、bash classifier、sprint contract（三 Agent 模式） |
| **权限** | 工具自带 checkPermissions，多级模式（default / plan / acceptEdits / bypass / dontAsk / autoModeAcceptAll），deny 永远 > allow |
| **可借鉴的产品决策** | （1）skill 走平台注册，可审计；（2）hooks 直接嵌在 tool lifecycle；（3）UI 是终端但有完整 web 引擎渲染 |
| **痛点（用户视角）** | 终端 UI 不直观、价格贵、Skills 生态封闭、跨机器同步差 |

### 3.2 Cursor (Anysphere)

| 维度 | 关键信息 |
| --- | --- |
| **形态** | VSCode fork，IDE 形态 |
| **核心理念** | "AI-first IDE"，Tab 补全 + Agent Mode 双轨 |
| **典型 trick** | Tab 用本地小模型 / 短滑窗；Agent Mode 用 RAG + composer；codebase indexing 提前跑 |
| **PM 岗特点** | Anysphere PM JD 强调："PM 必须自己能 build & deploy"，是 Cursor 风格 |
| **痛点** | indexing 慢、跨仓库协同弱、Agent Mode 偶尔产生"幽灵编辑" |

### 3.3 OpenAI Codex CLI

| 维度 | 关键信息 |
| --- | --- |
| **形态** | CLI / Web / 远程沙箱 |
| **核心理念** | OpenAI 内部 Codex Agent 写了 100 万行代码（OpenAI 自报），all-in agent-first SWE |
| **典型 trick** | 双层安全（sandbox + policy）、本地 / 云端两种执行模式 |
| **痛点** | CLI 体验较 raw、文档不如 Claude Code 系统 |

### 3.4 OpenCode

| 维度 | 关键信息 |
| --- | --- |
| **形态** | 开源 CLI（最像 Claude Code 的开源版本） |
| **核心理念** | 让 Claude Code 风格走向开源、跨模型供应商 |
| **典型 trick** | Skills 走文件系统挂载（可移植，但缺审计） |
| **痛点** | 早期社区，质量参差；模型适配不一致 |

### 3.5 Gemini CLI (Google)

| 维度 | 关键信息 |
| --- | --- |
| **形态** | CLI |
| **核心理念** | Plan → Act → Validate 三段式 |
| **典型 trick** | Policy Engine（规则级权限）、validation 闭环 |
| **痛点** | Gemini 在长 agent 任务上 tool 调用不稳；UX 偏 Google 风格 |

### 3.6 OpenClaw / OpenHands (前 OpenDevin)

| 维度 | 关键信息 |
| --- | --- |
| **形态** | Web + Container，浏览器 + 终端 + 文件三视图 |
| **核心理念** | 通用 SWE Agent，强调"在真实环境里干活" |
| **典型 trick** | ACP（Agent Communication Protocol）、可视化 trajectory |
| **痛点** | 启动重，性能依赖容器，普通用户上手门槛高 |

### 3.7 Cowork / Hermes / Manus（多 Agent / 通用 Agent）

| 产品 | 关键信息 |
| --- | --- |
| **Cowork** | 国内多 Agent 协作产品（多 PM 协同 / 多角色） |
| **Hermes** | 通用桌面/工作流 Agent（按文档 / 计算机使用） |
| **Manus** | 通用 Agent 工具，以"自动跑活"出圈 |
| **共同特点** | 都试图用"多 Agent + 工作流可视化"打通用市场，但代码 SWE 场景下竞争不过 Claude Code 这种垂直 harness |

> **背诵口诀**：
> - **Claude Code = 终端 + 高度工程化 harness + 平台 skills**
> - **Cursor = IDE + 双轨（Tab/Agent）+ 强 codebase index**
> - **Codex = CLI + 双层安全 + 云沙箱**
> - **OpenCode = 开源 CC 风、跨模型**
> - **Gemini CLI = Plan/Act/Validate**
> - **OpenClaw = Web 容器 + 通用 SWE**
> - **Manus/Cowork/Hermes = 通用 / 多 Agent 路线**

---

## 四、Anthropic Harness 系列博客的关键观点（务必读熟）

> 这是和陈德里这种研究员背景 leader 对话的"共同语言"。

### 4.1 [Effective harnesses for long-running agents] (2025-11)
- **核心方法**：Initializer Agent + Coding Agent 两段式
- **关键问题**：长任务里 Agent 要么"太贪"，要么"过早宣布完工"
- **解法**：context reset（清空上下文重启）+ 结构化 handoff（用文件传 state）
- **可面试引用句**："长任务的核心不是更大上下文，而是更好的 handoff。"

### 4.2 [Harness design for long-running application development] (2026-03)
- **核心方法**：GAN 启发的三 Agent：Planner / Generator / Evaluator
- **关键问题**：模型自评偏正、Sonnet 4.5 出现"context anxiety"
- **解法**：分离评估者、Sprint Contract（实施前先谈"做完了长什么样"）
- **结果**：solo harness $9/20min vs full harness $200/6h，质量代差明显
- **可面试引用句**："Agent 不擅长自评，但很擅长配合一个外部评估者迭代——这就是为什么 Harness 设计的下半场是 evaluator design。"

### 4.3 [Making Claude Code more secure and autonomous with sandboxing] (2025-10)
- **核心方法**：FS + 网络隔离的 sandbox
- **目标**：减少 permission prompt 的同时不降安全性
- **可面试引用句**："不打断用户和不犯错之间的取舍，是 Harness 的核心 UX 设计问题。"

---

## 五、DeepSeek 可能的差异化路径（你的"产品判断"弹药）

DeepSeek 起步晚一年，靠"也做一个 Claude Code"是赢不了的。下面 5 条是合理的差异化假设，**你在面试中可以挑 1~2 条讲，证明你有自己的判断**。

### 路径 A：开源 + 国产模型生态
- 把 DeepSeek Harness 做成 **完全开源的 Claude Code 替代品**，支持插任意模型（DeepSeek 自家 + Qwen + Kimi + GLM + GPT-OSS）
- **关键差异化**：模型可换 = 用户不被 Anthropic 锁死
- **风险**：开源工程力消耗大，团队太小做不动

### 路径 B：与 DeepSeek 自家模型深度共生
- Harness 设计成"只对 DeepSeek 模型最优"的 reference harness——类似 Claude Code 之于 Claude
- 把 KV Cache、tool_use 输出格式、reasoning trace 都和模型训练对齐
- **关键差异化**：模型 + harness 紧耦合，跑分更高
- **风险**：用户被绑死在 DeepSeek 模型上

### 路径 C：桌面端 + 本地优先
- 桌面 GUI（不是终端 TUI），强本地化中文体验，深度集成中国开发者工具链（VSCode 中文版 / 通义灵码 / 飞书文档）
- **关键差异化**：中文优先、中国 IDE 习惯、文件级隐私
- **风险**：失去全球开发者社区

### 路径 D：内部"狗粮"驱动的训练数据飞轮
- 把 DeepSeek 自己 200 人左右研究员/工程师的真实编码任务做成训练飞轮
- 每个 trace 直接进 RLHF/RFT pipeline
- **关键差异化**：data flywheel，飞轮一旦转起来对手追不上
- **风险**：内部数据量小，需要外部用户量级支撑

### 路径 E：Subagent + Skills 开放标准
- 把 Subagent / Skills 做成跨工具开放标准（参考 OpenCode Skills 文件系统挂载）
- 任何 Agent 工具都能跑同一份 Skill
- **关键差异化**：生态破局点，吃 ecosystem play
- **风险**：标准之争，需要联合战友

> **应试技巧**：当被问"你觉得 DeepSeek Harness 应该走什么路"，**别选一个**——选两个组合：例如 A + D（开源 + 内部飞轮），并说为什么不选 B（避免和 Anthropic 同质化）。这样显示你有 trade-off 思维。

---

## 六、Harness 团队组织结构推测（用于聊到"你想入职哪条线"）

```
                陈德里（Harness 团队 lead，研究员背景）
                        │
        ┌───────────────┼───────────────┐
        │               │               │
   Harness PM       Harness 研发       与模型团队的接口
    （你应聘的）      （工程师/研究员）   （研究员 ↔ harness 接口）
        │               │               │
   ┌────┴───┐      ┌────┴────┐     ┌────┴────┐
   产品定义      Agent Loop /    数据飞轮 /
   指标体系      Tool / Skills /  Eval Set /
   用户社群      MCP / 沙箱       Reward 信号
   路线图        渲染 / 状态      训练接口
```

> **面试可以说**："如果团队结构如我所推测，PM 这个角色更像 Anthropic Labs 的 PM——同时做产品定义、用户研究、和评估工程的桥梁。"

---

## 七、可能出现的"开放性产品题"提前预热

下面这些题型，你需要在 [文档 04~07] 里准备扎实答案。这里只列题，便于你建立全局感。

1. **桌面端 vs 终端**：DeepSeek Harness 应该走 GUI 还是 TUI？为什么？
2. **付费 vs 开源**：完全开源 vs 部分开源 vs SaaS 订阅，怎么选？
3. **模型解耦 vs 紧耦合**：能否跨模型？代价是什么？
4. **指标设计**：怎么衡量"Agent 真的在更多场景帮到更多人"？
5. **冷启动**：如果你只有 100 个种子用户（公司内部 + 邀请制），第一个 30 天 roadmap 是什么？
6. **vs Claude Code**：3 个月内 DeepSeek Harness 怎么和 Claude Code 形成第一个差异化卖点？
7. **数据飞轮**：用户用 Harness 干活产生的 trajectory，怎么进训练？合规边界在哪？
8. **Subagent 设计**：默认开还是按需开？深度限制几层？
9. **失败模式**：DeepSeek Harness 12 个月后失败，最可能的失败原因是？
10. **Skills 开放性**：Skills 走 Anthropic 风格的平台注册，还是 OpenCode 风格的 FS 挂载？

> 这 10 题里，**至少要能脱口而出 3 题的完整答案**，剩下能在 30 秒内搭出框架。

---

## 八、信源与可继续深挖的链接

> 面试前把这些都点开看一遍。

- DeepSeek 招聘公告与陈德里发帖：新浪科技 2026-05-20、网易科技、AIWW、Digg
- Anthropic Harness 系列：
  - https://www.anthropic.com/engineering/effective-harnesses-for-long-running-agents
  - https://www.anthropic.com/engineering/harness-design-long-running-apps
  - https://www.anthropic.com/engineering/claude-code-sandboxing
- Claude Code 源码级深度分析：DuoCode 2026-03-31
- 同级目录已有的 [《主流 AI Agent Harness Engineering 深度分析》](../ai-agent-harness-engineering-analysis.md)
- DeepSeek 公司文化：搜狐《DeepSeek 的人才管理精髓》、36 氪长文、腾讯云开发者《DeepSeek 线上面试》
- Anysphere/Cursor PM JD：https://cursor.com/careers/product-manager

---

## 九、一句应试钥匙

> **当面试官提到 DeepSeek 或行业动向，永远多带一个"具体细节 + 你的判断"组合，而不是"我也看到这个新闻"。**

例：
- ❌ "我看到 DeepSeek 在招 Harness 团队。"
- ✅ "我注意到这个团队 5 月 20 号才公开组建，陈德里在帖子里强调'对标 Claude Code'。结合 Anthropic 11 月那篇 long-running harness、3 月那篇三 Agent harness，DeepSeek 走研究员牵头 product 这条路，对 PM 来说意味着 evaluator 设计可能比 feature 设计更重要——我专门为这个角色准备了一份指标体系草案，能否聊一下？"
