---
name: langchain-agents
description: Build, extend, or fix Python AI agents with LangChain v1 (`create_agent`), including installation, OpenAI/Anthropic/DeepSeek/Ollama providers, tools, middleware, structured JSON output with validation retries, short-term and long-term memory (in-memory, SQLite, PostgreSQL), streaming, and token usage. Use when adding an LLM agent or chat feature to a Python project, switching model providers, or debugging LangChain agent behavior; use langgraph-workflows for custom multi-step graphs and langsmith-tracing for observability.
metadata:
  author: "mohammedashrafdagga"
  version: "1.0.0"
  supported-agents: "codex,claude-code,cursor"
  library-versions: "langchain>=1.4,langchain-core>=1.6,langgraph>=1.2"
---

# LangChain Agents

Build production-quality agents with LangChain v1 in Python. `create_agent` is the default: it gives a model-plus-tools loop that runs on LangGraph, so it gets persistence, streaming, human-in-the-loop, and LangSmith tracing without extra wiring. Change application code only as far as the user's request requires, and follow the project's existing structure, configuration, and dependency tooling.

## Choose the right layer

- **Single model call** (classification, extraction, summarization, no tools): use a chat model directly with `init_chat_model(...)` and `with_structured_output(...)`. Do not create an agent.
- **Agent** (the model decides which tools to call and loops until done): use `langchain.agents.create_agent`. Customize it with middleware rather than rewriting the loop.
- **Custom control flow** (fixed steps, branching, parallel fan-out, several cooperating agents, durable long-running jobs): use the `langgraph-workflows` skill. A `create_agent` graph can be used as a node inside it.
- **Tracing, debugging, cost visibility**: use the `langsmith-tracing` skill alongside this one.

## Workflow

1. **Inspect the project first.** Find the Python version (LangChain requires 3.10+), the dependency manager (`pyproject.toml` with uv or Poetry, or `requirements.txt`), existing LangChain/LangGraph packages and their versions, how settings and secrets are loaded, and whether the app is sync or async (for example FastAPI). Match what exists. Do not add a second configuration system.
2. **Install packages** with the project's tool. Read [references/setup-and-providers.md](references/setup-and-providers.md) for package names, the extras for each provider, environment variables, and provider-specific pitfalls. Install only the providers the user needs.
3. **Configure the model** from settings or environment variables, never from hard-coded keys. Keep the model identifier in configuration (for example `LLM_MODEL=openai:gpt-5.5`) so the provider can be switched without changing code. Check the provider's current model list before pinning a model name; model IDs change often.
4. **Build the agent.** Read [references/agents-tools-middleware.md](references/agents-tools-middleware.md) for `create_agent`, tool design, `ToolRuntime`, runtime `context`, built-in middleware (retry, fallback, limits, summarization, PII, human-in-the-loop), and custom middleware hooks.
5. **Add structured output** when callers need typed or JSON data. Read [references/structured-output.md](references/structured-output.md). Prefer a Pydantic schema. Use `ToolStrategy(..., handle_errors=...)` to control validation retries.
6. **Add memory** when conversations must continue across calls or facts must persist across sessions. Read [references/memory.md](references/memory.md) to choose between short-term memory (checkpointer + `thread_id`) and long-term memory (store + namespaces), and to choose a backend: in-memory for tests, SQLite for local and single-process use, PostgreSQL for production.
7. **Add streaming and token accounting** for user-facing or cost-sensitive paths. Read [references/streaming-and-usage.md](references/streaming-and-usage.md).
8. **Verify.** Write or update tests that use `GenericFakeChatModel` and `InMemorySaver`, so the tests are deterministic and need no API key (see the testing section in the agents reference). Run the project's test and lint commands. Only call real providers when the user has configured credentials and agreed to incur the cost.

## Non-negotiable rules

- Never hard-code or log API keys. Read them from the environment or the project's secret manager, and add new variables to `.env.example` (never to a committed `.env`).
- Import from the v1 namespaces: `langchain.agents`, `langchain.tools`, `langchain.messages`, `langchain.chat_models`, `langchain.agents.middleware`, `langchain.agents.structured_output`. Do not generate legacy `AgentExecutor`, `initialize_agent`, `LLMChain`, `ConversationBufferMemory`, or `langgraph.prebuilt.create_react_agent` code for new work. When you touch existing legacy code, point out the migration path instead of expanding the legacy code.
- Give every tool type hints and a precise docstring; the model uses them as the tool schema. Do not name tool parameters `config` or `runtime`, because both names are reserved.
- Pass per-request data (user ID, tenant, permissions, DB handles) through `context_schema` and `context=`, not through prompts or global variables. Enforce authorization inside tools; the model is not a security boundary.
- A `thread_id` persists history only when the agent has a checkpointer. Human-in-the-loop, `thread_limit` call limits, and resuming all require a checkpointer.
- Use `InMemorySaver` and `InMemoryStore` only in tests and demos. They lose all data on restart.
- Bound every agent: set model/tool call limits or a `recursion_limit`, and configure provider `timeout` and `max_retries` for production traffic.
- Treat tool outputs and retrieved documents as untrusted input. Gate side-effecting tools (payments, email, deletes, writes) behind `HumanInTheLoopMiddleware` or explicit application checks.
- On async stacks (FastAPI, async workers), use `ainvoke`, `astream`, `astream_events`, and the async checkpointers and stores. Create the database-backed savers and stores once at application startup and close them on shutdown.

## Look up current documentation

The APIs above were verified against `langchain` 1.4, `langchain-core` 1.6, and `langgraph` 1.2. When the installed version differs, or a feature is not covered here, read the official Markdown docs rather than guessing. The page index is at `https://docs.langchain.com/llms.txt`, and each page is available as `.md`, for example `https://docs.langchain.com/oss/python/langchain/agents.md`, `.../models.md`, `.../structured-output.md`, `.../short-term-memory.md`, `.../long-term-memory.md`, `.../streaming.md`, `.../middleware/built-in.md`, and `https://docs.langchain.com/oss/python/integrations/chat/<provider>.md`.

## Finish

Report what was added or changed, which packages and environment variables are now required, which memory backend and model provider were chosen and why, how to run the tests, and any behavior that still needs a real-provider check.
