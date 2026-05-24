# 03 · Harness 技术体系与代码模式速查（工程师深度版）

> JD 任职要求第 5 条原文："**熟悉** LLM 以及 Agent 基本机制及其技术原理，包括 LLM API、KV Cache、Agent Loop、Tool Use、Reasoning、Planning、Skills、MCP、Memory、Subagent、Multi-Agent 等相关知识。对 Prompt Engineering、Context Engineering、Harness Engineering 等课题有**较深入的了解**。"
>
> 注意：PM 版的对应文档（[../interview-prep-deepseek-harness-pm/03-Harness核心知识体系速查.md](../interview-prep-deepseek-harness-pm/03-Harness核心知识体系速查.md)）已经覆盖了**概念**层面的速查。
>
> 本文档是**工程师深度版**——每个概念配 **代码片段 / 数据结构 / API spec / 工程陷阱 / 源码位置**。

---

## 一、Agent Loop · 代码模式

### 1.1 最小可运行实现（Anthropic Messages API）

```python
from anthropic import Anthropic

client = Anthropic()

TOOL_SCHEMAS = [
    {
        "name": "read_file",
        "description": "Read file content",
        "input_schema": {
            "type": "object",
            "properties": {"path": {"type": "string"}},
            "required": ["path"],
        },
    },
    # ...更多 tool
]

TOOLS = {
    "read_file": lambda path: open(path).read(),
    # ...
}

def run_agent(user_input: str, max_turns: int = 30):
    messages = [{"role": "user", "content": user_input}]

    for turn in range(max_turns):
        response = client.messages.create(
            model="claude-opus-4-5",
            max_tokens=4096,
            system=SYSTEM_PROMPT,
            tools=TOOL_SCHEMAS,
            messages=messages,
        )

        messages.append({"role": "assistant", "content": response.content})

        if response.stop_reason == "end_turn":
            return messages
        if response.stop_reason != "tool_use":
            raise RuntimeError(f"Unexpected stop: {response.stop_reason}")

        tool_results = []
        for block in response.content:
            if block.type == "tool_use":
                try:
                    result = TOOLS[block.name](**block.input)
                except Exception as e:
                    result = f"ERROR: {type(e).__name__}: {e}"
                tool_results.append({
                    "type": "tool_result",
                    "tool_use_id": block.id,
                    "content": str(result),
                })
        messages.append({"role": "user", "content": tool_results})

    raise RuntimeError("Exceeded max_turns")
```

### 1.2 工业级要素清单（白板加分项）

```
[ ] 流式输出（SSE / streaming）
[ ] 中断点（AbortController / signal）
[ ] Budget 控制（max_turns / max_tokens / max_budget_usd）
[ ] Diminishing-returns detector（连续 N 轮 token 增量 < 阈值就停）
[ ] Concurrency-safe 并发 tool 调度
[ ] Tool error 自动喂回（不是抛异常）
[ ] Schema validation（Zod / Pydantic）
[ ] Tool timeout
[ ] State checkpoint（可中断恢复）
[ ] Lifecycle hooks（PreToolUse / PostToolUse / PreCompact）
```

### 1.3 工程陷阱

| 陷阱 | 表现 | 修复 |
| --- | --- | --- |
| Tool 错误吞掉 | agent 不知道 tool 失败 | 把 error 当 tool_result 注入下一轮 |
| 不存 assistant content | 下一轮模型不知道自己说了什么 | 整段塞回 messages |
| messages 序列损坏 | API 拒绝（"tool_use without tool_result"） | 严格按 user/assistant/user/... 交替 |
| 并发 tool 抢资源 | write 同一文件 race | concurrency-safe 标记 + 分批 |
| 长输出爆 context | model 直接 stop_reason=max_tokens | maxResultSize + 持久化到 file + 返回 path |

### 1.4 源码参考

