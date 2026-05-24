# 08 · 高频面试题·编码与 PPO/RLHF 实现篇

> 研究员的 coding 面**不像工程师**——不要求你写 production-ready 系统，要求你能 **30~45 分钟内白板/在线手撕 PPO loss / DPO loss / RM 训练 loop / sampling loop**。
>
> 评分标准是 "**数学正确 + 关键 trick 体现 + 能 debug 自己代码 + 跑过一遍**"，而不是 "代码风格"。
>
> 6 题分 2 组：
> - A 组：核心 loss 实现（3 题）
> - B 组：训练 / sampling loop（3 题）

---

## A 组·核心 loss 实现

### Q1·白板手撕 PPO loss（含 GAE）

**题目**：用 PyTorch 实现 PPO 的 clipped surrogate loss + value loss + entropy bonus，并写一个 GAE 计算函数。

**参考实现**：

```python
import torch
import torch.nn.functional as F

def compute_gae(
    rewards: torch.Tensor,        # (T,)
    values: torch.Tensor,          # (T+1,)  # 多一位是 boostrap V(s_{T+1})
    masks: torch.Tensor,           # (T,) 0/1, 0 表示 episode 结束
    gamma: float = 0.99,
    lam: float = 0.95,
) -> tuple[torch.Tensor, torch.Tensor]:
    """
    Returns:
      advantages: (T,) - GAE advantage
      returns: (T,) - bootstrapped returns for value training
    """
    T = rewards.shape[0]
    advantages = torch.zeros_like(rewards)
    gae = 0.0
    for t in reversed(range(T)):
        delta = rewards[t] + gamma * values[t+1] * masks[t] - values[t]
        gae = delta + gamma * lam * masks[t] * gae
        advantages[t] = gae
    returns = advantages + values[:-1]
    return advantages, returns


def ppo_loss(
    logprobs_new: torch.Tensor,   # (B, T)  current policy log p
    logprobs_old: torch.Tensor,   # (B, T)  rollout-time log p
    advantages: torch.Tensor,     # (B, T)
    returns: torch.Tensor,        # (B, T)
    values_new: torch.Tensor,     # (B, T)  current critic prediction
    values_old: torch.Tensor,     # (B, T)  rollout-time value (for clipping)
    entropy: torch.Tensor,        # (B, T)
    mask: torch.Tensor,           # (B, T)  pad mask (1 = real token)
    eps: float = 0.2,
    vf_coef: float = 0.5,
    ent_coef: float = 0.01,
    vf_clip: float = 0.2,
) -> dict:
    # 1. Policy loss (clipped surrogate)
    log_ratio = logprobs_new - logprobs_old
    ratio = torch.exp(log_ratio)
    # advantage normalization (per batch)
    adv = (advantages - advantages.mean()) / (advantages.std() + 1e-8)
    surr1 = ratio * adv
    surr2 = torch.clamp(ratio, 1 - eps, 1 + eps) * adv
    pg_loss = -torch.min(surr1, surr2)
    pg_loss = (pg_loss * mask).sum() / mask.sum()

    # 2. Value loss (clipped)
    v_clipped = values_old + torch.clamp(values_new - values_old, -vf_clip, vf_clip)
    vf_loss1 = (values_new - returns) ** 2
    vf_loss2 = (v_clipped - returns) ** 2
    vf_loss = 0.5 * torch.max(vf_loss1, vf_loss2)
    vf_loss = (vf_loss * mask).sum() / mask.sum()

    # 3. Entropy bonus
    ent_loss = -(entropy * mask).sum() / mask.sum()

    total = pg_loss + vf_coef * vf_loss + ent_coef * ent_loss

    # 4. Metrics
    with torch.no_grad():
        approx_kl = ((ratio - 1) - log_ratio).mean()  # Schulman blog unbiased
        clip_frac = ((ratio - 1).abs() > eps).float().mean()
    return {
        "loss": total,
        "pg_loss": pg_loss,
        "vf_loss": vf_loss,
        "ent_loss": ent_loss,
        "approx_kl": approx_kl,
        "clip_frac": clip_frac,
    }
```

