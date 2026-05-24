# 07｜高频面试题 — PyTorch + CUDA 手撕代码篇

> 这一篇是 **白板 + 现场编码** 题。研究员面试常考 5~6 道，约 60~90 分钟。
>
> 主轴：
> 1. 手撕 attention（必考）
> 2. 手撕 RL loss（DPO / PPO / GRPO）
> 3. PyTorch 工程题（distributed / mixed precision / 自定义 op）
> 4. CUDA / Triton kernel（加分项）
> 5. 数值稳定性 + log-sum-exp 等技巧
>
> 每题给：**题面 / 满分代码 / 讲解 / 易踩 / 追问扩展**。

---

## Q1. 手写 Scaled Dot-Product Attention（必考）

### 题面

实现 multi-head attention forward，包括 mask + 因果（causal）支持。

### 满分代码

```python
import torch
import torch.nn as nn
import torch.nn.functional as F
import math

class MultiHeadAttention(nn.Module):
    def __init__(self, d_model: int, n_heads: int, dropout: float = 0.0):
        super().__init__()
        assert d_model % n_heads == 0
        self.d_model = d_model
        self.n_heads = n_heads
        self.d_head = d_model // n_heads
        
        self.W_q = nn.Linear(d_model, d_model, bias=False)
        self.W_k = nn.Linear(d_model, d_model, bias=False)
        self.W_v = nn.Linear(d_model, d_model, bias=False)
        self.W_o = nn.Linear(d_model, d_model, bias=False)
        self.dropout = nn.Dropout(dropout)
    
    def forward(self, x, mask=None, causal=False):
        # x: [B, T, D]
        B, T, D = x.shape
        H, Dh = self.n_heads, self.d_head
        
        Q = self.W_q(x).view(B, T, H, Dh).transpose(1, 2)  # [B, H, T, Dh]
        K = self.W_k(x).view(B, T, H, Dh).transpose(1, 2)
        V = self.W_v(x).view(B, T, H, Dh).transpose(1, 2)
        
        scores = (Q @ K.transpose(-2, -1)) / math.sqrt(Dh)  # [B, H, T, T]
        
        if causal:
            mask_c = torch.triu(torch.ones(T, T, device=x.device), diagonal=1).bool()
            scores = scores.masked_fill(mask_c, float('-inf'))
        
        if mask is not None:
            # mask: [B, T] 1=keep, 0=pad
            mask = mask[:, None, None, :]  # [B, 1, 1, T]
            scores = scores.masked_fill(~mask.bool(), float('-inf'))
        
        attn = F.softmax(scores, dim=-1)
        attn = self.dropout(attn)
        
        out = attn @ V  # [B, H, T, Dh]
        out = out.transpose(1, 2).contiguous().view(B, T, D)
        return self.W_o(out)
```

### 讲解

- 形状变换：`[B,T,D] → [B,T,H,Dh] → [B,H,T,Dh]`（reshape + transpose，不能直接 view）
- `1/sqrt(d_head)` 缩放：防止 logits 过大 → softmax 饱和
- Causal mask 用上三角；padding mask 来自外部
- `-inf` 填充：softmax 后变成 0

### 易踩

- 忘 `transpose` → shape 错
- causal 用 `diagonal=0` → 错位（自己看不见自己）
- mask 形状没 broadcast 对齐
- `view` 之前没 `contiguous` → 报错

### 追问

**Q：怎么用 Flash Attention？**
A：`torch.nn.functional.scaled_dot_product_attention(Q, K, V, is_causal=True)`——PyTorch 2.x 内置；底层会自动选 Flash / memory-efficient / math kernel。

**Q：MQA / GQA 怎么改？**
A：K, V 的 head 数变少（MQA = 1, GQA = G < H），forward 时把 K/V 在 head 维 repeat。

**Q：MLA 怎么改？**
A：W_q 正常；K/V 经低秩投影到 latent c → 推理时再升回。RoPE 部分需 decoupled。

