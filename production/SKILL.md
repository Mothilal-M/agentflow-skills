---
name: production
description: Deploy 10xscale-agentflow agents as APIs, with human-in-the-loop, callbacks, and event publishers
metadata:
  tags: agentflow, fastapi, deployment, docker, human-in-the-loop, observability
---

## When to use

Load this skill when the user moves from a working prototype to a deployable service: serving over HTTP, persisting state, pausing for human approval, emitting observability events, or shipping a Docker image.

## The AgentFlow CLI

Install the CLI alongside the library:

```bash
pip install 10xscale-agentflow-cli
```

Available commands:

```bash
agentflow init              # scaffold agentflow.json + graph/react.py
agentflow api               # run the FastAPI server against your graph
agentflow play              # run the server and open the hosted playground
agentflow build             # generate a Dockerfile (and optional docker-compose.yml)
agentflow skills add NAME   # install an agent skill into this project
agentflow version
```

The typical flow:

```bash
agentflow init
# edit graph/react.py to describe your agent
agentflow api       # http://127.0.0.1:8000 — /invoke, /stream, /ping
agentflow build --docker-compose
docker compose up
```

## agentflow.json

Created by `agentflow init`. The key field is `graph_path`, which points to a Python module exporting a compiled graph. Example:

```json
{
  "graph_path": "graph.react:app",
  "env_file": ".env"
}
```

The CLI imports `graph.react` and serves its `app` attribute.

## Persistent state (checkpointers)

For anything beyond local demos, replace `InMemoryCheckpointer` with the PostgreSQL + Redis checkpointer:

```bash
pip install 10xscale-agentflow[pg_checkpoint]
```

```python
from agentflow.storage.checkpointer import PgCheckpointer

checkpointer = PgCheckpointer(
    pg_dsn="postgresql://user:pass@localhost/agentflow",
    redis_url="redis://localhost:6379/0",   # optional cache
)
app = graph.compile(checkpointer=checkpointer)
```

Postgres is the durable store; the optional Redis URL caches recent thread IDs. The same `thread_id` resumes the same conversation.

## Human-in-the-loop

Pause the graph for approval; resume later with full state intact. Two moving parts:

1. A **checkpointer** (so state survives the pause).
2. A node that **marks state as interrupted** and returns; the graph engine halts and persists.

```python
from agentflow.core.state import AgentState

async def approve_refund(state: AgentState):
    amount = state.context[-1].content
    state.set_interrupt(reason=f"Approve refund of {amount}?")
    return {}  # halts here; checkpointer saves state under the current thread_id
```

When the next call comes in (with the same `thread_id` and the human's decision merged into state), check `state.is_interrupted()` / clear it with `state.clear_interrupt()` and continue. Use a checkpointer-backed flow so the pause survives process restarts.

## Callbacks (monitoring, validation, prompt-injection guards)

Register callbacks at compile time:

```python
app = graph.compile(
    checkpointer=checkpointer,
    callbacks=[
        on_tool_call(log_tool_usage),
        on_llm_request(block_prompt_injection),
    ],
)
```

Typical uses: request/response logging, PII scrubbing, blocking suspicious inputs, metrics emission.

## Event publishers (observability)

Publish execution events to Redis, Kafka, RabbitMQ, or a custom sink. Pick the extra you need:

```bash
pip install 10xscale-agentflow[redis]     # Redis pub/sub
pip install 10xscale-agentflow[kafka]
pip install 10xscale-agentflow[rabbitmq]
```

```python
from agentflow.publisher import RedisPublisher

publisher = RedisPublisher(url="redis://localhost:6379/0", channel="agent-events")
app = graph.compile(checkpointer=checkpointer, publisher=publisher)
```

Every node start/end, tool call, and LLM response is published — feed it into your observability stack.

## Serving over HTTP

`agentflow api` boots a FastAPI server exposing:

- `POST /invoke` — synchronous run.
- `POST /stream` — server-sent streaming.
- `GET /ping` — health check.
- `GET /docs` — OpenAPI.

Flags you'll reach for:

```bash
agentflow api --host 0.0.0.0 --port 8080 --no-reload
```

## Docker

```bash
agentflow build --python-version 3.13 --port 8080
# or with compose
agentflow build --docker-compose --service-name my-agent
```

The generated `Dockerfile` installs from `requirements.txt` (create one with `uv pip compile` or `pip freeze`) and runs `agentflow api`.

## Production checklist

- [ ] Replace `InMemoryCheckpointer` with `PgRedisCheckpointer`.
- [ ] Bind LLM/tool API keys via env vars — never hardcode.
- [ ] Add a prompt-injection callback before any tool call.
- [ ] Set a sensible `recursion_limit` in each invoke `config` to cap runaway loops.
- [ ] Wire a publisher (Redis/Kafka/OTEL) for observability.
- [ ] Run behind a real ASGI server (`gunicorn` with `uvicorn.workers.UvicornWorker`) — what `agentflow api` does by default in `--no-reload` mode.
- [ ] Use a persistent volume / managed DB for Postgres; don't rely on the Redis cache alone.

## Where to go next

- Integrating external tools in production? Load `tool-integrations`.
- Need a RAG / router scaffold? Load `prebuilt-patterns`.
- Starting from scratch? Load `agentflow` first.