**面试时口述要点**：

1. **GAE 是 reverse loop**——从最后一步算 advantage，避免 O(T²)
2. **Advantage normalization** in-batch（mean=0, std=1）显著稳定
3. **min(surr1, surr2)** 取 pessimistic side
4. **Value clipping** 是 trick，不在原 paper，但 OpenAI baselines 用过
5. **Approx KL 用 Schulman unbiased estimator** $(r-1) - \log r$ 而非 naive
6. **Mask 是 LLM 必需** —— pad token 不能算 loss

**易错点**：
- 忘记 mask pad token → NaN
- Advantage 没 detach → 梯度污染
- 用 `clip` 而非 `clamp` (PyTorch API)
- Reward / advantage 没 normalize → 训练不稳

**反诘 Q**：为什么 value clipping？  
A: Value 也可能跟 policy 一样跑偏；clipping value 防止 critic 学坏；和 policy clip 镜像。

---

### Q2·白板手撕 DPO loss

**题目**：实现 DPO loss + log probability 计算（含 prompt mask）。

**参考实现**：

```python
import torch
import torch.nn.functional as F

def compute_logps(
    logits: torch.Tensor,         # (B, T, V)
    labels: torch.Tensor,         # (B, T)  -100 表示 prompt token 不算
) -> torch.Tensor:
    """
    Returns: per-sample sum of log p(token) over response tokens
    Shape: (B,)
    """
    log_probs = F.log_softmax(logits[:, :-1, :], dim=-1)   # next-token pred
    labels_shifted = labels[:, 1:].clone()
    loss_mask = (labels_shifted != -100)
    labels_shifted[~loss_mask] = 0    # gather 不能用 -100
    per_token_logp = log_probs.gather(2, labels_shifted.unsqueeze(2)).squeeze(2)
    per_token_logp = per_token_logp * loss_mask
    return per_token_logp.sum(dim=1)


def dpo_loss(
    pi_chosen_logps: torch.Tensor,    # (B,)
    pi_rejected_logps: torch.Tensor,  # (B,)
    ref_chosen_logps: torch.Tensor,   # (B,)  no_grad
    ref_rejected_logps: torch.Tensor, # (B,)  no_grad
    beta: float = 0.1,
) -> dict:
    pi_logratios = pi_chosen_logps - pi_rejected_logps
    ref_logratios = ref_chosen_logps - ref_rejected_logps
    logits = beta * (pi_logratios - ref_logratios)
    loss = -F.logsigmoid(logits).mean()

    with torch.no_grad():
        chosen_reward = beta * (pi_chosen_logps - ref_chosen_logps)
        rejected_reward = beta * (pi_rejected_logps - ref_rejected_logps)
        reward_margin = (chosen_reward - rejected_reward).mean()
        win_rate = (chosen_reward > rejected_reward).float().mean()
    return {
        "loss": loss,
        "chosen_reward": chosen_reward.mean(),
        "rejected_reward": rejected_reward.mean(),
        "margin": reward_margin,
        "win_rate": win_rate,
    }


def train_step(model, ref_model, batch, beta=0.1):
    pi_chosen_logits = model(batch["chosen_input_ids"], attention_mask=batch["chosen_mask"]).logits
    pi_rejected_logits = model(batch["rejected_input_ids"], attention_mask=batch["rejected_mask"]).logits
    pi_chosen_logps = compute_logps(pi_chosen_logits, batch["chosen_labels"])
    pi_rejected_logps = compute_logps(pi_rejected_logits, batch["rejected_labels"])
    with torch.no_grad():
        ref_chosen_logps = compute_logps(
            ref_model(batch["chosen_input_ids"], attention_mask=batch["chosen_mask"]).logits,
            batch["chosen_labels"],
        )
        ref_rejected_logps = compute_logps(
            ref_model(batch["rejected_input_ids"], attention_mask=batch["rejected_mask"]).logits,
            batch["rejected_labels"],
        )
    return dpo_loss(pi_chosen_logps, pi_rejected_logps, ref_chosen_logps, ref_rejected_logps, beta)
```

