# 02 · DeepSeek C 端产品与 AI Chat 赛道调研

> DeepSeek 公司背景请先读：[../interview-prep-deepseek-harness-pm/02-DeepSeek与Harness背景调研.md](../interview-prep-deepseek-harness-pm/02-DeepSeek与Harness背景调研.md)。
>
> 本文档**只列 C 端 PM 岗特化**：DeepSeek 现有 C 端产品现状 + 国内 4 款 + 海外 4 款主流 AI Chat 竞品的速览 + 行业关键数据。
>
> **使用提示**：以下数据是基于公开信息汇总，**面试前务必通过 SimilarWeb / QuestMobile / 七麦数据 / 量子位 等 fresh source 核实**——C 端 PM 引用过时数据是大忌。

---

## 一、DeepSeek 现有 C 端产品矩阵

### 1.1 产品分布（推测）

| 端 | 产品 | 主要功能 |
| --- | --- | --- |
| Web | chat.deepseek.com | 通用对话 + R1 reasoning + 联网搜索 + 文件分析 |
| iOS / Android | DeepSeek App | 同上，移动端适配 |
| API | platform.deepseek.com | 给开发者用 |
| (推测) Desktop | DeepSeek Code Agent | Harness 团队在做 |

**本岗位主要聚焦**：Web + 移动 App，可能含付费版（如有）。

### 1.2 公开关键数字（截止 2026 上半年范围）

> ⚠️ 这些数字是合理推测/历史公开数据，**面试前请通过 SimilarWeb 等核对最新**。

| 维度 | 数据 | 注 |
| --- | --- | --- |
| Web 月访问量 | 数亿~十亿级（2025 R1 发布后曾入全球前 10 网站）| 流量爆发期 |
| App 下载量 | 国内多个应用商店登顶过 | App Store Top 1 历史 |
| MAU | 国内 1 亿 + | 含 Web + App |
| 主用户分布 | 国内 60-70% + 海外 30-40% | 海外份额高于其他国内厂商 |
| 用户构成 | 程序员 + 学生 + 知识工作者 + 普通好奇用户 | 偏 power user |
| 主要场景 | 写作 / 编程 / 思考辅助 / 数学 / 搜索替代 | 偏理性场景 |

### 1.3 DeepSeek C 端的"独特资产"

1. **R1 reasoning 模型**——开源 + 性能 ≈ o1，是 C 端的核心差异化
2. **MIT 开源** + 透明 paper——技术品牌强
3. **价格优势**（API 端）——但 C 端目前免费
4. **不投流文化**——天然降低 user acquisition cost
5. **海外认知度**——国内厂商中最高

### 1.4 DeepSeek C 端的"已知瓶颈"

1. **App 设计相对简洁**——多模态 / 语音 / 实时性弱于豆包
2. **没有原生 image generation**——靠模型本身回复
3. **没有 Agent / Tool use 在 C 端**——主要是纯 Chat
4. **国际化 / 多语言 UX**——海外用户量大但 UI 中文优先
5. **付费 / 商业化 path**——目前免费，未来如何 monetize？

---

## 二、海外主流 AI Chat 竞品（4 款）

### 2.1 ChatGPT (OpenAI)

| 维度 | 内容 |
| --- | --- |
| 推出 | 2022-11 |
| MAU | 5 亿+（2025 数据，全球最大）|
| 付费 | Plus $20/月、Pro $200/月、Enterprise |
| 核心差异 | 多模态最完整：voice / image / video / code interpreter / browse / GPTs / canvas / advanced voice |
| Reasoning 模型 | o1 / o3 系列 |
| 独特创新 | GPT Store + Canvas + Advanced Voice + Operator (浏览器 agent) |
| 主用户场景 | 通用任务 + 编程 + 学习 + creative work |
| 重要 UX | 对话 thread 管理强、Voice mode 转化高 |
| Take-away | **完整度 + 创新速度 + 多模态** = world standard |

**给 DeepSeek 的启示**：
- Canvas / 编辑视图——长文写作场景必备
- Voice mode——移动端打开率提升 keystone
- Custom GPT——长尾场景的 holder

