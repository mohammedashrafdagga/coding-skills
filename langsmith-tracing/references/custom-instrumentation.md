# Custom Instrumentation

Use these tools for code that LangChain does not trace automatically: request handlers, business steps, direct provider SDK calls, retrieval code, and background jobs.

## `@traceable`

```python
from langsmith import traceable, get_current_run_tree


@traceable(name="POST /chat", run_type="chain", tags=["api"])
async def handle_chat(request: ChatRequest, user: User) -> ChatResponse:
    rt = get_current_run_tree()
    if rt is not None:                         # None when tracing is disabled
        rt.metadata.update({"user_id": user.pseudonymous_id, "thread_id": request.thread_id})
    docs = await retrieve(request.message)
    result = await agent.ainvoke(
        {"messages": [{"role": "user", "content": request.message}]},
        {"configurable": {"thread_id": request.thread_id}},   # the agent run nests under this trace
    )
    return ChatResponse(text=result["messages"][-1].text)


@traceable(run_type="retriever")
async def retrieve(query: str) -> list[dict]:
    hits = await vector_index.search(query, k=5)
    return [{"page_content": h.text, "type": "Document", "metadata": {"source": h.source}} for h in hits]
```

- Nested traced calls nest automatically, for both sync and async functions and for generators, whose output is aggregated.
- `run_type` is one of `chain` (default), `llm`, `tool`, `retriever`, `prompt`, `embedding`, or `parser`. The LLM and retriever types get special rendering, and `llm` runs get token and cost accounting.
- Decorator options: `name`, `run_type`, `tags`, `metadata`, `project_name`, `client`, `process_inputs` / `process_outputs` (functions that transform what is recorded), and `reduce_fn` (for generators).
- Per-call options: `fn(..., langsmith_extra={"tags": [...], "metadata": {...}, "run_id": uuid, "parent": run_tree_or_headers, "project_name": "..."})`.
- `get_current_run_tree()` returns `None` when tracing is off. Guard every use of it.
- Record only what is useful. Use `process_inputs=lambda inputs: {"query": inputs["query"]}` to drop large or sensitive arguments such as file bytes, database sessions, or credentials.

## `trace` context manager

Use it for a block of code instead of a whole function:

```python
import langsmith as ls

with ls.trace("rerank", run_type="chain", inputs={"n": len(candidates)}, metadata={"model": "bge-reranker"}) as rt:
    ranked = reranker.rank(query, candidates)
    rt.end(outputs={"top_ids": [c.id for c in ranked[:5]]})
```

`tracing_context(...)` does not create a run. It sets defaults (project, tags, metadata, parent, enabled, client) for the runs created inside it.

## Direct provider SDK calls

```python
import anthropic
import openai
from langsmith.wrappers import wrap_anthropic, wrap_openai

oai = wrap_openai(openai.OpenAI(), tracing_extra={"tags": ["raw-sdk"]})
claude = wrap_anthropic(anthropic.Anthropic())

oai.chat.completions.create(model="gpt-5.4-mini", messages=msgs,
                            langsmith_extra={"metadata": {"thread_id": tid}})
```

Wrapped clients record model, parameters, messages, token usage, and cost automatically.

## Custom LLM spans with tokens and cost

For models called through your own HTTP client:

```python
from langsmith import traceable, get_current_run_tree


@traceable(run_type="llm", metadata={"ls_provider": "acme", "ls_model_name": "acme-large-2"})
def call_acme(messages: list[dict]) -> dict:
    resp = acme_client.chat(messages)
    if (rt := get_current_run_tree()) is not None:
        rt.set(usage_metadata={
            "input_tokens": resp.usage.prompt,
            "output_tokens": resp.usage.completion,
            "total_tokens": resp.usage.prompt + resp.usage.completion,
        })
    return {"role": "assistant", "content": resp.text}   # message-shaped output renders nicely
```

Costs are computed when a price exists for `ls_model_name` in the LangSmith model price map. For non-linear pricing, send the costs directly (see the cost-tracking docs). You can also attach costs to non-LLM runs, such as paid tool APIs.

## Concurrency

- Async functions and `asyncio.gather` keep context (Python 3.11+).
- Threads: use `from langsmith.utils import ContextThreadPoolExecutor` instead of `concurrent.futures.ThreadPoolExecutor`, or pass `langsmith_extra={"parent": get_current_run_tree()}` into work run on threads.
- Background tasks started after the response is sent should either start their own trace (with the same `thread_id` metadata) or pass the parent explicitly.

## Distributed tracing across services

Only accept tracing headers from trusted internal services. Never from the public internet.

```python
# caller
@traceable
async def call_worker(payload: dict):
    headers = {}
    if (rt := get_current_run_tree()) is not None:
        headers.update(rt.to_headers())          # langsmith-trace and baggage headers
    return await http.post("http://worker/run", json=payload, headers=headers)


# internal worker (FastAPI or Starlette)
from langsmith.middleware import TracingMiddleware
app.add_middleware(TracingMiddleware)            # continues the trace from the headers

# or, manually:
with ls.tracing_context(parent=request.headers):
    process(payload)
```

For LangChain runnables in the receiving service, wrap the call in `ls.tracing_context(parent=headers)`. Strip `langsmith-trace` and `baggage` headers from external traffic at the gateway.

## Low-level RunTree API

`langsmith.RunTree` lets you post and end runs manually (`run.post()`, `child = run.create_child(...)`, `run.end(outputs=...)`, `run.patch()`). Use it only when the decorator and context managers cannot express the control flow, for example for callbacks from a queue.
