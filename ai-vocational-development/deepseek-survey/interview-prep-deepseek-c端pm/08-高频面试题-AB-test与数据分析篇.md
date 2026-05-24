# 08 · 高频面试题·A/B Test 与数据分析篇

> C 端 PM 必备硬技能——A/B 设计 / 显著性 / 数据陷阱 / Funnel debug。
>
> 8 道高频题 + 必背数字 + 常见 trap。

---

## 一、必背数字 / 公式（**面试前烂熟**）

### 1.1 样本量估算（最高频）

> **简版公式**（基线转化率 p，期望提升 effect Δ，alpha=0.05, power=0.8）：
>
> $$n = \frac{16 \cdot p (1-p)}{\Delta^2}$$ 
>
> （每组各 n，total 2n；16 是 0.05/0.8 时的 z 因子近似）

**经验值（一组样本量）**：

| baseline p | 期望 Δ | 单组 n | 总样本 2n |
| --- | --- | --- | --- |
| 30% | 1pp (3.3% rel) | ~33,000 | 66K |
| 30% | 3pp (10% rel) | ~3,700 | 7.5K |
| 30% | 5pp (17% rel) | ~1,350 | 2.7K |
| 10% | 1pp (10% rel) | ~14,400 | 28.8K |
| 10% | 3pp (30% rel) | ~1,600 | 3.2K |
| 5% | 1pp (20% rel) | ~7,600 | 15.2K |

**速估法则**：
- baseline 30%+：要测 1pp 差异需 ~30K/组；要测 3pp 差 ~4K/组
- baseline 5-10%：要测 1pp 差异需 10-15K/组；要测 3pp 差 ~2K/组

### 1.2 显著性常识

| 概念 | 经典阈值 |
| --- | --- |
| Significance (alpha) | 0.05 |
| Power (1-beta) | 0.80 |
| p-value < 0.05 | 拒绝 H0，认为有差异 |
| Confidence interval 95% | 区间不含 0 = 显著 |
| 最小可检测效应 MDE | 业务可解释最小差 |

### 1.3 测试周期

| 类别 | 推荐周期 |
| --- | --- |
| Conversion / quick action | 1-2 周 |
| Retention / engagement | 4-6 周（要看次日 + 周次留存）|
| LTV / 付费 | 8-12 周+ |
| Habit-forming feature | ≥6 周 |

> **黄金原则**：A/B 必须跑过 ≥1 个完整 user cycle（周日/周一不同行为；月初/月底不同）。

---

## 二、8 道高频题

### Q1·"设计一个 A/B test 评估新 onboarding 效果"

**示范答**：

> **第 1 步·明确 hypothesis**：
> - H：新 onboarding（3 个 sample prompt + 简化注册流程）比旧版能提升 'Day-1 retention'
> - H0：两者无差异
>
> **第 2 步·设计 spec**：
>
> | 项 | 设置 |
> | --- | --- |
> | **Variant** | A (control 旧 onboarding) / B (new) |
> | **Allocation** | 50/50 random |
> | **Population** | 新注册用户（不影响老用户） |
> | **Randomization unit** | 用户 ID (不是 session) |
> | **Primary metric** | Day-1 retention |
> | **Secondary** | Day-7 retention, 首次对话完成率, 注册转化率 |
> | **Guardrail** | 老用户体验不受影响（隔离）, NPS 不跌 |
>
> **第 3 步·样本量估算**：
> - Baseline Day-1 retention 30%
> - 期望 effect +5pp (30% → 35%, ~17% relative)
> - 用公式：n = 16 × 0.3 × 0.7 / 0.05² = ~1,350/组
> - 总：2,700 用户
> - 按日 800 新用户算 → 4 天可达样本
> - **保守 run 14 天**（覆盖 weekly cycle）
>
> **第 4 步·分析 + 决策**：
> - 14 天后看 primary metric 是否显著
> - 如显著 + secondary 一致 + guardrail OK → ship B 100%
> - 如显著但 guardrail 触发 → root cause + 迭代
> - 如不显著 → 看 effect size，如 too small 可能 metric 不敏感（换 metric）
>
> **常见陷阱**：
> - ❌ Peek too early（每天看 p-value，多次比较 inflate false positive）
>   - Mitigation：pre-register 分析时点，或用 sequential testing
> - ❌ 用 session 而非 user 作 randomization unit
> - ❌ Population 没隔离（老用户被卷入）
> - ❌ 没设 guardrail（核心指标涨但 NPS 跌没看到）
>
> **预期结果 storytelling**：
> - "我假设 effect +3-5pp，95% CI 不含 0；如 effect <1pp 即使 'significant' 也不 ship（business meaningless）"
> - "如观察到 'positive on primary but negative on Day-7'——可能 onboarding 太好做了 'sugar high' 短期 retention 高但用户认知 inflate 后期 disappointed → 这是 'short-term win, long-term loss' 经典 pattern"

