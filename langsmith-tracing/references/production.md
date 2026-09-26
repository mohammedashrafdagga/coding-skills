# Production Tracing

## Protect sensitive data

Traces store prompts, tool arguments and results, retrieved documents, state, and outputs. Decide what may leave the process before enabling production tracing.

| Need | Mechanism |
| --- | --- |
| Hide all inputs/outputs | `LANGSMITH_HIDE_INPUTS=true`, `LANGSMITH_HIDE_OUTPUTS=true` (metadata still sent) |
| Transform inputs/outputs for all runs of a client | `Client(hide_inputs=fn, hide_outputs=fn)` (and `hide_metadata=`) |
| Redact patterns (keys, emails, card numbers) everywhere | `Client(anonymizer=create_anonymizer([...]))` |
| Limit what one function records | `@traceable(process_inputs=..., process_outputs=...)` |
| Skip tracing for a flow | `with ls.tracing_context(enabled=False): ...` |
| Omit middleware payloads | `TracePolicy(process_inputs=omit_payload)` in `langchain.agents.middleware` |
| Keep PII out of the model and traces | `PIIMiddleware(..., apply_to_output=True)` in agents |

```python
import langsmith as ls
from langsmith.anonymizer import create_anonymizer

anonymizer = create_anonymizer([
    {"pattern": r"sk-[A-Za-z0-9_-]{16,}", "replace": "<api-key>"},
    {"pattern": r"[\w.+-]+@[\w-]+\.[\w.]+", "replace": "<email>"},
    {"pattern": r"\b(?:\d[ -]*?){13,16}\b", "replace": "<card>"},
])
client = ls.Client(anonymizer=anonymizer)
ls.configure(client=client)   # use this client for LangChain/LangGraph and @traceable runs
```

- The anonymizer is skipped for inputs or outputs that are already hidden by `LANGSMITH_HIDE_INPUTS` / `LANGSMITH_HIDE_OUTPUTS`. When data is sent, the anonymizer takes precedence over `hide_inputs` / `hide_outputs`. It walks up to 10 nesting levels by default (`max_depth=`).
- For deeper PII detection, call Microsoft Presidio or Amazon Comprehend from a `create_anonymizer(fn)` function.
- Never put secrets, raw tokens, or full personal records into `metadata` or `tags`.
- Restrict project and workspace access, and set trace retention to match your data policy.

## Control volume

- **Sampling**: `LANGSMITH_TRACING_SAMPLING_RATE=0.1` logs about 10% of traces. A per-client rate is also available (`Client(tracing_sampling_rate=...)`, used through `tracing_context(client=...)`). Sampling is random: combine it with conditional tracing when specific requests (errors, flagged users) must always be traced.
- **Conditional tracing**: decide per request in code, for example always trace errors, internal users, or flagged tenants, and sample the rest:

```python
enabled = request.headers.get("x-debug-trace") == "1" or tenant.trace_all or random.random() < 0.05
with ls.tracing_context(enabled=enabled):
    await agent.ainvoke(inputs, config)
```

- Each trace is limited to 25,000 runs. Split very long batch jobs into one trace per item.

## Make sure traces are sent

Tracing is asynchronous (a background thread), so short-lived processes can exit before runs are sent.

| Runtime | Do this |
| --- | --- |
| Long-running web server or worker | Keep background sending (the default) |
| Script, CLI, or cron job | Call `wait_for_all_tracers()` (`langchain_core.tracers.langchain`) in `finally`; for SDK-only code, call `client.flush()` |
| Serverless (Lambda, Cloud Functions) | Set `LANGCHAIN_CALLBACKS_BACKGROUND=false`, or flush before returning |
| Tests | Disable tracing unless the test checks it |

```python
from langchain_core.tracers.langchain import wait_for_all_tracers

def main():
    try:
        run_batch()
    finally:
        wait_for_all_tracers()
```

LangSmith outages or network errors do not raise inside your application code. Do not wrap agent calls in extra tracing error handling.

## Cost and usage tracking

- LangChain models, the `wrap_openai` / `wrap_anthropic` clients, and `@traceable(run_type="llm")` with OpenAI- or Anthropic-shaped outputs get token counts and costs automatically.
- For custom or local models, set `ls_provider` and `ls_model_name` metadata, attach `usage_metadata`, and define prices in the model price map. Cache reads and reasoning tokens are priced separately when reported.
- Group spend by user, tenant, feature, or thread with metadata keys, and use project dashboards or `Client().list_runs(project_name=..., filter=...)` for reports. Thread views sum tokens and cost across turns when `thread_id` is on all runs.
- Local token totals (`usage_metadata`, `get_usage_metadata_callback`) are useful for in-app limits. LangSmith is the source for historical cost analysis.

## Operational checklist

- [ ] Separate projects (or workspaces) per environment; production access is restricted.
- [ ] API key stored in a secret manager; region endpoint and workspace ID are set correctly.
- [ ] Root run per request, with `run_name`, environment and release tags, and `user_id`/`tenant_id`/`request_id` metadata.
- [ ] `thread_id` passed for conversational flows.
- [ ] Masking or anonymization reviewed for PII and secrets; large payloads excluded.
- [ ] Sampling or conditional tracing configured for high-volume paths.
- [ ] Flushing configured for scripts and serverless functions.
- [ ] Custom and local model names and prices configured for cost tracking.
- [ ] Distributed tracing headers accepted only from trusted internal services.
