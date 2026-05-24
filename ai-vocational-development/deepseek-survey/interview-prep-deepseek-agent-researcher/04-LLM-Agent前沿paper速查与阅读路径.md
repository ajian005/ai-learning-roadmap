# 04 · LLM / Agent 前沿 Paper 速查与阅读路径

> 研究员面试最常见的"灵魂问题"：**"你最近读的最 inspired 的 paper 是什么？为什么？"**
>
> 这一份按 **5 个主题 × 5 篇核心**整理 25 篇必读 paper，每篇配 30 秒摘要 + 关键 contribution + ablation 看点 + 你的批判性 take。**重点是后两个**——能 critique 才说明真读过。
>
> 阅读优先级：先 Tier 1 ★★★（必背），再 Tier 2 ★★，最后 Tier 3 ★ 拓展。

---

## 一、5 个主题分布

| 主题 | paper 数 | 对应能力 |
| --- | --- | --- |
| A. RL / Alignment 经典 | 5 | RLHF / DPO / GRPO / RLAIF 基础 |
| B. Reasoning 与 Process Reward | 5 | reasoning model / o1 / R1 思路 |
| C. Agent System / Tool Use | 5 | tool use / planning / scaffolding |
| D. Code Generation / Coding Agent | 5 | DeepSeek-Coder / SWE-Bench |
| E. Data / Eval / Self-improvement | 5 | data-centric AI / agent benchmark |

---

## 二、主题 A · RL / Alignment 经典 5 篇

### A1·InstructGPT (Ouyang et al. 2022) ★★★

- **题**：Training language models to follow instructions with human feedback
- **arXiv**: 2203.02155
- **30 秒**：RLHF 三阶段范式开山——SFT → RM → PPO。GPT-3 1.3B + RLHF > GPT-3 175B 原版（在 instruction following 上）。这篇是当代 ChatGPT 的 spiritual ancestor。
- **关键 contribution**：(1) 把 helpful/harmless/honest 分解；(2) 证明 alignment 不靠 scaling 而靠 method
- **Ablation 看点**：1.3B+RLHF vs 175B base 的对比 → small but aligned > big but raw
- **批判性 take**：reward model 训练数据偏 ranker 主观；只看 helpfulness 不看深 reasoning；没解决 reward hacking

### A2·DPO (Rafailov et al. 2023) ★★★

- **题**：Direct Preference Optimization: Your Language Model is Secretly a Reward Model
- **NeurIPS 2023 Outstanding Paper**
- **30 秒**：用闭式解把 RLHF 的 reward + PPO 两阶段合一阶段。loss = $-\log\sigma(\beta \log\frac{\pi_w/\pi_{ref}}{\pi_l/\pi_{ref}})$。简单稳定。
- **关键 contribution**：(1) 数学推导漂亮 (BT model + KL-constrained RL 最优解)；(2) 实验证明 ≈ PPO 但简单
- **Ablation 看点**：sentiment / summarization / dialogue 上和 PPO 平手或微弱优；$\beta$ 0.1 vs 0.5 影响
- **批判性 take**：实际产业部署 PPO 仍然上限更高（OpenAI/Anthropic 没切 DPO）；DPO 对 preference data 噪声敏感；只能 fit pair，OOD 外推差

### A3·Constitutional AI (Bai/Anthropic 2022) ★★

- **题**：Constitutional AI: Harmlessness from AI Feedback
- **arXiv**: 2212.08073
- **30 秒**：用一组 ~20 条 principles 让 AI 自评 → RLAIF 鼻祖。Claude 的精神 root。
- **关键 contribution**：(1) 用 prompt 引导 LLM self-critique；(2) AI feedback 替代人类 label
- **Ablation 看点**：CoT critique 比 direct preference 好；principle 数量影响
- **批判性 take**：principle 是英文 hard-coded，跨文化 / 中文场景需要重设计；判官 LLM 偏 verbose

### A4·DeepSeek-R1 / GRPO (DeepSeek 2025) ★★★

- **题**：DeepSeek-R1: Incentivizing Reasoning Capability in LLMs via Reinforcement Learning
- **30 秒**：自家旗舰。R1-Zero 纯 RL 涌现 reasoning ("aha moment")；R1 加 cold-start SFT + multi-stage RL。GRPO 替代 PPO。
- **关键 contribution**：(1) 证明 pure RL 也能学 reasoning；(2) GRPO 工程简化；(3) distill 到小模型仍强
- **Ablation 看点**：R1-Zero vs R1 pipeline；distillation 7B vs original RL trained 7B
- **批判性 take**：language-mixing 问题（之前需要 cold-start trick）；纯 RL 时间长 + 不稳；reward 必须是 verifiable（数学题）才好用

