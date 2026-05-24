# DeepSeek「C 端产品经理」面试准备包

> **岗位**：DeepSeek C 端产品经理（通用 AI Chat / Assistant 类 C 端产品方向）
> **本包整理时间**：2026-05-22
> **目标**：把这份材料当作"面试前 7~14 天集中冲刺手册"——覆盖从理解 JD、AI Chat 赛道竞品、AI 产品方法论、用户洞察、增长指标、A/B test、跨团队协作、临场、复盘到 follow-up 全流程。

---

## 一、这个岗位的"定位"——和已有四包的关系

> DeepSeek Agent / Harness / 模型研究院体系下目前已知的 5 个岗位（同一公司不同 BU/team）：
>
> | 岗位 | 定位 | 准备包 |
> | --- | --- | --- |
> | Harness **产品经理** | 定义 Code Agent **桌面产品** + 路线图 | `../interview-prep-deepseek-harness-pm/` |
> | Harness **研发工程师** | 造 Code Agent 桌面产品 | `../interview-prep-deepseek-harness-engineer/` |
> | **Agent 全栈工程师** | 造给算法用的评测平台 + 容器 + Scaffold | `../interview-prep-deepseek-agent-fullstack/` |
> | **Agent 算法研究员** | 训模型本身——RL、对齐、reasoning | `../interview-prep-deepseek-agent-researcher/` |
> | **C 端产品经理**（本岗）| 定义 **DeepSeek 通用 C 端 AI Chat 产品** | **本目录** |

**核心差异（一句话）**：
- Harness PM = "**给程序员造 Code Agent 桌面产品**"（工具型，To Developer，高技术深度）
- **C 端 PM = "给亿级普通用户造 AI 助手产品"**（消费型，To Consumer，强用户增长）

> **本岗的本质是 "ChatGPT/豆包/Kimi"型 C 端 PM**——你的产品是 deepseek.com 网页版 + DeepSeek App + 移动端，竞争对手是国内豆包/Kimi/腾讯元宝/通义/智谱清言/MiniMax 海螺，海外 ChatGPT/Claude/Gemini/Perplexity。
>
> **你的核心战场**：DAU/MAU、留存、Day-N 留存、新功能转化、对话深度、付费转化（如有）。

详细的五岗对照：[../99-五岗位差异对照与选择建议.md](../99-五岗位差异对照与选择建议.md)（5 岗版统一更新）

---

## 二、JD 关键能力一句话总结

> **"极强用户共情 + AI 产品深度用户 + 复杂项目落地 + 大模型理解 + 数据驱动决策 + 跨团队协作。"**

加上**必须**懂：C 端产品方法论（PMF / 增长 / 漏斗 / 留存）+ AI 能力边界（Context、Token、幻觉、reasoning 边界）+ 数据分析（A/B test / 指标拆解）+ 竞品分析（国内外 6+ 款 AI Chat 主流产品）。

---

## 三、目录与使用建议

| 序号 | 文件 | 用途 |
| --- | --- | --- |
| 01 | [01-JD深度解读与岗位画像.md](01-JD深度解读与岗位画像.md) | 逐句拆解 + 隐藏信号 + 与 Harness PM 对照 |
| 02 | [02-DeepSeek-C端产品与AI-Chat赛道调研.md](02-DeepSeek-C端产品与AI-Chat赛道调研.md) | DeepSeek 网页/App 现状 + 国内外 8 款竞品速览 + 行业关键数据 |
| 03 | [03-AI产品方法论速查.md](03-AI产品方法论速查.md) | PMF / 增长 / AI UX / Prompt UI / 数据飞轮 |
| 04 | [04-竞品深度分析.md](04-竞品深度分析.md) | ChatGPT/Claude/Gemini/豆包/Kimi/元宝/智谱/海螺 深度拆 |
| 05 | [05-高频面试题-用户洞察与产品判断篇.md](05-高频面试题-用户洞察与产品判断篇.md) | 12 道用户/共情/判断题 |
| 06 | [06-高频面试题-AI能力转化与功能设计篇.md](06-高频面试题-AI能力转化与功能设计篇.md) | 10 道 AI 能力如何变功能 |
| 07 | [07-高频面试题-增长与指标体系篇.md](07-高频面试题-增长与指标体系篇.md) | 增长 / 北极星指标 / 留存设计 |
| 08 | [08-高频面试题-AB-test与数据分析篇.md](08-高频面试题-AB-test与数据分析篇.md) | A/B 设计 / 显著性 / 数据陷阱 |
| 09 | [09-高频面试题-跨团队协作与行为篇.md](09-高频面试题-跨团队协作与行为篇.md) | STAR-C 行为题 + 算法/研究员协作 |
| 10 | [10-反问环节话术库.md](10-反问环节话术库.md) | C 端 PM 专属反问 |
| 11 | [11-面试前-中-后全流程行动手册.md](11-面试前-中-后全流程行动手册.md) | T-7 ~ T+0 + 临场策略 + 复盘 |
| 12 | [12-自带作品集-DeepSeek-C端改进提案+PRD样例.md](12-自带作品集-DeepSeek-C端改进提案+PRD样例.md) | 杀手锏：DeepSeek 现版 deep critique + 3 个改进提案 + 1 个完整 PRD |

---

## 四、整体策略（一句话）

