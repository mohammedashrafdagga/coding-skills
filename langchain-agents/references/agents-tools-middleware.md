# Agents, Tools, and Middleware

## Create an agent

```python
from dataclasses import dataclass

from langchain.agents import create_agent
from langchain.tools import tool, ToolRuntime
from langgraph.checkpoint.memory import InMemorySaver


@dataclass
class Context:
    user_id: str
    tenant_id: str


@tool
def get_order_status(order_id: str, runtime: ToolRuntime[Context]) -> str:
    """Return the shipping status of one of the current user's orders.

    Args:
        order_id: The order identifier shown to the customer, e.g. "A-1042".
    """
    order = orders_repo.get(order_id, tenant_id=runtime.context.tenant_id)
    if order is None or order.user_id != runtime.context.user_id:
        return "No order with that ID belongs to this user."
    return f"Order {order_id}: {order.status}"


agent = create_agent(
    model="openai:gpt-5.5",               # or a model instance
    tools=[get_order_status],
    system_prompt="You are a concise support assistant. Use tools; never guess order data.",
    context_schema=Context,
    checkpointer=InMemorySaver(),          # use SQLite/Postgres outside tests (see memory.md)
    name="support_agent",                  # labels the agent in traces, streams, and multi-agent graphs
)

result = agent.invoke(
    {"messages": [{"role": "user", "content": "Where is order A-1042?"}]},
    config={"configurable": {"thread_id": conversation_id}},
    context=Context(user_id=user.id, tenant_id=user.tenant_id),
)
answer = result["messages"][-1].text
```

`create_agent` parameters: `model`, `tools`, `system_prompt` (string or `SystemMessage`), `middleware`, `response_format`, `state_schema` (a subclass of `AgentState`; Pydantic state is not supported), `context_schema`, `checkpointer`, `store`, `interrupt_before` / `interrupt_after`, `name`, `cache`, `debug`. The return value is a compiled LangGraph graph, so `invoke`, `ainvoke`, `stream`, `astream`, `stream_events`, `get_state`, and `update_state` all work on it.

- `config={"configurable": {"thread_id": ...}}` selects the conversation. It is needed for memory, interrupts, and thread-level limits.
- `context=` carries immutable per-run data (identity, permissions, feature flags, clients). Tools and middleware read it from `runtime.context`.
- `config={"recursion_limit": N}` caps graph steps. The default is 1000. `GraphRecursionError` is raised when the cap is exceeded.
- Pass `version="v2"` to `invoke` to get a `GraphOutput` with `.value` (final state) and `.interrupts`, instead of a plain dict that uses the `__interrupt__` key.
- Add custom state fields by subclassing `AgentState` (for example `class State(AgentState): user_name: str`) and passing `state_schema=State`. Pass initial values in the input dict.

## Design tools

- Type hints are required: they become the JSON schema. The docstring (or `description=`) tells the model when to use the tool, and an `Args:` section documents each parameter. Use `snake_case` names.
- For complex inputs, use `@tool(args_schema=PydanticModel)` with `Field(description=...)`, `Literal[...]` enums, and bounds.
- Return concise, model-readable strings or small JSON-serializable objects. Do not return raw large payloads; paginate or summarize.
- Reserved parameter names: `config` and `runtime`. Access runtime data through a parameter typed `ToolRuntime`. The model does not see it:
  - `runtime.state`: current agent state (messages and custom fields)
  - `runtime.context`: per-run context
  - `runtime.store`: long-term memory store (`None` when no store is configured)
  - `runtime.stream_writer`: emit custom progress for `stream_mode="custom"`
  - `runtime.tool_call_id`, `runtime.config`, `runtime.execution_info`
  - Generic form: `ToolRuntime[ContextT, StateT]`
- To update state from a tool, return `Command(update={...})` from `langgraph.types`. Include a `ToolMessage(content=..., tool_call_id=runtime.tool_call_id)` in `messages`, so that every tool call receives a response.
- `@tool(return_direct=True)` ends the run with the tool output and skips the extra model call. Use it only for tools whose output is the final answer.
- Async tools: define them with `async def`. The agent must then be driven by `ainvoke` or `astream`.
- MCP servers: `from langchain.mcp import MCPAdapter` (beta, requires `langchain[mcp]>=1.4`). Use `async with MCPAdapter(url) as adapter: tools = await adapter.list_tools()`. The established alternative is the `langchain-mcp-adapters` package.
- Provider built-in tools (web search, code execution) are passed as dicts, for example `{"type": "web_search"}`. They run on the provider side and appear as `server_tool_call` content blocks.

