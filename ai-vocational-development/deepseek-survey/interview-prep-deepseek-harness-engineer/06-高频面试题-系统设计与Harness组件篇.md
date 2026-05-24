# 06 · 高频面试题 · 系统设计与 Harness 组件篇

> 这一篇是给**系统设计轮**准备的。题目大概率围绕 Harness 七大组件展开。
>
> **系统设计的得分公式**：
> - 30% **结构清晰**（分层 + 数据流图）
> - 25% **trade-off 显式**（每个决定列优劣）
> - 20% **scale 思考**（10× / 100× 怎么撑）
> - 15% **可观测 + 安全**
> - 10% **驱动对话**（让面试官引导你聚焦）

---

## 系统设计 4 阶段答题模板（每题都用）

```
Stage 1·Clarify (5 min)
  - 复述问题
  - 确认 scale（QPS / 用户量 / 模型规模）
  - 确认 SLO（latency / availability / cost）
  - 确认 out-of-scope（什么不做）

Stage 2·High-level Design (10 min)
  - 画系统组件图
  - 标数据流方向
  - 标关键 API

Stage 3·Deep Dive (15 min)
  - 选 2~3 个最关键组件深聊
  - 给数据结构 / DB schema / API spec
  - Trade-off 显式

Stage 4·Scale + Failure + Observability (10 min)
  - 10× 怎么扩
  - 单点故障怎么处理
  - 怎么监控
  - 怎么 rollout（灰度 / canary）
```

---

## 题 1：设计一个 Agent Harness 系统（45-60 分钟开放题）

**题面**：从 0 设计 DeepSeek 桌面端 Code Agent Harness。

### Stage 1·Clarify

主动问：
- 桌面 vs 服务端？→ 假设桌面（本地运行 + 远程模型调用）
- 单用户 vs 多租户？→ 假设单用户
- Target 模型？→ 假设 DeepSeek + 跨模型支持
- Day 1 用户量？→ 假设内部 200 → 外部 5000
- 平台？→ 跨 macOS / Linux / Windows

### Stage 2·High-level Design

```
┌──────────────────────────────────────────────────────────┐
│                    DeepSeek Code Harness                 │
│                                                          │
│  ┌──────────────┐    ┌──────────────┐                   │
│  │ UI Layer     │    │ CLI Layer    │  ← entry points    │
│  │ (TUI / GUI)  │    │              │                   │
│  └──────┬───────┘    └──────┬───────┘                   │
│         │                   │                            │
│         └─────────┬─────────┘                            │
│                   ▼                                      │
│  ┌──────────────────────────────────────────────────┐    │
│  │              Query Engine（核心）                 │    │
│  │  - Agent Loop with stream/abort                  │    │
│  │  - Budget / max_turns / diminishing detector     │    │
│  │  - Multi-model adapter (DS / OAI / Anthropic)    │    │
│  └──────┬───────────────────────────────────────────┘    │
│         │                                                │
│  ┌──────┴────────┬───────────────┬────────────────┐      │
│  ▼               ▼               ▼                ▼      │
│ Context        Tool System    Permission     Lifecycle   │
│ Engineer       (40+ tools     Engine         Hooks       │
│ (build/        + MCP client)  (Policy +      (8 events)  │
│  compress)                    Sandbox)                   │
│         │                          │                     │
│         ▼                          ▼                     │
│  ┌──────────────┐         ┌──────────────────┐           │
│  │ Local store  │         │ OS Sandbox       │           │
│  │ - SQLite     │         │ - macOS sb_init  │           │
│  │ - File art   │         │ - Linux landlock │           │
│  │ - Telemetry  │         │ - Win AppCont    │           │
│  └──────────────┘         └──────────────────┘           │
└──────────────────────────────────────────────────────────┘
                      │
                      ▼ (HTTPS, opt-in)
              Remote services:
                - DeepSeek API / Anthropic / OAI
                - Telemetry collector (anonymized)
                - Update server
                - Skills registry
```

### Stage 3·Deep Dive（选 3 个）

**Pick 1·Query Engine（agent loop）**：

