# 06 · 高频面试题·Agent 能力提升与训练方法篇

> JD 加分项 3：**"有 Agent 系统、工具调用、代码生成相关研究或实践背景"**——这一份覆盖 Agent capability training 的核心方法。
>
> 10 道题分 3 组：
> - A 组：Agent SFT/RL 方法（4 题）
> - B 组：Reasoning / tool use 训练（3 题）
> - C 组：数据闭环与 capability eval（3 题）

---

## A 组·Agent SFT/RL 方法

### Q1·怎么训一个会调 tool 的模型？SFT 还是 RL？

**SFT 路径**：

```
1. 收集 (prompt, tool_call_sequence, final_answer) 高质量轨迹
   - 来源：人工标 / GPT-4 生成 / 真实 trajectory 过滤
2. SFT on full trajectory（特别注意 tool_call format token）
3. Eval on tool use benchmark (ToolBench / API-Bank / NexusBench)
```

**RL 路径**（更上限）：

```
1. SFT cold-start（必经，否则 tool format 学不会）
2. Define reward:
   - End-to-end: 任务成功率（如 SWE-Bench pass）
   - Step-wise: tool 选择对、参数填对
3. PPO/GRPO with environment（真实跑 tool）
4. Filter trajectories: 成功的进 SFT/DPO replay buffer
```

**对比**：

| 维度 | SFT | RL |
| --- | --- | --- |
| 数据 | trajectory data | env + reward |
| 成本 | 低（一次） | 高（多次 rollout）|
| 上限 | 模仿 expert | 超越 expert |
| 失败模式 | 学 surface format | reward hacking |
| 工程 | 简单 | 复杂（env 稳定性）|

**实战推荐**：**SFT cold-start → RL polish**——纯 RL 太慢，纯 SFT 上限低。

**面试金句**："Agent SFT 解决 'how to format'，Agent RL 解决 'how to win'。两者不矛盾，是 pipeline 两阶段。"

---

### Q2·Agent SFT data 怎么造？

**4 种来源**：

1. **人工标注 expert trajectory**
   - 质量最高，成本最高
   - 适合：高价值 task / training start
   - 一般 1K-5K 条 expert demo

2. **Distill from strong model**（GPT-4 / Claude）
   - 用 strong model 跑 task → 过滤成功 trajectory → SFT
   - 量大但有 model bias（学 Claude/GPT 风格）

3. **Self-instruct / self-play**
   - 模型自生成 task 自完成 → 过滤
   - Agent-FLAN / xLAM 系列用过
   - 容易 distribution narrow

4. **真实 trajectory 反哺**
   - 从 deployment 收集真实用户 trajectory
   - 过滤成功的进 SFT
   - 闭环最强，但需 deployment

**关键过滤标准**：

```
- 任务成功（output 正确 / unit test pass）
- 步数合理（不能 100 步绕远）
- Tool call 格式正确
- 无 hallucination（tool 返回值被尊重）
- Token cost 合理
```

**关键 trick**：

- **Trajectory diversity**：同 task 多种解法都收（避免 mode collapse）
- **Failure injection**：故意加少量 fail trajectory + recovery → 教 self-correction
- **Multi-tool mix**：单 tool task / multi-tool task / no-tool task 都覆盖

---

### Q3·Agent RL 的 reward 怎么设计？

**Reward 类别**：

| 类型 | 例子 | 适合 |
| --- | --- | --- |
| **Outcome reward (verifiable)** | 数学答案对/错；unit test pass | math / code |
| **Outcome reward (judge)** | LLM judge 评 final response | open-ended |
| **Process reward (rule)** | 每步 tool call 格式对 | format learning |
| **Process reward (LLM)** | PRM 评每步合理性 | long reasoning |
| **Step penalty** | 每步 -0.01 鼓励少绕 | efficiency |
| **Cost penalty** | token 数 / tool 调用数 | cost-aware |

**典型 composition**（coding agent 例子）：

```python
def reward(trajectory):
    r = 0.0
    if final_test_pass(trajectory.solution):
        r += 1.0
    elif final_compiles(trajectory.solution):
        r += 0.2
    r -= 0.01 * len(trajectory.steps)  # step penalty
    r -= 0.001 * trajectory.token_cost  # cost penalty
    if has_repeated_tool_call(trajectory):
        r -= 0.1
    return r
```

**常见坑**：

- **Sparse reward**：纯 outcome reward (0/1) RL 难 train → 加 dense signal（partial credit）
- **Reward sub-component 权重**：scale 不同导致一个主导 → reward normalization
- **Reward hacking**：模型学 cheap reward（短答案 / 复读 tool）→ adversarial set 检测

---

### Q4·Agent capability uplift 怎么 measure？

**Eval benchmark 矩阵**：

