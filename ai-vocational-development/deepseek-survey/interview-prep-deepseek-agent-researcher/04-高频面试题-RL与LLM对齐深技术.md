# 04｜高频面试题 — RL 与 LLM 对齐深技术篇

> 共 18 道高频深度题。每题给：**面试官想考什么 / 满分回答骨架 / 易踩坑 / 加分点 / 追问**。
>
> 这一篇是研究员面试的"主战场"——预计 60% 一面时间在这。

---

## Q1. 推导 PPO 的 clipped surrogate objective，并解释为什么用 min 而不是直接 clip

### 面试官想考什么
- 是否理解 importance sampling
- 是否理解 trust region 思想
- 是否能讲清楚"clip 不对称"为什么必要

### 满分骨架

```
原始 surrogate (TRPO/REINFORCE++):
  L = E[ ratio(θ) * A ],  ratio = π_θ / π_old

为防止 ratio 暴涨 → 加 trust region：
  KL[π_old || π_θ] ≤ δ

TRPO 用二阶优化解 → 复杂；PPO 改用 ratio clip：
  
  L^CLIP = E[ min(
    ratio * A,
    clip(ratio, 1-ε, 1+ε) * A
  ) ]
```

**为什么 min**：
- 当 A > 0：希望提升该动作概率 → ratio 不应过大 → clip 上界
- 当 A < 0：希望降低该动作概率 → ratio 不应过小（即不能 1-ε 之下还在惩罚） → clip 下界
- **min** 保证 "永远取更保守的目标"——即使 ratio 越过 clip 边界，loss 也不会进一步鼓励

### 易踩
- 把 clip 当对称——其实方向取决于 A 正负
- 忘了 importance sampling 是用 π_old 当 behavior policy

### 加分
- 提到 "PPO is conservative when policy is better, aggressive when policy is worse"
- 提到 GAE 和 advantage normalization 的工程实践

---

## Q2. 推导 DPO loss

### 面试官想考
- 是否能从 KL-regularized RLHF objective 推到闭式解
- 是否理解 partition function Z(x) 怎么消掉

### 满分骨架

```
Step 1: RLHF objective with KL
  max_π E_x,y~π [r(x,y)] - β * D_KL[π || π_ref]

Step 2: 闭式解
  对 π(y|x) 求导 + Lagrangian → 
  π*(y|x) = (1/Z(x)) * π_ref(y|x) * exp(r(x,y)/β)
  Z(x) = Σ_y π_ref(y|x) * exp(r(x,y)/β)

Step 3: 反解 reward
  r(x,y) = β * log[π*(y|x)/π_ref(y|x)] + β * log Z(x)

Step 4: 代入 Bradley-Terry
  P(y_w > y_l | x) = σ(r(x,y_w) - r(x,y_l))
  
  注意：log Z(x) 在 y_w 和 y_l 上都出现且相等 → 抵消
  
  → σ(β * log[π(y_w|x)/π_ref(y_w|x)] - β * log[π(y_l|x)/π_ref(y_l|x)])

Step 5: 极大似然
  L_DPO = -E[ log σ(...) ]
```

### 易踩
- 推到 Z(x) 不会消的——记得 Bradley-Terry 是 reward 之差，Z(x) 自动抵消
- 没有强调 β 既是 KL 系数又出现在 loss 里

### 加分
- 提到 "DPO 隐式定义了一个 reward = β * log(π/π_ref)"
- 提到推导假设 reference policy 固定 + preference 是 Bradley-Terry 分布——这两个假设破了 DPO 就有问题

### 追问
**Q：如果 reference policy 不是最优会怎样？**
A：会出现 **likelihood displacement**——chosen 的 logp 反而降低，只是 rejected 降得更多。IPO/SimPO 在解。

---

## Q3. GRPO 为什么去掉 critic 还能 work？

### 满分骨架

