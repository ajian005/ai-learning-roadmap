# 07 · 高频面试题·Paper 深读与研究品味篇

> 研究员面试**最高频且区分度最大的题型**——"讲个 paper" / "你怎么看 X paper" / "你最近 inspired 的 paper"。
>
> 这一份给 8 道核心题 + 答题模板 + 必备 5 篇 paper 的"30 秒 / 3 分钟 / 10 分钟"三档讲法。

---

## 一、答题"通用 4-part 模板"

> 任何 paper 题，按 4 part 答。每 part 30~60 秒，总长 2~4 分钟。

```
1. WHAT (30秒)
   - 题目 + venue + 一作 + 1 句核心 contribution
   - "X 在 Y 上提出 Z，证明 Z > baseline"

2. WHY (60秒)
   - 我为什么 inspired
   - 哪个具体 figure / result 让我思考
   - 这个 idea 如何 reshape 我对某个问题的理解

3. CRITIQUE (60秒)
   - 我同意 / 不同意什么
   - paper 没解决 / 没说清的地方
   - 可能的 confound / limitation

4. ACTION (30秒)
   - 我做了什么 follow up
   - 复刻 / 写 blog / 引用到自己工作
```

**评分标准**：
- WHAT 平淡 → "应试式"
- WHY + CRITIQUE 强 → "thinking like a researcher"
- ACTION 实 → "action-oriented researcher"

---

## 二、必准备的 5 篇"金句库"

> 每篇能 30s/3min/10min 三档讲——根据面试官追问深度切换。

### Paper A·DeepSeek-R1（自家，必背 + 加分）

**30 秒**：
> R1 是 DeepSeek 2025-01 旗舰 reasoning model。核心两点：(1) R1-Zero 证明纯 RL 不需 SFT 起步也能涌现 reasoning（"aha moment"）；(2) 引入 GRPO 替代 PPO，省 critic + sample efficient。distill 到小模型仍强——distilled-32B 接近 R1 671B 全模型。

**3 分钟**（加 method 和 ablation）：
> ...上面30秒+
> Method：R1-Zero 用纯 GRPO + rule-based reward（math 答案对/错、code unit test pass）。最让人意外是 ~6K step 后模型自发涌现 "wait, let me reconsider" 这种 reflection 模式——paper 第 5 节有截图。
>
> R1 是 R1-Zero 的产品化版本：加 cold-start SFT（5K 高质量 long CoT）→ multi-stage RL（reasoning + alignment） → distill 7B/14B/32B/70B。Trick 之一是防 language mixing（纯 RL 时模型偏好生成混合中英文）。

**10 分钟**（加 critique 和 implications）：
> ... 上面3分钟+
>
> **Critique**：
> 1. **Aha moment 是 robust property 还是 lucky?** Paper 用一个例子说明，但没做 systematic study。我倾向认为是 emergent—因为 distilled 版本（无 RL）也能学到 reflection 模式。
> 2. **GRPO 没和 PPO/RLOO 做严谨 ablation**——只是说"用 GRPO，效果好"，没量化 GRPO 相对 PPO 的优势具体值。
> 3. **Reward 设计未充分披露**——什么 task / 什么 weight，paper 模糊。研究界复刻很多 (TinyZero / Open-Reasoner-Zero) 部分确认部分质疑。
>
> **Implications**：
> - 证明 pure RL 路线（vs OpenAI 的"先 SFT 教 reasoning，再 RL polish"）也走得通 → alignment field 重大启示
> - GRPO 成为 RLHF 简化范式之一 → TRL / OpenRLHF / verl 都加了支持
> - Distillation 路线 + open ckpt → 7B reasoning 民主化
>
> **我的 follow up**：
> 我自己用 TRL 跑了一个 mini GRPO 在 GSM8K 上 finetune Qwen 0.5B，发现 group size=8 + KL coeff=0.04 是 sweet spot；reward shaping 必须有 partial credit 否则前期 advantage 全 0。代码在我 GitHub xxx。

---

### Paper B·DPO（必背）

**30 秒**：
> NeurIPS 2023 Outstanding Paper。把 RLHF 三阶段简化成一阶段——用 BT preference model + KL-constrained RL 最优解的闭式，直接在 preference data 上 supervised loss 训 policy。"Your LM is secretly a reward model"。