---

## Q2. 手写 DPO Loss（必考）

### 题面

实现 DPO loss，输入是 chosen / rejected 的 logp 和 reference logp。

### 满分代码

```python
import torch
import torch.nn.functional as F

def dpo_loss(
    policy_logp_chosen: torch.Tensor,   # [B]
    policy_logp_rejected: torch.Tensor, # [B]
    ref_logp_chosen: torch.Tensor,      # [B]
    ref_logp_rejected: torch.Tensor,    # [B]
    beta: float = 0.1,
) -> tuple[torch.Tensor, dict]:
    """DPO loss with diagnostics."""
    pi_logratio_chosen = policy_logp_chosen - ref_logp_chosen
    pi_logratio_rejected = policy_logp_rejected - ref_logp_rejected
    
    logits = beta * (pi_logratio_chosen - pi_logratio_rejected)
    loss = -F.logsigmoid(logits).mean()
    
    with torch.no_grad():
        reward_chosen = beta * pi_logratio_chosen
        reward_rejected = beta * pi_logratio_rejected
        reward_acc = (reward_chosen > reward_rejected).float().mean()
        reward_margin = (reward_chosen - reward_rejected).mean()
    
    metrics = {
        "loss": loss.item(),
        "reward_chosen": reward_chosen.mean().item(),
        "reward_rejected": reward_rejected.mean().item(),
        "reward_acc": reward_acc.item(),
        "reward_margin": reward_margin.item(),
    }
    return loss, metrics


@torch.no_grad()
def compute_logp(
    model,
    input_ids: torch.Tensor,  # [B, T]
    response_mask: torch.Tensor,  # [B, T] 1=response token, 0=prompt/pad
) -> torch.Tensor:
    """Sum log-prob of response tokens."""
    logits = model(input_ids).logits  # [B, T, V]
    # shift: predict token t+1 from t
    log_probs = F.log_softmax(logits[:, :-1], dim=-1)
    target = input_ids[:, 1:].unsqueeze(-1)
    token_logp = torch.gather(log_probs, dim=-1, index=target).squeeze(-1)
    mask = response_mask[:, 1:]
    return (token_logp * mask).sum(dim=-1)  # [B]
```

### 讲解

- `logsigmoid` 比 `log(sigmoid)` 数值稳定
- 不需要算 partition function（DPO 推导时已消）
- 计算 `reward_acc` 监控：判断 policy 学得对——chosen 是否比 rejected 更被偏好
- `compute_logp` 注意 shift（自回归 LM 是 next-token prediction）
- 要在 response token 上 sum，不算 prompt + padding

### 易踩

- 忘 shift → 等于在算 prompt 的 logp
- mask 弄错 → padding 也被算
- ref model 没 `eval()` 或没 `no_grad` → OOM
- 没 detach ref logp → 多余梯度
- `F.logsigmoid(-x)` 等价 `-F.softplus(x)`——更稳定

### 追问

**Q：为什么不用 `log(sigmoid(x))`？**
A：sigmoid(x) 在 x 大时趋近 1，log(1) = 0 但因浮点丢精度可能算成 -inf。`logsigmoid` 内部用 `-softplus(-x)` 避免。

**Q：怎么算 KL[π||π_ref]？**
A：精确 KL = E[π * log(π/π_ref)]，需要全 vocab → 贵；常用近似：`KL ≈ logp_policy - logp_ref`（per sample）。

---

## Q3. 手写 PPO Loss（clipped surrogate）

### 满分代码

