# DeepSeek「Agent 全栈开发工程师」面试准备包

> **岗位**：Agent 全栈开发工程师（DeepSeek，内部 RL 训练 + Agent 基础设施方向）
> **本包整理时间**：2026-05-21
> **目标**：把这份材料当作"面试前 7 天集中冲刺手册"——覆盖从理解 JD、技术弹药、手撕代码（前端 + 后端 + 容器）、系统设计、临场策略到复盘跟进的全流程。

---

## 一、这个岗位的"定位"——和已有两包的关系

> DeepSeek Agent / Harness 体系下目前已经在招的 3 个岗位（同一团队网络）：
>
> | 岗位 | 定位 | 准备包 |
> | --- | --- | --- |
> | Agent Harness **产品经理** | 定义产品 + 路线图 + 指标 + 用户研究 | `../interview-prep-deepseek-harness-pm/` |
> | Agent Harness **研发工程师** | 开发桌面端 Code Agent 产品本身 | `../interview-prep-deepseek-harness-engineer/` |
> | **Agent 全栈开发工程师**（本岗） | **支撑 RL 训练 + 算法团队的 infra & 评测平台** | **本目录** |

**核心差异（一句话）**：
- PM = **定义"做什么"**
- Harness Eng = **造给用户用的 Agent 产品**
- **全栈 Eng（本岗）= 造给研究员/算法团队用的 Agent infra + 评测平台 + 调试可视化**

> **这个岗位是"内部研发的研发"**——你的用户不是普通开发者，而是 **DeepSeek 的算法 / 研究员 / RL infra 工程师**。你的产出是 eval platform / trajectory viewer / container service / framework adapter。

详细的三岗对照：[99-三岗位差异对照与选择建议.md](99-三岗位差异对照与选择建议.md)

---

## 二、JD 关键能力一句话总结

> **"全栈" = 前端 (TS/React + 可视化) + 后端 (Python/Rust + API + DB) + Infra (Docker/K8s + CI/CD) + AI (LangChain / OpenAI Agents SDK / MCP 集成 + 训练 pipeline 对接)。**

加上你必须**懂 LLM/Agent 基础原理**（context window / token / tool use / MCP）+ **深度用过 Claude Code / Cursor**。

---

## 三、目录与使用建议

| 序号 | 文件 | 用途 |
| --- | --- | --- |
| 01 | [01-JD深度解读与岗位画像.md](01-JD深度解读与岗位画像.md) | 逐句拆解 + 隐藏信号 + 三岗对照 |
| 02 | [02-DeepSeek与Agent评测平台背景调研.md](02-DeepSeek与Agent评测平台背景调研.md) | DeepSeek + 业界 eval platform / trajectory viewer 调研 |
| 03 | [03-技术栈速查.md](03-技术栈速查.md) | TS/Python/Rust/Docker/K8s/LangChain/OAI SDK/MCP 全栈技术速查 |
| 04 | [04-高频面试题-前端与可视化篇.md](04-高频面试题-前端与可视化篇.md) | React + D3/timeline 实现轨迹查看器手撕题 |
| 05 | [05-高频面试题-后端服务篇.md](05-高频面试题-后端服务篇.md) | FastAPI/axum + DB schema + streaming + 调试 endpoint |
| 06 | [06-高频面试题-容器与运维篇.md](06-高频面试题-容器与运维篇.md) | Dockerfile / Helm / K8s probe / CI/CD / 故障定位 |
| 07 | [07-系统设计-Agent评测平台与轨迹Pipeline.md](07-系统设计-Agent评测平台与轨迹Pipeline.md) | 完整设计 eval platform + trajectory pipeline |
| 08 | [08-系统设计-RL训练集成与Scaffold适配层.md](08-系统设计-RL训练集成与Scaffold适配层.md) | 把外部 framework 接进内部 RL pipeline 的设计 |
| 09 | [09-行为与协作篇.md](09-行为与协作篇.md) | STAR + 和算法 / SRE / 研究员协作 + 文化 |
| 10 | [10-反问环节话术库.md](10-反问环节话术库.md) | 全栈向高质量反问 |
| 11 | [11-面试前-中-后全流程行动手册.md](11-面试前-中-后全流程行动手册.md) | T-7 行动 + 临场策略 + T+0 复盘（三合一） |
| 12 | [12-自带作品集-Mini-Eval-Platform开源项目.md](12-自带作品集-Mini-Eval-Platform开源项目.md) | 一个 minimal eval platform 项目设计 — 工程师面试杀手锏 |
| 99 | [99-三岗位差异对照与选择建议.md](99-三岗位差异对照与选择建议.md) | PM/Harness Eng/全栈 Eng 三岗全景对照 |

---

## 四、整体策略（一句话）

> **"用全栈的硬实力，帮 DeepSeek 的研究员把'看模型在 agent 里怎么跑'这件事变得显然。"**

