# DeepSeek「Agent Harness 产品经理」面试准备包

> **岗位**：Agent Harness 产品经理（DeepSeek，桌面端 Agent 产品）  
> **团队**：Harness 团队（陈德里牵头，对标 Anthropic Claude Code）  
> **地点**：北京海淀区融科资讯中心  
> **本包整理时间**：2026-05-21  
> **目标**：把这份材料当作"面试前 7 天集中冲刺手册"，覆盖从理解 JD、行业调研、答题素材、临场策略到复盘跟进的全流程。

---

## 一、为什么要做这份准备包

DeepSeek Harness 团队是 2026 年 5 月才公开组建的"刚起步、第一波招聘"型小团队，由资深研究员陈德里牵头，对标 Anthropic Claude Code。这是一个典型的 **"小团队 / 高赌注 / 还没成型"** 的 0→1 岗位，意味着：

1. **没有先例可抄**：DeepSeek 还没正式发布过桌面端 Agent，JD 里所有动词（"定义""规划""维护""共同进化"）都不是套话，而是真正等你去做的事。
2. **PM 必须自己会动手**：JD 中明确写"能用 vibe coding 写代码""熟悉各种 Agent 形态"——意味着面试官会直接考你**用过没用过、能不能独立做出原型、对模型行为有没有品味**。
3. **跨界协作密度极高**：上接研究员、下接工程师、外接开源社区。这就是 Anthropic 的 PMM/Researcher-PM 那一套打法。
4. **DeepSeek 的面试以"高强度、压力面、扁平、不看花架子"著称**（3 小时连问、深挖项目细节、不设 KPI、不赛马、扁平化）。

**这份准备包就是围绕"能不能撑住这种面试 + 这种岗位"来设计的。**

---

## 二、目录与使用建议

| 序号 | 文件 | 内容 | 何时用 |
| --- | --- | --- | --- |
| 01 | [01-JD深度解读与岗位画像.md](01-JD深度解读与岗位画像.md) | 逐句拆解 JD、显性能力 + 隐藏信号、岗位画像、自检对照表 | T-7 ~ T-5 天，先吃透岗位 |
| 02 | [02-DeepSeek与Harness背景调研.md](02-DeepSeek与Harness背景调研.md) | DeepSeek 公司/团队/产品脉络、Harness 团队组建始末、对标 Claude Code 路径分析、可能的产品路线图 | T-7 ~ T-4 天，搭背景 |
| 03 | [03-Harness核心知识体系速查.md](03-Harness核心知识体系速查.md) | Agent Loop / KV Cache / Tool Use / Context Engineering / Skills / MCP / Memory / Subagent / Harness 七大组件——一份能直接背的速查卡 | T-4 ~ T-2 天，技术弹药 |
| 04 | [04-高频面试题与参考答案-产品认知篇.md](04-高频面试题与参考答案-产品认知篇.md) | 产品定位、用户洞察、指标体系、商业化、PMF | T-3 天，答题素材 |
| 05 | [05-高频面试题与参考答案-技术与Agent篇.md](05-高频面试题与参考答案-技术与Agent篇.md) | LLM/Agent/Harness 技术理解题、横向对比题、Codex/Cursor/CC/OpenCode 对比 | T-3 天，答题素材 |
| 06 | [06-高频面试题与参考答案-数据与实验篇.md](06-高频面试题与参考答案-数据与实验篇.md) | 数据收集、A/B、灰度、统计学、模型行为评测 | T-3 天，答题素材 |
| 07 | [07-高频面试题与参考答案-行为与文化篇.md](07-高频面试题与参考答案-行为与文化篇.md) | STAR 模板、跨团队冲突、与研究员协作、开源社区运营、抗压面 | T-3 天，答题素材 |
| 08 | [08-反问环节话术库.md](08-反问环节话术库.md) | 对面试官的高质量提问（HR/PM/研究员/工程师/老板四类） | T-2 天 |
| 09 | [09-面试前7天行动清单.md](09-面试前7天行动清单.md) | 日级 to-do、作品/简历/装备、心态/休息/演练节奏 | T-7 启动 |
| 10 | [10-面试中临场策略与话术.md](10-面试中临场策略与话术.md) | 开场 60 秒、卡壳话术、结构化答题模板、技术失误挽回 | T-1 复读 |
| 11 | [11-面试后复盘模板与跟进话术.md](11-面试后复盘模板与跟进话术.md) | 24h 复盘表、感谢信模板、被拒后的二次转化话术 | T+0 ~ T+7 |
| 12 | [12-自带作品集-DeepSeek-Harness产品提案.md](12-自带作品集-DeepSeek-Harness产品提案.md) | 一份你可以带去面试的"DeepSeek Code Harness V0.1 产品提案"——把它当反向开场 | T-2 ~ T-1 打磨 |
| 13 | [13-自带作品集-Harness指标体系与实验方案.md](13-自带作品集-Harness指标体系与实验方案.md) | 北极星指标 + 二级指标 + 实验/灰度/调研方案 demo | T-2 ~ T-1 打磨 |

> 文件命名以 **"序号 + 用途"** 而非"知识分类"组织，目的是按你"准备进度"取用，而不是当百科查。

---

## 三、整体策略（一句话总结）

> **"用产品经理的语言、研究员的严谨、工程师的动手能力，去回答一个还没有标准答案的问题：DeepSeek 的 Harness 到底应该长什么样。"**

具体拆三件事：