```python
def ppo_loss(
    new_logp: torch.Tensor,   # [B, T]
    old_logp: torch.Tensor,   # [B, T]  detached
    advantages: torch.Tensor, # [B, T]
    clip_eps: float = 0.2,
    mask: torch.Tensor = None,  # [B, T]
) -> tuple[torch.Tensor, dict]:
    ratio = (new_logp - old_logp).exp()
    
    surr1 = ratio * advantages
    surr2 = ratio.clamp(1 - clip_eps, 1 + clip_eps) * advantages
    policy_loss = -torch.min(surr1, surr2)
    
    if mask is not None:
        policy_loss = (policy_loss * mask).sum() / mask.sum().clamp(min=1)
    else:
        policy_loss = policy_loss.mean()
    
    with torch.no_grad():
        clip_frac = ((ratio - 1.0).abs() > clip_eps).float().mean()
        approx_kl = ((ratio - 1) - (new_logp - old_logp)).mean()  # k3 estimator
    
    return policy_loss, {
        "policy_loss": policy_loss.item(),
        "clip_frac": clip_frac.item(),
        "approx_kl": approx_kl.item(),
        "ratio_mean": ratio.mean().item(),
    }
```

### 讲解

- `clip_frac` 监控：被 clip 的比例；如果一直 0 → step 太小没必要；如果 > 50% → 学得过激
- `approx_kl` (k3 estimator)：`(r-1) - log(r)`，无偏且非负；老版本用 `mean(logp_new - logp_old)` 是有偏的
- 要 mask 掉 padding 和 prompt token

### 追问

**Q：完整 PPO loss 还有什么？**
A：`L = L_policy + c1 * L_value - c2 * L_entropy`。Value loss = `(V_pred - V_target)^2`；entropy 鼓励探索。

---

## Q4. 手写 GRPO Advantage 计算

### 满分代码

```python
def grpo_advantage(rewards: torch.Tensor) -> torch.Tensor:
    """
    rewards: [G] - G samples for one prompt
    returns: [G] - group-normalized advantage
    """
    mean = rewards.mean()
    std = rewards.std(unbiased=False)
    return (rewards - mean) / (std + 1e-8)


def grpo_loss(
    new_logp: torch.Tensor,    # [B*G, T]
    old_logp: torch.Tensor,    # [B*G, T]
    ref_logp: torch.Tensor,    # [B*G, T]
    rewards: torch.Tensor,     # [B*G] one scalar per sample
    group_size: int,
    response_mask: torch.Tensor, # [B*G, T]
    beta: float = 0.04,
    clip_eps: float = 0.2,
) -> tuple[torch.Tensor, dict]:
    """GRPO loss in DeepSeek-R1 style."""
    BG = rewards.shape[0]
    B = BG // group_size
    rewards_grouped = rewards.view(B, group_size)  # [B, G]
    
    mean = rewards_grouped.mean(dim=1, keepdim=True)
    std = rewards_grouped.std(dim=1, keepdim=True, unbiased=False)
    advantages_per_sample = ((rewards_grouped - mean) / (std + 1e-8)).view(BG)  # [B*G]
    
    advantages = advantages_per_sample.unsqueeze(1).expand(BG, new_logp.shape[1])  # [B*G, T]
    
    ratio = (new_logp - old_logp).exp()
    surr1 = ratio * advantages
    surr2 = ratio.clamp(1 - clip_eps, 1 + clip_eps) * advantages
    policy_loss = -torch.min(surr1, surr2)
    
    log_r = ref_logp - new_logp
    per_token_kl = log_r.exp() - log_r - 1
    
    loss = policy_loss + beta * per_token_kl
    loss = (loss * response_mask).sum() / response_mask.sum().clamp(min=1)
    
    return loss, {
        "loss": loss.item(),
        "policy_loss": (policy_loss * response_mask).mean().item(),
        "kl": (per_token_kl * response_mask).mean().item(),
        "reward_mean": rewards.mean().item(),
        "reward_std": rewards.std().item(),
    }
```

### 讲解

- Group normalization 是 GRPO 灵魂
- KL 用 unbiased k3 estimator (`exp(log_r) - log_r - 1`)——保证非负
- Per-token KL 加入 loss（DeepSeek-R1 推荐做法），不像 InstructGPT 加在 reward 里
- Mask 处理 response token 以外的位置

