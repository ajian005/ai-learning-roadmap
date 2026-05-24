# 05｜高频面试题 — Agent 训练与前沿研究篇

> 14 道题，覆盖 Agent 模型训练独有的难题：long-horizon credit assignment、tool-use 训练、SWE-RL、PRM、self-improvement、inference scaling。

---

## Q1. Agent 训练相比普通 chat 模型训练，有哪些独特挑战？

### 满分答

| 挑战 | chat 模型 | agent 模型 |
| --- | --- | --- |
| **Episode 长度** | 1~2 turn (~1K tokens) | 50~500 turn (10~100K tokens) |
| **Reward sparsity** | turn-level RM | 只有 final success/fail (extreme sparse) |
| **Action space** | 自然语言 | language + tool call + structured output |
| **Credit assignment** | trivial | 极难——失败发生在第 3 步还是第 30 步？ |
| **Trajectory cost** | 1 个 prompt = 1 个 response | 1 个 task = 50 step × 5K token = 巨大上下文 |
| **Reproducibility** | 高 | 低——sandbox state / 网络 / 外部 API 不可控 |
| **Eval** | 用 RM 或 judge | 用 task success（pass@1 等） |
| **Tool failure** | 不存在 | 第三方 API 挂了会污染训练 |

### 重点强调

- **稀疏 reward + 长 horizon** = agent RL 最大难点
- 解法：1) 人工/启发式 guidance（Agent-RLVR）2) PRM step-level reward 3) 子任务分解 + curriculum
- **数据 cost**：一条 SWE-Bench trajectory 可能要 30 分钟跑 + 100K token——RL 一个 epoch 可能要几千 trajectory

---

## Q2. SWE-Bench 是什么？为什么是 agent 能力的"圣杯 benchmark"？

### 满分

**SWE-Bench** (Jimenez et al., 2310.06770)：
- 数据：从 GitHub 真实 repo（如 Django、Flask）抓 2,294 个 PR
- 每个 task：给一个 issue 描述 + repo 当前 state，agent 要产生 patch 让 PR 的测试都过
- 评测：跑 PR 的单元测试，pass = 解决

**子集**：
- **SWE-Bench Full**：2,294 题
- **SWE-Bench Lite**：300 题易版
- **SWE-Bench Verified**：500 题（OpenAI 人工筛过的"题面清晰"版本）—— 目前主流 benchmark

**为什么是圣杯**：
- **真实**——不是合成的 toy task，是真实 PR
- **可验证**——单元测试给 0/1 binary reward
- **多步**——需要 navigate repo + 读多文件 + 改多文件 + 跑 test + debug
- **覆盖多种能力**：代码理解、推理、long context、tool use、planning

**当前 SOTA（截至 2026）**：
- 顶级闭源（Claude 4 / GPT-5）≈ 70%+ pass@1 on Verified
- 开源最强（Qwen-72B + RL）：60%+
- 1 年前（2024 末）SOTA 只有 30%

**面试常被追问**：
- "SWE-Bench 测的是 model 还是 scaffold？" → 二者都测——同一模型不同 scaffold（OpenHands / SWE-agent / Aider）差 20+ 分
- "怎么避免 contamination？" → 用 cutoff date 后的 PR + Verified 集人工 audit

---

## Q3. 在 SWE-Bench 上跑 RL，关键技术挑战是什么？

### 满分

1. **Trajectory 巨长** → 1 个 task 可能 50K~150K token，KV cache 显存压力大
2. **Reward 极稀疏** → 只有最后跑 test 才有 binary reward，中间所有步都"没有信号"
3. **环境不可控** → docker image 启动慢、依赖冲突、test flaky
4. **Sample 成本巨高** → 一个 GRPO group=8 × 50 task × 100K token = 4000万 token/epoch
5. **Credit assignment** → 失败是因为第 1 步看错文件还是第 30 步改错？
6. **Tool failure ≠ model failure** → 工具挂了不应惩罚 model
7. **Contamination** → 训练数据里的 repo 可能就是 eval 题

### 主流解法

- **Agent-RLVR**：用"老师"提供 high-level plan 当 dense guidance，把稀疏 reward 变成密集 reward
- **PRM-guided**：训一个 step-level reward model，给每步打分
- **Curriculum**：从简单 task（单文件改）到复杂（跨文件 refactor）渐进
- **Sub-sampling**：长 trajectory 只在关键 turn 算 loss
- **Length normalization**：避免长 trajectory 主导梯度
- **GRPO group=8~16** + rule-based reward + KL=0.001 是常见 config

