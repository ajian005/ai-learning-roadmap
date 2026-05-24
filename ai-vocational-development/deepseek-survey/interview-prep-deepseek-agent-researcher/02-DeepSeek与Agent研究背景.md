# 02｜DeepSeek 与 Agent 研究背景调研（研究员特化版）

> 公司背景的"通用"部分见 [PM 包 02](../interview-prep-deepseek-harness-pm/02-DeepSeek与Harness背景调研.md)。
>
> 本文聚焦研究员**必须深读 + 能复述**的 DeepSeek 自家技术、关键算法、前沿 paper 地图。

---

## 一、DeepSeek 模型谱系（研究员必背）

```
2023  DeepSeek-LLM 7B / 67B          → 起家模型
2024  DeepSeek-MoE 16B               → 第一次 MoE 尝试
2024  DeepSeek-Coder v1 / v2         → 代码模型
2024  DeepSeek-V2 (Jan 2024)         → MLA + DeepSeekMoE 架构定型
2024  DeepSeek-V3 (Dec 2024)         → 671B MoE / 37B 激活 / FP8 训练 / Aux-loss-free
2025  DeepSeek-R1 / R1-Zero (Jan 2025) → 纯 RL → reasoning emergence / GRPO
2025+ DeepSeek-V3.1 / V3.2 / R2 ...  → 持续迭代
```

### 必背核心架构创新

#### 1. MLA (Multi-Head Latent Attention)
- **痛点**：标准 MHA 的 KV cache 太大（每 head 一份 K/V）
- **方案**：将 K、V 通过低秩矩阵 `W^DKV` 压到一个 latent vector `c^KV`，推理时再用 `W^UK / W^UV` 升回
- **效果**：KV cache 减少 93%，性能接近无损
- **面试常问**：
  - "MLA 和 GQA / MQA 的本质区别？" → GQA 是 head 之间共享 K/V；MLA 是用低秩投影**对单 head 内部降维**
  - "MLA 推理时怎么和 RoPE 兼容？" → 用"decoupled RoPE"——一部分维度 latent，一部分维度直接走 RoPE

#### 2. DeepSeekMoE
- **痛点**：标准 MoE 容易 expert 不均衡 + routing 不稳定
- **方案**：
  - **细粒度专家**（fine-grained experts）：把每个 expert 切更细 → 更多组合可能
  - **共享专家**（shared experts）：always-on 的专家 + routed 专家
  - **Aux-loss-free balancing**（V3 创新）：不用辅助 loss 而是动态调 bias 来均衡 routing
- **面试常问**：
  - "为什么要 shared expert？" → 共性知识不应分散到 routed 中
  - "Aux-loss-free 怎么实现 balancing？" → 给每个 expert 维护一个 bias `b_i`，每个 batch 后根据 expert 使用量调整（用多 → 降 bias；用少 → 升 bias）

#### 3. FP8 训练（V3 工程突破）
- 用 FP8 mixed precision 训练 600B+ MoE，节省 ~50% 显存
- 关键：online scaling factor + per-tile quantization
- **面试常问**：
  - "FP8 训练的主要挑战？" → 数值范围窄 + outlier 处理 + 反传梯度精度

#### 4. GRPO (Group Relative Policy Optimization) ⭐⭐⭐
- DeepSeek-R1 / R1-Zero 用的核心 RL 算法
- 详细推导见 [03 算法速查](03-核心算法速查.md)
- **一句话**：去掉 PPO 的 critic，用一组 sample 的均值作 baseline，组内归一化算 advantage

---

## 二、DeepSeek-R1（必须深读）

### 关键 facts（背下来）

- **2 个模型**：
  - **R1-Zero**：从 base 模型直接 RL（无 SFT）→ 出现 self-reflection / "aha moment"
  - **R1**：先 SFT 冷启动 → RL → rejection sampling SFT → 二次 RL
- **R1 训练 4 stages**（必背）：
  1. **Cold-start SFT**：少量高质量 CoT 数据（人写 + few-shot R1-Zero 输出过滤）
  2. **RL for reasoning**：在 math / code / logic 上跑 GRPO，**rule-based reward**（答对/答错 + format reward）
  3. **Rejection sampling + SFT**：从 RL 模型采样 → 用 reward / 人工过滤 → 600K reasoning + 200K general 数据再 SFT
  4. **RL for all scenarios**：第二轮 RL，加入 helpfulness / harmlessness reward
- **rewards** 用规则（rule-based），**不用 learned reward model**——这是 DeepSeek 一个关键选择，避免 reward hacking
- **R1-Zero 关键发现**：
  - **响应长度**会自发增长（自己学会"think more"）
  - 出现"aha moment"：模型会写出 "Wait, let me reconsider..." 这种 self-reflection
  - 但有 "language mixing" 问题（中英混杂）→ R1 加 language consistency reward

### 高频追问

