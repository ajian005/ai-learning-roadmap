# 12 · 自带作品集 · Mini Eval Platform 开源项目

> 这是全栈岗面试**最关键的杀手锏**——证明你真的为 agent trajectory 做过 viewer / 把 LangChain 跑到 pipeline 里 / 容器化部署过。
>
> JD 加分项第 1 条就是"**有 Agent 评测平台、轨迹回放或可视化工具的开发经验**"——一份能跑的 mini 项目直接命中。
>
> **使用方法**：T-7 ~ T-2 把这个 repo 写出来 push 到 GitHub，简历首屏链上去；面试开场提一句"我做了一个 mini eval platform，可以一起看"。
>
> 不需要 100 个 star，需要 **"面试官打开 5 分钟能看懂 + 想 clone"**。

---

## 一、项目命名与定位

### 名字候选

| 名字 | 风格 |
| --- | --- |
| `mini-eval-platform` | 直白 |
| `traje` | 短，trajectory + e |
| `agent-scope` | 强调"看" |
| `lookat` | 极简 |
| `[你的 initial]-eval` | 个人化 |

### 一句话定位

> **"A minimal agent eval platform in ~1500 lines — React + FastAPI + Postgres + Dockerfile + Helm. Implements trajectory viewer, batch scoring, LangChain adapter, all the basics of LangSmith / LangFuse but learnable in one sitting."**

---

## 二、项目结构

```
mini-eval-platform/
├── README.md
├── docker-compose.yml          # 一行 up 跑全栈
├── Makefile                    # 常用命令
├── .github/
│   └── workflows/
│       ├── ci.yml              # test + build + image push
│       └── deploy.yml          # k8s rollout（可选）
│
├── api/                        # Python FastAPI 后端
│   ├── Dockerfile
│   ├── pyproject.toml
│   ├── alembic/                # DB migration
│   ├── src/
│   │   ├── main.py
│   │   ├── db.py
│   │   ├── schema.py           # Pydantic
│   │   ├── trajectory.py       # CRUD endpoint
│   │   ├── score.py            # LLM-as-judge
│   │   ├── stream.py           # SSE
│   │   └── adapters/
│   │       ├── langchain_adapter.py
│   │       └── openai_agents_adapter.py
│   └── tests/
│
├── web/                        # React + TS 前端
│   ├── Dockerfile
│   ├── package.json
│   ├── vite.config.ts
│   ├── tsconfig.json
│   ├── src/
│   │   ├── main.tsx
│   │   ├── App.tsx
│   │   ├── api.ts              # TanStack Query hooks
│   │   ├── components/
│   │   │   ├── TrajectoryList.tsx
│   │   │   ├── TrajectoryViewer.tsx
│   │   │   ├── Timeline.tsx
│   │   │   ├── DiffView.tsx
│   │   │   ├── Leaderboard.tsx
│   │   │   └── ScoreCard.tsx
│   │   └── pages/
│   │       ├── ListPage.tsx
│   │       ├── DetailPage.tsx
│   │       └── DashboardPage.tsx
│   └── tests/
│
├── proto/                      # 共享 schema
│   └── trajectory.proto
│
├── helm/                       # K8s 部署
│   └── mini-eval-platform/
│       ├── Chart.yaml
│       ├── values.yaml
│       └── templates/
│
├── examples/
│   ├── 01_run_langchain.py     # 用 LangChain 跑 task + 上传 trajectory
│   ├── 02_run_openai_agents.py
│   └── 03_batch_score.py
│
└── docs/
    ├── ARCHITECTURE.md         # 详细图 + trade-off
    ├── ROADMAP.md
    └── ATTRIBUTION.md
```

**总代码量目标**：~1500 行（API 600 + Web 700 + 其余 200）。

---

## 三、最小可运行版本（V0）—— 2 小时跑通

> **T-7 当天就要做出来 V0**。功能极简，但 docker-compose up 能起来 + 一个 trajectory 能 list / detail。

### V0 docker-compose.yml

```yaml
services:
  db:
    image: postgres:16-alpine
    environment:
      POSTGRES_USER: eval
      POSTGRES_PASSWORD: eval
      POSTGRES_DB: eval
    ports: ["5432:5432"]
    volumes: [db_data:/var/lib/postgresql/data]
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U eval"]
      interval: 3s
      retries: 10

  api:
    build: ./api
    ports: ["8000:8000"]
    environment:
      DATABASE_URL: postgresql://eval:eval@db:5432/eval
    depends_on:
      db: { condition: service_healthy }
    command: ["uvicorn", "src.main:app", "--host", "0.0.0.0", "--port", "8000", "--reload"]
    volumes: ["./api:/app"]

  web:
    build: ./web
    ports: ["3000:3000"]
    environment:
      VITE_API_BASE_URL: http://localhost:8000
    volumes: ["./web:/app", "/app/node_modules"]
    command: ["pnpm", "dev", "--host"]

volumes:
  db_data:
```