```typescript
interface QueryEngineConfig {
  model: string;
  maxTurns: number;             // 默认 30
  maxBudgetUsd: number;         // 默认 5
  fallbackModel?: string;       // 主模型 fail 时降级
  thinkingConfig?: ThinkingConfig;
  abortSignal?: AbortSignal;
}

class QueryEngine {
  async *run(input: UserInput): AsyncGenerator<StreamEvent> {
    let messages = [...await buildContext(input)];
    let turn = 0;
    let totalCost = 0;
    const tokenHistory: number[] = [];

    while (turn < this.config.maxTurns && totalCost < this.config.maxBudgetUsd) {
      if (this.config.abortSignal?.aborted) {
        yield { type: 'aborted' };
        return;
      }

      const stream = this.modelAdapter.stream(messages, this.tools);
      const response = await this.consumeStream(stream, yield);
      messages.push({ role: 'assistant', content: response.content });
      totalCost += response.cost;
      tokenHistory.push(response.outputTokens);

      // diminishing returns
      if (tokenHistory.slice(-3).every(t => t < 500)) {
        yield { type: 'stopped_diminishing' };
        return;
      }

      if (response.stopReason === 'end_turn') {
        yield { type: 'done' };
        return;
      }
      if (response.stopReason !== 'tool_use') {
        throw new Error(`Unexpected stop: ${response.stopReason}`);
      }

      const toolResults = await this.dispatchTools(response.toolCalls);
      messages.push({ role: 'user', content: toolResults });
      turn++;
    }

    yield { type: 'budget_exhausted' };
  }
}
```

**Pick 2·Permission Engine**：

```
┌─ Decision Pipeline ──────────────────────────────────┐
│ Tool call comes in                                   │
│   ↓                                                  │
│ 1. Configured deny rules → if match: DENY (final)    │
│   ↓                                                  │
│ 2. Hook PreToolUse → may return decision             │
│   ↓                                                  │
│ 3. Speculative classifier (with 2s timeout)          │
│    - high risk → ask                                 │
│    - low risk → allow                                │
│   ↓                                                  │
│ 4. Tool-specific checkPermissions                    │
│   ↓                                                  │
│ 5. Mode override (plan/acceptEdits/bypass)           │
│   ↓                                                  │
│ 6. If ask → UI dialog (with timeout)                 │
└──────────────────────────────────────────────────────┘
```

**Pick 3·Tool System with MCP**：

```typescript
interface ToolRegistry {
  register(tool: ToolDef): void;
  registerMCPServer(config: MCPServerConfig): Promise<void>;
  list(): ToolDescriptor[];
  call(name: string, input: unknown, ctx: Context): Promise<ToolResult>;
}

interface MCPServerConfig {
  name: string;
  transport: { type: 'stdio'; command: string; args: string[] }
           | { type: 'sse'; url: string };
  permission: PermissionPolicy;
  timeout: number;
}

class MCPClient {
  private process: ChildProcess | null = null;
  private requestId = 0;
  private pending = new Map<number, Deferred>();

  async start() { /* spawn + initialize */ }
  async listTools(): Promise<MCPTool[]>;
  async callTool(name: string, args: any): Promise<MCPResult>;
  async stop();

  // supervisor: 崩溃自动重启 with exponential backoff
  private async supervise();
}
```

### Stage 4·Scale + Failure + Observability

- **Scale**：桌面端单用户低 QPS，瓶颈在远程 API；做 token cache + batch 请求
- **Failure**：
  - 远程 API down → fallback model
  - MCP server 崩 → supervisor 重启 + 降级提示
  - sandbox 启动失败 → fail-closed（拒绝 tool）
  - LLM 输出格式异常 → schema validator + retry with structured error
- **Observability**：OpenTelemetry → 本地 SQLite → 用户可视化导出 → opt-in 上报
- **Rollout**：4 阶段灰度（internal / beta / 10% / 100%）+ feature flag

---

## 题 2：设计一个 Tool Permission Engine（30 分钟）

### Stage 1·Clarify

- 哪些 tool 需要 permission？→ 假设：所有有副作用的（write/bash/network）
- 用户模式 vs CI 模式？→ 两种都要支持
- 跨 session 持久化？→ Yes，用户偏好需要

### Stage 2·High-level

```
Sources (5):
  1. Global deny list (system)
  2. User config (settings.json)
  3. Project config (.harness/rules.yaml)
  4. Hook returns
  5. Tool's own checkPermissions

Combiner:
  - Priority: deny > ask > allow
  - Deny rule 任一命中 → DENY
  - 否则按优先级聚合

Cache:
  - "accept once" / "accept session" / "accept persist" 三档
  - SQLite 存 persist 决策

UI:
  - Permission dialog with:
    - tool name + input preview
    - reason (from rule)
    - 4 选项：deny / ask once / session / persist
```

