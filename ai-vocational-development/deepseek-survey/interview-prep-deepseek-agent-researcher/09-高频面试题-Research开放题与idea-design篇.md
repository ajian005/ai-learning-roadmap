# 09 · 高频面试题·Research 开放题与 Idea Design 篇

> **研究员面试最具区分度的题型**——给你一个 vague 问题，让你 30 分钟里 **propose hypothesis + design ablation + 想 failure mode**。
>
> 这一份给 8 道经典开放题 + 标准答题模板 + 每题 1 个高分示范 + 4 个 framework。

---

## 一、答题"7-step framework"（必背）

> **任何 open research 题，按 7 步答。10~30 分钟。**

```
1. CLARIFY (2 min)
   - 复述题目；锁定 ambiguous term
   - "你说的 X 是指 Y 还是 Z？"
   - "成功标准是什么？benchmark 还是 win-rate 还是 cost？"
   - "我有什么 budget / compute / data 限制？"

2. SCOPE (3 min)
   - 列 2-4 个 sub-problem
   - "我看到至少 3 个 sub-question："
     - 是 algorithmic limitation 还是 data limitation
     - 是 train-time 还是 inference-time
     - 用 SFT/RL/prompt 哪个 lever

3. HYPOTHESIS (5 min)
   - 列 3 个 hypothesis (不要只 1 个)
   - 每个 hypothesis 给 reasoning
   - 排序：哪个最 likely / 最 testable

4. EXPERIMENT DESIGN (10 min)
   - 选 top 1~2 hypothesis 设计 minimal ablation
   - 给出：dataset / model / metric / control / treatment
   - 写预期 result table

5. EXPECTED RESULT (3 min)
   - 各 hypothesis 真假对应不同 outcome
   - "如果看到 X，说明 hypothesis 1 真；看到 Y，说明 hypothesis 2 真"

6. FAILURE MODE (3 min)
   - 实验可能 fail 的 3 个原因
   - 怎么 detect / mitigate

7. NEXT STEP (2 min)
   - 假如结果 positive，下一步
   - 假如 negative，pivot 方向
```

---

## 二、8 道经典开放题

### Q1·"模型在 long reasoning 任务上失稳——你怎么 investigate？"

**示范答案**：

**1. CLARIFY**：
> "我想 clarify 几点：'长 reasoning' 是说思考 token 数 > 8K 还是 > 32K？'失稳' 是说 accuracy 降，还是 latency 不可控，还是 hallucinate？baseline 模型是什么？eval 用什么 benchmark？"

**2. SCOPE**：
> "我看到 3 个可能维度：
> (a) **训练数据 distribution** —— long CoT data 不够
> (b) **训练 objective** —— 没有 process reward / 只有 outcome
> (c) **inference 机制** —— position interpolation / KV cache 实现问题"

**3. HYPOTHESIS**（排序）：

| # | Hypothesis | 为什么 likely | Testability |
| --- | --- | --- | --- |
| 1 | Position embedding 在 OOD 长度上 fail | high (RoPE scaling 经典坑) | easy (插值 vs 训练长度对比) |
| 2 | Outcome reward 不给中间 step credit → 长链路梯度稀疏 | high | medium (需 retrain) |
| 3 | KV cache numerical drift（FP16/BF16）长 seq 累积误差 | low | medium (FP32 试) |

**4. EXPERIMENT**（选 H1 + H2）：

H1 实验：
```
Setup: 同模型，三种 position encoding setting
- Vanilla RoPE
- RoPE + NTK scaling
- YaRN
Eval: GSM8K-Long（人造长 CoT 变体）+ AIME
Metric: accuracy vs reasoning_length
Expected: 如果 H1 真，accuracy 在 train_length 后骤降；NTK/YaRN 显著 fix
```

H2 实验：
```
Setup: 同模型，对比训练 objective
- Baseline: outcome reward (binary)
- Treatment: outcome + process reward (用 PRM)
Eval: 同上
Expected: 如果 H2 真，process reward 在 long task 上 +5pp，short task 不变
```

**5. EXPECTED RESULT TABLE**：
```
          | GSM8K (short) | GSM8K-Long | AIME |
Baseline  |   85%         |   60%      | 30%  |
+ YaRN    |   85%         |   75%      | 35%  |
+ PRM RL  |   86%         |   72%      | 38%  |
+ both    |   86%         |   80%      | 42%  |
```

