# 08 · 系统设计 · RL 训练集成 与 Scaffold 适配层

> 对应 JD 第 1 和第 4 条职责：
> - 把业界 Agent 开发工具（LangChain / OAI SDK 等）集成到内部 RL 基础设施
> - 维护内部 Agent 集成框架，支持便捷对接 RL 训练框架及外部 Agent Scaffold
>
> 这一篇侧重"**适配层设计**"——这是这个岗位的工程特色，比普通后端复杂。

---

## 一、题：设计统一 Agent Scaffold 适配层（45 分钟）

### 题面

> 内部已有自研 Agent；现在要支持 LangChain / OAI Agents SDK / Pydantic AI 三个外部 framework，让算法可以"换一行 import" 就把任意 framework 接到内部 RL trainer 数据 pipeline。设计这个适配层。

### Stage 1·Clarify

- 接入方式？→ Python in-process（统一进程跑）vs 子进程？ → 假设 in-process（性能 + state 共享）
- 外部 framework 版本控制？→ 我们 pin minor 版本，每月审查升级
- Agent state 需要持久化？→ 是，要能 trajectory replay
- Tool 接入？→ 内部 tool set + 各 framework 自带 tool 都要支持
- 接入 RL trainer？→ 输出 protobuf trajectory（见 [07 文档] schema）

### Stage 2·High-level

```
┌────────────────────────────────────────────────────────────────┐
│                                                                │
│       Algo 代码（写 task）                                      │
│         from deepseek_harness import Agent, Task               │
│         agent = Agent.from_scaffold("langchain", config=...)   │
│         result = agent.run(task)                               │
│                                                                │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │              Unified Agent Interface                      │  │
│  │  - run(task) -> RunResult                                 │  │
│  │  - emit_trajectory() -> proto.Trajectory                  │  │
│  │  - get_tools() -> list[Tool]                              │  │
│  │  - get_model_calls() -> list[ModelCall]                   │  │
│  └────────┬─────────────────────────────────────────────────┘  │
│           │                                                    │
│           │  factory + dispatcher                              │
│           │                                                    │
│  ┌────────┼──────────────┬───────────────────┬───────────┐    │
│  ▼        ▼              ▼                   ▼           ▼    │
│ Internal  LangChain      OpenAI Agents       PydanticAI  …     │
│ Harness   Adapter        SDK Adapter         Adapter           │
│ (native)                                                       │
│  │         │              │                  │                 │
│  └────┬────┴──────────────┴──────────────────┘                 │
│       │ all adapters output unified proto                      │
│       ▼                                                        │
│  ┌──────────────────────┐                                     │
│  │ Trajectory Collector │  ← emit to S3 / Kafka              │
│  └──────────┬───────────┘                                     │
│             ▼                                                  │
│        RL Trainer Pipeline                                     │
│        (consume Protobuf trajectory)                           │
│                                                                │
│  Cross-cutting:                                                │
│   - Common Tool Registry（统一 tool 跨 framework）              │
│   - Common Model Adapter（统一 LLM 调用）                       │
│   - OTel Tracing（每 framework auto-instrument）                │
│   - Permission / Sandbox                                       │
└────────────────────────────────────────────────────────────────┘
```

### Stage 3·Deep Dive（选 3 组件）

#### Pick 1·Unified Agent Interface

```python
# deepseek_harness/agent.py
from abc import ABC, abstractmethod
from typing import Protocol
from .schema import Trajectory, Task, RunResult, Tool, ModelCall

class Agent(ABC):
    @classmethod
    def from_scaffold(cls, name: str, **config) -> "Agent":
        """Factory: 'langchain' | 'openai_agents' | 'pydantic_ai' | 'internal'."""
        if name == "langchain":
            from .adapters.langchain_adapter import LangChainAgent
            return LangChainAgent(**config)
        if name == "openai_agents":
            from .adapters.oai_agents_adapter import OAIAgentsAgent
            return OAIAgentsAgent(**config)
        if name == "pydantic_ai":
            from .adapters.pydantic_ai_adapter import PydanticAIAgent
            return PydanticAIAgent(**config)
        if name == "internal":
            from .native import InternalAgent
            return InternalAgent(**config)
        raise ValueError(f"Unknown scaffold: {name}")

    @abstractmethod
    def run(self, task: Task) -> RunResult: ...

    @abstractmethod
    def emit_trajectory(self) -> Trajectory:
        """转 Protobuf trajectory for RL pipeline."""

    @abstractmethod
    def get_tools(self) -> list[Tool]: ...

    @abstractmethod
    def get_model_calls(self) -> list[ModelCall]: ...
```

#### Pick 2·LangChain Adapter（最复杂）

