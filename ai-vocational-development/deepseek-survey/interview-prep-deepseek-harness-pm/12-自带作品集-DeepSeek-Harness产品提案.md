# 12 · 自带作品集 · DeepSeek Code Harness V0.1 产品提案（示范模板）

> **这是你面试时可以带去的"反向开场材料"**——它把你从"被考核者"切换成"潜在同事"。
>
> **使用方法**：
> 1. 把这份文档另存为 PDF（用 Markdown→PDF 工具，建议 Typora / Obsidian / Marp）
> 2. 命名：`DeepSeek-Code-Harness-V0.1-Proposal-[你的名字]-2026-05.pdf`
> 3. 8~15 页范围（不要更长）
> 4. 面试开场说："**我面试前根据公开信息试着写了一份 V0.1 产品提案，里面有些点我也不确定，想顺着这个一起聊聊**"
> 5. **本文档是模板**——你必须**用你自己的判断替换 / 加私货**才有效。直接照搬会被识破。

---

## 提案核心信息（封面页）

> **DeepSeek Code Harness V0.1 产品提案**
>
> 子标题：**让中国研究员和开发者拥有自己的 reference harness**
>
> 作者：[你的名字]
> 版本：V0.1 / 2026-05-21
> 状态：候选人面试备选稿（公开信息推断版）

> **声明**：本提案基于公开信息（DeepSeek 招聘公告、Anthropic Harness 系列博客、Claude Code 源码级分析），不含任何 DeepSeek 内部信息；所有判断为候选人个人，供讨论与挑战，欢迎团队 review。

---

## 一、问题陈述（1 页）

### 1.1 我看到的市场状态（2026-05）

```
┌────────────────────┬───────────────────────────────────────────┐
│ Claude Code (CC)   │ 标杆 + 高门槛 + 高定价 + 模型锁定          │
│ Cursor             │ IDE-first，最大用户量，GUI 体验最好         │
│ Codex CLI          │ OpenAI 内部用 + 双层 sandbox + 云远程       │
│ Gemini CLI         │ Plan-Act-Validate，但 Gemini 长任务弱       │
│ OpenCode           │ 开源 CC 风、跨模型，社区起步                │
│ Manus/Cowork/...   │ 通用 Agent，编码 SWE 场景非主战             │
└────────────────────┴───────────────────────────────────────────┘

中国开发者真实选择：
  - 顶尖工程师：Claude Code / Cursor（付费用）
  - 普通开发者：通义灵码 / Cursor 试用 / GitHub Copilot
  - 学生 / 独立开发者：开源拼凑

→ 痛点：没有一个 "国产模型 + 中文优先 + 开源 + 桌面端" 的 reference harness
```

### 1.2 DeepSeek 的独特位置

- **唯一拥有 frontier-level 自家模型 + 长期开源信誉 + 中国市场原生性的玩家**
- 但**起步比 Claude Code 晚 1 年**——必须用差异化打开缺口，不是抄

### 1.3 我推荐的"破局命题"

> "**做一个 model-aware、开源、桌面端先行、对国产模型友好、对研究员透明的 reference harness——并让它成为 DeepSeek 模型 ↔ 真实用户场景的飞轮基座。**"

---

## 二、产品定位（1 页）

### 2.1 一句话定位

> **DeepSeek Code Harness = 让 DeepSeek 模型在真实编码任务中 100% 发挥其能力的 reference harness，同时也对所有开源国产模型友好。**

### 2.2 与对标的差异化矩阵

| 维度 | Claude Code | Cursor | Codex | OpenCode | **DeepSeek Harness（推荐）** |
| --- | --- | --- | --- | --- | --- |
| **形态** | TUI | IDE | CLI + 云 | CLI | **桌面 GUI + CLI 双轨** |
| **模型** | 锁定 Claude | OpenAI / Claude / Custom | OpenAI | 任意 | **DeepSeek 原生优化 + 任意开源模型** |
| **开源** | 闭源 | 闭源 | 闭源 | 开源 | **核心开源 + 企业增强商业** |
| **中文** | 弱 | 中 | 弱 | 中 | **中文优先** |
| **平台 Skills** | 强 | 中 | 弱 | 弱 | **混合（平台 + FS）** |
| **训练飞轮** | Anthropic 内部 | 弱 | OpenAI 内部 | 无 | **DeepSeek 内 + opt-in 用户** |
| **审计** | 强 | 中 | 强 | 弱 | **企业级强审计** |

### 2.3 目标用户 3 圈

```
                ┌─ 内圈：DeepSeek 内部研究员/工程师（dogfooding，~200 人）
                │
                ├─ 中圈：关注国产模型的开发者社区（GitHub 中文 dev / V2EX / 小红书）（~5000 人）
                │
                └─ 外圈：全球开源 LLM 用户、合规敏感企业（~50000 人）
```

### 2.4 用户场景优先级（90 天聚焦）