### 1. 把自己"重新封装"成 Harness PM
- 找到你过去经历里 **"决策驱动型 / 数据驱动型 / 开发者体验型 / 0→1 型"** 的素材，按 STAR 重写。
- 公开作品（GitHub / 博客 / Twitter / 小红书 / 知乎）整理成"作品集 URL 一行清单"，简历里贴。
- 把"用 Claude Code / Cursor / Codex 做出过的真东西"截图、整理成 1 页 PDF 作为聊天素材。

### 2. 准备一个"反向开场"——主动带方案进面试
DeepSeek 面试官最讨厌"背概念"。**最高效的破局是：你不是来回答问题的，你是来对齐问题的。**

具体落地见 [12-自带作品集-DeepSeek-Harness产品提案.md](12-自带作品集-DeepSeek-Harness产品提案.md)：

> "在准备这次面试前，我先按照公开信息试着做了一个 V0.1 的 DeepSeek Code Harness 产品提案，包括北极星指标、路线图猜想、和 Claude Code/Codex/Cursor 的差异化定位，里面有些点我也不确定，想顺着这个一起聊聊。"

这一句话就把你从"被考核者"切到了"潜在同事"。

### 3. 用"模型行为品味"碾压同质化候选人
JD 里反复出现 **"对模型行为有品味有判断力"**。这是 Anthropic / DeepSeek / Cursor 这一批公司共同的隐藏门槛。

你需要能在 30 秒内说出 3 个具体观察，例如：
- "Claude Code 在 PreCompact 之前会主动写 handoff 文件，Codex CLI 则倾向 in-place compaction，所以 Codex 在长任务上更容易出现 context anxiety。"
- "Cursor 的 Tab 补全和 Agent Mode 是两套上下文，Tab 用本地 AST + 短滑窗，Agent 用 RAG + 长上下文；如果两边状态不同步会出现幽灵编辑。"
- "OpenCode 的 Skills 走文件系统挂载，Claude Skills 走 Anthropic 平台注册——后者审计强，前者可移植。"

这种"具体到能复现"的观察，才是 PM 候选人能给出的真正稀缺信号。

---

## 四、面试评估维度的内部模型

按公开信息推断，面试官（陈德里 + 团队成员 + HR）会从 **5 个维度** 给你打分：

| 维度 | 权重（推测） | 关键问题 | 怎么过 |
| --- | --- | --- | --- |
| **Agent 品味与一手实践** | 30% | "你最近一周用什么 Agent 工具？做出了什么？哪里不爽？" | 必须有真实使用故事，截图+具体感受 |
| **产品系统性思考** | 25% | "如果让你来定 Harness 的北极星指标，你会怎么定？" | 用[文档 13](13-自带作品集-Harness指标体系与实验方案.md) 那套框架 |
| **技术理解深度（够用就行）** | 15% | "解释 KV Cache 在 Agent Loop 里的作用" | 用[文档 03](03-Harness核心知识体系速查.md) 弹药 |
| **跨界协作（尤其与研究员）** | 15% | "你怎么跟一个不愿意做评估的研究员合作？" | STAR 故事 + Anthropic Labs 的协作模式举例 |
| **0→1 执行力与抗压** | 15% | "一周内只能做一件事，你会做哪件？为什么不做另外那些？" | trade-off 训练、不要打太极 |

---

## 五、怎么使用这份包（推荐节奏）

```
T-7 ──► 把 01 / 02 / 03 看完，画一张自己的「岗位 ↔ 我的过往经历」对照表
T-5 ──► 写一遍 12 / 13（这是面试的杀手锏，别拖到最后一天）
T-3 ──► 集中啃 04 / 05 / 06 / 07，每题至少口头讲一遍（录音回放）
T-2 ──► 把 08 反问话术挑出 3 个最适合你的、改成自己语气
T-1 ──► 整体过 10 / 09，准备好装备（电脑/简历/作品集/反问条）
T-0 ──► 面试当天，按 10 的"开场 60 秒"和"卡壳话术"
T+0 ──► 回家立即写 11 的复盘表（不要等过夜）
T+1 ──► 发感谢邮件（11 里有模板）
T+7 ──► 如果没消息，按 11 的"follow-up 话术"礼貌跟进
```

---

## 六、参考资料锚点

主要素材来源（贯穿全包）：

1. **DeepSeek Harness 团队公开报道**（2026-05-20，新浪 / 网易 / AIWW / Digg）
2. **Anthropic 官方 Harness 系列博客**
   - [Effective harnesses for long-running agents](https://www.anthropic.com/engineering/effective-harnesses-for-long-running-agents)（2025-11）
   - [Harness design for long-running application development](https://www.anthropic.com/engineering/harness-design-long-running-apps)（2026-03-24）
   - [Making Claude Code more secure and autonomous with sandboxing](https://www.anthropic.com/engineering/claude-code-sandboxing)（2025-10）
3. **Claude Code 源码级分析**（DuoCode Tech Blog，2026-03-31，~51 万行 TS / 1902 文件 / 40 工具）
4. **同级目录已有的深度分析**：[../ai-agent-harness-engineering-analysis.md](../ai-agent-harness-engineering-analysis.md)（七大组件、横向对比、自研路线图）
5. **DeepSeek 公司文化与面试经验**（搜狐 / 36氪 / 腾讯云开发者）
6. **Anysphere/Cursor PM JD**（参照同类岗位口径）

---

## 七、一句忠告

> **DeepSeek 现在最缺的不是"会写 PRD 的产品经理"，而是"能在模型团队和用户之间，把模糊问题翻译成可衡量信号、再把信号翻译回模型训练目标"的人。**

如果你能用一个具体例子（哪怕是小红书评论区里的一个用户抱怨）演示这条翻译链路——你已经领先 80% 候选人了。

祝顺利。
