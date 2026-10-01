# Streaming and Token Usage

## Two streaming APIs

| API | Shape | When |
| --- | --- | --- |
| `agent.stream(input, config, stream_mode=[...], version="v2")` / `astream` | Every chunk is `{"type", "ns", "data"}` | Stable default for servers, SSE, and websockets |
| `agent.stream_events(input, config, version="v3")` / `await astream_events(...)` | A run object with typed projections: `.messages`, `.tool_calls`, `.values`, `.subagents`, `.subgraphs`, `.interrupts`, `.output` | New typed API recommended by the docs. In `langgraph` 1.2 it emits a `LangChainBetaWarning` (experimental), so pin versions and test upgrades |

Always pass `version=` explicitly. Without `version="v2"`, `stream()` returns the legacy v1 format, whose shape changes with the options.

## Stream modes (`stream`, `version="v2"`)

| Mode | `chunk["data"]` |
| --- | --- |
| `"updates"` | `{node_name: state_update}` after each step, for example `{"model": {"messages": [AIMessage]}}` or `{"tools": {...}}` |
| `"messages"` | `(message_chunk, metadata)` for each LLM token; `metadata["langgraph_node"]` names the node |
| `"custom"` | Whatever tools or nodes emit through the stream writer |
| `"values"` | The full state after each step |

```python
for chunk in agent.stream(
    {"messages": [{"role": "user", "content": "Weather in SF?"}]},
    config,
    context=ctx,
    stream_mode=["messages", "updates", "custom"],
    version="v2",
):
    if chunk["type"] == "messages":
        token, meta = chunk["data"]
        if meta["langgraph_node"] == "model" and token.text:
            send_to_client(token.text)                       # incremental text
    elif chunk["type"] == "updates":
        for node, update in chunk["data"].items():
            last = update["messages"][-1] if update and "messages" in update else None
            if node == "model" and last is not None and last.tool_calls:
                notify_tool_start(last.tool_calls)            # complete, parsed tool calls
    elif chunk["type"] == "custom":
        send_progress(chunk["data"])
```

- To show the final answer only, filter `messages` chunks from the `model` node that contain text and no `tool_call_chunks`.
- Reasoning/thinking tokens: iterate `token.content_blocks` and handle `block["type"] == "reasoning"`. Reasoning must be enabled on the model (Anthropic `thinking=...`, OpenAI `reasoning={...}`, Ollama `reasoning=True`, OpenRouter `reasoning={"effort": ...}`).
- Tool-call arguments stream as `tool_call_chunk` blocks. Use the `updates` mode to get the completed `tool_calls`.
- Emit progress from a tool with `runtime.stream_writer({"step": "fetched", "count": 10})` (or `langgraph.config.get_stream_writer()`). A tool that uses `get_stream_writer()` can only run inside a graph.
- When an agent is used inside a parent graph, pass `subgraphs=True` to see its tokens. `chunk["ns"]` identifies the nested graph.
- To stop a model from streaming tokens (for example an internal classifier), create it with `disable_streaming=True`.

## Event streaming (`stream_events`, `version="v3"`)

```python
stream = agent.stream_events({"messages": [{"role": "user", "content": "Weather in SF?"}]}, config, version="v3")

for kind, item in stream.interleave("messages", "tool_calls"):
    if kind == "messages":                 # one ChatModelStream per LLM call
        for delta in item.text:
            send_to_client(delta)
    elif kind == "tool_calls":             # tool execution lifecycle
        notify_tool(item.tool_name, item.input)
        result = item.output                # or item.error

final_state = stream.output               # drives the run to completion
if stream.interrupted:                    # human-in-the-loop pause
    pending = stream.interrupts
```

- Per message: `message.text` (deltas; `str(message.text)` for the full text), `message.reasoning`, `message.tool_calls` (chunks; `.get()` for the finalized calls), and `message.output` (the final `AIMessage`, including `usage_metadata`).
- `stream.subagents` yields named `create_agent` sub-agents (each has `.name`, `.messages`, `.tool_calls`, `.output`). `stream.subgraphs` covers all nested graphs.
- Async: `stream = await agent.astream_events(..., version="v3")`, then consume projections with `async for`. Use `asyncio.gather` to read several projections at once.