**面试口述要点**：

1. **关键是 prompt token 不能算 logp**（用 -100 mask）
2. **next-token shift** —— logits[:, :-1] 预测 labels[:, 1:]
3. **ref model 必须 no_grad + eval mode**
4. **logits = β × (pi 差 - ref 差)** —— pi/ref 差 = implicit reward
5. **常 monitor margin（chosen vs rejected 差）和 win_rate** —— 健康 DPO margin 应 ↑

**易错点**：
- `labels[-100] = 0` 之前没 backup → mask 全错
- 忘记 sum over tokens 直接 mean → scale 不对
- ref logp 计算重复 forward → 慢；可缓存

**反诘 Q**：为什么用 sum 不用 mean？  
A: DPO 是 sequence-level preference，不是 token-level；sum 自然反映 sequence likelihood。但有人 argue length-normalize（SimPO 思路）防 length bias。

---

### Q3·白板手撕 GRPO loss

**题目**：实现 GRPO 的 group-relative advantage + GRPO loss。

**参考实现**：

```python
import torch
import torch.nn.functional as F

def compute_group_advantage(rewards: torch.Tensor) -> torch.Tensor:
    """
    rewards: (B, G) - B prompts, G samples each
    Returns: (B, G) - normalized advantages
    """
    mean = rewards.mean(dim=1, keepdim=True)
    std = rewards.std(dim=1, keepdim=True) + 1e-8
    return (rewards - mean) / std


def grpo_loss(
    logprobs_new: torch.Tensor,    # (B, G, T)
    logprobs_old: torch.Tensor,    # (B, G, T)
    logprobs_ref: torch.Tensor,    # (B, G, T)  no_grad
    advantages: torch.Tensor,      # (B, G)
    mask: torch.Tensor,            # (B, G, T)  response mask
    eps: float = 0.2,
    kl_coef: float = 0.04,
) -> dict:
    # 1. Sequence-level log ratio (sum log p over response tokens)
    seq_logp_new = (logprobs_new * mask).sum(dim=-1)   # (B, G)
    seq_logp_old = (logprobs_old * mask).sum(dim=-1)   # (B, G)
    seq_log_ratio = seq_logp_new - seq_logp_old
    seq_ratio = torch.exp(seq_log_ratio)

    # 2. Clipped surrogate
    surr1 = seq_ratio * advantages
    surr2 = torch.clamp(seq_ratio, 1 - eps, 1 + eps) * advantages
    pg_loss = -torch.min(surr1, surr2).mean()

    # 3. KL penalty (Schulman unbiased estimator, token-level)
    # KL ≈ ratio_ref - log_ratio_ref - 1, ratio_ref = exp(log p_ref - log p_new)
    log_ratio_ref = logprobs_ref - logprobs_new  # token-level
    ratio_ref = torch.exp(log_ratio_ref)
    kl_per_token = ratio_ref - log_ratio_ref - 1.0
    kl_per_token = kl_per_token * mask
    kl_loss = (kl_per_token.sum(dim=-1) / mask.sum(dim=-1).clamp(min=1)).mean()

    total = pg_loss + kl_coef * kl_loss

    with torch.no_grad():
        approx_kl = ((seq_ratio - 1) - seq_log_ratio).mean()
        clip_frac = ((seq_ratio - 1).abs() > eps).float().mean()
    return {
        "loss": total,
        "pg_loss": pg_loss,
        "kl_loss": kl_loss,
        "approx_kl": approx_kl,
        "clip_frac": clip_frac,
    }
```

**面试口述要点**：

1. **Advantage 是 prompt-conditional normalization**（同 prompt 的 G 个 reward 标准化）
2. **Ratio 是 sequence-level**（不是 token-level）——这是和 PPO 区别
3. **KL 是 token-level + 用 Schulman unbiased estimator** $(ratio - log ratio - 1)$
4. **不需要 critic / value model**
5. **G 必须保证 reward 有 variance**（否则 advantage 全 0）