```python
# deepseek_harness/adapters/langchain_adapter.py
from typing import Any
from .base import Agent
from .._collector import TrajectoryCollector
from langchain.agents import AgentExecutor, create_openai_tools_agent
from langchain_openai import ChatOpenAI
from langchain.tools import Tool as LCTool
from langchain_core.callbacks import BaseCallbackHandler

class LangChainAgent(Agent):
    def __init__(
        self,
        model_name: str = "deepseek-coder",
        tools: list[Tool] = None,
        system_prompt: str = "",
        max_iterations: int = 30,
    ):
        self.model_name = model_name
        self.tools = tools or []
        self.collector = TrajectoryCollector()
        # 把内部 tool 转 LC tool（接口归一）
        lc_tools = [self._to_lc_tool(t) for t in self.tools]
        # 用内部 model gateway（不直连 OpenAI）
        llm = self._build_internal_llm(model_name)
        prompt = self._build_prompt(system_prompt)
        agent = create_openai_tools_agent(llm, lc_tools, prompt)
        self.executor = AgentExecutor(
            agent=agent, tools=lc_tools,
            max_iterations=max_iterations,
            return_intermediate_steps=True,
            callbacks=[self.collector],
        )

    def run(self, task) -> RunResult:
        result = self.executor.invoke({"input": task.prompt})
        return RunResult(
            text=result.get("output"),
            success=self._infer_success(result),
        )

    def emit_trajectory(self) -> Trajectory:
        return self.collector.to_proto()

    def get_tools(self) -> list[Tool]:
        return self.tools

    def get_model_calls(self) -> list[ModelCall]:
        return self.collector.get_model_calls()

    def _to_lc_tool(self, t: Tool) -> LCTool:
        from langchain_core.tools import StructuredTool
        return StructuredTool.from_function(
            func=t.func,
            name=t.name,
            description=t.description,
            args_schema=t.args_schema,
        )

    def _build_internal_llm(self, model_name: str):
        from langchain_openai import ChatOpenAI
        return ChatOpenAI(
            model=model_name,
            openai_api_base="https://internal-llm-gateway.deepseek.internal/v1",
            openai_api_key="internal",
            temperature=0,
        )

    def _build_prompt(self, system_prompt: str):
        from langchain_core.prompts import ChatPromptTemplate, MessagesPlaceholder
        return ChatPromptTemplate.from_messages([
            ("system", system_prompt),
            MessagesPlaceholder("chat_history", optional=True),
            ("human", "{input}"),
            MessagesPlaceholder("agent_scratchpad"),
        ])
```

#### Pick 3·Trajectory Collector（核心归一）

> 把任意 framework 的内部事件转 unified Protobuf Trajectory。

```python
# deepseek_harness/_collector.py
from langchain_core.callbacks import BaseCallbackHandler
import time
import uuid
from .schema import Trajectory, Step, StepRole, ToolCall, ToolResult, Outcome
from typing import Any

class TrajectoryCollector(BaseCallbackHandler):
    def __init__(self):
        self.id = str(uuid.uuid4())
        self.steps: list[Step] = []
        self.start_ts = time.time()
        self._step_index = 0
        # tracking model calls for cost
        self._model_calls: list[dict] = []

    def _next_index(self) -> int:
        i = self._step_index
        self._step_index += 1
        return i

    def _ts_ms(self) -> int:
        return int((time.time() - self.start_ts) * 1000)

    # --- LangChain Callback Hooks ---
    def on_chat_model_start(self, serialized, messages, **kw):
        # 把用户 message 转为 step
        last_human = next((m for m in messages[0]
                          if m.type == "human"), None)
        if last_human and not self.steps:  # 仅首次
            self.steps.append(Step(
                index=self._next_index(),
                role=StepRole.USER,
                ts_ms=self._ts_ms(),
                content=last_human.content,
            ))

    def on_llm_end(self, response, **kw):
        gen = response.generations[0][0]
        message = gen.message
        tool_calls = []
        if hasattr(message, 'tool_calls') and message.tool_calls:
            for tc in message.tool_calls:
                tool_calls.append(ToolCall(
                    id=tc['id'],
                    name=tc['name'],
                    input_json=str(tc['args']).encode(),  # serialize 化
                ))
        self.steps.append(Step(
            index=self._next_index(),
            role=StepRole.ASSISTANT,
            ts_ms=self._ts_ms(),
            content=getattr(message, 'content', '') or '',
            tool_calls=tool_calls,
        ))
        # capture model call meta for cost
        usage = response.llm_output or {}
        token_usage = usage.get('token_usage', {})
        self._model_calls.append({
            'input_tokens': token_usage.get('prompt_tokens', 0),
            'output_tokens': token_usage.get('completion_tokens', 0),
            'model': usage.get('model_name'),
        })

    def on_tool_end(self, output, **kw):
        self.steps.append(Step(
            index=self._next_index(),
            role=StepRole.TOOL,
            ts_ms=self._ts_ms(),
            tool_results=[ToolResult(
                tool_use_id='',  # LangChain 没暴露准确 id，需 backfill
                output=str(output)[:50000],
                is_error=False,
            )],
        ))

    def to_proto(self) -> Trajectory:
        total_in = sum(m['input_tokens'] for m in self._model_calls)
        total_out = sum(m['output_tokens'] for m in self._model_calls)
        return Trajectory(
            id=self.id,
            model=self._model_calls[0]['model'] if self._model_calls else '',
            step_count=len(self.steps),
            input_tokens=total_in,
            output_tokens=total_out,
            total_tokens=total_in + total_out,
            duration_ms=self._ts_ms(),
            outcome=Outcome.SUCCESS,  # caller 决定
            steps=self.steps,
        )

    def get_model_calls(self) -> list:
        return self._model_calls
```

