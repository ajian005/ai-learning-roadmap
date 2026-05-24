# 12｜自带作品集 — Mini RL / Alignment 实验项目（杀手锏）

> 这一篇是**研究员面试的最大差异化项**。能在 7~10 天内交付一个**从头跑通 + 有 ablation + 有 blog** 的 mini RL 项目——你已经领先 80% 候选人。
>
> 本文给：项目蓝图 + 代码结构 + 关键 snippet + README 模板 + blog 模板。

---

## 一、为什么这是"杀手锏"

DeepSeek 研究员候选池里：
- 能背 PPO/DPO/GRPO 公式的 ~ 60%
- 真正自己 implement 过 loss 跑过收敛 curve 的 ~ 15%
- 跑过 + 有 ablation + 有 blog 的 ~ 3%

→ **你只要进 top 3%，offer 大幅提高**。

---

## 二、项目选型 — 推荐两条路线

### 路线 A（保守稳赢）：Mini DPO

**优点**：
- 算法简单（pure offline，类似 SFT loop）
- 7 天能跑通
- 显存友好（0.5B 模型够）
- DPO 是过去 2 年 alignment 必读

**缺点**：
- DPO 这两年很热，候选人也多——差异化稍弱

### 路线 B（高 ROI 高风险）：Mini GRPO on Reasoning

**优点**：
- 直接和 DeepSeek-R1 对标 ⭐⭐⭐
- 涉及 RL sampling、rule-based reward、long generation——研究员看了直接 wow
- 复现 DeepSeek 自家算法 = 完美 fit

**缺点**：
- 算法复杂（online RL，需要 sampling infra）
- 需要 GPU 资源（7B 模型 + GRPO 需 80GB）
- 容易 stuck 在 reward 不上升

### 推荐策略

```
T-14 ~ T-9: 做 Mini DPO (保证有东西)
T-9 ~ T-5: 如果还有时间, 加 Mini GRPO
T-5 后: 不再加新东西, 只 polish
```

> **下面以 Mini DPO 为主线**，最后给 Mini GRPO 的关键差异。

---

## 三、Mini DPO 项目蓝图

### 目标

```
1 句话: "从头实现 DPO trainer，在 Qwen-0.5B 上用 HH-RLHF 1K pair 训练，
        证明 DPO 比 SFT-only 的 win rate 高，可视化收敛曲线 + ablation。"
```

### 关键指标（可写进 README）

| 指标 | 期望 |
| --- | --- |
| `reward_acc` (chosen > rejected 的频率) | 训前 ~0.50 → 训后 > 0.70 |
| `reward_margin` | 从 0 涨到 > 1.0 |
| MT-Bench / AlpacaEval subset win rate | DPO > SFT by 5~15% |
| 训练时长 | < 30 min on 1×A100 |
| 代码量 | < 500 行 (不含 dataset/eval) |

### Repo 结构

```
mini-dpo/
├── README.md
├── requirements.txt
├── configs/
│   └── dpo_qwen_05b.yaml
├── data/
│   ├── prepare_hh.py       # 下载 + 预处理 HH-RLHF
│   └── sample_1k.jsonl     # cached subset
├── src/
│   ├── dpo_loss.py         # 核心：DPO loss + 算 logp
│   ├── dpo_trainer.py      # 训练 loop
│   ├── eval.py             # win rate eval (GPT-4 judge)
│   └── utils.py
├── scripts/
│   ├── train_sft.sh        # baseline
│   ├── train_dpo.sh
│   └── run_eval.sh
├── notebooks/
│   ├── 01_explore_data.ipynb
│   ├── 02_smoke_test.ipynb
│   └── 03_plot_results.ipynb
├── results/
│   ├── training_curves/    # W&B 截图
│   ├── ablations/          # ablation 表
│   └── win_rates.json
└── BLOG.md                 # 1500 字技术 blog
```

---

## 四、核心代码 snippet

### `src/dpo_loss.py` (~80 行)

