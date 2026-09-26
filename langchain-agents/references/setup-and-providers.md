# Setup and Providers

## Install

LangChain requires Python 3.10+. Install the core package plus one integration package per provider. Use the project's dependency manager; `uv add` is shown with the pip equivalent.

| Need | uv | pip |
| --- | --- | --- |
| Core (agents, messages, tools, middleware) | `uv add langchain` | `pip install -U langchain` |
| OpenAI / Azure OpenAI | `uv add langchain-openai` | `pip install -U langchain-openai` |
| Anthropic (Claude) | `uv add langchain-anthropic` | `pip install -U langchain-anthropic` |
| DeepSeek | `uv add langchain-deepseek` | `pip install -U langchain-deepseek` |
| Ollama (local models) | `uv add langchain-ollama` | `pip install -U langchain-ollama` |
| SQLite checkpointer/store | `uv add langgraph-checkpoint-sqlite` | `pip install -U langgraph-checkpoint-sqlite` |
| PostgreSQL checkpointer/store | `uv add langgraph-checkpoint-postgres "psycopg[binary]"` | `pip install -U langgraph-checkpoint-postgres "psycopg[binary]"` |
| LangSmith SDK (custom tracing, see `langsmith-tracing`) | `uv add langsmith` | `pip install -U langsmith` |

`langchain` already depends on `langgraph` and `langchain-core`. The extras form `langchain[openai]` and `langchain[anthropic]` installs the core package plus that provider in one step. Pin compatible lower bounds in the project (for example `langchain>=1.4,<2`) rather than exact versions, unless the project already pins exact versions.

For other providers (Google Gemini, AWS Bedrock, Azure, Mistral, Groq, OpenRouter, and more), look up the package and model string on `https://docs.langchain.com/oss/python/integrations/providers/overview.md`.

## Configure credentials

Load secrets from the environment or the project's settings object (for example `pydantic-settings`). Add placeholders to `.env.example`; never commit real values.

```dotenv
# Pick the providers you use
OPENAI_API_KEY=
ANTHROPIC_API_KEY=
DEEPSEEK_API_KEY=
# Ollama runs locally; override only for a remote server
OLLAMA_HOST=http://localhost:11434

# Application-level model choice ("provider:model")
LLM_MODEL=openai:gpt-5.5
```

## Initialize a model

Use `init_chat_model` for provider-agnostic code. It accepts `"provider:model"` strings and standard parameters, and returns the provider's chat model class.

```python
from langchain.chat_models import init_chat_model

model = init_chat_model(
    settings.llm_model,          # e.g. "openai:gpt-5.5", "anthropic:claude-sonnet-4-6",
                                 # "deepseek:deepseek-chat", "ollama:gpt-oss:20b"
    timeout=60,                  # seconds
    max_retries=6,               # default 6; retries 429/5xx/network errors with backoff
    max_tokens=2048,
)
response = model.invoke("Why do parrots talk?")
print(response.text)
```

`create_agent(model=...)` also accepts the same string directly. Build the model instance yourself when you need provider-specific parameters, a rate limiter, or a shared instance.

Standard parameters: `model`, `api_key`, `temperature`, `max_tokens`, `timeout`, `max_retries`, `base_url`, `rate_limiter`. Check provider pages for the rest.

### OpenAI

```python
from langchain_openai import ChatOpenAI

model = ChatOpenAI(
    model="gpt-5.5",
    stream_usage=True,          # include token usage when streaming (Chat Completions)
    # reasoning={"effort": "medium", "summary": "auto"},  # reasoning models; routes to the Responses API
    # use_responses_api=True,   # opt into the Responses API explicitly
)
```

- Env: `OPENAI_API_KEY`. Base URL resolution order: the `base_url` argument, then `OPENAI_API_BASE`, then `OPENAI_BASE_URL`.
- Any OpenAI-compatible server (vLLM, Together, LiteLLM proxy) can be used with `init_chat_model(model, model_provider="openai", base_url=..., api_key=...)`. Prefer a dedicated integration when one exists (for example `langchain-openrouter`), because provider-specific fields may be dropped otherwise.
- `max_tokens` is converted to `max_completion_tokens` automatically.
- Azure OpenAI uses `AzureChatOpenAI` / `init_chat_model("azure_openai:...", azure_deployment=...)` with `AZURE_OPENAI_API_KEY`, `AZURE_OPENAI_ENDPOINT`, and `OPENAI_API_VERSION`.

### Anthropic (Claude)

```python
from langchain_anthropic import ChatAnthropic

model = ChatAnthropic(
    model="claude-sonnet-4-6",
    max_tokens=4096,
    # thinking={"type": "enabled", "budget_tokens": 2000},  # extended thinking (budgeted)
    # thinking={"type": "adaptive"},                        # adaptive thinking on newer Opus models
    # effort="medium",                                      # effort control on supported models
)
```

