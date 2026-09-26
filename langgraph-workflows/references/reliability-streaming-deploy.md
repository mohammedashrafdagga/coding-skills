# Reliability, Streaming, Testing, and Deployment

## Fault tolerance (langgraph >= 1.2)

The order is fixed: an attempt fails (including by timeout) → the retry policy decides whether to retry → after retries are exhausted, the error handler runs → if there is no handler, the exception propagates.

```python
from langgraph.errors import NodeError
from langgraph.types import Command, RetryPolicy, TimeoutPolicy, default_retry_on


def retry_transient(exc: BaseException) -> bool:
    if isinstance(exc, PaymentDeclined):   # business errors must not be retried
        return False
    return default_retry_on(exc)


def compensate(state: OrderState, error: NodeError) -> Command:
    log.warning("node %s failed: %r", error.node, error.error)
    return Command(update={"status": "payment_failed"}, goto="release_inventory")   # saga compensation


builder = StateGraph(OrderState)
builder.set_node_defaults(retry_policy=RetryPolicy(max_attempts=3))       # graph-wide defaults
builder.add_node(
    "charge_payment", charge_payment,                                     # async def for timeouts
    retry_policy=RetryPolicy(max_attempts=4, initial_interval=0.5, backoff_factor=2.0, max_interval=10,
                             retry_on=retry_transient),
    timeout=TimeoutPolicy(run_timeout=30, idle_timeout=10),               # or timeout=30
    error_handler=compensate,
)
```

- `RetryPolicy` defaults: `max_attempts=3`, `initial_interval=0.5`, `backoff_factor=2.0`, `max_interval=128`, `jitter=True`. `default_retry_on` does not retry `ValueError`, `TypeError`, `LookupError`, `RuntimeError`, `OSError`, and other programming errors. For `requests`/`httpx` errors it retries only 5xx responses. `NodeTimeoutError` is retryable.
- Timeouts apply only to `async def` nodes. A sync node with `timeout=` fails at compile time; wrap blocking I/O in `asyncio.to_thread`.
- Error handlers take `(state)`, `(state, runtime)`, or `(state, error: NodeError)`, and return an update or a `Command`. Failure details are checkpointed, so handlers behave the same after a resume.
- Model-level retries (`max_retries` on the chat model, `ModelRetryMiddleware` in agents) are separate from node retries. Avoid stacking both on the same call without a reason, because the attempts multiply.
- Cap loops with `recursion_limit`, and catch `GraphRecursionError` at the application boundary.
- Graceful shutdown: create `control = RunControl()` (`from langgraph.runtime import RunControl`), pass `control=control` to `invoke`/`stream`, and call `control.request_drain("sigterm")` from a signal handler to stop at a super-step boundary. Resume later from the checkpoint.

## Streaming

`graph.stream(input, config, stream_mode=[...], version="v2")` (or `astream`) yields `{"type", "ns", "data"}`:

| Mode | Data |
| --- | --- |
| `values` | Full state after each step |
| `updates` | `{node: update}` after each node |
| `messages` | `(token_chunk, metadata)` from every LLM call inside nodes; `metadata["langgraph_node"]` and `tags` let you filter |
| `custom` | Anything written with `runtime.stream_writer(...)` or `get_stream_writer()` |
| `checkpoints`, `tasks`, `debug` | Checkpoint and task lifecycle events (need a checkpointer) |

- Pass `subgraphs=True` to include nested graphs. `chunk["ns"]` holds the namespace path.
- Filter tokens by node (`metadata["langgraph_node"] == "writer"`), or by tags set on a model (`llm.with_config(tags=["final_answer"])`, then check `"final_answer" in metadata["tags"]`).
- LLM calls inside nodes stream tokens automatically in `messages` mode, even when the node calls `model.invoke()`.
- Typed event streaming: `graph.stream_events(input, config, version="v3")` returns `.messages`, `.values`, `.subgraphs`, `.interrupts`, `.interrupted`, and `.output`; `.interleave(...)` merges projections in sync code. It is marked experimental in `langgraph` 1.2 (`LangChainBetaWarning`), so pin versions.
- An interactive loop with interrupts: stream, then check `interrupted` (v3) or `__interrupt__`, collect input, and stream again with `Command(resume=...)` until finished.

## Testing

```python
import pytest
from langgraph.checkpoint.memory import InMemorySaver


def build_graph():
    ...  # returns an uncompiled StateGraph; the same factory is used by the app


def test_happy_path():
    graph = build_graph().compile(checkpointer=InMemorySaver())   # fresh saver per test
    out = graph.invoke({"topic": "x"}, {"configurable": {"thread_id": "t1"}}, context=Ctx(user_id="u"))
    assert out["report"]


def test_single_node():
    graph = build_graph().compile()
    assert graph.nodes["classify"].invoke({"question": "refund please"})["destination"] == "billing"


def test_interrupt_resume():
    graph = build_graph().compile(checkpointer=InMemorySaver())
    cfg = {"configurable": {"thread_id": "t2"}}
    first = graph.invoke({"order_id": "o1", "amount": 10.0}, cfg)
    assert first["__interrupt__"][0].value["order_id"] == "o1"
    final = graph.invoke(Command(resume={"approved": False, "reason": "fraud"}), cfg)
    assert final["denial_reason"] == "fraud"


def test_partial_path():
    graph = build_graph().compile(checkpointer=InMemorySaver())
    cfg = {"configurable": {"thread_id": "t3"}}
    graph.update_state(cfg, {"draft": "seeded"}, as_node="write_draft")   # start mid-graph
    out = graph.invoke(None, cfg, interrupt_after=["review"])            # stop after the node under test
```

- Inject models through a factory or `context` so that tests can pass `GenericFakeChatModel` (subclass it with `bind_tools` returning `self` when tools are bound). Script structured-output responses as tool calls or JSON text, depending on the method used.
- Test every conditional edge, the retry and error-handler paths (raise from a stub node), and parallel branches (reducer results).
- Keep live-model tests (with LangSmith evaluations when available) separate from unit tests.

## Project layout and local server

```text
my-app/
├── src/my_agent/
│   ├── state.py        # State/Context schemas
│   ├── nodes.py        # node functions (pure logic; I/O via injected clients)
│   ├── tools.py        # @tool definitions
│   ├── graph.py        # build_graph() and `graph = build_graph().compile()`
│   └── __init__.py
├── tests/
├── .env.example
├── langgraph.json
└── pyproject.toml
```

```json
{
  "dependencies": ["."],
  "graphs": {
    "my_agent": "./src/my_agent/graph.py:graph"
  },
  "env": "./.env"
}
```

- `graphs` maps a name to `path:variable`. The variable is a compiled graph, or a function returning one. Add `"store": {"index": {...}}` to enable semantic search in the platform store.
- Local development server and Studio: `uv add --dev "langgraph-cli[inmem]"`, then run `langgraph dev`. It serves the API on `http://127.0.0.1:2024` and opens LangSmith Studio for visual debugging, with in-memory persistence.
- In a self-managed service (for example FastAPI), compile the graph once at startup with a PostgreSQL checkpointer and store, and expose `invoke`/`astream` behind authenticated endpoints that check thread ownership.
- On LangSmith Deployment (Agent Server), the platform supplies persistence, task queues, streaming endpoints, and cron jobs. Do not compile deployed graphs with your own checkpointer.