> **"用 AI 重度用户视角 + DeepSeek 现版 deep critique + 增长方法论 + 1 个可 ship 的 PRD——让面试官在 5 分钟内意识到我比一般 PM 更懂 AI、比一般 AI 研究员更懂用户。"**

具体落地三件事：

### 1. 一份 DeepSeek 现版 deep critique（**最关键的杀手锏**）

C 端 PM 面试最稀缺的不是 "懂方法论"——是 "**你真的天天在用产品、有真实 friction 点、有可执行改进方案**"。

如果你能在 T-7 之前完成 [12 文档] 提到的：
- 连续 7 天深度使用 DeepSeek 网页 + App
- 列出 20 个 specific friction 点（截图 + 描述）
- 凝练成 3 个改进提案（含优先级 + 影响估算）
- 把其中 1 个写成完整 PRD

**这一件事就足以让你和 80% 候选人区分开**——简历上写"懂用户体验" 是 zero proof；带一份 30 页 critique deck 是 ten proof。

### 2. 竞品矩阵 + 行业数据烂熟

C 端 PM 一定被问 "你怎么看竞品 X"。准备 8 款主流 AI Chat（4 海外 + 4 国内）每款能讲：定位 / 核心差异 / 数据 / 我的体验 / 启发 DeepSeek 的点。[04 文档] 全部。

### 3. 数据驱动话术库

每个答案都带数字——"我观察 [产品] 的 Day-7 留存约 X%"、"我会设计 A/B 检测 N=1k 样本可达 95% 显著"——这种 calibrated 表达瞬间拉开档次。

---

## 五、面试评估维度的内部模型（推测）

| 维度 | 权重 | 关键问题 |
| --- | --- | --- |
| **用户洞察 / 共情** | 25% | "你最近发现哪个 AI 产品体验痛点？" / "DeepSeek 体验哪里最差？" |
| **AI 能力边界理解** | 20% | "什么任务 LLM 干不了？" / "幻觉怎么处理？" |
| **产品判断 / 决策** | 20% | "假如 LLM 多 modal 突破，DeepSeek 该不该追？" |
| **数据驱动 / 增长** | 15% | "你怎么定北极星？" / "Day-7 留存掉了 5%，怎么 debug？" |
| **AB / 数据分析** | 10% | "你怎么设计 A/B？" "样本量怎么算？" |
| **跨团队协作 / 文化** | 10% | "和算法分歧怎么办？" / "为什么 DeepSeek" |

---

## 六、推荐节奏（T-7 ~ T+0）

> C 端 PM 比研究员准备时间短，因为不需要硬啃数学；但 DeepSeek critique + 竞品分析需要"真用"——这个不能跳过。

```
T-7  战略 + 启动 DeepSeek 7 天深度使用 + 列 friction 点
T-6  竞品深度体验：海外 4 款 (ChatGPT/Claude/Gemini/Perplexity)
T-5  竞品深度体验：国内 4 款 (豆包/Kimi/元宝/智谱清言/海螺挑 4)
T-4  方法论速查 + 用户洞察题
T-3  AI 能力转化 + 功能设计题 + 增长指标
T-2  A/B 数据题 + 行为题 + 反问准备
T-1  PRD 终稿 + 简历 + 静心
T-0  面试当天
T+0  24h 复盘 + 感谢邮件 + 后续
```

总投入 **30~60 小时**。比研究员包少（无数学），但**竞品和 DeepSeek 体验时间不能压缩**——这是 C 端 PM 的"硬通货"。

---

## 七、和已有四包的引用约定

为避免重复，本包**直接引用**已有四包的相关章节：

| 内容 | 直接引用 |
| --- | --- |
| DeepSeek 公司背景 / 文化 / 不发奖金 | [Harness PM 包 02](../interview-prep-deepseek-harness-pm/02-DeepSeek与Harness背景调研.md) |
| LLM 核心概念（Context / Token / KV Cache）| [Researcher 包 02](../interview-prep-deepseek-agent-researcher/02-DeepSeek研究文化与Agent研究方向调研.md) |
| 通用 STAR-C 行为题框架 | [Researcher 包 10](../interview-prep-deepseek-agent-researcher/10-高频面试题-行为与跨团队协作篇.md) |
| 通用临场策略（开场 60 秒 / 卡壳话术） | [Engineer 包 10](../interview-prep-deepseek-harness-engineer/10-面试中临场策略与话术.md) |
| 通用复盘模板 + V0.2 commit 策略 | [Engineer 包 11](../interview-prep-deepseek-harness-engineer/11-面试后复盘模板与跟进话术.md) |

本包只写**C 端 PM 特化**内容——用户洞察、竞品分析、增长指标、A/B test、AI 产品方法论、DeepSeek critique。

---

## 八、一句忠告

> **DeepSeek C 端 PM ≠ ChatGPT PM。DeepSeek 的产品哲学是"模型先于产品"——这家公司没有重运营文化，没有 KPI，PM 必须是"用产品力 + 模型 insight 自驱"型，不是"数据驱动 KPI"型。**

> **面试时，"我能帮 DeepSeek 把豆包式增长打法搬过来" 是错答案；正确答案是 "我能帮 DeepSeek 把世界级模型能力 translate 成普通用户能感知的产品价值，让用户因为'好用'而留下不是因为'红包'。"**

> **C 端 PM 的精髓在于：你看到的不是按钮，是用户场景；不是 DAU，是用户的真实问题被解决。这个 mindset 决定你是 PM 还是真正的 product person。**

祝顺利。