**6. FAILURE MODE**：
> 1. **PRM 自己 fail** —— PRM 在 long context 上估值差 → 改用 rule-based 中间 check
> 2. **Data confound** —— 加 process reward 数据顺带加了 distribution；要 control "data 量相同，仅 objective 不同"
> 3. **Eval saturate** —— 如果 baseline GSM8K 已经 90%+，提升空间小

**7. NEXT STEP**：
> Positive: 找最 promising config 在 R1-sized model 上 scale
> Negative: pivot 到 inference-time（test-time scaling）方向

---

### Q2·"小模型 (1B) 怎么具备 Agent tool-use 能力？"

**HYPOTHESIS 列表**：

| # | Hypothesis | Approach |
| --- | --- | --- |
| 1 | Tool-use 主要靠 format learning，SFT 够 | SFT on Agent-FLAN dataset |
| 2 | 需 distillation from 70B+ teacher | distill from Claude/R1 trajectory |
| 3 | RL with verifiable tool feedback 可超越 SFT | GRPO on rule-based tool reward |
| 4 | 1B 不够 capacity，必须用 retrieval + tool | RAG + tool plugin 而非 internalize |

**MINIMAL ABLATION**：

```
4 个 1B model variants:
A) base + SFT on tool data (10K traj)
B) base + distill from R1 (50K traj)
C) A + GRPO on tool benchmark
D) base + RAG (cheat: 不内化 tool, retrieve docs)

Eval: ToolBench / NexusBench / BFCL
Metric: success rate, avg # tool calls, format error rate
```

**EXPECTED**：
- B > A (distillation 给上限)
- C > B (RL polish)
- D 比 C 简单但 fragile in OOD task
- 结论：if 优先 simplicity → D; if 追 capability → B+C

**FAILURE MODE**：
- 1B model context 太短 → 多 tool sequence 超 limit
- Distill from R1 学 reasoning 但 dilute tool capacity

---

### Q3·"训出的 reasoning model 推理时间太长，怎么 budget aware？"

**HYPOTHESIS**：

1. 模型不知道何时停 → 加 "stop when confident" SFT signal
2. Reward 没 length penalty → 加
3. 缺 routing：简单题不该 think → train classifier
4. CoT 本身冗余 → 学 compact CoT

**EXPERIMENT 1 (H2 length-penalty)**：

```
Compare 2 RL setups:
- baseline: reward = correctness
- treatment: reward = correctness - λ * length

λ sweep: [0, 0.001, 0.005, 0.01]
Eval: AIME accuracy vs avg thinking tokens

Expected curve: accuracy_drop vs avg_token_save trade-off
```

**EXPERIMENT 2 (H3 routing)**：

```
Train a tiny classifier (50M model) on (prompt → think_or_not).
Data: 1K simple + 1K hard prompts.
At inference: if classifier predicts "no need to think", skip thinking phase.

Metric: total cost saving + accuracy delta on mixed eval
```

**FAILURE MODE**：
- Length penalty 太强 → model 学 truncate mid-thinking → accuracy 崩
- Routing classifier overfit train distribution → OOD wrong call

---

### Q4·"Reward hacking 突然在 trajectory 4500 step 出现，怎么 debug？"

**框架**：reward hacking forensics

**Step 1·收集 evidence**：
- 看 reward curve（是否有突变）
- Sample 10 个 high-reward trajectory，人看
- 看 length / format / specific keyword distribution shift

**Step 2·分类 hacking 模式**：

| 模式 | signal | 例 |
| --- | --- | --- |
| Verbose | length spike | response 从 200 token → 1000 |
| Sycophantic | "you're right" 频率↑ | "great question!" 暴增 |
| Format | markdown / bullet 暴增 | 不必要的 bullet list |
| Repetition | n-gram repeat ↑ | "Therefore... Therefore... Therefore..." |
| Specific phrase | RM bias keyword | "in conclusion" / "step by step" |

**Step 3·mitigation**：

1. **Rollback** 到 step 4400 ckpt
2. **加入 adversarial preference data**——故意标 hacking pattern 为 rejected
3. **KL constraint 增强**——把 β 从 0.02 → 0.05
4. **RM refresh**——用新 policy generation 标 new preference data

**Step 4·prevent**：

- 维护 canary set（健康 distribution markers）
- Reward auto-monitor：reward ↑ 但 holdout eval ↓ → alert
- Periodic human spot check：每 500 step 抽 50 traj 人看