### 追问

**Q：group_size 怎么选？**
A：典型 8~64。小 → 信号噪；大 → rollout 贵。R1 默认 ~64。

**Q：如果 std=0 怎么办？**
A：所有 sample 同 reward → advantage = 0，没信号——加 1e-8 防除零；并监控 reward variance 提示数据/任务设计问题。

---

## Q5. 手写 Log-Sum-Exp（数值稳定 softmax）

### 题面

实现 softmax 的数值稳定版本。

### 满分代码

```python
def stable_softmax(x: torch.Tensor, dim: int = -1) -> torch.Tensor:
    x_max = x.max(dim=dim, keepdim=True).values
    x_shifted = x - x_max
    exp_x = x_shifted.exp()
    return exp_x / exp_x.sum(dim=dim, keepdim=True)


def stable_log_softmax(x: torch.Tensor, dim: int = -1) -> torch.Tensor:
    x_max = x.max(dim=dim, keepdim=True).values
    x_shifted = x - x_max
    log_sum_exp = x_shifted.exp().sum(dim=dim, keepdim=True).log()
    return x_shifted - log_sum_exp
```

### 讲解

- 不减 max → `exp(1000)` 溢出
- `log_sum_exp` 公式：`log Σ exp(x_i) = m + log Σ exp(x_i - m)`，其中 `m = max(x)`
- PyTorch 内置 `F.softmax` / `F.log_softmax` 已自动做这事

### 追问

- "如果 batch size 大，dim=1 求 max 怎么 efficient？" → CUDA reduction（torch 内部实现）
- "为什么 fused log_softmax 而不是 log(softmax)？" → 两次 exp 浪费 + 精度损失

---

## Q6. 手写一个简易 Triton Kernel（加分）

### 题面

用 Triton 写一个 elementwise add（kernel 入门）。

### 满分代码

```python
import torch
import triton
import triton.language as tl


@triton.jit
def add_kernel(
    x_ptr,
    y_ptr,
    out_ptr,
    n_elements,
    BLOCK_SIZE: tl.constexpr,
):
    pid = tl.program_id(axis=0)
    block_start = pid * BLOCK_SIZE
    offsets = block_start + tl.arange(0, BLOCK_SIZE)
    mask = offsets < n_elements
    
    x = tl.load(x_ptr + offsets, mask=mask)
    y = tl.load(y_ptr + offsets, mask=mask)
    out = x + y
    tl.store(out_ptr + offsets, out, mask=mask)


def triton_add(x: torch.Tensor, y: torch.Tensor) -> torch.Tensor:
    out = torch.empty_like(x)
    n = x.numel()
    BLOCK_SIZE = 1024
    grid = (triton.cdiv(n, BLOCK_SIZE),)
    add_kernel[grid](x, y, out, n, BLOCK_SIZE=BLOCK_SIZE)
    return out
```

### 讲解

- `tl.program_id` ≈ CUDA threadIdx + blockIdx 合体
- `BLOCK_SIZE` 是 compile-time constant
- mask 处理 last block 越界
- grid 是一维（这题 1D），实际复杂 kernel 可以多维

### 追问

**Q：Triton vs CUDA 区别？**
A：Triton 自动处理 block-level memory + 自动 schedule；CUDA 需手动管 shared memory + warp 同步。Triton 写起来快 10x，性能差不多。

**Q：写 attention kernel 难点？**
A：1) Tiling Q/K/V 分块 2) Online softmax (Flash Attention) 3) Causal mask 处理 4) backward 重算

---

## Q7. 实现 Distributed Data Parallel（DDP）的核心思想

### 满分（讲 + 伪代码）