**3 分钟**：
> ...30秒+
> 数学核心：BT 给 $P(y_w \succ y_l) = \sigma(r(y_w) - r(y_l))$。RLHF KL-constrained 最优 policy 是 $\pi^* \propto \pi_{ref} \exp(r/\beta)$。反解 $r = \beta \log(\pi^*/\pi_{ref}) + \beta \log Z$。代入 BT，$Z$ 在两端相消 → 直接 MLE on preference pairs。
>
> 实验：sentiment / summarization / dialogue 上 ≈ PPO，但不需 RM + PPO 两阶段 → 简单稳定。

**10 分钟**（加 critique）：
> ...3分钟+
> **Critique**：
> 1. **DPO 是 supervised loss 但被叫 "RL"** — 实际是 off-policy supervised；这是 marketing 也是数学事实
> 2. **对噪声 sensitive**：preference data 标错率 10% 时 DPO 明显劣于 PPO + ensemble RM
> 3. **OOD generalization 差**：只 fit train pair distribution，外推到新 distribution 时 worse than RM-based PPO
> 4. **Industrial 实践**：OpenAI / Anthropic 至今没切 DPO 主线 → 侧面说明上限低于 PPO
> 5. **变种繁多** (IPO/KTO/SimPO/ORPO/CPO)：每个 patch DPO 一个 weakness，但说明 DPO 不是 final answer
>
> **Implication**：
> - 让 RLHF 民主化（不需 PPO 工程能力）
> - 启发 "preference learning as supervised" 一系列工作
> - Tülu 3 的 RLOO recipe 是 DPO 的工业改进
>
> **Follow up**：
> 我跑过 DPO vs RLOO 对比在 Llama-3-8B SFT 后 → DPO 训快但 win rate 比 RLOO 低 ~3pp。说明 DPO 是 fast iter 工具，不是 SOTA chaser。

---

### Paper C·OpenAI o1 / Reasoning model paradigm

**30 秒**：
> OpenAI 2024-09 release 的 reasoning model 系列首作（仅 blog 无论文）。核心：用 RL 训 long CoT thinking；test-time compute scaling（推理时间越长，AIME / GPQA 越高，log-linear）；隐藏 thinking token（用户只看 final answer）。

**3 分钟**（加 critique）：
> ...30秒+
>
> **Method 推测**：纯 RL on math/code/reasoning task（OpenAI 没公开 detail）。Test-time：模型自己 generate long thinking trace (几 K~100K token)，期间多次 self-correct/explore alternative。
>
> **Critique**：
> 1. **No paper, no detail**：reproducibility 困境
> 2. **Hidden CoT** 阻碍 alignment research
> 3. **Over-thinking 短问题**：简单题花几 K thinking token (cost 浪费)
> 4. **Cost**：o1-preview 比 GPT-4 贵 60×，普通 task 不划算
>
> **Implications**：
> - "test-time compute scaling" 成为新 scaling law
> - 倒逼 DeepSeek-R1 / Qwen-QwQ 复刻
> - Process reward + RL on reasoning 成主流

---

### Paper D·SWE-Bench / SWE-Bench Verified

**30 秒**：
> Jimenez et al. (ICLR 2024). 用真实 GitHub issue + 单测作 coding agent eval。Verified 是 OpenAI 后续 release 的人工 vetted subset。是当代 coding agent eval 黄金标准——Claude Code、Aider、Cursor、Cognition 都 report。

**3 分钟**：
> ...30秒+
>
> **Setup**：从 12 个 popular Python repo 抓 2294 个 (PR, issue) pair。Agent 给 repo + issue 描述，要 propose patch 让 PR test pass。
>
> **数据**：
> - SWE-Bench full: 2294 example
> - SWE-Bench Lite: 300 (curated)
> - SWE-Bench Verified: 500 (人审核，去掉错 issue/坏 test)
>
> **当前 SOTA** (2025-05)：~70%+ on Verified (Claude Sonnet 4.x + agent harness)
>
> **Critique**：
> 1. **Data leakage**：issue 都在 GitHub 公开，可能被 pretrain 数据包含 → "memorize" 风险
> 2. **Verified 仍有问题**：有 ambiguous test、underspecified issue
> 3. **Only Python**：单语言，泛化性？
> 4. **Single repo / 单 issue 边界明确**：不像真实 PM 那样跨多 repo
> 5. **Pass/Fail binary 太粗**：partial fix / 不完美 fix 没 credit
>
> **Implications**：
> - 推动 coding agent 整个 field 提速
> - 衍生 SWE-Bench Multilingual / SWE-Bench Pro / Multi-SWE-Bench
> - Bencher 和 modeler arms race

