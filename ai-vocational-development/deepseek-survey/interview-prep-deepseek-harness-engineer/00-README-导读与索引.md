# DeepSeek「Agent Harness 研发工程师」面试准备包

> **岗位**：Agent Harness 研发工程师（DeepSeek，桌面端 Agent 产品）
> **团队**：Harness 团队（陈德里牵头，对标 Anthropic Claude Code）
> **地点**：北京海淀区融科资讯中心
> **本包整理时间**：2026-05-21
> **目标**：把这份材料当作"面试前 7 天集中冲刺手册"——覆盖从理解 JD、行业调研、技术弹药、手撕代码 / 系统设计、临场策略到复盘跟进的全流程。

---

## 一、和 PM 版的关系

> 同一个 Harness 团队（陈德里牵头）、同一个产品（DeepSeek Code Harness）、同一批面试官——但 PM 和工程师两个岗位的**考察方式天差地别**。

| 维度 | PM 版 | **工程师版（本包）** |
| --- | --- | --- |
| 主要考察 | 产品判断 / 指标体系 / 用户研究 / Trade-off | **手撕代码 / 系统设计 / 源码理解 / LLM 技术深问** |
| 核心动作 | 写产品提案 + 用 Agent | **写 minimal harness + 改 Agent 源码** |
| 演练焦点 | 1 套 STAR + 1 份提案 PDF | **LeetCode + 系统设计 + 开源贡献 + 技术博客** |
| 面试时长 | 偏短 + 多轮 case | **可能 3 小时长面（DeepSeek 风格压力面）** |

> **背景与公司调研部分高度重合**——本包中相关章节直接引用 PM 版的 [02-DeepSeek与Harness背景调研.md](../interview-prep-deepseek-harness-pm/02-DeepSeek与Harness背景调研.md)，避免重复。
>
> **建议**：如果你不确定自己投 PM 还是工程师，先读 [99-PM与工程师两岗差异对照与选择建议.md](99-PM与工程师两岗差异对照与选择建议.md)。

---

## 二、目录与使用建议

| 序号 | 文件 | 内容 | 何时用 |
| --- | --- | --- | --- |
| 01 | [01-JD深度解读与岗位画像.md](01-JD深度解读与岗位画像.md) | 逐句拆解 JD、能力雷达、隐藏信号、对 PM 版的差异 | T-7 ~ T-5 |
| 02 | [02-DeepSeek与Harness背景调研-引用与工程师补充.md](02-DeepSeek与Harness背景调研-引用与工程师补充.md) | 引用 PM 版背景 + 工程师视角补充（Claude Code 源码、技术栈对比） | T-7 ~ T-4 |
| 03 | [03-Harness技术体系与代码模式速查.md](03-Harness技术体系与代码模式速查.md) | 14 个核心概念的代码模式、调用链、源码片段——工程师深度 | T-5 ~ T-2 |
| 04 | [04-高频面试题-编码与算法篇.md](04-高频面试题-编码与算法篇.md) | 手撕题：Mini Agent Loop / Tool Dispatcher / Streaming Parser / KV-cache 友好 prompt 设计 + LeetCode 重点 | T-4 ~ T-2 |
| 05 | [05-高频面试题-LLM与Agent原理深技术篇.md](05-高频面试题-LLM与Agent原理深技术篇.md) | KV Cache 细节、tool_use 格式、SSE 流式、context window 管理、reasoning model 推理路径 | T-3 ~ T-2 |
| 06 | [06-高频面试题-系统设计与Harness组件篇.md](06-高频面试题-系统设计与Harness组件篇.md) | 设计 Sandbox / Permission Engine / Multi-Agent Coordinator / Eval Pipeline | T-3 ~ T-2 |
| 07 | [07-高频面试题-行为与研究员协作篇.md](07-高频面试题-行为与研究员协作篇.md) | STAR + 跨界（和研究员协作 / 开源贡献 / 压力面 / DeepSeek 文化匹配） | T-3 |
| 08 | [08-反问环节话术库.md](08-反问环节话术库.md) | 工程师向的高质量反问（HR / 工程师 / 研究员 / Leader） | T-2 |
| 09 | [09-面试前7天行动清单.md](09-面试前7天行动清单.md) | T-7 ~ T-0 每天的"今天该做什么" + LeetCode/Design 演练节奏 | T-7 启动 |
| 10 | [10-面试中临场策略与话术.md](10-面试中临场策略与话术.md) | 白板编码 think aloud、系统设计 4 阶段、代码失误挽回 | T-1 复读 |
| 11 | [11-面试后复盘模板与跟进话术.md](11-面试后复盘模板与跟进话术.md) | 工程师特化复盘表 + 感谢信 + 跟进话术 | T+0 ~ T+7 |
| 12 | [12-自带作品集-Minimal-Harness开源项目设计.md](12-自带作品集-Minimal-Harness开源项目设计.md) | 一份 < 500 行的 minimal harness 开源项目设计 + 仓库 README 模板——**面试杀手锏** | T-7 ~ T-2 反复迭代 |
| 13 | [13-自带作品集-技术深度博客与开源贡献选题.md](13-自带作品集-技术深度博客与开源贡献选题.md) | 一篇技术博客选题 + 10 个开源 contribution 候选 issue | T-5 ~ T-2 |
| 99 | [99-PM与工程师两岗差异对照与选择建议.md](99-PM与工程师两岗差异对照与选择建议.md) | 两个岗位的对照 + 自我评估建议 | 投递前 |

---

## 三、整体策略（一句话总结）