| 问题 | 关键答 |
| --- | --- |
| 为什么 rule-based reward 比 RM 好？| 避免 reward hacking；math/code 有 ground truth；RM 训练成本高 |
| 为什么要 cold-start？| R1-Zero readability 差；冷启动加可读性 + format |
| 为什么 stage 3 用 SFT 而不是继续 RL？| 拓展到 general domain；RL 在无 verifier domain 上 reward 设计困难 |
| GRPO 相比 PPO 优势？| 省 critic（一半显存）；批内归一化 → 自适应 reward scale |
| R1 的可复现性？| 训练数据未开源；很多实验组复现到 70%~85% 水平（如 Open-R1 / TinyZero） |

---

## 三、DeepSeek-V3（必读）

### 关键 facts
- **架构**：61 层 / 7168 hidden / 671B 总参 / 37B 激活
- **MoE**：256 routed experts + 1 shared expert / 8 个 top-k
- **训练量**：14.8T tokens；2048 H800 GPU × ~3.7 天/T tokens
- **关键技术**：MLA + DeepSeekMoE + FP8 + MTP（Multi-Token Prediction）

### MTP（Multi-Token Prediction）
- 训练时**同时预测下一个 + 下下个 token**（共享主干 + extra head）
- 优点：1) 训练信号更密 2) inference 可做 speculative decoding（额外 head 当 draft model）

### 高频追问
- "V3 用 MTP 训练时怎么处理 loss？" → 主 loss + λ × MTP loss
- "MTP 推理怎么用？" → 用 MTP head 做 1-step lookahead，主模型 verify → speculative decoding 加速

---

## 四、Agent 训练 / RL 的前沿 paper 地图（必读 + 选读）

### A. RLHF / 对齐核心

| Paper | 年份 | 一句话 |
| --- | --- | --- |
| **InstructGPT** (Ouyang et al.) | 2022 | RLHF 鼻祖：SFT → RM → PPO |
| **DPO** (Rafailov et al., 2305.18290) ⭐ | 2023 | 把 RLHF 简化成 classification loss，无需 RM |
| **Constitutional AI** (Bai et al., 2212.08073) | 2022 | RLAIF 先驱：用 AI 反馈代替人工 |
| **IPO** (Azar et al.) | 2023 | DPO 改进：避免 likelihood-displacement |
| **KTO** (Ethayarajh et al.) | 2024 | 不需 pairwise 偏好，单点 binary 反馈即可 |
| **SimPO** (Meng et al., 2405.14734) | 2024 | DPO 去掉 reference model，用长度归一化 |
| **ORPO** (Hong et al.) | 2024 | SFT + odds-ratio preference 一步训完 |

### B. 过程奖励 / Verifier-based RL

| Paper | 年份 | 一句话 |
| --- | --- | --- |
| **Let's Verify Step by Step** (OpenAI, 2305.20050) ⭐ | 2023 | PRM800K 数据集 + 证明 PRM > ORM |
| **Math-Shepherd** (Wang et al., 2312.08935) | 2023 | 用 MCTS 自动生成 process supervision |
| **OmegaPRM** (Luo et al.) | 2024 | 二分搜索式 PRM 数据生成 |
| **PRIME** (Cui et al., 2502.01456) | 2025 | implicit PRM——用 reward model 当 logp 估计 |

### C. Agent / Code 训练 RL

| Paper | 年份 | 一句话 |
| --- | --- | --- |
| **DeepSeek-R1** (2501.12948) ⭐⭐⭐ | 2025 | 纯 RL 训 reasoning + GRPO |
| **SWE-RL** (Wei et al.) | 2025 | 在 GitHub PR 数据上做 RL 训 SWE 能力 |
| **Agent-RLVR** (2506.11425) | 2025 | guidance-based RL，Qwen-2.5-72B 在 SWE-Bench 9.4% → 22.4% |
| **SWE-Gym** (Pan et al.) | 2024 | 用真实 GitHub task 构 agent training env |
| **OpenHands / SWE-agent** | 2024 | 开源 SWE agent baseline |
| **Voyager** (Wang et al.) | 2023 | Minecraft 长期 agent，skill library |
| **ToolFormer** (Schick et al.) | 2023 | 自监督学 tool use |
| **ReAct** (Yao et al.) | 2022 | reasoning + acting 交错 prompting |

### D. Scaling RL / Long-horizon credit assignment

| Paper | 年份 | 一句话 |
| --- | --- | --- |
| **GRPO 理论分析** (2503.06639) | 2025 | 证明 GRPO 渐近等价于带最优 critic 的 PPO |
| **RLOO** (Ahmadian et al.) | 2024 | Reinforce w/ leave-one-out baseline——比 PPO 更简单 |
| **REMAX** (Li et al.) | 2024 | 用 greedy decode 当 baseline 的 reinforce |
| **VinePPO** | 2024 | 中间 step value 估计改进 |
| **REINFORCE++** | 2025 | 改进的 reinforce，多项工程优化 |

### E. 推理能力 emergence

