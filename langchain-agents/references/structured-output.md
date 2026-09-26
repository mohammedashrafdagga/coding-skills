# Structured Output (JSON) and Retries

Use structured output whenever code, not a person, consumes the result. Prefer Pydantic models: they give runtime validation, field descriptions the model can read, and typed access. `TypedDict` and dataclasses return plain dicts without validation. JSON Schema dicts give interoperability but need explicit handling.

## Pick the entry point

| Situation | Use |
| --- | --- |
| One model call, no tools (extraction, classification, routing) | `model.with_structured_output(Schema)` |
| Agent that uses tools and must end with typed data | `create_agent(..., response_format=Schema or ToolStrategy(...) or ProviderStrategy(...))` |
| Need token usage or the raw message as well as the parsed object | `with_structured_output(Schema, include_raw=True)` |

## Define schemas

```python
from typing import Literal
from pydantic import BaseModel, Field


class TicketTriage(BaseModel):
    """Triage result for a customer support ticket."""

    category: Literal["billing", "technical", "account", "other"] = Field(description="Primary topic of the ticket")
    priority: int = Field(ge=1, le=4, description="1 = urgent, 4 = low")
    summary: str = Field(max_length=280, description="One-sentence summary in English")
    needs_human: bool = Field(description="True if an agent must reply personally")
```

- The class docstring and each `Field(description=...)` are sent to the model. Write them as instructions.
- Use `Literal`/`Enum` for closed sets and numeric bounds for scores. Nested models and `list[Model]` are supported.
- Keep optional fields explicit (`str | None = None`). Do not ask the model to invent values: say in the prompt that unknown fields should be null.
- For JSON Schema dicts, include top-level `title` and `description`, and wrap them in `ToolStrategy` or `ProviderStrategy`; a bare dict is not auto-detected.

## Direct model call

```python
from langchain.chat_models import init_chat_model

model = init_chat_model("openai:gpt-5.5")
triage_model = model.with_structured_output(TicketTriage)          # method chosen per provider
result: TicketTriage = triage_model.invoke(ticket_text)

raw_model = model.with_structured_output(TicketTriage, include_raw=True)
out = raw_model.invoke(ticket_text)   # {"raw": AIMessage, "parsed": TicketTriage | None, "parsing_error": Exception | None}
```

`method=` options: `"json_schema"` (provider-native constrained output; use it for Anthropic native structured output), `"function_calling"` (a forced tool call; works on any tool-capable model), and `"json_mode"` (valid JSON only; the schema must also be described in the prompt). With `include_raw=True`, parse errors are returned instead of raised. Check `parsing_error` and retry or fall back.

A simple bounded retry for direct calls (native JSON mode, so the raw reply has no pending tool calls):

```python
json_model = model.with_structured_output(TicketTriage, method="json_schema", include_raw=True)

def triage(text: str, attempts: int = 2) -> TicketTriage:
    messages = [("system", "Classify the ticket. Use null for unknown fields."), ("human", text)]
    for _ in range(attempts):
        out = json_model.invoke(messages)
        if out["parsed"] is not None:
            return out["parsed"]
        messages += [out["raw"], ("human", f"Your output was invalid: {out['parsing_error']}. Return valid data.")]
    raise ValueError("Model did not return valid TicketTriage")
```

With `method="function_calling"`, the raw message contains tool calls. Answer them with matching `ToolMessage`s before appending a corrective human message, or use the agent `ToolStrategy`, which does this automatically.

## Agent structured output

```python
from langchain.agents import create_agent
from langchain.agents.structured_output import ToolStrategy, ProviderStrategy

agent = create_agent(model="openai:gpt-5.5", tools=[lookup_customer], response_format=TicketTriage)
result = agent.invoke({"messages": [{"role": "user", "content": ticket_text}]})
triage: TicketTriage = result["structured_response"]
```

Passing the schema type lets LangChain choose the strategy from the model profile:

- `ProviderStrategy(schema, strict=None)`: the provider's native structured output (OpenAI, Anthropic, xAI, Gemini). This is the most reliable option when it is supported, and it requires the model to support tools and structured output together. `strict=True` enables strict schema adherence where supported (`langchain>=1.2`).
- `ToolStrategy(schema, tool_message_content=None, handle_errors=True)`: structured output through an artificial tool call. It works with any tool-calling model (DeepSeek, Ollama, others) and supports `Union[A, B]` schemas, where the model picks one.

When the model profile is missing or wrong, force the strategy explicitly or pass `init_chat_model(..., profile={"structured_output": True})`.

## Validation retries with ToolStrategy

With `ToolStrategy`, validation failures are sent back to the model as a `ToolMessage` ("...Please fix your mistakes."), and the model tries again inside the same run. This covers schema validation errors (for example `priority=7` violating `le=4`) and multiple structured outputs returned when only one was expected.

Control this with `handle_errors`:

| Value | Behavior |
| --- | --- |
| `True` (default) | Catch all errors and retry with the default error text |
| `"custom text"` | Catch all errors and retry with this fixed message |
| `ValueError` or `(ValueError, TypeError)` | Retry only these error types; raise the rest |
| `callable(exc) -> str` | Build a tailored retry message |
| `False` | No retry; raise immediately |

```python
from langchain.agents.middleware import ModelCallLimitMiddleware
from langchain.agents.structured_output import (
    MultipleStructuredOutputsError,
    StructuredOutputValidationError,
    ToolStrategy,
)

def explain(error: Exception) -> str:
    if isinstance(error, StructuredOutputValidationError):
        return f"Output did not match the schema: {error}. Return corrected values only."
    if isinstance(error, MultipleStructuredOutputsError):
        return "Return exactly one result."
    return f"Error: {error}"

agent = create_agent(
    model,
    tools=[lookup_customer],
    response_format=ToolStrategy(TicketTriage, handle_errors=explain),
    middleware=[ModelCallLimitMiddleware(run_limit=6)],   # bound retries and tool loops together
)
```

Every retry is another model call. Always bound the run (`ModelCallLimitMiddleware`, `recursion_limit`), and treat a missing `structured_response` as a failure at the application boundary.

## Checklist

- The schema lives in the domain or application layer, not inside the prompt string.
- The prompt says what to do with unknown values. The schema allows `None` where data may be missing.
- Validate business rules after parsing (for example, that referenced IDs exist). Schema validity is not business validity.
- Log parse failures and retry counts. With LangSmith tracing, each retry is visible as a separate model run.
- For Ollama or other small models, prefer `ToolStrategy` with a small, flat schema, and test it with real outputs.