### 2.2 Claude (Anthropic)

| 维度 | 内容 |
| --- | --- |
| 推出 | 2023-03 |
| MAU | 1 亿+ 估算 |
| 付费 | Pro $20/月、Max $100/月、Team |
| 核心差异 | 写作质量 + reasoning + Coding 强、Artifacts (canvas)、Projects（长 context 项目）、MCP 接入 |
| Reasoning 模型 | Claude 4.x thinking |
| 独特创新 | Artifacts（侧边栏直接 render React/HTML）、Projects（10M token 持久记忆）、Computer Use |
| 主用户场景 | 高质量写作、复杂 reasoning、Coding（开发者最爱）|
| 重要 UX | Token-aware 上下文管理、Artifact 切分清晰 |
| Take-away | **质量 + 写作体验 + Artifacts** 是产品级护城河 |

**给 DeepSeek 的启示**：
- Artifacts 这种"右侧边栏 render"——长输出体验关键
- Projects 持久 context——重度用户 retention 神器
- Computer Use——下一代 agent 入口

### 2.3 Gemini (Google)

| 维度 | 内容 |
| --- | --- |
| 推出 | Bard 2023-03 → Gemini 2024-02 |
| MAU | 难统计（捆绑 Google 服务）|
| 核心差异 | 集成 Google 搜索 + Workspace + Android 默认助手；多模态 + 长 context (1M tokens) |
| 独特创新 | Deep Research、NotebookLM (audio 笔记)、Veo (视频生成) |
| 主用户场景 | 工作流（Gmail/Docs/Sheets）+ Research + 视频 |
| Take-away | **生态整合 + 长 context + 多模态** = 体系优势 |

**给 DeepSeek 的启示**：
- Deep Research mode——专业用户高价值场景
- NotebookLM 模式——可以差异化 niche

### 2.4 Perplexity

| 维度 | 内容 |
| --- | --- |
| 推出 | 2022 |
| MAU | 8000 万+ 估算 |
| 付费 | Pro $20/月 |
| 核心差异 | **搜索原生**——每个回答附 citation、Pro Search (multi-step research)、Spaces (项目)|
| Take-away | **垂直定位** + **可信度** = 抢搜索市场 |

**给 DeepSeek 的启示**：
- Citation 强化——可信度直接提升使用深度
- Pro Search 模式——可作为 R1 thinking 的产品包装

---

## 三、国内主流 AI Chat 竞品（4 款 + 拓展）

### 3.1 豆包 (字节)

| 维度 | 内容 |
| --- | --- |
| 推出 | 2023-08 |
| MAU | **国内 No.1**，几亿级（2025 数据）|
| 核心差异 | **运营 + 投流 + 多模态 + AI 视频** + 全场景覆盖（智能体、播客、视频通话）|
| 独特创新 | 智能体 (AI 角色 100K+)、视频通话、AI 播客、桌面客户端深度集成（剪映等）|
| 主用户场景 | C 端娱乐 + 学生 + 创作者 |
| 重要 UX | 颜值即正义；移动端体验极致；语音深度 |
| Take-away | **运营驱动 + 产品矩阵** = 字节式增长 |

**给 DeepSeek 的启示**：
- 不学其投流（不符 DeepSeek 文化）
- 学其语音通话体验——移动端 retention 大杀器
- 学其智能体——降低长尾场景门槛

### 3.2 Kimi (月之暗面)

| 维度 | 内容 |
| --- | --- |
| 推出 | 2023-10 |
| MAU | ~3000 万级 |
| 核心差异 | **长 context** + 学术 / 工作场景 + Kimi+ (智能体) |
| 独特创新 | 200 万字超长 context、深度搜索、Kimi 探索版（reasoning） |
| 主用户场景 | 知识工作者 + 学生 + 研究 |
| Take-away | **垂直深耕长 context + 知识场景** |

**给 DeepSeek 的启示**：
- Kimi 用户和 DeepSeek 用户高度重叠（理性、知识工作者）
- 长 context 是 retention 利器——DeepSeek 也可强化
- "探索版"产品包装 reasoning 模型——可参考