### V0 minimal API (`api/src/main.py`)

```python
from fastapi import FastAPI, HTTPException
from fastapi.middleware.cors import CORSMiddleware
from pydantic import BaseModel
from typing import Optional
from datetime import datetime
import os, uuid
import asyncpg

app = FastAPI(title="mini-eval-platform-api")
app.add_middleware(CORSMiddleware, allow_origins=["*"], allow_methods=["*"], allow_headers=["*"])

DB_URL = os.environ["DATABASE_URL"]
pool: asyncpg.Pool = None

@app.on_event("startup")
async def startup():
    global pool
    pool = await asyncpg.create_pool(DB_URL, min_size=1, max_size=5)
    async with pool.acquire() as c:
        await c.execute("""
          CREATE TABLE IF NOT EXISTS trajectories (
            id UUID PRIMARY KEY,
            model TEXT NOT NULL,
            task_id TEXT NOT NULL,
            outcome TEXT NOT NULL,
            step_count INT NOT NULL,
            duration_ms INT NOT NULL,
            created_at TIMESTAMPTZ DEFAULT now(),
            steps JSONB NOT NULL
          );
          CREATE INDEX IF NOT EXISTS trajectories_model_idx ON trajectories(model, created_at DESC);
        """)

class TrajectoryIn(BaseModel):
    model: str
    task_id: str
    outcome: str  # success | fail | abandoned
    duration_ms: int
    steps: list[dict]

@app.post("/trajectory")
async def upload(t: TrajectoryIn):
    tid = str(uuid.uuid4())
    async with pool.acquire() as c:
        await c.execute(
            "INSERT INTO trajectories (id, model, task_id, outcome, step_count, duration_ms, steps) "
            "VALUES ($1, $2, $3, $4, $5, $6, $7::jsonb)",
            tid, t.model, t.task_id, t.outcome, len(t.steps), t.duration_ms,
            __import__('json').dumps(t.steps),
        )
    return {"id": tid}

@app.get("/trajectory")
async def list_(model: Optional[str] = None, limit: int = 50):
    where = ""
    params = []
    if model:
        params.append(model); where = "WHERE model = $1"
    sql = f"SELECT id, model, task_id, outcome, step_count, duration_ms, created_at FROM trajectories {where} ORDER BY created_at DESC LIMIT {limit}"
    async with pool.acquire() as c:
        rows = await c.fetch(sql, *params)
    return [dict(r) for r in rows]

@app.get("/trajectory/{tid}")
async def detail(tid: str):
    async with pool.acquire() as c:
        row = await c.fetchrow("SELECT * FROM trajectories WHERE id = $1", tid)
    if not row:
        raise HTTPException(404)
    return dict(row)

@app.get("/health")
async def health():
    return {"status": "ok"}
```

### V0 minimal Dockerfile (`api/Dockerfile`)

```dockerfile
FROM python:3.12-slim
WORKDIR /app
RUN pip install --no-cache-dir uv
COPY pyproject.toml ./
RUN uv pip install --system fastapi uvicorn asyncpg pydantic 'sse-starlette'
COPY . .
EXPOSE 8000
CMD ["uvicorn", "src.main:app", "--host", "0.0.0.0", "--port", "8000"]
```

### V0 minimal Web (`web/src/App.tsx`)

```tsx
import { QueryClient, QueryClientProvider, useQuery } from '@tanstack/react-query';
import { useState } from 'react';

const qc = new QueryClient();
const API = import.meta.env.VITE_API_BASE_URL || 'http://localhost:8000';

interface Traj {
  id: string; model: string; task_id: string; outcome: string;
  step_count: number; duration_ms: number; created_at: string;
}

function TrajectoryList() {
  const { data, isLoading } = useQuery({
    queryKey: ['list'],
    queryFn: () => fetch(`${API}/trajectory`).then(r => r.json() as Promise<Traj[]>),
  });
  const [selected, setSelected] = useState<string | null>(null);
  return (
    <div className="flex gap-4 p-4">
      <ul className="w-1/3 border-r pr-4">
        {isLoading && <li>Loading...</li>}
        {(data ?? []).map(t => (
          <li key={t.id} className={`p-2 cursor-pointer ${selected === t.id ? 'bg-blue-100' : 'hover:bg-gray-50'}`}
              onClick={() => setSelected(t.id)}>
            <div className="font-mono text-xs">{t.id.slice(0, 8)}</div>
            <div>{t.model} · {t.task_id}</div>
            <div className={t.outcome === 'success' ? 'text-green-600' : 'text-red-600'}>{t.outcome}</div>
          </li>
        ))}
      </ul>
      <div className="flex-1">
        {selected ? <TrajectoryDetail id={selected} /> : <div className="text-gray-400">Select a trajectory.</div>}
      </div>
    </div>
  );
}

function TrajectoryDetail({ id }: { id: string }) {
  const { data } = useQuery({
    queryKey: ['detail', id],
    queryFn: () => fetch(`${API}/trajectory/${id}`).then(r => r.json()),
  });
  if (!data) return <div>Loading...</div>;
  return (
    <pre className="bg-gray-50 p-4 overflow-x-auto text-xs">{JSON.stringify(data, null, 2)}</pre>
  );
}

export default function App() {
  return (
    <QueryClientProvider client={qc}>
      <header className="bg-blue-600 text-white p-3 font-bold">🛞 mini-eval-platform</header>
      <TrajectoryList />
    </QueryClientProvider>
  );
}
```