```python
# DDP 核心步骤
# 1. 每个 rank 复制完整 model
# 2. 每个 rank 独立 forward + backward
# 3. backward 完成后 all-reduce gradients
# 4. 每个 rank 用同步后的 grad 各自 step

import torch.distributed as dist

# 初始化
dist.init_process_group(backend='nccl')
local_rank = dist.get_rank()
torch.cuda.set_device(local_rank)

model = MyModel().cuda()
model = torch.nn.parallel.DistributedDataParallel(
    model, device_ids=[local_rank]
)
# DDP wrapper 自动:
# - forward 时各 rank 独立
# - backward 时自动 hook，每个 grad ready 就开始 all-reduce
# - all-reduce 用 ring algorithm

# Data loader
sampler = DistributedSampler(dataset, num_replicas=world_size, rank=local_rank)
loader = DataLoader(dataset, sampler=sampler, batch_size=...)

# Optimizer 各自维护，但 grad 同步所以参数同步
optimizer = AdamW(model.parameters())

for batch in loader:
    output = model(batch)  # 自动 all-reduce
    loss = compute_loss(output)
    loss.backward()
    optimizer.step()
    optimizer.zero_grad()
```

### 追问

**Q：DDP vs FSDP vs DeepSpeed ZeRO？**
- **DDP**：完整复制 model + grad sync。简单，model 必须 fit single GPU。
- **FSDP** (Fully Sharded DP)：参数 / 梯度 / 优化器状态都分片，forward/backward 时按需 all-gather。支持大模型。
- **ZeRO-1/2/3**：DeepSpeed 的渐进分片（优化器 → 梯度 → 参数），ZeRO-3 ≈ FSDP。

**Q：Gradient accumulation 怎么用？**
A：累积多 mini-batch 的 grad 再 step，等效大 batch。注意要 scale loss by accumulation steps。

---

## Q8. 实现 Mixed Precision Training (AMP)

### 满分代码

```python
from torch.cuda.amp import autocast, GradScaler

scaler = GradScaler()
optimizer = AdamW(model.parameters())

for batch in loader:
    optimizer.zero_grad()
    with autocast(dtype=torch.bfloat16):  # 或 float16
        output = model(batch)
        loss = compute_loss(output)
    
    if scaler is not None:  # FP16 用 scaler
        scaler.scale(loss).backward()
        scaler.unscale_(optimizer)
        torch.nn.utils.clip_grad_norm_(model.parameters(), 1.0)
        scaler.step(optimizer)
        scaler.update()
    else:  # BF16 不需要 scaler
        loss.backward()
        torch.nn.utils.clip_grad_norm_(model.parameters(), 1.0)
        optimizer.step()
```

### 讲解

- **FP16**：动态范围窄 → 容易 underflow → 需要 loss scaler
- **BF16**：动态范围 = FP32，但精度低 → 不需要 scaler
- **FP8**：更激进，需要 per-tensor scaling factor，目前主要靠 Transformer Engine
- AMP 自动把白名单 op（matmul, conv）跑 FP16/BF16；黑名单 op（softmax, loss）保持 FP32

### 追问

**Q：BF16 训练注意什么？**
A：1) Adam 状态用 FP32 2) Loss 用 FP32 reduce 3) gradient accumulation 用 FP32 buffer

---

## Q9. 手写 LayerNorm 和 RMSNorm

### 满分代码

```python
class LayerNorm(nn.Module):
    def __init__(self, d, eps=1e-5):
        super().__init__()
        self.weight = nn.Parameter(torch.ones(d))
        self.bias = nn.Parameter(torch.zeros(d))
        self.eps = eps
    
    def forward(self, x):
        # x: [..., d]
        mean = x.mean(dim=-1, keepdim=True)
        var = x.var(dim=-1, keepdim=True, unbiased=False)
        x_normed = (x - mean) / (var + self.eps).sqrt()
        return self.weight * x_normed + self.bias


class RMSNorm(nn.Module):
    def __init__(self, d, eps=1e-6):
        super().__init__()
        self.weight = nn.Parameter(torch.ones(d))
        self.eps = eps
    
    def forward(self, x):
        rms = x.pow(2).mean(dim=-1, keepdim=True).sqrt()
        x_normed = x / (rms + self.eps)
        return self.weight * x_normed
```