- Env: `ANTHROPIC_API_KEY`.
- Newer Claude models (`claude-sonnet-5`, `claude-opus-5`, `claude-opus-4-7`, `claude-opus-4-8`, `claude-fable-5`) reject non-default `temperature`, `top_p`, and `top_k` with a 400 error. Omit sampling parameters for these models and steer style through the prompt. When upgrading a model string, remove sampling parameters instead of changing their values.
- Native structured output: `with_structured_output(Schema, method="json_schema")`, or `ProviderStrategy(Schema)` in an agent.
- Prompt caching: use `AnthropicPromptCachingMiddleware` (from `langchain_anthropic.middleware`) for agents, or `cache_control` content blocks for direct calls. Cache reads and writes appear in `usage_metadata["input_token_details"]`.
- Token usage is streamed by default (`stream_usage=True`).

### DeepSeek

```python
from langchain_deepseek import ChatDeepSeek

model = ChatDeepSeek(model="deepseek-chat", temperature=0, max_retries=2)
```

- Env: `DEEPSEEK_API_KEY`; optional `DEEPSEEK_API_BASE` (or `api_base=`) for a different endpoint.
- The integration docs state that `deepseek-chat` supports tool calling and structured output and that `deepseek-reasoner` does not. Before using a DeepSeek model in an agent, confirm tool-calling support for that exact model on the ChatDeepSeek page and in DeepSeek's API docs. For a reasoning model without tool support, use it only for plain generation, or pair it with a tool-capable model.
- No image input. DeepSeek open-weight models can also run through Ollama.

### Ollama (local)

```python
from langchain_ollama import ChatOllama

model = ChatOllama(
    model="gpt-oss:20b",           # must match the tag you pulled
    temperature=0,
    validate_model_on_init=True,   # fail fast when the model is not pulled
    # base_url="http://gpu-box:11434",  # defaults to OLLAMA_HOST or localhost:11434
    # num_ctx=16384,                # context window; Ollama's default is small
    # reasoning=True,               # thinking models: reasoning goes to additional_kwargs["reasoning_content"]
)
```

- Setup: install Ollama, run `ollama pull gpt-oss:20b`, and check with `ollama list`. Use the same tag in code.
- Only models tagged with tool support on the Ollama library (`https://ollama.com/search?c=tools`) can drive an agent. Small local models often call tools unreliably: prefer `ToolStrategy` over native structured output, keep tool counts small, and test carefully.
- Raise `num_ctx` for agents with long histories or many tools. Otherwise the prompt is silently truncated.
- Ollama's LangChain integration does not report token usage in the model features table. Do not rely on `usage_metadata` for cost accounting with Ollama.

## Switching and combining providers

- Keep provider choice in configuration: `init_chat_model(settings.llm_model)`.
- For runtime switching, use a configurable model: `init_chat_model(configurable_fields=("model", "model_provider", "temperature"))`, then pass `config={"configurable": {"model": "anthropic:claude-sonnet-4-6"}}`. Inside agents, prefer the dynamic-model middleware pattern (`@wrap_model_call` plus `request.override(model=...)`).
- For resilience across providers, add `ModelFallbackMiddleware("anthropic:claude-sonnet-4-6", "openai:gpt-5.4-mini")` to the agent.
- Inspect capabilities through `model.profile` (for example `max_input_tokens`, `tool_calling`, `structured_output`). When the profile data is missing or wrong, pass a `profile={...}` override, because summarization triggers and structured-output strategy selection rely on it.

## Standard errors

Integrations raise shared exception types from `langchain_core.exceptions`. Each type also inherits from the provider SDK's own error type:

| Exception | Retryable |
| --- | --- |
| `ModelAuthenticationError`, `ModelPermissionDeniedError`, `ModelInvalidRequestError`, `ModelNotFoundError`, `ContextOverflowError` | No. Fix the configuration or the input. |
| `ModelRateLimitError`, `ModelAPIError`, `ModelConnectionError`, `ModelTimeoutError` | Yes. `ModelRetryMiddleware` retries these by default. |

Catch these types at the application boundary and return a controlled error. Do not leak provider responses to end users.

## Rate limiting

```python
from langchain.rate_limiters import InMemoryRateLimiter

limiter = InMemoryRateLimiter(requests_per_second=2, check_every_n_seconds=0.1, max_bucket_size=10)
model = init_chat_model("openai:gpt-5.5", rate_limiter=limiter)
```

This limiter is per process and counts requests, not tokens. Use a shared gateway or queue when many processes need a combined limit.
