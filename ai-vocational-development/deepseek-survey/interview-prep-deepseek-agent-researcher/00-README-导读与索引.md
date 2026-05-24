# DeepSeek「Agent 深度学习算法研究员」面试准备包

> **岗位**：Agent 深度学习算法研究员（DeepSeek，模型推理 / 生成 / 指令遵循 / RL 对齐方向）
> **本包整理时间**：2026-05-21
> **目标**：把这份材料当作"面试前 7~14 天集中冲刺手册"——覆盖从理解 JD、研究文化、RL 数学弹药、paper 深读、研究品味、coding（PPO/RLHF 实现）、open research 题、临场、复盘到 follow-up 全流程。

---

## 一、这个岗位的"定位"——和已有三包的关系

> DeepSeek Agent / Harness 体系下目前已经在招的 4 个岗位（同一团队网络 + 研究院）：
>
> | 岗位 | 定位 | 准备包 |
> | --- | --- | --- |
> | Harness **产品经理** | 定义产品 + 路线图 | `../interview-prep-deepseek-harness-pm/` |
> | Harness **研发工程师** | 造给用户的 Code Agent 产品 | `../interview-prep-deepseek-harness-engineer/` |
> | **Agent 全栈工程师** | 造给算法用的评测平台 + 容器 + Scaffold | `../interview-prep-deepseek-agent-fullstack/` |
> | **Agent 算法研究员**（本岗） | **训模型本身**——RL、对齐、reasoning、Agent capability | **本目录** |

**核心差异（一句话）**：
- 三个工程 / PM 岗 = "**把模型变成产品**"
- **本岗 = "把模型本身变得更强"**——你的产出是新的训练方法 / 新的 reward 设计 / 新 paper / 新 ckpt

> **这个岗位的本质是 ML researcher**——你的同事是顶校博士 / IMO 金牌 / 顶会一作。考察 bar 远高于纯工程岗。日常工作是"读 paper → 提 hypothesis → 设计 ablation → 跑实验 → 写 paper / 训出 SOTA"。

详细的四岗对照：[99-四岗位差异对照与选择建议.md](99-四岗位差异对照与选择建议.md)

---

## 二、JD 关键能力一句话总结

> **"独立 0→1 提研究 idea + 快速 prototype + 跑 RL 实验闭环 + 跨界和数据/工程协作 + 跟踪复刻前沿 paper。"**

加上**必须**懂：Python + C/C++ + PyTorch + RL 基础（PPO/DPO/Actor-Critic 起步）+ LLM 训练全 pipeline（pretrain / SFT / RL / eval）。

加分项里"**顶会一作**" + "**Agent 系统经验**" + "**深度用过 CC/Cursor 对模型边界有实践认知**" 是最重要的三个差异化信号。

---

## 三、目录与使用建议

| 序号 | 文件 | 用途 |
| --- | --- | --- |
| 01 | [01-JD深度解读与岗位画像.md](01-JD深度解读与岗位画像.md) | 逐句拆解 + 隐藏信号 + 四岗对照 |
| 02 | [02-DeepSeek研究文化与Agent研究方向调研.md](02-DeepSeek研究文化与Agent研究方向调研.md) | DeepSeek 研究文化 + 自家 paper（V3/R1/GRPO）+ Agent research roadmap |
| 03 | [03-RL核心算法数学速查.md](03-RL核心算法数学速查.md) | **核心弹药**：PPO / DPO / GRPO / RLHF / RLAIF / Process Reward 数学推导 + 实现关键 |
| 04 | [04-LLM-Agent前沿paper速查与阅读路径.md](04-LLM-Agent前沿paper速查与阅读路径.md) | 必读 25 篇 paper 简评 + 阅读路径 + 高频被引 |
| 05 | [05-高频面试题-RL与对齐数学深问篇.md](05-高频面试题-RL与对齐数学深问篇.md) | 12 道数学深问题（研究员对话级） |
| 06 | [06-高频面试题-Agent能力提升与训练方法篇.md](06-高频面试题-Agent能力提升与训练方法篇.md) | Agent 训练 / tool use / planning / process reward / reasoning 训练方法题 |
| 07 | [07-高频面试题-Paper深读与研究品味篇.md](07-高频面试题-Paper深读与研究品味篇.md) | paper review / 提 hypothesis / ablation / replication 题 |
| 08 | [08-高频面试题-编码与PPO实现篇.md](08-高频面试题-编码与PPO实现篇.md) | 手撕 PPO loss / DPO loss / RLHF pipeline / reward model |
| 09 | [09-高频面试题-Research开放题与idea-design篇.md](09-高频面试题-Research开放题与idea-design篇.md) | 开放性 research 题：从问题到 idea 到 ablation 设计 |
| 10 | [10-高频面试题-行为与跨团队协作篇.md](10-高频面试题-行为与跨团队协作篇.md) | 研究员特化行为题 + 和数据 / 工程协作 + 抗压 |
| 11 | [11-反问环节话术库.md](11-反问环节话术库.md) | 研究员级反问（compute / 自由度 / 评测 / paper policy） |
| 12 | [12-面试前-中-后全流程行动手册.md](12-面试前-中-后全流程行动手册.md) | T-14 ~ T+0 行动 + 临场策略 + 复盘三合一 |
| 13 | [13-自带作品集-research-portfolio与mini复刻.md](13-自带作品集-research-portfolio与mini复刻.md) | research portfolio 一页纸 + mini paper 复刻项目设计 + 技术博客 |
| 99 | [99-四岗位差异对照与选择建议.md](99-四岗位差异对照与选择建议.md) | PM/Harness Eng/全栈 Eng/研究员 四岗位全景对照 |

