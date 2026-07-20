# AI Agents

```mermaid
flowchart TD
    A["👤 User sends a request"] --> B["🧠 Agent receives input"]
    B --> C{"Agent has context?<br/>(memory)"}
    C -->|"No"| D["📚 Load from short-term / long-term memory"]
    C -->|"Yes"| E["🤔 LLM reasons about next action"]
    D --> E
    E --> F{"Does the LLM<br/>call a tool?"}
    F -->|"Yes 🛠️"| G["Execute tool<br/>(API, Calculator, Search, DB)"]
    G --> H["Store tool result in memory"]
    H --> I{"Goal achieved?"}
    I -->|"No"| E
    I -->|"Yes"| J["📤 Return final response to user"]
    F -->|"No ✋"| J

    style A fill:#e1f5fe,stroke:#01579b
    style B fill:#e1f5fe,stroke:#01579b
    style C fill:#fff3e0,stroke:#e65100
    style D fill:#f3e5f5,stroke:#4a148c
    style E fill:#e8f5e9,stroke:#1b5e20
    style F fill:#fff3e0,stroke:#e65100
    style G fill:#ffebee,stroke:#b71c1c
    style H fill:#f3e5f5,stroke:#4a148c
    style I fill:#fff3e0,stroke:#e65100
    style J fill:#e1f5fe,stroke:#01579b
```

> **What you're seeing:** An AI agent is a loop that keeps thinking, acting, and learning until it finishes your task. It's like a smart assistant who can use tools (calculator, search) and remember what happened before.

## 1. What is it?
An AI agent is an autonomous software entity that perceives its environment, makes decisions, and executes actions to achieve specific goals. Unlike traditional LLM wrappers that respond statically, agents maintain state, use tools, and operate in loops — observe, think, act, repeat. They are the fundamental building block of autonomous AI systems.

## 2. Why do we need it?
LLMs alone are reactive — they answer once and stop. Agents enable persistence, multi-step reasoning, tool orchestration, and goal-directed behavior. In production, you need agents to handle complex workflows, recover from errors, interact with APIs, and operate without human hand-holding. FAANG systems like Google's AutoML, Meta's recommendation engines, and Amazon's supply chain all use agentic architectures.

## 3. Real-world Example
**Amazon Supply Chain Agent**: An agent monitors inventory levels across 200+ warehouses. When stock for a SKU drops below threshold, it: (1) queries demand forecast API, (2) checks supplier lead times, (3) places reorder via procurement system, (4) schedules cross-dock logistics, (5) updates warehouse management system. All without human intervention.

## 4. Architecture Diagram (ASCII)
```
+------------------+       +------------------+
|   Environment    |       |   External APIs  |
| (DBs, APIs, Web) |<----->|   (Tools, Funcs) |
+------------------+       +------------------+
         ^                           ^
         |                           |
         v                           v
+--------------------------------------------------+
|                  AGENT LOOP                        |
|  +----------+  +-----------+  +---------------+   |
|  | Perception|->| Reasoning |->| Action        |   |
|  | (Observe) |  | (Plan)    |  | (Execute)     |   |
|  +----------+  +-----------+  +---------------+   |
|        ^                            |              |
|        |                            v              |
|  +----------+                 +--------------+     |
|  | Memory   |<----------------| Feedback     |     |
|  | (Store)  |                 | (Evaluate)   |     |
|  +----------+                 +--------------+     |
+--------------------------------------------------+
```

## 5. Internal Working
An agent operates in a perception-action loop. The perception module ingests observations (sensor data, API responses, user input). The reasoning module (typically an LLM) processes this against goals and memory to produce a plan. The action module executes steps via tool calls. Feedback from actions is stored in memory and fed back into perception. The loop continues until the goal is achieved or a terminal condition met.

## 6. Production Flow
```
User Request -> Input Validation -> Agent Initialization
    -> Context Assembly (memory + state)
    -> LLM Call (reasoning step)
    -> Tool Selection & Execution
    -> Result Evaluation
    -> Memory Update
    -> Repeat or Respond
```

## 7. HLD
```
+------------------+       +------------------+
|   API Gateway    |       |   Load Balancer  |
+------------------+       +------------------+
         |                          |
         v                          v
+--------------------------------------------------+
|              Agent Orchestrator Service            |
|  +----------------+  +-------------------------+  |
|  | Session Manager|  | Agent Runtime Pool      |  |
|  +----------------+  +-------------------------+  |
+--------------------------------------------------+
         |              |              |
         v              v              v
+----------+    +----------+    +----------+
| LLM Proxy|    | Tool Exec|    | Memory DB|
+----------+    +----------+    +----------+
```

## 8. LLD
```
AgentService {
    session_store: Redis (TTL-based sessions)
    agent_pool: ThreadPoolExecutor (max_workers=50)
    llm_client: OpenAI/Azure client with retry + backoff
    tool_registry: Dict[str, Tool] (dynamically loaded)
    memory_store: PostgreSQL (long-term) + Redis (short-term)
    
    async def run_agent(session_id, user_input):
        state = load_state(session_id)
        while not state.goal_achieved:
            action = reason(state, user_input)
            result = execute_tool(action)
            state = update_state(state, result)
            persist_state(session_id, state)
        return state.final_output
}
```

