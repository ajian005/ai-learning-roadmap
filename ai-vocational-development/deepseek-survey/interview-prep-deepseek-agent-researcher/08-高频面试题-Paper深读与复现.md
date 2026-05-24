# 08｜高频面试题 — Paper 深读与复现篇

> 研究员面试**必考**：会让你讲一篇你最近读过的 paper / 复现过的 paper。**60% 候选人在这一环节挂掉**——不是没读，是只读了 abstract。

> 本文给：1) 如何"深读"一篇 paper 2) 6 篇必读 paper 的 1 页 summary 3) 复现 case study：DeepSeek-R1 / DPO 怎么从 paper 跑到 code

---

## 一、"深读"一篇 paper 的 SOP（必须形成习惯）

### 第 1 遍：5 分钟扫读

1. 读 **abstract + intro 第一段** → 知道 problem
2. 看 **figure 1**（method overview） → 知道 approach
3. 看 **table 1**（main result） → 知道 claim
4. 决定：要不要继续读

### 第 2 遍：30 分钟精读

1. **Method 部分逐句读**——能否复述算法？
2. **数学公式必须 derive**——不能跳过
3. **超参 + 实现细节**——附录里通常藏宝
4. **Ablation**——哪些 component 是真有用？

### 第 3 遍：1 小时反思 + 复盘

写一份 **1 页 summary**（必备模板）：

```
# Paper Title (arxiv id, year)

## Problem
[1~2 句]

## Key Insight  
[非 trivial 的核心想法是什么]

## Method
[Pseudo-code or formula]

## Main Result
[具体 number on 具体 benchmark]

## Why It Works (Hypothesis)
[作者解释 + 你的理解]

## Limitations
[1~3 个]

## What I'd Change
[research taste 体现 — 1~3 个 future work / fix]

## Connection to Other Work  
[和哪些 paper 是 prerequisite / follow-up]
```

### 第 4 遍（可选）：复现关键 component

- 至少 reimplementation 一个 1 页代码的 component
- 形成 GitHub 上 self-doc 的小 demo

---

## 二、必读 7 篇 paper 的 1 页 summary

### Paper 1: DeepSeek-R1 (2501.12948) ⭐⭐⭐

**Problem**：能不能不用 SFT 也不用 reward model，只用 RL + rule-based reward 训出 reasoning model？

**Key Insight**：在足够强的 pretrained base 上，rule-based 0/1 reward + GRPO 能让模型**自发涌现**长思考、self-reflection 等行为（"aha moment"）。

**Method**：
- **R1-Zero**：DeepSeek-V3-base + GRPO + (accuracy reward + format reward)
- **R1**：先 cold-start SFT (CoT 数据) → RL for reasoning → rejection sampling + SFT (600K reasoning + 200K general) → 第二轮 RL (all scenarios)

**Main Result**：
- R1-Zero on AIME 2024: 15.6% → 71.0%（仅 RL，无 SFT）
- R1 on AIME 2024: 79.8%（pass@1，与 o1-1217 平手）
- R1 on MATH-500: 97.3%

**Why It Works**：
- 强 base 提供良好"思考能力" prior
- Rule-based reward 无 hacking 空间
- GRPO 省 critic 让 RL 更稳

**Limitations**：
- 仅适合可验证任务（math/code）
- R1-Zero readability 差（language mixing）
- 训练数据 / infra 未开源

**What I'd Change**：
- 加 PRM 信号缓解长 trajectory credit assignment
- 探索在 agent task（如 SWE-Bench）上扩展
- 研究 RL signal 在 helpfulness 等 non-verifiable 域上的迁移

**Connection**：
- 前置：PPO (1707), GRPO 概念在 DeepSeekMath (2402)
- Follow-up：Open-R1, TinyZero, DeepSeek-R1 改进 paper

### 高频追问

**Q：为什么 R1-Zero 出现 "aha moment"？**
A：RL 在 math 任务上 reward 正确，自然推动模型 explore 更长思考。当某次 sample 因为 "reconsider" 答对，advantage 高 → policy 学到此 pattern → 强化循环。