| 优先级 | 场景 | 为什么 |
| --- | --- | --- |
| P0 | 多文件 refactor / feature | SWE-bench 类，能直接对比 CC |
| P0 | 代码 review / PR triage | DeepSeek 内部高频，dogfooding 起点 |
| P1 | 中文/英文文档生成 | 突显中文优势 |
| P1 | 测试编写 + 修复 | 可 verifiable，eval 友好 |
| P2 | 代码迁移 / 升级（如 Python 2→3） | 长任务训练场 |
| P3 | UI 设计 / 前端 | 见 Anthropic 3 月那篇 |

---

## 三、北极星指标与衡量（1 页）

> 详见 [13-自带作品集-Harness 指标体系与实验方案.md]

### 3.1 北极星

> **Weekly Active Deep User × Autonomous Task Resolution Rate**
>
> （定义：在一周内有 ≥ 3 个 30 分钟以上完整 task 且其中 ≥ 50% 任务用户没有打断 / reject 的用户数）

理由：把 JD 里"更多人 × 更多场景 × 更深入"三轴压成一个可优化的乘积。光看用户数会鼓励"刷量"，光看完成率会鼓励"挑简单"，乘积让两边必须同涨。

### 3.2 二级 / 护栏指标

| 类型 | 指标 |
| --- | --- |
| 二级（depth） | Autonomous Task Resolution Rate by difficulty bucket |
| 二级（width） | Active Scenario Coverage（28 天 ≥ N 次的场景 bucket 数） |
| 二级（intensity） | Mean Session Duration、Tasks-per-User-per-Week |
| 二级（trust） | Interrupt Rate（按 Esc 的比例） |
| 护栏（quality） | Hallucinated Tool Args Rate |
| 护栏（cost） | Token-per-Success |
| 护栏（safety） | Permission Bypass Incident Rate |
| 反向（防刷） | NPS / 自发推荐次数 |

### 3.3 落地工具链

OpenTelemetry → 数仓 → LLM-augmented trajectory tagging → Looker/Superset dashboard → 周会 review

---

## 四、3 个月 v0.1 路线图（2 页）

### 4.1 总览

```
Month 1 ─► 单 Agent 跑通（CLI 形态）
            7 核心工具 + DeepSeek 模型适配 + 基础 sandbox + 1 个 lifecycle hook
            内部 50 人 dogfooding 启动

Month 2 ─► 长任务能力 + 评测
            PreCompact handoff + StuckDetector + 100 任务内部 eval set
            外部 closed beta 200 人

Month 3 ─► 飞轮 + 第一次外发
            trajectory 数据 pipeline + opt-in 上报 + Skills FS 挂载（社区起步）
            开源 v0.1 + 公开 leaderboard + 首篇技术 blog
```

### 4.2 详细 Sprint 表

| Sprint | 时间 | 目标 | 关键交付 | Risk |
| --- | --- | --- | --- | --- |
| S1 | M1 W1-2 | Agent Loop + 7 tools | minimal_agent + tool registry + Zod schema | tool 之间并发 race |
| S2 | M1 W3 | DeepSeek 模型原生适配 | tool_use parser + cache breakpoint | DeepSeek tool_use 格式漂移 |
| S3 | M1 W4 | Sandbox L1 + 基础权限 | OS-level sandbox + safe/medium/high 三档 + bash classifier | 不同 OS 兼容性 |
| S4 | M2 W1 | PreCompact hook + handoff | file-based handoff format + UI hint | handoff 格式标准化 |
| S5 | M2 W2 | StuckDetector + Circuit Breaker | repeater/wanderer/looper 检测 + 自动重提示 | 误判率 |
| S6 | M2 W3 | 100 task eval set | 内部 task 收集 + LLM-as-judge pipeline | task 多样性 |
| S7 | M2 W4 | Closed beta 200 人 | onboarding + Discord + 反馈 pipeline | beta user 留存 |
| S8 | M3 W1 | Trajectory 数据 pipeline | opt-in + 脱敏 + 合规审查 + 数仓 | 合规 |
| S9 | M3 W2 | Skills FS 挂载 | skill 协议设计 + 5 个示范 skill | 与未来平台兼容 |
| S10 | M3 W3 | 开源 v0.1 | 文档 + 示例 + README + LICENSE | 工程债 |
| S11 | M3 W4 | 公开 leaderboard + blog | benchmark 对比 + 技术 deep dive | 数据真实性 |

### 4.3 明确"不做的事"

- ❌ Subagent / Multi-Agent（v0.2+）
- ❌ GUI（v0.3+，先 CLI 跑稳）
- ❌ 平台 Skills（v0.2+，先用 FS 验证生态）
- ❌ 私有部署 / 企业版（v0.4+，先跑 PMF）
- ❌ Browser tool / Computer use（v0.5+，攻击面太大）

---

## 五、Model ↔ Harness 接口契约（1 页）

> 这是 JD 里"模型与 Harness 共同进化"的具体落地。

### 5.1 责任划分

