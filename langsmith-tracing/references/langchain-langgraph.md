# Tracing LangChain and LangGraph

With `LANGSMITH_TRACING=true`, every `create_agent` run, `StateGraph` run, chat model call, tool call, retriever, and middleware hook is traced automatically, with correct nesting. Most of the work is naming runs well and attaching the right metadata.

## Attach run name, tags, and metadata

Pass them in the `config` of the top-level call. Tags and metadata are inherited by every child run. `run_name` applies only to the run you invoke.

```python
config = {
    "run_name": "support_chat_turn",
    "tags": ["support-bot", f"env:{settings.env}", f"release:{settings.version}"],
    "metadata": {
        "user_id": user.pseudonymous_id,     # never raw emails or tokens
        "tenant_id": user.tenant_id,
        "request_id": request_id,
        "channel": "web",
    },
    "configurable": {"thread_id": conversation_id},
}
result = agent.invoke({"messages": [{"role": "user", "content": text}]}, config, context=ctx)
```

- **Threads**: LangSmith groups traces into a thread by the `thread_id` (or `session_id`) metadata key. LangGraph copies `configurable.thread_id` into the metadata of every child run automatically, so using the same `thread_id` for persistence and tracing makes the conversation appear as one thread, with thread-level token and cost totals. For code outside LangGraph, put `thread_id` in `metadata` yourself, on every run.
- Static configuration on an object: `model.with_config({"tags": ["primary-llm"], "metadata": {...}, "run_name": "..."})`, and `graph.with_config(run_name="refund_workflow")`.
- Name agents and graphs: `create_agent(..., name="billing_agent")` and `builder.compile(name="refund_workflow")`. Names show up on runs and in multi-agent traces (`lc_agent_name` metadata).
- Custom run ID: `config={"run_id": uuid}` on the root call makes it the trace ID. This is useful for linking a trace to your request logs or attaching feedback later.

## Model naming for cost and filtering

LangChain chat models report the provider and model automatically, and LangSmith computes costs for known models. For local, self-hosted, or proxied models, set metadata so cost and filters work:

```python
from langchain_ollama import ChatOllama

llm = ChatOllama(model="llama3.1:8b", metadata={"ls_provider": "ollama", "ls_model_name": "llama3.1-8b-onprem"})
```

`ChatOpenRouter` reports `model_provider="openrouter"` and the routed slug (for example `anthropic/claude-sonnet-4.5`). LangSmith may not price slugs it does not recognize; add model prices for the slugs you use or rely on OpenRouter's own billing. OpenRouter's `session_id` and `trace` parameters feed OpenRouter's broadcast feature and are independent of LangSmith; for LangSmith, keep passing `thread_id` and metadata through the run config.

Add model prices for custom `ls_model_name` values in the LangSmith model price map.

## Trace selectively

```python
import langsmith as ls

with ls.tracing_context(enabled=True):          # trace this call even if LANGSMITH_TRACING is off
    agent.invoke(inputs, config)

with ls.tracing_context(enabled=False):         # never trace this call (for example health checks or PII-heavy flows)
    agent.invoke(inputs, config)
```

Decide per request, for example trace only opted-in tenants or only a debug header: `with ls.tracing_context(enabled=should_trace(request)): ...`.

## Reduce noise and payload size

- Middleware spans record their inputs and outputs by default. For middleware whose inputs are the whole message history, set `trace_policy = TracePolicy(process_inputs=omit_payload)` on the class, or globally with `configure_trace_policy(TracePolicy(process_inputs=omit_payload))` (from `langchain.agents.middleware`, `langchain>=1.3.15`).
- Keep state small. LangGraph node runs record the state they receive and the updates they return.
- Do not trace large binary data. Store it externally and keep references in state.

## Mixing with custom code

LangChain runs invoked inside a `@traceable` function become its children, and `@traceable` functions called from inside a LangGraph node or tool nest under that node. So you can wrap an HTTP handler with `@traceable(name="POST /chat")` and get one trace that contains the whole agent run. See `custom-instrumentation.md`.

## Streaming, async, and threads

- `stream`, `astream`, and `stream_events` produce the same trace as `invoke`.
- Async code propagates context automatically on Python 3.11+. On 3.10, pass `config` explicitly into nested runnables.
- Work submitted to a plain `ThreadPoolExecutor` loses the trace context. Use `langsmith.utils.ContextThreadPoolExecutor`, or pass `langsmith_extra={"parent": ls.get_current_run_tree()}` to traceable functions.

## Debugging checklist

| Symptom | Likely cause |
| --- | --- |
| No traces at all | `LANGSMITH_TRACING` not `true` in the process environment, wrong key or region (`LANGSMITH_ENDPOINT`), multi-workspace key without `LANGSMITH_WORKSPACE_ID`, or the process exited before flushing |
| Traces in `default` | `LANGSMITH_PROJECT` not set in the environment of the running process (for example set in a settings class but not exported) |
| Child runs appear as separate traces | Context lost across threads, processes, or services; use `ContextThreadPoolExecutor` or distributed tracing headers |
| Thread view is missing turns or token totals | `thread_id` not passed in `configurable` (LangGraph) or not set on every custom run |
| No cost shown | Unknown model name or provider; set `ls_model_name` / `ls_provider` and a model price, or send usage manually |
| Secrets or PII visible | Masking not configured; see `production.md` |
