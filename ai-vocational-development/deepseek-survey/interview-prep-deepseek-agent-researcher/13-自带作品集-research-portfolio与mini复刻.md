# 13 · 自带作品集：Research Portfolio + Mini RLHF 复刻 + 技术博客

> 研究员面试的"硬通货 trio"：
> 1. **1 页 Research Portfolio**（让面试官 5 分钟内 grok 你的研究品味）
> 2. **1 个 GitHub mini RLHF 复刻项目**（证明你能 0→1 ship 实验）
> 3. **1~3 篇技术博客**（证明你能 articulate 思想）
>
> 这 3 件事是顶校博士级 candidate 也要做的功课——你做完，比一般"我知道 PPO"的应聘者强 10×。
>
> 推荐 T-14 ~ T-3 之间完成。

---

## PART A·Research Portfolio（1 页 PDF）

### 目的
- 面试前 1 周邮件给 HR / hiring manager
- 面试当天打印 3 份带去
- LinkedIn / GitHub README 链接

### 1 页布局

```
┌────────────────────────────────────────────┐
│  [姓名] · Research Portfolio · 2026         │
│  [Email] · [GitHub] · [Personal site]      │
├────────────────────────────────────────────┤
│                                            │
│  RESEARCH INTEREST (3 行)                   │
│  RL Alignment / Agent Capability / Process │
│  Reward / Open-source LLM Frontier         │
│                                            │
├────────────────────────────────────────────┤
│                                            │
│  PUBLICATIONS / PREPRINTS (如有)            │
│  ★ Paper A: "X" (NeurIPS/ICML/ACL 20YY)    │
│    First/co-first author. arXiv:XXXX       │
│  • Paper B: "Y" (workshop / preprint)      │
│                                            │
├────────────────────────────────────────────┤
│                                            │
│  KEY PROJECTS (3 个，每个 3-5 行)            │
│                                            │
│  1. mini-grpo-replication                   │
│     - 复刻 DeepSeek R1 GRPO 在 Qwen-0.5B     │
│     - GSM8K acc 35% → 60%; group size sweep │
│     - 200+ GitHub stars; blog 引用 1K+     │
│     - github.com/you/mini-grpo              │
│                                            │
│  2. rm-drift-investigation                  │
│     - 发现 RLHF 训练 step 4500+ RM-eval gap  │
│     - 提出 RM refresh strategy 提升 correl   │
│       从 0.4 → 0.78                         │
│     - Internal tech report (公司认可)        │
│                                            │
│  3. process-reward-mining-poc               │
│     - 用 MC rollout 自动 mine PRM label      │
│     - 在 MATH 上 reproduce MathShepherd 80% │
│     - 写成 blog: yoursite.com/prm           │
│                                            │
├────────────────────────────────────────────┤
│                                            │
│  TECHNICAL DEEP-DIVES (blog 链接)           │
│  • "Understanding GRPO: From PPO to        │
│     DeepSeek's Math Reasoning Engine"      │
│  • "RM Drift in Long RLHF Runs: Diagnosis  │
│     and Mitigation"                        │
│  • "Process Reward Mining with Monte Carlo │
│     Rollout: A Practical Guide"            │
│                                            │
├────────────────────────────────────────────┤
│                                            │
│  TECHNICAL STACK                            │
│  ML: PyTorch / TRL / OpenRLHF / DeepSpeed  │
│  Lang: Python / C++ / CUDA                 │
│  Infra: SLURM / K8s / W&B / MLflow         │
│                                            │
├────────────────────────────────────────────┤
│                                            │
│  EDUCATION                                  │
│  PhD/MS [校] [年] · Advisor: [name]         │
│  Thesis: "..."                              │
│                                            │
│  EXPERIENCE                                 │
│  [Lab/公司] · Research [intern/scientist]   │
│  · [年]-[年]                                │
│                                            │
└────────────────────────────────────────────┘
```

### 关键设计原则
- **极简密度**：1 页装下所有；用空白引导眼球
- **数字驱动**：每个项目带 metric（"35% → 60%" 比 "improved" 强 10×）
- **链接 actionable**：GitHub / blog 都可点
- **顶会一作首位**：如有，置 PUBLICATIONS 第一行 + ★ 标记
- **如无顶会论文**：用 mini 复刻 + blog + GitHub 补——**有 published paper 是 nice-to-have，不是 must-have**

