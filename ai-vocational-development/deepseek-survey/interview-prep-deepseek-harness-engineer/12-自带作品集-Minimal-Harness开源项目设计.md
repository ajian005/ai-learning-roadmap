# 12 · 自带作品集 · Minimal Harness 开源项目设计

> **这是工程师面试最关键的杀手锏**——让你从"候选池一个简历"变成"团队已经在用我的代码"。
>
> **使用方法**：
> 1. 在 T-7 ~ T-2 期间真的把这个 repo 写出来 + push 到 GitHub
> 2. README 在你简历首屏链上去
> 3. 面试开场说："**我面试前用 ~500 行实现了一个 minimal harness，包含 agent loop / tool dispatcher / sandbox / lifecycle hook，代码在 [URL]，今天可以一起 review**"
> 4. 不强求 100 个 star，强求 **"面试官打开 5 分钟能看懂 + 想 clone"**
>
> 本文档是你的**项目设计 + 实施 checklist + README 模板**。

---

## 一、项目命名与定位

### 名字候选（挑一个你喜欢的）

| 名字 | 风格 |
| --- | --- |
| `mini-harness` | 直白 |
| `pico-claude` | 致敬 Claude Code |
| `tiny-agent` | 致敬 tinyAgent-TS |
| `nano-coder` | 强调 coder 场景 |
| `boot-strap` | 双关：启动 + 自举 |
| `harness-zero` | "从 0 开始" |
| `[你名字 initial]-harness` | 个人化 |

> 建议挑短 + 易记 + 可全小写 + 不冲突的 GitHub 名。

### 一句话定位（写到 repo description）

> **"A minimal yet idiomatic Agent Harness in ~500 lines of [Python/TypeScript]. Implements core patterns from Claude Code, OpenCode, and Anthropic's harness blogs — agent loop, tool dispatch, sandbox, lifecycle hooks."**

---

## 二、项目结构（参考）

### Python 版（推荐先做这个）

```
mini-harness/
├── README.md
├── LICENSE                    # MIT or Apache 2.0
├── pyproject.toml
├── requirements.txt
├── .env.example               # ANTHROPIC_API_KEY / DEEPSEEK_API_KEY
├── .gitignore
│
├── mini_harness/
│   ├── __init__.py
│   ├── core/
│   │   ├── agent_loop.py      # ~80 lines: Agent Loop with budget/abort
│   │   ├── messages.py        # ~30: message helpers, validation
│   │   ├── models.py          # ~50: model adapter (DS / Anthropic / OpenAI)
│   │   └── stream.py          # ~40: SSE parser
│   │
│   ├── tools/
│   │   ├── registry.py        # ~30: ToolRegistry
│   │   ├── dispatcher.py      # ~50: concurrency-safe dispatcher
│   │   ├── builtin/
│   │   │   ├── read_file.py
│   │   │   ├── write_file.py
│   │   │   ├── list_files.py
│   │   │   ├── grep.py
│   │   │   └── bash.py        # ~40: subprocess + timeout + sandbox
│   │   └── mcp/
│   │       └── client.py      # ~60: minimal MCP stdio client
│   │
│   ├── permission/
│   │   ├── engine.py          # ~50: rule-based + ask user
│   │   └── rules.py
│   │
│   ├── sandbox/
│   │   ├── base.py            # ~20: abstract
│   │   ├── macos.py           # ~40: sandbox_init
│   │   └── linux.py           # ~40: bubblewrap wrapper
│   │
│   ├── context/
│   │   ├── builder.py         # ~30: build_context
│   │   └── compactor.py       # ~50: keep_recent + summarize
│   │
│   ├── hooks/
│   │   └── registry.py        # ~40: 8 lifecycle hooks
│   │
│   ├── cli.py                 # ~60: entry point (typer / argparse)
│   └── config.py              # ~30: load .harnessrc
│
├── tests/
│   ├── test_agent_loop.py
│   ├── test_tool_dispatch.py
│   ├── test_context.py
│   └── test_e2e.py            # 1~2 real LLM calls (mocked or live)
│
├── examples/
│   ├── 01_hello.py            # Hello world: 1 tool, 1 task
│   ├── 02_refactor.py         # 跨文件重构 demo
│   └── 03_mcp_integration.py  # MCP server integration
│
└── docs/
    ├── ARCHITECTURE.md        # 详细架构图 + trade-off
    ├── COMPARE_TO_CC.md       # 和 Claude Code 的对照
    └── ROADMAP.md             # V0.1 ~ V1.0 路线
```

