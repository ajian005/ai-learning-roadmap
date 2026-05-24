# 13 · 自带作品集 · DeepSeek Harness 指标体系 + 实验/调研方案

> 这份文档是 [12 产品提案] 的"姊妹篇"。提案讲"做什么"，这份讲"怎么衡量是否做对"。
>
> **如果时间紧只能带一份去面试，带提案 [12]；如果能带两份，把这份当成深度展示**——研究员面试官看到这份会立刻判断你"data 真懂"。

---

## 一、指标体系总览（1 页可视化）

```
                    ┌──────────────────────────────┐
                    │     ⭐ 北极星指标 (NSM)        │
                    │ Weekly Active Deep User       │
                    │   × Autonomous Task Rate      │
                    └────────────┬─────────────────┘
                                 │
        ┌────────────────────────┼────────────────────────┐
        ▼                        ▼                        ▼
  ┌──────────┐            ┌──────────┐            ┌──────────┐
  │ Width    │            │ Depth    │            │ Trust    │
  │ 场景广度  │            │ 任务深度  │            │ 用户信任  │
  └──────────┘            └──────────┘            └──────────┘
        │                        │                        │
   ┌────┴────┐             ┌─────┴─────┐            ┌─────┴─────┐
   │Scenario │             │Autonomous │            │ Interrupt│
   │Coverage │             │Resolution │            │   Rate    │
   │         │             │  Rate     │            │           │
   ├─────────┤             ├───────────┤            ├───────────┤
   │ New     │             │ Mean      │            │ NPS       │
   │Scenario │             │Intervention│           │           │
   │  Unlock │             │/Task      │            ├───────────┤
   ├─────────┤             ├───────────┤            │ WoM (自发  │
   │Scenario │             │ Self-Fix  │            │ 推荐)     │
   │Retention│             │  Rate     │            └───────────┘
   └─────────┘             └───────────┘

护栏 (Guardrail) ─────────────────────────────────────────────
  Cost-per-Success │ Hallucinated Tool Args Rate │ p95 Latency │
  Permission Bypass Incidents │ Safety Red Lines              │
─────────────────────────────────────────────────────────────────
```

---

## 二、北极星指标详解

### 2.1 定义

> **Weekly Active Deep User × Autonomous Task Resolution Rate**

| 子项 | 定义 |
| --- | --- |
| Weekly Active Deep User (WADU) | 一周内有 ≥ 3 个 30 分钟以上完整 task 的 user_id 数量 |
| Autonomous Task Resolution Rate (ATRR) | 该周 task 中，"用户没有 reject 最终结果"且"用户中途打断 < 1 次"的占比 |
| 北极星 = WADU × ATRR | 二者乘积 |

### 2.2 为什么是这个

| 候选指标 | 否决理由 |
| --- | --- |
| DAU | 鼓励"刷量"，对 Agent 这种重决策产品没意义 |
| Task Completion Rate | 鼓励 agent "挑容易做"，reward hacking |
| User Satisfaction | 调研频次低、滞后、噪音大 |
| Token Consumption | 鼓励 verbose 输出，错误激励 |
| **WADU × ATRR** | **乘积形式让两边必须同涨——既不能水量也不能水质** |

### 2.3 跨产品对标

| 产品 | 类似定位的指标 |
| --- | --- |
| GitHub Copilot | "Suggestion acceptance rate × Active users" |
| Cursor | "AI completion 接受率 × Daily active" |
| Claude Code | （未公开，但据外部观察）"User-resolved task per session" |

### 2.4 落地

- **数据源**：opt-in telemetry，user_id 哈希
- **计算频率**：daily 算，weekly 报
- **决策接入**：每周一全员看板 → 月度产品 review → 季度 OKR 锚定

---

## 三、二级指标详解（每个都给定义 + 反向防 hack）

### 3.1 Width 类（场景广度）

#### Active Scenario Coverage

| 项 | 内容 |
| --- | --- |
| **定义** | 过去 28 天 ≥ N 次活跃使用的场景 bucket 数（bucket 总数 = 50） |
| **数据源** | task → LLM 自动 tagging → 进 scenario taxonomy |
| **Bucket 设计** | hierarchical：6 大类（debug / feature / refactor / docs / test / 其他）× 各 5-10 小类 |
| **N 阈值** | 起步 N=10，每季度根据用户基数调整 |
| **防 hack** | bucket taxonomy 不让 PM/团队随意改；每月 review 一次只允许加 / 拆 / 合 |

#### New Scenario Unlock Rate