---

## PART B·Mini RLHF 复刻项目

### 项目 spec：`mini-rlhf-toolkit`

**目标**：
- 1 个 ~500-1500 行 Python 项目
- 包含 PPO / DPO / GRPO 3 个核心算法实现
- 在 toy task / small model 上跑通
- 1 个 main figure 复刻自顶 paper
- 完整 README + blog

**Repo structure**：
```
mini-rlhf-toolkit/
├── README.md                    # 主入口 + sample result
├── LICENSE                      # MIT
├── pyproject.toml               # uv / poetry config
├── src/
│   ├── algorithms/
│   │   ├── ppo.py               # PPO loss + GAE
│   │   ├── dpo.py               # DPO loss
│   │   └── grpo.py              # GRPO loss
│   ├── models/
│   │   ├── value_head.py
│   │   └── reward_model.py
│   ├── data/
│   │   ├── preference_dataset.py
│   │   └── math_dataset.py
│   └── trainers/
│       ├── ppo_trainer.py
│       ├── dpo_trainer.py
│       └── grpo_trainer.py
├── experiments/
│   ├── 01_ppo_cartpole.py       # sanity check
│   ├── 02_dpo_qwen_05b.py       # toy LLM DPO
│   ├── 03_grpo_gsm8k.py         # main: GRPO on GSM8K
│   └── 04_ablation_group_size.py
├── notebooks/
│   ├── reproduce_r1_aha.ipynb
│   └── compare_ppo_vs_grpo.ipynb
├── results/
│   ├── grpo_acc_curve.png
│   └── group_size_ablation.png
└── tests/
    └── test_loss_functions.py
```

### V0 (T-14 ~ T-10)·搭骨架

**Day 1-2**：
- Repo init + 基础结构
- Loss 函数 implement (见 [08 文档])
- Unit test loss 数学正确

**Day 3-4**：
- PPO on CartPole 跑通 (sanity)
- DPO on Qwen-0.5B + UltraFeedback (mini subset)

### V0.5 (T-10 ~ T-7)·核心实验

**Day 5-7**：
- GRPO on GSM8K + Qwen-0.5B
- Group size sweep (4, 8, 16, 32)
- Reward shaping 调试
- 跑出第一个 figure：acc vs step

### V1.0 (T-7 ~ T-3)·polish + blog

**Day 8-10**：
- README 完善
- 主图复刻 (R1 paper figure 2-3 类似 emergence curve)
- 写 1 篇 blog "Reproducing GRPO on small models"

### V1.1 (T-3 ~ T-0)·宣传

**Day 11-14**：
- Tweet/LinkedIn 分享
- 提交到 Hugging Face Trending（可选）
- 准备面试 demo（jupyter notebook 形式 show）

### README template

````markdown
# mini-rlhf-toolkit

A minimal, educational implementation of PPO/DPO/GRPO for LLM RLHF.

## Why

Most production RLHF frameworks (TRL, OpenRLHF, verl) hide complexity behind
abstractions. This toolkit is a **single-author readable** implementation
that reproduces the key intuitions of:
- PPO (Schulman 2017)
- DPO (Rafailov 2023)
- GRPO (DeepSeek 2024)

## Featured Results

### GRPO on GSM8K (Qwen-0.5B)

![](results/grpo_acc_curve.png)

- Baseline (SFT only): 35% acc
- + GRPO 5K steps: 60% acc
- Group size sweep: 8 is sweet spot (see ablation)

### "Aha moment" reproduction

After step 3K, model spontaneously emerges "let me reconsider" pattern
on hard problems—similar to DeepSeek R1 paper figure 2.

[Notebook: notebooks/reproduce_r1_aha.ipynb]

## Quick Start

```bash
pip install -e .
python experiments/03_grpo_gsm8k.py --model Qwen/Qwen2.5-0.5B \
    --group_size 8 --kl_coef 0.04
```

## Implementation Notes

- PPO uses Schulman unbiased KL estimator
- DPO loss masks prompt tokens (only response tokens count)
- GRPO normalizes group advantage with std (not just mean)
- All losses are sequence-level for LLM (not token-level PPO original)

## Lessons Learned

1. **Reward must have variance**: rule-based 0/1 reward fails when group
   has all-correct or all-wrong. Add partial credit.