```python
"""Pure DPO loss implementation - reproducible."""
import torch
import torch.nn.functional as F
from typing import Tuple


def compute_response_logp(
    model: torch.nn.Module,
    input_ids: torch.Tensor,        # [B, T]
    response_mask: torch.Tensor,    # [B, T] 1 for response tokens
    requires_grad: bool = True,
) -> torch.Tensor:
    """Sum log-prob over response tokens."""
    ctx = torch.enable_grad() if requires_grad else torch.no_grad()
    with ctx:
        logits = model(input_ids).logits  # [B, T, V]
        log_probs = F.log_softmax(logits[:, :-1, :], dim=-1)
        labels = input_ids[:, 1:]
        token_logp = torch.gather(log_probs, dim=-1, index=labels.unsqueeze(-1)).squeeze(-1)
        mask = response_mask[:, 1:].float()
        return (token_logp * mask).sum(dim=-1)


def dpo_loss(
    policy_logp_chosen: torch.Tensor,
    policy_logp_rejected: torch.Tensor,
    ref_logp_chosen: torch.Tensor,
    ref_logp_rejected: torch.Tensor,
    beta: float = 0.1,
) -> Tuple[torch.Tensor, dict]:
    """
    Returns:
        loss: scalar
        metrics: dict for logging
    """
    pi_log_ratio_w = policy_logp_chosen - ref_logp_chosen
    pi_log_ratio_l = policy_logp_rejected - ref_logp_rejected
    
    logits = beta * (pi_log_ratio_w - pi_log_ratio_l)
    loss = -F.logsigmoid(logits).mean()
    
    with torch.no_grad():
        reward_chosen = beta * pi_log_ratio_w
        reward_rejected = beta * pi_log_ratio_l
        reward_margin = (reward_chosen - reward_rejected).mean()
        reward_acc = (reward_chosen > reward_rejected).float().mean()
    
    return loss, {
        "loss": loss.item(),
        "reward_chosen": reward_chosen.mean().item(),
        "reward_rejected": reward_rejected.mean().item(),
        "reward_margin": reward_margin.item(),
        "reward_acc": reward_acc.item(),
        "logits_mean": logits.mean().item(),
    }
```

### `src/dpo_trainer.py` (~150 行)