## Stream to a web client (FastAPI SSE)

```python
import json
from fastapi.responses import StreamingResponse

@app.post("/chat/{thread_id}")
async def chat(thread_id: str, body: ChatIn, user=Depends(current_user)):
    ensure_thread_owner(thread_id, user)
    config = {"configurable": {"thread_id": thread_id}}

    async def events():
        async for chunk in app.state.agent.astream(
            {"messages": [{"role": "user", "content": body.message}]},
            config, context=Context(user_id=user.id), stream_mode=["messages", "updates"], version="v2",
        ):
            if chunk["type"] == "messages":
                token, meta = chunk["data"]
                if meta["langgraph_node"] == "model" and token.text:
                    yield f"data: {json.dumps({'type': 'token', 'text': token.text})}\n\n"
        yield "data: {\"type\": \"done\"}\n\n"

    return StreamingResponse(events(), media_type="text/event-stream")
```

Handle client disconnects (cancellation stops the run), send errors as a final event rather than raising in the middle of the stream, and never stream raw tool outputs that may contain sensitive data.

## Streaming a model directly

```python
full = None
for chunk in model.stream("Explain vector clocks briefly"):
    print(chunk.text, end="", flush=True)
    full = chunk if full is None else full + chunk      # AIMessageChunk supports +
print(full.usage_metadata)
```

`model.astream_events(...)` yields `on_chat_model_start`, `on_chat_model_stream`, and `on_chat_model_end` events for fine-grained UIs.

## Token usage

Each `AIMessage` carries normalized usage when the provider reports it:

```python
msg.usage_metadata
# {"input_tokens": 8, "output_tokens": 304, "total_tokens": 312,
#  "input_token_details": {"cache_read": 0, "cache_creation": 0},
#  "output_token_details": {"reasoning": 256}}
```

Total usage for one agent run:

```python
from langchain.messages import AIMessage

def run_usage(messages) -> dict[str, int]:
    totals = {"input_tokens": 0, "output_tokens": 0, "total_tokens": 0}
    for m in messages:
        if isinstance(m, AIMessage) and m.usage_metadata:
            for k in totals:
                totals[k] += m.usage_metadata.get(k, 0)
    return totals
```

With a checkpointer, `result["messages"]` contains the whole thread. Sum only the messages added in this run (compare with the message count before the call, or use the callback below).

Usage across models, including summarization and sub-agent calls:

```python
from langchain_core.callbacks import get_usage_metadata_callback

with get_usage_metadata_callback() as cb:
    agent.invoke(input, config)
print(cb.usage_metadata)   # {"gpt-5.5-...": {"input_tokens": ..., "output_tokens": ..., ...}, ...}
```

Alternatively pass `UsageMetadataCallbackHandler()` in `config={"callbacks": [handler]}` for one run.

Provider notes:

- **OpenAI / Azure (Chat Completions)**: set `stream_usage=True` to receive usage while streaming. It defaults to off when a custom `OPENAI_BASE_URL` is set.
- **Anthropic**: usage is streamed by default. Cache reads and writes are reported in `input_token_details`.
- **DeepSeek**: reports token usage.
- **OpenRouter**: reports usage, including reasoning tokens and (with `cache_control` blocks) cache reads/writes. When streaming, read it from the aggregated output. Cost depends on the upstream provider OpenRouter routed to; reconcile with OpenRouter's dashboard rather than a fixed price table.
- **Ollama**: token usage is not reported by the integration. Estimate it or measure it elsewhere if needed.

For cost in currency, dashboards, and per-user or per-thread breakdowns, enable LangSmith tracing (see the `langsmith-tracing` skill). It computes cost from these token counts automatically for LangChain models. Add `thread_id` and user metadata to runs so costs can be grouped.
