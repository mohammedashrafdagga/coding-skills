# Coding Skills

A public collection of reusable Agent Skills for software work. It is designed to be shared, installed, and improved by other developers. Each skill lives in its own top-level directory, so more skills can be added without changing the repository's installation model.

## Supported agents

This repository officially supports only:

- Codex
- Claude Code
- Cursor

The `skills` CLI supports many agents, but this collection is maintained and tested only for these three. Always use the explicit agent flags below; do not use `--all` or `--agent '*'`.

## Available skills

| Skill | Purpose |
| --- | --- |
| `software-security-baseline` | Reviews minimum practical security controls, then checks only changed and affected security surfaces on later branches or revisions. |
| `api-validation-principle` | Establishes a full API baseline, then validates changed and affected operations with concise issue-focused reports. |
| `clean-code-review` | Establishes a code-quality baseline, then reviews changed and affected features for maintainability and architecture issues. |
| `langchain-agents` | Builds Python agents with LangChain v1: providers (OpenAI, Anthropic, DeepSeek, Ollama), tools, middleware, structured output with retries, short- and long-term memory, streaming, and token usage. |
| `langgraph-workflows` | Builds complex Python workflows and multi-agent systems with LangGraph: state, nodes, routing, fan-out, subgraphs, human-in-the-loop, persistence, fault tolerance, and deployment layout. |
| `langsmith-tracing` | Adds LangSmith tracing to LangChain, LangGraph, and custom code: projects, metadata, threads, custom spans, cost tracking, masking, sampling, and flushing. |

## Install from the public repository

The repository is public, so no GitHub authentication is required to list or install its skills.

First, list the available skills:

```bash
npx skills add mohammedashrafdagga/coding-skills --list
```

Install one skill into the current application project:

```bash
npx skills add mohammedashrafdagga/coding-skills --skill software-security-baseline --agent codex --agent claude-code --agent cursor
```

Install every current and future skill from the collection:

```bash
npx skills add mohammedashrafdagga/coding-skills --skill '*' --agent codex --agent claude-code --agent cursor
```

Project scope is the default. Add `--global` to any install command to make the selected skills available across projects:

```bash
npx skills add mohammedashrafdagga/coding-skills --skill software-security-baseline --agent codex --agent claude-code --agent cursor --global
```

To update installed skills:

```bash
npx skills update
```

## Use the security skill

Ask the agent to use `software-security-baseline` and review the current application. For example:

```text
Use software-security-baseline to review this application before release.
```

The skill will:

1. determine the application and review scope;
2. ensure the application has a `security/` directory;
3. inspect earlier numbered security reports and their Git checkpoints;
4. run a full first review, then assess only changed and transitively affected security surfaces on comparable later revisions;
5. create the next concise `security/report_NNN.md`, recording the current branch and reviewed revision;
6. keep issue detail in the report while summarizing clean coverage by feature or trust boundary.

Security reports can contain sensitive architectural and vulnerability information. Keep the application repository and its reports access-controlled, and never place live secrets in a report.

## Use the API validation skill

Install it into the application project:

```bash
npx skills add mohammedashrafdagga/coding-skills --skill api-validation-principle --agent codex --agent claude-code --agent cursor
```

Ask the agent:

```text
Use api-validation-principle to review every API endpoint across this system and create an API report.
```

The first comparable run inventories all services and operations in scope. Later runs use the recorded Git branch and revision to validate only added, changed, removed, and transitively affected API behavior, including changes made on child branches. Each run creates the next `docs/api_report/report_NNN.md`, starting at `report_001.md` and preserving earlier reports.

Reports summarize passing coverage by feature or API surface. They keep endpoint-level detail for issues and evidence gaps, and link to earlier reports for unresolved findings that were not touched by the current change set.

A system passes only when inventory coverage is complete and every applicable check passes. Unverified endpoints remain visible in the report. To remediate findings, explicitly ask the agent to fix them and revalidate; an audit by itself changes only the reports.

## Use the clean-code skill

Install it into the application project:

```bash
npx skills add mohammedashrafdagga/coding-skills --skill clean-code-review --agent codex --agent claude-code --agent cursor
```

Ask the agent to use `clean-code-review` for a repository-wide quality and architecture assessment. For example:

```text
Use clean-code-review to assess the frontend and backend, including DDD and feature boundaries.
```

The first comparable run establishes full-project coverage. Later runs use the prior report's Git checkpoint to review changed code and its affected features, dependencies, and boundaries. Every run creates the next concise `docs/clean-code-report/report_NNN.md` with grouped coverage, evidence-backed issues, and links to unchanged earlier findings. Reviewing does not modify application code unless the user explicitly asks for fixes.

