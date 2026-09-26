# Setup

## Install

LangChain and LangGraph already depend on `langsmith`, so tracing works without extra packages. Add it explicitly when the code imports it (`@traceable`, `Client`, wrappers):

```bash
uv add langsmith          # or: pip install -U langsmith
```

## Environment variables

```dotenv
LANGSMITH_TRACING=true                 # turn tracing on
LANGSMITH_API_KEY=                     # from LangSmith settings; keep it in a secret manager in production
LANGSMITH_PROJECT=my-app-dev           # traces go to "default" when unset
# LANGSMITH_ENDPOINT=https://eu.api.smith.langchain.com   # non-US region or self-hosted; no trailing slash
# LANGSMITH_WORKSPACE_ID=              # required when one API key has access to several workspaces
# LANGSMITH_TRACING_SAMPLING_RATE=1.0  # 0..1 fraction of traces to keep
# LANGSMITH_HIDE_INPUTS=false          # hide all inputs
# LANGSMITH_HIDE_OUTPUTS=false         # hide all outputs
# LANGCHAIN_CALLBACKS_BACKGROUND=true  # set to false in serverless functions so traces finish before exit
```

| Region | `LANGSMITH_ENDPOINT` |
| --- | --- |
| GCP US (default) | `https://api.smith.langchain.com` |
| GCP EU | `https://eu.api.smith.langchain.com` |
| GCP APAC | `https://apac.api.smith.langchain.com` |
| AWS US | `https://aws.api.smith.langchain.com` |
| Self-hosted | Your instance's API URL |

An API key for the wrong region fails authentication. Set the endpoint to match the account.

Older code may use `LANGCHAIN_TRACING_V2`, `LANGCHAIN_API_KEY`, and `LANGCHAIN_PROJECT`. They still work, but new configuration should use the `LANGSMITH_*` names. Do not set both families with conflicting values.

## Load them in the application

Load environment variables before LangChain objects are created (for example, `load_dotenv()` at the entry point, or the settings class). Tracing configuration is read from the process environment, so a settings class alone does not enable it unless it exports the values. The clearest approach is to keep them as real environment variables in each deployment environment.

Suggested project naming: `<app>-<environment>` (`support-bot-dev`, `support-bot-staging`, `support-bot-prod`). Keep production traces in their own project (or workspace) with restricted access.

## Configure in code (no environment variables)

```python
import langsmith as ls

client = ls.Client(
    api_key=secrets.get("langsmith_api_key"),
    api_url="https://api.smith.langchain.com",
)

with ls.tracing_context(client=client, project_name="support-bot-prod", enabled=True):
    agent.invoke(inputs, config)
```

Process-wide defaults: `ls.configure(client=client, enabled=True, project_name="...")`. Precedence: `tracing_context(enabled=...)` beats `ls.configure(enabled=...)`, which beats the environment variables.

## Choose a project per call

```python
with ls.tracing_context(project_name="support-bot-evals"):
    agent.invoke(inputs, config)
```

`@traceable(project_name=...)` does the same for a decorated function. Use this to separate experiments or background jobs from interactive traffic.

## Quick verification

```python
import os, langsmith as ls
from langchain.chat_models import init_chat_model

assert os.getenv("LANGSMITH_TRACING") == "true" and os.getenv("LANGSMITH_API_KEY")
init_chat_model("openai:gpt-5.4-mini").invoke("ping", config={"run_name": "tracing_smoke_test"})
from langchain_core.tracers.langchain import wait_for_all_tracers
wait_for_all_tracers()   # make sure the run is sent before this script exits
```

Then open the project in LangSmith and confirm the `tracing_smoke_test` run is there.