### A5·Tülu 3 (AI2 2024) ★★

- **题**：Tülu 3: Pushing Frontiers in Open Language Model Post-Training
- **30 秒**：最 transparent 的 open RLHF recipe——SFT + DPO + RLVR (RL with Verifiable Reward)。
- **关键 contribution**：(1) 公开 8B/70B 训练完整 recipe + data；(2) 比较 PPO/DPO/RLOO/SimPO 等
- **Ablation 看点**：DPO vs RLOO；RLVR vs RLHF；data mixing
- **批判性 take**：closed model（GPT-4/Claude）还是强；很多 trick 是 "magic"，没解释清

---

## 三、主题 B · Reasoning 与 Process Reward 5 篇

### B1·Chain-of-Thought (Wei et al. 2022) ★★★

- **题**：Chain-of-Thought Prompting Elicits Reasoning in Large Language Models
- **arXiv**: 2201.11903
- **30 秒**：在 prompt 里加 "Let's think step by step" → 大模型 reasoning 涌现。100B+ 模型有效，10B 几乎无效。
- **关键 contribution**：发现 emergent reasoning
- **Ablation 看点**：scale curve：8B 几乎无效，60B+ 才有；CoT vs direct answer
- **批判性 take**："think step by step" 只是激活模型已 latent 的 reasoning；不教新 capability；后续 RL on CoT 才真正提升

### B2·Let's Verify Step by Step (Lightman/OpenAI 2023) ★★★

- **题**：Let's Verify Step by Step
- **arXiv**: 2305.20050
- **30 秒**：发布 PRM800K（80 万 step-level 数学 label），证明 process reward > outcome reward。
- **关键 contribution**：(1) 大规模 process reward dataset；(2) process > outcome；(3) 减少 false positive (反向 reasoning 蒙对)
- **Ablation 看点**：PRM vs ORM (outcome RM) 选 best-of-N 时；K↑ 时 PRM 优势↑
- **批判性 take**：标注成本贵；只在 math；scale 到 code / agent 待解

### B3·OpenAI o1 (technical blog 2024) ★★★

- **题**：Learning to Reason with LLMs
- **30 秒**：纯 RL 训练 reasoning；test-time compute scaling；隐藏 CoT。
- **关键 contribution**：(1) test-time scaling 第一次成产品；(2) RL on reasoning 工业级 ship
- **Ablation 看点**：thinking token 数 vs benchmark 准确率（log-linear）
- **批判性 take**：blog 不是 paper，技术细节藏；价格贵；over-thinking 短问题；不开放 CoT 限制 research

### B4·MathShepherd (Wang et al. 2024) ★★

- **题**：Math-Shepherd: Verify and Reinforce LLMs Step-by-step without Human Annotations
- **30 秒**：用 Monte Carlo rollout 自动生成 PRM label——无需人标。
- **关键 contribution**：免人标 PRM；用 N rollout 成功率作 step value
- **Ablation 看点**：rollout 数 N vs PRM 质量；自动 vs PRM800K 比较
- **批判性 take**：rollout 成本高（N 倍 generation）；估值有噪声；只适用于 verifiable task

### B5·DeepSeek-Prover (DeepSeek 2024) ★

- **题**：DeepSeek-Prover: Advancing Theorem Proving in LLMs through Large-Scale Synthetic Data
- **30 秒**：用 synthetic Lean proof 训证明专用模型。
- **批判性 take**：narrow domain；但思路（synthetic data + RL）非常 transferable

---

## 四、主题 C · Agent System / Tool Use 5 篇

### C1·ReAct (Yao et al. 2022) ★★★

- **题**：ReAct: Synergizing Reasoning and Acting in Language Models
- **arXiv**: 2210.03629
- **30 秒**：Thought / Action / Observation 交替——agent paradigm 鼻祖。
- **关键 contribution**：把 reasoning trace 和 action 交织；HotpotQA / ALFWorld 改善
- **批判性 take**：现在被 native tool_use 替代；prompt 形式 brittle；但思想活在所有 modern agent loop 里

### C2·Toolformer (Schick et al. 2023) ★★

- **题**：Toolformer: Language Models Can Teach Themselves to Use Tools
- **arXiv**: 2302.04761
- **30 秒**：self-supervised 训练 LLM 学会调 tool——用 perplexity 反向判断"调 tool 会更好吗"。
- **关键 contribution**：免人标 tool data
- **Ablation 看点**：哪些 tool 用得对（calculator / search 强；QA tool 差）
- **批判性 take**：6B 模型；现代 70B+ 直接 prompt 就会用 tool；但 self-supervised data generation 思路重要