---

### Paper E·Anthropic Constitutional AI

**30 秒**：
> Anthropic 2022 提出。用 ~20 条 principles + AI self-critique-revise loop 替代部分 human RLHF。Stage 1 SL-CAI（critique-revise SFT），Stage 2 RL-CAI（AI preference RM + PPO）。是 RLAIF 范式开山。Claude 的"宪法"由来。

**3 分钟 + critique**：
> ...30秒+
>
> **Why important**：解决 human label 不可 scale 问题——AI judge 比人 100× 快 / 便宜。
>
> **Critique**：
> 1. **Principle 是英文 hard-coded** —— 跨文化 / 中文场景需要重设计；DeepSeek 应有自己中文 constitution
> 2. **Judge bias**：strong LLM 偏 verbose / format / formal，未校准会 hack
> 3. **Mode collapse risk**：iter 后 AI judge feedback 越来越 narrow → policy 趋同
> 4. **Helpful vs Harmless trade-off**：CAI 容易学 over-refuse
>
> **Implication**：
> - 现代所有 alignment lab 都用 RLAIF + mixed human
> - 启发 self-rewarding LLM / iterative DPO 路线
> - "rule-based safety" 替代 "supervised safety" 思想根源

---

## 三、8 道高频题

### Q1·最近 6 个月最让你 inspired 的 paper？

**模板**：用 Paper A (R1) 或 Paper D (SWE-Bench)，4-part 答。

**Tip**：
- 选**自己真读过 + 有 critical take 的**——不要选 hot 但你只看 abstract 的
- 第一选 DeepSeek 自家（R1/V3）——直接 align 公司
- 第二选 alignment classic（DPO/InstructGPT）——证明基础扎实
- 第三选 Agent 相关（SWE-Bench/Reflexion）——证明 hit JD

**反诘**：那这个 paper 启发你做了什么？  
A: 一定要有 follow up（复刻 / blog / idea write up）——空说"inspired"不加分。

---

### Q2·你最不同意的 paper？为什么？

**核心**：选**有争议**的 paper，给**有依据**的 critique。

**安全选项**：
- **Self-Rewarding LLM** (Yuan 2024)：本质 self-reinforcing bias，iter 后 hack；Anthropic 11 月 harness paper 推 separate evaluator 正是 push back
- **Magpie** (data 合成)：data 模型 distribution-bound，"自举"概念在数学上不可能严格成立
- **某些 "1B model beats GPT-4" 论文**：通常 cherry-pick benchmark；scrutinize sample efficiency / OOD 后翻盘

**模板**：
> "我对 [paper] 持保留——主要因为 [3 个具体 reason]。每个 reason 我用 [evidence: paper 第 N 节 / 我自己复刻 / 别人 follow up paper] 支持。"

**反诘**：你怎么提出 better alternative？  
A: 提 1~2 个 hypothesis（不必 mature），表明你 thinking forward。

---

### Q3·给一个 vague paper（or "PRM 在 reasoning model 里效果"），你怎么评价？

**框架**：

```
1. Paper 想解决什么 problem
2. Method 是什么（哪 1~2 个 key innovation）
3. Main result 强度（vs prior baseline 多少）
4. Ablation 充分吗（去掉每个 component 影响）
5. 复刻友好吗（code / data / hardware）
6. 可能的 limitation / future work
```

**Tip**：研究员经常 "30 秒 scan abstract → 决定 read more"——面试时也演这一手。

---

### Q4·你怎么判断哪些 paper 值得 follow？

**优先级（我的过滤器）**：

| 优先级 | 标准 |
| --- | --- |
| 1 | **顶会 + 高 citation/year**（NeurIPS/ICML/ICLR/ACL）|
| 2 | **顶 lab paper**（OpenAI/Anthropic/DeepMind/DeepSeek/Meta/Tülu）|
| 3 | **解决我当前 work 的具体痛点**（high relevance）|
| 4 | **被 Twitter 顶级人物 endorse**（Karpathy/Lilian Weng/Stas Bekman/Hamel）|
| 5 | **复刻友好**（open code + data + recipe）|