### 讲解

- LayerNorm = 减 mean + 除 std + 仿射变换
- RMSNorm = 不减 mean，只除 RMS——计算少一半，效果接近，LLaMA / DeepSeek 都用
- `var(unbiased=False)` = 用 N 而不是 N-1，PyTorch 默认 True 是统计学的，但 nn.LayerNorm 用 False

### 追问

**Q：为什么 Transformer 用 LayerNorm 不用 BatchNorm？**
A：1) batch 间样本 length 不同 2) inference 时 batch 可能为 1 3) seq 维度统计更有意义

**Q：Pre-norm vs Post-norm？**
A：Pre-norm（norm 在 residual 之前）训练更稳，是现代 Transformer 主流。

---

## Q10. 实现 Top-K + Nucleus (Top-P) 采样

### 满分代码

```python
def sample(logits: torch.Tensor, temperature: float = 1.0, top_k: int = 0, top_p: float = 0.0) -> torch.Tensor:
    """
    logits: [B, V]
    returns: [B] sampled token ids
    """
    if temperature == 0:
        return logits.argmax(dim=-1)
    
    logits = logits / temperature
    
    if top_k > 0:
        top_k_vals, _ = logits.topk(top_k, dim=-1)
        kth = top_k_vals[..., -1:].expand_as(logits)
        logits = torch.where(logits < kth, torch.full_like(logits, float('-inf')), logits)
    
    if 0 < top_p < 1.0:
        sorted_logits, sorted_idx = logits.sort(dim=-1, descending=True)
        probs = sorted_logits.softmax(dim=-1)
        cum_probs = probs.cumsum(dim=-1)
        
        sorted_mask = cum_probs > top_p
        sorted_mask[..., 1:] = sorted_mask[..., :-1].clone()
        sorted_mask[..., 0] = False
        
        mask = torch.zeros_like(logits, dtype=torch.bool).scatter(
            dim=-1, index=sorted_idx, src=sorted_mask
        )
        logits = logits.masked_fill(mask, float('-inf'))
    
    probs = logits.softmax(dim=-1)
    return torch.multinomial(probs, num_samples=1).squeeze(-1)
```

### 讲解

- Temperature: 越低越 deterministic（极限 = argmax）
- Top-k: 只保留 top-k 个 logit，其他 -inf
- Top-p (nucleus): 累积概率 > p 的最小集合
- 注意 top-p mask 要 shift 1 位——保证至少有 1 个 token

### 追问

**Q：Min-p sampling？**
A：保留所有 prob ≥ p × max_prob 的 token，比 top-p 更鲁棒。

**Q：Speculative decoding？**
A：用小 draft model 一次生成 N token，大 model verify，accept/reject。加速 inference 2~3x。

---

## Q11. 实现简易 Adam Optimizer

### 满分代码

```python
class AdamW:
    def __init__(self, params, lr=1e-3, betas=(0.9, 0.999), eps=1e-8, weight_decay=0.01):
        self.params = list(params)
        self.lr = lr
        self.b1, self.b2 = betas
        self.eps = eps
        self.wd = weight_decay
        self.state = {id(p): {'step': 0, 'm': torch.zeros_like(p), 'v': torch.zeros_like(p)} 
                      for p in self.params}
    
    def step(self):
        for p in self.params:
            if p.grad is None:
                continue
            g = p.grad
            st = self.state[id(p)]
            st['step'] += 1
            
            st['m'].mul_(self.b1).add_(g, alpha=1 - self.b1)
            st['v'].mul_(self.b2).addcmul_(g, g, value=1 - self.b2)
            
            m_hat = st['m'] / (1 - self.b1 ** st['step'])
            v_hat = st['v'] / (1 - self.b2 ** st['step'])
            
            p.data.addcdiv_(m_hat, v_hat.sqrt() + self.eps, value=-self.lr)
            
            if self.wd > 0:
                p.data.add_(p.data, alpha=-self.lr * self.wd)
    
    def zero_grad(self):
        for p in self.params:
            if p.grad is not None:
                p.grad.detach_().zero_()
```