## 9. Python Implementation
```python
from pydantic import BaseModel, Field
from typing import Optional, Any
import asyncio
import uuid
from enum import Enum

class AgentState(Enum):
    IDLE = "idle"
    RUNNING = "running"
    WAITING = "waiting"
    DONE = "done"
    FAILED = "failed"

class ToolCall(BaseModel):
    tool_name: str
    args: dict[str, Any]
    result: Optional[Any] = None
    error: Optional[str] = None

class AgentMessage(BaseModel):
    role: str  # system, user, assistant, tool
    content: str
    tool_calls: list[ToolCall] = Field(default_factory=list)

class AgentSession(BaseModel):
    session_id: str = Field(default_factory=lambda: uuid.uuid4().hex)
    state: AgentState = AgentState.IDLE
    messages: list[AgentMessage] = Field(default_factory=list)
    metadata: dict[str, Any] = Field(default_factory=dict)
    max_steps: int = 25
    current_step: int = 0

class Tool(BaseModel):
    name: str
    description: str
    input_schema: dict[str, Any]
    fn: callable

    async def execute(self, **kwargs) -> Any:
        try:
            return await self.fn(**kwargs)
        except Exception as e:
            return {"error": str(e)}

class Agent:
    def __init__(self, llm_client, tool_registry: dict[str, Tool]):
        self.llm = llm_client
        self.tools = tool_registry

    async def run(self, session: AgentSession, user_input: str) -> AgentSession:
        session.state = AgentState.RUNNING
        session.messages.append(AgentMessage(role="user", content=user_input))

        while session.current_step < session.max_steps:
            response = await self.llm.chat(
                messages=[m.model_dump() for m in session.messages],
                tools=[t.model_dump() for t in self.tools.values()]
            )
            msg = AgentMessage(
                role="assistant",
                content=response.choices[0].message.content or "",
                tool_calls=self._parse_tool_calls(response)
            )
            session.messages.append(msg)

            if not msg.tool_calls:
                session.state = AgentState.DONE
                return session

            for tc in msg.tool_calls:
                tool = self.tools.get(tc.tool_name)
                if not tool:
                    tc.error = f"Unknown tool: {tc.tool_name}"
                else:
                    tc.result = await tool.execute(**tc.args)
                session.messages.append(
                    AgentMessage(role="tool", content=str(tc.result or tc.error))
                )
            session.current_step += 1

        session.state = AgentState.FAILED
        return session

    def _parse_tool_calls(self, response) -> list[ToolCall]:
        calls = []
        for tc in response.choices[0].message.tool_calls or []:
            calls.append(ToolCall(
                tool_name=tc.function.name,
                args=json.loads(tc.function.arguments)
            ))
        return calls
```

## 10. Folder Structure
```
agents/
├── core/
│   ├── agent.py          # Base agent class
│   ├── session.py        # Session management
│   ├── tool_registry.py  # Tool registration & discovery
│   └── types.py          # Pydantic models
├── tools/
│   ├── calculator.py
│   ├── web_search.py
│   ├── code_executor.py
│   └── api_client.py
├── memory/
│   ├── short_term.py     # Redis-based
│   ├── long_term.py      # PostgreSQL/pgvector
│   └── episodic.py       # Vector store
├── llm/
│   ├── client.py         # LLM client with retry
│   ├── prompts.py        # System prompts
│   └── schemas.py        # Structured output schemas
├── api/
│   ├── routes.py         # FastAPI endpoints
│   └── middleware.py     # Auth, rate limiting
└── main.py               # Entry point
```

## 11. Configuration
```yaml
agent:
  max_steps: 25
  max_concurrent_sessions: 1000
  session_ttl_seconds: 3600
  default_model: gpt-4o
  temperature: 0.7
  retry:
    max_retries: 3
    base_delay: 1.0
    max_delay: 30.0
  memory:
    short_term: redis://localhost:6379/0
    long_term: postgresql+asyncpg://user:pass@localhost:5432/agents
    vector_dimension: 1536
  tools:
    timeout_seconds: 30
    max_result_size: 1000000
```

## 12. Flowchart
```
   [Start]
      |
   [User Input]
      |
   [Initialize/Resume Session]
      |
   [LLM Reasoning]
      |
   [Tool Call?] --Yes--> [Execute Tool] --> [Store Result]
      |                           |
      No                          |
      |                           |
   [Check Goal] --No--> [Continue]--+
      |
      Yes
      |
   [Return Response]
```