---

## 四、整体策略（一句话）

> **"用顶会一作的硬通货 + 复刻业界前沿的速度 + 和 Sonnet 4.5 一样深的模型行为品味——证明我能成为 DeepSeek 研究员的 peer，不是 PhD intern。"**

具体落地三件事：

### 1. 一个能 demo 的 mini paper 复刻（最关键的杀手锏）
研究员岗位最稀缺的不是 "知道 PPO" ，是 "**你自己跑过 PPO 训过 reward model 看过 reward hacking 的 trajectory**"。

如果你能在 T-14 ~ T-3 这两周里完成 [13 文档] 的 mini 复刻——比如：
- 复刻一篇 NeurIPS/ICLR 2025 RLHF paper 的 1 个核心 figure（不需复刻全部）
- 或写一个 simplified GRPO 实现 + 训一个 toy task
- 或做一个 DPO vs PPO 对照实验

**这一件事就足以让你和 80% 候选人区分开**——简历上写"懂 PPO" 是 zero proof；GitHub 上有一个能跑的 RLHF mini repo 是 ten proof。

### 2. 30 个核心概念 + 5 篇 paper 烂熟
研究员面试会反复在概念之间游走。30 个核心概念（[03 文档] 全部）+ 5 篇 cornerstone paper（[04 文档] 顶部 5 篇）每篇能 30 秒讲核心 + 3 分钟讲细节——这是基础功。

### 3. 能在 30 分钟里给一个 open research 问题设计 ablation
研究员面试的高分动作是 "面试官给一个 vague 问题，你能 30 分钟内提出 hypothesis + 设计 minimal ablation + 列 expected result + 想清楚 failure mode"。

[09 文档] 全是这种练习题。

---

## 五、面试评估维度的内部模型（推测）

| 维度 | 权重 | 关键问题 |
| --- | --- | --- |
| **RL/对齐数学深度** | 25% | "推导 PPO 的 clipped surrogate loss" / "DPO 为什么不需 reward model" |
| **Paper 阅读品味** | 20% | "你最近读的最让你 inspired 的 paper / 最不同意的 paper" |
| **0→1 research idea** | 20% | "给你 vague 问题 X，30 分钟设计 ablation" |
| **复刻 / 实现能力** | 15% | "白板写 PPO loss" / "用 PyTorch 实现 DPO" |
| **Agent / 模型行为品味** | 10% | "深度用过 CC/Cursor 体验到的模型边界" |
| **跨团队协作 / 文化** | 10% | "和数据 / 工程 / 其他研究员合作过吗" |

---

## 六、推荐节奏（T-14 ~ T+0）

> 研究员岗位比工程岗准备时间长——RL 数学和 paper 阅读不能临时抱佛脚。

```
T-14  战略 + 启动 mini 复刻项目
T-12  RL 数学：PPO 完整推导 + 实现
T-10  RL 数学：DPO / GRPO / RLHF pipeline
T-8   paper 阅读：5 篇 cornerstone 深读
T-6   paper 阅读：10 篇近 12 月 + 自己的批判性 take
T-4   open research 题练习（10+ 道）
T-3   系统 / 行为 + 1 次 mock interview
T-2   作品集 polish + 反问 + 简历
T-1   静心 + 装备
T-0   面试当天
T+0   24h 复盘 + 感谢邮件
```

总投入 **60~100 小时**——建议分摊到 2 周里，每天 4~7 小时。
（这个比前几个岗位准备时间长 50%，因为 paper 阅读 + RL 数学不能压缩。）

---

## 七、和已有三包的引用约定

为避免重复，本包**直接引用**已有三包的相关章节：

| 内容 | 直接引用 |
| --- | --- |
| DeepSeek 公司背景 / Harness 团队时间线 | [PM 包 02](../interview-prep-deepseek-harness-pm/02-DeepSeek与Harness背景调研.md) |
| 14 个 LLM/Agent 核心概念 | [Eng 包 03](../interview-prep-deepseek-harness-engineer/03-Harness技术体系与代码模式速查.md) |
| DeepSeek 文化 / 不发奖金 / 不赛马 | [PM 包 07](../interview-prep-deepseek-harness-pm/07-高频面试题与参考答案-行为与文化篇.md) |
| 通用临场策略（开场 60 秒 / 卡壳话术） | [Eng 包 10](../interview-prep-deepseek-harness-engineer/10-面试中临场策略与话术.md) |
| 通用复盘模板 | [Eng 包 11](../interview-prep-deepseek-harness-engineer/11-面试后复盘模板与跟进话术.md) |

本包只写**研究员岗特化**内容——RL 数学、paper 品味、idea design、复刻能力。

---

## 八、一句忠告

> **DeepSeek 是中国少数 'research-first' 的 frontier 模型公司——研究员岗位的 bar 接近 Anthropic / OpenAI，不接近字节腾讯。**

> **会跑 PyTorch 训 SFT 的人很多，能讲清 'GRPO 比 PPO 在 RLAIF 场景下的优势是 group-relative baseline 减去 critic 的偏差'的人极少。**

> **你这两周的本质工作不是 '学技术'，是 '让面试官在 5 分钟里把你归类为 peer 而不是 candidate'——peer 的标志是：你能和他/她吵 paper 的 trade-off。**

祝顺利。