---

## Q4. 介绍 SWE-RL / Agent-RLVR 这一类工作的核心思想

### 满分

#### SWE-RL (Wei et al., Meta 2025)
- 在 GitHub PR 数据上做 RL 训练
- reward = patch 是否通过原 PR 的测试
- 创新点：把 issue → PR 转化成"agent observe issue → produce patch"的 RL task

#### Agent-RLVR (Lu et al., 2506.11425)
- **核心 trick**：除了环境 reward，还加 "guidance reward"——一个 teacher model 给出 step-level 建议
- 当 agent 步骤跟 guidance 一致 → 加 reward
- 解决稀疏 reward：把 50 步只有 final reward 变成每步都有反馈
- 效果：Qwen-2.5-72B 在 SWE-Bench Verified 从 9.4% → 22.4%

#### SWE-Fuse
- entropy-aware RLVR：在高 entropy step（探索期）放宽 PPO clip；在低 entropy step（exploitation）收紧 clip
- 32B 模型 60% pass + test-time scaling 到 65%

### 共通思路

```
sparse-reward agent task 的解法 menu:
1. Dense reward （PRM, guidance, sub-goal）
2. Curriculum learning
3. Better exploration (entropy bonus, sampling diversity)
4. Better credit assignment (step-level value, MCTS)
5. Sample efficiency (off-policy, replay)
6. Test-time scaling (best-of-N, MCTS, verifier rerank)
```

---

## Q5. 如何训一个模型学会"什么时候 call 什么 tool"？

### 满分

**几种范式**：

#### 1. SFT only（最简单）
- 数据：`{prompt, [tool_call(args), tool_response, ..., final_answer]}`
- 让 model imitate 正确的 tool use 轨迹
- 代表：OpenAI function calling 训练

#### 2. ToolFormer（Self-supervised, 2023）
- 用 base 模型 sample 一些"插入 tool call"的位置
- 跑 tool 看 response 后，比较"有 tool" vs "无 tool"的 perplexity
- 选 tool-helpful 的位置当训练数据
- → 不需要人工标 tool use 轨迹

#### 3. RL with tool reward
- reward = task success + tool use 是否合理（用 PRM 评每步）
- 用 GRPO/PPO 训
- 难点：sparse + tool failure 处理

#### 4. RL with judge
- 用 GPT-4 judge 每步 tool use 是否合适
- 类似 RLAIF 但 step-level

### 关键 trade-off

- **召回 vs 准确**：过度 call tool → 浪费 + 慢；漏 call → 答错
- **训练侧**：要平衡"应该 call"和"不应该 call"的数据比例（一般 30:70）
- **inference 侧**：tool definition prompt 的写法（function schema）显著影响 model 召唤倾向

### 高频追问

- "Tool 数量从 10 涨到 1000 会怎样？" → in-context tool list 太长 → token 浪费 + 选择困难 → 解法：retrieval-augmented tool selection
- "MCP 和 OpenAI function calling 模型怎么处理？" → 训练时统一成 schema，inference 时 wrapper 转换

---

## Q6. 长链 reasoning（CoT, o1-style）训练怎么做？

### 满分

**几条路线**：

#### 1. SFT on CoT data
- 收集"thinking..."CoT 数据（人写 + Claude/GPT-4 蒸馏 + filter）
- SFT 让 model 学会 "先 think 再 answer"

#### 2. STaR / Quiet-STaR
- 让 model 自己生成 reasoning → 用 final answer 正确性 filter → 再 SFT
- "自给自足"循环提升

#### 3. RL on reasoning（DeepSeek-R1 路线）
- 在 math/code 上跑 GRPO，reward = answer 正确性
- 自发涌现"think more, self-reflect, verify"等行为
- 关键：让 model 自由生成 thinking process，不约束格式

#### 4. PRM-guided RL
- 训 PRM 评每步 reasoning quality
- inference 时用 PRM 做 best-of-N rerank 或 MCTS 引导

#### 5. Inference-time scaling
- 多 sample + majority vote
- best-of-N + PRM rerank
- MCTS / Tree-of-Thoughts

### 关键发现（来自 R1）

- **长度自发增长**：R1-Zero 训练中 response 长度从 ~500 涨到 ~5K——model 自己学会"想更多"
- **Aha moment**：出现 "Wait, let me reconsider"
- **Language mixing**：中英混杂——加 language consistency reward 缓解

---

## Q7. 解释 inference-time scaling 的几种方法

### 满分