### 讲解

- m: 一阶动量（gradient EMA）
- v: 二阶动量（gradient² EMA）
- bias correction：`/(1 - β^t)` 修正初始化偏差
- AdamW: weight decay 直接作用于参数（不是加在 grad）
- `addcdiv_` / `addcmul_` 是 in-place fused op，省内存

### 追问

**Q：为什么 AdamW > Adam？**
A：原 Adam 的 weight decay 实际作用在 normalized grad 上，效果不对；AdamW 直接 decay 参数。

**Q：Lion / Sophia 等新 optimizer 有什么进步？**
A：Lion（Google 2023）只用 sign(grad)，省内存；Sophia 用 Hessian diagonal 估计加速。

---

## Q12. 实现 KV Cache（推理优化）

### 满分代码

```python
class CachedAttention(nn.Module):
    """Attention with KV cache for incremental decoding."""
    
    def __init__(self, d_model, n_heads):
        super().__init__()
        self.n_heads = n_heads
        self.d_head = d_model // n_heads
        self.W_q = nn.Linear(d_model, d_model)
        self.W_k = nn.Linear(d_model, d_model)
        self.W_v = nn.Linear(d_model, d_model)
        self.W_o = nn.Linear(d_model, d_model)
    
    def forward(self, x, cache=None):
        """
        x: [B, T_new, D] - new tokens only (1 token for incremental)
        cache: dict with 'K' [B, H, T_past, Dh] and 'V' same shape, or None
        Returns: (output, new_cache)
        """
        B, T, D = x.shape
        H, Dh = self.n_heads, self.d_head
        
        Q = self.W_q(x).view(B, T, H, Dh).transpose(1, 2)
        K_new = self.W_k(x).view(B, T, H, Dh).transpose(1, 2)
        V_new = self.W_v(x).view(B, T, H, Dh).transpose(1, 2)
        
        if cache is not None:
            K = torch.cat([cache['K'], K_new], dim=2)
            V = torch.cat([cache['V'], V_new], dim=2)
        else:
            K = K_new
            V = V_new
        
        new_cache = {'K': K, 'V': V}
        
        scores = (Q @ K.transpose(-2, -1)) / math.sqrt(Dh)
        attn = scores.softmax(dim=-1)
        out = attn @ V
        out = out.transpose(1, 2).contiguous().view(B, T, D)
        return self.W_o(out), new_cache
```

### 讲解

- KV cache：把过去 K/V 存起来，每步只 compute 新 token 的 Q + KV
- 复杂度从 O(T²) → O(T)（per step）
- 显存：O(L × H × T × Dh × 2)——LLaMA-70B 长 context 时 cache 比 weight 还大
- 优化方向：MLA（低秩压缩）、GQA（共享 head）、PagedAttention（分页存储）

### 追问

**Q：MLA 怎么省 cache？**
A：把 K/V 投影到 latent c（远小于 d_model），cache 只存 c；推理时再升回 K/V。R1/V3 用这个。

**Q：PagedAttention？**
A：vLLM 的思想——把 KV cache 分成 page（block）非连续存储，类似 OS 虚拟内存。提高内存利用率 + 支持 multi-query batching。

---

## Q13. 实现 RoPE (Rotary Position Embedding)

### 满分代码