```
传统 PPO 的 critic 干什么用？
- 估 V(s) → 算 advantage A = r - V (or GAE)
- 减小 variance

GRPO 的替代方案：
- 对同一 prompt sample G 个 response
- 用 batch 内 reward 均值当 baseline:
    A_i = (r_i - mean(r)) / std(r)
- 这是 Monte Carlo estimate of advantage

Why work?
1. 当 G 足够大 (典型 G=8~64)，sample mean 是 V(s) 的无偏估计
2. 除以 std 自动归一化 reward scale
3. 对 LLM 这种 episode 很短（一次 generate）的场景，MC 比 GAE 反而更简单
4. 省一个模型 → 节省 ~25% 显存（critic = 7B 时）
```

### 易踩
- 说 "GRPO 完全等价 PPO"——不对，是 "asymptotic"
- 忽略 group size 的重要性

### 加分
- 引 (Yulai et al., 2503.06639) 证明 GRPO policy gradient asymptotically equals oracle critic 的 PPO
- 提到 trade-off：group size 大 → 稳定但 rollout 贵；小 → 信号 noisy

### 追问
**Q：GRPO 有什么 token-level 信号吗？**
A：默认所有 token 共享同 advantage（episode-level）——这是个缺点；改进有 step-level GRPO (e.g., 加 PRM)、Dr.GRPO 等。

---

## Q4. RLHF 训练为什么需要 SFT 阶段？能不能直接 RL？

### 满分
- **SFT 提供良好初始化**——没有 SFT，policy 一开始几乎所有 response 都是 garbage，reward signal 接近全 0，没法学
- **action space 的 prior**：SFT 把模型缩到"可读 text"分布内，RL 才有 exploration 起点
- **R1-Zero 是反例**：DeepSeek 证明在 base model（强 pretrained）+ 简单 rule-based reward 下可以**跳过 SFT**——但 R1 论文也承认 readability 差，所以正式版本还是用 cold-start SFT
- **可解释性**：SFT 提供"风格 baseline"，后续 RL 可对比看效果

### 加分
- 提到"R1-Zero 出现 language mixing 问题——SFT 在 readability/format 上至关重要"
- 提到"未来 base 模型够强可能 SFT 也不需要了"

---

## Q5. KL penalty 在 RLHF 里起什么作用？太大太小分别什么后果？

### 满分

```
KL[π_θ || π_ref] 控制 policy 不要偏离 reference 太远
→ 避免 reward hacking + 保留 base 能力

太大 (β=1.0+):
- policy 几乎不动 → 没学到东西
- KL 变化 << reward 提升
- "锚定" reference 过强

太小 (β<0.001):
- reward hacking 爆发
- policy 输出胡言乱语但 reward 高
- 失去通用能力 (catastrophic forgetting / alignment tax)
- KL 暴涨

实战:
- 一般 β = 0.01 ~ 0.1
- KL adaptive controller：盯 KL 值，超阈值 → 升 β；低阈值 → 降 β
- 监控指标：每步 KL、reward 趋势、SFT eval 准确率不掉
```

### 加分
- 提到 OpenAI Schulman 的"PPO ratios / KL controller"实现细节

---

## Q6. Reward Hacking 是什么？如何 detect 和 mitigate？

### 满分

**定义**：policy 学到"刷分技巧"——输出在 reward model 评分高，但实际质量差。本质是 RM ≠ 真实目标函数。

**典型表现**：
- 输出超长（RM 偏好长 response）
- 重复某个高 reward pattern
- 输出"safe但无用"的 hedge
- 出现 RM 未见过的对抗性 pattern（adversarial inputs）

**Detect**：
1. **Held-out eval**：定期跑 GPT-4 judge / 人工 eval，对比 reward model 分数曲线 vs 真实质量曲线
2. **KL 监控**：KL 异常飙升时通常伴随 reward hacking
3. **Length / repetition metric**：response 长度/重复 token 占比
4. **Reward gap**：训练时 reward 飙升但 eval 不涨 → 警报