---

### Q2·"A/B 跑了 1 周看 p=0.04 显著，可以 ship 吗？"

**Trap question**——表面 OK，实际有多个 caveat。

**示范答**：

> "**不能立刻 ship。3 个原因要 check**：
>
> **1·样本量 / 周期是否够？**
> - 1 周可能未跨 weekly cycle（行为周末 vs 工作日差异）
> - 用户 cohort 不完整（新人 vs 老人）
> - **建议**：至少跑 2 周覆盖 1 个完整 cycle
>
> **2·是否 peeking?**
> - 如果每天看了 p-value，1 周内 5 次 peek，type-I error 实际 inflate 到 ~15%（不是 5%）
> - p=0.04 在多次 peek 下可能 false positive
> - **修正**：用 Bonferroni / α-spending / sequential test
>
> **3·业务可解释吗？**
> - p<0.05 只说"统计上有差"，不说"业务上有意义"
> - 看 effect size + CI lower bound
> - 比如 effect +0.3pp 即使显著也不值得 ship complexity
>
> **决策矩阵**：
>
> | p | Effect size | CI | 行动 |
> | --- | --- | --- | --- |
> | <0.05 | 大且 meaningful | 不含 0 | Ship |
> | <0.05 | 小或边缘 | 接近 0 | 继续跑 or 不 ship |
> | <0.05 | 大但有 guardrail trigger | - | 不 ship，调研 trade-off |
> | 0.05-0.1 | 看 effect size | - | 继续跑增大样本 |
> | >0.1 | - | - | 不 ship |
>
> **额外 check**：
> - **Novelty effect**：新功能上线 1 周内用户因好奇 click 多——长期持平 → 跑足时间看 effect 是否衰减
> - **AA test sanity**：开始前跑 AA test 确认平台 randomization 正常
> - **Sub-population**：按 segment 拆——可能 overall 显著但某 segment 负面
>
> **最稳决策**：
> - **2 周 + effect size > MDE + guardrail OK + 业务可解释**——4 个 condition 全满足才 ship"

---

### Q3·"如何 debug 一个 funnel 转化率突然下降？"

**示范答**：

> **systematic 5-step**：
>
> **Step 1·确认数据 calibration**：
> - 是真下降还是测量改变？
> - 数据上报 / 埋点是否被改？
> - 用 'sanity check'：总流量 ↑ 但转化率 ↓——分子分母都需 verify
>
> **Step 2·拆 funnel 每一步**：
>
> 经典 funnel：
> ```
> 着陆页 → 点击 CTA → 注册 → 首次行为 → 转化
> ```
>
> 哪一步 step transition 率降？
>
> 例：
> - 着陆 → CTA：90% → 88%（小降）
> - CTA → 注册：60% → 45%（**主因！**）
> - 注册 → 首次：85% → 84%（无影响）
>
> Root cause 集中在 '注册' 环节。
>
> **Step 3·按维度 cross-tab**：
>
> | 维度 | 找什么 |
> | --- | --- |
> | 时间 | 何时开始降；突变 vs 渐降 |
> | 渠道 | 某渠道 only 降还是普遍 |
> | 设备 | iOS / Android / Web |
> | 地域 | 全球还是某区域 |
> | UA | 某 browser / OS 版本 |
> | 用户 cohort | 新 vs 老 |
>
> 例：iOS 注册转化大降，Android 不变 → 高概率 iOS-specific issue。
>
> **Step 4·关联事件**：
>
> - 当时哪些事件发生？
>   - 版本发布（最常见！）
>   - 后端服务 deploy
>   - 第三方 SDK 更新
>   - 模型 update
>   - 政策 / 法规变化
> - 用 deployment log 时间轴 vs metric 时间轴
>
> 例：iOS App v3.5 发布时间和注册下降时间吻合 → ~90% 是 v3.5 bug。
>
> **Step 5·验证 + 修复**：
>
> - 找一个 power user 实测重现
> - 看 error log 是否有 spike
> - Roll out 修复 → 监测恢复
>
> **常见 root cause**（排序）：
> 1. 版本发布 bug
> 2. 后端服务 degradation (latency / error rate)
> 3. 第三方依赖问题（如 OAuth 服务挂）
> 4. UI 改动导致 button 不可见 / 不可点
> 5. 文案改动让用户困惑
> 6. 季节 / 节假日
>
> **mindset**：
> - 转化率 debug 像医生看病——**先 vital signs (calibration), 再 anamnesis (timeline), 最后 examination (cross-tab)**
> - 不要 jump to conclusion；找 dominant 信号