**易错点**：
- Group advantage 算成跨 prompt → 错（必须 per-prompt）
- KL 用 naive $\log p_{ref} - \log p$ → 可能负 → 不稳
- 忘记 mask response token

---

## B 组·训练 / sampling loop

### Q4·写一个 minimal RLHF rollout + PPO training loop

**题目**：给定 actor / critic / RM / ref 4 个 model，实现 1 个 RLHF iteration。

**参考实现**：

```python
import torch
from torch.utils.data import DataLoader

@torch.no_grad()
def rollout(actor, critic, ref, rm, prompts, gen_kwargs):
    """
    Generate responses, compute rewards / values / log probs.
    Returns: dict for PPO training
    """
    actor.eval(); critic.eval(); ref.eval(); rm.eval()
    # 1. Generate
    out = actor.generate(prompts, **gen_kwargs)  # (B, T)
    response = out[:, prompts.shape[1]:]
    full = out  # prompt + response

    # 2. Compute log probs (actor + ref)
    logits = actor(full).logits
    logits_ref = ref(full).logits
    logp_old = compute_token_logp(logits, full)        # (B, T)
    logp_ref = compute_token_logp(logits_ref, full)    # (B, T)

    # 3. Compute values (critic)
    values = critic(full).values   # (B, T)

    # 4. Compute reward (RM on full seq, only at EOS)
    rm_scores = rm(full)           # (B,)  scalar
    rewards = torch.zeros_like(values)
    rewards[:, -1] = rm_scores

    # 5. KL penalty as auxiliary token reward
    kl_pen = -0.02 * (logp_old - logp_ref)  # negative KL ≈ reward
    rewards = rewards + kl_pen

    # 6. GAE
    advantages, returns = compute_gae(rewards, values, mask=..., gamma=1.0, lam=0.95)

    return {
        "input_ids": full,
        "logp_old": logp_old,
        "values_old": values,
        "advantages": advantages,
        "returns": returns,
        "mask": ...,
    }


def ppo_train(actor, critic, batch, optimizer, n_epochs=4, mini_batch=8):
    actor.train(); critic.train()
    for epoch in range(n_epochs):
        for mb in iter_minibatches(batch, mini_batch):
            logits = actor(mb["input_ids"]).logits
            logp_new = compute_token_logp(logits, mb["input_ids"])
            values_new = critic(mb["input_ids"]).values
            entropy = compute_entropy(logits)

            out = ppo_loss(
                logp_new, mb["logp_old"],
                mb["advantages"], mb["returns"],
                values_new, mb["values_old"],
                entropy, mb["mask"],
            )
            optimizer.zero_grad()
            out["loss"].backward()
            torch.nn.utils.clip_grad_norm_(actor.parameters(), 1.0)
            torch.nn.utils.clip_grad_norm_(critic.parameters(), 1.0)
            optimizer.step()


def train_iteration(actor, critic, ref, rm, prompts_loader, optimizer, gen_kwargs):
    for prompt_batch in prompts_loader:
        rollout_batch = rollout(actor, critic, ref, rm, prompt_batch, gen_kwargs)
        ppo_train(actor, critic, rollout_batch, optimizer)
```

**面试口述要点**：

1. **Rollout 在 no_grad 下**——节省 memory
2. **KL penalty 加到 token-level reward**（or loss-level，两种）——经典做法
3. **PPO 一次 rollout 跑 4 epochs**（vs vanilla policy gradient 一次）——sample efficient
4. **Mini-batch shuffle** within rollout batch
5. **Gradient clipping** —— max norm 1.0 是经典
6. **Generate 时用 temperature / top-p / nucleus** —— 不能 greedy（无 exploration）

**易错点**：
- 把 generation 放 train mode → dropout 影响 logp
- forgot detach old logp → 梯度污染
- KL 加 reward 时正负号写反

---

### Q5·写一个 RM 训练 loop

**题目**：给 (prompt, chosen, rejected) 数据，训一个 reward model。

**参考实现**：

