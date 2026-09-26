---
name: langsmith-tracing
description: Add, configure, or debug LangSmith tracing and observability for Python LLM applications built with LangChain, LangGraph, or plain SDK code, including environment setup, projects and regions, run names, tags and metadata, conversation threads, `@traceable` custom spans, token and cost tracking, sampling, selective tracing, masking sensitive data, distributed tracing, and flushing traces in serverless or short-lived processes. Use when a user wants to trace or monitor an agent, see token usage or cost per user or thread, or fix missing, broken, or leaking traces.
metadata:
  author: "mohammedashrafdagga"
  version: "1.0.0"
  supported-agents: "codex,claude-code,cursor"
  library-versions: "langsmith>=0.14,langchain>=1.4,langgraph>=1.2"
---

# LangSmith Tracing

Make every agent run inspectable: each user request becomes a **trace** (a tree of **runs** for the graph, nodes, model calls, tools, and retrievers), traces are grouped into **projects**, and the turns of a conversation are linked into **threads**. LangChain and LangGraph trace automatically once tracing is enabled. Other code is traced with `@traceable`. Tracing is configuration plus a small amount of instrumentation: do not restructure application logic for it.

## Workflow

1. **Inspect the project**: framework (LangChain `create_agent`, LangGraph `StateGraph`, raw provider SDKs, or a mix), runtime (web server, worker, CLI, serverless), how settings and secrets load, and existing logging or OpenTelemetry. Find the natural "request" boundary (HTTP handler, job, CLI command). Each should become one trace.
2. **Enable tracing through configuration.** Read [references/setup.md](references/setup.md) for the environment variables (`LANGSMITH_TRACING`, `LANGSMITH_API_KEY`, `LANGSMITH_PROJECT`, `LANGSMITH_ENDPOINT` for EU/APAC/AWS/self-hosted, `LANGSMITH_WORKSPACE_ID`), per-environment projects, and programmatic configuration. Add the variables to `.env.example` and the settings class. Never commit keys.
3. **Enrich LangChain/LangGraph traces.** Read [references/langchain-langgraph.md](references/langchain-langgraph.md). Pass `run_name`, `tags`, and `metadata` (including `thread_id` and a user identifier) through the run config, name agents and graphs, set `ls_model_name` / `ls_provider` for custom or local models, and decide which parts to trace.
4. **Instrument non-LangChain code.** Read [references/custom-instrumentation.md](references/custom-instrumentation.md) for `@traceable`, `trace` context managers, `wrap_openai` / `wrap_anthropic`, run types, attaching token usage and cost to custom LLM spans, thread pools, async code, and distributed tracing across services.
5. **Make it production-safe.** Read [references/production.md](references/production.md) for masking and anonymizing sensitive inputs and outputs, sampling, conditional tracing, flushing before exit, cost tracking, and failure isolation.
6. **Verify.** Run one real request with tracing enabled and confirm in LangSmith that there is a single root run per request, child runs nest correctly (no orphan traces), metadata and tags are present, threads group correctly, token counts and costs appear on model runs, and no secrets or unmasked PII are visible. In automated tests, disable tracing (`LANGSMITH_TRACING=false` or `tracing_context(enabled=False)`), unless a test explicitly checks tracing.

## Non-negotiable rules

- Read `LANGSMITH_API_KEY` from the environment or a secret manager. Never hard-code it, log it, or put it in traced inputs or metadata.
- Tracing must not change behavior or break requests. LangSmith sends data in a background thread; keep that default in long-running servers, and flush explicitly in short-lived processes (see the production reference).
- Treat traces as sensitive data stores. Traces capture prompts, tool arguments, retrieved documents, and outputs. Before tracing production traffic, configure masking or anonymization for PII, credentials, and regulated data, and use separate projects or workspaces per environment.
- Put identifiers in `metadata`, not in `run_name`. Use stable, low-cardinality `tags` (environment, feature, version) and high-cardinality values (user ID, thread ID, request ID) in `metadata`. Prefer pseudonymous user IDs over emails.
- Use the same `thread_id` value in the LangGraph `configurable` config and in the LangSmith `metadata`, so that the conversation state and the trace thread match.
- Do not create root traces inside loops or background tasks by accident. Propagate context (`tracing_context(parent=...)`, `ContextThreadPoolExecutor`, `langsmith_extra={"parent": ...}`), so that work belongs to the request that caused it.
- Keep the LangSmith SDK and LangChain packages reasonably current. Check the docs before using features newer than the versions verified here.

## Look up current documentation

Verified against `langsmith` 0.14, `langchain` 1.4, and `langgraph` 1.2. The official Markdown docs are indexed at `https://docs.langchain.com/langsmith/llms.txt`. Useful pages: `https://docs.langchain.com/langsmith/trace-with-langchain.md`, `.../trace-with-langgraph.md`, `.../annotate-code.md`, `.../add-metadata-tags.md`, `.../threads.md`, `.../cost-tracking.md`, `.../mask-inputs-outputs.md`, `.../sample-traces.md`, `.../conditional-tracing.md`, `.../distributed-tracing.md`, and `.../log-traces-to-project.md`.

## Finish

Report the environment variables and packages now required, the project naming per environment, the metadata and tags attached (and where), how threads are grouped, what is masked or excluded, the sampling and flushing configuration, and a link or description of the verified example trace. State any part that could not be verified without a LangSmith API key.