| 项 | 内容 |
| --- | --- |
| **定义** | 每月新增"首次出现在 N 阈值上"的 bucket 数 |
| **意义** | 衡量产品向新场景扩张速度 |
| **预期** | M1-M3 每月 ≥ 5 个；M4-M6 每月 ≥ 3 个；M7+ 进入精细化 |

#### Scenario Retention

| 项 | 内容 |
| --- | --- |
| **定义** | 某 bucket 用户使用 7/30/90 天后还活跃的占比 |
| **意义** | 衡量该场景是否"PMF"——用户用了一次还会回来吗 |
| **关键发现**：retention 按场景看比整体 retention 更可决策 |

### 3.2 Depth 类（任务深度）

#### Autonomous Task Resolution Rate by Difficulty

| 项 | 内容 |
| --- | --- |
| **定义** | 按 difficulty bucket（easy / medium / hard / expert）切分的 ATRR |
| **意义** | 防止 overall ATRR 被 easy task 稀释 |
| **划分方式** | 估算 token / 步数 / 涉及文件数三维度自动分级 |
| **防 hack** | hard / expert 必须独立看，且 hard 涨幅 < easy 涨幅 = 红灯 |

#### Mean Intervention per Task

| 项 | 内容 |
| --- | --- |
| **定义** | 完成一个 task 平均需要用户介入（按 Esc、改 plan、reject 一次 tool call）的次数 |
| **意义** | 衡量 agent 的"独立性" |
| **基线** | M1 起步 ~5 次，目标 M6 ≤ 2 次，M12 ≤ 1 次 |

#### Self-Fix Rate

| 项 | 内容 |
| --- | --- |
| **定义** | tool/verify 失败后，agent 在不依赖用户介入的情况下成功 self-recover 的比率 |
| **意义** | 衡量 harness 错误恢复能力 |
| **数据源** | 在 verification loop 中埋点 |

### 3.3 Trust 类（用户信任）

#### Interrupt Rate

| 项 | 内容 |
| --- | --- |
| **定义** | 一个 task 中用户按 Esc / Ctrl-C 中断的比率 |
| **意义** | 反映用户对 agent 的信任度——信任越高、越少打断 |
| **基线** | M1 起步 ~40%，目标 M12 < 20% |
| **细节** | 必须按场景分层（debug 任务打断率天然高于 docs） |

#### NPS (季度调研)

| 项 | 内容 |
| --- | --- |
| **定义** | "你向同事推荐 DeepSeek Harness 的可能性？" 0-10 |
| **计算** | (% promoter) - (% detractor) |
| **意义** | 用户长期满意度信号 |

#### Word-of-Mouth Self-Recommend

| 项 | 内容 |
| --- | --- |
| **定义** | 用户在公开渠道（GitHub fork / Twitter / Discord 邀请） 自发推荐 / 提及次数 |
| **意义** | 比 NPS 更真实——人花时间推荐才是真信号 |
| **数据源** | 自动爬 + 月度人工 verify |

### 3.4 护栏指标

#### Cost-per-Success

| 项 | 内容 |
| --- | --- |
| **定义** | 单个成功 task 的 token 成本 |
| **意义** | 防止"加大 reasoning 提升完成率但成本爆炸" |
| **红线** | 不允许季度环比 + 50% 以上 |

#### Hallucinated Tool Args Rate

| 项 | 内容 |
| --- | --- |
| **定义** | tool call 输入参数包含不存在文件/函数/API 的比率 |
| **数据源** | 自动校验（文件存在 / API 真存在 / 函数名匹配） |
| **红线** | < 2% |

#### p95 Latency

| 项 | 内容 |
| --- | --- |
| **定义** | 单轮 LLM call 的 p95 延迟 |
| **意义** | 用户体验底线 |
| **红线** | < 5s（普通模型）/ < 15s（reasoning 模型） |

#### Permission Bypass Incidents

| 项 | 内容 |
| --- | --- |
| **定义** | 每月发生的 permission 系统被绕过的安全事件数 |
| **红线** | 0（任何 1 个都是 P0） |

---

## 四、指标看板 mockup（文字版 wireframe）