### Stage 3·Deep Dive

```typescript
interface PolicyRule {
  match: {
    tool?: string | RegExp;
    args?: Record<string, any>;  // JSON path query
    context?: { project?: string; user?: string };
  };
  decision: 'allow' | 'deny' | 'ask';
  priority: number;
  reason?: string;
  source: 'system' | 'user' | 'project' | 'hook';
}

class PolicyEngine {
  private rules: PolicyRule[] = [];
  private cache: Map<string, CachedDecision> = new Map();

  addRule(rule: PolicyRule) { /* sort by priority */ }

  async evaluate(call: ToolCall, ctx: Context): Promise<Decision> {
    // 1. Check session cache
    const cacheKey = this.fingerprint(call);
    if (this.cache.has(cacheKey)) return this.cache.get(cacheKey)!.decision;

    // 2. Run rules
    const matched = this.rules.filter(r => this.match(r, call, ctx));
    const denies = matched.filter(r => r.decision === 'deny');
    if (denies.length) return { type: 'deny', reason: denies[0].reason };

    const asks = matched.filter(r => r.decision === 'ask');
    if (asks.length) return { type: 'ask', reason: asks[0].reason };

    const allows = matched.filter(r => r.decision === 'allow');
    if (allows.length) return { type: 'allow' };

    // 3. Default
    return { type: 'ask', reason: 'no rule matched, default ask' };
  }

  recordApproval(call: ToolCall, scope: 'once' | 'session' | 'persist') {
    /* store in cache or persist */
  }
}
```

### Stage 4·Scale + Failure

- **Scale**：本地评估，无 scale issue
- **Failure**：
  - Hook 超时 → 5s 后视为 deny（fail-closed）
  - Cache 损坏 → 清空 cache，回退到 rule 评估
  - Rule 配置语法错误 → 启动时校验，错误不让启
- **Audit**：所有 deny 进 audit log；所有 persist 进 export

---

## 题 3：设计一个 LLM Inference Gateway（Anthropic 常考）

> Anthropic 系统设计面试经典题：低延迟 LLM 推理网关，支持 streaming + rate limit + 观测。

### Stage 1·Clarify

- QPS？→ 10K QPS（高峰），平均 token 4K input + 1K output
- SLO？→ p95 TTFT (time-to-first-token) < 200ms；p99 total < 5s
- 多租户？→ Yes，按 user/org tier 限流
- 多模型？→ 3~5 个模型，可路由

### Stage 2·High-level

```
Client (SDK / HTTP / SSE)
  ↓
Edge (CDN / Anycast)
  ↓
API Gateway
  ├── Auth (JWT verify)
  ├── Rate limiter (Redis token bucket)
  ├── Quota check
  └── Route
  ↓
Request Router
  ├── Model selector (load + cost + tier)
  ├── Sticky session for streaming
  └── Fallback chain
  ↓
Inference Cluster
  ├── Batch scheduler (continuous batching)
  ├── KV cache pool
  └── GPU workers (vLLM / SGLang / TGI)
  ↓
Response Stream (SSE)

Cross-cutting:
  - Observability (OpenTelemetry → metrics / traces / logs)
  - Safety filter (input + output)
  - Audit log (for compliance)
```

### Stage 3·Deep Dive

**Batch Scheduling**：

```
- Continuous batching (vLLM 风格)：动态 join 新请求到正在跑的 batch
- max_batch_size = 256
- max_prefill_tokens per batch = 32K
- Eviction：LRU on KV cache
```

**Rate Limit + Quota**：

```
- Token bucket per user/org/tier
- Tier:
  - Free: 100K TPM
  - Pro: 1M TPM
  - Enterprise: 10M TPM (priority queue)
- 用 Redis (sliding window log) 实现
- 429 response with Retry-After header
```

**Streaming**：

```
- SSE 协议 (event: + data:)
- Sticky session by request id（同一 stream 必须到同一 worker）
- TTFT 优化：predicted batch slot + speculative decoding
- Backpressure：client 慢消费时 buffer cap，超出断开
```

### Stage 4·Scale + Failure + Safety

- **Scale**：水平扩 worker + GPU pool；KV cache pool 用 RDMA / NVLink 共享
- **Failure**：
  - Worker crash → request retry on another worker（idempotent via request_id）
  - Cache pool down → fallback to recompute（latency 涨）
  - Total cluster down → fallback model（GPT-4 → GPT-3.5 风格 degrade）