## Invocation patterns

| Need | Call |
| --- | --- |
| Final answer | `agent.invoke(input, config, context=...)` → `result["messages"][-1].text` |
| Async server | `await agent.ainvoke(...)` |
| Typed output | `result["structured_response"]` (see `structured-output.md`) |
| Progress/tokens | `stream(..., version="v2")` or `stream_events(..., version="v3")` (see `streaming-and-usage.md`) |
| Inspect a thread | `agent.get_state(config).values["messages"]` |
| Many independent inputs | `agent.batch([...], config={"max_concurrency": 5})` |

Input messages can be dicts (`{"role": "user", "content": "..."}`) or message objects from `langchain.messages` (`HumanMessage`, `SystemMessage`, `AIMessage`, `ToolMessage`). Read text with `message.text`, and provider-neutral parts (text, reasoning, tool calls, citations, images) with `message.content_blocks`.

## Prompts

- Use a static `system_prompt=` for fixed instructions.
- For per-user or per-state prompts, use `@dynamic_prompt`:

```python
from langchain.agents.middleware import dynamic_prompt, ModelRequest

@dynamic_prompt
def personalized_prompt(request: ModelRequest) -> str:
    ctx = request.runtime.context
    return f"You help {ctx.user_id}. Answer in {ctx.locale}. Be brief."

agent = create_agent(model, tools, middleware=[personalized_prompt], context_schema=Context)
```

## Built-in middleware

Import from `langchain.agents.middleware`. Add only what the use case needs.

| Middleware | Purpose | Key options |
| --- | --- | --- |
| `ModelRetryMiddleware` | Retry failed model calls with backoff | `max_retries=2`, `retry_on`, `on_failure="continue" or "error"`, `initial_delay`, `backoff_factor`, `max_delay`, `jitter` |
| `ModelFallbackMiddleware` | Try other models when the primary fails | `ModelFallbackMiddleware("openai:gpt-5.4-mini", "anthropic:claude-sonnet-4-6")` |
| `ToolRetryMiddleware` | Retry failing tools | `max_retries`, `tools=[...]`, `retry_on`, `on_failure` |
| `ToolErrorMiddleware` | Turn tool exceptions into `ToolMessage`s the model can recover from | `on_error(exc, request) -> str or None`; place `ToolRetryMiddleware(on_failure="error")` after it |
| `ModelCallLimitMiddleware` | Stop runaway loops and cost | `run_limit`, `thread_limit` (needs checkpointer), `exit_behavior="end" or "error"` |
| `ToolCallLimitMiddleware` | Cap tool calls globally or per tool | `tool_name`, `run_limit`, `thread_limit`, `exit_behavior="continue"/"error"/"end"` |
| `SummarizationMiddleware` | Compress old history near the context limit | `model`, `trigger=("tokens", 4000)` or `("fraction", 0.8)` or `("messages", 50)`, `keep=("messages", 20)` |
| `HumanInTheLoopMiddleware` | Pause for approve/edit/reject/respond before a tool runs | `interrupt_on={"send_email": {"allowed_decisions": ["approve", "edit", "reject"]}, "read_email": False}`; needs a checkpointer |
| `PIIMiddleware` | Detect and handle PII | `PIIMiddleware("email", strategy="redact" or "mask" or "hash" or "block", apply_to_input=True, apply_to_output=False, apply_to_tool_results=...)` |
| `LLMToolSelectorMiddleware` | Pre-select relevant tools when there are many | `model`, `max_tools`, `always_include` |
| `ContextEditingMiddleware` | Clear old tool outputs from context | `edits=[ClearToolUsesEdit(...)]` |
| `TodoListMiddleware` | Give the agent a planning to-do tool | none |
| `ShellToolMiddleware`, `FilesystemFileSearchMiddleware` | Controlled shell and file search | Sandbox and execution policies; use only with explicit user approval |

Provider-specific middleware lives in the provider packages, for example `langchain_anthropic.middleware.AnthropicPromptCachingMiddleware`.

## Human-in-the-loop flow

```python
from langchain.agents.middleware import HumanInTheLoopMiddleware
from langgraph.types import Command

agent = create_agent(
    model, tools=[send_email, read_email],
    checkpointer=checkpointer,
    middleware=[HumanInTheLoopMiddleware(interrupt_on={
        "send_email": {"allowed_decisions": ["approve", "edit", "reject"]},
        "read_email": False,
    })],
)
config = {"configurable": {"thread_id": thread_id}}
result = agent.invoke({"messages": [...]}, config, version="v2")
if result.interrupts:
    request = result.interrupts[0].value            # {"action_requests": [...], "review_configs": [...]}
    decisions = [{"type": "approve"}]               # one decision per action, in order
    # {"type": "edit", "edited_action": {"name": "send_email", "args": {...}}}
    # {"type": "reject", "message": "Do not email external domains."}   # tool not executed
    # {"type": "respond", "message": "Blue."}  # only for ask-the-human tools; reported as a successful result
    result = agent.invoke(Command(resume={"decisions": decisions}), config, version="v2")
```