2. **KL estimator matters**: naive log ratio can be negative; Schulman's
   `ratio - log ratio - 1` is unbiased + non-negative.
3. **Group size = 8** is the sweet spot for 0.5B model—larger groups give
   diminishing returns; smaller has noisy advantage.

## Citation

If this helped your understanding, cite:
```bibtex
@misc{yourname2026minirlhf,
  title={mini-rlhf-toolkit},
  url={https://github.com/you/mini-rlhf-toolkit},
  year={2026}
}
```

## References

- Schulman et al. 2017. PPO.
- Rafailov et al. 2023. DPO.
- DeepSeek-Math 2024. GRPO.
- DeepSeek-R1 2025. RL for reasoning.

## License

MIT
````

---

## PART C·技术博客（3 篇推荐）

### Blog 1·"Understanding GRPO: From PPO to DeepSeek's Math Reasoning Engine"

**结构**（~2500 words）：

```
1. Why this blog now (R1 + GRPO 在火，但缺 didactic 解释)
2. From REINFORCE to PPO (1 段)
3. From PPO to GRPO (重点)
   - 公式对比 (PPO vs GRPO)
   - 为什么 group baseline 比 critic 好
   - Schulman KL estimator
4. Code walk-through
   - 30 行 GRPO loss
   - 关键 trick
5. My experiment: GRPO on Qwen-0.5B
   - Group size sweep
   - Reward shaping
   - Aha moment observation
6. Limitations & open questions
7. References
```

**发布渠道**：
- 个人 blog (Hugo / Jekyll / Substack)
- 转发 Twitter/LinkedIn
- Hugging Face Community

### Blog 2·"RM Drift in Long RLHF Runs: Diagnosis and Mitigation"

**核心 finding**：你自己实验观察到的 RM-eval gap 在 step 4500+。

**结构**：
```
1. The phenomenon (curve图)
2. Hypothesis (OOD shift)
3. Diagnosis (cosine distance)
4. Mitigation (RM refresh)
5. Open questions
```

### Blog 3·"Practical Process Reward Model Mining with Monte Carlo"

**核心**：复刻 MathShepherd 的免人标 PRM 思路。

**结构**：
```
1. PRM800K vs auto-mining (cost trade-off)
2. MC rollout details
3. Code: rollout + label aggregation
4. Toy experiment (math task)
5. Findings + open questions
```

---

## PART D·"如果我没有顶会论文" 的替代方案

> 大多数 candidates 没顶会一作——这是 OK 的。下面是高质量替代：

| 替代 | 等价 signal | 难度 |
| --- | --- | --- |
| **顶 venue workshop 论文** | ~50% of 主会 | 中 |
| **arXiv preprint + 数百引用** | 视质量等价 | 中 |
| **顶级 lab intern + acknowledgement** | 工作经历 signal | 高 |
| **OSS major contribution** (vLLM/TRL/transformers core PR) | 工程能力 signal | 中 |
| **GitHub mini 复刻 + 100+ star** | 实操能力 signal | 中 |
| **技术博客 + Twitter influence** | thought leadership signal | 低-中 |
| **HackerNews / 知乎 高赞技术文** | community signal | 低 |

**推荐组合**（无顶会论文的 candidates）：
1. mini RLHF 复刻 (GitHub 100+ star 目标)
2. 2-3 篇深度技术博客
3. 1 个 OSS PR（哪怕是 TRL 的 minor bug fix）
4. 活跃 Twitter/知乎（500+ followers）

这套组合在 6 个月内可建立——比追顶会一作快得多，signal 仍强。

---

## 最后忠告

> **研究员作品集的真正价值不是"展示成就"——是 "证明 you think and ship like a researcher"。**

> 1 篇顶会一作 ≈ 1 个 well-executed 复刻 + 1 篇有 critical insight 博客 + 1 个 honest negative result 反思。

> **任何顶级 lab 招人最看重的 3 件事**：
> 1. **能不能独立 0→1**（mini 复刻 prove it）
> 2. **有没有 research taste**（blog 的 critique quality prove it）
> 3. **能不能持续 ship**（GitHub commit cadence prove it）

> 这 3 件你在 T-14 ~ T-3 这 2 周里能 ship 出来——比刷一年 LeetCode 价值高 100×。