| Paper | 年份 | 一句话 |
| --- | --- | --- |
| **CoT** (Wei et al., 2201.11903) | 2022 | Chain-of-Thought prompting 原文 |
| **Self-Consistency** (Wang et al.) | 2022 | 多 sample + majority vote |
| **Tree of Thoughts** (Yao et al.) | 2023 | 搜索式 reasoning |
| **OpenAI o1 system card** | 2024 | 闭源 o1 的对外说明 |
| **STaR / Quiet-STaR** | 2022/24 | self-taught reasoner |

### F. 数据 / Eval

| Paper | 年份 | 一句话 |
| --- | --- | --- |
| **SWE-Bench** (Jimenez et al., 2310.06770) ⭐ | 2023 | 用真实 GitHub issue 评测 code agent |
| **SWE-Bench Verified** (OpenAI) | 2024 | 人工筛过的 500 题 subset |
| **MMLU / GPQA / MATH / HumanEval / LiveCodeBench** | - | 各通用 benchmark |
| **IFEval** | 2023 | 指令遵循评测 |

---

## 五、DeepSeek 内部研究文化（推测，基于公开信息）

> 来自梁文锋公开访谈、员工博文、Reddit/HN 讨论的综合：

- **研究员驱动**：不是 product / business 给 research 提需求，而是研究员定义方向
- **小团队**：每个项目通常 5~10 人，研究员是核心
- **fast iteration**：内部周会 / 双周节奏；不是月度
- **开源 + paper first**：V3 / R1 都有详细技术报告——研究员的 portfolio 要"在阳光下"
- **"做有意思的事"** > "刷 paper"：V3 / R1 不为 SOTA paper 而做，是想看能不能 push 一个方向

> **面试启示**：你的 motivation 要回答"为什么是 DeepSeek 而不是其他公司"——
> 标准答：**"我想做 LLM 研究本身，不想做 LLM applied research / business-driven research。DeepSeek 是中国少数让研究员 own 方向的公司。"**

---

## 六、值得关注的 Agent 训练前沿方向（你可能被问"你想做什么"）

| 方向 | 关键问题 | 推荐切入 |
| --- | --- | --- |
| **long-horizon agent RL** | 怎么在 100+ step agent task 上做 credit assignment？ | 复现 SWE-RL / Agent-RLVR |
| **PRM for agent** | step-level reward 能不能 transfer 到 agent 场景？ | 复现 Math-Shepherd 思路 in code agent |
| **self-play / self-improvement** | agent 怎么自我提升 without 标注？ | Voyager 启发 |
| **tool-use 训练** | 怎么训 model 学会"什么时候 call 什么 tool"？ | 复现 ToolFormer 改进 |
| **Multi-agent RL** | 多 agent 协作怎么训？ | 较前沿，paper 较少 |
| **Inference scaling** | 怎么 trade compute for performance at inference time？ | best-of-N / MCTS |
| **reward model 鲁棒性** | RM 容易被 over-optimize → 怎么 detect & fix？ | reward hacking 检测 |
| **对齐 vs 能力的 tension** | 对齐为什么会损害能力？怎么 mitigate？ | alignment tax 研究 |

> **面试黄金回答**："我对 X 方向感兴趣，因为 Y。我看到最近 Z paper 在做这块，但 [某个 limitation]。如果在 DeepSeek，我想 propose [具体 idea]。"

---

## 七、必背一组"DeepSeek 友好"数字

- DeepSeek-V3：671B 参数 / 37B 激活 / 14.8T 训练 tokens / FP8 训练
- DeepSeek-R1：纯 RL 训出 reasoning / GRPO / rule-based reward
- MLA：KV cache 减少 ~93%
- GRPO：相比 PPO 减少 ~50% RL 显存
- R1 训练成本：训练总耗费 < 600 万美元（公开称约 557 万美元 for V3 base）
- Open-R1 / TinyZero 等社区复现项目：可在小模型上验证 R1 思想

---

## 八、推荐这周通读的 7 篇必读 paper

| 序 | paper | 优先级 | 备注 |
| --- | --- | --- | --- |
| 1 | **DeepSeek-R1** 技术报告 | P0 | 必须能从头讲到尾 |
| 2 | **DeepSeek-V3** 技术报告 | P0 | 至少能讲架构 + 训练 recipe |
| 3 | **DPO** (Rafailov 2023) | P0 | 必须能推导 loss |
| 4 | **PPO** (Schulman 2017) | P0 | 必须能写 clipped surrogate objective |
| 5 | **Let's Verify Step by Step** | P1 | PRM 思想必懂 |
| 6 | **SWE-RL** 或 **Agent-RLVR** | P1 | agent RL 至少读一篇 |
| 7 | **Constitutional AI** | P2 | RLAIF 思想 |

> 每篇精读 1~2 小时，做 1 页 summary（problem / method / insight / limitation / 你的 take）。

---

下一文档：[03-核心算法速查](03-核心算法速查.md)