**Mitigate**：
1. **KL penalty** + 调高 β
2. **Reward model ensemble**：多个 RM 投票
3. **Iterative RLHF**：每轮重新训 RM（加新数据，包含 policy 当前输出）
4. **Rule-based reward**（DeepSeek-R1 思路）：彻底避开 learned RM 的 hacking 风险
5. **Constrained RL**：明确约束 length / format
6. **Reward shaping**：rewards = quality + α * length_penalty
7. **DPO 替代 RLHF**：offline 数据 → 无 online hacking 空间

---

## Q7. DPO 的 Likelihood Displacement 是什么？怎么 fix？

### 满分

**现象**：训完 DPO 后，**chosen 的 logp 反而降低**——只不过 rejected 降得更多，所以差值仍正。后果：
- 模型不"擅长"生成 chosen
- 训练时 logp 都向下漂 → ref/policy 越来越远
- 对 OOD prompt 容易崩

**原因**：DPO loss 只 care 差值，不 care 绝对 logp。

**Fix（按时间线）**：
1. **加 SFT loss term** (CPO/RPO)：`L = L_DPO + α * L_SFT(y_w)` 让 chosen logp 不要降
2. **IPO** (Azar et al.)：用 squared loss 替 sigmoid，避免 σ 饱和导致梯度漂
3. **SimPO**：去掉 reference + 长度归一，从根上去除 ref 依赖
4. **过滤 preference data**：只保留 chosen 在 ref 下 logp 较高的 pair（避免 chosen OOD）
5. **降低 β / 加 KL 监控**：β 太大反而易塌

### 加分
- 引 (Pal et al., 2024 "Smaug") 或 (Tang et al. "GPO") 等系统分析 paper

---

## Q8. PRM 训练时数据怎么生成？什么是 Math-Shepherd 思想？

### 满分

**手动标**：每步打 0/1（PRM800K，OpenAI）——金标但成本巨大（80 万 step level annotation）。

**Math-Shepherd（自动）**：
1. 给一个 math problem 的解题过程，prefix 到第 k 步 `s_<=k`
2. 从 `s_<=k` 开始 sample N 个 rollout 到 final answer
3. 看这 N 个 rollout 有多少能到正确答案 → 这是 prefix value `V_k`
4. 用 `V_k - V_{k-1}` 估计第 k 步的 reward（advantage estimate）

**OmegaPRM**：用二分搜索定位首个错误 step → 比 Math-Shepherd 高效

**Implicit PRM (PRIME)**：不显式训 PRM，而是直接用 reward model 的 logp 当 step score

### 加分
- 提到 "PRM 的 inter-step credit assignment 是 RLHF 在 reasoning 上下一步的关键"
- 提到 process reward 也可以做 inference-time 引导（best-of-N 时排序）

---

## Q9. RLHF 和 RLAIF 各自适合什么场景？

### 满分

| 维度 | RLHF | RLAIF |
| --- | --- | --- |
| 标注成本 | 高（$1~3/label） | 低（$0.01~0.1/label） |
| 一致性 | 低（标注者差异） | 高（同一个 judge） |
| Bias | 标注者 bias | judge model bias |
| 适合场景 | helpfulness、tone、品牌风格 | harmlessness、format、可规则化 |
| 主流实践 | Anthropic Helpful（RLHF）+ Harmless（RLAIF） |

**Constitutional AI 的核心**：用"宪法原则" + LLM-as-Judge 替代 H 反馈，标注成本 100x 下降。

**Trade-off**：RLAIF 容易出现 **mode collapse + judge bias amplification**——judge 的偏好会越来越被强化。

### 加分
- "DeepSeek-R1 用 rule-based reward 既不是 RLHF 也不是 RLAIF——是 RLVR (RL from Verifiable Reward)，第三条路"

---

## Q10. 解释 RLVR / Verifier-based RL，相比 RM-based 有什么优势？

### 满分

**RLVR (RL with Verifiable Reward)**：reward 来源是**可验证的规则/程序**——
- math: 检查最终数字是否等于 ground truth
- code: 跑单元测试看是否通过
- format: 正则匹配
- length: 字符数

**优势**：
1. **无 reward hacking**——规则是 deterministic
2. **零标注成本**（如果有 ground truth）
3. **可大规模并行**（裁判函数比 RM 推理快 1000x）
4. **可解释**——为什么对/错都能讲清楚