- [liteagent](https://github.com/drchrislevy/liteagent) ~200 行 Python，看 `agent.py`
- [simple-agent-loop](https://pypi.org/project/simple-agent-loop/) ~200 行
- [tinyAgent-TS](https://github.com/alchemiststudiosDOTai/tinyagent-ts) TS 版
- Claude Code 反编译版（社区有，搜 "claude-code source"）

---

## 二、Tool Use · 代码模式

### 2.1 Tool 定义结构（Claude Code 风格）

```typescript
import { z } from "zod";

interface ToolDef<I, O> {
  name: string;
  description: (ctx: ToolContext) => Promise<string>;
  prompt: (ctx: ToolContext) => Promise<string>;
  inputSchema: z.ZodType<I>;
  outputSchema: z.ZodType<O>;
  call: (input: I, ctx: ToolContext) => Promise<O>;
  checkPermissions: (input: I, ctx: ToolContext) => Promise<PermissionDecision>;
  validateInput: (raw: unknown) => I;
  userFacingName: string;
  isConcurrencySafe: boolean;
  isReadOnly: boolean;
  shouldDefer?: boolean;
  maxResultSizeChars?: number;
  renderToolUseMessage: (input: I) => React.ReactNode;
  renderToolResultMessage: (output: O) => React.ReactNode;
}

type PermissionDecision =
  | { behavior: "allow"; reason?: string }
  | { behavior: "ask"; reason: string }
  | { behavior: "deny"; reason: string }
  | { behavior: "passthrough" };
```

### 2.2 Tool 执行 8 步生命周期（背诵）

```
1. 模型 emit tool_use block
   ↓
2. validateInput（Zod schema）
   ↓
3. PreToolUse hooks fire
   ├── 可返回 permissionDecision（allow/deny/ask）
   ├── 可改 input via updatedInput
   ├── 可注入 additionalContext
   └── 可直接 block
   ↓
4. checkPermissions（整合 hook + tool 自身决定）
   ↓
5. 如果 behavior=ask → UI permission dialog
   ↓
6. tool.call(input, ctx)
   ↓
7. PostToolUse hooks fire（cleanup / modify output）
   ↓
8. 映射成 ToolResultBlockParam，超出 maxResultSize 落盘 + 返回 path
```

**铁律**：deny rule 永远 > allow（hook 即使返回 allow，命中 deny 仍 deny）。

### 2.3 OpenAI / Anthropic / Gemini Tool Use 协议差异

| 维度 | Anthropic | OpenAI | Gemini |
| --- | --- | --- | --- |
| 块名 | `tool_use` | `tool_calls` | `functionCall` |
| 输入字段 | `input` (JSON obj) | `arguments` (JSON **string**) | `args` |
| 工具结果 block | `tool_result` with `tool_use_id` | `tool` role with `tool_call_id` | `functionResponse` |
| 并行 tool call | 同消息多 block 默认支持 | 同消息多 calls 支持 | 视 model 而定 |
| 流式 chunk | `input_json_delta` 增量 JSON | delta chunks | 增量 |

**Harness 跨模型适配的 idiomatic pattern**：抽象中间表示（`ToolCall { id, name, input: object }`）+ 三个 adapter。

### 2.4 工程陷阱

- **OpenAI 返回的 `arguments` 是 JSON 字符串**，不是对象——需要 `JSON.parse` 且容错（流式中 chunk 可能不完整）
- **Anthropic 的 stream 在 `tool_use` 期间分两步：先 `tool_use` 块开始 + `input_json_delta`** —— 不能一上来就 parse，要先 accumulate
- **schema 演进**：tool spec 改了，旧 trajectory 重放会失败——需要版本号

---

## 三、KV Cache / Prompt Cache · 工程细节

### 3.1 物理基础

- Transformer 每层 attention 算 K、V matrices；前缀不变就可复用
- Anthropic / OpenAI 把它**商业化**成 prompt cache：折扣 50%~90%（视提供方）
- Anthropic 显式 `cache_control` 标记：

```python
messages = [
    {
        "role": "user",
        "content": [
            {
                "type": "text",
                "text": LONG_SYSTEM_PROMPT,
                "cache_control": {"type": "ephemeral"},  # 标记为可缓存
            },
            {"type": "text", "text": user_dynamic_input},
        ],
    }
]
```

### 3.2 Harness 设计的 cache-friendly 三原则

1. **System prompt 切两段**：稳定段（角色 / tool spec / 规则）放前面 + cache_control；动态段（git status / 时间）后面
2. **Tool result 大块靠后**：变化部分都尾追，前缀稳定
3. **Compaction 时机和 cache 边界对齐**：每次 compact 后立即建立新 breakpoint

### 3.3 数字感

| 场景 | 不用 cache | 用 cache |
| --- | --- | --- |
| 30 turn long session | $5 | ~$0.5（约 90% off） |
| First call cache write | 1.25× input price | — |
| Cache hit | 1× input | 0.1× input |

> **面试金句**："Cache 是 Harness 真正能省钱的地方，比换模型省得多。"

### 3.4 工程陷阱

- **cache breakpoint 必须放在稳定 boundary**——动态部分前的最后位置
- **system prompt 顺序错乱**——会导致每次 cache miss（典型：插入时间戳到顶部）
- **多 cache_control 块的限制**：Anthropic 最多 4 个 cache breakpoint，必须规划

---

## 四、Reasoning / Thinking Block · 工程接口

### 4.1 Anthropic 接口

```python
response = client.messages.create(
    model="claude-opus-4-5",
    thinking={"type": "enabled", "budget_tokens": 10000},  # 显式开启 + 预算
    messages=[...],
)

# response.content 现在会包含 thinking block:
# [
#   {"type": "thinking", "thinking": "Let me analyze..."},
#   {"type": "text", "text": "Based on my analysis..."}
# ]
```

### 4.2 OpenAI o1 / o3 接口

- 自动开 reasoning，**不暴露 thinking trace**
- 计费走 `reasoning_tokens`

### 4.3 DeepSeek-R 系列接口

- 暴露 thinking trace（早期版本可见）
- 工程上需注意：thinking trace **走 KV cache 但不计入 messages history**——拿到 response 时通常 thinking 不要回灌

### 4.4 Harness 设计的 reasoning 策略

| 决策点 | 建议 |
| --- | --- |
| 全局开关 | 给用户暴露"快/平衡/深思"三档 |
| 自动判断 | 简单 tool call (read_file) 关 reasoning；复杂规划开 |
| Subagent 分配 | planner 开 reasoning，generator 关 |
| Budget | 长 agent loop 中 cap reasoning_tokens，防失控 |
| UI | "model is thinking" affordance（防用户以为卡死）|

---

## 五、Context Engineering · 工程细节

### 5.1 三层 context pipeline（Claude Code 范式）

```typescript
async function buildContext(): Promise<MessagesArray> {
  const [userCtx, systemCtx, promptParts] = await Promise.all([
    getUserContext(),    // memoized: CLAUDE.md + 用户偏好
    getSystemContext(),  // memoized: git status (≤2KB) + cache breakers
    fetchSystemPromptParts(),  // parallel fetch sections
  ]);

  return composeMessages({
    sections: [
      ...promptParts,  // 走 cache
      userCtx,
      systemCtx,
      ...history,
    ],
  });
}
```

### 5.2 4 阶段 compression（按顺序执行）

```
Phase 1·snip compacting
  └── 移除被后续消息 supersede 的旧消息（如：read_file 重复读同一文件）

Phase 2·micro-compact
  └── 单个 assistant response 内部去重 / 截断长输出

Phase 3·context collapse
  └── 多个相关 tool_result 折叠成 summary（如：连续 5 次 grep 结果）

Phase 4·auto-compact trigger
  └── 接近 budget 阈值时触发整体压缩（生成 summary + truncate history）
```

### 5.3 Context Reset vs Compaction

| 维度 | Compaction（in-place） | Reset（new agent + handoff） |
| --- | --- | --- |
| 上下文 | summary in place | 完全清空 |
| Context anxiety | 仍可能 | 完全消除 |
| State 传递 | messages 自动保留 | 显式写 file artifact |
| 适用 | 短任务 / 强模型 | 长任务 / Sonnet 4.5 这种 anxiety-prone |
| 实现复杂度 | 低 | 高（需要 handoff schema） |

**Anthropic 11 月文章**：long-running 任务**必须用 reset**。

### 5.4 工程陷阱

- **Compaction 把 cache 也丢了** → compact 后立刻建新 cache breakpoint
- **Summary 丢失关键 ID**（tool_use_id / file path） → summary template 要白名单这些字段
- **Reset 后第一个 agent 不知道做什么** → handoff file 必须包含"接下来要做的下一步"

---

## 六、Memory · 工程细节

### 6.1 三层 memory 模型

| 层 | 范围 | 实现 | 例子 |
| --- | --- | --- | --- |
| 短期 | 当前 context window | messages array | 用户上一轮说什么 |
| 中期 | 当前 session | file artifact / SQLite | 任务进度、handoff |
| 长期 | 跨 session | CLAUDE.md / 向量库 / RAG | 用户偏好、项目背景 |

### 6.2 实现陷阱

- **memory 注入位置**：CLAUDE.md 注入到 system prompt（cache friendly）；用户偏好放 user message 头部
- **memory 大小控制**：Claude Code 给 MEMORY.md 设上限 200 行 / 25KB——超出会冲走 cache
- **memory 来源去重**：多 source memory 容易冲突，需要 priority + merge 规则

---

## 七、Subagent · 实现模式

### 7.1 Task tool 范式（Claude Code）

```typescript
const TaskTool: ToolDef<TaskInput, TaskOutput> = {
  name: "task",
  description: "Launch a sub-agent in isolated context",
  inputSchema: z.object({
    description: z.string(),
    prompt: z.string(),
    subagent_type: z.enum(["explore", "shell", ...]),
  }),
  async call(input, ctx) {
    // 创建新 QueryEngine 实例
    const subEngine = new QueryEngine({
      model: chooseModelFor(input.subagent_type),
      maxTurns: 30,
      maxBudgetUsd: 1.0,
      tools: filterToolsFor(input.subagent_type),
      parentRequestId: ctx.requestId,
    });

    let summary = "";
    for await (const event of subEngine.run(input.prompt)) {
      if (event.type === "text") summary += event.content;
      // subagent 的中间 stream 不返回父 agent，只返回最终 summary
    }

    return { summary };
  },
  isConcurrencySafe: false,  // 子 agent 占资源不能并发太多
};
```

### 7.2 设计要点

- **Context 隔离**：子 agent 用自己的 messages，**不共享父的 context**
- **Tool 子集**：父 agent 决定子 agent 能用哪些 tool（如 explore subagent 只能 read）
- **预算独立**：每个 subagent 自己 cap，防止递归爆炸
- **Summary 汇报**：子 agent 跑完只返回 summary，不返回完整 trajectory
- **深度限制**：默认 2 层（父→子），防止递归（subagent 调 subagent 调 subagent...）

### 7.3 工程陷阱

- **Subagent 不会省 token**（甚至更费），但能防失焦
- **Stream 设计**：父侧需要决定"显示 / 不显示"子 agent 的进度
- **Cancellation 传播**：父 abort 后子也要 abort

---

## 八、Multi-Agent · 模式速记

| 模式 | 角色 | 通信 | 典型 |
| --- | --- | --- | --- |
| Controller-Worker | 主控 + N 个 worker | 主控分配 / 聚合 | Claude Code Task |
| Planner-Executor | Planner（只读）+ Executor（写） | Plan artifact | Cursor Composer |
| GAN-style | Generator + Evaluator (+Planner) | 文件 communication | Anthropic 2026-03 三 Agent |
| Pipeline | A → B → C 流水线 | 串行 message | docs gen workflow |

> **DeepSeek Harness v0.1 建议**：先做 Subagent；v0.2 上 GAN-style；Pipeline 视场景需要

---

## 九、MCP · 协议工程细节

### 9.1 MCP 传输

- **stdio** transport（最常见）：MCP server 启 child process，通过 stdin/stdout JSON-RPC
- **SSE / HTTP** transport：远程 MCP server

### 9.2 MCP 消息格式（JSON-RPC 2.0）

```json
// Client → Server: 列出 tools
{"jsonrpc":"2.0","id":1,"method":"tools/list","params":{}}

// Server → Client
{"jsonrpc":"2.0","id":1,"result":{"tools":[{"name":"...","inputSchema":{...}}]}}

// Client → Server: 调 tool
{"jsonrpc":"2.0","id":2,"method":"tools/call","params":{"name":"...","arguments":{...}}}

// Server → Client
{"jsonrpc":"2.0","id":2,"result":{"content":[{"type":"text","text":"..."}]}}
```

### 9.3 Harness MCP client 实现要点

- 子进程 supervisor（崩了重启 + backoff）
- 协议版本协商（capabilities）
- 工具 schema 解析为内部 ToolDef
- Tool call timeout（默认 30s，可配置）
- 内存 leak 防范（长进程 child process 累积）

### 9.4 工程陷阱

- **MCP server 用户机器装环境**：node / python / docker 不同，需要 graceful degradation
- **Schema 不规范的 MCP server**：第三方 server 经常返回乱七八糟的格式，client 要宽容
- **Stdio buffer flush**：JSON-RPC 一定要 line-based + flush，否则死锁
- **Permission**：每个 MCP server 的 tool 都要单独配 permission（不能整 server 一刀切）

---

## 十、Sandbox · 跨平台细节

### 10.1 OS-level sandbox 三平台

| 平台 | API | 隔离能力 |
| --- | --- | --- |
| **macOS** | [sandbox_init(3)](https://developer.apple.com/documentation/security/sandbox_init) + Seatbelt profile | FS / 网络 / IPC |
| **Linux** | [landlock(7)](https://landlock.io/) + seccomp + cgroups | FS / syscall / 资源 |
| **Linux 替代** | [bubblewrap](https://github.com/containers/bubblewrap) | namespaces |
| **Windows** | [AppContainer](https://docs.microsoft.com/en-us/windows/win32/secauthz/appcontainer-isolation) + Job Object | FS / 网络 / 权限 |

### 10.2 抽象层设计

```typescript
interface Sandbox {
  start(): Promise<void>;
  exec(cmd: string, opts: ExecOptions): Promise<ExecResult>;
  allowPath(path: string, mode: "ro"|"rw"): Promise<void>;
  allowNetwork(host: string, port: number): Promise<void>;
  stop(): Promise<void>;
}

class MacOSSandbox implements Sandbox { /* sandbox_init */ }
class LinuxLandlockSandbox implements Sandbox { /* landlock */ }
class WindowsACSandbox implements Sandbox { /* AppContainer */ }
```

### 10.3 工程陷阱

- **GUI 应用 sandbox 后无法访问剪贴板 / 显示器**——需要 IPC 桥
- **Sandbox 失败必须 fail-closed**（默认拒绝，不是默认放行）
- **Network 完全 block 还是允许 whitelist** —— 默认 block + 显式 allow
- **临时文件**：sandbox 自带 tmpfs / 限定 path

---

## 十一、Skills · 实现模式

### 十一.1 文件结构（OpenCode 风格）

```
~/.config/harness/skills/
├── frontend-design/
│   ├── SKILL.md              # 描述 + 触发条件
│   ├── examples/             # few-shot
│   └── tools/                # 推荐 tool 子集
└── api-design/
    ├── SKILL.md
    └── ...
```

### 11.2 SKILL.md schema

```markdown
---
name: frontend-design
description: Best practices for frontend design (typography, layout, color)
triggers:
  - "design"
  - "frontend"
  - "UI"
tools:
  - playwright
  - file_read
---

# Frontend Design Skill

When designing frontends, follow these principles:
1. ...
2. ...

## Examples

[few-shot 示范]
```

### 11.3 加载机制

```typescript
async function loadSkills(dir: string): Promise<Skill[]> {
  const skills = [];
  for (const entry of await readdir(dir)) {
    const skillPath = path.join(dir, entry, "SKILL.md");
    if (await fileExists(skillPath)) {
      const raw = await readFile(skillPath);
      const { frontmatter, body } = parseMd(raw);
      skills.push({
        name: frontmatter.name,
        description: frontmatter.description,
        triggers: frontmatter.triggers,
        tools: frontmatter.tools,
        prompt: body,
      });
    }
  }
  return skills;
}

// 触发：每次 user message 来时，匹配 trigger 决定是否注入 skill prompt
```

### 11.4 Skills vs Subagent vs Tools

| | Skills | Subagent | Tools |
| --- | --- | --- | --- |
| 本质 | prompt + tool subset bundle | 独立 context 执行单元 | 单个能力 |
| 触发 | user 意图匹配自动注入 | 父 agent 主动 spawn | model 主动 call |
| 共享 context | 是（注入主 agent context） | 否 | 是 |
| 状态 | 无 | 独立 messages | 无 |

---

## 十二、Lifecycle Hooks · 实现接口

### 12.1 8 个最小可用 hook（Claude Code 12 个的精简版）

```typescript
type HookEvent =
  | "SessionStart" | "SessionEnd"
  | "UserSubmit" | "AgentResponse"
  | "PreToolUse" | "PostToolUse"
  | "PreCompact"
  | "Error";

type HookHandler<T> = (data: T) => Promise<HookResult>;

type HookResult = {
  decision: "allow" | "block" | "modify";
  modified?: any;
  additionalContext?: string;
};
```

### 12.2 Hook 类型

- **Shell command hook**：exec 一个 shell 脚本，stdin 传 JSON，exit_code 决定（0=allow / 2=block / 其他=warn）
- **HTTP hook**：POST 到 user 配置的 endpoint，response 决定
- **In-process hook**：直接 JS/TS 函数（最快，但风险高）

### 12.3 典型用法

```bash
# ~/.harness/hooks/pre-tool-use.sh
#!/bin/bash
input=$(cat)  # stdin 是 JSON
tool_name=$(echo "$input" | jq -r '.tool')

if [[ "$tool_name" == "bash" ]]; then
  cmd=$(echo "$input" | jq -r '.input.command')
  if [[ "$cmd" == *"rm -rf /"* ]]; then
    echo '{"decision":"block","reason":"dangerous rm"}' >&2
    exit 2  # block
  fi
fi
exit 0  # allow
```

### 12.4 工程陷阱

- **Hook 超时**：默认 5s，避免阻塞 agent loop
- **Hook 错误处理**：hook 自己崩，不能让 agent 崩——降级为 warn
- **Hook 串行 vs 并行**：同 event 多 hook，按优先级串行；并行风险大

---

## 十三、必背的 10 个"数字 / 阈值"

| # | 数字 | 含义 |
| --- | --- | --- |
| 1 | **DIMINISHING_THRESHOLD = 500** | Claude Code 连续 3 次新增 <500 token 就停 |
| 2 | **COMPLETION_THRESHOLD = 0.9** | 进度 90% 视为完成 |
| 3 | **max concurrency = 10** | 默认 tool 并发上限 |
| 4 | **classifier timeout = 2s** | bash classifier 超时降级到 ask user |
| 5 | **MEMORY.md cap = 200 行 / 25KB** | 防 memory 占爆 context |
| 6 | **hook timeout = 5s** | 默认 hook 阻塞上限 |
| 7 | **cache hit 折扣 ~90%** | Anthropic prompt cache（vs 1× input） |
| 8 | **cache_control breakpoint max = 4** | Anthropic 显式标记上限 |
| 9 | **subagent 深度默认 = 2** | 防递归爆炸 |
| 10 | **maxResultSizeChars** | tool 输出超出落盘+返 path |

**面试金句**："Harness 的工程艺术其实在这些 numbers——每一个都是 trade-off。"

---

## 十四、源码学习路径推荐（T-5 ~ T-2）

按从简单到复杂顺序读：

1. **liteagent** (200 行 Python) —— 看 ReAct loop 最小骨架
2. **simple-agent-loop** (200 行) —— 看 subagent + parallel tool
3. **tinyAgent-TS** —— 看 TS 风格 + Zod schema
4. **OpenCode** (open) —— 看 production-grade Go/TS 实现
5. **Aider** —— 看 git-aware coding agent
6. **OpenHands / OpenClaw** —— 看 web + container 形态
7. **Claude Code 反编译版** —— 看商业级深度（要在 GitHub 搜，公开仓库有部分）

每个 repo 至少做一次：
- [ ] clone + 跑起来
- [ ] 读 main entry / agent loop / tool dispatcher 三个核心文件
- [ ] 找一个 issue 提 PR（哪怕只是 docs / 小 bug）

---

## 十五、一句应试钥匙

> **PM 答题时讲"概念 + trade-off"；工程师答题时讲"代码 + 数字 + 源码出处"。**

> 例："Tool 并发安全这件事 Claude Code 是用 `isConcurrencySafe` boolean 在 toolOrchestration.ts 里 partition——只有 currently running tools 都是 safe 才能加新的；默认 max 10。我自己 minimal harness 里抄了这个 pattern——可以一起看。"