## 13. Sequence Diagram
```
User         API Gateway       Agent Service       LLM          Tools        Memory
 |               |                  |               |             |            |
 |-- Request --->|                  |               |             |            |
 |               |-- Route -------->|               |             |            |
 |               |                  |--- Load ----->|             |            |
 |               |                  |    History    |             |            |
 |               |                  |<-- History ---|             |            |
 |               |                  |               |             |            |
 |               |                  |--- Prompt --->|             |            |
 |               |                  |<-- Response --|             |            |
 |               |                  |               |             |            |
 |               |                  |--- Call ----->|             |            |
 |               |                  |    Tool       |             |            |
 |               |                  |<-- Result ----|             |            |
 |               |                  |               |             |            |
 |               |                  |--- Store ---->|             |            |
 |               |                  |               |             |            |
 |               |<-- Response -----|               |             |            |
 |<-- Result ----|                  |               |             |            |
```

## 14. Pros
- Autonomous decision-making reduces human overhead
- Handles complex multi-step workflows
- Recovers from partial failures
- Scales horizontally (stateless sessions)
- Plugs into any external API/system

## 15. Cons
- LLM latency compounds across steps
- Token costs are unpredictable
- Debugging agent behavior is difficult
- Can get stuck in loops without guardrails
- Tool execution introduces security surface area

## 16. Alternatives
- **Rules-based automation** — deterministic, no LLM, works for simple flows
- **RPA (UiPath, Automation Anywhere)** — UI-scraping, no reasoning
- **Workflow engines (Temporal, Airflow)** — DAG-based, fixed logic
- **Chatbots (no agent loop)** — single-turn Q&A, no tool use

## 17. Performance Considerations
- LLM call dominates latency (2–10s per step). Use streaming for user-facing apps
- Batch tool calls that are independent
- Cache LLM responses for identical inputs
- Use session affinity to keep hot sessions on same node
- Pre-warm LLM connections with connection pooling

## 18. Scaling to Millions
```
Horizontal Scaling:
  - Stateless sessions stored in Redis/PG (not in-memory)
  - Agent pool scaled via K8s HPA (CPU + custom LLM latency metric)
  - Tool executors in separate service with autoscaling
  - LLM client with circuit breaker per model deployment

Database Sharding:
  - Session store: hash on session_id (256 shards)
  - Long-term memory: timeseries partitioning by week
  - Vector store: IVF_PQ indexing for 10M+ vectors

Caching:
  - L1: in-memory LRU for frequent tool results (100ms)
  - L2: Redis for session state (1ms)
  - L3: CDN for static tool outputs (static reports)
```

## 19. Failure Scenarios
| Scenario | Impact | Mitigation |
|----------|--------|------------|
| LLM timeout | Step stalls | Retry with backoff, fallback LLM |
| Tool 5xx | Partial failure | Idempotency keys, retry |
| Token limit | Context truncated | Sliding window, summarization |
| Infinite loop | Resource exhaustion | Max step limit, anomaly detection |
| Memory corruption | Wrong decisions | Session snapshot, rollback |

## 20. Security
- Tool execution runs in sandboxed containers (gVisor/Firecracker)
- All tool inputs sanitized — no raw SQL/command injection
- API keys stored in Vault, never in agent context
- Rate limiting per session/user
- Audit log every action with trace ID
- PII redaction before LLM calls

## 21. Monitoring
```prometheus
agent_steps_total{status="success|error|timeout"}
agent_llm_latency_seconds{p50,p95,p99}
agent_tool_latency_seconds{name="web_search|calc|api"}
agent_sessions_active
agent_sessions_completed_total
agent_token_usage_per_step
agent_error_rate_by_code
agent_loop_iterations_distribution
```

## 22. Interview Questions
**Q1**: How does an agent decide when to stop? (Saturation detection, explicit "done" signal, max step enforcement)

**Q2**: How do you prevent infinite loops? (Max step limit, similarity check on consecutive actions, diversity penalty)

**Q3**: How do you handle rate-limited tools? (Token bucket per tool, priority queue, exponential backoff)

## 23. Cheat Sheet
```
Agent Loop  = Perceive -> Reason -> Act -> Evaluate -> Memory
Action Types = ToolCall, Response, Wait, Terminate
Memory Types = Short-term (session), Long-term (persistent), Episodic (experiences)
Evaluation   = Success? Partial? Failed? Needs human?
State Mgmt   = Session ID -> Redis -> Restore on failure
```

## 24. Common Mistakes
- No step limit → infinite loops burn tokens and $$$
- No timeout per tool → one slow tool stalls the entire agent
- Shared memory across sessions → data leakage
- Over-prompting the system message → context waste, worse performance
- Not validating tool outputs → LLM hallucinates based on bad data

## 25. Production Best Practices
1. Always set `max_steps` per agent type
2. Use `asyncio.timeout()` per tool execution
3. Persist session after every step (crash recovery)
4. Monitor token usage per session — alert on anomalies
5. A/B test agent prompts like any model change
6. Log every tool call with input/output for debugging
7. Use structured outputs (JSON mode) for parsing stability
8. Implement a human-in-the-loop circuit breaker for high-cost actions