| 能力 | benchmark |
| --- | --- |
| **Tool use** | ToolBench / API-Bank / NexusBench / BFCL |
| **Code agent** | SWE-Bench Verified / SWE-Bench Pro / LiveCodeBench / Aider Bench |
| **Web agent** | WebArena / VisualWebArena / Mind2Web |
| **Embodied** | ALFWorld / AgentBench |
| **Math reasoning** | GSM8K / MATH / AIME / MathBench |
| **General reasoning** | MMLU-Pro / GPQA / BIG-Bench-Hard |
| **Long horizon** | AgentBoard / TauBench |
| **Instruction following** | IFEval / FollowBench |

**好 benchmark 的 4 条标准**：

1. **Verifiable**：可自动判 pass/fail（不靠 LLM judge）
2. **No leakage**：rolling / 实时数据避免 train data 污染
3. **Distinguishable**：能区分 SOTA 间细微差异
4. **Realistic**：和真实 use case 一致

**eval 时常见错误**：

- ✗ 只看 mean，不看 distribution / std
- ✗ 用 N=10 评估（need N≥100）
- ✗ Pass@1 评一次 vs Pass@k 评 k 次混淆
- ✗ Best-of-N 评估时没控 N 一致
- ✗ 没 holdout test set，反复在 dev set 上 tune

---

## B 组·Reasoning / Tool use 训练

### Q5·怎么让 small model（7B）有强 reasoning？

**3 条路径**：

**路径 1·Distillation from strong reasoning model**：
- 用 R1 / o1 生成大量 (problem, CoT, answer) 数据
- SFT 小模型
- R1 paper 实验：7B distilled 接近 R1 原 32B
- ✅ 简单有效；❌ 受限于 teacher quality

**路径 2·Process supervision SFT**：
- 用 PRM-rated step 数据训
- 重点教 step-level correctness
- MathShepherd 在 7B 上接近 GPT-4 数学

**路径 3·小模型 RL（GRPO）+ verifiable reward**：
- 数学题 rule-based reward
- 7B GRPO 训 GSM8K / MATH
- ✅ 可超越 SFT；❌ 较慢

**Best practice (DeepSeek R1 distilled 系列)**：
- **路径 1 cold start → 路径 2 process tuning → 路径 3 RL polish**

---

### Q6·Reasoning model（o1/R1）的 "test-time scaling" 是什么？怎么 optimize？

**Test-time scaling**：推理时多生成 token / 多采样 → 性能↑。

**3 种 form**：

1. **Long CoT**（o1/R1 主用）：单次 generation 但 thinking 长，model 自己 self-correct
2. **Best-of-N**：sample N response，RM/PRM 选最优
3. **MCTS / search**：tree search 探索多 branch（AlphaGeometry 用）

**Scaling law**（OpenAI o1 blog）：
- 推理 token ↑ → AIME 准确率 ↑（对数线性）
- 一直 scale 到 plateau

**优化空间**：

- **Adaptive budget**：简单题少 thinking，难题多——early-exit when confidence 高
- **Speculative decoding**：thinking phase 用更激进 spec decoding
- **Beam search + PRM**：PRM 引导 beam，比纯 sampling 高效
- **Prefix caching**：thinking 阶段 KV cache 大，重用 prefix

**研究热点**：reasoning model 经常 over-think 短问题（"2+2 等于多少" 思考 1K token）——budget aware reasoning 是 hot direction。

---

### Q7·Long horizon agent task 训练有什么 unique 挑战？

**挑战 1·Credit assignment 难**：

- 100 step 中 step 5 出错，但只在 step 100 才知道 fail
- 怎么 assign 责任给 step 5？

**解法**：
- Process reward model
- GAE($\lambda$=0.95) 让 credit propagate 远
- Step-wise verifier（unit test 每改一个文件就跑）

**挑战 2·Context length**：

- 100 步 trajectory 可能 100K+ tokens
- Train 时 long context 慢；inference cost 大

**解法**：
- Subagent / handoff：切到 fresh context
- Summary / compaction：定期压缩历史
- 用 long context model（200K+）

**挑战 3·Rollout cost 高**：

- 单次 trajectory 5~30 分钟
- RL batch size = 512 → 几天 / iter

**解法**：
- Parallel rollout（K8s + isolated env）
- Off-policy RL（重用旧 trajectory）
- 用 small model rollout，big model train

**挑战 4·Environment instability**：

- 真实 env（如代码执行）有随机性、网络问题
- Reward noisy

**解法**：
- Env retry + timeout
- Deterministic seed
- Multi-sample 平均 reward

**挑战 5·Mode collapse to suboptimal strategy**：

- Agent 学会一种 "safe" 策略反复用，不探索
- 如 always run grep before edit

**解法**：
- Entropy bonus
- Curiosity reward
- Diverse training task

---

## C 组·数据闭环与 Capability Eval

### Q8·"数据-训练-评测" 闭环具体怎么落？

**经典闭环（Anthropic / DeepSeek 推测都用）**：