```python
"""DPO training loop with W&B logging."""
import torch
from transformers import AutoModelForCausalLM, AutoTokenizer
from torch.utils.data import DataLoader
import wandb
from copy import deepcopy
from src.dpo_loss import dpo_loss, compute_response_logp


def train_dpo(
    model_name: str,
    dataset,
    config: dict,
):
    wandb.init(project="mini-dpo", config=config)
    
    tokenizer = AutoTokenizer.from_pretrained(model_name)
    if tokenizer.pad_token is None:
        tokenizer.pad_token = tokenizer.eos_token
    
    policy = AutoModelForCausalLM.from_pretrained(
        model_name, torch_dtype=torch.bfloat16
    ).cuda()
    
    ref = AutoModelForCausalLM.from_pretrained(
        model_name, torch_dtype=torch.bfloat16
    ).cuda()
    ref.eval()
    for p in ref.parameters():
        p.requires_grad = False
    
    optimizer = torch.optim.AdamW(
        policy.parameters(),
        lr=config["lr"],
        weight_decay=0.0,
    )
    
    loader = DataLoader(
        dataset,
        batch_size=config["batch_size"],
        shuffle=True,
        collate_fn=lambda b: collate(b, tokenizer, config["max_len"]),
    )
    
    step = 0
    for epoch in range(config["epochs"]):
        for batch in loader:
            batch = {k: v.cuda() for k, v in batch.items()}
            
            pi_logp_w = compute_response_logp(
                policy, batch["chosen_ids"], batch["chosen_mask"]
            )
            pi_logp_l = compute_response_logp(
                policy, batch["rejected_ids"], batch["rejected_mask"]
            )
            
            with torch.no_grad():
                ref_logp_w = compute_response_logp(
                    ref, batch["chosen_ids"], batch["chosen_mask"], requires_grad=False
                )
                ref_logp_l = compute_response_logp(
                    ref, batch["rejected_ids"], batch["rejected_mask"], requires_grad=False
                )
            
            loss, metrics = dpo_loss(
                pi_logp_w, pi_logp_l, ref_logp_w, ref_logp_l, beta=config["beta"]
            )
            
            loss.backward()
            torch.nn.utils.clip_grad_norm_(policy.parameters(), 1.0)
            optimizer.step()
            optimizer.zero_grad()
            
            step += 1
            wandb.log({**metrics, "step": step, "epoch": epoch})
            
            if step % 20 == 0:
                print(f"step={step} loss={metrics['loss']:.4f} "
                      f"acc={metrics['reward_acc']:.3f} "
                      f"margin={metrics['reward_margin']:.3f}")
    
    policy.save_pretrained(config["output_dir"])
    tokenizer.save_pretrained(config["output_dir"])
    wandb.finish()


def collate(batch, tokenizer, max_len):
    """Pad to max in batch, build response_mask."""
    out = {}
    for key in ["chosen", "rejected"]:
        prompts = [b["prompt"] for b in batch]
        responses = [b[key] for b in batch]
        
        full_texts = [p + r for p, r in zip(prompts, responses)]
        enc = tokenizer(full_texts, return_tensors="pt", padding=True, truncation=True, max_length=max_len)
        
        response_mask = torch.zeros_like(enc.input_ids)
        for i, p in enumerate(prompts):
            p_len = len(tokenizer(p).input_ids)
            response_mask[i, p_len:] = 1
        response_mask = response_mask * enc.attention_mask
        
        out[f"{key}_ids"] = enc.input_ids
        out[f"{key}_mask"] = response_mask
    
    return out
```

### `src/eval.py` (~100 行) — GPT-4 judge win rate

```python
"""Evaluate win rate using GPT-4 as judge."""
import json
from openai import OpenAI
from transformers import AutoModelForCausalLM, AutoTokenizer
import torch


JUDGE_PROMPT = """You are evaluating two AI assistants' responses to the same user query.

User query: {prompt}

Assistant A response: {response_a}

Assistant B response: {response_b}

Which response is better? Consider helpfulness, harmlessness, and quality.
Reply with exactly one of: "A", "B", or "TIE"."""


def generate_response(model, tokenizer, prompt: str, max_new_tokens=256) -> str:
    inputs = tokenizer(prompt, return_tensors="pt").to(model.device)
    with torch.no_grad():
        out = model.generate(
            **inputs,
            max_new_tokens=max_new_tokens,
            do_sample=True,
            temperature=0.7,
            top_p=0.9,
        )
    return tokenizer.decode(out[0][inputs.input_ids.shape[1]:], skip_special_tokens=True)


def judge_pair(client, prompt: str, response_a: str, response_b: str) -> str:
    resp = client.chat.completions.create(
        model="gpt-4o-mini",
        messages=[
            {"role": "user", "content": JUDGE_PROMPT.format(
                prompt=prompt, response_a=response_a, response_b=response_b
            )}
        ],
        temperature=0,
    )
    label = resp.choices[0].message.content.strip().upper()
    if label not in ["A", "B", "TIE"]:
        label = "TIE"
    return label


def compute_win_rate(model_a, model_b, prompts, tokenizer, openai_client):
    """Returns (a_wins, b_wins, ties)."""
    results = {"A": 0, "B": 0, "TIE": 0}
    for prompt in prompts:
        resp_a = generate_response(model_a, tokenizer, prompt)
        resp_b = generate_response(model_b, tokenizer, prompt)
        
        label = judge_pair(openai_client, prompt, resp_a, resp_b)
        results[label] += 1
        
        flipped = judge_pair(openai_client, prompt, resp_b, resp_a)
        if flipped == "A":
            results["B"] += 1
        elif flipped == "B":
            results["A"] += 1
        else:
            results["TIE"] += 1
    
    total = sum(results.values())
    return {k: v / total for k, v in results.items()}
```

