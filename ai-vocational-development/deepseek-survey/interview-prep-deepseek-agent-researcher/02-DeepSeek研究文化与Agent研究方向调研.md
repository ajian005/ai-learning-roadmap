# 02 · DeepSeek 研究文化与 Agent 研究方向调研

> 公司背景请先读：[../interview-prep-deepseek-harness-pm/02-DeepSeek与Harness背景调研.md](../interview-prep-deepseek-harness-pm/02-DeepSeek与Harness背景调研.md)。
>
> 本文档**只列研究员岗特化补充**：DeepSeek 研究方法论、自家 paper / 算法（V3、R1、GRPO、MLA、MoE）、Agent capability 研究路线图。

---

## 一、DeepSeek 研究文化（研究员必知）

### 1.1 三个公开 signal

1. **"无 KPI、无赛马、研究优先"**：内部公开报道反复提到。意味着 researcher 不为短期 metric 服务，可以 take risk。
2. **"模型与 Harness 共同进化"**：公开 statement，证明 research team 和 product team 是 paired，不是 silo。
3. **"open weight + paper-first"**：V3 / R1 全开 + 详细 technical report。这种公司不会 hire "只会调 prompt" 的 researcher。

### 1.2 招聘暗示

| 信号 | 解读 |
| --- | --- |
| 招博士 + IMO/ICPC 金牌 | bar 很高 |
| 团队组成 ~ 200+ researcher（推测）| 比 Anthropic 早期还小 + 研究密度高 |
| 北京 + 杭州两地 | 研究和工程团队跨地 collab |
| 不在 KPI 文化里成长的人来了不适应 | 反向：长期主义 +自驱必备 |

### 1.3 研究方法论（推测）

DeepSeek 的 paper 风格是 **"hypothesis + scaled experiment + 极致 efficiency"**——和 OpenAI 路线相似（vs DeepMind 偏 theory）。看 R1 paper 就知道：作者敢 say "we tried SFT first，failed，then tried pure RL，worked"——这种 "honest about failure" 是 research 品味的标志。

---

## 二、DeepSeek 自家 paper / 算法（**必背**）

### 2.1 DeepSeek-V3（2024-12）

> **核心创新**：MoE 大规模 efficiency + MLA + FP8 训练

| 概念 | 30 秒 |
| --- | --- |
| **MLA**（Multi-head Latent Attention） | 把 K/V 压缩到低维 latent，再展开；显著减 KV cache 内存（vs MHA / GQA / MQA） |
| **DeepSeekMoE** | 细粒度专家（细分到 ~256 个，每个小专家）+ shared expert（捕获 common knowledge）+ auxiliary-loss-free load balancing（替代传统 aux loss） |
| **multi-token prediction** | training 时一次预测多 token；推理时可做 speculative decoding 起点 |
| **FP8 训练** | 工程上首批大规模 FP8 训练实现；省 memory + 通信 |

**面试金句**："MLA 是 DeepSeek 解决 long context KV cache 爆炸的关键——比 GQA 更激进。"

### 2.2 DeepSeek-R1（2025-01）

> **核心创新**：用 RL 自己学会 reasoning + GRPO + cold-start trick

| 概念 | 30 秒 |
| --- | --- |
| **R1-Zero** | **纯 RL** + rule-based reward（无 SFT 起步！）→ 模型自主 emerge "等等让我重新想想" 这种 reasoning 模式 |
| **R1** | 在 R1-Zero 基础上加 cold-start SFT + multi-stage RL + 防 mix-language（中英文混合）trick |
| **GRPO**（Group Relative Policy Optimization） | **替代 PPO 的 critic**——同一 prompt 采样 group of responses，用 group mean 作 baseline，省一个 value model |
| **distill 到小模型** | 用 R1 生成的 CoT 训 1.5B/7B/14B/32B/70B distilled 版本 |
| **"Aha moment"** | R1-Zero 训练中观察到的 self-reflection 涌现现象——成为 RL emergence 的 iconic 例子 |

**面试金句**："R1 paper 的 'aha moment' 是 emergence 的标志性 finding——纯 RL 让模型自己学会 self-correction，这是 DeepSeek 给 alignment field 最大贡献之一。"

### 2.3 GRPO 数学（**特别重要**——自家算法必考）

> 详细推导见 [03 文档] 第 4 节。这里给 30 秒版本。

**PPO 标准目标**：
$$
L^{PPO}(\theta) = E_t \left[ \min(r_t(\theta) A_t, \text{clip}(r_t(\theta), 1-\epsilon, 1+\epsilon) A_t) \right] - \beta \cdot KL(\pi_\theta \| \pi_{ref})
$$
其中 $A_t = R_t - V_\phi(s_t)$ 需要一个 value model $V_\phi$。

