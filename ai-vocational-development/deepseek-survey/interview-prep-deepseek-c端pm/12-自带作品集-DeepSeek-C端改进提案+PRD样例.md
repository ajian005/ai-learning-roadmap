# 12 · 自带作品集：DeepSeek C 端改进提案 + 完整 PRD 样例

> **C 端 PM 面试最有效的"硬通货"**——一份能 demo 的 DeepSeek deep critique deck + 3 个改进提案 + 1 个完整 PRD。
>
> **目标**：面试前 30 分钟摊在桌上 / 屏幕分享，让面试官 5 分钟内意识到 "**这候选人已经在头脑里 ship 过我们的产品**"。
>
> 这一份给：
> - PART A：deck 结构（10 页框架，你填内容）
> - PART B：3 个改进提案的 outline（直接可改）
> - PART C：1 个完整 PRD 样例（"Thinking Stream" 功能）

---

## PART A·Deck 结构（10 页 PDF / Keynote）

> **整 deck 12-15 页，30 分钟讲完整 / 5 分钟讲核心**。

```
Page 1·Cover
  - DeepSeek C 端产品改进提案
  - by [你的名字]
  - 2026-X-Y

Page 2·About Me (30 sec)
  - 3 句话背景
  - 为什么准备这份 deck

Page 3·我作为 DeepSeek 用户的 1 周观察
  - 7 天 use case list
  - 关键数字（如 'used reasoning mode 12 times'）

Page 4·20 个 Friction 点 (聚类 visualization)
  - 按 5 大类聚类（如 reasoning UX / 历史管理 / 移动适配 / 长 context / 搜索 citation）
  - 每类配 1 个 representative 截图

Page 5·Top 3 改进提案 overview
  - 表格：提案 / impact / effort / priority

Page 6-7·提案 1 详细 (含截图 / wireframe)

Page 8-9·提案 2 详细

Page 10-11·提案 3 详细

Page 12·How they affect DeepSeek north star
  - Each 提案 → 北极星 metric impact 估算

Page 13·30 天 / 90 天 行动建议
  - 如我加入会怎么 sequence

Page 14·我没解决的开放问题 (humility)
  - 3 个 questions to discuss with team

Page 15·Thank you + 问答 prompt
```

### 设计原则

- **少字 + 多视觉**：每页 ≤ 5 个 bullet；多用 screenshot / wireframe
- **数字驱动**：每提案配 metric impact + A/B 假设
- **承认 limitation**：不假装完美——开放问题让面试官参与
- **关注 ICE**：高 Impact + Confidence + Easy 优先

---

## PART B·3 个改进提案 outline

> **以下是高质量 candidate idea**——你可以根据自己真实观察改 / 替换。**关键是要有 specific friction 支撑**。

### 提案 1·Thinking Stream（reasoning UX 改进）

#### Problem
- DeepSeek R1 reasoning 在 web 端有 thinking trace，移动端 collapse 隐藏
- 用户开 reasoning 后看到 spinner 转 8 秒，感觉 "AI didn't do anything"
- 估算 reasoning 模式使用率 <15%（用户不会主动开）

#### Hypothesis
- Reasoning 过程**实时流式展示** + **可视化** 让用户感知 "AI 在深度思考"
- 提升 reasoning 模式使用率 + 提升 reasoning 完成率 + 提升用户 perceived value

#### Solution
- Phase 1·移动端 streaming thinking（mvp）
- Phase 2·thinking 步骤可视化（5 步骤 visual like Kimi）
- Phase 3·thinking 完成后可回看 + 分享

#### Metric
- Reasoning 模式使用率：15% → 30%
- Reasoning 模式完成率：65% → 80%
- 深度对话率（>5 turn）：+5pp

#### Effort
- 前端 4 周
- 设计 2 周
- 后端 minor (logging)
- 总：1 sprint cycle

#### Risk
- Reasoning 太长用户失去耐心 → 加 'show summary' 折叠选项
- 移动数据消耗 → 'low data mode' 开关

#### **ICE = 9 × 8 × 7 = 504**（高优）