具体落地三件事：

### 1. 一个能跑的 Mini Eval Platform（最关键的杀手锏）
DeepSeek 这个岗位最稀缺的不是"会写 React"或"会用 K8s"，是 **"你真的为 agent trajectory 做过 viewer 吗？你真的把 LangChain 接到自己的 pipeline 跑过 batch eval 吗？"**

如果你能在 T-7 ~ T-2 这一周把 [12 文档] 的 mini-eval-platform 跑出来：
- 前端 React + timeline 可视化 trajectory
- 后端 FastAPI 拉 trajectory JSON
- 一个 Dockerfile + docker-compose
- 跑一个 toy benchmark（10 个任务，3 个模型）

**这一件事就足以让你和 80% 候选人区分开**——CC/Cursor 用得溜的人多，真做过 eval platform 的极少。

### 2. 3 个关键 framework 能源码级讲
从 LangChain / OpenAI Agents SDK / MCP 中挑 3 个最关键的源码片段（agent executor、tool calling adapter、stdio transport），**能用自己的话讲清"为什么这么设计、有什么 trade-off、如果让我设计会怎么改"**。

### 3. 准备和研究员 + SRE 都聊得起来
这个岗位的特殊在于——你的同事既有研究员（"我希望 trajectory 多记一个 reward signal"），也有 SRE / Infra eng（"你这个容器内存涨了 8G"）。你需要双语言对话。

---

## 五、面试评估维度的内部模型（推测）

| 维度 | 权重（推测） | 关键问题 |
| --- | --- | --- |
| **全栈编码硬功夫** | 30% | "白板写一个 trajectory viewer 时间线组件" / "用 FastAPI 写一个流式 endpoint" |
| **容器 + 运维** | 15% | "写 Dockerfile + K8s deployment + liveness probe" |
| **系统设计** | 20% | "设计一个支持 10K trajectory/天的 eval platform" |
| **LangChain/OAI SDK 集成** | 15% | "把 LangChain agent 接到内部 RL pipeline" |
| **LLM/Agent 原理** | 10% | "context window 怎么管理 / MCP 工程细节" |
| **学习能力 + 跨栈** | 5% | 现场给陌生栈 / 框架 |
| **文化 + 协作** | 5% | 和算法 / SRE 协作 |

---

## 六、推荐节奏（T-7 ~ T+0）

```
T-7  战略 + Mini Eval Platform V0 启动
T-6  技术弹药：14 个核心概念 + LangChain/OAI SDK 源码扫
T-5  前端 + 可视化专项；启动 Mini Platform V1
T-4  后端 + 容器专项；LeetCode 中等 5 题
T-3  系统设计 + Mock Interview
T-2  作品集 + 反问 + 简历精修
T-1  静心 + 装备
T-0  面试当天
T+0  复盘 + 感谢邮件
```

总投入 **40~55 小时**——可分摊到 7 天。

---

## 七、和已有两包重叠时的引用约定

为了避免重复，本包**直接引用**已有 PM / Harness Engineer 包的章节：

| 内容 | 直接引用 |
| --- | --- |
| DeepSeek 公司背景 / Harness 团队时间线 | [PM 包 02](../interview-prep-deepseek-harness-pm/02-DeepSeek与Harness背景调研.md) |
| 14 个 LLM/Agent 核心概念速查 | [PM 包 03](../interview-prep-deepseek-harness-pm/03-Harness核心知识体系速查.md) + [Eng 包 03](../interview-prep-deepseek-harness-engineer/03-Harness技术体系与代码模式速查.md) |
| DeepSeek 文化 / 不发奖金 / 不赛马 | [PM 包 07](../interview-prep-deepseek-harness-pm/07-高频面试题与参考答案-行为与文化篇.md) |
| 通用临场策略（开场 60 秒 / 卡壳话术） | [PM 包 10](../interview-prep-deepseek-harness-pm/10-面试中临场策略与话术.md) + [Eng 包 10](../interview-prep-deepseek-harness-engineer/10-面试中临场策略与话术.md) |
| 通用复盘模板 | [Eng 包 11](../interview-prep-deepseek-harness-engineer/11-面试后复盘模板与跟进话术.md) |

本包只写**全栈岗特化**内容——评测平台、轨迹可视化、容器运维、框架集成、RL pipeline 对接。

---

## 八、一句忠告

> **DeepSeek 这个岗位想要的不是"通才"，而是"能让 50 个算法 / 研究员每天用得顺、debug 得起来"的工具人。这是高 leverage 的岗位——你的产出 × 50 个用户 = 整个 DeepSeek agent 训练效率的乘数。**

> **会用 Cursor 的人很多，能造出"Cursor for Researcher's Eval Workflow" 的人极少。这就是你要在面试前一周训成的能力。**

祝顺利。