### `configs/dpo_qwen_05b.yaml`

```yaml
model_name: "Qwen/Qwen2.5-0.5B-Instruct"
output_dir: "outputs/dpo_qwen_05b"
batch_size: 2
gradient_accumulation: 4
lr: 5e-7
beta: 0.1
epochs: 1
max_len: 1024
dataset_size: 1000
seed: 42
```

---

## 五、Ablation 计划（必做）

```
跑 5~6 个 config 对照:

| Config        | β   | lr   | data  | reward_acc | win_rate vs SFT |
| ------------- | --- | ---- | ----- | ---------- | --------------- |
| baseline SFT  | -   | 5e-6 | 1K    | -          | 50% (ref)       |
| dpo β=0.1     | 0.1 | 5e-7 | 1K    | 0.72       | 58%             |
| dpo β=0.5     | 0.5 | 5e-7 | 1K    | 0.65       | 53%             |
| dpo β=0.01    | 0.01| 5e-7 | 1K    | 0.80       | 55% (overfits)  |
| dpo + 5K data | 0.1 | 5e-7 | 5K    | 0.78       | 62%             |
| dpo 3 epoch   | 0.1 | 5e-7 | 1K    | 0.85       | 56% (overfits)  |
```

**Insights from ablation**（写进 BLOG）：
- β 小（0.01）→ reward_acc 高但 win_rate 反而低 = 过拟合到 preference data
- 数据多 → win_rate 提升明显
- 训练长（多 epoch）→ overfit
- 推荐 sweet spot: β=0.1, 1 epoch, 5K+ data

---

## 六、README 模板

````markdown
# Mini DPO

A minimal, self-contained, reproducible implementation of Direct Preference Optimization (DPO) on Qwen-0.5B with Anthropic HH-RLHF.

**Done in ~7 days** as a learning + interview prep project. Code under 500 LOC excluding data + eval.

## Why this exists

- I wanted to deeply understand DPO end-to-end, not just `pip install trl && trainer.train()`
- This repo: reimplements DPO loss from first principles, walks through derivation in code comments, includes ablations + blog write-up

## Results

| Model           | reward_acc | reward_margin | Win rate vs SFT (GPT-4 judge) |
|-----------------|-----------|---------------|-------------------------------|
| Qwen-0.5B SFT (baseline) | -    | -    | 50% (ref) |
| Qwen-0.5B + DPO          | 0.72 | 1.4  | **58%** |
| Qwen-0.5B + DPO + 5K data | 0.78 | 1.7 | **62%** |

[Training curves](./results/training_curves/) and [ablation table](./results/ablations/).

## Quick start

```bash
pip install -r requirements.txt
python data/prepare_hh.py
bash scripts/train_dpo.sh
bash scripts/run_eval.sh
```

## Repo layout

[tree as above]

## Key files

- `src/dpo_loss.py` - DPO loss with full derivation comments
- `src/dpo_trainer.py` - Training loop, < 150 LOC

## Ablations

See [BLOG.md](./BLOG.md) for full discussion. TL;DR:
- β=0.1 is sweet spot; β too small → overfits to preference data
- 1 epoch optimal; multi-epoch overfits despite higher reward_acc
- Data quantity > preference label quality

## Lessons learned

1. **DPO is deceptively simple but tricky to train well** - β tuning is critical
2. **reward_acc ≠ win_rate** - high reward_acc can hide overfitting
3. **Reference model freezing matters** - I once forgot to `eval()` ref and got 50% slower training

## What I'd do next

- Try IPO/SimPO and compare
- Iterative DPO with online sampling
- Scale to 7B and re-run ablations
- Step-level PRM for reasoning datasets

## Citation

Built as part of DeepSeek Agent Researcher interview prep, May 2026.
```
````

---

## 七、BLOG 模板（~1500 字）

```markdown
# Reproducing DPO in 500 Lines: What I Learned

