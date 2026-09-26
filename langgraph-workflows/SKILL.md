---
name: langgraph-workflows
description: Design and implement complex Python AI workflows and multi-agent systems with LangGraph `StateGraph`, including state and reducers, nodes, conditional edges, `Send` fan-out, `Command` routing, subgraphs, supervisor/handoff/router patterns, human-in-the-loop interrupts, persistence (checkpointers and stores on SQLite or PostgreSQL), time travel, retries, timeouts, error handlers, streaming, testing, and `langgraph.json` deployment layout. Use when a task needs explicit control flow, parallel steps, long-running durable execution, or several cooperating agents beyond a single LangChain `create_agent` loop.
metadata:
  author: "mohammedashrafdagga"
  version: "1.0.0"
  supported-agents: "codex,claude-code,cursor"
  library-versions: "langgraph>=1.2,langchain>=1.4"
---

# LangGraph Workflows

Build explicit, durable orchestration with LangGraph in Python. A graph is a shared **state**, **nodes** that do work and return partial state updates, and **edges** that choose what runs next. Use LangGraph when the control flow matters: deterministic steps mixed with LLM steps, branching, parallelism, loops with exit conditions, pauses for humans, and several agents with clear responsibilities.

## Decide the architecture first

1. **Could a single `create_agent` with the right tools and middleware do it?** If yes, use the `langchain-agents` skill. Multi-agent systems add latency, cost, and failure modes.
2. **Is the path known in advance?** Build a **workflow** (prompt chaining, parallelization, routing, orchestrator-worker, evaluator-optimizer) with deterministic edges and LLM calls inside nodes.
3. **Must the model choose the path?** Use an **agent loop** (a model node plus `ToolNode`), or a `create_agent` graph as a node.
4. **Several specialized agents?** Choose from subagents-as-tools (supervisor), handoffs, a router with parallel fan-out, or a custom workflow. Read [references/multi-agent.md](references/multi-agent.md) before choosing.

Sketch the graph before coding. List the nodes, the state keys each node reads and writes, the reducers for keys written by several nodes, every exit condition, where a human must approve, and which steps have side effects.

## Workflow

1. **Inspect the project**: Python 3.10+, dependency tool, existing LangChain/LangGraph versions, sync or async runtime, database availability, and where settings live. Install `langgraph` (with `langchain` and a provider package if nodes call models). Provider setup is described in the `langchain-agents` skill's setup reference.
2. **Define state and graph structure.** Read [references/graph-api.md](references/graph-api.md) for state schemas, reducers, `MessagesState`, nodes, `Runtime` context, edges, `Send`, `Command`, recursion limits, and caching.
3. **Implement the pattern.** Read [references/workflow-patterns.md](references/workflow-patterns.md) for tested templates of each workflow and agent-loop pattern.
4. **Add persistence and human-in-the-loop** when runs must resume, remember, or wait for people. Read [references/persistence-hitl.md](references/persistence-hitl.md) for checkpointers, stores, `interrupt()`, resuming with `Command(resume=...)`, state inspection, time travel, and the backend choice (in-memory for tests, SQLite locally, PostgreSQL in production).
5. **Make it reliable, observable, and deployable.** Read [references/reliability-streaming-deploy.md](references/reliability-streaming-deploy.md) for `RetryPolicy`, timeouts, error handlers, streaming, testing, project layout, and `langgraph.json`. Enable tracing with the `langsmith-tracing` skill.
6. **Verify.** Test each node in isolation with `graph.nodes["name"].invoke(...)`, and test end-to-end paths with a fresh `InMemorySaver` per test and fake chat models. Cover every conditional branch, the interrupt/resume path, and failure handlers. Run the project's test and lint commands.

## Non-negotiable rules

- Nodes return **partial updates** (a dict of changed keys) or a `Command`. They never mutate the incoming state in place.
- Give every key written by parallel branches or several nodes a reducer (`Annotated[list, operator.add]`, `add_messages`). Otherwise concurrent writes raise `INVALID_CONCURRENT_GRAPH_UPDATE`.
- For each node, route with **one** mechanism: static `add_edge`, `add_conditional_edges`, or a returned `Command(goto=...)`. Static edges still run alongside a dynamic `goto`. Annotate `Command[Literal["a", "b"]]` return types.
- Every loop needs an exit condition plus a `recursion_limit` (and optionally `RemainingSteps`) backstop.
- `interrupt()` and resuming require a checkpointer and a `thread_id`. Resume only with `Command(resume=...)`; continue a finished conversation with a normal input dict. Never wrap `interrupt()` in `try/except`, never reorder `interrupt()` calls within a node, and make side effects before an interrupt idempotent, because the node reruns from its beginning when resumed.
- Nodes may re-execute after failures, retries, and resumes. Make writes idempotent (upserts, idempotency keys), and move multi-step side effects into separate nodes or `@task`s.
- Pass run-scoped dependencies and identity through `context_schema` and `Runtime[Context]`, not state and not globals. Keep secrets out of state, because state is checkpointed and traced.
- Use durable checkpointers and stores (SQLite or PostgreSQL) outside tests. `InMemorySaver` loses every thread on restart.
- Use async nodes (`async def`, `ainvoke`) on async servers. Node `timeout=` works only on async nodes.
- Do not use deprecated patterns in new code: `langgraph.prebuilt.create_react_agent` (use `langchain.agents.create_agent`), `set_entry_point`/`set_finish_point` (use edges from `START` and to `END`), and `config["configurable"]` for runtime dependencies (use `context`).

## Look up current documentation

These instructions were verified against `langgraph` 1.2 and `langchain` 1.4. For anything not covered, or when versions differ, read the official Markdown docs. The index is at `https://docs.langchain.com/llms.txt`, and key pages include `https://docs.langchain.com/oss/python/langgraph/graph-api.md`, `.../use-graph-api.md`, `.../workflows-agents.md`, `.../persistence.md`, `.../interrupts.md`, `.../use-subgraphs.md`, `.../fault-tolerance.md`, `.../streaming.md`, and `https://docs.langchain.com/oss/python/langchain/multi-agent/index.md`.

## Finish

Report the graph topology (a Mermaid diagram from `graph.get_graph().draw_mermaid()` helps reviewers), the state schema and reducers, the persistence backend and its setup steps, the human approval points, the retry/timeout/error policies, the required packages and environment variables, and the tests added. Call out any path that was not verified against a real model.