```
┌─────────────────────────────────────────────────────────────┐
│  DeepSeek Harness · Product Health Dashboard  Week 21·2026 │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  ⭐ NSM: WADU × ATRR     ┌────────────────────────┐         │
│                          │ 1,234 × 47%  = 580     │         │
│                          │ ▲ 12% WoW              │         │
│                          └────────────────────────┘         │
│                                                             │
│  Width    Scenario Coverage    34 / 50   ▲ 2                │
│  Depth    ATRR by difficulty   E:68% M:52% H:31% X:11%      │
│  Trust    Interrupt Rate       28% ▼ 2pp                    │
│           NPS                   42  ▲ 5                     │
│                                                             │
│  Guardrails                                                 │
│  Cost/Success    $0.42  ✓ (red @ $0.65)                     │
│  Halluc Tool Args 1.3%  ✓ (red @ 2%)                        │
│  p95 Latency     4.2s   ✓ (red @ 5s)                        │
│  Permission Incidents  0  ✓                                 │
│                                                             │
│  Top Scenarios (28d)                                        │
│  1. multi-file refactor    18,234 task                      │
│  2. PR review              15,789                           │
│  3. test write/fix         12,453                           │
│  4. docs gen (zh)          9,567 ← 中文场景增速最快         │
│  5. ...                                                     │
└─────────────────────────────────────────────────────────────┘
```

---

## 五、数据收集方案（5 源 × 3 层 × 2 用途）

### 5.1 数据源

| 源 | 类型 | 量级 | 准确度 |
| --- | --- | --- | --- |
| 1. Client telemetry (opt-in) | 自动 | 高 | 高 |
| 2. UI 结构化反馈 (👍👎) | 半自动 | 中 | 中 |
| 3. 季度问卷 | 主动 | 低 | 中 |
| 4. 月度深访 | 主动 | 低 | 高 |
| 5. 公开渠道挖掘（GH/Discord/小红书/Twitter） | 自动 | 高 | 低-中 |

### 5.2 数据层

```
L1 事件层（最细）：tool_call / token / stop_reason
   ↓ aggregate
L2 会话层：trajectory / outcome / cost
   ↓ aggregate
L3 用户层：retention cohort / NPS / scenario distribution
```

### 5.3 用途

- **产品迭代**：bug 修 / feature ship / UX 重设计
- **模型迭代**（喂给研究员）：trajectory 当 SFT/RL 样本、failure case 当 eval set

### 5.4 合规设计

| 数据类型 | 合规要求 |
| --- | --- |
| Telemetry | opt-in，首次安装弹窗，可随时关 |
| Code content | 默认**不上传**，可选 opt-in（清晰提示） |
| User 身份 | 哈希后存，原 user_id 仅本地 |
| 留存期 | 90 天默认，可设置 |
| 跨境 | 大陆数据存大陆，海外数据存海外 |

---

## 六、实验方案模板（拿来即用）

### 6.1 假设模板

```
H1 (alternative): 在 [feature/intervention] 上线后，
                  [主指标] 在 [user segment] 上提升 ≥ [effect size]
H0 (null):        无显著差异
```

### 6.2 样本量估算（示例）

| 参数 | 值 |
| --- | --- |
| baseline | 50% |
| MDE (minimum detectable effect) | 5pp (绝对) |
| α | 0.05 |
| power | 0.80 |
| **每组样本** | **~1,565** |
| 日均长 session | 500 |
| **预计跑天数** | **~7 天 (双臂 50/50)** |