### 跑通

```bash
docker compose up --build

# 用 curl 上传一个 trajectory
curl -X POST http://localhost:8000/trajectory \
  -H 'Content-Type: application/json' \
  -d '{
    "model": "claude-sonnet-4-5",
    "task_id": "task-001",
    "outcome": "success",
    "duration_ms": 12000,
    "steps": [
      {"role": "user", "content": "List files"},
      {"role": "assistant", "tool_calls": [{"name": "list_files", "input": {"directory": "."}}]},
      {"role": "tool", "tool_name": "list_files", "output": "main.py\nREADME.md"},
      {"role": "assistant", "content": "Done."}
    ]
  }'

# 浏览 http://localhost:3000 看到列表
```

**T-7 完成度目标**：
- [ ] docker-compose up 起来
- [ ] curl 能上传 + list 能看到 + detail 能看
- [ ] push 到 GitHub
- [ ] README 写一段话

---

## 四、V1 渐进增加（T-6 ~ T-3）

| Day | 加什么 | 行数 | 难度 |
| --- | --- | --- | --- |
| T-6 | LangChain adapter + example | +200 | 中 |
| T-5 | 前端 Timeline 组件（virtual list）+ filter | +250 | 中 |
| T-5 | SSE streaming endpoint + 前端 EventSource | +150 | 中 |
| T-4 | LLM-as-judge score 异步队列 + 前端展示 | +200 | 中 |
| T-4 | OpenAI Agents SDK adapter | +150 | 中 |
| T-3 | Helm chart + K8s manifest（minikube 验证）| +150 | 中 |
| T-3 | GitHub Actions CI（test + build image push）| +80 | 易 |
| T-2 | Diff view + Leaderboard 简化版 | +250 | 难 |
| T-2 | README 美化 + 1 分钟 demo GIF | — | 易 |

> **优先级**：T-5 之前必须有 V1（LangChain adapter + Timeline + SSE）；T-3 之前必须有 Helm + CI。Diff / Leaderboard / OAI Agents SDK adapter 是 nice-to-have，时间不够可以放 V0.2 在面试后做。

---

## 五、README 模板

````markdown
# 🛞 mini-eval-platform

> A minimal agent eval platform in ~1500 lines — React + FastAPI + Postgres + Dockerfile + Helm. Implements trajectory viewer, batch scoring, LangChain adapter — all the basics of LangSmith / LangFuse but learnable in one sitting.

**Why this exists**: I wanted to understand "what does it take to build an internal eval platform for agent training?" by building one. Not a replacement for LangSmith / LangFuse — a reference implementation small enough to read in one weekend.

---

## ✨ Features

- ✅ **Trajectory viewer**: timeline + filter + virtual list (handles 10K+ steps)
- ✅ **LangChain adapter**: drop-in callback → unified trajectory schema
- ✅ **OpenAI Agents SDK adapter** (basic)
- ✅ **SSE streaming** for live trajectory playback
- ✅ **LLM-as-judge scoring**: async queue + retry + idempotent
- ✅ **Leaderboard**: model × benchmark grid with sort
- ✅ **Diff view**: align two trajectories side-by-side
- ✅ **Helm chart + K8s manifest** for K8s deploy
- ✅ **GitHub Actions CI**: test + build + push image

## 🚀 Quick Start

```bash
git clone https://github.com/you/mini-eval-platform
cd mini-eval-platform
docker compose up --build
# Open http://localhost:3000
```

```bash
# Upload a sample trajectory
make seed

# Run a LangChain agent and upload its trajectory
export OPENAI_API_KEY=...
python examples/01_run_langchain.py
```

## 🏗 Architecture

```
React UI ────► FastAPI ────► Postgres
                  │
                  ├──► S3 (steps blob, optional)
                  └──► Score Queue ────► Judge LLM
```

