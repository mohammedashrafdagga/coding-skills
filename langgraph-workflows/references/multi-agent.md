# Multi-Agent Systems

Use several agents only for a concrete reason: one agent has too many tools or too much context to choose well, separate teams own separate capabilities, work can run in parallel, or steps must unlock in a fixed order. The core design problem is **context engineering**: deciding exactly what each agent sees.

## Pick a pattern

| Pattern | How it works | Strengths | Costs |
| --- | --- | --- | --- |
| **Subagents (supervisor)** | A main agent calls specialist agents wrapped as tools | Central control, context isolation, parallel calls, independent teams | One extra model call per delegation; subagents cannot talk to the user directly |
| **Handoffs** | A tool call changes an `active_agent` state value; control moves to that agent, which talks to the user | Natural multi-turn specialist conversations, fewer calls on repeat turns | Careful message-history handling; less central oversight |
| **Skills** | One agent loads specialized prompts or knowledge on demand | Simple, cheap, one agent in control | Everything shares one context window |
| **Router** | A classifier step sends the input to one or more agents (in parallel with `Send`), then results are merged | Fast parallel fan-out, predictable | Stateless per request; no multi-hop reasoning |
| **Custom workflow** | A `StateGraph` mixing deterministic nodes and agents | Full control; can embed the other patterns | You own the design and testing |

Start with the simplest pattern that meets the requirement. The patterns combine: for example, a supervisor whose tools include a router workflow.

## Subagents as tools (supervisor)

```python
from langchain.agents import create_agent
from langchain.tools import tool

calendar_agent = create_agent(llm, tools=[list_events, create_event],
                              system_prompt="You manage calendars. Return a short factual result.", name="calendar")
email_agent = create_agent(llm, tools=[search_email, draft_email],
                           system_prompt="You handle email. Never send without explicit instruction.", name="email")


@tool("calendar", description="Schedule, move, or look up calendar events. Input: a self-contained request.")
def ask_calendar(request: str) -> str:
    result = calendar_agent.invoke({"messages": [{"role": "user", "content": request}]})
    return result["messages"][-1].text


@tool("email", description="Search or draft emails. Input: a self-contained request.")
def ask_email(request: str) -> str:
    result = email_agent.invoke({"messages": [{"role": "user", "content": request}]})
    return result["messages"][-1].text


supervisor = create_agent(llm, tools=[ask_calendar, ask_email], checkpointer=checkpointer, name="supervisor",
                          system_prompt="Delegate to specialists. Combine their answers for the user.")
```

- Tool descriptions are the routing logic. Make them specific and non-overlapping.
- Inputs: by default pass only the task. Pass more context deliberately, for example by reading `runtime.state` through a `ToolRuntime` parameter.
- Outputs: return a concise result, not the subagent's full transcript. Ask subagents for a final summary format.
- With many subagents, use one dispatch tool (`task(agent_name: Literal[...], request: str)`) instead of one tool per agent.
- Subagent persistence: by default (per-invocation), each call starts fresh but inherits the parent's checkpointer, so interrupts inside subagents work. Compile or create a subagent with `checkpointer=True` for per-thread memory. Do not call the same per-thread subagent in parallel (limit it with `ToolCallLimitMiddleware`), or checkpoints conflict.
- Name every agent (`name=`) so traces and `stream.subagents` label them.

## Handoffs

**Single agent whose behavior changes with state** (preferred for most handoff needs): a tool updates a `current_step` or `active_agent` value, and `@wrap_model_call` middleware swaps the system prompt and tools accordingly:

```python
from langchain.agents import AgentState, create_agent
from langchain.agents.middleware import wrap_model_call
from langchain.messages import ToolMessage
from langchain.tools import tool, ToolRuntime
from langgraph.types import Command
from typing_extensions import NotRequired


class SupportState(AgentState):
    current_step: NotRequired[str]


@tool
def escalate_to_specialist(reason: str, runtime: ToolRuntime) -> Command:
    """Hand the conversation to a technical specialist."""
    return Command(update={
        "current_step": "specialist",
        "messages": [ToolMessage(f"Escalated: {reason}", tool_call_id=runtime.tool_call_id)],
    })


STEPS = {
    "triage": ("Collect the problem details, then escalate if technical.", [escalate_to_specialist]),
    "specialist": ("You are a senior engineer. Diagnose and fix.", [run_diagnostics, open_ticket]),
}


@wrap_model_call
def apply_step(request, handler):
    prompt, tools = STEPS[request.state.get("current_step") or "triage"]
    return handler(request.override(system_prompt=prompt, tools=tools))


agent = create_agent(llm, tools=[escalate_to_specialist, run_diagnostics, open_ticket],
                     state_schema=SupportState, middleware=[apply_step], checkpointer=checkpointer)
```