Resume with the same `thread_id`. `Command(resume=...)` is the only kind of `Command` to pass as input. To continue a finished conversation, pass a normal input dict.

## Custom middleware

Hooks run inside the agent graph:

| Hook | Style | When |
| --- | --- | --- |
| `before_agent` / `after_agent` | node | once per invocation |
| `before_model` / `after_model` | node | around every model call; return a dict of state updates, or `None` |
| `wrap_model_call` | wrap | around each model call; can change the request, retry, short-circuit, or swap the model |
| `wrap_tool_call` | wrap | around each tool call |

Use the decorators (`@before_model`, `@after_model`, `@wrap_model_call`, `@wrap_tool_call`, `@dynamic_prompt`) for single hooks. Subclass `AgentMiddleware` for configurable or multi-hook middleware, and declare `state_schema` on the class when the middleware needs extra state.

```python
from typing import Callable
from langchain.agents.middleware import wrap_model_call, ModelRequest, ModelResponse
from langchain.chat_models import init_chat_model

fast = init_chat_model("openai:gpt-5.4-mini")
strong = init_chat_model("openai:gpt-5.5")

@wrap_model_call
def route_by_length(request: ModelRequest, handler: Callable[[ModelRequest], ModelResponse]) -> ModelResponse:
    model = strong if len(request.state["messages"]) > 10 else fast
    return handler(request.override(model=model))   # override also accepts system_prompt=, tools=
```

- Execution order for `middleware=[a, b, c]`: `before_*` hooks run a→b→c, `after_*` hooks run c→b→a, and `wrap_*` hooks nest (a wraps b wraps c). Put guardrails first.
- To exit early, return `{"jump_to": "end"}` (or `"tools"` / `"model"`) from a node hook decorated with `can_jump_to=[...]` (`@before_model(can_jump_to=["end"])` or `@hook_config(can_jump_to=[...])`).
- Pre-bound models (`bind_tools` already called) cannot be used for dynamic model selection together with structured output.
- To reduce trace noise, set `trace_policy = TracePolicy(process_inputs=omit_payload)` on a middleware class, or call `configure_trace_policy(...)` globally.

## Use inside an existing web app

- Build the agent once (at module import or in application startup) and reuse the compiled graph across requests. Keep per-request data out of module globals: create a new `thread_id` per conversation and pass `context=` per request.
- In FastAPI, open async checkpointers and stores in the lifespan handler (see `memory.md`), use `await agent.ainvoke(...)`, and stream with `astream` or `astream_events` into a `StreamingResponse` or SSE.
- Map `GraphRecursionError`, `ModelRateLimitError`, and timeouts to controlled HTTP errors, and log with correlation IDs.

## Testing without API keys

```python
from langchain_core.language_models.fake_chat_models import GenericFakeChatModel
from langchain.messages import AIMessage, ToolCall
from langgraph.checkpoint.memory import InMemorySaver


class FakeToolModel(GenericFakeChatModel):
    def bind_tools(self, tools, **kwargs):   # create_agent binds tools; the fake ignores them
        return self


def test_agent_calls_order_tool():
    model = FakeToolModel(messages=iter([
        AIMessage(content="", tool_calls=[ToolCall(name="get_order_status", args={"order_id": "A-1"}, id="c1")]),
        AIMessage(content="It shipped."),
    ]))
    agent = create_agent(model, tools=[get_order_status], context_schema=Context, checkpointer=InMemorySaver())
    out = agent.invoke({"messages": [{"role": "user", "content": "status A-1?"}]},
                       {"configurable": {"thread_id": "t"}}, context=Context("u1", "t1"))
    assert out["messages"][-1].text == "It shipped."
    assert out["messages"][2].type == "tool"
```

- Script each model turn: tool-call turns, then a final text turn. The iterator raises an error when it runs out, which catches unexpected extra loops.
- `GenericFakeChatModel` does not stream tool calls. Test streaming token output with text-only turns.
- Test tools directly with `tool.invoke({...})`, and middleware by driving a small agent with a fake model.
- Keep real-provider integration tests behind a marker or environment flag.