---

### 提案 2·DeepSeek Projects（持久 workspace）

#### Problem
- 用户有 100+ 历史对话，只有时间排序 + 搜索
- 找具体对话困难（"昨天那个 GRPO 讨论..."）
- 跨对话无 context 共享（每次重新 explain background）

#### Inspiration
- Claude Projects: 持久 workspace + 10M token context
- ChatGPT GPTs: custom context
- Kimi 长 context: 文件上传

#### Solution
- "My DeepSeek" 模块（侧边栏入口）
- 每个 Project：name + description + uploaded files + persistent prompt + 对话 list
- 跨 Project 对话独立 context；Project 内对话 share context

#### Metric
- Day-30 retention for Project users vs non-user: +15pp
- 重度用户 (>10 对话/周) 占比：+20%
- 平均 sessions/user/week：+1.5

#### Effort
- 后端 6 周（DB schema / context management / file storage）
- 前端 4 周
- 设计 3 周
- 总：~2.5 sprint cycle

#### Risk
- Context window 限制 → 用 RAG retrieval 而非全 context 注入
- 隐私 (上传 file 处理) → 明确 data policy
- 重度用户 only feature → 占整体 use base 小

#### **ICE = 8 × 7 × 5 = 280**（中优——high impact but heavy build）

---

### 提案 3·Smart Citation (搜索 citation 强化)

#### Problem
- DeepSeek 联网搜索结果质量 OK，但 citation 是 "answer 末尾几个链接" 风格
- 用户难知哪句来自哪 source
- 对比 Perplexity 每段后 inline footnote，可信度感受落后

#### Hypothesis
- 加强 citation 可见性 → 提升用户信任 → 提升搜索类查询的使用频次

#### Solution
- Inline footnote (每段 markdown 后加 [1][2] 链接到 sources)
- Sources panel: 列所有引用 + 每个 source preview snippet
- Highlight: hover footnote → highlight source 中对应段

#### Metric
- 搜索类 query 满意度 (thumbs up): +10pp
- 搜索结果 click-through (用户 verify 行为): +50%
- Day-7 retention for 搜索用户：+3pp

#### Effort
- 前端 3 周
- 模型/后端 2 周（输出格式 + citation extraction）
- 总：1 sprint cycle

#### Risk
- 模型有时 confabulate citation → 必须 grounded
- Citation 太多视觉杂乱 → 默认 inline footnote，detail 在 panel

#### **ICE = 7 × 9 × 8 = 504**（高优）

---

### 3 提案对比表

| # | 提案 | ICE | 优先级 |
| --- | --- | --- | --- |
| 1 | Thinking Stream | 504 | High |
| 3 | Smart Citation | 504 | High |
| 2 | Projects | 280 | Medium |

**Sequencing 建议**：
- Q1: 提案 1 + 提案 3 并行（同 sprint cycle，effort 小）
- Q2: 提案 2 (Projects)
- Q3-Q4: 基于 Q1-Q2 learning iterate

---

## PART C·完整 PRD 样例：Thinking Stream

> **完整 PRD ~30 页内容浓缩 8-page 版本**。这是 demo level；实际 ship 会有更细 spec。

---

### **PRD: Thinking Stream**

**Owner**: [你的名字]
**Status**: V1 Draft
**Last updated**: 2026-X-Y
**Target launch**: Q3 2026

---

#### 1·TL;DR

把 R1 reasoning 过程从"隐藏在 spinner 后"改成"实时流式展示"，覆盖 Web + Mobile（iOS + Android）。预期 reasoning 模式使用率 15% → 30%，深度对话率 +5pp。

---

#### 2·Problem

**User pain (verbatim from 5 user interviews)**：

- "我开了 R1 但等了 8 秒看不到东西，以为坏了"（用户 A，iOS）
- "明明知道它在 thinking，但看不见过程，感觉不知道值不值得等"（用户 B，Web）
- "Kimi 的 5 步骤 thinking 比 DeepSeek 直观"（用户 C）

