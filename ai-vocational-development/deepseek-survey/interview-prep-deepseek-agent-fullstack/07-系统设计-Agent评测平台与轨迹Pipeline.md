# 07 · 系统设计 · Agent 评测平台与轨迹 Pipeline

> 这是本岗位**最有可能整场出的系统设计题**——直接对应 JD 第 3 条职责。
>
> 同 [Eng 包 06](../interview-prep-deepseek-harness-engineer/06-高频面试题-系统设计与Harness组件篇.md) 的 4 阶段答题模板（Clarify → High-level → Deep Dive → Scale）。

---

## 一、题：设计 DeepSeek 内部 Agent Eval Platform（45-60 分钟）

> **现实背景**：DeepSeek 算法团队 50+ 人，每天产 5K~50K trajectory，需要 view / debug / score / 横评 / 反馈给模型训练。设计一个内部 platform 支撑这些 use case。

### Stage 1·Clarify（5 min）

主动问：
- 用户量？→ 内部 50 算法 + 20 研究员
- Trajectory 量？→ 假设 50K/天，每个 ~100 步
- 单个 trajectory 大小？→ 100 步 × ~10KB/步 ≈ 1MB；50K × 1MB = 50GB/天
- SLO？→ trajectory 上传后 1 min 内 visible；list latency p95 < 500ms
- 模型种类？→ 5~10 个 active；historical 100+
- benchmark？→ 内部 100 task + 公开 SWE-Bench / AiderBench
- Multi-tenant？→ 仅内部 + 项目隔离

### Stage 2·High-level Design（10 min）

```
┌────────────────────────────────────────────────────────────────┐
│                                                                │
│  ┌──────────────┐     ┌──────────────┐     ┌──────────────┐   │
│  │ Agent runner │     │ Algo dev local│     │ CI 自动 ckpt │   │
│  │ (K8s job)    │     │   (CLI tool) │     │     hook    │   │
│  └──────┬───────┘     └──────┬───────┘     └──────┬──────┘   │
│         │                    │                    │           │
│         └────────────┬───────┴────────────────────┘           │
│                      │ HTTP/gRPC upload                        │
│                      ▼                                         │
│             ┌────────────────────┐                             │
│             │  Ingestion API     │ ← validate schema           │
│             │  - rate limit      │ ← PII / size cap            │
│             │  - dedup           │                             │
│             └────────┬───────────┘                             │
│                      │                                         │
│                      ▼                                         │
│             ┌────────────────────┐                             │
│             │ Stream Processor   │                             │
│             │  - enrich          │                             │
│             │  - tag (LLM cls)   │                             │
│             │  - score (async)   │                             │
│             └────────┬───────────┘                             │
│                      │                                         │
│         ┌────────────┼─────────────┐                           │
│         ▼            ▼             ▼                           │
│    ┌────────┐  ┌──────────┐  ┌──────────────┐                 │
│    │ PG meta│  │S3 parquet│  │ Score Queue  │                 │
│    │  table │  │  step blob│  │ (LLM judge) │                 │
│    └────────┘  └──────────┘  └──────────────┘                 │
│         ▲                                                      │
│         │ Query API (REST/gRPC)                                │
│         │                                                      │
│  ┌──────┴──────────────────────────────────────────────────┐   │
│  │                  Web UI (React)                          │   │
│  │  - trajectory viewer (timeline + diff + playback)        │   │
│  │  - leaderboard / dashboard                               │   │
│  │  - filter / search / tag                                 │   │
│  │  - LLM-as-judge result display + comment                 │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                │
│  Cross-cutting:                                                │
│   - Auth (内部 SSO)                                            │
│   - Audit log                                                  │
│   - OTel trace + Prometheus metrics                            │
│   - Backup / Retention policy                                  │
└────────────────────────────────────────────────────────────────┘
```

### Stage 3·Deep Dive（15-20 min，选 3 组件）

#### Pick 1·Trajectory Storage 策略（最重要）

**Hot vs Cold 分层**：

```
Hot (90 day)
  - PG: metadata (id, model, task_id, outcome, step_count, ...)
  - PG: 最近 7 天 steps in JSONB （快速访问）
  - 索引：(model, created_at DESC) / (task_id) / outcome

Warm (90 day ~ 1 year)
  - S3 parquet：按 month partition
  - DuckDB over parquet 做 OLAP query
  - PG 只留 metadata + path 指针

Cold (1+ year)
  - S3 Glacier
  - 元数据删除前归档导出
```

**为什么 PG + S3 双存**：
- PG 强事务 + 强索引 + JSONB query → 适合"按 id 取一条 + filter list"
- S3 parquet + DuckDB → 适合"扫 100 万条做聚合"（leaderboard / 趋势 chart）
- 不用 ClickHouse / Druid → 50K/天小规模 overkill

#### Pick 2·LLM-as-Judge 异步队列