**Q：R1 vs R1-Zero 谁更值得复现？**
A：R1-Zero 更纯净（无 SFT 数据依赖），更适合学术复现；R1 需大量精标 CoT 数据，门槛高。

---

### Paper 2: DPO (Rafailov et al., 2305.18290) ⭐⭐⭐

**Problem**：RLHF 复杂（SFT → RM → PPO 三阶段 + online sampling）。能否简化？

**Key Insight**：KL-regularized RLHF objective 有闭式解 → reward 可以反解为 log(π/π_ref) → preference 模型 → classification loss。**不需要显式 RM + 不需要 RL sampling**。

**Method**：
- Loss: `L = -E[ log σ(β · (log π/π_ref(y_w) - log π/π_ref(y_l))) ]`
- Pure offline，类似 SFT 的训练循环

**Main Result**：
- Anthropic Helpful & Harmless 数据集上 win rate ≈ PPO+RM
- TL;DR summarization 上 win rate 略胜
- 训练快 ~10x，简单 ~10x

**Why It Works**：DPO 实际是在做 implicit reward 上的 PPO 等价物。Bradley-Terry assumption + KL 闭式解 → 公式化等价。

**Limitations**：
- Offline——错过 online exploration 收益
- Likelihood displacement（chosen logp 也降）
- 对 reference model 强依赖
- 假设 preference 是 Bradley-Terry 分布（实际人类 noisy）

**What I'd Change**：
- IPO / SimPO / KTO 等已 propose 改进
- 加 SFT regularization（CPO/RPO）
- Iterative DPO 解 offline 问题

---

### Paper 3: PPO (Schulman et al., 1707.06347)

**Problem**：TRPO 复杂（二阶优化、约束求解）；REINFORCE variance 大。要简单高效的 policy gradient。

**Key Insight**：用 ratio clip 近似 trust region，避免二阶。`clip(r, 1-ε, 1+ε)` 的 min 操作保证 policy 不要"激进 update"。

**Method**：
- `L^CLIP = E[ min(r·A, clip(r, 1-ε, 1+ε)·A) ]`
- `L^total = L^CLIP - c1·L^VF + c2·L^Entropy`

**Main Result**：Atari、MuJoCo、Roboschool 上接近 SOTA，**显著比 A2C/TRPO 简单**。

**Why It Works**：
- Clip 限制 ratio 不漂太远 → 近似 trust region
- Multi-epoch on same rollout → sample efficiency

**Limitations**：
- 超参敏感（clip_eps, KL 系数）
- LLM 上需要 critic warmup
- Sample 仍偏多

**Connection**：所有后续 RLHF（InstructGPT/Claude/GPT-4）几乎都基于 PPO。

---

### Paper 4: Let's Verify Step by Step (OpenAI, 2305.20050)

**Problem**：reasoning task 上 ORM 信号 sparse。能否用 step-level supervision？

**Key Insight**：在每个 reasoning step 标 0/1（PRM800K），训 PRM；inference 时用 PRM 重排 best-of-N → 显著超过 ORM。

**Method**：
- 数据：MATH 数据集，人工标注每个 reasoning step（80 万 step）
- PRM：transformer + token-level classifier
- Inference：sample N → 每个 trajectory PRM 打分（min / product / sum）→ 选最好

**Main Result**：
- MATH 上 PRM > ORM 显著（如 78% vs 72% on best-of-256）
- 不需要 ground truth answer 即可重排

**Why It Works**：sparse vs dense reward 的经典论证——dense reward 在 reasoning task 上更准确

**Limitations**：
- 80 万 step 标注成本极高
- PRM 训练数据分布 vs inference 时 policy 分布 mismatch

**What I'd Change**：用 Math-Shepherd 思想自动生成 PRM 数据（已有 follow-up）

---

### Paper 5: Constitutional AI (Bai et al., 2212.08073)

**Problem**：RLHF 需要大量人工 harmful label，成本高 + 标注员心理负担大。

**Key Insight**：用一组"宪法原则" + LLM 自我 critique → RLAIF 替代人工 harmless 标注。

