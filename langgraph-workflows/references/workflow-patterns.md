# Workflow and Agent Patterns

Templates for common graph shapes. Replace the placeholder prompts and schemas with domain logic. In all examples, `llm = init_chat_model(settings.llm_model)` from `langchain.chat_models`, and imports come from `langgraph.graph` (`StateGraph`, `START`, `END`, `MessagesState`) and `langgraph.types` (`Send`, `Command`).

## Choosing a shape

| Shape | Use when | Key mechanism |
| --- | --- | --- |
| Prompt chaining | A task splits into fixed sequential steps, with quality gates | Linear edges plus a conditional gate |
| Parallelization | Independent subtasks can run at once | Several static edges from one node, joined afterwards |
| Routing | Inputs fall into categories that need different handling | Structured-output classifier plus conditional edges |
| Orchestrator-worker | The number of subtasks is only known at runtime | `Send` fan-out plus a reducer key |
| Evaluator-optimizer | Output must meet criteria and can be improved from feedback | A loop with a structured critique and a max-iteration exit |
| Agent loop | The model must decide which tools to call and when to stop | Model node, `ToolNode`, `tools_condition` |
| Agent as a node | Agent behavior inside a larger deterministic flow | A `create_agent` graph added as a node |

## Prompt chaining with a gate

```python
class ChainState(TypedDict):
    topic: str
    outline: str
    draft: str
    final: str


def make_outline(state: ChainState):
    return {"outline": llm.invoke(f"Outline an article about {state['topic']}").text}

def outline_ok(state: ChainState) -> Literal["write_draft", "make_outline"]:
    return "write_draft" if state["outline"].count("\n") >= 3 else "make_outline"   # deterministic check

def write_draft(state: ChainState):
    return {"draft": llm.invoke(f"Write the article from this outline:\n{state['outline']}").text}

def polish(state: ChainState):
    return {"final": llm.invoke(f"Edit for clarity and concision:\n{state['draft']}").text}

chain = (
    StateGraph(ChainState)
    .add_node("make_outline", make_outline)
    .add_node("write_draft", write_draft)
    .add_node("polish", polish)
    .add_edge(START, "make_outline")
    .add_conditional_edges("make_outline", outline_ok, ["write_draft", "make_outline"])
    .add_edge("write_draft", "polish")
    .add_edge("polish", END)
    .compile()
)
```

A gate that loops back needs a bound. Add an attempt counter or rely on `recursion_limit`.

## Parallelization (fixed branches)

```python
class ParState(TypedDict):
    text: str
    sentiment: str
    entities: list[str]
    summary: str
    report: str


builder = StateGraph(ParState)
builder.add_node("sentiment", lambda s: {"sentiment": classify_sentiment(s["text"])})
builder.add_node("entities", lambda s: {"entities": extract_entities(s["text"])})
builder.add_node("summary", lambda s: {"summary": summarize(s["text"])})
builder.add_node("aggregate", lambda s: {"report": render(s)})
for branch in ("sentiment", "entities", "summary"):
    builder.add_edge(START, branch)
builder.add_edge(["sentiment", "entities", "summary"], "aggregate")   # waits for all three
builder.add_edge("aggregate", END)
graph = builder.compile()
```

The branches write different keys, so no reducer is needed. If they write the same key, give it a reducer. Parallel branches run in the same super-step. One failing branch fails the step unless it has a retry policy or error handler.

## Routing

```python
from pydantic import BaseModel


class Route(BaseModel):
    destination: Literal["billing", "technical", "general"]


router_llm = llm.with_structured_output(Route)


class RouteState(TypedDict):
    question: str
    destination: str
    answer: str


def classify(state: RouteState):
    return {"destination": router_llm.invoke(
        [("system", "Route the support question."), ("human", state["question"])]
    ).destination}

def pick(state: RouteState) -> Literal["billing", "technical", "general"]:
    return state["destination"]

builder = StateGraph(RouteState)
builder.add_node("classify", classify)
for name, handler in {"billing": billing_node, "technical": tech_node, "general": general_node}.items():
    builder.add_node(name, handler)
    builder.add_edge(name, END)
builder.add_edge(START, "classify")
builder.add_conditional_edges("classify", pick, ["billing", "technical", "general"])
graph = builder.compile()
```

To route to several handlers at once, return a list of node names, or `Send` objects, from the routing function.

## Orchestrator-worker (dynamic fan-out)