> 用 `statsmodels.stats.power.NormalIndPower` 或 [SampleSizeCalculator](https://www.evanmiller.org/ab-testing/sample-size.html) 算

### 6.3 分配 + 分层

- **分配单位**：user_id hash（不可按 session）
- **强制分层 stratification**：
  - by user cohort（新用户 / 老用户）
  - by primary scenario（refactor / review / test / ...）
  - by difficulty preference
  - by model used (DeepSeek / OpenAI / ...)

### 6.4 分析检验

| 指标类型 | 检验 |
| --- | --- |
| 主指标 (转化率) | two-proportion z-test or chi-square |
| 主指标 (连续) | Welch's t-test (方差不等) |
| 多分组比较 | ANOVA + Tukey HSD |
| 多次 peek 时 | sequential testing (mSPRT) |
| 多指标多检验 | Bonferroni / BH 校正 |
| 同用户多 session | cluster-robust SE |
| 用户跑分非正态 | Mann-Whitney U |

### 6.5 决策矩阵

| 主指标 | 护栏 | 决策 |
| --- | --- | --- |
| 显著 + 方向对 | 全部通过 | ✅ Ship |
| 显著 + 方向反 | / | ❌ Rollback |
| 不显著 | 全部通过 | 🟡 看定性 + 辅指标，再决定下一版 |
| 任一指标 | 任一红线 | ❌ Rollback |
| 显著 + 方向对 | 有 yellow flag | 🟡 等更多数据 + 团队讨论 |

---

## 七、用户调研方案模板

### 7.1 季度问卷模板（5 节、12 题、5 分钟）

```
Section 1·筛选 (2 题)
  Q1: 你过去 30 天用 Harness 多频繁？[5 档]
  Q2: 你主要用它干什么？[多选 6 类]

Section 2·NPS (2 题)
  Q3: 0-10 推荐可能性
  Q4: 你为什么打这个分？（开放）

Section 3·使用质量 (3 题)
  Q5: 任务成功率自评 [Likert 1-5]
  Q6: 信任度（敢离开屏幕跑 30min 吗）[Likert 1-5]
  Q7: 和 [CC/Cursor] 对比 [更好/差不多/更差/没用过]

Section 4·痛点 (2 题)
  Q8: 过去 7 天最大问题（开放）
  Q9: 希望 agent 更强的场景 [多选 6 类]

Section 5·人口学 + 联系 (3 题)
  Q10: 角色 [前/后/全/算法/PM/学生/其他]
  Q11: 主要语言 [中/英/其他]
  Q12: 愿意 30 分钟深访吗？[留邮箱]
```

### 7.2 用户访谈半结构化大纲（30 分钟）

```
[3 min] 破冰：自我介绍 + 强调没有错答案

[5 min] 背景：你在 [team/role]，平常 workflow

[10 min] 使用过程回溯（核心！）
  "上次用 Harness 是什么时候？"
  "从你打开它开始，能详细讲一遍那 30 分钟发生了什么吗？"
  → 听用户用什么词、什么时候皱眉、什么时候开心

[8 min] 痛点 + 假设验证
  "你刚才提到 [X]，能再细说吗？"
  "如果未来 Harness 多了 [假设 feature Y]，你会用吗？"
  → 注意：用户说"会用"≠真用；要追问"会用在哪个具体情况下"

[4 min] 延伸
  "还有谁你觉得我应该聊聊？"
  "对我们 PM 团队，你最想说的一句话是什么？"
```

### 7.3 访谈后 24h 产出"1 页 brief"模板

```
受访者：[hash id] · [角色] · [使用时长]
日期：YYYY-MM-DD · 时长 30 min

【3 个原话 quote】
1. "..."
2. "..."
3. "..."

【3 个 insight】
1. [现象] → [推测原因] → [产品 implication]
2.
3.

【1 个 action item】
- owner: PM / Engineer / Researcher
- 内容：
- deadline：
```

---

## 八、灰度发布 SOP

```
Stage 0·Internal dogfood    │ 1-3 天 │ ~200 人  │ 抓崩溃 + 安全
Stage 1·Closed beta         │ 5-7 天 │ 1%       │ 看核心指标方向
Stage 2·Open beta           │ 7-14 天│ 10%      │ 严肃 A/B 分析
Stage 3·Full rollout        │ 1-3 天 │ 25→50→100│ 护栏盯
```

每 stage 的 go/no-go：

| 指标 | 红线 |
| --- | --- |
| 崩溃率 | ≤ baseline × 1.1 |
| p95 latency | ≤ baseline × 1.2 |
| NSM | ≥ baseline − 2% |
| Safety incident | 0 |

---

## 九、Model Behavior Eval Pipeline（给研究员看）

```
触发：ckpt upload / harness release / weekly
  ↓
Stage 1·Capability    : SWE-Bench V/P + 内部 100 task → Pass@1/3 + step/cost
  ↓
Stage 2·Alignment     : 100 adversarial set + 50 sycophancy probe → 拒答/越界/sycophancy
  ↓
Stage 3·Honesty       : 50 calibration + 200 halluc probe → confession rate / halluc rate
  ↓
Stage 4·Efficiency    : reuse Stage 1 traj → steps/token/wall vs baseline
  ↓
Stage 5·LLM-as-judge  : 双盲对比上一版
  ↓
Stage 6·雷达图 + diff report → Slack + email → 研究员 + PM
```

**关键**：测试集 holdout / 季度更新 20% / LLM judge 人工 calibration（月度 50 条）

---

## 十、附：5 个我推荐的"反 Goodhart"配对

| 主指标 | 反向对冲 | 一起涨才算赢 |
| --- | --- | --- |
| Task Completion Rate | Cost-per-Success | 别为完成率烧 token |
| Trajectory Efficiency (步数少) | User-Accept Rate | 别草率收工 |
| Active Scenario Coverage | Scenario Retention | 别撒胡椒面 |
| WAU | Average Session Depth | 别追水量 |
| NPS | Self-Recommend WoM | 别只盯口头满意度 |

---

## 一句应试钥匙

> **指标体系最 PM 的 signature 不是"我能列多少指标"，是"我知道每个指标会被谁怎么 hack，所以我配了对冲"。**

> 当你能给每个北极星指标讲出 1 个对冲机制，研究员和工程师会立刻知道你不是来背模板的。