**Method**：
- **SL-CAI**：sample harmful prompt → model response → 用 constitution prompt 让 model self-critique + revise → SFT on revised data
- **RL-CAI**：用另一个 model 做 preference judge → 训 RM → PPO

**Main Result**：
- Harmlessness comparable to RLHF-trained Claude
- Helpfulness 几乎不掉
- 标注成本下降 ~100x

**Why It Works**：harmlessness 的"规则"比 helpfulness 更容易写下；LLM judge 在规则化任务上能力够

**Limitations**：
- Constitution 写作本身需 expertise
- Helpfulness 仍用 RLHF（CAI 不替代）
- Judge model 的 bias 会被强化

---

### Paper 6: Agent-RLVR (Lu et al., 2506.11425)

**Problem**：SWE-Bench 这类长 horizon agent task reward 极稀疏（只有最终 patch 是否过测试），RL 难学。

**Key Insight**：加 "guidance reward"——一个 teacher（如 GPT-4）给出 step-level 建议，agent 步骤跟 guidance 一致就额外加 reward。**把稀疏 reward 转密**。

**Method**：
- 环境 reward = patch 是否过测试（0/1）
- Guidance reward = teacher 给的 step-by-step plan，agent action 与 plan 相似度
- 总 reward = env + α × guidance
- GRPO 训练

**Main Result**：Qwen-2.5-72B on SWE-Bench Verified 9.4% → 22.4%（pass@1）

**Why It Works**：guidance 把 long-horizon 稀疏 reward 转成 step-level dense reward；agent 早期能从 imitation learning 中加速

**Limitations**：
- Teacher quality bound——guidance 错了 agent 也错
- 训练后期 guidance 收益递减
- Computation overhead

**What I'd Change**：动态 anneal guidance weight；从全程 guidance 逐渐过渡到纯 env reward

---

### Paper 7: SimPO (Meng et al., 2405.14734)

**Problem**：DPO 的 reference policy 是 baseline——储存 + compute 都是负担。

**Key Insight**：在 DPO loss 里去掉 reference model；用**长度归一化** + 一个 margin term 平衡。

**Method**：
- Loss: `L = -E[ log σ((β/|y_w|)·log π(y_w|x) - (β/|y_l|)·log π(y_l|x) - γ) ]`
- 关键改动：
  1. 去掉 `log π_ref` 项
  2. 除以长度（避免长度 bias）
  3. 加 margin γ 强制更大 separation

**Main Result**：AlpacaEval 2 / Arena-Hard 上 SimPO > DPO 显著；训练 memory 减半

**Why It Works**：argues that reference 在 DPO 里其实是 length proxy；显式归一化更直接

**Limitations**：
- 仍 offline
- γ 需调
- 没有 ref 时可能某些情况下 policy 漂得更远

---

## 三、复现案例：从 DPO paper 到自己跑通

### Phase 1: 读论文 + 1 页 summary（半天）

按上面的 SOP 写出 1 页。

### Phase 2: 找开源参考（半天）

- TRL 的 `DPOTrainer`（HuggingFace 官方）
- Eric Mitchell 的原版 https://github.com/eric-mitchell/direct-preference-optimization
- Open RLHF 等

### Phase 3: 准备数据 + Base model（1 天）

- Base model: Qwen-0.5B-Chat 或 Phi-3-mini（能本地跑）
- Data: Anthropic HH-RLHF subset（10K pair 够）
- 算 reference logp（重要：要 cache 不然每次重算）

### Phase 4: 实现 + smoke test（1 天）

```python
# 最小可运行版
from trl import DPOTrainer
from transformers import AutoModelForCausalLM, AutoTokenizer
from datasets import load_dataset

model_name = "Qwen/Qwen2.5-0.5B-Instruct"
model = AutoModelForCausalLM.from_pretrained(model_name)
ref_model = AutoModelForCausalLM.from_pretrained(model_name)
tokenizer = AutoTokenizer.from_pretrained(model_name)

dataset = load_dataset("Anthropic/hh-rlhf", split="train").select(range(1000))

trainer = DPOTrainer(
    model=model,
    ref_model=ref_model,
    args=DPOConfig(
        learning_rate=5e-7,
        per_device_train_batch_size=2,
        beta=0.1,
        num_train_epochs=1,
    ),
    train_dataset=dataset,
    tokenizer=tokenizer,
)

trainer.train()
```