**总代码量目标**：500 ~ 800 行 Python（不含测试和 docs）。

### TypeScript 版（如果有时间做第二个）

```
mini-harness-ts/
├── package.json (bunfig.toml)
├── tsconfig.json
├── README.md
├── src/
│   ├── core/{agentLoop,messages,models,stream}.ts
│   ├── tools/{registry,dispatcher,builtin/,mcp/}.ts
│   ├── permission/{engine,rules}.ts
│   ├── sandbox/{base,darwin,linux,win32}.ts
│   ├── context/{builder,compactor}.ts
│   ├── hooks/registry.ts
│   └── cli.ts
└── ...
```

---

## 三、最小可运行版本（V0）—— 1 小时跑通

> **T-7 当天就要做出来 V0**。功能极简，但 end-to-end 跑通。

```python
# mini_harness/v0.py  — ALL IN ONE FILE，~80 lines
import os
import json
from anthropic import Anthropic

client = Anthropic()

TOOLS = {
    "read_file": {
        "fn": lambda path: open(path).read()[:5000],
        "schema": {
            "name": "read_file",
            "description": "Read content of a file",
            "input_schema": {
                "type": "object",
                "properties": {"path": {"type": "string"}},
                "required": ["path"],
            },
        },
    },
    "list_files": {
        "fn": lambda directory: "\n".join(sorted(os.listdir(directory))),
        "schema": {
            "name": "list_files",
            "description": "List files in a directory",
            "input_schema": {
                "type": "object",
                "properties": {"directory": {"type": "string"}},
                "required": ["directory"],
            },
        },
    },
}

SYSTEM_PROMPT = """You are a minimal coding agent. Use tools to explore and read files.
When you have enough info, give a final answer."""

def run(user_input: str, max_turns: int = 20) -> str:
    messages = [{"role": "user", "content": user_input}]
    tool_schemas = [t["schema"] for t in TOOLS.values()]

    for turn in range(max_turns):
        resp = client.messages.create(
            model="claude-sonnet-4-5",
            max_tokens=2048,
            system=SYSTEM_PROMPT,
            tools=tool_schemas,
            messages=messages,
        )
        messages.append({"role": "assistant", "content": resp.content})

        if resp.stop_reason == "end_turn":
            final = "".join(b.text for b in resp.content if b.type == "text")
            return final

        tool_results = []
        for blk in resp.content:
            if blk.type != "tool_use":
                continue
            print(f"[turn {turn}] calling {blk.name}({json.dumps(blk.input)})")
            try:
                out = TOOLS[blk.name]["fn"](**blk.input)
            except Exception as e:
                out = f"ERROR {type(e).__name__}: {e}"
            tool_results.append({
                "type": "tool_result",
                "tool_use_id": blk.id,
                "content": str(out)[:5000],
            })
        messages.append({"role": "user", "content": tool_results})

    return "[Max turns exceeded]"

if __name__ == "__main__":
    print(run("List the current directory and tell me what kind of project this is."))
```

**T-7 完成度目标**：
- [ ] 上面 80 行能跑通
- [ ] push 到 GitHub
- [ ] README 写一段：what it is + how to run

---

## 四、V1 渐进增加（T-6 ~ T-4）

| Day | 加什么 | 行数 | 难度 |
| --- | --- | --- | --- |
| T-6 | 拆模块（agent_loop / tools / models）+ 多模型 adapter | +150 | 易 |
| T-6 | Schema validator + retry on schema error | +80 | 易 |
| T-5 | Concurrency-safe dispatcher (asyncio) | +60 | 中 |
| T-5 | SSE streaming output | +80 | 中 |
| T-4 | Permission engine（rule-based + ask user）| +100 | 中 |
| T-4 | Bash tool with sandbox（subprocess timeout + cwd jail）| +60 | 中 |
| T-3 | Lifecycle hooks（PreToolUse + PostToolUse + PreCompact + Error）| +80 | 中 |
| T-3 | Context compactor（keep_recent + summarize）| +60 | 中 |
| T-2 | Minimal MCP stdio client（1 个 server 跑通即可）| +80 | 难 |
| T-2 | 跑 4 个 examples + 写 ARCHITECTURE.md | — | 易 |

> **优先级建议**：T-5 之前必须有 V1（streaming + concurrency + multi-model）；T-3 之前必须有 hooks + permission。MCP 是 nice-to-have，时间不够可以放 V0.2 在面试后做。