**GRPO 改造**：把 $A_t$ 替换为 **group-relative advantage**——
$$
A_i = \frac{r_i - \text{mean}(\{r_1, ..., r_G\})}{\text{std}(\{r_1, ..., r_G\})}
$$
其中 $r_i$ 是同一 prompt 的 G 个 sampled responses 的 reward。

**优势**：
- ✅ 省 value model（memory & compute -25%~33%）
- ✅ baseline 在 group 内 normalize，方差小
- ✅ 特别适合 LLM RLHF——reward sparse + episode 短

**劣势**：
- ❌ 不能用于长 episode（需要 step-level value）
- ❌ group size 需要平衡（G 太小 baseline 不准；G 太大 compute 爆）

### 2.4 DeepSeek-V3.1 / V3.2（推测）

> 截止 2026-05，DeepSeek 持续迭代。研究员面试前**必查最新 release notes / 技术报告**。

可能的演进方向：
- Agent capability uplift（tool use / planning）
- 多模态（VLM）
- 长 context（>200K）
- inference efficiency

---

## 三、Agent capability 研究的 6 大方向（DeepSeek 重点）

研究员日常 research 主要围绕下面 6 件事——每件事都要能聊出 SOTA / 限制 / 方向。

### 方向 1·Reasoning（推理能力）

| 子问题 | 主要工作 |
| --- | --- |
| Chain-of-Thought 生成 | CoT prompting (Wei 2022) → SFT on CoT → RL on outcome |
| Process supervision | OpenAI PRM800K (2023)、Let's verify step by step |
| Reasoning model | OpenAI o1 → DeepSeek R1 → Claude 4.5 thinking |
| Self-correction | 让模型识别自己错误后重做 |
| Search + Reasoning | MCTS + LLM (e.g., AlphaGeometry) |

### 方向 2·Tool Use / Function Calling

| 子问题 | 主要工作 |
| --- | --- |
| Tool spec 设计 | OpenAI function calling / Anthropic tool_use / MCP |
| Tool selection | "Toolformer" (Schick 2023) |
| Multi-tool orchestration | "ReAct" (Yao 2022) |
| Agent SFT/RL on tool use | "Agent-FLAN" / "AgentBoard" / Tülu series |
| 长 horizon tool use | Anthropic 11 月 harness blog 涉及 |

### 方向 3·Instruction Following

| 子问题 | 主要工作 |
| --- | --- |
| Instruction tuning | "FLAN" (Wei 2021) → InstructGPT (Ouyang 2022) |
| Complex instruction | "Conifer" / "FollowBench" |
| Multi-turn following | "MT-Bench" / "Arena-Hard" |
| 指令 robustness | adversarial reformulation |

### 方向 4·Code Generation / Coding Agent

| 子问题 | 主要工作 |
| --- | --- |
| 代码生成 SFT | "Code Llama" / "DeepSeek-Coder" |
| RL on code | "AlphaCode" / "Reflexion" |
| Coding agent | SWE-Agent / Aider / OpenHands |
| Eval | "SWE-Bench" / "SWE-Bench Pro" / "AiderBench" / "LiveCodeBench" |

### 方向 5·RL 对齐 (RLHF / RLAIF)

| 子问题 | 主要工作 |
| --- | --- |
| RLHF | InstructGPT (2022) - 经典 3 stage（SFT → RM → PPO）|
| DPO | Rafailov 2023 - 直接优化偏好，无 reward model |
| RLAIF | Anthropic Constitutional AI (2022)，AI 自评代替人评 |
| Process Reward | OpenAI PRM800K, MathShepherd |
| GRPO | DeepSeek R1 |
| RLOO | "RL with Limited Online Sampling" |

### 方向 6·长 Horizon / Multi-step Reasoning

| 子问题 | 主要工作 |
| --- | --- |
| Long-horizon planning | Voyager (Wang 2023) |
| Subagent / multi-agent | Anthropic harness paper, AutoGen |
| Context engineering | Anthropic context engineering blog |
| Eval | LongBench / RULER / Multi-step benchmarks |

---

## 四、DeepSeek Harness 团队和研究院的协作（推测）

```
Research Team                     Harness Team
─────────────                    ─────────────
- pretrain                        - 产品 (Code Agent)
- alignment (RL)                  - 评测平台
- Agent capability                - Container service
- paper                           - Scaffold 集成
        │                                │
        └────────"模型与 Harness 共同进化"─┘
                        │
                trajectory feedback
                + capability eval
                + RL training data
```

> **研究员日常和 Harness team 的接口点**：
> 1. Harness 提供 trajectory 数据（real-world distribution）
> 2. Research 用 trajectory 训 reward model / 调 RL pipeline
> 3. Research push 新 ckpt 给 Harness deploy
> 4. Harness eval 平台跑 capability benchmark
> 5. Failure mode 反馈给 Research 提下一 hypothesis