**Limitation**：
- 只适合**可验证的任务**（math, code, structured output）
- helpfulness / harmlessness 这类**主观任务**不行
- 容易让 model "学到走捷径"——比如只学应付 verifier，不真懂

**实践**：DeepSeek-R1 在 math/code 用 RLVR；general scenario 用 mixed reward。

---

## Q11. 解释 Catastrophic Forgetting / Alignment Tax

### 满分

**Alignment Tax**：对齐后模型在某些 benchmark（尤其学术 benchmark）下降——RLHF/DPO 让模型"更人话"但"少 capability"。

**机制**：
- KL penalty 不够强 → policy 漂太远 → pretrained 能力丢
- 偏好数据 distribution 与 pretrain 不同 → SFT 阶段 + RL 阶段都在"窄化"分布
- Mode collapse：sample 多样性下降

**Mitigate**：
1. **KL 调大** + 监控
2. **Mix pretraining data**（replay）—— InstructGPT 论文做法
3. **Regularize with SFT loss**：每 N 步插 SFT 数据
4. **EMA of weights**（少见但有用）
5. **Iterative DPO + 数据轮换**

**最近趋势**：
- "Alignment tax" 在 capability 强的 base 上变小（V3/R1 几乎没掉）
- Anthropic Sonnet 3.5/4 显示对齐与能力可同时提升（不冲突）

---

## Q12. 解释 reward model 训练过程 + 关键挑战

### 满分

**数据**：preference triple `(prompt, response_A, response_B, label ∈ {A>B, B>A, tie})`

**模型**：LLM + linear head → scalar score

**Loss**：Bradley-Terry MLE
```
L_RM = -E[ log σ(s(y_w) - s(y_l)) ]
```

**挑战**：
1. **OOD generalization**：RM 训练时见的是 SFT 输出；RL 时 policy 改了，RM 没见过——需要 iterative
2. **Calibration**：RM score 是 logit，不是 probability—— scalar 之间难比较跨任务
3. **Annotation noise**：人工标注 inter-rater agreement 经常 < 0.7
4. **Reward hacking 漏洞**：RM 见的少 → 容易被 policy 找到漏洞
5. **Length bias**：RM 偏好长 response → policy 越训越长

**实战**：
- 用 model_size = policy_size（如 7B policy + 7B RM）
- 训 RM 时 epoch 不超过 1~2，否则 overfit
- 用 dropout / weight decay 加强 generalization
- Ensemble RM 投票更鲁棒

---

## Q13. 解释 Iterative DPO / Online DPO

### 满分

**标准 DPO 是 offline**：preference data 固定，训完就完了。

**Problem**：当 policy 远离 reference，preference data 反而 stale。

**Iterative DPO**：
1. 用当前 policy sample 一批 response pair
2. 用 reward model / judge / 人工 标新的 preference
3. 用新数据 + DPO 继续训
4. 重复

→ 近似把 DPO 变成 "近 on-policy"

**Online DPO**：直接 streaming——sample + score + update，类似 PPO 的 loop。

**优点**：缓解 distribution shift；接近 PPO 性能。
**缺点**：失去 DPO 的 "纯 offline 简单"卖点；引入 RM/judge → 复杂度回升。

---

## Q14. 解释 SFT → RM → PPO 整套 InstructGPT pipeline 的每步动机

### 满分

| 阶段 | 数据 | 目标 | 动机 |
| --- | --- | --- | --- |
| **SFT** | 高质量 instruction-response | 把 base 模型从 "续写器" 调成 "对话器" | 给 RL 提供 prior，避开 cold-start |
| **RM** | preference pair | 学一个 scalar reward function | 把人工偏好压缩成可微 function |
| **PPO** | prompt only + RM | 优化 policy 让 RM 分高 + KL 不漂 | 把 RM 的判断注入 policy |

**为什么三段**：
- SFT 单独不够——只学 imitation，遇到 OOD 表现差
- RM 单独不够——它只能 score 不能生成
- 需要 PPO 把 RM 的 "判断力" 转化成 policy 的 "生成力"