> **"用工程师的硬功夫，承接研究员的前沿创新——证明你不是只会用 Agent 的 'AI 应用工程师'，是能从 0 设计 Harness 的 'AI 系统工程师'。"**

具体落地三件事：

### 1. 一份能跑的 Minimal Harness（最关键的杀手锏）
DeepSeek Harness 工程师候选人最稀缺的不是 LeetCode 能力，是 **"你真的自己写过 agent loop / tool dispatcher / sandbox 吗"**。

如果你能在 T-7 ~ T-2 这一周里把 [12 文档] 那个 minimal harness 跑出来、push 到 GitHub、写一份 README，**这一件事就足以让你和 80% 候选人区分开**。

### 2. 5 个 Harness 关键源码片段烂熟
从 Claude Code、Cursor、OpenCode、Codex CLI 中挑 5 个最关键的代码片段（agent loop、tool dispatcher、context compaction、sandbox、permission engine），**能用自己的话讲清"为什么这么设计、有什么 trade-off、如果让我设计会怎么改"**。

### 3. 准备和研究员 1:1 聊得起来
DeepSeek 不是字节腾讯。技术面里至少 1 轮会有研究员（陈德里就是研究员），他/她会问你：
- "我训出 deepseek-coder-V3.5，你怎么把它的新能力第一时间放进 harness？"
- "我希望 harness 在 trajectory 上喂回的反馈是什么格式？"
- "你怎么看 reasoning trace 在长 agent loop 里的 cost？"

这些不是技术题——是 **"你能不能成为研究员的 partner"** 的测试。本包文档 05 + 07 专门覆盖。

---

## 四、面试评估维度的内部模型（推测）

按公开信息推断，面试官（陈德里 + 团队工程师 + HR）会从 **6 个维度** 给你打分：

| 维度 | 权重（推测） | 关键问题 | 怎么过 |
| --- | --- | --- | --- |
| **编码硬功夫** | 25% | "白板写一个 ReAct loop，支持并发 tool" | 文档 04 + 12 reps to自动手感 |
| **系统设计** | 20% | "设计一个支持 100K QPS 的 LLM 推理网关 / Agent sandbox 服务" | 文档 06 |
| **LLM/Agent 技术深度** | 20% | "KV cache 在多轮 tool use 里怎么工作？" | 文档 03 + 05 |
| **学习能力 / 适应力** | 10% | "我给你一个你没用过的语言 / 框架，1 小时做出 X" | 真实历史 + 当场表现 |
| **研究员协作 / 跨界品味** | 15% | "模型行为有哪些细节你印象深的？" | 文档 05 + 07 |
| **0→1 执行力与抗压** | 10% | "3 小时连面 / 改方向 / 接受批评" | DeepSeek 风格 + 心态 |

---

## 五、推荐节奏（T-7 ~ T+0）

```
T-7 ──► 战略：读 JD / 公司 / 团队，确定方向；启动 Minimal Harness 开发
T-6 ──► 技术弹药：14 个核心概念速查 + 啃 Anthropic harness 博客
T-5 ──► Minimal Harness 跑起来 V0；做 LeetCode warm-up（5 题）
T-4 ──► 编码题专项：手撕 Agent Loop / Tool Dispatcher（4 篇文档 04）
T-3 ──► 系统设计专项（文档 06）；mock interview 1 次
T-2 ──► 反问 + 简历 + GitHub README 精修；技术博客或开源贡献开发
T-1 ──► 静心日，过 10 + 09；不再加新内容
T-0 ──► 面试当天
T+0 ──► 24h 内复盘 + 感谢邮件（含 V0.2 link）
```

总投入 **40~55 小时**——可以分配在 7 天里，每天 6~8 小时。

---

## 六、参考资料锚点

主要素材来源（贯穿全包）：

1. **DeepSeek Harness 团队公开报道**（2026-05-20）
2. **Anthropic 官方 Harness 系列博客**：
   - [Effective harnesses for long-running agents](https://www.anthropic.com/engineering/effective-harnesses-for-long-running-agents)（2025-11）
   - [Harness design for long-running application development](https://www.anthropic.com/engineering/harness-design-long-running-apps)（2026-03-24）
   - [Making Claude Code more secure and autonomous with sandboxing](https://www.anthropic.com/engineering/claude-code-sandboxing)（2025-10）
3. **Claude Code 源码级分析**（DuoCode Tech Blog，2026-03-31）
4. **同级目录已有的深度分析**：[../ai-agent-harness-engineering-analysis.md](../ai-agent-harness-engineering-analysis.md)
5. **DeepSeek 公司文化与面试经验**（搜狐 / 36氪 / 腾讯云开发者）
6. **同级 PM 版准备包**：[../interview-prep-deepseek-harness-pm/](../interview-prep-deepseek-harness-pm/)（背景与公司部分可直接读用）
7. **Anthropic 系统设计面试公开资料**（Exponent / SystemDesignHandbook）
8. **Open source minimal agent loop 参考实现**：liteagent / simple-agent-loop / tinyAgent-TS / awesome-agent-sdk

---

## 七、一句忠告

> **DeepSeek Harness 团队现在最缺的不是"会写 SaaS CRUD 的资深工程师"，是"能看 Anthropic 博客一眼就知道哪里能复刻、哪里要重做、哪里能比它做得更好"的 Agent-native engineer。**

**这种 engineer 的简历不是看年限，是看你最近 12 个月写过什么 agent 代码。**

> 如果你的 GitHub 上能让面试官在 5 分钟内点开一个 minimal harness repo，读完之后说"嗯，这人会写"——offer 基本就到了。

祝顺利。