```python
class RoPE(nn.Module):
    def __init__(self, dim, max_len=8192, base=10000):
        super().__init__()
        inv_freq = 1.0 / (base ** (torch.arange(0, dim, 2).float() / dim))
        self.register_buffer('inv_freq', inv_freq)
        self.max_len = max_len
        self._build_cache(max_len)
    
    def _build_cache(self, seq_len):
        t = torch.arange(seq_len, dtype=self.inv_freq.dtype, device=self.inv_freq.device)
        freqs = torch.outer(t, self.inv_freq)  # [T, dim/2]
        cos = freqs.cos()
        sin = freqs.sin()
        self.register_buffer('cos_cache', cos, persistent=False)
        self.register_buffer('sin_cache', sin, persistent=False)
    
    def forward(self, x, position_ids=None):
        # x: [..., T, dim]
        T = x.shape[-2]
        if position_ids is None:
            cos = self.cos_cache[:T]
            sin = self.sin_cache[:T]
        else:
            cos = self.cos_cache[position_ids]
            sin = self.sin_cache[position_ids]
        
        x1, x2 = x[..., 0::2], x[..., 1::2]
        rotated = torch.stack([x1 * cos - x2 * sin, x1 * sin + x2 * cos], dim=-1)
        return rotated.flatten(start_dim=-2)
```

### 讲解

- RoPE 不加在 input 上，而是在 attention 内对 Q, K 旋转
- 旋转角度 = position × inv_freq
- 性质：`<R_m·q, R_n·k> = <q, k> · cos(m-n) - <q, R_perp k> · sin(m-n)`——相对位置编码
- 优点：长 context 外推友好（YaRN/NTK 等技巧可 extend）

### 追问

**Q：YaRN 是什么？**
A：长 context extension 技巧——降低低频部分 RoPE 角度增长率，让训过短 context 的 model 处理长 context。

---

## Q14. （加分）用 PyTorch hook 实现 gradient 监控

### 满分代码

```python
class GradMonitor:
    def __init__(self, model):
        self.stats = {}
        self.hooks = []
        for name, p in model.named_parameters():
            if p.requires_grad:
                hook = p.register_hook(self._make_hook(name))
                self.hooks.append(hook)
    
    def _make_hook(self, name):
        def hook(grad):
            self.stats[name] = {
                'norm': grad.norm().item(),
                'max': grad.abs().max().item(),
                'mean': grad.mean().item(),
                'std': grad.std().item(),
                'has_nan': torch.isnan(grad).any().item(),
            }
            return None  # 不修改 grad
        return hook
    
    def report(self):
        for name, s in sorted(self.stats.items(), key=lambda kv: -kv[1]['norm']):
            print(f"{name}: norm={s['norm']:.3e}, max={s['max']:.3e}, nan={s['has_nan']}")
    
    def remove(self):
        for h in self.hooks:
            h.remove()


# 使用
monitor = GradMonitor(model)
loss.backward()
monitor.report()
```

### 用途

- 找梯度爆炸位置
- 找梯度消失层
- 找 NaN 起源
- 算 grad norm distribution

---

## 通用刷题建议

| 题型 | 优先级 | 建议 |
| --- | --- | --- |
| Attention 手写 | P0 | 闭眼能写 |
| DPO / PPO / GRPO loss | P0 | 三个都能默写 |
| Softmax / log-sum-exp | P0 | 数值稳定性烂熟 |
| Sampling（top-k/p, temperature） | P1 | 推理基本功 |
| LayerNorm / RMSNorm / RoPE | P1 | 常考组件 |
| DDP / FSDP / AMP | P1 | 至少能讲 + 写伪代码 |
| KV Cache | P1 | 推理优化 |
| Triton kernel | P2 | 加分 |
| Optimizer | P2 | Adam 至少能写 |
| Hook / Debug | P2 | 工程能力体现 |

---

下一文档：[08-高频面试题-Paper深读与复现](08-高频面试题-Paper深读与复现.md)