All three review skills fall back to a full review when history has diverged without a trustworthy reviewed base, the previous checkpoint is unavailable, or broad structural changes invalidate the earlier baseline.

## Use the LangChain, LangGraph, and LangSmith skills

These three skills are build guides for Python AI agents. They do not write reports. They guide the coding agent to install the right packages, write code against the current APIs (verified with `langchain` 1.4, `langgraph` 1.2, and `langsmith` 0.14), and add tests that use fake chat models, so no API key is needed for the test suite.

Install all three into the application project:

```bash
npx skills add mohammedashrafdagga/coding-skills --skill langchain-agents --skill langgraph-workflows --skill langsmith-tracing --agent codex --agent claude-code --agent cursor
```

| Skill | Use it when | Covers |
| --- | --- | --- |
| `langchain-agents` | You need an LLM feature or a tool-using agent | Installation with uv or pip; OpenAI, Anthropic, DeepSeek, and Ollama setup; `create_agent`; tools and runtime context; built-in and custom middleware; human approval; structured JSON output with validation retries; short-term memory (checkpointers) and long-term memory (stores) on in-memory, SQLite, or PostgreSQL backends; streaming; token usage |
| `langgraph-workflows` | The flow needs explicit steps, branches, parallel work, pauses, or several agents | `StateGraph` state and reducers; nodes and edges; `Send` and `Command`; workflow patterns (chaining, routing, parallelization, orchestrator-worker, evaluator-optimizer, agent loop); multi-agent supervisor, handoff, and router designs; subgraphs; interrupts and time travel; retries, timeouts, and error handlers; testing; `langgraph.json` |
| `langsmith-tracing` | You want to see, debug, or measure what the agent does | Environment and region setup; per-environment projects; run names, tags, and metadata; conversation threads; `@traceable` and SDK wrappers; token and cost tracking; masking and anonymizing data; sampling; flushing in scripts and serverless functions; distributed tracing |

Example prompts:

```text
Use langchain-agents to add a support agent with OpenAI, two tools, a Pydantic response format, and PostgreSQL memory.
```

```text
Use langgraph-workflows to build a refund workflow: classify the request, fan out checks in parallel, pause for manager approval, then issue or deny the refund.
```

```text
Use langsmith-tracing to trace the agent per conversation thread, tag runs by environment, and mask emails and API keys.
```

The skills refer to each other: agent code comes from `langchain-agents`, orchestration from `langgraph-workflows`, and observability from `langsmith-tracing`. Each skill links to the official Markdown documentation, so the coding agent can check APIs that are newer than these guides.

## Add future skills

Add each new skill as a sibling directory at the repository root. The directory name must match the `name` in its `SKILL.md` frontmatter.

```text
coding-skills/
├── README.md
├── api-validation-principle/
│   ├── SKILL.md
│   ├── agents/
│   │   └── openai.yaml
│   └── references/
│       ├── api-baseline.md
│       └── report-format.md
├── clean-code-review/
│   ├── SKILL.md
│   ├── agents/
│   │   └── openai.yaml
│   └── references/
│       ├── quality-baseline.md
│       └── report-format.md
├── langchain-agents/
│   ├── SKILL.md
│   ├── agents/
│   │   └── openai.yaml
│   └── references/
│       ├── setup-and-providers.md
│       ├── agents-tools-middleware.md
│       ├── structured-output.md
│       ├── memory.md
│       └── streaming-and-usage.md
├── langgraph-workflows/
│   ├── SKILL.md
│   ├── agents/
│   │   └── openai.yaml
│   └── references/
│       ├── graph-api.md
│       ├── workflow-patterns.md
│       ├── multi-agent.md
│       ├── persistence-hitl.md
│       └── reliability-streaming-deploy.md
├── langsmith-tracing/
│   ├── SKILL.md
│   ├── agents/
│   │   └── openai.yaml
│   └── references/
│       ├── setup.md
│       ├── langchain-langgraph.md
│       ├── custom-instrumentation.md
│       └── production.md
├── software-security-baseline/
│   ├── SKILL.md
│   ├── agents/
│   │   └── openai.yaml
│   └── references/
│       ├── minimum-baseline.md
│       └── report-format.md
└── future-skill/
    └── SKILL.md
```

After adding a skill, validate it and confirm repository discovery with:

```bash
npx skills add . --list
```

The collection follows the Agent Skills `SKILL.md` format. Files under a skill's `agents/` directory may provide agent-specific metadata and can be ignored by agents that do not use them.