```
范式 1: Sampling-based
- Greedy → 最快但 1 个答案
- Temperature sampling → N 个不同答案
- Best-of-N + reward model rerank
- Majority voting (self-consistency)

范式 2: Search-based  
- Tree-of-Thoughts: BFS / DFS 思考树
- MCTS: UCB 选下一步，PRM 当 value
- Beam search with PRM rerank

范式 3: Sequential refinement
- Reflexion: model 自己 critique 自己再改
- 多轮 self-correction

范式 4: o1-style long reasoning
- 模型自带 long CoT
- 一个 sample 但极长（10K+ tokens）
- 训练时强化"think long"
```

**Trade-off**：
- 算力换效果——成本可能 10x~100x
- 不同任务收益曲线不同：math/code 收益大；闲聊收益小
- 容易触顶——超过一定 N 后 marginal gain → 0

### 高频追问

- "为什么 o1 不开放 chain？" → safety + 防止竞品蒸馏
- "MCTS 适合 LLM 吗？" → 适合 reward 易评估的任务（math/game），不适合开放对话
- "如何 train inference scaling 的 model？" → 训练时多 sample + 学 self-correct + PRM 引导

---

## Q8. Multi-Agent 系统训练的挑战是什么？

### 满分

**Multi-Agent**：多个 LLM agent 协作完成任务（Specialist + Orchestrator / Code + Review / Debate）。

**训练挑战**：
1. **Credit assignment 跨 agent**：失败是 A 的责任还是 B 的？
2. **Non-stationarity**：每个 agent 都在变 → 环境对单个 agent 来说 non-stationary
3. **Communication**：agent 间用自然语言通信，channel 是 LLM-readable，难 RL 优化
4. **Reward design**：team reward vs individual reward 权衡
5. **Computation**：N 个 agent 同时跑，sample 成本 N x

**现有方法**：
1. **Independent learning**（最简单）：每个 agent 单独 RL，不管别人
2. **Centralized critic**（多 agent RL 经典）：critic 看所有 agent，actor 各自看自己
3. **Self-play**（debate / negotiation）：两个同模型 agent 互相 critique

**实际工业界**：multi-agent 训练还较前沿，DeepSeek/Anthropic 主要用 **single-model + role prompting** 模拟多 agent，避免训练复杂度。

---

## Q9. 介绍 Self-Improvement / Self-Play 在 LLM 上的应用

### 满分

**Self-Improvement 模式**：

#### 1. STaR (Self-Taught Reasoner)
- model 生成 reasoning → 用 final answer 正确性 filter → SFT 自己
- 局限：能解的题才能学，难题永远学不到

#### 2. Self-Rewarding LM (Meta)
- model 同时当 generator 和 judge
- 自己 sample → 自己评 preference → DPO 自训
- 风险：echo chamber + bias amplification

#### 3. Constitutional AI (Anthropic)
- 用宪法原则 self-critique → revise → SFT
- 再用 model 评 preference → RL

#### 4. AlphaZero-style for LLM
- self-play on code/math game
- MCTS + value network 引导
- 早期工作（如 PoT, code-self-play）

### Key Insight

- Self-improvement 能 push capability ceiling **but** 需要：
  - 有 verifier（math/code 有，对话没有）
  - 有 diversity preservation 机制（防 collapse）
  - 有人工 anchor data 防偏

### 高频追问

- "self-improvement 能让 model 超越 base capability 吗？" → 在 verifiable domain 能（如 chess, math）；在 open domain 极难——容易 mode collapse
- "怎么 detect collapse？" → 监控 sample diversity (n-gram, embedding distance) + held-out task performance

---

## Q10. 解释 RLHF / RLAIF / RLVR / RLEF 这些术语区别

### 满分

```
RLHF  = RL from Human Feedback        ← 人工标 preference → RM → PPO
RLAIF = RL from AI Feedback          ← LLM 标 preference → RM → PPO
RLVR  = RL with Verifiable Reward    ← rule-based reward（math/code）
RLEF  = RL with Execution Feedback   ← 跑代码看结果当 reward
RLCF  = RL with Critic Feedback      ← PRM 给 step-level 反馈
```

**对比**：

| | reward 来源 | scale | quality | 主要风险 |
| --- | --- | --- | --- | --- |
| RLHF | 人 | 慢 | 高 | 标注成本 + 不一致 |
| RLAIF | LLM judge | 快 | 中 | judge bias |
| RLVR | 规则/程序 | 极快 | 高（无噪） | 仅适合可验证任务 |
| RLEF | 执行环境 | 中 | 高 | 环境复杂度 |