See [docs/ARCHITECTURE.md](docs/ARCHITECTURE.md) for deep dive.

## 📊 Comparison

| | LangSmith | LangFuse | **mini-eval-platform** |
| --- | --- | --- | --- |
| Lines of code | proprietary | ~100K+ | **~1500** |
| Self-host | (paid) | yes | **yes** |
| LangChain adapter | yes | yes | yes |
| OAI Agents SDK | yes | yes | yes (basic) |
| Trajectory diff | limited | no | **yes** |
| LLM-as-judge | yes | yes | yes |
| K8s ready | no | helm | yes |

## 🧪 Examples

```bash
python examples/01_run_langchain.py    # LangChain agent → upload
python examples/02_run_openai_agents.py # OAI SDK → upload
python examples/03_batch_score.py       # score 100 trajectories
```

## 🛣 Roadmap

V0.2:
- [ ] Pydantic AI adapter
- [ ] Protobuf trajectory schema
- [ ] OpenTelemetry export
- [ ] Failure clustering (embedding-based)

V1.0:
- [ ] Multi-tenant
- [ ] RBAC
- [ ] DuckDB OLAP queries on S3 parquet

## 📚 References

- [LangFuse](https://github.com/langfuse/langfuse) — heavy inspiration for trace UI
- [Phoenix (Arize)](https://github.com/Arize-ai/phoenix) — OpenInference standard
- [Anthropic Harness blogs](https://www.anthropic.com/engineering)
- [OpenTelemetry GenAI conventions](https://github.com/open-telemetry/semantic-conventions/tree/main/docs/gen-ai)

## License
MIT
````

---

## 六、面试中怎么演示这个 repo（剧本）

### 5 分钟版

> "我面试前用 ~1500 行做了一个 mini eval platform。GitHub 在 [URL]。
>
> 高层架构是 React 前端 + FastAPI 后端 + Postgres + Dockerfile + Helm chart。核心功能：trajectory viewer / LangChain adapter / SSE 回放 / LLM-as-judge 评分。
>
> 我刻意做了几个 production-grade 决定：
> 1. **业务代码不直接 import LangChain**——通过 adapter 层吸收
> 2. **trajectory schema 用 Pydantic 强类型 + Protobuf 备份**——schema 向前兼容
> 3. **scoring 异步队列 + idempotent**——重跑跳已成功
>
> 如果想看代码，[adapter 那块](./api/src/adapters/langchain_adapter.py) ~150 行特别能体现思路。"

### 15 分钟版

加：deep dive 一个组件（如 adapter / SSE 流 / Helm chart）+ 演示 live demo + 讨论 trade-off。

---

## 七、Push 前的 checklist

- [ ] LICENSE（MIT or Apache 2.0）
- [ ] .gitignore（不要 push API key / node_modules / __pycache__）
- [ ] .env.example 完整
- [ ] README 顶端有 GIF 或截图（asciinema / vhs）
- [ ] docker compose up 真的能跑（fresh clone 验证）
- [ ] 至少 1 个 examples 跑通
- [ ] GitHub Actions CI workflow 跑通（badge 显示绿色）
- [ ] 关键文件有 docstring
- [ ] commit message clean
- [ ] repo description + topics（agent, llm, eval, langchain, deepseek, observability）

---

## 八、面试官能 30 秒看出"你做过 vs 没做过"的细节

| 细节 | 做过的人会有 | 没做过的人不会写 |
| --- | --- | --- |
| **virtual list** for 大列表 | TanStack Virtual 引入 | map 渲染 10K 直接卡 |
| **EventSource** for SSE | 浏览器 EventSource API | fetch 流式但忘 reader.read |
| **partial JSON parse** for streaming tool input | json5 / partial-json | JSON.parse 等完 |
| **Pydantic 强类型** | BaseModel + Field | dict 到处用 |
| **asyncio.Semaphore** for concurrent control | acquire / release | 直接 gather 100 个并发 |
| **idempotent score** | unique constraint + ON CONFLICT | 不防重 |
| **distroless image** | gcr.io/distroless/python3 | python:slim 直接用 |
| **probe 三件套** | liveness + readiness + startup | 只有 liveness |
| **GitHub Actions detect-changes** | dorny/paths-filter | 每次跑全部 job |

> 这些细节 = 工程师的"指纹"。在面试中讲出来 = 让面试官知道你"真做过"，不是"读过"。

---

## 九、一句应试钥匙

> **简历可以是 PDF，但 "能力" 必须是 git URL + docker compose up。**
>
> **1 个能 5 分钟拉起跑通的 repo，胜过 50 行简历描述。**
>
> **DeepSeek 面试官能在 5 分钟里 docker compose up 看到你的 mini eval platform 跑起来——offer 基本就到了。**