```python
import torch
import torch.nn.functional as F
from torch.utils.data import DataLoader

class RewardModel(torch.nn.Module):
    def __init__(self, base_model):
        super().__init__()
        self.transformer = base_model
        hidden = base_model.config.hidden_size
        self.value_head = torch.nn.Linear(hidden, 1, bias=False)

    def forward(self, input_ids, attention_mask=None):
        outputs = self.transformer(input_ids, attention_mask=attention_mask, output_hidden_states=True)
        hidden = outputs.hidden_states[-1]   # (B, T, H)
        rewards = self.value_head(hidden).squeeze(-1)   # (B, T)
        # 用最后一个非 pad token 的 reward
        if attention_mask is not None:
            seq_len = attention_mask.sum(dim=1) - 1
            batch_idx = torch.arange(input_ids.size(0))
            return rewards[batch_idx, seq_len]   # (B,)
        return rewards[:, -1]


def rm_loss(reward_chosen: torch.Tensor, reward_rejected: torch.Tensor):
    """Bradley-Terry preference loss"""
    return -F.logsigmoid(reward_chosen - reward_rejected).mean()


def train_rm(rm, dataloader, optimizer, n_epochs=1):
    rm.train()
    for epoch in range(n_epochs):
        for batch in dataloader:
            r_w = rm(batch["chosen_input_ids"], batch["chosen_attention_mask"])
            r_l = rm(batch["rejected_input_ids"], batch["rejected_attention_mask"])
            loss = rm_loss(r_w, r_l)
            optimizer.zero_grad()
            loss.backward()
            torch.nn.utils.clip_grad_norm_(rm.parameters(), 1.0)
            optimizer.step()

            with torch.no_grad():
                acc = (r_w > r_l).float().mean()
            print(f"loss={loss.item():.4f} acc={acc.item():.4f}")
```

**口述要点**：

1. **RM = transformer + value head** —— value head 是 linear(hidden → 1)
2. **用最后一个非 pad token 的 reward** —— pooling 选择（也可 mean）
3. **Loss = `-log sigmoid(r_w - r_l)`** —— BT loss
4. **Acc = (r_w > r_l).mean()** —— hold-out 上看 baseline
5. **Watch out：reward 在 train 时无 bound，可能 explode** —— 加 KL constraint or LayerNorm

**反诘 Q**：RM 怎么 eval calibration？  
A: 维护 held-out preference set；除了 acc，画 calibration plot：bin 不同 reward gap，看实际 win rate 是否对应 BT 预测。

---

### Q6·实现一个 best-of-N sampler + PRM-based selection

**题目**：用 PRM 选 best-of-N 推理时增强 reasoning。

**参考实现**：

```python
import torch
from typing import Callable

@torch.no_grad()
def best_of_n_with_prm(
    policy,
    prm,
    prompt: str,
    n_samples: int = 8,
    temperature: float = 0.7,
    top_p: float = 0.95,
) -> tuple[str, float]:
    """
    Generate n_samples, use PRM to score each, return best.
    """
    samples = policy.generate(
        prompt,
        do_sample=True, temperature=temperature, top_p=top_p,
        num_return_sequences=n_samples,
        max_new_tokens=1024,
    )

    scores = []
    for s in samples:
        steps = split_into_steps(s)
        step_scores = [prm.score(prompt, prefix) for prefix in growing_prefixes(steps)]
        # Aggregation: min / mean / last / product
        # OpenAI Let's Verify 用 product（min of probs ≈ all steps right）
        aggregated = float(torch.tensor(step_scores).min())
        scores.append(aggregated)

    best_idx = int(torch.tensor(scores).argmax())
    return samples[best_idx], scores[best_idx]


def split_into_steps(response: str) -> list[str]:
    """简单按 '\n\n' 或 'Step N:' 切分"""
    return [s.strip() for s in response.split("\n\n") if s.strip()]


def growing_prefixes(steps: list[str]) -> list[str]:
    """[s1, s1+s2, s1+s2+s3, ...]"""
    return ["\n\n".join(steps[:i+1]) for i in range(len(steps))]
```

**口述要点**：

