---
name: streaming
description: invoke vs astream, event types, deltas, and forwarding to WebSocket / SSE
metadata:
  tags: streaming, async, astream, invoke, sse, websocket
---

A compiled graph (`app = graph.compile()`) exposes both sync and async entry points.

## `invoke(input, config)` — sync

Blocks until the graph reaches `END`. Returns the final state dict.

```python
result = app.invoke(
    {"messages": [Message.text_message("plan my week")]},
    config={"thread_id": "user-1"},
)
for m in result["messages"]:
    print(f"{m.role}: {m.content}")
```

Best for: scripts, batch jobs, simple HTTP request/response.

## `astream(input, config)` — async streaming

Yields `StreamChunk` events as the graph progresses. Each chunk carries either an LLM token, a state update, a node start/end marker, or a tool call.

```python
import asyncio
from agentflow.core.state import StreamChunk, StreamEvent

async def run():
    async for chunk in app.astream(input, config={"thread_id": "1"}):
        if chunk.event == StreamEvent.LLM_DELTA:
            print(chunk.delta, end="", flush=True)
        elif chunk.event == StreamEvent.NODE_END:
            print(f"\n[node {chunk.node} done]")

asyncio.run(run())
```

`StreamEvent` enum members typically include:
- `NODE_START` / `NODE_END` — graph node boundaries.
- `LLM_DELTA` — incremental token from the LLM.
- `TOOL_CALL` — the LLM requested a tool.
- `TOOL_RESULT` — the tool returned.
- `STATE_UPDATE` — state was merged.
- `ERROR` — something failed.

The exact set depends on the AgentFlow version; iterate and `print(chunk.event)` once to see what your version emits.

## Streaming over HTTP

### Server-Sent Events (FastAPI)

```python
from fastapi import FastAPI
from fastapi.responses import StreamingResponse

api = FastAPI()

@api.post("/stream")
async def stream_endpoint(payload: dict):
    async def event_source():
        async for chunk in app.astream(payload, config={"thread_id": payload["thread_id"]}):
            if chunk.event == StreamEvent.LLM_DELTA:
                yield f"data: {chunk.delta}\n\n"
        yield "event: done\ndata: {}\n\n"
    return StreamingResponse(event_source(), media_type="text/event-stream")
```

`agentflow api` (the CLI server) already exposes `/stream` with this shape — only roll your own when you need a different protocol.

### WebSockets

```python
@api.websocket("/chat")
async def chat(ws: WebSocket):
    await ws.accept()
    payload = await ws.receive_json()
    async for chunk in app.astream(payload, config={"thread_id": payload["thread_id"]}):
        await ws.send_json({"event": str(chunk.event), "data": chunk.model_dump()})
    await ws.close()
```

## Cancellation

Cancel an in-flight `astream` by cancelling the asyncio task. The graph engine catches `asyncio.CancelledError` and persists state up to the last completed node (if a checkpointer is set), so the user can resume.

```python
task = asyncio.create_task(consume(app.astream(...)))
# later:
task.cancel()
```

## When to pick which

| Scenario | Use |
|----------|-----|
| Script / batch job / one-shot CLI | `invoke` |
| Web app showing tokens as they arrive | `astream` + SSE |
| Voice / phone app needing low TTFT | `astream` |
| Long-running multi-tool plan with progress UI | `astream` (watch `NODE_START` / `TOOL_CALL`) |
| Server-side automation, no UI | `invoke` |

## See also

- [graph.md](./graph.md) — `compile()` and config keys (`thread_id`, `recursion_limit`).
- [checkpointing.md](./checkpointing.md) — persistence so cancelled streams can resume.