---

## 五、README 模板（重要：面试官只看 README）

````markdown
# 🛞 mini-harness

> A minimal yet idiomatic Agent Harness in ~500 lines of Python. Inspired by Claude Code, OpenCode, Anthropic's [harness](https://www.anthropic.com/engineering/effective-harnesses-for-long-running-agents) [series](https://www.anthropic.com/engineering/harness-design-long-running-apps).

**Why this exists**: To understand "what does it really take to build a coding agent harness?" by building one. Not a replacement for Claude Code—a reference implementation that's small enough to read in one sitting.

---

## ✨ Features

- ✅ **Agent Loop** with streaming, abort, max_turns, diminishing-returns detector
- ✅ **Multi-model adapters**: DeepSeek, Anthropic, OpenAI
- ✅ **Concurrency-safe tool dispatcher** (Claude Code pattern)
- ✅ **Schema validation** with structured retry
- ✅ **Permission engine** (rule-based + ask user + persist)
- ✅ **OS-level sandbox** (macOS sandbox_init / Linux bubblewrap)
- ✅ **8 lifecycle hooks** (PreToolUse, PostToolUse, PreCompact, ...)
- ✅ **Context compactor** (keep_recent + LLM summarize + cache-aware)
- ✅ **Minimal MCP stdio client**

## 🚀 Quick Start

```bash
git clone https://github.com/you/mini-harness
cd mini-harness
pip install -e .
cp .env.example .env  # add your API key

# Hello world
python -m mini_harness "list files in current dir, tell me what this project is about"
```

## 🏗 Architecture

```
┌──────────────────────────────────────────┐
│              CLI / Python API            │
└──────┬───────────────────────────────────┘
       │
┌──────▼───────────────────────────────────┐
│           AgentLoop                       │
│  - stream / abort / budget               │
│  - diminishing-returns detector          │
└──────┬───────────────────────────────────┘
       │
       ├── ModelAdapter (DS / Anthropic / OAI)
       ├── ToolDispatcher (concurrency-safe partition)
       │     ├── Permission Engine (rule + ask)
       │     ├── Sandbox (macOS / Linux)
       │     └── Tools (read/write/bash/grep/mcp)
       ├── ContextBuilder (cache-aware)
       └── HookRegistry (8 events)
```

See [docs/ARCHITECTURE.md](docs/ARCHITECTURE.md) for deep dive.

## 📊 Comparison with Claude Code

| | Claude Code | mini-harness |
| --- | --- | --- |
| Lines of code | ~513K | ~500 |
| Tools | 40 | 6 (extensible) |
| Lifecycle hooks | 12 | 8 |
| Sandbox | macOS+Linux+Win | macOS+Linux |
| Multi-model | No | DS/Anthropic/OAI |
| MCP support | Yes | Yes (stdio only) |
| Cache control | Yes | Yes |

See [docs/COMPARE_TO_CC.md](docs/COMPARE_TO_CC.md).

## 🧪 Examples

```bash
# Refactor multi-file
python examples/02_refactor.py

# MCP integration
python examples/03_mcp_integration.py
```

## 🛣 Roadmap

V0.2:
- [ ] GAN-style 3-agent (Planner / Generator / Evaluator)
- [ ] Windows sandbox
- [ ] Context reset with handoff artifact
- [ ] Better cache breakpoint heuristics

V1.0:
- [ ] Subagent (Task tool)
- [ ] TUI with Ink-like rendering
- [ ] Telemetry pipeline

## 🤝 Why I Built This

I built this to (a) deeply understand harness engineering by re-implementing core patterns; (b) explore what trade-offs change when you optimize for ~500 lines instead of ~500K; (c) provide an actually-readable reference for others learning agent design.

If you're learning agent harness design, the code is structured to be read top-to-bottom. Start with `mini_harness/core/agent_loop.py` and follow the imports.

## 📚 References