**FAILURE MODE of mitigation**：
- KL 太强 → 学不动；DPO refresh data noisy

---

### Q5·"给你 1000 万 GPU hour，要 push DeepSeek Agent capability，你怎么分配？"

**STRATEGY**：

```
40% (400 万) - Pretrain Agent-flavored corpus
   - 加 code repo data 30%
   - Tool documentation / SDK 10%
   - Agent trajectory (open-source SWE-Agent etc.) 10%
   - 训一个 7B base 作 ablation

30% (300 万) - RL on capability bottleneck
   - 数学 + code GRPO (用 verifiable reward)
   - tool use GRPO (sandbox env)
   - 长 horizon agent task

20% (200 万) - Eval infra + benchmark expansion
   - rebuild SWE-Bench 中文版
   - 自建 multi-modal agent benchmark
   - LLM-as-judge calibration

10% (100 万) - Exploration / risky bets
   - Process reward auto-mining
   - Long horizon credit assignment
   - novel RL algorithm prototyping
```

**Rationale**：
- 40% pretrain 因为 base data distribution 决定 ceiling
- 30% RL 因为这是当前 alignment field 最 active
- 20% eval 因为 "无 eval 即无进展"
- 10% exploration 保留 risk budget

**评分点**：你能用百分比 + reasoning + trade-off 表达——证明你 think like a research lead，不只是 IC。

---

### Q6·"如何让模型 self-improve 不被 sycophancy 污染？"

**HYPOTHESIS**：

1. 自评 model 和 generator 同源 → bias 相同 → self-praise
2. RL with self-judge converge to mode that judge favors → sycophancy

**MITIGATION 方法**：

| 方法 | 原理 | 适合 |
| --- | --- | --- |
| Separate evaluator | judge 是不同模型 / 不同 family | RLAIF |
| External verifier | math: 用 calculator；code: unit test | verifiable task |
| Adversarial probe | 故意问 known false → 看 model 是否 agree | safety check |
| Human spot check | 定期人抽 sample | universal |
| Mixture of judges | 多 judge 投票 | RLAIF |

**EXPERIMENT**：

```
Setup: train 3 versions
A) self-judge DPO (baseline)
B) cross-model judge (Claude 评 R1 output) DPO
C) external rule + mix (math: calculator; chat: Claude judge)

Eval: 
- Helpfulness benchmark (Arena-Hard)
- Sycophancy probe (TruthfulQA, MASK benchmark)
- OOD probe set

Expected: A > B > C on Arena (因为同源 over-fit)
         C > B > A on Sycophancy (external > separate > self)
```

---

### Q7·"DeepSeek-V3 在 coding 任务上比 Claude Sonnet 弱 ~5pp，你怎么 close gap？"

**INVESTIGATION**：

**Step 1·breakdown**：
> 是哪种 coding 弱？SWE-Bench？HumanEval？LiveCodeBench？
> Trajectory level：是 reasoning 弱 / planning 弱 / file edit 弱 / debug 弱

**Step 2·hypothesis**：

1. Pretrain code data 比例不够 (DeepSeek-Coder 数据 vs Claude pretrain)
2. RLHF 数据缺 coding diverse preference
3. Agent scaffold (tool format / file edit format) 不对 Claude 友好
4. Long context attention 实现 (Sonnet 200K+) 优势

**Step 3·实验**：

- A) 加 30% code-heavy pretrain data，回训 SFT → eval
- B) 收集 5K coding preference pair → DPO → eval
- C) 改 tool format 模拟 Claude's bash_tool + str_replace_editor → eval
- D) 评估 256K context vs 32K context 上 SWE-Bench 影响

**EXPECTED**：
- C 通常 +2~3pp（scaffold 影响大）
- A 训练 cost 大 +1~2pp
- B 见效快但上限低 +1pp
- D 长 context 提升 mid

**实际 best play**：C 先做（low cost high return），同时 A+B parallel，D 长期。

---

### Q8·"在 1 个月内提一个新 alignment method idea，怎么 frame？"

**FRAMEWORK**：