### C3·Reflexion (Shinn et al. 2023) ★★

- **题**：Reflexion: Language Agents with Verbal Reinforcement Learning
- **30 秒**：失败后写一段"教训"放回 context → next iteration 用 → verbal RL。
- **关键 contribution**：text-based memory 替代参数更新；HumanEval 91% (GPT-4)
- **批判性 take**：cost 高（多次 iter）；reflection 质量 sensitivity；不适用 long horizon

### C4·Voyager (Wang et al. 2023) ★★

- **题**：Voyager: An Open-Ended Embodied Agent with Large Language Models
- **30 秒**：Minecraft 永生 agent——skill library 动态扩张；env feedback 驱动学新 skill。
- **关键 contribution**：long horizon + lifelong learning 范式
- **批判性 take**：narrow env；skill library 是关键，可扩展到 coding agent

### C5·Agent-FLAN / AgentTuning (2023~2024) ★

- **30 秒**：用 mixed agent trajectory data SFT base model → agent capability uplift。
- **关键 contribution**：agent SFT data recipe
- **批判性 take**：SFT 上限低于 RL；但 cold-start 必经

---

## 五、主题 D · Code Generation / Coding Agent 5 篇

### D1·SWE-Agent (Yang et al. 2024) ★★★

- **题**：SWE-agent: Agent-Computer Interfaces Enable Automated Software Engineering
- **30 秒**：在 SWE-Bench 上证明 ACI（Agent-Computer Interface 设计） 比 base model 还重要——good UI for LLM。
- **关键 contribution**：(1) ACI 概念；(2) SWE-Bench 显著提升（从 ~5% → ~20%）
- **批判性 take**：现在被 Claude Code / Aider 等 superceded；但 ACI 理念延续

### D2·SWE-Bench / SWE-Bench Verified (Jimenez et al. 2024) ★★★

- **题**：SWE-bench: Can Language Models Resolve Real-World GitHub Issues?
- **30 秒**：用真实 GitHub issue + 单测作 eval——coding agent 黄金 benchmark。
- **关键 contribution**：真实 distribution + 可自动 grade
- **批判性 take**：Verified subset 重要（原版有错 issue）；data leakage 风险（issues 在 train data 里）

### D3·DeepSeek-Coder (DeepSeek 2024) ★★

- **题**：DeepSeek-Coder: When the Large Language Model Meets Programming
- **30 秒**：自家 code 模型——repo-level pretraining + FIM + 多语言。
- **关键 contribution**：(1) repo-level 数据组织；(2) 1.3B/6.7B/33B 全开 ckpt
- **批判性 take**：现在被 Claude Sonnet 4.5 / GPT-4.x 等超越 in coding；但 base 在中文场景仍强

### D4·Aider (开源工具，无 paper) ★★

- **30 秒**：开源 coding agent，diff-based editing；最早的 production 级 coding agent。
- **批判性 take**：和 Claude Code 设计对照——Aider 偏 diff，CC 偏 tool-use 多文件

### D5·Reflexion for code / Self-Debugging (Chen 2023) ★

- **30 秒**：写代码 → 跑 → fail → 看 error → 重写。verbal RL 在 code 场景。
- **批判性 take**：cost 高但效果显著；现代 coding agent loop 标准

---

## 六、主题 E · Data / Eval / Self-improvement 5 篇

### E1·Self-Instruct (Wang et al. 2022) ★★★

- **题**：Self-Instruct: Aligning Language Models with Self-Generated Instructions
- **30 秒**：用 LLM 生成 instruction data → SFT → 改进 LLM → 循环。data 自举的开山。
- **关键 contribution**：免人标 SFT data
- **批判性 take**：早期质量低；现代用 Magpie / 强模型 distill 替代

### E2·Magpie (2024) ★★

- **题**：Magpie: Alignment Data Synthesis from Scratch by Prompting Aligned LLMs with Nothing
- **30 秒**：仅用 BOS token 让 aligned LLM 自己生成"问题 + 回答"对——免人 prompt seed。
- **关键 contribution**：纯 generation，零人工
- **批判性 take**：data 模型 distribution-bound；和原模型相似 → 提升有限

### E3·Self-Rewarding LLM (Yuan et al. 2024) ★★