1. **N 个 sample 用同一 prompt** —— `num_return_sequences=N`
2. **PRM 在 step prefix 上 score** —— 累积分数
3. **Aggregation 选择**：min / mean / product / last——OpenAI Let's Verify 用 min（保守）
4. **Cost analysis**：N×generation cost + N×O(steps) PRM cost；N=8 经典
5. **如果 N 大**：用 tournament selection（每 round eliminate half）

**反诘 Q**：怎么 batch 加速？  
A: N=8 时 batch 整 8 个 generation 一起跑（generate 支持 num_return）；PRM scoring 也可 batch（所有 prefix 一起送）。Token cost 是主要瓶颈。

---

## C 组（拓展）·LeetCode 类（轻量考察）

研究员岗 LeetCode 题相对少，主要看 Python proficiency。常见：

| 题 | 难度 | 考察点 |
| --- | --- | --- |
| 实现 softmax + log_softmax (numerical stable) | E | numerical |
| 实现 Top-K sampling | E | argpartition + 概率 |
| 实现 Nucleus (top-p) sampling | M | cumsum + mask |
| 实现一个 attention block（Q/K/V，masked） | M | matrix ops |
| 实现 layer norm | E | mean/std/bias |
| 实现一个 streaming softmax（看过 FlashAttention 加分） | H | online |

**示例·numerical stable softmax**：

```python
def softmax(x: torch.Tensor, dim: int = -1) -> torch.Tensor:
    x_max = x.max(dim=dim, keepdim=True).values
    x_shifted = x - x_max
    exp_x = torch.exp(x_shifted)
    return exp_x / exp_x.sum(dim=dim, keepdim=True)

def log_softmax(x: torch.Tensor, dim: int = -1) -> torch.Tensor:
    x_max = x.max(dim=dim, keepdim=True).values
    x_shifted = x - x_max
    return x_shifted - torch.log(torch.exp(x_shifted).sum(dim=dim, keepdim=True))
```

**示例·Top-p sampling**：

```python
def top_p_sample(logits: torch.Tensor, p: float = 0.95) -> int:
    probs = F.softmax(logits, dim=-1)
    sorted_probs, sorted_idx = torch.sort(probs, descending=True)
    cumsum = sorted_probs.cumsum(dim=-1)
    # Keep tokens up to first one that crosses p
    sorted_keep = cumsum <= p
    sorted_keep[..., 0] = True  # always keep top-1
    sorted_probs_kept = sorted_probs * sorted_keep
    sorted_probs_kept = sorted_probs_kept / sorted_probs_kept.sum(dim=-1, keepdim=True)
    sampled_sorted_idx = torch.multinomial(sorted_probs_kept, 1)
    return sorted_idx.gather(-1, sampled_sorted_idx).item()
```

---

## 答题"通用 framework"（coding 题）

```
1. CLARIFY (1 分钟)
   - "输入 shape 我 confirm 下 (B, T, V) 对吧"
   - "我假设 PyTorch；不用纯 numpy 行吗"
   - "edge case：T=0 / batch=1 我怎么处理"

2. SKETCH (2 分钟)
   - 在 comment 里写 5 步 plan
   - 标关键 trick (mask / detach / log_softmax)

3. IMPLEMENT (15 分钟)
   - 关键 line 边写边讲为什么
   - 写完简单 sanity test：构造 toy input verify

4. DEBUG / OPTIMIZE (5 分钟)
   - "可优化点：vectorize / batch / cache"
   - "production 我会加 mixed precision / DDP"

5. EXTEND (5 分钟)
   - 如果面试官追问"再加 entropy bonus" / "改成 GRPO"
   - 表现 flexibility
```

---

## 最后忠告

> **研究员 coding 不靠 brute force memorize——靠 "我自己实现过 PPO/DPO/GRPO 跑过 toy task"。这种亲自跑过的 muscle memory 永远不会被 "假装看过" 的人模仿出来。**

> **强烈建议 T-14 ~ T-7 完成至少 1 个 mini RLHF 项目**——用 TRL 或自己写。这是这门面试的"硬通货"。