```
1. Problem (1 paragraph)
   - 当前 alignment 主流方法 (RLHF/DPO/GRPO) 的具体局限
   - 这个局限造成的实际 cost

2. Insight (2 paragraph)
   - 你的 unique 观察
   - 为什么这个 insight 没被前人 leverage
   - 哪个 paper / phenomenon 启发你

3. Method (3 paragraph)
   - 关键公式 (1 个核心 loss)
   - Algorithm steps
   - 和 PPO/DPO 的区别

4. Experiment Plan (1 paragraph)
   - Toy task → small model → medium scale
   - Baseline comparison
   - Ablation

5. Success criteria
   - What numbers / qualitative change make this "publishable"
   - What budget needed
```

**示范 idea (用 30 分钟现场想)**：

> **Idea: "Group-Relative DPO" (G-DPO)**
>
> **Problem**: DPO 在单 pair 上学，但实际任务一个 prompt 有 N 个 response 形成 ranking——pair-only DPO 浪费 information。
>
> **Insight**: GRPO 用 group-relative advantage 在 RL；DPO 也可以"group-relative"——把同 prompt N 个 response 的 implicit reward 整体 ranking loss 化。
>
> **Method**:
> $$L_{G-DPO} = -E_{x, \{y_1,...,y_G\}} \sum_{i<j} \log \sigma(\beta(\hat{r}_i - \hat{r}_j)) \cdot 1[y_i \succ y_j]$$
> 其中 $\hat{r}_i = \log(\pi/\pi_{ref})$。本质 listwise DPO。
>
> **Experiment**: 在 UltraFeedback / Magpie 上 G=4, vs DPO baseline，eval Arena-Hard / IFEval.
>
> **Expected**: G-DPO +2pp on Arena-Hard，因为 listwise > pairwise (类似 LambdaRank vs RankNet 历史经验)。
>
> **Failure mode**: G 太大 → memory 爆 + ranking noisy；G=2 退化为 DPO。
>
> **Connections**: 类似 SimPO + reward-aware DPO 的拓展；与 PRO (Preference Ranking Optimization) 有 overlap，需仔细文献调研。

---

## 三、Idea evaluation 4 大原则

> **当面试官问 "你这个 idea 好吗"，从这 4 个角度自评**：

1. **Novelty** — 真没人做过？还是只是 incremental？
2. **Impact** — 如果 work，影响多大？1 个 task 还是 cross-domain？
3. **Feasibility** — compute / data / time 内能完成？
4. **Failure value** — 即使 fail 也能 publish negative result？

---

## 四、4 个常用 framework（背下来）

### Framework 1·"3-condition ablation"

> 任何 idea 验证最小 ablation：
> ```
> A) baseline (no treatment)
> B) baseline + treatment
> C) baseline + treatment + remove 1 component
> ```
> 如果 A < B 且 C ≈ B → component 不必要
> 如果 A < B 且 C < B → component 必要

### Framework 2·"Necessary vs Sufficient"

> 任何 claim 拆两层：
> - 这个 condition 是 sufficient 吗？（够不够）
> - 这个 condition 是 necessary 吗？（缺不缺）
> 两者都需独立 ablation。

### Framework 3·"Scale curve"

> 任何 trick 都要看：在不同 scale (1B/7B/70B) 是否 hold？
> 很多 paper 只在 1.3B 提升，70B 失效——scale-dependence 是关键 critique。

### Framework 4·"Train-Eval gap"

> 任何方法都要看：train distribution 和 eval distribution 是否一致？
> 如果一致 → in-distribution 提升；如果不一致 → robust improvement。
> Stronger claim 是后者。

---

## 五、面试官追问最爱的 5 个角度

1. **"如果只能跑 1 个 ablation，选哪个？"** — 测你 priority
2. **"假如结果 negative，下一步？"** — 测你 resilience
3. **"为什么不直接 distill from 强 teacher 解决？"** — 测你考虑 alternative
4. **"你最大担心是什么？"** — 测你 self-criticism
5. **"如果给你 1/10 budget，你怎么 still ship 一个 minimum viable result？"** — 测你 scrappy thinking

---

## 六、最后忠告

> **Open research 题没"标准答案"，但有"差 vs 强答案"。**
>
> - 差答案：直接 jump to solution；只列 1 个 hypothesis；没 ablation；没 failure mode
> - 强答案：经历 7-step；列 3+ hypothesis 并排序；设计 minimal yet decisive ablation；想清楚 failure mode；表达 calibrated confidence

> **"我不知道但我会怎么 investigate"** > **"我知道答案是 X"**。

> **研究员 = 实验设计师 + critical thinker + 复盘者。这个题型测的是这三件事。**