- [Claude Code source-level study](https://duocodetech.com/blog/claude-code-harness-engineering)
- [Anthropic: Effective harnesses for long-running agents](https://www.anthropic.com/engineering/effective-harnesses-for-long-running-agents)
- [Anthropic: Harness design for long-running application development](https://www.anthropic.com/engineering/harness-design-long-running-apps)
- [Anthropic: Claude Code sandboxing](https://www.anthropic.com/engineering/claude-code-sandboxing)
- [OpenCode](https://github.com/sst/opencode)
- [liteagent](https://github.com/drchrislevy/liteagent)

## License

MIT
````

---

## 六、关键代码片段（拷贝即用）

### 1. Multi-model Adapter（~50 行）

```python
# mini_harness/core/models.py
from abc import ABC, abstractmethod
import os

class ModelAdapter(ABC):
    @abstractmethod
    def stream(self, messages: list, tools: list, **kw):
        """Yields normalized stream events."""
        ...

class AnthropicAdapter(ModelAdapter):
    def __init__(self, model: str = "claude-sonnet-4-5"):
        from anthropic import Anthropic
        self.client = Anthropic()
        self.model = model

    def stream(self, messages, tools, **kw):
        with self.client.messages.stream(
            model=self.model,
            max_tokens=kw.get("max_tokens", 4096),
            system=kw.get("system"),
            tools=tools,
            messages=messages,
        ) as stream:
            for event in stream:
                yield from self._normalize_anthropic_event(event)

class DeepSeekAdapter(ModelAdapter):
    def __init__(self, model: str = "deepseek-coder"):
        from openai import OpenAI
        self.client = OpenAI(
            api_key=os.environ["DEEPSEEK_API_KEY"],
            base_url="https://api.deepseek.com/v1",
        )
        self.model = model

    def stream(self, messages, tools, **kw):
        # Convert Anthropic-format tools to OpenAI format
        oai_tools = [
            {"type": "function", "function": {
                "name": t["name"],
                "description": t["description"],
                "parameters": t["input_schema"],
            }} for t in tools
        ]
        stream = self.client.chat.completions.create(
            model=self.model,
            messages=self._convert_messages(messages),
            tools=oai_tools,
            stream=True,
        )
        for chunk in stream:
            yield from self._normalize_oai_event(chunk)
```

### 2. Concurrency-Safe Dispatcher（~50 行）

```python
# mini_harness/tools/dispatcher.py
import asyncio
from dataclasses import dataclass

@dataclass
class ToolCall:
    id: str
    name: str
    input: dict

async def dispatch(tool_calls: list[ToolCall], tool_registry, max_concurrency=10) -> list:
    """Partition by concurrency-safety, run safe ones in parallel."""
    results: list = [None] * len(tool_calls)
    sem = asyncio.Semaphore(max_concurrency)

    async def run_one(idx: int, tc: ToolCall):
        async with sem:
            tool = tool_registry.get(tc.name)
            if tool is None:
                results[idx] = {"error": f"Unknown tool {tc.name}"}
                return
            try:
                results[idx] = await tool.call(tc.input)
            except Exception as e:
                results[idx] = {"error": f"{type(e).__name__}: {e}"}

    batches: list = []
    current: list = []
    for i, tc in enumerate(tool_calls):
        tool = tool_registry.get(tc.name)
        is_safe = tool and tool.is_concurrency_safe
        if is_safe:
            current.append((i, tc))
        else:
            if current:
                batches.append(current)
                current = []
            batches.append([(i, tc)])
    if current:
        batches.append(current)

    for batch in batches:
        await asyncio.gather(*[run_one(i, tc) for i, tc in batch])
    return results
```

### 3. Permission Engine 骨架（~50 行）

```python
# mini_harness/permission/engine.py
from dataclasses import dataclass, field
from typing import Literal, Optional

@dataclass
class PolicyRule:
    tool_pattern: Optional[str] = None  # regex
    decision: Literal["allow", "deny", "ask"] = "ask"
    priority: int = 100
    reason: str = ""

@dataclass
class PermissionEngine:
    rules: list[PolicyRule] = field(default_factory=list)
    session_cache: dict = field(default_factory=dict)

    def evaluate(self, tool_name: str, tool_input: dict, ctx) -> dict:
        import re
        # 1. Session cache
        key = (tool_name, str(sorted(tool_input.items())))
        if key in self.session_cache:
            return self.session_cache[key]

        # 2. Match rules by priority
        matched = [
            r for r in sorted(self.rules, key=lambda x: -x.priority)
            if not r.tool_pattern or re.match(r.tool_pattern, tool_name)
        ]
        # 3. Deny wins
        deny = next((r for r in matched if r.decision == "deny"), None)
        if deny:
            return {"behavior": "deny", "reason": deny.reason}

        ask = next((r for r in matched if r.decision == "ask"), None)
        if ask:
            # 4. Prompt user (CLI input or callback)
            user_resp = ctx.prompt_user(tool_name, tool_input, ask.reason)
            if user_resp == "yes_session":
                self.session_cache[key] = {"behavior": "allow"}
            return {"behavior": "allow" if user_resp.startswith("yes") else "deny"}

        allow = next((r for r in matched if r.decision == "allow"), None)
        if allow:
            return {"behavior": "allow"}

        # 5. Default: ask
        return {"behavior": "ask"}
```

### 4. Lifecycle Hook 注册器（~40 行）

```python
# mini_harness/hooks/registry.py
import asyncio
from typing import Callable, Awaitable, Literal

HookEvent = Literal["SessionStart", "SessionEnd", "UserSubmit", "AgentResponse",
                    "PreToolUse", "PostToolUse", "PreCompact", "Error"]

class HookRegistry:
    def __init__(self):
        self._handlers: dict[HookEvent, list] = {}

    def on(self, event: HookEvent, handler: Callable[[dict], Awaitable[dict]]):
        self._handlers.setdefault(event, []).append(handler)

    async def emit(self, event: HookEvent, data: dict, timeout: float = 5.0) -> dict:
        """Run all handlers serially; collect modifications.
        deny rule wins over allow even if a later hook returns allow.
        """
        decision = "allow"
        modified = data
        for handler in self._handlers.get(event, []):
            try:
                result = await asyncio.wait_for(handler(modified), timeout=timeout)
            except asyncio.TimeoutError:
                continue  # treat as no-op
            if not isinstance(result, dict):
                continue
            if result.get("decision") == "deny":
                return {"decision": "deny", "reason": result.get("reason", "")}
            if result.get("decision") == "allow":
                decision = "allow"
            if "modified" in result:
                modified = result["modified"]
        return {"decision": decision, "modified": modified}
```

---

## 七、demo script（让面试官 30 秒看到价值）

```bash
# README 顶端嵌入一个 GIF 或 asciinema 录屏

$ mini-harness "Refactor this Python file to use type hints"
[turn 0] calling read_file({"path": "main.py"})
[turn 0] calling list_files({"directory": "tests"})
[turn 1] calling write_file({"path": "main.py", "content": "..."})
[turn 1] calling bash({"command": "pytest tests/ -q"})
✓ Type hints added; all 12 tests pass.

🛞 mini-harness  6 turns  $0.02  18s
```

> 用 [vhs](https://github.com/charmbracelet/vhs) 或 [asciinema](https://asciinema.org/) 录一个 1 分钟 demo，传 README 顶端。

---

## 八、面试中怎么演示这个 repo（剧本）

### 剧本 A·5 分钟版

> "我面试前用 500 行实现了一个 minimal harness。这是 [GitHub URL]。
>
> 高层架构是这样——[展示 README 架构图]——核心是 5 层：agent loop / tool dispatcher / permission / sandbox / hooks。
>
> 设计上我刻意做了几个 Claude Code 那种风格的决定：
> 1. **concurrency-safe partition** —— Claude Code 用 isConcurrencySafe boolean 在 toolOrchestration.ts 里分批，我抄了这个 pattern
> 2. **deny rule 永远 > allow** —— 即便 hook 返回 allow，命中 deny 仍 deny
> 3. **diminishing-returns detector** —— 连续 3 轮 token 增量 < 500 就停
>
> 如果时间够，可以一起看 [具体文件，30 行级]——这是 dispatcher 的 partition 逻辑。"

### 剧本 B·15 分钟版

加上：deep dive 一个组件 + 演示 live run + 讨论 trade-off。

---

## 九、Push 前的 checklist

- [ ] LICENSE 文件（MIT or Apache 2.0）
- [ ] .gitignore（不要 push API key / __pycache__）
- [ ] .env.example（不要 push 真的 .env）
- [ ] README 顶端有 1 段话说"why this exists"
- [ ] Quick Start 命令真的能跑（你自己 fresh clone 验证过）
- [ ] 至少 1 个 example 能跑
- [ ] requirements.txt / pyproject.toml 完整
- [ ] 至少 5 个 test passing
- [ ] 关键文件有 docstring
- [ ] commit message clean（不要 'wip' 'asdf'）
- [ ] repo description 一句话
- [ ] topics: agent / llm / harness / claude-code / deepseek

---

## 十、一句应试钥匙

> **你的简历可以是 PDF，但你的"能力"必须是 git URL。**
>
> 1 个能 clone + 能 run + 能 read 的 repo，胜过 10 句 "I'm familiar with agent design"。
>
> **面试官能在 5 分钟里点开你的 repo + 心里说"这人会写"——offer 基本就到了。**