---

### Q4·"如果 A/B test 结果模糊（p=0.07），怎么办？"

**示范答**：

> **3 个方向 explore**：
>
> **方向 1·增大样本（如果可行）**：
> - 继续 run 1-2 周看 p 是否降至 0.05 以下
> - 但注意：n 翻倍只让 SE √2 倍小——effect 真小则永远不显著
>
> **方向 2·sub-segment 分析**：
> - Overall p=0.07 但某 segment 可能强显著
> - 例：新用户 effect 大（p=0.01），老用户 0 effect → 可能 ship 给新用户 only
> - **caveat**：post-hoc segment analysis 易 cherry-pick；要 pre-register
>
> **方向 3·变体调整重测**：
> - 当前 variant effect 可能不够强
> - 改进 variant（更激进 design）重测
> - 例：onboarding 改动太微小 → 试 'aggressive simplification'
>
> **决策 framework**：
>
> | 情况 | 行动 |
> | --- | --- |
> | Effect 大但 p=0.07 | 继续跑；高概率最终显著 |
> | Effect 小 p=0.07 | 不 ship；effect size 无 business meaningful |
> | 有强 segment effect | Sub-targeted ship + pre-register followup test |
> | Guardrail 触发 | 不 ship；root cause |
>
> **不该做的**：
> - ❌ 'p-hacking'：换不同 metric / sub-cut 直到找显著
> - ❌ 强行 ship 解释 'effectively positive'——这是 self-deception
> - ❌ 直接判失败放弃——可能 design 需迭代
>
> **mindset**：A/B test 没结果是好事——告诉你 'this hypothesis 不成立'，避免 ship 无价值 feature 浪费长期 maintenance。 

---

### Q5·"如果 ChatGPT 把 Voice mode 卡掉成功率 30%，DeepSeek 类似功能怎么 design A/B 来 avoid 这种 issue？"

**示范答**：

> **Lesson from ChatGPT**：
> - Voice mode 早期 latency / accuracy 问题导致 user frustration
> - 30% 卡掉意味着 70% 完成——sounds OK 但其实是 disaster（vs text mode 95%+ 完成）
> - 通常 root cause：流量 spike / network spike / TTS/STT chain failure
>
> **我设计 DeepSeek Voice A/B 的 4 个 lessons**：
>
> **Lesson 1·tiered rollout（不是 50/50 直接 split）**：
>
> ```
> Day 0-7:   1% rollout (~10K user)  - infra stability
> Day 7-14:  5% rollout (~50K user)  - quality / completion
> Day 14-28: 20% rollout (~200K)     - retention / NPS
> Day 28+:   full GA if all green
> ```
>
> **Lesson 2·multiple success criteria**：
>
> | 指标 | 阈值 |
> | --- | --- |
> | Voice session 完成率 | >85%（vs ChatGPT 70% 教训）|
> | First Token Latency P95 | <1.5s |
> | STT accuracy (WER) | <8% |
> | Crash rate | <0.5% |
> | 用户 thumbs down 率 | <5% |
> | NPS（voice 用户）| ≥ text 用户 |
>
> 任一指标 trigger threshold → halt rollout + investigate。
>
> **Lesson 3·circuit breaker**：
>
> - 实时 monitor 完成率
> - 完成率 drop > 10% in 1h → 自动 disable for new requests
> - 等 root cause 解决再 re-enable
>
> **Lesson 4·user opt-out 友好**：
>
> - Voice mode 入口显眼但**可一键退回 text**
> - 失败 fallback：voice fail → 自动 fall to text，不强用户重试
>
> **第 3 步·post-launch monitoring**：
>
> 即使 GA 后仍维持：
> - Daily voice success dashboard
> - 周 quality 抽样 (LLM-as-judge + 人工)
> - 月用户访谈
>
> **mindset**：
> - **A/B 不是终点是 ramp**——大功能 tiered rollout > one-shot
> - **多指标比单指标可靠**——单一 metric 易 game / 偏狭
> - **预防 > 修复**——circuit breaker 比事后 rollback 体验好

