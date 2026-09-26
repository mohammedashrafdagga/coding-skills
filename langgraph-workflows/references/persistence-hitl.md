# Persistence, Memory, and Human-in-the-Loop

## Checkpointers and stores

- **Checkpointer**: saves the full graph state at every super-step, per `thread_id`. It enables conversation memory, `interrupt()`, resume after a crash, state inspection, and time travel. Compile with `builder.compile(checkpointer=...)` and always pass `config={"configurable": {"thread_id": ...}}`.
- **Store**: application-defined JSON documents shared across threads (long-term memory). Compile with `builder.compile(store=...)`. Nodes access it as `runtime.store`, and tools as `ToolRuntime.store`.

| Backend | Checkpointer | Store | Package |
| --- | --- | --- | --- |
| Tests | `InMemorySaver` (`langgraph.checkpoint.memory`) | `InMemoryStore` (`langgraph.store.memory`) | included |
| Local / single process | `SqliteSaver`, `AsyncSqliteSaver` (`langgraph.checkpoint.sqlite[.aio]`) | `SqliteStore`, `AsyncSqliteStore` (`langgraph.store.sqlite[.aio]`) | `langgraph-checkpoint-sqlite` |
| Production | `PostgresSaver`, `AsyncPostgresSaver` (`langgraph.checkpoint.postgres[.aio]`) | `PostgresStore`, `AsyncPostgresStore` (`langgraph.store.postgres[.aio]`) | `langgraph-checkpoint-postgres`, `psycopg[binary,pool]` |

```python
from langgraph.checkpoint.postgres import PostgresSaver
from langgraph.store.postgres import PostgresStore

with PostgresSaver.from_conn_string(DB_URI) as checkpointer, PostgresStore.from_conn_string(DB_URI) as store:
    checkpointer.setup()     # once per database (migration step)
    store.setup()
    graph = builder.compile(checkpointer=checkpointer, store=store)
    graph.invoke(inputs, {"configurable": {"thread_id": thread_id}}, context=Ctx(user_id=uid))
```

- In async apps, open `AsyncPostgresSaver` / `AsyncPostgresStore` in the application lifespan, keep them open, and compile the graph once. For concurrency, pass a `psycopg_pool.AsyncConnectionPool` opened with `kwargs={"autocommit": True, "prepare_threshold": 0, "row_factory": dict_row}`.
- SQLite: `SqliteSaver.from_conn_string("data/graph.sqlite")`. Do not share one SQLite file between several processes under write load.
- On LangSmith Agent Server deployments, persistence is provided automatically. Do not pass a checkpointer or store to graphs exported in `langgraph.json`.
- `durability="exit" | "async" | "sync"` on `invoke`/`stream` trades write overhead against crash recovery. `"sync"` is the safest; `"exit"` only saves at the end or at an interrupt.
- Security: checkpoints contain full state. Keep secrets out of state. Use `EncryptedSerializer` (`langgraph.checkpoint.serde.encrypted`) when state holds sensitive data. Check that the requesting user owns a `thread_id` before using it. Prune old threads (`checkpointer.delete_thread(thread_id)`).

Short-term message management (trim, summarize, delete with `RemoveMessage`) and long-term memory design (namespaces, semantic search, memory types, hot-path versus background writes) are the same as in the `langchain-agents` memory reference. In a raw graph, do trimming or summarization in a node before the model call, or embed a `create_agent` that uses `SummarizationMiddleware`.

## Human-in-the-loop with `interrupt()`

```python
from typing import Literal
from langgraph.types import Command, interrupt


def approve_refund(state: RefundState) -> Command[Literal["issue_refund", "notify_denied"]]:
    decision = interrupt({                      # JSON-serializable payload shown to the reviewer
        "type": "refund_approval",
        "order_id": state["order_id"],
        "amount": state["amount"],
    })
    if decision.get("approved"):
        return Command(update={"approved_amount": decision.get("amount", state["amount"])}, goto="issue_refund")
    return Command(update={"denial_reason": decision.get("reason", "")}, goto="notify_denied")


graph = builder.compile(checkpointer=checkpointer)
config = {"configurable": {"thread_id": f"refund-{order_id}"}}

result = graph.invoke({"order_id": order_id, "amount": 120.0}, config)
if "__interrupt__" in result:
    payload = result["__interrupt__"][0].value      # persist or show to a reviewer; the graph waits indefinitely

# Later, possibly in another process or request:
final = graph.invoke(Command(resume={"approved": True, "amount": 100.0}), config)
```

Ways to detect interrupts:

- `invoke(...)` returns the state with an `"__interrupt__"` key.
- `invoke(..., version="v2")` returns a `GraphOutput` with `.value` and `.interrupts`.
- `stream_events(..., version="v3")` has `.interrupted` and `.interrupts`. Drive it with `.output` first.
- `graph.get_state(config).next` is non-empty, and `.tasks[i].interrupts` lists the pending interrupts.

Rules:

1. A checkpointer and the **same** `thread_id` are required for pausing and resuming.
2. On resume, the interrupted **node reruns from its first line**. `interrupt()` then returns the resume value. Code before `interrupt()` runs twice, so make it idempotent, or move side effects after the interrupt or into another node.
3. Never wrap `interrupt()` in a bare `try/except`, because it pauses by raising a special exception. Keep the order of several `interrupt()` calls in one node fixed, and do not make them conditional on values that change between runs.
4. Interrupt payloads and resume values must be JSON-serializable.
5. Parallel branches can interrupt together. Resume them all at once with `Command(resume={interrupt.id: value, ...})`.
6. Validation loops: call `interrupt()` inside a `while` loop until the human input validates, changing the question each time.
7. Interrupts inside tools pause before the side effect. For standard tool approval in `create_agent`, use `HumanInTheLoopMiddleware` instead.
8. Only `Command(resume=...)` is valid as input when resuming. Start a new turn with a plain input dict.

Static breakpoints for debugging only: `compile(interrupt_before=["node"], interrupt_after=[...])`. Continue with `graph.invoke(None, config)`.

## Inspect, edit, and time travel

```python
snapshot = graph.get_state(config)                    # values, next, config, metadata, created_at, parent_config, tasks
history = list(graph.get_state_history(config))       # newest first

# Fork from an earlier checkpoint with modified state, then continue from there:
before_review = next(s for s in history if s.next == ("review",))
fork_config = graph.update_state(before_review.config, {"draft": "Edited draft"})
graph.invoke(None, fork_config)                        # re-runs from that point on a new branch

# Replay without changes: invoke with a checkpoint_id
graph.invoke(None, {"configurable": {"thread_id": tid, "checkpoint_id": before_review.config["configurable"]["checkpoint_id"]}})
```

- `update_state(config, values, as_node="node")` applies values through reducers as though the given node had produced them, and so controls what runs next.
- Replaying re-executes the later nodes, including model calls, side effects, and interrupts. Guard side effects accordingly.
- Use `get_state(config, subgraphs=True)` to see the state of interrupted subgraphs.