- **题**：Self-Rewarding Language Models
- **30 秒**：模型既当 generator 又当 judge → DPO 自更新 → 迭代。
- **批判性 take**：3 iter 后开始 reward hack；sycophancy；Anthropic harness paper 推 separate evaluator 正是 push back

### E4·HumanEval / MBPP / LiveCodeBench (eval benchmarks) ★★

- **30 秒**：HumanEval 164 题、MBPP 974 题，已经 saturate；LiveCodeBench 用 contest 题滚动更新避数据污染。
- **批判性 take**：HumanEval 太简单；用 LiveCodeBench / SWE-Bench 才能区分 SOTA

### E5·Arena-Hard / WildBench / MT-Bench (judge eval) ★★

- **30 秒**：用 strong LLM 作 judge 评 chat 质量。Arena-Hard 用 LMSYS Arena 实战数据。
- **批判性 take**：judge bias（verbose preferred、格式 preferred）；和真用户偏好相关性 ~80%；需 disclose judge model

---

## 七、Tier 3·研究品味加分 5+ 篇（极简）

| Paper | 30 秒 |
| --- | --- |
| **AlphaGeometry (DeepMind 2024)** | LLM + symbolic engine + MCTS 解 IMO 几何题。LLM-as-controller 范式 |
| **Process reward + MCTS (rStar)** | small LLM + MCTS search + PRM → math 接近 GPT-4 |
| **Constitutional AI v2** | RLAIF 第二代，加 dpo-style RLAIF |
| **Anthropic harness paper 三连发**（[02 文档]已提）| effective context / multi-agent / coding agent |
| **Llama 3 paper** | 现代 pretrain 工业级 recipe（375 trillion token）|
| **Phi-3** | small model 极致 data curation |
| **OpenAI Practices for Governing Agentic AI** | 治理 / safety / agentic eval |

---

## 八、研究员"日常读 paper SOP"（再贴一次）

```
每天 (1h)：arXiv-sanity / Twitter / DeepLearningAI / RSS
  → scan title + abstract，挑 1 篇深读

每周深读 1~3 篇（90 分钟 / 篇）：
  - 5 分钟：abstract + figure 1 + intro 核心
  - 30 分钟：method + main result
  - 20 分钟：ablation + limitation
  - 20 分钟：写 200 字 personal notes（critical take + 是否复刻）
  - 15 分钟：浏览相关 work / 找 reference

每月复刻 1 篇（10~30h）：
  - 找 minimal config / toy task
  - 复刻 1 个 main figure
  - 找 paper 没说的细节 → 写 blog

每季度尝试主导 1 个研究 thread。
```

---

## 九、面试时"被问最近读什么"的高分答题模板

> **3-part answer**：
> 1. **What** (30 秒)：paper 名字 + venue + author + 1 句核心
> 2. **Why inspired** (60 秒)：哪个具体 idea / figure / finding 让你 think
> 3. **Critical take** (60 秒)：你的批判 / 不同意 / 可改进点
> 4. **Action** (30 秒)：你做了什么 follow up（复刻？写 blog？）

**示例答 GRPO/R1**：

> "我最近重读了 DeepSeek-R1 paper，特别是 5.2 节 GRPO 部分。  *(what)*
>
> 最让我 inspired 是 'aha moment' figure——R1-Zero 训练 6K step 后 'wait, let me reconsider' 这种 self-reflection 模式自发涌现，没人 SFT 教它。这证明 emergent reasoning 不必 distill from 强 model，pure RL 也能学。 *(why)*
>
> 但我有 critical take——paper 没披露 reward 的具体设计，特别是 reasoning task 之外的 OOD generalization；另外 R1-Zero 的 language mixing 问题暗示 pure RL 优化压力过大，'aha moment' 可能是 lucky emergence 而非 robust property。 *(critical)*
>
> 我自己实现了一个 mini GRPO 在 GSM8K 上 finetune Qwen 0.5B，发现 group size = 8 是 sweet spot，且 reward variance 必须 > 某阈值否则 advantage 全 0 train 不动——这个细节 paper 没强调。GitHub repo 在这里：xxx" *(action)*

这套答法瞬间证明：(1) 读过原文；(2) 有独立 critical thinking；(3) 自己跑过实验。

---

## 十、一句应试钥匙

> **不要 "我读过 paper X"——要 "我读了 X 第 N 节 figure M，最让我意外的是 Y，我自己复刻发现 Z paper 没说的细节"。**

> 高分研究员 ≠ 读 paper 最多 = **能在 paper 的 trade-off / failure mode / 未来 direction 上有自己 calibrated 的判断**。