---

## 五、必读的 paper / 资料地图（按优先级）

### Tier 1·绝对必读（5 篇）

1. **InstructGPT** (Ouyang et al. 2022) — RLHF 经典 3 stage
2. **DPO** (Rafailov et al. 2023, NeurIPS Outstanding) — preference learning 简化
3. **DeepSeek-R1** (DeepSeek 2025) — 自家旗舰 + GRPO
4. **DeepSeek-V3** (DeepSeek 2024) — MoE/MLA 公司基石
5. **Constitutional AI** (Anthropic 2022) — RLAIF 鼻祖

### Tier 2·重要 paper（10 篇）

6. **PPO** (Schulman 2017) — RL 基础
7. **Toolformer** (Schick 2023) — tool use 训练
8. **ReAct** (Yao 2022) — agent 基础范式
9. **Reflexion** (Shinn 2023) — self-correction
10. **Voyager** (Wang 2023) — skill library
11. **PRM800K / Let's verify** (OpenAI 2023) — process reward
12. **OpenAI o1 blog / report** (2024) — reasoning model
13. **GRPO** 出处 (DeepSeekMath, 2024-02) — 自家算法历史
14. **Anthropic Constitutional AI v2** — 改进 RLAIF
15. **Tülu 3** (AI2 2024) — open RLHF recipe + DPO/RLOO 比较

### Tier 3·研究品味加分（10 篇）

16. **SWE-Bench (Pro)** — coding agent eval
17. **AgentBench / AgentBoard** — agent eval
18. **MathShepherd** — process reward in math
19. **Group relative DPO** (KTO / SimPO 系列) — DPO 变体
20. **RLOO** — 简化 RL 算法
21. **Anthropic harness blog ×3** — engineering vs research interface
22. **OS Atlas** — vision-based GUI agent
23. **Magpie** (2024) — synthetic data generation
24. **Self-Rewarding LLM** (Yuan 2024) — LLM 自评
25. **Process reward model + MCTS** (e.g., AlphaGeometry) — search + LLM

### 必读 blog / 资料

- **Lilian Weng's blog** — RL / Alignment / Agent 综述（fastest learning curve）
- **Sebastian Raschka's blog** — RLHF / DPO/PPO 实操
- **Chip Huyen's blog** — ML system 视角
- **Hamel Husain's LLM eval blog**
- **Anthropic engineering blog** —especially harness 三篇
- **Lex Fridman × Dario Amodei interview** — Anthropic alignment view
- **Lex × Aravind Srinivas** — Perplexity / search agent

---

## 六、研究员"读 paper SOP"

```
1. Title + Abstract       30s     决定 read more / pass
2. Figure 1 + intro       3min    抓 core idea
3. Method + main results  10min   懂 contribution
4. Ablation              5min    判断 robustness
5. Limitations + future  3min    自己 take
6. 复刻评估              2min    决定要不要 reproduce

每篇 < 25 分钟决定 follow / pass。
follow 的写 200 字带读 → personal notes。
```

---

## 七、面试可能被深问的"研究问题"清单

> 主动 surface 出来证明你想过 frontier 问题。

| 研究问题 | 难点 | 我的初步思路 |
| --- | --- | --- |
| Reasoning model 怎么避免 over-thinking 短 query？| budget 控制；early stop | adaptive budget based on perplexity / confidence |
| Process reward 怎么 scale 标注？| 人标注成本高 | LLM-as-PRM；rule-based PRM；mixed |
| Tool use trajectory 数据怎么自动 mining | 标注昂贵 | self-instruct + execution feedback filter |
| Agent long-horizon context anxiety | Sonnet 4.5 提前收尾 | context reset；handoff；多 agent |
| RL reward hacking | hard to detect | adversarial probe + holdout eval set |
| 多模态 Agent（GUI / VLM） | benchmark 少 | OS Atlas / WebArena / VisualWebBench |
| Self-improvement 上限 | 模型自评偏正 | separate evaluator agent；human ground truth |
| 小模型如何具备 Agent 能力 | tool use / planning 难 | distillation from R1 / structured CoT SFT |

---

## 八、一句应试钥匙（研究员特化）

> **当面试官提到 DeepSeek 或行业，"我看新闻知道" 是初级答法。研究员答法是 "我看了 R1 paper 第 5 节 GRPO 的具体推导，对比 PPO 主要 saving 是去掉 value model 的 ~30% 参数 + ~25% compute。我自己实现过 GRPO 在 toy task 上的 mini 版本，发现 group size = 8 是性能/cost 拐点——可以 share notebook。"**