```python
import operator
from typing import Annotated


class Section(BaseModel):
    name: str
    description: str


class Plan(BaseModel):
    sections: list[Section]


class ReportState(TypedDict):
    topic: str
    sections: list[Section]
    completed: Annotated[list[str], operator.add]   # every worker appends here
    report: str


class WorkerState(TypedDict):
    section: Section


def orchestrate(state: ReportState):
    plan = llm.with_structured_output(Plan).invoke(f"Plan report sections for: {state['topic']}")
    return {"sections": plan.sections}

def assign(state: ReportState):
    return [Send("write_section", {"section": s}) for s in state["sections"]]

def write_section(state: WorkerState):
    s = state["section"]
    return {"completed": [f"## {s.name}\n" + llm.invoke(f"Write: {s.description}").text]}

def synthesize(state: ReportState):
    return {"report": "\n\n".join(state["completed"])}

graph = (
    StateGraph(ReportState)
    .add_node("orchestrate", orchestrate)
    .add_node("write_section", write_section)
    .add_node("synthesize", synthesize)
    .add_edge(START, "orchestrate")
    .add_conditional_edges("orchestrate", assign, ["write_section"])
    .add_edge("write_section", "synthesize")
    .add_edge("synthesize", END)
    .compile()
)
```

Workers may finish in any order. Sort results when order matters, for example by carrying an index in the `Send` payload. Cap the fan-out size (for example with `max_concurrency` in the config, or by limiting the plan) to control cost and rate limits.

## Evaluator-optimizer loop

```python
class Critique(BaseModel):
    passed: bool
    feedback: str


class LoopState(TypedDict):
    task: str
    draft: str
    feedback: str
    attempts: int


def generate(state: LoopState):
    prompt = state["task"] + (f"\nFix this feedback: {state['feedback']}" if state.get("feedback") else "")
    return {"draft": llm.invoke(prompt).text, "attempts": state.get("attempts", 0) + 1}

def evaluate(state: LoopState) -> Command[Literal["generate", "__end__"]]:
    verdict = llm.with_structured_output(Critique).invoke(f"Grade strictly:\n{state['draft']}")
    if verdict.passed or state["attempts"] >= 3:
        return Command(goto=END)
    return Command(update={"feedback": verdict.feedback}, goto="generate")

graph = (
    StateGraph(LoopState)
    .add_node("generate", generate)
    .add_node("evaluate", evaluate)
    .add_edge(START, "generate")
    .add_edge("generate", "evaluate")
    .compile()
)
```

A human can play the evaluator: call `interrupt({"draft": ...})` in the evaluate node (see `persistence-hitl.md`).

## Agent loop with ToolNode

```python
from langchain.tools import tool
from langgraph.prebuilt import ToolNode, tools_condition


@tool
def search_docs(query: str) -> str:
    """Search internal documentation."""
    return docs_index.search(query)


tools = [search_docs]
model_with_tools = llm.bind_tools(tools)


def call_model(state: MessagesState):
    return {"messages": [model_with_tools.invoke([("system", SYSTEM_PROMPT), *state["messages"]])]}


agent = (
    StateGraph(MessagesState)
    .add_node("model", call_model)
    .add_node("tools", ToolNode(tools))                 # parallel tool calls, error handling, ToolRuntime injection
    .add_edge(START, "model")
    .add_conditional_edges("model", tools_condition)    # -> "tools" if tool calls, else END
    .add_edge("tools", "model")
    .compile(checkpointer=checkpointer)
)
```

- By default, `ToolNode` returns only argument-validation errors (`ToolInvocationError`) to the model. Any other exception raised by a tool is re-raised. Pass `handle_tool_errors=True` to send every tool exception back as a tool message, a string or callable to customize the message, an exception type (or tuple of types) to catch selectively, or `False` to always raise.
- Tools in a `ToolNode` can read graph state and context through a `ToolRuntime[Ctx, State]` parameter.
- Prefer `langchain.agents.create_agent` when you do not need a custom loop. It includes middleware, structured output, and the same persistence support.

## Agent as a node inside a workflow

```python
from langchain.agents import AgentState, create_agent

research_agent = create_agent(llm, tools=[search_docs], name="researcher")


class Flow(AgentState):
    ticket_id: str
    resolution: str


def load_ticket(state: Flow):
    return {"messages": [("user", tickets.get(state["ticket_id"]).body)]}

def save(state: Flow):
    return {"resolution": state["messages"][-1].text}

graph = (
    StateGraph(Flow)
    .add_node("load_ticket", load_ticket)
    .add_node("research", research_agent)   # compiled graph as a node; shares the messages key
    .add_node("save", save)
    .add_edge(START, "load_ticket")
    .add_edge("load_ticket", "research")
    .add_edge("research", "save")
    .add_edge("save", END)
    .compile(checkpointer=checkpointer)
)
```

Middleware (human-in-the-loop, summarization, retries) keeps working inside the embedded agent. When the parent and the agent should not share message history, call the agent inside a wrapper node and map the inputs and outputs explicitly (see `multi-agent.md`).

## Retrieval-augmented generation (agentic RAG)

Common layout: `retrieve` (a tool or node) → `grade_documents` (structured yes/no) → either `generate` or `rewrite_question` → back to `retrieve`, with an attempt cap. Keep retrieved documents in state as references or short snippets, and cite sources in the answer. See `https://docs.langchain.com/oss/python/langgraph/agentic-rag.md` for a full example.