- **Safety**：
  - 输入 prompt injection 过滤（pattern + ML）
  - 输出 PII / harm filter（streaming 内 chunk check）
  - 触发 → 返回 modified content + flag

### Anthropic 风格的"安全 SLI"

> Anthropic 系统设计独特点：safety incident rate **平级于** latency / availability SLI。
> 一个 fast 但 produce harmful 的系统 = 坏了。

---

## 题 4：设计 Multi-Agent Orchestrator（30 分钟）

### Stage 1·Clarify

- 模式？→ 假设 Controller-Worker + 可扩 Planner-Generator-Evaluator
- 通信？→ 文件 + in-memory state，不走 message queue
- Scale？→ 单 host，最多 10 agent 并行

### Stage 2·High-level

```
Orchestrator
  ├── Agent Registry (Planner/Generator/Evaluator/Worker)
  ├── State Store (file artifacts + SQLite metadata)
  ├── Communication Bus (file watcher + in-memory channel)
  └── Lifecycle Manager
        ├── spawn / monitor / kill
        ├── timeout enforcement
        └── budget allocation

Each Agent:
  - Own QueryEngine instance
  - Own context window
  - Own tool subset
  - Reports to orchestrator via:
    - Status updates (running/done/failed)
    - Artifacts (files written)
    - Summary (final output)
```

### Stage 3·Deep Dive

**Sprint Contract 实现**：

```typescript
interface SprintContract {
  sprintId: string;
  generatorAgentId: string;
  evaluatorAgentId: string;
  spec: string;
  definitionOfDone: string[];  // testable criteria
  status: 'negotiating' | 'agreed' | 'in_progress' | 'evaluating' | 'done' | 'failed';
  iterations: number;
  maxIterations: number;
}

// 流程：
// 1. Orchestrator spawn Generator + Evaluator
// 2. Generator 提 proposal → 写 sprint-contract-draft.md
// 3. Evaluator 读 → 反馈 → Generator 改 → 直到 status='agreed'
// 4. Generator 实施 → 写 sprint-N-output/
// 5. Evaluator 跑测试（Playwright MCP / pytest）→ 给分
// 6. 通过 → done；不通过 + iter < max → 回 4
```

### Stage 4·Scale + Failure

- **Scale**：max 10 agent；超出 queue
- **Failure**：
  - Agent timeout → kill + report orchestrator
  - Communication 文件冲突 → file lock / atomic write
  - Sprint contract negotiation 死循环 → max negotiation rounds = 3
- **Observability**：
  - 每个 agent 的 trajectory 单独存
  - Orchestrator 看板（哪个 agent 在干什么，多久）

---

## 题 5：设计 Trajectory Telemetry Pipeline（30 分钟）

### Stage 1·Clarify

- 用途？→ 模型训练反馈 + 产品分析
- 合规？→ opt-in + GDPR / PIPL friendly
- Scale？→ 5K~50K user，每天 100K~1M trajectory

### Stage 2·High-level

```
Client (Harness instance)
  ├── In-process telemetry SDK
  ├── Local SQLite buffer (5MB ring)
  └── Background uploader (every 5 min when online)
  ↓ HTTPS, batched, compressed
Collector Gateway
  ├── Auth
  ├── Validate schema
  ├── PII detector (extra layer)
  └── Drop / accept
  ↓
Stream Processor (Kafka / Flink)
  ├── Enrich (geo / version)
  ├── Classify (LLM-based tagging)
  └── Route
  ↓
Storage:
  - Hot: ClickHouse / OLAP (30 days)
  - Cold: S3 Parquet (90 days)
  - Training: separate, opt-in-only
```

### Stage 3·Deep Dive

**Schema**（三层）：

| 层 | 字段 | 必填 | 默认 opt-in |
| --- | --- | --- | --- |
| Mandatory | trajectory_id, model, harness_ver, turn_count, stop_reason, outcome, duration, tokens | Yes | Yes |
| Aggregated | scenario_tag, difficulty, self_fix_count, tool_breakdown | No | Yes |
| Sensitive | messages, tool_inputs, file_diffs | No | **No (require explicit consent)** |

**合规设计**：