**演化**：
- DPO 把 RM + PPO 合并 → 跳过 RM
- DeepSeek-R1 用 rule-based reward → 跳过 RM
- 趋势：尽可能简化 pipeline

---

## Q15. Self-Rewarding LM 是什么？有什么 risk？

### 满分

**Self-Rewarding** (Yuan et al., Meta 2024)：
1. 模型既是 actor（生成 response）又是 judge（给 preference）
2. 自己 sample 一批 response → 自己评 preference → 自己 DPO 训
3. 迭代多轮

**优点**：完全无需人工偏好数据。

**Risk**：
1. **Mode collapse / echo chamber**：模型强化自己偏好 → 多样性丢
2. **Bias amplification**：初始 judge bias 被反复放大
3. **Capability ceiling**：judge 能力上限 = actor 当前能力上限——很难真正"超越自己"
4. **Reward hacking**：自我评估有 self-serving bias

**缓解**：
- 混入人工 preference 作 anchor
- Multi-judge / consistency 检查
- 限制迭代轮数

---

## Q16. RLOO / REMAX / REINFORCE++ 这些"轻量 RL"算法的核心动机

### 满分

**动机**：PPO 太复杂——critic 难训、超参敏感、显存大。能不能用更简单的 REINFORCE？

**REINFORCE**：`∇L = E[ (R - b) * ∇log π ]`，b 是 baseline。
- 问题：方差大；选 b 是 art

**RLOO (REINFORCE with Leave-One-Out)** (Ahmadian et al., 2024)：
- sample K 个 response
- 第 i 个的 baseline = 其他 K-1 个 reward 的均值
- 无需 critic，方差更低

**REMAX**：用 greedy decode 当 baseline。

**REINFORCE++**：reinforce + 工程优化（KL/normalization/clipping）→ 比 PPO 更稳。

**洞察**：当 LLM 这种 episode 短 + reward 稀疏的场景，**REINFORCE 家族 + 好的 baseline 通常足够 work，且远比 PPO 简单**。GRPO 也是这条路线的产物。

---

## Q17. 如果 reward 完全错了（比如 RM 训成 negation），训练会怎样？

### 满分（思维题）
- policy 会向 RM 偏好的方向 push → 但 RM 偏好是反的 → policy 越训越差
- KL 一开始可能仍小 → 不会立刻 trigger 监控
- 但 task eval 会立刻掉
- → **必须建立"独立 eval"——用 GPT-4 judge 或 held-out task，不能只看训练 reward**

**真实案例**：Anthropic 文章描述过 RM 训反后 PPO 学会"说脏话"（因为脏话 RM 给高分）的失败实验。

---

## Q18. （开放题）你最近读到一篇 RL/对齐的 paper，最 exciting 的 insight 是什么？

### 满分模板（要准备 2~3 个候选）

```
"最近读到 X paper。

它的核心问题是：[一句话]
关键 insight：[一句话——非 trivial 的部分]
方法：[1~2 句]
为什么我觉得 exciting：[1) 解决了一个长期 pain point  2) 启发我可能可以 transfer 到 Y 场景]
我会怎么改：[1~2 句——展现 research taste]"
```

**推荐候选**：
1. **DeepSeek-R1**：纯 RL 训出 reasoning + GRPO + rule-based reward 三个 insight 叠加
2. **SimPO**：去掉 reference model 同时性能持平 → DPO 真的需要 ref 吗
3. **Agent-RLVR**：guidance + 环境 reward 怎么解 sparse reward
4. **PRIME**：implicit PRM 思想——不显式训 PRM，token-level 信号自动出来
5. **Dr. GRPO**：分析 GRPO bias + 改 normalization

> **务必能讲第 4 层"为什么"。** 这是研究员面试和工程师面试最大区别——你要能进 paper "内部" 而不是停在 abstract。

---

下一文档：[05-高频面试题-Agent训练与前沿研究](05-高频面试题-Agent训练与前沿研究.md)
