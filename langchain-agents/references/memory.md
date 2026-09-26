# Memory

LangChain agents have two memory systems. Both come from LangGraph persistence.

| | Short-term memory | Long-term memory |
| --- | --- | --- |
| What | Conversation state of one thread: messages plus custom state fields | Application-defined JSON documents shared across threads |
| Mechanism | Checkpointer (`checkpointer=`) + `thread_id` | Store (`store=`) + namespaces and keys |
| Scope | One conversation/session | A user, organization, or the whole app, across sessions |
| Typical data | Chat history, intermediate tool results, HITL pauses | User profile and preferences, learned facts, past episodes, instructions |
| Read/write | Automatic every step; manual via `get_state` / `update_state` | Explicit `store.put/get/search` from tools, middleware, or nodes |

Most chat products need both: a checkpointer for the current conversation and a store for what should be remembered next time.

## Backend choice

| Backend | Checkpointer (short-term) | Store (long-term) | Use for |
| --- | --- | --- | --- |
| In-memory | `langgraph.checkpoint.memory.InMemorySaver` | `langgraph.store.memory.InMemoryStore` | Unit tests and demos only; lost on restart |
| SQLite | `langgraph.checkpoint.sqlite.SqliteSaver` / `.aio.AsyncSqliteSaver` | `langgraph.store.sqlite.SqliteStore` / `.aio.AsyncSqliteStore` | Local development, CLIs, single-process apps |
| PostgreSQL | `langgraph.checkpoint.postgres.PostgresSaver` / `.aio.AsyncPostgresSaver` | `langgraph.store.postgres.PostgresStore` / `.aio.AsyncPostgresStore` | Production and multi-process deployments |

Packages: `langgraph-checkpoint-sqlite` (includes `aiosqlite`) and `langgraph-checkpoint-postgres` plus `psycopg[binary]` (`psycopg[binary,pool]` when using a connection pool). Other supported backends include Redis, MongoDB, and Azure Cosmos DB; see `https://docs.langchain.com/oss/python/integrations/checkpointers.md`. When deployed on LangSmith Agent Server, persistence is provisioned automatically, and you should not pass your own checkpointer or store.

Call `.setup()` (or `await .setup()`) once to create the tables. Run it at deployment/migration time or at startup, not on every request.

## Short-term memory

### Development (SQLite)

```python
from langchain.agents import create_agent
from langgraph.checkpoint.sqlite import SqliteSaver

with SqliteSaver.from_conn_string("data/checkpoints.sqlite") as checkpointer:
    agent = create_agent("openai:gpt-5.5", tools=tools, checkpointer=checkpointer)
    config = {"configurable": {"thread_id": "conv-42"}}
    agent.invoke({"messages": [{"role": "user", "content": "Hi, I'm Bob."}]}, config)
    agent.invoke({"messages": [{"role": "user", "content": "What's my name?"}]}, config)  # remembers Bob
```

### Production (PostgreSQL, async app)

```python
from contextlib import asynccontextmanager

from fastapi import FastAPI
from langchain.agents import create_agent
from langgraph.checkpoint.postgres.aio import AsyncPostgresSaver
from langgraph.store.postgres.aio import AsyncPostgresStore


@asynccontextmanager
async def lifespan(app: FastAPI):
    async with (
        AsyncPostgresSaver.from_conn_string(settings.database_url) as checkpointer,
        AsyncPostgresStore.from_conn_string(settings.database_url) as store,
    ):
        await checkpointer.setup()   # idempotent; prefer running it in a migration step
        await store.setup()
        app.state.agent = create_agent(
            settings.llm_model, tools=tools, checkpointer=checkpointer, store=store, context_schema=Context
        )
        yield


app = FastAPI(lifespan=lifespan)
```

- For high concurrency, pass a `psycopg_pool.AsyncConnectionPool` to `AsyncPostgresSaver(pool)` instead of one connection. Open it with `kwargs={"autocommit": True, "prepare_threshold": 0, "row_factory": dict_row}`; these are the same settings `from_conn_string` uses.
- Keep `thread_id` under 255 characters. Use UUIDs (`langchain_core.utils.uuid.uuid7()` gives time-ordered IDs).
- Scope thread IDs to their owner. Check in the application that the caller owns the `thread_id` before invoking, because the checkpointer does not enforce ownership.
- Checkpoints grow with every step. Schedule pruning of old threads (`checkpointer.delete_thread(thread_id)` or a retention job), and look at `durability=` (`"exit"`, `"async"`, `"sync"`) when write overhead matters.

### Custom state

```python
from langchain.agents import AgentState

class SupportState(AgentState):
    customer_id: str
    open_ticket_ids: list[str]

agent = create_agent(model, tools, state_schema=SupportState, checkpointer=checkpointer)
agent.invoke({"messages": [...], "customer_id": "c-9", "open_ticket_ids": []}, config)
```

Tools read state with `runtime.state[...]` and write it by returning `Command(update={...})` with a matching `ToolMessage`.

### Keep history inside the context window

Choose one strategy and make it explicit:

1. **Summarize (recommended default):**

   ```python
   from langchain.agents.middleware import SummarizationMiddleware

   SummarizationMiddleware(model="openai:gpt-5.4-mini", trigger=("tokens", 6000), keep=("messages", 20))
   ```

   `trigger` also accepts `("fraction", 0.8)` of the model's context window (this needs model profile data), `("messages", n)`, a dict for AND conditions, or a list for OR conditions.
2. **Trim** before each model call with `@before_model`: return `{"messages": [RemoveMessage(id=REMOVE_ALL_MESSAGES), *kept]}`. Import `RemoveMessage` from `langchain.messages` and `REMOVE_ALL_MESSAGES` from `langgraph.graph.message`.
3. **Delete** specific messages with `RemoveMessage(id=m.id)` from `@after_model` or a tool.
4. **Clear old tool results** with `ContextEditingMiddleware`.

Whatever you remove, keep the history valid: every `AIMessage` with `tool_calls` must be followed by its `ToolMessage`s, and some providers require the first non-system message to come from the user.

### Inspect and edit a thread

```python
snapshot = agent.get_state(config)             # .values, .next, .config, .metadata, .created_at
history = list(agent.get_state_history(config)) # newest first
agent.update_state(config, {"messages": [AIMessage("Correction: ...")]})  # new checkpoint; reducers apply
```

## Long-term memory

```python
import uuid
from dataclasses import dataclass

from langchain.agents import create_agent
from langchain.tools import tool, ToolRuntime
from langgraph.store.sqlite import SqliteStore


@dataclass
class Context:
    user_id: str


@tool
def remember_preference(preference: str, runtime: ToolRuntime[Context]) -> str:
    """Save a lasting user preference, such as tone or language, for future conversations."""
    runtime.store.put(("users", runtime.context.user_id, "preferences"), str(uuid.uuid4()), {"text": preference})
    return "Saved."


@tool
def recall_preferences(runtime: ToolRuntime[Context]) -> str:
    """List the user's saved preferences."""
    items = runtime.store.search(("users", runtime.context.user_id, "preferences"), limit=50)
    return "\n".join(item.value["text"] for item in items) or "No saved preferences."


with SqliteStore.from_conn_string("data/memory.sqlite") as store:
    store.setup()
    agent = create_agent(model, tools=[remember_preference, recall_preferences], store=store, context_schema=Context)
```

Store API (sync, with async `a*` variants): `put(namespace, key, value, index=None)`, `get(namespace, key)` → `Item | None` (`.value`, `.key`, `.namespace`, `.created_at`, `.updated_at`), `search(namespace_prefix, query=None, filter=None, limit=10, offset=0)`, `delete(namespace, key)`, and `list_namespaces(prefix=..., max_depth=...)`.

- Namespaces are tuples such as `("users", user_id, "preferences")` or `("orgs", org_id, "policies")`. `search` matches by prefix. Always include the tenant or user ID, and take it from `runtime.context`, never from model-supplied arguments.
- `search` truncates silently at `limit`, and ordering differs between backends. Page with `offset`, and sort by `updated_at` when order matters.
- Semantic search: configure an index when creating the store, then pass `query=`:

  ```python
  from langchain.embeddings import init_embeddings

  index = {"embed": init_embeddings("openai:text-embedding-3-small"), "dims": 1536, "fields": ["text"]}
  with PostgresStore.from_conn_string(DB_URI, index=index) as store:   # also SqliteStore.from_conn_string(..., index=...)
      store.setup()                                                    # or InMemoryStore(index=index) for tests
      store.search(("users", uid, "facts"), query="What food does the user like?", limit=5)
  ```

  Pass `index=False` on `put` to skip embedding an item. PostgreSQL semantic search requires the `pgvector` extension in the database. SQLite uses `sqlite-vec`, which is installed with `langgraph-checkpoint-sqlite`.
- Stores accept `ttl=` configuration for expiring items. Use it for data that should not be kept forever.
- Inject memories into the prompt with `@dynamic_prompt` or `@before_model` (read from `request.runtime.store`), or let the model fetch them through a recall tool. Keep injected memory short and relevant.

### Memory types

- **Semantic** (facts about the user or domain): either a single profile document that is updated in place (pass the old profile and ask for a new one; use structured output for the schema), or a collection of small fact documents (better recall; needs update/delete handling).
- **Episodic** (past experiences): store successful past interactions and retrieve similar ones as few-shot examples.
- **Procedural** (how to behave): store instructions or prompt fragments that the agent or an offline reflection job refines based on feedback, and load them into the system prompt.

### When to write memories

- **Hot path**: the agent calls a `save_memory` tool during the conversation. Memories are available immediately, but this adds latency and the model decides what is kept.
- **Background**: after the conversation (or on a schedule), a separate job reads the thread and writes memories. No user-facing latency and better focus, but memories are delayed. Consider this for production chat.

### Privacy and safety

Long-term memory is persistent personal data. Store only what the feature needs, and support per-user export and deletion (delete by namespace). Never store secrets or credentials. Do not let one user's namespace be reachable from another user's context. Treat memories as untrusted text when injecting them into prompts.