---

### Q6·"如果新模型 quality 指标和用户满意度反向（quality ↑ NPS ↓），怎么办？"

**示范答**：

> **第 1 步·confirm 测量正确**：
> - Quality 怎么测？(internal benchmark vs holdout)
> - NPS 样本量 / cohort 一致吗？
> - 时间窗口对齐吗？
>
> **第 2 步·形成 hypothesis**：
>
> **H1·Benchmark / user 期望 misaligned**：
> - 模型在 academic benchmark (MMLU/GSM8K) 涨，但用户 daily query 不需要这些 capability
> - 例：reasoning 加强了数学，但用户主要写邮件
>
> **H2·Quality 提升带来 side effect**：
> - Reasoning 更强 → response 变长 / 变复杂 → 用户嫌啰嗦
> - Safety 加强 → more refusal → 用户被拒
> - Format 更严谨 → less conversational → 用户感觉机械
>
> **H3·Latency 增加**：
> - 新模型质量↑ 但 latency↑ → 用户 perceive worse
>
> **H4·特定 use case regression**：
> - 整体 quality↑ 但某 use case 实质退化
> - 例：math↑ 但 creative writing↓
>
> **第 3 步·诊断 + 行动**：
>
> **诊断**：
> - 拆 NPS by use case / query type
> - 抽样 user feedback 文本 → LLM 聚类抱怨主题
> - A/B 内部 holdout 新旧模型对比
>
> **行动 priority**：
>
> **如 H1**：
> - 改进 benchmark：加 'real-user-distribution' query 测试
> - 不再用 academic benchmark 作 ship gating
>
> **如 H2**：
> - 用户特定抱怨 → 用 SFT/RL 修正
> - 加 'response style' toggle 让用户选
>
> **如 H3**：
> - 用 model routing：简单 query 用快模型
> - 用 streaming / progressive content 缓解 perceived latency
>
> **如 H4**：
> - 模型 patch fix 退化部分
> - 或 dual model：default + creative variant
>
> **不应做**：
> ❌ 直接 rollback（损失 quality 提升的 benefit）
> ❌ 忽视 NPS（用户离开是 ultimate measure）
> ❌ 加 disclaimer 告诉用户 'AI 升级了' 期望调整（这是 cope，不是 fix）
>
> **mindset**：
> - **Quality metric ≠ user value**
> - **Engineering 满意 ≠ product 成功**
> - C 端 PM 的工作是 'translate model capability to user perception'——如果 capability 升但 perception 反，PM 失职

---

### Q7·"DAU 涨了 3% 怎么知道是 'real' 还是 'noise'？"

**示范答**：

> **3 层 verify**：
>
> **Layer 1·统计可靠性**：
> - DAU 历史波动 σ 是多少？
> - 3% 涨在 historical noise 内吗？
> - 用 control chart / EWMA 看是否 'special cause variation'
>
> **Layer 2·attributable cause**：
> - 同期有什么 trigger event？（release / PR / 竞品事件）
> - 没 trigger → 可能 noise / 季节 / 自然 trend
>
> **Layer 3·breakdown 验证**：
> - 按 cohort / channel / segment 拆——是否 broad-based 增长 vs 单点
> - 例：全 platform 涨 vs 仅 iOS 涨
> - Broad-based 更可能 real
>
> **Tools**：
>
> 1. **Control chart**：设 ±2σ control limit
>    - DAU 在 limit 内 = noise
>    - 超过 limit = signal
> 2. **Time series decomposition**：trend + season + residual
>    - 看 trend 是否真涨
> 3. **A/B test (best)**：
>    - 如果是新 feature 导致，A/B test 才能 clean attribute
>
> **数字示例**：
> - DAU = 1M, std σ = 30K (3%)
> - 一天涨 3% (30K) → 在 1σ 内 → 噪声
> - 连续 5 天涨 3% → 极不可能噪声 → real signal
>
> **mindset**：
> - **N=1 数据点不能下结论**
> - **多源 triangulation 比单 metric 可靠**
> - **统计意识比 ML 意识更重要给 PM**