```
┌─ Deployment ─┐
│              │
│  Trajectory  │
│  collection  │
│              │
└──────┬───────┘
       │ failed trajectory
       ▼
┌──────────────┐
│  Fail mode   │
│   triage     │ ← 研究员 + 数据标注 collab
│              │
│ - Bucket by  │
│   capability │
└──────┬───────┘
       │
       ▼
┌──────────────┐
│ Targeted     │
│ data gen     │ ← 数据标注 lead
│              │
│ - Human label│
│ - Synthetic  │
└──────┬───────┘
       │
       ▼
┌──────────────┐
│ SFT / DPO /  │
│ RL training  │ ← 研究员 lead
└──────┬───────┘
       │
       ▼
┌──────────────┐
│   Eval       │
│              │ ← 工程 + 研究员
│ - Benchmark  │
│ - Holdout    │
│ - A/B in dep │
└──────┬───────┘
       │
       └─→ 回到 deployment
```

**研究员在这个 loop 里的 role**：

- Triage fail mode（看 trajectory，归类）
- 设计 data spec（多少条 / 什么 distribution / rubric）
- 跑 training experiments
- 设计 eval metric

**关键产物**：

- **Capability matrix**：(capability × eval score) 矩阵每个 ckpt update
- **Fail mode taxonomy**：定期 review，看是否新 cluster
- **Data efficiency curve**：每加 1K 标注，capability 涨多少

---

### Q9·怎么 collaborate with 数据标注团队设计 rubric？

**5 个步骤**：

**Step 1·研究员自己先标 50~100 条**

- 不能 outsource design 给标注员——研究员先 dogfood
- 自己标过才知道 boundary case

**Step 2·写初版 rubric（1 页 PDF）**

```
Task: 评估 model response 的 helpfulness

5 = 完美回答，所有 sub-question 都 cover
4 = 主要点 cover，小遗漏
3 = 部分 cover，但缺关键
2 = 几乎不相关
1 = 完全错 / 拒答

Examples (per score):
  Score 5: ... (3 个 example)
  Score 4: ...
  ...

Boundary cases:
  - 部分对部分错 → 取低
  - 题不清 → score 评 "ambiguous"
```

**Step 3·和 1~3 个 senior 标注员 calibrate**

- 共标 100 条
- 算 Cohen's kappa / agreement rate
- Disagreement 反过来改 rubric

**Step 4·Pilot run 1K 条**

- 大规模标
- Spot check 5-10%
- 算 inter-annotator agreement

**Step 5·定期 audit + iterate**

- 每月 spot check + rubric v2

**面试金句**：
> "好 rubric 的标志不是 '完整' 是 'inter-annotator agreement 高' + 'researcher 用起来能 reproduce'。我自己实习时设计过 X rubric，初版 agreement 0.6，经过 2 轮 calibration 到 0.85。"

---

### Q10·怎么判断"模型的某个能力变弱了"（regression）？

**3 层 detect**：

**1·Automated eval regression**

- 每 ckpt 跑 standard benchmark
- 主指标 + 子指标 alert（如 整体不变但 math sub 降 5%）
- 用 CI 自动跑

**2·Trajectory-level pattern**

- 看 deployment trajectory 的 length distribution / tool call frequency / error rate
- 如发现 tool call 突然减少 → 可能 tool use 退化

**3·Adversarial probe / capability probe**

- 维护一组 50~200 "canary" example—覆盖核心 capability
- 每 ckpt 必跑 canary
- Canary fail 立刻 stop deployment

**Root cause 排查**：

| 现象 | 可能原因 |
| --- | --- |
| Math regress | RL reward 过分 outcome；data mix 缺 math |
| Long context regress | training data 太短 / position interpolation 错 |
| Code regress | dependence on tool format 太死 |
| Multilingual regress | 多语 data 比例下降 |
| Refusal increase | safety RM 过强 |

**Mitigation tactics**：

- **Capability gating**：deploy 前 100% canary pass
- **Mixture-of-Experts checkpoint**：不同 capability 用不同 ckpt（Anthropic 推测）
- **Rollback strategy**：发现 regression 1h 内 rollback

---

## 答题"通用 framework"（Agent 能力题）

**3 步框架**：

1. **拆问题**：这是 capability training / eval / data / deployment 哪一环？
2. **给 2~3 解法 + trade-off**：不要给 1 个答案，要 framing alternatives
3. **加一个 specific example**：你自己跑过 / 读过的 paper / 业界 case

**示例答 Q5 (small model reasoning)**：

> "可以走 3 条路：distillation / process SFT / RL with verifiable reward。Trade-off 是 (compute, ceiling, complexity)。
>
> Distillation 最快——R1-distill-7B 几乎接近 R1，但 ceiling 受 teacher 限。
>
> Process SFT 中等——MathShepherd 在 7B 上接近 GPT-4 GSM8K，但需要 process reward data，标注贵。
>
> RL 上限最高——GRPO + rule-based reward 可让 7B 在 verifiable task 上超越 distillation。但训练时间数倍。
>
> 我自己跑过 Qwen 0.5B 在 GSM8K 上 GRPO finetune（小实验），发现 group size = 8 + warm-up SFT 5K steps + RL 5K steps 这套 recipe 可让 acc 从 35% → 60%。最大踩坑是 reward 全 0/1，前期 advantage variance 太小训不动——加了一个 partial credit reward 才走通。"