**实战**：DeepSeek-R1 = RLVR + GRPO；ChatGPT = RLHF + PPO；Claude = RLHF + Constitutional (RLAIF)。

---

## Q11. 评测 Agent 模型有哪些挑战？

### 满分

#### 主流 agent benchmark
- **SWE-Bench / SWE-Bench Verified** - 真实 GitHub task
- **WebArena / VisualWebArena** - 浏览器操作
- **OSWorld** - 桌面操作
- **AgentBench** - 多 domain
- **τ-Bench** - 客服 agent
- **BrowseComp** (OpenAI) - 复杂网页搜索
- **GAIA** - 通用 agent 评测

#### 评测挑战
1. **环境状态难复现**——网络变化、API 改变、website 改版
2. **成本高**——一次完整 SWE-Bench 跑可能花 $500+ API + 几小时
3. **Flaky**——pass rate 抖动大，需要多次跑取平均
4. **Contamination**——训练数据可能见过 eval 题
5. **scaffold 影响**——同 model 不同 scaffold 差 20+ 分
6. **指标局限**：pass@1 / pass@k 只能 binary 评——partial success 无法体现

#### 改进方向
- **Held-out 时间 cutoff**：用 cutoff 后数据
- **Multiple seed 多次跑**：取 mean ± std
- **分组评测**：按难度、领域分桶
- **细粒度指标**：除了 pass，还看 partial credit、token cost、time、tool call efficiency

---

## Q12. Reward Model 在 Agent RL 里能不能用？

### 满分

**答**：能但 tricky。

**怎么用**：
- ORM：给整条 trajectory 一个 final reward (类似 chat RM)
- PRM：给每步 reward
- Trajectory-level RM：评 trajectory 整体质量（含 efficiency / safety）

**挑战**：
1. **Trajectory 长** → RM 上下文压力大
2. **OOD generalization** → policy 探索的 trajectory 可能 RM 没见过
3. **Adversarial trajectory** → policy 学会"对 RM 友好但实际差"

**实战推荐**：
- agent 场景**优先 RLVR**（用执行结果）
- 不可验证的场景（如 helpfulness）才用 RM
- PRM 在 agent 上很有前景但训练数据极贵

---

## Q13. （开放）你来 DeepSeek 第一周想 own 什么 agent 训练实验？

### 满分模板

```
"我会从 [一个具体 + 可在 1~2 周内做出来] 的实验起：

例：'用 GRPO 在 SWE-Bench Lite 的一个 sub-domain（如 Django repo 子集）上做 fine-tune'

理由：
1. 范围小但有信号 → 能在两周给出第一版 result
2. Sub-domain 让 reward signal 更密 → 不容易 stuck 在 sparse reward
3. 能复用 R1 的 GRPO infra（如果内部有）
4. 结果可对照 baseline (vanilla SFT + same model)

预期产出：
- 一份 1 页 result note：baseline % vs GRPO % 提升
- 一份 ablation：group size / KL coef / reward signal 影响
- 一个 follow-up proposal：哪里 promising 值得 scale up

风险：
- sparse reward 可能 stuck → 备选方案：加 PRM-style guidance reward
- contamination → 用 cutoff date 后的 PR"

不展示 specific idea，但展示了：
1. 能 propose concrete experiment
2. 知道现有 SOTA + DeepSeek infra context
3. 有 risk awareness
4. 能 ship 出实验
```

---

## Q14. 你怎么 stay current on agent / RL research？

### 满分骨架

```
日常:
- arxiv-sanity / Hugging Face daily papers
- Twitter follow: @karpathy @sama @giffmana @AnthropicAI @DeepSeek_AI
- arxiv RSS: cs.LG, cs.CL, cs.AI

每周深读:
- 选 1~2 篇 must-read，30~60 分钟精读
- 写 1 页 summary（problem / insight / limitation / 你的 take）

每月:
- 主流 lab 的官方博客（OpenAI, Anthropic, DeepMind, Meta AI）
- NeurIPS / ICML / ICLR 接收列表扫一遍

季度:
- 跑通至少 1 个 paper 的开源实现
- 写 1 篇技术 blog

社区:
- 参加 1~2 个 reading group（如 lab 内部、Discord）
- 关注 Eleuther / LM-eval-harness 等开源讨论
```

**核心**：展示你**自己 own 学习——不靠别人 push**。研究员的最 valuable 特质。

---

下一文档：[06-高频面试题-数据训练评测闭环与实验设计](06-高频面试题-数据训练评测闭环与实验设计.md)