**反过滤器**：

- 标题 "X beats GPT-4" 但只在一个 narrow benchmark → 谨慎
- 没 ablation / 没 hyperparameter detail → red flag
- 只有 figure，没 number → red flag

---

### Q5·复刻一个 paper 失败，怎么办？

**经典故事**：

> "Y month 我尝试复刻 X paper，按 paper 里 hyperparameter 跑，结果 baseline-only 而不是 paper claim 的 10pp 提升。我做了 3 件事：
>
> 1. **Cross-check details**：发邮件给作者 → 作者回复 'use cosine LR schedule with warmup 1K step'，paper 没说
> 2. **Sweep hyperparameter**：发现 batch size 必须 ≥ 256 才效果显著
> 3. **Look at exact data prep**：发现 paper 的 deduplication 比我严格 — 改后 close gap
>
> 最终复出 ~80% 提升，写了一篇 blog 总结 'X paper replication note'，作者后续 endorsement。"

**Key 点**：研究员看你**能不能 debug 实验** + **能不能不放弃** + **能不能写成 contribution**。

---

### Q6·你怎么决定 paper idea 值得发？

**4 个判断标准**：

1. **Novelty**：你解决一个 prior 没解决的问题，或用新角度解决老问题
2. **Significance**：解决这个问题 matters（业界 / academia 关心）
3. **Rigor**：method 数学/实验严谨
4. **Generalizability**：不仅 toy task，能推广

**Negative signals**：

- 只在一个 narrow benchmark 提升
- 提升幅度小（<2pp）且没 statistical test
- Hyperparameter sweep 偷偷做了 dev set
- 没 ablation

---

### Q7·讲一个你"3 分钟读完决定 pass" 的 paper

**示例**：

> "上周看到 X paper claim 'novel attention 比 standard 快 10×'。3 分钟过：
>
> - Abstract：claim 10× speedup
> - Figure 1：speedup 在 batch=1 + seq=128 下测的
> - Method：本质是 sparse attention 的小改
> - Table 2：没和 FlashAttention 比，只和 standard attention 比
>
> Pass 原因：(1) 测试 setting 不 production； (2) 没 strong baseline (FlashAttn)；(3) 已有大量 sparse attn paper。我标记 'not follow' 继续看下一篇。"

**signal**：你能**快速过滤**，不浪费时间。

---

### Q8·你怎么追前沿但不被 hype 带偏？

**5 个原则**：

1. **首选 venue / lab 而非 hype**：顶会 outstanding + top lab paper 优先
2. **delay 1 周**：让 Twitter 热度过 → 看是否 still talked about
3. **看 follow up**：如果一个月没人 follow → 可能是 narrow
4. **复刻 sanity check**：自己跑 mini 复刻看是否 robust
5. **maintain 一个 personal reading queue**（Notion / arxiv-sanity）：而不是被推送 dictate

---

## 四、面试官追问最爱的 5 个角度

> 这些追问是 "你真读过吗" 的 calibration。提前练。

1. **"具体 figure N 想说什么？"** — 你必须记住 main figure
2. **"如果去掉 component X，性能怎么变？"** — ablation 表
3. **"这个方法 fails 在什么场景？"** — limitation
4. **"为什么作者选这个 baseline 而不是 Y？"** — methodology critique
5. **"你能在 2 分钟内推导 main equation 吗？"** — 数学深度

---

## 五、研究员 paper 库的"分层维护"

```
Tier S (烂熟于心)：5~10 篇——能 10 分钟讲完整推导和 critique
Tier A (熟知)：20~30 篇——能 3 分钟讲核心 + critique
Tier B (知道存在)：50~100 篇——能 30 秒讲 abstract
Tier C (待读)：reading list 100+ 篇
```

**面试前**：Tier S 至少 5 篇能背 + 至少 2 篇能讲到 critique。

---

## 六、最后一句

> **能讲 paper 是 mid-tier researcher；能 critique paper 是 senior researcher；能从 critique 看到 next research thread 是 staff researcher。**

> **DeepSeek 招的是 senior +——你的"critique"质量是核心 signal。**