```
┌─ Model 责任（让研究员训出来的能力）─────────────────────┐
│ - 准确的 tool_use 输出（schema valid）                  │
│ - 听从 deny rule 的 instruction following                │
│ - reasoning 在长任务里不过度消耗 token                   │
│ - 输出格式稳定（不在 tool_use 中插自然语言）             │
│ - 对 'I don't know' 的 calibration                       │
└──────────────────────────────────────────────────────────┘
                       ⇅
┌─ Harness 责任（用产品/工程兜底）────────────────────────┐
│ - Tool schema validation + retry with structured error  │
│ - Permission deny rule 永远 > allow                      │
│ - Reasoning budget 全局开关 + 场景自动判断               │
│ - 输出 parser 宽容 + 重提示                              │
│ - Hallucination detector + 自动 verifier                 │
└──────────────────────────────────────────────────────────┘
```

### 5.2 共同进化的机制

1. **Quarterly contract review**：每季度 PM + 研究员 + 工程师 review 一次 contract，决定哪些 harness 兜底可以"还给"模型
2. **Trajectory → reward signal pipeline**：harness 自动 surface 那些"频繁需要兜底"的 case，给研究员当训练材料
3. **Eval set 共建**：harness 测的 100 task 是模型团队和 PM 共同维护，每季度替换 20%

---

## 六、风险与开放问题（1 页）

> **故意留 5 个开放问题，邀请面试官 challenge / 一起讨论**——这是"反向开场"的真正杀招

| # | 问题 | 我的初步判断 | 想听团队的意见 |
| --- | --- | --- | --- |
| 1 | CLI 还是 GUI 先做？ | CLI 先（开发者优先） | DeepSeek 是否已经有 GUI 资源/团队 ready？ |
| 2 | 完全开源还是部分开源？ | 核心开源 + 企业增强 | 法务和商业化的边界 |
| 3 | 跨模型兼容 vs DeepSeek 紧耦合？ | 默认 DeepSeek + 兼容开源 | 团队是否担心被指责"为自家模型卖货" |
| 4 | Subagent 何时引入？ | v0.2，待模型 instruction following 提升 | 团队对当前模型能力的内部评估 |
| 5 | trajectory 数据合规如何处理？ | opt-in + 脱敏 + 分级 | 法务和研究团队对中国 PIPL / 欧美 GDPR 的优先 |

---

## 七、6 个月 / 12 个月愿景（半页）

### 6 个月

- 内部 200 名研究员/工程师 daily 用
- GitHub 5K+ star, 50+ external contributor
- DeepSeek+Harness 在自家 eval set 上跑赢 CC+Sonnet 4.5 ≥ 5pp
- 第一份公开 trajectory dataset 发布（脱敏，社区可用）

### 12 个月

- 月活 50K+ 外部用户
- 桌面 GUI v0.1 ship（中文优先）
- Subagent / Multi-Agent 支持
- 国产模型生态 5+ 个原生适配
- 形成 "DeepSeek Harness + 模型 + 数据飞轮" 三件套，对外讲故事完整

---

## 八、关于我（半页 - 候选人简介）

> 提案最后一页：把简历精华版放在这里，让面试官看到提案最后想得起"是谁写的"。

```
[你的名字]
[你的 title / 之前所在公司]

过去 [N] 年：[1 句最有 leverage 的项目]
最近 [N] 个月：[Agent 一手实践的 1 个证据]

公开作品：
- GitHub: [链接]
- Blog: [链接]
- Twitter: [链接]

为什么我适合 Harness PM：
1. Agent 一手高强度用户（CC/Cursor/Codex/OpenCode 日活）
2. 能用研究员语言（SFT/DPO/eval/reward）
3. 能用 PM 语言（指标/路线/用户研究）
4. 能 vibe coding 做 demo（这份提案的代码示例链接）

期待和团队深聊。
```

---

## 提案使用 checklist

面试当天：

- [ ] PDF 已生成、文件名清晰
- [ ] U 盘 + 云盘双备份
- [ ] 视频面试场景：提前共享屏幕跑通，不要现场切
- [ ] 线下：3 份打印（自己 + 2 个面试官），订书钉装订
- [ ] 开场 60 秒第一句话就提到它
- [ ] 不要求面试官现场读，只在被问到对应话题时翻到对应页
- [ ] 面试结束邮件 follow-up 时附上 V0.2（已加入今天聊到的内容）

---

## 给自己的提醒

> **这份提案的目的不是"惊艳"，是"启动一段对话"。**
>
> 如果面试官问"你这页第 3 点为什么这么设计"，你要能现场答；
> 如果面试官说"我们不会这么做"，你要能感谢并问"为什么"——记下来当作 V0.2 的更新。
>
> 你不是来证明你是对的。你是来证明你**值得被一起讨论**。

---

## 附：提案的 1 句 Twitter pitch（写给自己看，避免越写越偏）

> "Make DeepSeek's Code Harness the **open, model-aware, Chinese-first reference** that lets every developer harness frontier models in real coding — not just Anthropic's customers."

每次写一页提案前，复读这句话。**写偏了就停下来重写**。