[Date]

## Why DPO?

[200 words - 解释 RLHF 复杂性 + DPO 的简化]

## My setup

- Qwen-0.5B-Instruct as base
- Anthropic HH-RLHF, 1K (later 5K) preference pairs
- 1×A100 80GB
- Bf16 training

## DPO in 1 page math

[200 words - 推导从 RLHF objective 到 DPO loss]

## Implementation walkthrough

[300 words + code snippets - 关键的 logp computation + loss + ref freeze]

## Training curves

[200 words + screenshots - reward_acc / reward_margin / loss over steps]

## Ablations

[300 words - 5 个 config 对比 + insights]

## Surprises

1. `reward_acc` 涨快但 win_rate 不一定涨 — overfitting signal
2. β=0.01 时 chosen logp 反而降（likelihood displacement 现象）
3. ref model 必须 `eval()` + `requires_grad=False` 否则 OOM

## Limitations + Future work

- 没跑 IPO/SimPO 对比
- Win rate 用 GPT-4 judge 而不是 human
- 0.5B 模型小，scaling 到 7B+ 可能 dynamic 不同

## Takeaway

[100 words - 给读者的核心 insight]

## Repo

[Link to GitHub]
```

---

## 八、Mini GRPO 路线（如果时间够）

### 关键差异 vs Mini DPO

| | Mini DPO | Mini GRPO |
| --- | --- | --- |
| 算法 | offline classification | online RL with sampling |
| Reward | preference label | rule-based (e.g., math answer correct) |
| 显存 | 1 base model | policy + ref (no critic) + sampling KV cache |
| 数据 | preference pairs | reasoning prompts |
| Eval | win rate | accuracy on held-out math |

### 推荐设置

```yaml
model_name: "Qwen/Qwen2.5-Math-1.5B-Instruct"
dataset: "GSM8K" or "MATH-500"
group_size: 8
beta: 0.04
lr: 1e-6
sampling_temperature: 0.7
max_response_len: 1024
total_steps: 500
```

### 关键 snippet（GRPO 核心 loop）

```python
def grpo_step(policy, ref, prompts, reward_fn, config):
    """One GRPO update step."""
    # 1. Sample G responses per prompt
    all_responses = []
    for prompt in prompts:
        responses = sample_n(policy, prompt, n=config.group_size, 
                             temperature=config.temp, max_len=config.max_len)
        all_responses.append(responses)
    
    # 2. Compute rewards
    all_rewards = [[reward_fn(p, r) for r in resps] 
                   for p, resps in zip(prompts, all_responses)]
    
    # 3. Group-normalize advantages
    advantages = []
    for rewards in all_rewards:
        rewards_t = torch.tensor(rewards)
        mean = rewards_t.mean()
        std = rewards_t.std() + 1e-8
        adv = (rewards_t - mean) / std
        advantages.append(adv)
    
    # 4. Recompute logp under current + reference + take ratio
    policy_loss = 0
    kl_loss = 0
    for prompt, responses, advs in zip(prompts, all_responses, advantages):
        for resp, adv in zip(responses, advs):
            input_ids, response_mask = make_inputs(prompt, resp)
            
            new_logp = compute_response_logp(policy, input_ids, response_mask)
            with torch.no_grad():
                ref_logp = compute_response_logp(ref, input_ids, response_mask, requires_grad=False)
                old_logp = new_logp.detach()
            
            ratio = (new_logp - old_logp).exp()
            policy_loss += -torch.min(
                ratio * adv,
                ratio.clamp(1 - config.clip_eps, 1 + config.clip_eps) * adv
            ).sum()
            
            log_r = ref_logp - new_logp
            kl_loss += (log_r.exp() - log_r - 1).sum()
    
    loss = (policy_loss + config.beta * kl_loss) / len(prompts)
    return loss