```
Trajectory upload → Score Required Tag
                         ↓
                  Score Queue (Redis Streams / Kafka)
                         ↓
              Score Workers (asyncio + semaphore)
                         ↓
            Judge LLM API (with retry + rate limit)
                         ↓
            Score Result Store (PG: scores table)
                         ↓
         Notify UI via WebSocket / poll
```

**关键设计**：
- 上传 ≠ scoring（解耦）；scoring 失败不影响 trajectory visible
- Idempotent：score_id = hash(trajectory_id + judge_model + prompt_version)
- Retry policy：3 次指数退避；最终失败标 "score_error" + 不阻塞展示
- Judge model 升级：增加 prompt_version，老分数不删
- 成本控制：每 trajectory 默认只评 1~2 个最关键指标；详细评 by user click

#### Pick 3·前端 UI 模块

**核心 view**：
- **List view**：filter（model / task / outcome / date range / score range）+ sort + tag + 增量加载
- **Single view**：timeline + step detail + LLM/tool/cost 标记 + 评论 + 标 tag
- **Diff view**：两个 trajectory 对齐对比（同 task 不同 model / 同 model 不同 ckpt）
- **Leaderboard**：model × benchmark grid + 历史趋势 line chart
- **Failure cluster**：自动 cluster 类似 fail（用 trajectory embedding）+ 一键 dispatch 给算法

**性能**：
- 单 trajectory 上千步：virtual list（TanStack Virtual）
- 大表 5K 行：TanStack Table + virtual row + server-side filter
- 实时更新：WebSocket 推 new trajectory event

### Stage 4·Scale + Failure + Observability（10 min）

#### Scale

- **10×**（500K/day）：partition PG 表 by month；S3 写入用 batch + 增 worker
- **100×**（5M/day）：上 ClickHouse / Kafka；DuckDB 已扛不住
- **read 热点**：Redis cache aside + ETag

#### Failure

- **PG down**：read-only mode（从 S3 直接渲染 trajectory metadata）
- **S3 down**：写入退化到本地 disk + 后台异步重传
- **Judge LLM 限流**：降级到 small judge / 暂停 scoring，trajectory 照样 visible
- **数据丢失风险**：上传走 idempotent + transaction；S3 跨 region 备份
- **架构灰度**：双写 → 切读 → 删旧（DB migration 标准做法）

#### Observability

```
OpenTelemetry SDK in:
  - upload API（http_server_duration / errors）
  - score worker（job_processing_time / queue_lag）
  - judge LLM call（external_api_duration / errors）
  - DB query（slow_query log + EXPLAIN）

Metrics in Prometheus:
  - trajectories_per_day (gauge / counter)
  - upload_latency_seconds (histogram)
  - score_queue_depth
  - judge_api_error_rate
  - storage_size_bytes (per model)

Logs in Loki/Elastic:
  - 结构化（JSON）
  - trace_id 贯穿

Alerts:
  - upload error rate > 1% sustained 5min → page
  - score queue depth > 10000 → page
  - judge api error rate > 5% → warn
  - storage > 80% capacity → warn
```

#### Rollout 策略

- 内部 alpha（5 个算法 dogfood，2 周）
- 内部 beta（全部内部用户，1 个月）
- production GA + 旧工具下架（提前 1 个月 announce）

---

## 二、题：设计 Trajectory Pipeline（30 分钟·独立题）

> 假设你只设计**数据采集 + 存储 + 给训练用的 pipeline**——不含 UI。

### Stage 1·Clarify

- 输入源？→ Agent runner（K8s job）/ 用户 CLI / CI
- 输出消费者？→ Eval platform（read）+ RL trainer（read）+ 研究员 analytics
- 数据要求？→ 完整 trajectory + step-level metadata + reward signal（如果有）
- 删除/合规？→ opt-in（mandatory 字段没敏感）；right-to-be-forgotten
- 实时性？→ 1 min visible OK

### Stage 2·High-level

```
[Producer]               [Pipeline]                  [Consumer]
agent runner   ─────►    Ingestion API
algo CLI       ─────►   (HTTP/gRPC)         ──────► PG metadata
CI hook        ─────►        │                        │
                             ▼                        │
                       Schema Validator                │
                             │                        │
                             ▼                        │
                       Stream Processor                │
                       (Kafka / pulsar)                │
                             │                        │
            ┌────────────────┼──────────────────┐     │
            ▼                ▼                  ▼     ▼
       S3 parquet      Score Queue       Tag Service  Read API
       (steps blob)    (judge LLM)       (LLM cls)
                                                       │
                                                       ▼
                              ┌──────────┬────────────┴───────────┐
                              ▼          ▼                        ▼
                         Eval UI    RL Trainer            研究员 notebook
                                     batch dataloader     (DuckDB / Polars)
```

### Stage 3·Deep Dive