**Separate agent subgraphs**: each agent is a node. A handoff tool returns `Command(goto="other_agent", graph=Command.PARENT, update={...})`. Include in `messages` exactly the `AIMessage` that made the handoff call plus a `ToolMessage` answering it, so the next agent receives a valid history. Route after each agent: end when its last message is an `AIMessage` without tool calls, and otherwise go to `state["active_agent"]`. Every key updated through `Command.PARENT` needs a reducer in the parent state (`messages` via `AgentState` has one). Use this form only when agents need bespoke internals. Full example: `https://docs.langchain.com/oss/python/langchain/multi-agent/handoffs.md`.

## Router with parallel agents

```python
import operator
from typing import Annotated


class Classification(BaseModel):
    targets: list[Literal["docs", "tickets", "code"]]


class RouterState(TypedDict):
    question: str
    targets: list[str]
    answers: Annotated[list[str], operator.add]
    final: str


def classify(state: RouterState):
    c = llm.with_structured_output(Classification).invoke(f"Which sources answer: {state['question']}")
    return {"targets": c.targets or ["docs"]}


def route(state: RouterState) -> list[Send]:   # edge functions only read state
    return [Send(f"ask_{t}", {"question": state["question"]}) for t in state["targets"]]


def make_asker(agent):
    def ask(payload: dict):
        out = agent.invoke({"messages": [{"role": "user", "content": payload["question"]}]})
        return {"answers": [out["messages"][-1].text]}
    return ask


builder = StateGraph(RouterState)
builder.add_node("classify", classify)
for name, agent in {"docs": docs_agent, "tickets": tickets_agent, "code": code_agent}.items():
    builder.add_node(f"ask_{name}", make_asker(agent))
    builder.add_edge(f"ask_{name}", "synthesize")
builder.add_node("synthesize", lambda s: {"final": llm.invoke(f"Merge answers:\n{s['answers']}").text})
builder.add_edge(START, "classify")
builder.add_conditional_edges("classify", route, ["ask_docs", "ask_tickets", "ask_code"])
builder.add_edge("synthesize", END)
router = builder.compile()
```

A router is stateless per request. To keep conversation context, wrap the router graph as a tool of a stateful agent.

## Subgraph mechanics

| Parent and child state | Do this |
| --- | --- |
| Different schemas (private agent history) | Call `subgraph.invoke(mapped_input)` inside a wrapper node and map the result back to parent keys |
| Shared keys (for example `messages`) | `builder.add_node("child", compiled_subgraph)` directly |

Persistence of a subgraph, set with `.compile(checkpointer=...)` on the child:

| Setting | Behavior | Supports |
| --- | --- | --- |
| `None` (default) | Fresh per call; inherits the parent checkpointer | Interrupts, durable execution, and the same subgraph called several times |
| `True` | Keeps state across calls on the same thread | Multi-turn subagent memory; do not call the same instance in parallel |
| `False` | No checkpointing | Plain function-like calls only; no interrupts |

- The parent must have a checkpointer for subgraph interrupts and inspection.
- Inspect nested state with `graph.get_state(config, subgraphs=True)`.
- Stream nested output with `graph.stream(..., subgraphs=True, version="v2")` (`chunk["ns"]` gives the path), or `stream_events(..., version="v3")` with `stream.subgraphs` / `stream.subagents`.
- Name compiled subgraphs (`.compile(name="billing")`) for readable traces and streams.

## Context engineering rules

- Give each agent the minimum context for its job: a task description plus the essential facts, not the whole conversation.
- Keep message histories valid across boundaries: every tool call must have a response, and the final message returned to the user is an `AIMessage`.
- Summarize subagent work into the tool result instead of forwarding raw internal reasoning.
- Put shared durable knowledge in the store (namespaced), not in every prompt.
- Check authorization in each agent's tools. Delegation must not widen permissions: pass the caller's identity through `context`.
- Trace everything with LangSmith, using named agents and `thread_id` metadata, and evaluate routing decisions, because misrouting is the most common multi-agent failure.