def reward_fn(prompt: str, response: str) -> float:
    """Rule-based reward for math: 1 if final answer correct else 0."""
    extracted = extract_answer(response)
    target = extract_answer_from_prompt(prompt)
    return 1.0 if extracted == target else 0.0
```

### 关键挑战 + 解决

| 挑战 | 解决 |
| --- | --- |
| Sampling 慢 | 用 vLLM 做 inference，policy update 间隔 sync 一次 |
| Reward 几乎全 0 | 数据筛选——只用 model 当前能解对约 30~70% 的题 |
| KL 爆炸 | β 调小到 0.001~0.04；监控 KL 阈值 |
| 显存炸 | 用 gradient checkpointing + bf16 + 短 max_len 起步 |
| 收敛慢 | 接受——RL 本就慢，先看 reward mean 曲线趋势 |

---

## 九、面试中如何"使用"这个项目

### 5 秒电梯演讲

```
"我做了一个 mini DPO 实现，从 paper 推导到代码 < 500 行，
用 Qwen-0.5B + HH-RLHF 1K data 训练，win rate 比 SFT 高 8%。
还做了 5 个 ablation 写了 blog——你想看 repo 或 results 吗?"
```

### 算法面被问 DPO

→ "我自己 implement 过——`pi_logp - ref_logp` 在 chosen / rejected 都算……" → **真实经验的细节自然流出**

### Paper 面被问 DPO 限制

→ "我训练时观察到 likelihood displacement——β 调到 0.01 时 chosen logp 反而下降……" → **不是从书上看的，是自己跑出来的**

### Behavioral 面被问 "0→1 项目"

→ 直接讲这个 mini-DPO 的 7 天 journey——problem → setup → debug → ablation → blog

### 反问环节

→ "我有兴趣听 DeepSeek 自家 GRPO infra 怎么解 X 问题——我在 mini-GRPO 实现里遇到 [具体技术问题] 卡了 2 天……"

---

## 十、风险 + 备选

| 风险 | 备选 |
| --- | --- |
| 7 天没跑通 | 退回到只做"读 + summary"，不强行带不完整项目 |
| GPU 不够 | 用 Colab Pro / Lambda Cloud / 朋友 cluster |
| 收敛不好 | 写"negative result + 我的 debug 过程"——也是 valuable |
| Win rate 没涨 | 报告诚实——研究员看你的 process > 结果 |
| 怕代码被 copy | 训练时 private repo，T-3 切公开 |

---

## 十一、备选项目（如果 RL 时间不够）

如果完全做不出 RL，可以选这些**轻量替代**：

| 项目 | 工作量 | 信号 |
| --- | --- | --- |
| Mini SFT trainer + LoRA | 3 天 | 中 |
| Implement Flash Attention from scratch | 3 天 | 中 |
| Reproduce 1 个小 paper's main figure | 4 天 | 中高 |
| LLM eval harness clone (mini-version) | 3 天 | 中 |
| Custom Triton kernel + benchmark | 2 天 | 中 |
| Detailed paper deep-dive + critique blog | 2 天 | 中 |

**信号最高的是 RL 项目**——但有总比没好。

---

## 十二、Final checklist (T-1 前)

- [ ] Repo public 且 README 完整
- [ ] requirements.txt + 1 个 colab/notebook 可一键复现
- [ ] 训练 curve 截图清楚
- [ ] Ablation 表 + 5 个 config 数字
- [ ] BLOG.md 1500 字
- [ ] LinkedIn / GitHub bio 加链接
- [ ] 自己能在 5 分钟内 walkthrough
- [ ] 想好 "如果面试官现场问 X / Y 实现细节" 怎么答

---

> **完成这个项目本身不一定让你拿 offer——但你 will 成为更好的研究员。**
> 
> **DeepSeek 的研究员，每周都在做这种 mini experiment。把这件事 normalize 进你的日常，比 offer 更重要。**

下一文档：[99-四岗位差异对照与选择建议](99-四岗位差异对照与选择建议.md)