### Stage 4·Scale + Failure + Maintainability

#### Scale
- in-process 50 个 trajectory 并行 OK；100+ 起独立 worker process
- 跨 framework 内存隔离弱 → 容器化隔离

#### Failure
- LangChain crash / 改 API：adapter 层吸收
- 内部 tool 异常：try/except + 转 ToolResult is_error=True
- LLM rate limit：retry with backoff
- Trajectory collector OOM：超出 step 阈值落 partial proto

#### Maintainability
- **每周自动跑 framework upgrade compatibility test**
- Adapter test：fixture trajectory 跑各 framework → 同 task 应产 schema-相同 trajectory
- 文档：每 framework 一个"已知 limit + 不支持 feature"清单

---

## 二、关键追问

### Q1·LangChain 升级新版接口大改，adapter 不能再用怎么办？

> "三阶段：
> 1. **吸收**：旧版 adapter pin 旧 LC 版本，业务代码不动
> 2. **新版本 sandbox 验证**：新 adapter 用 canary user 跑 1 周
> 3. **切换 + 弃旧**：全量切，旧 adapter 标 deprecated；保留 6 月 backward compat
>
> 关键 invariant：**business code 永远不直接 import langchain**——只用 deepseek_harness Agent 接口。"

### Q2·内部 tool 怎么和 framework tool 互操作？

> "我会做 **2 层抽象**：
> 1. **Internal Tool spec**：内部统一 schema（name / description / inputSchema / func / metadata / permission）
> 2. **Per-framework converter**：
>    - 转 LangChain `StructuredTool`
>    - 转 OAI Agents SDK `@function_tool`
>    - 转 Pydantic AI `tool`
>
> 这样：内部 tool 写一次，跨 framework 可用；framework 自带 tool 也能 import 进 internal registry。"

### Q3·OAI Agents SDK 内置 OTel tracing，怎么和内部 tracing 不冲突？

> "OAI Agents SDK 默认上传 trace 到 OpenAI——必须关或重定向：
> 1. 启动时 `set_default_openai_client(None)` + 配 internal OTel exporter
> 2. 拦截 OAI tracer，把 span 转 internal trace 格式
> 3. 内部 OTel collector 接收所有 framework 的 span，统一 trace_id
>
> 这样研究员看 internal trace UI 就能完整看到任何 framework 的 trajectory，不需要去 OpenAI dashboard。"

### Q4·算法说"我用了 LangGraph 而不是 LangChain，你也支持吗？"

> "支持，但有边界：
> - **优先级 P1**：LangChain / OAI Agents SDK / 内部 harness（80% 算法在用的）
> - **优先级 P2**：LangGraph / Pydantic AI（应需求加）
> - **优先级 P3 (best-effort)**：CrewAI / AutoGen / etc
>
> P2 框架的 adapter 由 maintainer + 提需求的算法 co-own——避免 platform team 维护 8 个 adapter。
> 如果 P3 框架想加，先提 RFC + 性价比评估。"

---

## 三、可能的 follow-up 设计题

### Follow-up 题 A·RL trainer 怎么消费 trajectory？

> "Reader 端读 S3 parquet：
> 1. Polars / DuckDB 跨日 partition 读
> 2. 按 outcome / model / score 过滤
> 3. dedup（embedding 相似度）
> 4. shard 给训练 worker
> 5. tokenize 转训练样本
>
> trainer 不直接 join PG（不是 OLAP system）。"

### Follow-up 题 B·算法说 trajectory 数据格式不够好 train 怎么改？

> "三步：
> 1. 我先听具体不够什么——是字段缺、format 不好、还是 sample 偏？
> 2. 字段缺：schema v2 加字段 + 老数据回填（如果可能）
> 3. format：可能要新增 'training_view' 投影（trainer 友好）vs 'eval_view'（人友好）
>
> 关键：我不让 schema 兼顾所有 use case——一份原始 trajectory + 多个 view。"

---

## 四、必背 Adapter Pattern 心法

| 心法 | 应用 |
| --- | --- |
| **业务代码永不 import 框架** | 永远走 internal interface |
| **Adapter 单文件单 framework** | 解耦 + 易维护 |
| **Output 统一 schema** | RL pipeline 不知道 framework |
| **每 adapter 必有 contract test** | 同 fixture 跑各 framework，输出归一 |
| **Version 显式 pin** | requirements 锁 minor / patch |
| **Adapter 可以借鉴但不抽象过早** | 第一版各 framework 重复 OK；第三版再 refactor |

---

## 五、一句应试钥匙

> **"内部统一接口 + framework-specific adapter + 统一 output schema" 这三件事讲清楚 = 系统设计 80% 分。**
>
> **加分：你能讲清"什么时候不抽象（YAGNI）"——比如 LangGraph 第一年只有 1 个用户，先不做 adapter，让他直接 dump JSON 给 trainer——等真有第 5 个用户再抽象。**