### 3.3 腾讯元宝

| 维度 | 内容 |
| --- | --- |
| 推出 | 2024-05 |
| MAU | ~5000 万级 |
| 核心差异 | **接入 DeepSeek + 混元双模型** + 微信 / QQ 生态 |
| 独特创新 | 接 DeepSeek R1 后流量暴涨；多模型切换；社交分享路径 |
| 主用户场景 | 普通用户 + 微信导流 |
| Take-away | **生态分发 + 双模型策略**|

**给 DeepSeek 的启示**：
- DeepSeek 自家也要做好这个体验——别让元宝抢走流量
- 思考 DeepSeek 是否需要 social distribution channel

### 3.4 智谱清言 / 通义 / 海螺 / 文心一言（任选 1）

| 产品 | 核心差异 |
| --- | --- |
| **智谱清言**（智谱）| GLM 系列、AutoGLM agent、企业 + 学术 |
| **通义**（阿里）| 通义万相图像 + 通义听悟语音 + 通义灵码代码 |
| **海螺**（MiniMax）| 视频 + 语音（最强 voice）+ Talkie 社交 |
| **文心一言**（百度）| 搜索整合 + ERNIE bot + 国内 PR |

---

## 四、行业关键数据 + 趋势（2026 视角）

### 4.1 全球 AI Chat MAU 排名（推测）

```
1. ChatGPT      5 亿+
2. 豆包         3 亿+
3. DeepSeek     1.5 亿+
4. Claude       1 亿+
5. Gemini       不可统计（捆绑）
6. Kimi         3000 万+
7. 腾讯元宝     5000 万+
8. Perplexity   8000 万+
```

### 4.2 关键行业 trend

1. **Reasoning model 普及**：o1 → R1 → Claude 4.x thinking → 国产全跟上；C 端产品需要 "怎么把 thinking 展示给用户" 答案
2. **多模态加速**：voice / video / image 成标配；DeepSeek 这块是短板
3. **Agent 萌芽**：Computer Use / Operator / Manus / Devin；C 端 agent 还没成熟范式
4. **付费转化分化**：海外 ChatGPT Plus 已 1500 万付费；国内付费习惯弱
5. **垂直 AI 抢通用 AI**：教育 (Cursor Edu/Photomath)、医疗 (Hippo)、法律 (Harvey) 抢通用 AI 市场
6. **手机系统级集成**：Apple Intelligence / Google Gemini in Android / 华为小艺；通用 Chat 受冲击

### 4.3 国内市场独特性

1. **不付费习惯**：用户付费 willingness < 海外，更靠广告 / 增值服务
2. **微信生态主导**：流量入口高度集中
3. **多模型混用**：用户同时用 3~5 个，无锁定
4. **强运营文化**：豆包式投流是国内常态——DeepSeek 是异类
5. **政策合规**：内容审核 + 备案是 default requirement

---

## 五、DeepSeek C 端 vs 8 款竞品差异表（必背）

| 维度 | DeepSeek | ChatGPT | Claude | Gemini | 豆包 | Kimi | 元宝 | Perplexity |
| --- | --- | --- | --- | --- | --- | --- | --- | --- |
| **核心模型** | V3 / R1 | GPT-4.x / o-series | Claude 4.x | Gemini 2.x | 豆包 (Doubao) | Kimi (Moonshot) | 双模型 | 多模型 |
| **Reasoning** | ✅ R1 强 | ✅ o1/o3 | ✅ thinking | ✅ thinking | ⚠️ 弱 | ✅ 探索版 | ✅ R1 | ⚠️ |
| **Voice** | ❌ | ✅✅ | ⚠️ | ✅ | ✅✅ | ⚠️ | ⚠️ | ⚠️ |
| **Image gen** | ❌ | ✅✅ | ⚠️ | ✅✅ | ✅✅ | ⚠️ | ✅ | ❌ |
| **Video gen** | ❌ | ✅ Sora | ❌ | ✅ Veo | ✅ | ❌ | ❌ | ❌ |
| **Canvas / Artifact** | ❌ | ✅ | ✅✅ | ⚠️ | ⚠️ | ⚠️ | ⚠️ | ❌ |
| **长 context** | ⚠️ 64K | ✅ 128K+ | ✅ 200K | ✅ 1M+ | ⚠️ | ✅✅ 2M | ⚠️ | ⚠️ |
| **搜索 + Citation** | ✅ | ✅ | ⚠️ | ✅✅ | ✅ | ✅ | ✅ | ✅✅ |
| **Agent / Tool** | ⚠️ | ✅ Operator | ✅ Computer Use | ⚠️ | ✅ 智能体 | ⚠️ | ⚠️ | ⚠️ |
| **付费** | 免费 | $20-200 | $20-100 | 部分免费 | 免费 | 部分付费 | 免费 | $20 |
| **价值主张** | **模型强 + 开源透明** | 大而全 | 写作 + Code | Google 整合 | 全场景全平台 | 长 context | 微信入口 | 搜索 |

