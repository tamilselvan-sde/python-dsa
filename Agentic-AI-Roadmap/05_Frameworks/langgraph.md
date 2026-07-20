# LangGraph Framework

> Author: **Tamilselvan** · ✉️ tamilselvan.sde@gmail.com · 🔗 [LinkedIn](https://www.linkedin.com/in/tamilselvan-ai/)
>

## 1. What is it?

LangGraph is an open-source library from LangChain Inc. (released January 2024) for building **stateful, multi-actor applications** with LLMs. It extends LangChain by modelling agent workflows as directed graphs with shared state, cycles, branching, and persistence. LangGraph enables patterns impossible with linear chains: loops, human-in-the-loop, recursive refinement, and dynamic multi-agent coordination.

## 2. Why do we need it?

Linear chains (A → B → C) cannot express real-world agentic behaviour:
- **Loops:** agents that retry, verify, or iteratively improve
- **Branching:** conditional routes based on LLM decisions
- **State persistence:** maintaining context across turns, sleeps, and resumptions
- **Human-in-the-loop:** pausing for approval before critical actions
- **Parallelism:** concurrent sub-agents with merged results

LangGraph provides a **programmable graph runtime** for all these patterns. Without it, teams build brittle state machines atop chains that cannot handle cycles or persistence.

## 3. Real-world Example

**Automated code review agent:**
1. Developer submits PR → graph starts at `analyze` node
2. LLM reads diff, classifies issues
3. Graph routes to `test` node → runs test suite
4. Tests fail → graph loops back to `fix` node
5. Agent fixes code → re-runs tests
6. All pass → graph routes to `approve` node
7. Human-in-the-loop pauses → reviewer approves
8. Graph routes to `merge` node → PR merges

## 4. Architecture Diagram (ASCII)

```
                    +--------------+
                    |    State     |
                    | (TypedDict)  |
                    +------+-------+
                           |
    +----------------------+-------------------------+
    |                      |                         |
+---v----+          +------v----+          +--------v-+
| Node A |--------->|  Node B   |--------->|  Node C  |
| (LLM)  |  edge    |  (Tool)   |  edge    |  (LLM)   |
+---+----+          +------+----+          +----+-----+
    |                      |                    |
    +----------------------+--------------------+
                           |
                   +-------v-------+
                   |  Conditional  |
                   |  Edge Router  |
                   +-------+-------+
                           |
             +-------------+-----------+
             |                         |
      +------v----+            +-------v----+
      | Loop (A)  |            |  __END__   |
      +-----------+            +------------+
```

## 5. Internal Working

LangGraph models computation as a **graph with typed state**. Core concepts:

1. **State:** A `TypedDict` or Pydantic model defining the mutable data flowing through the graph. Each node receives and updates state.
2. **Nodes:** Python functions (`state -> dict`) that perform a unit of work (LLM call, tool execution, condition check).
3. **Edges:** Connections between nodes. Regular edges always fire. Conditional edges invoke a router function that returns the next node name.
4. **Graph compilation:** `StateGraph(...)` collects nodes and edges, compiles into `CompiledGraph` with a topological schedule.
5. **Checkpointing:** After each node, the graph snapshots state to a durable store (in-memory, SQLite, Postgres). This enables pause/resume, error recovery, and rollback.
6. **Execution loop:** Start at `__start__` → traverse nodes via edges → stop at `__end__` or when recursion limit is hit.

## 6. Production Flow

```
     +--------+     +---------+     +----------+     +----------+
     | Build  |---->| Compile |---->| Runtime  |---->| Output   |
     | Graph  |     | Graph   |     | Loop     |     |          |
     +--------+     +---------+     +----+-----+     +----------+
                                        |
                               +--------+--------+
                               |  Node Execution  |
                               +--------+--------+
                                        |
                               +--------+--------+
                               |  Checkpoint     |
                               +--------+--------+
                                        |
                               +--------+--------+
                               |  Edge Routing   |
                               +--------+--------+
```

## 7. HLD

```
                   +-------------------------+
                   |   API Gateway           |
                   +-----------+-------------+
                               |
                 +-------------+-------------+
                 |             |             |
           +-----v----+  +----v----+  +-----v----+
           | Chat     |  | Agent   |  | Automtn  |
           | Service  |  | Service |  | Service  |
           +-----+----+  +----+----+  +-----+----+
                 |             |             |
                 +------+------+-------------+
                        |
              +---------v----------+
              | LangGraph Runtime  |
              | (Graph Executor    |
              |  + Checkpointer)   |
              +----+-------+-------+
                   |       |
           +-------v+  +---v--------+
           | LLM   |  | Checkpoint |
           | APIs  |  | Store      |
           +-------+  +------------+
```

## 8. LLD

```
StateGraph
  ├── add_node(name, function)     # Register computation node
  ├── add_edge(src, dst)           # Regular edge
  ├── add_conditional_edges(src, router_fn, mapping)
  ├── set_entry_point(name)        # Starting node
  ├── set_finish_point(name)       # Terminal node
  └── compile(checkpointer)        # Returns CompiledGraph

CompiledGraph
  ├── invoke(input, config)        # Synchronous execution
  ├── astream(input, config)       # Async streaming
  ├── get_state(config)            # Read current checkpoint
  ├── get_state_history(config)    # Read all checkpoints
  └── update_state(config, vals)  # Manual state injection

Checkpointer (ABC)
  ├── MemorySaver                 # In-memory (dev only)
  ├── SqliteSaver                 # SQLite persistence
  └── PostgresSaver               # Production Postgres
```

## 9. Python Implementation

```python
from typing import TypedDict, Literal, Annotated
from langgraph.graph import StateGraph, END
from langgraph.checkpoint.memory import MemorySaver
from langchain_openai import ChatOpenAI
import operator

# 1. Define state
class AgentState(TypedDict):
    input: str
    messages: Annotated[list, operator.add]
    iterations: int

# 2. Define nodes
def analyst(state: AgentState) -> dict:
    llm = ChatOpenAI(model="gpt-4", temperature=0)
    response = llm.invoke(f"Analyze this: {state['input']}")
    return {"messages": [response.content], "iterations": state["iterations"] + 1}

def executor(state: AgentState) -> dict:
    # Simulate tool execution
    return {"messages": ["Tool result: valid"]}

# 3. Define router
def should_continue(state: AgentState) -> Literal["executor", "__end__"]:
    if state["iterations"] < 3:
        return "executor"
    return "__end__"

# 4. Build and compile
builder = StateGraph(AgentState)
builder.add_node("analyst", analyst)
builder.add_node("executor", executor)
builder.set_entry_point("analyst")
builder.add_edge("executor", "analyst")
builder.add_conditional_edges(
    "analyst",
    should_continue,
    {"executor": "executor", "__end__": "__end__"}
)

graph = builder.compile(checkpointer=MemorySaver())

# 5. Execute
config = {"configurable": {"thread_id": "session-1"}}
result = graph.invoke(
    {"input": "Review this PR: main.py changed 200 lines",
     "messages": [],
     "iterations": 0},
    config
)
print(result["messages"])
```

## 10. Folder Structure

```
graph-app/
├── graphs/
│   ├── __init__.py
│   ├── code_review.py      # Graph definition
│   ├── customer_support.py
│   └── research_assistant.py
├── nodes/
│   ├── __init__.py
│   ├── llm_nodes.py        # LLM call nodes
│   ├── tool_nodes.py       # Tool execution nodes
│   └── human_nodes.py      # Human-in-the-loop nodes
├── edges/
│   ├── __init__.py
│   └── routers.py          # Conditional edge functions
├── state/
│   ├── __init__.py
│   └── schemas.py          # TypedDict state definitions
├── checkpointer/
│   └── postgres.py         # Checkpoint store config
├── api/
│   └── main.py             # FastAPI entry point
├── tests/
├── docker-compose.yml
└── pyproject.toml
```

## 11. Configuration

```python
from pydantic_settings import BaseSettings

class LangGraphConfig(BaseSettings):
    recursion_limit: int = 25
    max_concurrency: int = 5
    checkpoint_postgres_uri: str = "postgresql://user:pass@localhost:5432/langgraph"
    llm_model: str = "gpt-4"
    llm_temperature: float = 0
    verbose: bool = True

    class Config:
        env_prefix = "LANGGRAPH_"
        env_file = ".env"
```

## 12. Flowchart

```
                           Start
                             |
                       __start__
                             |
                      +------v------+
                +---->|   Node A    |
                |     +------+------+
                |            |
                |     +------v------+
                |     | Conditional |
                |     |   Router    |
                |     +------+------+
                |            |
                |     +------+------+--------+
                |     |             |        |
                |  +--v----+   +---v--+  +---v--+
                |  | Node B|   | Node C|  | END |
                |  +-------+   +---+---+  +-----+
                |___________________|
                      (loop)
```

## 13. Sequence Diagram

```
  User       Graph        Node_A       Node_B    Checkpointer
   |           |            |            |           |
   |--invoke-->|            |            |           |
   |           |--run------>|            |           |
   |           |            |--snapshot--|---------->|
   |           |<--result---|            |           |
   |           |--route---->|            |           |
   |           |            |--run------>|           |
   |           |            |            |--snapshot>|
   |           |            |<--result---|           |
   |           |--route---->|            |           |
   |           |   (loop)   |            |           |
   |           |    ...     |            |           |
   |<--result--|            |            |           |
```

## 14. Pros

- **Cyclic graphs** — supports loops without hacks (impossible in standard chains)
- **Checkpointing** — durable state enables pause/resume and failure recovery
- **Human-in-the-loop** — `interrupt_before` pauses graph execution for approval
- **Parallel fan-out** — multiple nodes execute concurrently, results merged
- **LangSmith integration** — full traceability of entire graph execution
- **Subgraph support** — compose graphs within graphs for modularity

## 15. Cons

- **Steep learning curve** — requires understanding graph theory and state management
- **No built-in UI** — debugging requires LangSmith or custom dashboards
- **Complex debugging** — cyclic graphs can produce confusing tracebacks
- **Smaller community** — newer and less adopted than core LangChain
- **State schema rigid** — changes require migration of all checkpoints

## 16. Alternatives

| Framework | When to Choose |
|-----------|---------------|
| Temporal.io | Enterprise workflow orchestration (not AI-specific) |
| Prefect | Data pipeline DAGs, Python-native |
| AutoGen | Multi-agent conversation patterns |
| CrewAI | Role-based agent teams, simpler abstraction |
| Direct `asyncio` | Simple parallel tool calls |

## 17. Performance Considerations

- **State size:** Keep state minimal — every byte is serialised to checkpointer. Use `operator.add` reducer to append without recopying.
- **Checkpointer latency:** MemorySaver ~1ms, SQLite ~5ms, Postgres ~20ms. For high-throughput, batch checkpoint writes.
- **Node isolation:** Each node runs synchronously within the graph. Offload long-running nodes to background workers via message queues.
- **Streaming:** Use `astream_events()` for token-level streaming — significantly reduces perceived latency.
- **Recursion limit:** Default 25 — set lower for simple graphs, higher for complex agents.

## 18. Scaling to Millions

1. **Stateless graph definitions:** Graph structure is static and cached. Only state is per-thread.
2. **Distributed checkpointer:** Use CockroachDB or YugabyteDB (Postgres-compatible distributed SQL) for geo-replicated checkpoint store.
3. **Thread sharding:** Partition threads by `thread_id` hash across multiple graph runner pods.
4. **Horizontal scaling:** Stateless graph workers behind a load balancer. Each pod loads graph definitions from a shared volume.
5. **Caching:** Cache LLM responses within a thread's session using `langchain.InMemoryCache` per `thread_id`.
6. **Bulkhead pattern:** Dedicated worker pools for different graph types (chat vs. analysis vs. long-running agents).

## 19. Failure Scenarios

1. **Infinite loop:** `recursion_limit` is the last line of defence. Add a timeout on graph execution via `asyncio.wait_for()`.
2. **State corruption:** Validate state schema at every node boundary using Pydantic models. Reject invalid state before writing checkpoint.
3. **Checkpointer outage:** Queue writes to a dead-letter queue (DLQ). Read-only mode serves stale checkpoints.
4. **LLM timeout:** Node-level retry with exponential backoff. If all retries fail, the conditional edge routes to a `fallback` node.
5. **Missing thread:** Return a new thread instead of error; log the stale thread_id for debugging.

## 20. Security

- **State isolation:** Thread-level state isolation. Graph A cannot read Graph B's state. Enforce via `thread_id` namespace.
- **Node sandboxing:** Danger nodes (code execution, shell) run in isolated containers via gVisor or Firecracker.
- **Checkpointer encryption:** Encrypt sensitive state fields at rest using column-level encryption in Postgres.
- **Input validation:** All graph inputs must pass Pydantic schema validation before execution.
- **Human-in-the-loop approval:** Critical actions (deploy, delete, transfer) require explicit human approval via interrupt.

## 21. Monitoring

- **LangSmith:** Full graph execution traces — per-node latency, edge traversals, state size, token counts.
- **Metrics:** Track `graph_invoke_count`, `node_latency` (p50/p95/p99), `checkpointer_write_latency`, `recursion_limit_hits`.
- **Logging:** Structured JSON per node execution — include `graph_id`, `thread_id`, `node_name`, `duration_ms`, `state_bytes`.
- **Distributed tracing:** OpenTelemetry propagation across graph nodes, LLM calls, and tool invocations.
- **Alerting:** Alert on `recursion_limit` hits > 1%, node error rate > 5%, checkpointer latency > 100ms p99.

## 22. Interview Questions

1. How is LangGraph's execution model fundamentally different from LangChain's `RunnableSequence`?
2. Explain the role of `Annotated[..., operator.add]` in state updates.
3. How does checkpointing enable human-in-the-loop patterns?
4. Design a graph for a customer support escalation agent with three tiers.
5. What happens when a conditional edge returns an undefined node name?
6. Compare subgraph composition vs. node abstraction for modular graph design.
7. How would you implement parallel execution of three nodes and merge their results?
8. Explain how `interrupt_before` works and how you resume a paused graph.

## 23. Cheat Sheet

| Task | Code |
|------|------|
| Build graph | `graph = StateGraph(MyState)` |
| Add node | `graph.add_node("name", my_fn)` |
| Add edge | `graph.add_edge("src", "dst")` |
| Conditional edge | `graph.add_conditional_edges("src", router, mapping)` |
| Compile | `app = graph.compile(checkpointer=MemorySaver())` |
| Invoke | `app.invoke(input, config)` |
| Interrupt before | `app = graph.compile(interrupt_before=["node_name"])` |
| Get state | `app.get_state(config)` |
| Resume | `app.invoke(None, config)` |

## History

LangGraph was created by the LangChain engineering team in late 2023 to address fundamental limitations in chain-based agent orchestration. The first commit was in December 2023, with public release in January 2024. Version 0.1 supported only basic state graphs. Checkpointing and memory saver arrived in March 2024. Human-in-the-loop with `interrupt_before` was added in May 2024. Subgraph support came in v0.2. The library was extracted from `langchain-experimental` into its own `langgraph` package in June 2024. By 2025, LangGraph Cloud provided managed hosting with Postgres checkpointer and automatic scaling. LangGraph is now the recommended way to build agents in the LangChain ecosystem.

## Core Components Explained

- **StateGraph:** The primary graph builder. Extends `Graph` with typed state management. All nodes receive and update the same state object.
- **MessageGraph:** A simpler subclass where state is a `list[BaseMessage]`. Useful for pure chat-based agents without complex state.
- **Checkpointer (BaseCheckpointSaver):** Persists graph state after each node. Supports `put()` and `get()` operations. Backends: MemorySaver (dev), SqliteSaver (single-node), PostgresSaver (production).
- **State reducer:** `operator.add` makes a field append-only (for messages). Custom reducers enable overwrite, merge, or conditional updates.
- **Interrupts:** `interrupt_before=["node"]` pauses before a node. Resume by calling `invoke()` with `None` input. The graph reads checkpoint and continues.
- **Subgraphs:** Nest a `StateGraph` inside a node. The subgraph has its own state, start, end, and checkpoint isolation.

## Installation

```bash
pip install langgraph
pip install langgraph-checkpoint-postgres  # for Postgres checkpointer
# Optional visualisation
pip install grandalf
```

## Advanced Features

- **Dynamic graph construction:** Add nodes at runtime based on LLM output. E.g., create a new node for each newly discovered entity.
- **Parallel branches:** `graph.add_node("branch", branch_fn)` connected to the same parent — they execute concurrently, results merged via state reducers.
- **Command pattern:** Nodes return a `Command` object (value + goto) instead of using edges, enabling dynamic routing.
- **Node retries:** Compiled graphs support `retry_policy` for automatic node retries on failure.
- **Distributed checkpointer:** PostgresSaver with `pgvector` for storing state as embeddings (enable semantic state retrieval).

## Performance Tuning

1. **Reduce serialisation overhead:** Use `orjson` for faster JSON serialisation of state (LangGraph uses `json.loads`/`dumps` internally).
2. **Disable checkpointing for throughput:** For read-only queries, compile graph without checkpointer. Only enable for stateful sessions.
3. **Batch checkpoint writes:** Accumulate state updates and write in batches to reduce Postgres round-trips.
4. **Streaming mode:** Use `app.astream(input, config, stream_mode="updates")` for real-time node output without blocking.

## Debugging Tips

1. `app.get_graph().print_ascii()` — prints the graph topology in ASCII.
2. LangSmith trace: every node invocation, edge traversal, and state mutation is recorded with timestamps.
3. `app.get_state(config)` — returns current state snapshot. `get_state_history(config)` — returns all checkpoints.
4. `webull = set_debug(True)` on the graph — prints state before and after each node.
5. Common errors: `NodeNotFound` (typo in node name), `EdgeAlreadyExists` (duplicate edge), `RecursionLimit` (loop condition not terminating).

## Production Deployment

1. **Postgres checkpointer:** Essential for production. Use PgBouncer for connection pooling. Migrate schema with Alembic.
2. **Docker + Kubernetes:** Stateless graph pods behind a service mesh. State persisted in Postgres StatefulSet.
3. **Graph versioning:** Different graph versions served from different pods. Blue-green deployment for zero-downtime updates.
4. **Rate limiting:** Per-thread and per-user rate limiting at the API gateway. Use token bucket algorithm.
5. **Graceful shutdown:** `SIGTERM` handler waits for current node to finish, then checkpoints before shutdown.
6. **Data migration:** When state schema changes, run a migration script that reads old checkpoints, transforms them, and writes new ones.