---

### Q8·"如何识别 'Vanity Metric' vs '真价值 metric'？"

**示范答**：

> **Vanity Metric**：看着好但不带 long-term value。
>
> **5 个识别 signal**：
>
> | Signal | Vanity 例 | 真价值替代 |
> | --- | --- | --- |
> | **不影响用户行为** | 总 page views | Engagement depth |
> | **不影响 revenue / retention** | 注册数 | Day-30 retention |
> | **易被运营 game** | 短期 DAU spike via push | WMAU |
> | **没和 user value link** | 总 button clicks | Task success rate |
> | **缺 context** | 月度增长 50% (但 baseline 100) | 同 base 月度增长 |
>
> **常见 vanity metric**：
> - 总 downloads（installed ≠ used）
> - 总注册数（registered ≠ active）
> - DAU spike（acquisition 烟花 ≠ retention）
> - Page views（views ≠ value delivered）
> - Likes / shares（vanity ≠ behavioral commitment）
>
> **真价值 metric 5 大标准**：
> 1. **反映 user job done**
> 2. **leading or strong correlated to retention / LTV**
> 3. **难以 game**
> 4. **可 segment / cohort**
> 5. **action-driving**
>
> **AI Chat 真价值 metric 例**：
> - WMAU
> - 'Aha session' 率（首次/每周 thumbs up）
> - Sessions/active user（depth）
> - 持续 N+1 周活跃率（true habit）
>
> **测试一个 metric 是不是 vanity 的 trick**：
> - 问 'if this metric doubled overnight via gimmick, would the business be in much better shape 6 months later?'
> - 如答案 No → vanity
>
> **PM 防御 vanity 的 process**：
> - 任何新 metric 加 dashboard 前问 'what user action does this measure?'
> - 北极星不止 1 个的 organization 容易陷 vanity（每个 team 拉自己 metric）
> - 季度 review 砍掉低 ROI 监测 metric

---

## 三、常见 A/B 陷阱速查（背下来）

| 陷阱 | 描述 | Mitigation |
| --- | --- | --- |
| **Peeking** | 频繁看 p-value 多次比较 | Pre-register; sequential test |
| **SRM** (Sample Ratio Mismatch) | A/B 分流不均（应 50/50 实 48/52）| Chi-square test 检验 |
| **Novelty effect** | 新东西短期好奇 click | 跑足时间看是否 sustained |
| **Carryover effect** | 用户跨 variant 切换记忆 | User-level randomization, 隔离 |
| **Network effect** | A/B 互影响（如 social feature）| Cluster randomization |
| **Survivor bias** | 只看活跃用户 metric | 用 cohort-based analysis |
| **Simpson's paradox** | 整体涨但每 segment 跌 | Always cross-tab |
| **Multiple comparison** | 测 10 个 metric 必有 1 个假阳性 | Bonferroni / FDR |
| **Pre-experiment bias** | 流量在测前已偏 | AA test 前置 |
| **HTE (heterogeneous effect)** | 平均 effect 0，但 segment 一正一负 | Sub-group analysis |

---

## 四、面试官追问最爱的 5 角度

1. **"样本量你怎么算的？"** → 准备公式 + 经验值
2. **"如何防 peeking？"** → α-spending / pre-register
3. **"如果两组 baseline 不齐怎么办？"** → AA test + stratified randomization
4. **"长期 vs 短期 metric 冲突，跟哪个？"** → 看 business stage + guardrail
5. **"如果你的 hypothesis 错了你怎么 own it？"** → calibrated 表达 + 'fail fast learn fast'

---

## 五、最后忠告

> **数据题不是数学竞赛——考你的 'statistical wisdom'。**
>
> 普通 PM：'p<0.05 显著'
> 高分 PM：**'p=0.04 但 1 周 cycle 不够，effect size 小，且没考虑 novelty——我建议继续跑 1 周 + sub-segment 分析 + AA test sanity，再决定 ship/不 ship'**

> 这种"承认 noise / 怀疑数据 / 多 hypothesis / 行动 calibrated"的成熟度，是 DeepSeek 想要的 senior data-driven PM。
