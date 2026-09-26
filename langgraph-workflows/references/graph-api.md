# Graph API

## Minimal graph

```python
from typing_extensions import TypedDict
from langgraph.graph import StateGraph, START, END


class State(TypedDict):
    question: str
    answer: str


def answer(state: State) -> dict:
    return {"answer": f"You asked: {state['question']}"}


graph = (
    StateGraph(State)
    .add_node("answer", answer)
    .add_edge(START, "answer")
    .add_edge("answer", END)
    .compile()               # compile(checkpointer=..., store=..., cache=..., name=...)
)
graph.invoke({"question": "hi"})
```

You must compile the graph before using it. Compiling validates the structure (no orphaned nodes, valid edges) and attaches runtime features.

## State

- Use `TypedDict` for the schema (fast and documented), `dataclass` when you need default values, or a Pydantic `BaseModel` for recursive validation (slower). LangChain's `create_agent` state must be a `TypedDict` subclass of `AgentState`.
- The input and output schemas default to the state schema. Narrow the graph's public interface with `StateGraph(State, context_schema=Ctx, input_schema=In, output_schema=Out)`. Nodes can also read and write private keys: annotate a node's input type with another `TypedDict`.
- **Reducers** decide how updates merge. Without a reducer, the new value replaces the old one.

```python
import operator
from typing import Annotated
from langchain.messages import AnyMessage
from langgraph.graph.message import add_messages
from langgraph.types import Overwrite


class ResearchState(TypedDict):
    messages: Annotated[list[AnyMessage], add_messages]   # append; same-ID replaces; RemoveMessage deletes
    findings: Annotated[list[str], operator.add]          # parallel workers append safely
    errors: Annotated[list[str], operator.add]
    report: str                                          # last writer wins


def reset_errors(state: ResearchState) -> dict:
    return {"errors": Overwrite([])}   # returning [] would be merged in, not clear the list
```

- `MessagesState` (from `langgraph.graph`) is a ready-made `messages` key with `add_messages`. Subclass it to add fields: `class State(MessagesState): documents: list[str]`.
- `add_messages` accepts message objects or dicts such as `{"role": "user", "content": "..."}`.
- Keep state serializable (checkpointers persist it) and small. Store large blobs (files, documents) externally and keep references in state.
- Changing a state key's type incompatibly or renaming a key affects threads that already have checkpoints. Adding and removing keys is safe. For behavioral changes to existing threads, pin a version field in state.

## Nodes

A node is a sync or async function that takes `state` and optionally `config: RunnableConfig` and/or `runtime: Runtime[Ctx]`. It returns a partial update dict, a `Command`, or `None`.

```python
from dataclasses import dataclass
from langgraph.runtime import Runtime


@dataclass
class Ctx:
    user_id: str
    llm_model: str = "openai:gpt-5.5"


async def draft(state: ResearchState, runtime: Runtime[Ctx]) -> dict:
    model = get_model(runtime.context.llm_model)            # cache model instances; do not build per call
    memories = await runtime.store.asearch((runtime.context.user_id, "prefs"), limit=5) if runtime.store else []
    runtime.stream_writer({"stage": "drafting"})             # visible with stream_mode="custom"
    reply = await model.ainvoke(state["messages"])
    return {"messages": [reply]}


builder = StateGraph(ResearchState, context_schema=Ctx)
builder.add_node("draft", draft)                              # name defaults to the function name
graph = builder.compile()
await graph.ainvoke({"messages": [...]}, context=Ctx(user_id="u1"))
```

`Runtime` provides `context`, `store`, `stream_writer`, `execution_info` (for example `thread_id`), `server_info`, `heartbeat`, `previous`, and `control`.

`add_node` options: `retry_policy=`, `timeout=` (async nodes only), `error_handler=`, `cache_policy=`, `input_schema=`, `destinations=` (for rendering `Command` targets), `defer=True` (wait until all pending branches finish, useful for fan-in with branches of different lengths), and `metadata=`. `builder.add_sequence([step_1, step_2, step_3])` adds a linear chain.

Nodes can re-execute (retries, resume after interrupt or crash). Put side effects behind idempotency keys, or wrap sub-operations in `@task` from `langgraph.func` so their results are checkpointed and skipped on resume.

## Edges

```python
from typing import Literal

builder.add_edge(START, "classify")                     # entry point
builder.add_edge("a", "b")                              # always a -> b
builder.add_edge(["b", "c"], "join")                    # join waits for both b and c


def route(state: State) -> Literal["billing", "tech", "__end__"]:
    return state["category"] if state["category"] in ("billing", "tech") else "__end__"


builder.add_conditional_edges("classify", route)        # the return value is a node name (or list, or END)
builder.add_conditional_edges("classify", route_fn, {"yes": "act", "no": END})   # map labels to nodes
```

- A node with several outgoing static edges runs all targets in parallel in the next super-step.
- Routing functions must be pure reads of state. Put state changes in nodes.
- Type routing return values with `Literal[...]`, or pass the list of destinations, so the graph renders correctly.

## Send: dynamic fan-out (map-reduce)

```python
from langgraph.types import Send


def fan_out(state: OverallState) -> list[Send]:
    return [Send("summarize_doc", {"doc": d, "query": state["query"]}) for d in state["docs"]]


builder.add_conditional_edges("plan", fan_out, ["summarize_doc"])
builder.add_edge("summarize_doc", "combine")   # combine runs after all Sends finish
```

Each `Send` runs the target node with its own input (which may use a different schema from the parent state). Workers write results to a key with a reducer. `Send(node, arg, timeout=...)` sets a per-branch timeout.

## Command: update and route together

```python
from typing import Literal
from langgraph.types import Command


def review(state: State) -> Command[Literal["publish", "revise"]]:
    if state["score"] >= 8:
        return Command(update={"status": "approved"}, goto="publish")
    return Command(update={"feedback": critique(state)}, goto="revise")
```

- `goto` accepts a node name, a list, or `Send` objects. From inside a subgraph, `Command(goto="node_in_parent", graph=Command.PARENT, update=...)` jumps to the parent graph; every shared key updated that way needs a reducer in the parent state.
- Tools can return `Command(update=...)` (and `goto`) to change graph state.
- `Command(resume=...)` is only for resuming interrupts. It is never used as a node return value for routing.

## Limits and loops

- `config={"recursion_limit": 50}` caps super-steps per run (default 1000). The run raises `GraphRecursionError` when the cap is exceeded. It is a standalone config key, not inside `configurable`.
- Degrade gracefully by adding `remaining_steps: RemainingSteps` (from `langgraph.managed`) to state and routing to a fallback when it is low.
- The current step is available in `config["metadata"]["langgraph_step"]`.

## Caching

```python
from langgraph.cache.memory import InMemoryCache
from langgraph.types import CachePolicy

builder.add_node("embed", embed_node, cache_policy=CachePolicy(ttl=300))   # key defaults to a hash of the input
graph = builder.compile(cache=InMemoryCache())
```

Cache only pure, deterministic nodes. Never cache nodes with side effects or user-specific data under shared keys.

## Functional API (alternative)

`@entrypoint(checkpointer=...)` plus `@task` from `langgraph.func` expresses a workflow as ordinary Python control flow with checkpointed tasks, interrupts, and streaming. Use it when the logic is naturally sequential code; use `StateGraph` when you want explicit topology, visualization, and parallel branches. Keep the order of tasks and interrupts deterministic between runs, because resume matches cached results by order.

## Visualize

```python
print(graph.get_graph().draw_mermaid())       # Mermaid text for docs and pull requests
graph.get_graph(xray=True)                    # include subgraph internals
```