1. Mandatory **永不含代码 / PII**——开箱即开
2. Sensitive 必须 explicit UI consent + 可撤回
3. Right-to-be-forgotten：删 user_id 全 cascade（user_id_hash 索引）
4. Differential privacy：聚合统计用 noised 数据
5. 数据出境：中国大陆数据存大陆 region

### Stage 4·Scale + Failure

- **Scale**：100K/day events，Kafka 5 partition 够
- **Failure**：
  - 客户端离线 → SQLite 5MB ring buffer
  - 上报失败 → exponential backoff，max 24h retry
  - Collector down → drop with metric
  - Schema 变更 → 版本号 + dual writer
- **Cost**：S3 parquet + 90d 比 ClickHouse cheap 10×

---

## 题 6：设计 Eval / Benchmark Pipeline（30 分钟）

### Stage 1·Clarify

- 触发？→ 每次 ckpt 上传 / harness release / weekly
- benchmark 种类？→ SWE-Bench + 内部 task + adversarial probe
- 输出？→ leaderboard + diff report + failure case browser

### Stage 2·High-level

```
Trigger (ckpt / release / cron)
  ↓
Eval Orchestrator
  ├── Test set loader (versioned)
  ├── Parallel runner
  └── Result aggregator
  ↓
Test Workers (跑 task)
  ├── Sandbox per task
  ├── Time / token budget
  └── Capture trajectory
  ↓
Scorers
  ├── Auto: pass/fail (unit tests)
  ├── LLM-as-judge (calibrated)
  └── Human (sampling)
  ↓
Reporter
  ├── Leaderboard (vs baseline)
  ├── Diff vs previous version
  ├── Radar chart (4 dim)
  └── Failure case explorer
  ↓ (Slack / Email / Looker)
Researcher + PM
```

### Stage 3·Deep Dive

**Anti-contamination**：

- Test set 永不进 training data
- 每季度 holdout 20% 强制更新
- 公开 benchmark + 私有 benchmark 分开（公开用于宣传，私有用于真决策）

**LLM-as-Judge calibration**：

- Judge 用 stronger 模型（评 V3.5 用 V4 / Sonnet）
- 月度 50 条人工校准，accuracy < 0.85 重 prompt
- Judge 自己也跑 sycophancy probe

**Failure case explorer**：

- 每个 fail 链接完整 trajectory（messages + tool calls + screenshots）
- 按 failure mode 自动归类（schema error / timeout / wrong answer / refusal）
- 一键 "send to researcher" 按钮

### Stage 4·Scale + Failure

- **Scale**：100 task × 3 model = 300 runs，并行 30 workers × 10 task each，~1h 完成
- **Failure**：
  - Worker crash → task requeue
  - Eval 自身有 bug → 历史 baseline 重跑 sanity check
  - Test 不稳定（flaky）→ retry 3 次 + 标记 flaky

---

## 通用 trade-off "弹药库"

> 系统设计里几乎每题都会触发 trade-off。背熟这 10 个常见 trade-off pair：

| Trade-off | A 优 | B 优 | 选 A 适合 | 选 B 适合 |
| --- | --- | --- | --- | --- |
| **Latency vs Throughput** | 低延迟 | 高吞吐 | 用户面 | 后台 batch |
| **Consistency vs Availability** | 一致 | 可用 | 钱 / 关键状态 | 缓存 / 推荐 |
| **Strong typing vs Flexibility** | 强类型 | 灵活 | 长生命 schema | 早期探索 |
| **Compact vs Reset (Harness)** | 不丢历史 | 干净 slate | 短任务 / 强模型 | 长任务 / context anxiety |
| **In-process hook vs Out-of-process** | 快 / 简单 | 安全 / 隔离 | 内部信任 | 第三方扩展 |
| **OS sandbox vs Container** | 启动快 | 隔离强 | 桌面 | 服务端 |
| **Sync vs Async** | 简单 | 高并发 | < 100 QPS | > 1K QPS |
| **DB write-through vs write-back** | 持久 | 性能 | 关键 | 缓存场景 |
| **Centralized config vs Distributed** | 易管 | 抗故障 | 内部 | 分布式 |
| **Permissive default vs Deny default** | 灵活 | 安全 | 1st-party tool | 3rd-party MCP |

---

## 一句应试钥匙

> **系统设计的最高境界不是"我把所有组件都列出来"，是"我先讲为什么这么分层，再讲每层 trade-off，然后等面试官指方向"。**

> **永远让面试官决定深挖哪里**——你只负责把每一处的 trade-off 讲明白。