> **DeepSeek 的"短板组合"**：多模态 / Voice / Image / Video / Canvas / 长 context 都不够。
> **DeepSeek 的"独家王牌"**：开源 + reasoning + 透明品牌。

---

## 六、可被面试问的"行业宏观问题"

### Q·"C 端 AI Chat 这个市场最终会几家分？"

> "我倾向 3~4 家——国内 ChatGPT-like 不会一家通吃，因为：(1) 模型同质化加速；(2) 应用场景分化（豆包娱乐型、Kimi 知识型、DeepSeek 理性型）；(3) 入口分散（手机厂商内置、社交平台、纯独立 App）。但 DeepSeek 必入前 3，因为开源 + reasoning + 海外渗透 = 三大壁垒。"

### Q·"AI Chat 的盈利模式终局？"

> "三条路：付费订阅（ChatGPT Plus 模型，海外可行国内难）、API 商业化（B 端为 C 端补血）、广告 + 增值服务（国内豆包式）。DeepSeek 由于开源 + 低成本基因，最可能走 API + 适度增值订阅，不会重广告。"

### Q·"如果手机系统级 AI 助手普及，独立 App 还需要吗？"

> "需要——但定位会变 niche。系统级解决日常 query (≈ 70% 长尾)；独立 App 解决专业 / 深度 / 长 horizon task。DeepSeek 这种 reasoning 强的，恰好属于 niche but high-value。类比：Google 搜索普及了，但搜专业知识仍用 Stack Overflow / GitHub。"

---

## 七、推荐手动深度体验流程（T-7 ~ T-3）

每款产品至少 **2~3 小时** 真用，覆盖 5 个场景：

1. **冷启动**：注册 / 首次进入，看 onboarding
2. **基础对话**：闲聊 / 提问
3. **复杂任务**：让它写一篇报告 / 解一个数学题 / 翻译长文
4. **多模态**：语音 / 图像（如有）
5. **小众功能**：找一个被埋藏的小功能体验

**每个产品记录**：
- 3 个最让我"哇"的点
- 3 个最让我"卡"的点
- 1 个我想偷给 DeepSeek 的点

---

## 八、推荐资料

### 数据源
- SimilarWeb（Web traffic）
- 七麦数据 / Qimai（App Store / 应用市场）
- QuestMobile（国内 App MAU）
- a16z 季度榜单（"Top 100 GenAI consumer apps"）

### 行业 newsletter / 自媒体
- 量子位 / 机器之心 / AI 科技评论 / 硅星人（中文）
- Stratechery / The Information / TechCrunch（英文）
- Lenny's Newsletter（产品 PM 视角）

### 必读 blog
- 张鹏《AI 应用商业化报告》系列
- 周鸿祎 / 朱啸虎 等中国 AI 投资人观点
- Sam Altman / Dario Amodei 公开访谈

---

## 九、一句应试钥匙

> **C 端 PM 答题不是"我读过竞品分析"——是 "我连续 3 周深度用过 8 款 AI Chat，发现 [极具体] 体验差异，思考 DeepSeek 应不应该抄 / 抄哪个 / 怎么差异化"。**

> 这种 first-hand 体感，**永远不能被读 PPT 替代**。