**Data**:
- Reasoning 模式使用率：~15% （估算）
- Reasoning 启动后 user-cancel rate: ~25%
- Reasoning 完成后 thumbs up rate: ~60%（vs 标准模式 50%）—— 说明 reasoning 真有价值，但 access friction 高

**Root cause**:
- Web 端有 thinking trace 但折叠默认（用户不会展开）
- 移动端完全 hidden（只有 spinner）
- 用户对 'AI 在 thinking' 缺感知 → 容易 cancel / 切回标准模式

---

#### 3·Goal & Non-goal

**Goal**:
- ✅ 让用户实时看到 reasoning 过程
- ✅ 提升 reasoning 模式使用率 + 完成率
- ✅ 在 Web + iOS + Android 一致体验

**Non-goal**:
- ❌ 改变 reasoning 模型 quality（model 层不动）
- ❌ 重新设计 reasoning UI from scratch（incremental）
- ❌ 加新功能如 reasoning history / 分享（future phase）

---

#### 4·User Stories

**As a 数学解题用户**：
- 我开 R1 解题
- 想看到 AI 思考过程
- 即使等 30 秒也不焦虑，因为看得到 progress

**As a 知识工作者**：
- 我问复杂分析题
- 想知道 AI 在如何拆解
- 看 thinking trace 我能 learn AI reasoning

**As a 新用户**：
- 我不了解 R1
- 但看到"思考过程"出现，意识到 AI 真在 think
- 体验 wow → retention 提升

---

#### 5·Solution

##### 5.1 UX 设计

**Web 端**：

```
┌──────────────────────────────────────────┐
│ [User] 解这道数学题: ∫(x²+1)/(x³+x)dx     │
├──────────────────────────────────────────┤
│ 🧠 深度思考中... (展开 ▼)                 │
│ ┌────────────────────────────────────┐   │
│ │ Step 1: 观察被积函数...            │   │
│ │ 我可以把分子拆为 ...               │   │
│ │ ...                                │   │
│ └────────────────────────────────────┘   │
│                                          │
│ [Answer streaming here]                  │
└──────────────────────────────────────────┘
```

- **默认展开** (vs 当前折叠)
- **Thinking 文字实时 streaming**
- 用户可手动折叠（remembered preference）

**移动端**：

```
┌─────────────────────────┐
│ User: 解这道题          │
├─────────────────────────┤
│ 🧠 思考中...            │
│ ── 正在分析问题结构      │
│ ── 拆解被积函数 ✓        │
│ ── 应用部分分式          │
│                         │
│ [Answer streaming here] │
└─────────────────────────┘
```

- **简化版**：只显示当前 step + 已完成 step（dot 标记）
- 不显示 full thinking text（移动端阅读不友好）
- 长按 dot 可查看 full thinking

##### 5.2 技术实现

- Model 层：不变（reasoning 模型原本就 output thinking tokens）
- API 层：current API 已 stream thinking + answer，前端只用 surface
- 前端：refactor render 逻辑——thinking part 单独 component，realtime 更新

##### 5.3 配置 / 用户控制

- 用户可在 Settings → Reasoning preference 选：
  - "Show thinking" (default ON)
  - "Show thinking summary only" (5 step 简版 + 点击展开)
  - "Hide thinking" (return current 行为)

- "Low data mode" 开关（移动端）：thinking 折叠 spinner

---

#### 6·Metric

##### Primary
- **Reasoning 模式使用率**：从 15% → 30%（+15pp）

##### Secondary
- Reasoning 完成率（不被 cancel）：65% → 80%
- 包含 reasoning 的 session Day-7 留存：+3pp
- Reasoning 模式 thumbs up rate：60% → 65%

##### Guardrail
- 整体 latency P95 不退化
- 整体 NPS 不退化
- 数据消耗（移动端）增加可接受范围（<10%）

---

#### 7·A/B Test Plan

**Variants**:
- A (control)：current UX (折叠 thinking)
- B (treatment)：new Thinking Stream UX (default 展开 + streaming)