#### 关键：Trajectory Schema 设计

```protobuf
syntax = "proto3";
package deepseek.harness.v1;

message TrajectoryMeta {
  string id = 1;
  string model = 2;
  string harness_version = 3;
  string task_id = 4;
  Outcome outcome = 5;
  int32 step_count = 6;
  int32 total_tokens = 7;
  int32 input_tokens = 8;
  int32 output_tokens = 9;
  int32 thinking_tokens = 10;
  int32 duration_ms = 11;
  int64 timestamp = 12;
  map<string, string> labels = 13;  // arbitrary tags
}

enum Outcome {
  OUTCOME_UNKNOWN = 0;
  SUCCESS = 1;
  FAIL = 2;
  ABANDONED = 3;
  TIMEOUT = 4;
}

message Step {
  int32 index = 1;
  StepRole role = 2;
  int64 ts_ms = 3;
  string content = 4;  // text portion
  repeated ToolCall tool_calls = 5;     // assistant role
  repeated ToolResult tool_results = 6; // user role with tool_use_id
  optional ThinkingBlock thinking = 7;
}

enum StepRole {
  STEP_ROLE_UNKNOWN = 0;
  USER = 1;
  ASSISTANT = 2;
  TOOL = 3;
}

message ToolCall {
  string id = 1;
  string name = 2;
  bytes input_json = 3;  // raw to preserve fidelity
}

message ToolResult {
  string tool_use_id = 1;
  string output = 2;
  bool is_error = 3;
}

message ThinkingBlock { string content = 1; }
```

> **为什么用 Protobuf**：
> - 强类型 + 跨语言（Python / Rust / TS gen）
> - 体积小（vs JSON）
> - 向前兼容（reserved field）
> - 训练 pipeline 喜欢 binary

> **JSON 何时合适**：
> - 给前端展示（JSON 易调）
> - 通过 Protobuf binary 存，转 JSON 渲染（gRPC-gateway 或自家 wrapper）

#### 关键：Schema 演进

```
1. 新字段必须 optional + 默认值
2. 字段不删，弃用标 DEPRECATED 注释
3. 不复用 field number
4. 加 schema_version 字段，consumer 按版本 dispatch
5. 旧 schema trajectory 仍可读（兼容性回归测试）
```

#### 关键：Sample / Stratified Sampling for Training

```
Training 不是吃全部 trajectory，而是 sample：
  - 按 outcome 分层（success / fail 比例可控）
  - 按 task 难度分层
  - 按 model 版本（每版本均匀）
  - hard negative mining：高 reward 的 fail + 低 reward 的 success
  - dedup 高度相似 trajectory（用 embedding 距离）
```

### Stage 4·Scale + Failure

- **Scale**：50K/天 → Kafka 5 partition；500K/天 → 20 partition
- **Backpressure**：producer 慢一点也 OK；不要丢
- **Idempotent**：trajectory_id 是 client 生成 UUID；DB unique constraint 防重
- **Replay**：consumer 失败可重读 Kafka offset

---

## 三、常被追问的"陷阱问题"

### 陷阱 Q1·如果某算法上传"乱七八糟"的 trajectory（schema 不合规），系统怎么反应？

> "我会三层防御：
> 1. **Ingestion API schema validator**：直接 reject，返回 400 with detail
> 2. **Stream processor 二次校验**：reject 单独记到 quarantine 表
> 3. **DLQ（dead letter queue）**：写不进去的最终落到 DLQ S3 bucket，人工处理
>
> 同时给上传者 immediate feedback：CLI tool 上传后等 ack，DB 没拿到就报错。
>
> 不要"静默 drop"——算法以为成功了，结果丢数据，是最坏体验。"

### 陷阱 Q2·算法对你的平台不满意要自己造一套怎么办？

> "先听 + 修。
> 1. 1:1 听他具体痛点（不是问"你不喜欢什么"，是 shadow 他用一次 platform）
> 2. 评估：低成本（< 1 day）我立刻修；中成本（< 1 周）排到下周；高成本说清原因 + 给替代方案
> 3. 我承认平台不是万能——比如算法 prototype 需要 raw JSON 访问，我直接给他 S3 read perm + 一个 DuckDB query 模板
> 4. 长期跟着这些"想自己造"的人——他们是产品 mortality 的早期指标
>
> 不要"我做的就一定要他用"——那是 vendor lock-in 心态；内部工具应该是 useful by choice，不是 mandated。"

---

## 四、一句应试钥匙

> **系统设计这一题的高分关键**：你能把 "评测平台 + trajectory pipeline" 拆成可以分阶段交付的 module，每个 module 都能讲清 trade-off + 边界 + 失败兜底。
>
> **DeepSeek 不喜欢一上来就 overkill（不用 K8s 上 Service Mesh + Vault + 5 个微服务），喜欢"先 monolith ship + 必要时再拆"的姿态。**