### Phase 5: 收敛 + 可视化（1 天）

- 跑 1000 steps，记录：
  - `loss`
  - `reward_chosen_mean`
  - `reward_rejected_mean`
  - `reward_acc` (should approach 1.0)
  - `reward_margin` (should grow)
- 用 W&B / TensorBoard plot

### Phase 6: 评估（半天）

- 跑 MT-Bench / AlpacaEval 子集
- 对比 base vs DPO-trained 的 win rate（用 GPT-4 judge）

### Phase 7: 写 README + blog（半天）

- 训练 setup
- 关键 hyperparameter
- 训练 curve 截图
- Lessons learned
- 链接到 GitHub repo

### 整体投入：~5 天（pure work）

**这就是 mini DPO 项目**——见 [12 文档] 详细蓝图。

---

## 四、面试中讲 paper 的最佳模板

```
"我最近读到 X paper。

[15 秒] Problem: 它在解 ... 这是一个长期困扰 ... 的问题
[20 秒] Key Insight: 它的核心新颖性是 ... 这和之前的 Y / Z paper 不同
[30 秒] Method: 简单说就是 ... (公式 / 流程)
[15 秒] Result: 在 X benchmark 上 ... 提升
[20 秒] What I think: 我特别欣赏 ... 但我觉得 limitation 是 ... 如果我重做我会 ..."

总 100 秒。然后停下，让面试官问。

警惕:
- 不要超 2 分钟独白
- 不要把所有细节倒出
- 留 hook 让面试官追问
```

---

## 五、面试官追问的典型 4 层

```
L1: "你能讲一下 X paper 吗?"  (basic)
    → 上面 100 秒模板

L2: "为什么作者用 Y 不用 Z?"  (rationale)
    → 答出 trade-off

L3: "如果改 N 你认为会怎样?"  (research taste)
    → propose hypothesis + 验证方法

L4: "你认为 paper 哪里是 weak claim?"  (critical thinking)
    → 找 limitation + 给 evidence (e.g., "他们只在 7B 上证 X 但 70B 上未必")

L5: "你来 DeepSeek 想做这个方向，第一周做什么?"  (actionable)
    → 见 [05 文档] Q13 模板
```

---

## 六、研究员的 "paper habit" 长期建设

```
每天 (30 min):
- 扫 arxiv-sanity / HF daily papers 标题
- 标记 1~2 篇感兴趣的

每周 (3 hours):
- 精读 1 篇必读
- 写 1 页 summary 进 wiki
- Tweet / blog short 短文（可选）

每月 (1 day):
- 跑通 1 个开源 repo（哪怕只 demo）
- 整合 4 周笔记 → 1 篇 monthly digest

每季度 (3 days):
- 选 1 个方向做 deep dive（综述 + 复现 + 思考）
- 输出 1 篇技术 blog 或 GitHub repo

每年:
- 至少 1 篇自己的 paper / blog 上 arxiv / GitHub
- 1 次 reading group / 学术分享
```

---

## 七、面试官眼里的"会读 paper" vs "不会读"

| 信号 | 会读 | 不会读 |
| --- | --- | --- |
| 能讲 method | 用公式 / pseudo-code | 只讲 idea |
| 能讲 result | 具体数字 + benchmark | "效果很好" |
| 能讲 limitation | 主动提 + 给 evidence | "我没看到 limitation" |
| 能讲 follow-up | 知道 X paper 是 Y 的改进 | 孤立看一篇 |
| 能 critique | 提 weakness + improve idea | 全盘接受 |
| 能 connect | 联系 3+ 篇相关 paper | 只知道这一篇 |

---

下一文档：[09-行为与研究协作篇](09-行为与研究协作篇.md)