**Population**:
- 新 + 老用户 50/50 split
- 限定开了 reasoning 模式的 session
- 排除特殊场景（如 enterprise account）

**Randomization unit**: User ID

**Sample size**:
- Baseline reasoning 使用率 15%
- Expected effect +10pp (15% → 25% in A/B period)
- 每组 n ≈ 2,000 reasoning session（按公式）
- 按日 ~10K reasoning 用户算 → 2 天可达；保守 run 14 天

**Decision criteria**:
- p < 0.05 AND
- Effect ≥ +5pp absolute AND
- Guardrail (latency / NPS) 不退化
- → Ship to 100%

---

#### 8·Rollout Plan

| Week | Milestone |
| --- | --- |
| 1-3 | 前端 implement (Web + iOS + Android) |
| 4 | Internal alpha (50 dogfooder) |
| 5 | 1% beta rollout + 监控 |
| 6-7 | 10% rollout + A/B confirmed |
| 8 | 100% GA |
| 9-12 | Iterate based on user feedback |

---

#### 9·Risks & Mitigations

| Risk | Likelihood | Impact | Mitigation |
| --- | --- | --- | --- |
| Thinking 太长用户失去耐心 | M | H | "summary only" 模式 + 'show more' |
| 移动数据增加 | M | M | Low data mode toggle |
| Thinking 内容 confusing 普通用户 | H | M | 5-step visual + plain language summary |
| 增加 frontend load → 性能问题 | L | M | Component virtualization |
| User opted "hide thinking" 不可见 | L | L | Settings 易找 + onboarding 教育 |

---

#### 10·Open Questions (discuss with team)

1. **Thinking summary 5 步骤是预定义模板还是 LLM 动态生成？**
   - 预定义：稳定但 not 适用所有 task
   - LLM 动态：自然但增加 cost / latency
   - 建议：beta 用预定义，post-launch evolve

2. **Default 展开 vs 折叠？**
   - 我提议默认展开 (vs current 折叠)；可 A/B 验证

3. **Web 端"展开 thinking"是 inline 还是 side panel?**
   - Inline：简单，但长 thinking 影响 reading flow
   - Side panel：clean，但需大 screen
   - 建议：default inline，可拖到 side panel

4. **数据保留 thinking traces 多久 + 用户能否回看历史 thinking?**
   - 涉及 storage cost + 隐私 trade-off
   - 建议：30 天保留 + 用户可手动 pin

---

#### 11·Resources

- Eng cost: 4 人 × 2 月 = 8 人月
- Design cost: 1 人 × 1 月 = 1 人月
- PM (myself): 1 人 × 3 月 = 3 人月
- Infra cost: ~marginal (logging only)
- **Total: ~12 人月 + minor infra**

---

#### 12·Appendix

- User interview notes (5 people, anonymized)
- Competitive UX screenshots (Kimi 探索版 5-step, Claude thinking, GPT o1)
- Mock-ups (Figma link)

---

## PART D·使用建议

### 面试前 1 周

- T-7：选 PRD topic + 草拟 V0
- T-3：PRD V1 draft 完成
- T-1：V1.0 polish + 印 3 份

### 面试当天

- 开场 60 秒提到 "我准备了一份 PRD 想分享"
- 面试官如 indicate 兴趣 → 主动展示 (10-15 分钟)
- 强调几个亮点：specific friction observation / A/B design rigor / open questions humility

### 面试后

- T+0 当晚：根据面试讨论 update PRD V1.1
- T+1：感谢邮件附 V1.1
- T+7：如时间允许 ship V2.0 incorporating feedback

---

## 最后忠告

> **作品集的核心价值不是 deck 多精美——是证明你 "提前 ship 了 30% 的 PM 工作"。**

> 普通候选人：面试官问 "你会怎么改进 DeepSeek" → 临场 brainstorm
> 高分候选人：**已经把 critique 写成 PRD，A/B test plan 都有，今天来不是问 'can I do PM'，是来对齐 'which 30% of my PRD do you want me to ship first'**

> 这种 power dynamic shift 是 senior C 端 PM 的 ultimate signal。
