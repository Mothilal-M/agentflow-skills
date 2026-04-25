---
name: checkpointing
description: Persistent state with InMemoryCheckpointer and PgCheckpointer; thread_id semantics
metadata:
  tags: checkpointing, persistence, postgres, redis, thread-id, sessions
---

A checkpointer persists graph state between invocations. Without one, every `invoke()` starts from a blank slate. With one, the same `thread_id` resumes the exact prior conversation, including pending tool calls and partial state.

## Imports

```python
from agentflow.storage.checkpointer import (
    BaseCheckpointer,
    InMemoryCheckpointer,
    PgCheckpointer,
)
```

There is **no** `PgRedisCheckpointer` — older snippets that import it are wrong. Use `PgCheckpointer`.

## InMemoryCheckpointer

```python
checkpointer = InMemoryCheckpointer()
app = graph.compile(checkpointer=checkpointer)
```

Demos and tests only. State is lost when the process exits and is **not** safe across worker processes.

## PgCheckpointer (production)

Install: `pip install 10xscale-agentflow[pg_checkpoint]`.

```python
from agentflow.storage.checkpointer import PgCheckpointer

checkpointer = PgCheckpointer(
    pg_dsn="postgresql://user:pass@localhost:5432/agentflow",
    redis_url="redis://localhost:6379/0",   # optional, used as a fast cache
)
app = graph.compile(checkpointer=checkpointer)
```

- **Postgres** stores the durable state.
- **Redis**, when configured, caches recent `thread_id`s so reads don't always hit the DB.

Schema is created on first use; you don't need to run a migration manually for the default tables.

## thread_id

Every `invoke()` and `astream()` requires a `thread_id` in `config`. It's the partition key for the checkpointer.

```python
app.invoke(input, config={"thread_id": "user-42-session-7"})
```

Conventions:
- One `thread_id` per active conversation. Don't reuse across users.
- Stable across restarts: the same value tomorrow resumes today's conversation.
- Use any string scheme that makes sense (`f"{user_id}:{convo_id}"` is common).

## Resuming an interrupted run

If a graph run is interrupted (cancelled, crash, deliberate pause), the checkpointer holds the last persisted state. Calling `invoke` / `astream` with the same `thread_id` continues from where it left off.

```python
# First call
app.invoke({"messages": [Message.text_message("Plan my week")]},
           config={"thread_id": "user-42"})

# Process restarts...

# This continues, doesn't restart
app.invoke({"messages": [Message.text_message("Add a Friday workout.")]},
           config={"thread_id": "user-42"})
```

## Inspecting prior state

`BaseCheckpointer` exposes async methods for reading:

```python
state = await checkpointer.get(thread_id="user-42")
history = await checkpointer.list(thread_id="user-42")
```

Useful for building a "show me my recent conversations" UI without re-running the graph.

## Custom checkpointers

Subclass `BaseCheckpointer` and implement `put` / `get` / `list` / `delete`. Useful for:
- DynamoDB / Firestore / SQLite for non-Postgres stacks.
- Encrypted-at-rest stores.
- Multi-tenant isolation by injecting a tenant key into the partition.

```python
class S3Checkpointer(BaseCheckpointer):
    async def put(self, thread_id, state): ...
    async def get(self, thread_id): ...
    # etc.
```

## Common mistakes

- **Forgetting `thread_id`.** The graph runs but state never persists; `invoke` may also raise depending on the checkpointer.
- **Reusing `thread_id` across users.** State leaks. Always include the user identifier.
- **Using `InMemoryCheckpointer` behind multiple workers.** Each worker has its own dict — users hit different state on different requests.

## See also

- [graph.md](./graph.md) — passing `checkpointer=` to `compile()`.
- [streaming.md](./streaming.md) — how cancellation interacts with checkpointed state.
