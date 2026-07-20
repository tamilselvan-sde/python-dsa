# Multi-Agent System Design

## 1. Requirements

### Functional Requirements
- Dynamic agent orchestration — Supervisor delegates to specialized agents
- Tool-use capability — agents call external APIs, databases, and services
- Inter-agent communication — agents share intermediate results and context
- Human-in-the-loop approval for critical actions
- Persistence — conversation history and agent state survive restarts
- Observability — full traceability of agent decisions and tool calls
- Pluggable agent registry — add/remove agents without redeployment
- Parallel execution — independent agents run concurrently
- Memory management — short-term (session) and long-term (vector) memory
- Guardrails — prevent harmful or policy-violating agent actions

### Non-Functional Requirements
- **Latency**: Supervisor decision < 100ms, sub-agent task < 5s
- **Availability**: 99.9%
- **Concurrency**: 10K active multi-agent sessions
- **Trace completeness**: 100% of agent actions logged
- **Isolation**: Agent failures must not cascade
- **Determinism**: Retry of same input yields same result (with same context)
- **Cost control**: Token budget per task, per session, per tenant

## 2. Capacity Estimation

| Metric | Value |
|--------|-------|
| Concurrent sessions | 10,000 |
| Avg agents per task | 3-5 |
| Avg tool calls per task | 8-15 |
| Avg tokens per task | 4,000 (input) + 1,500 (output) |
| Total daily tokens | 10K sessions × 100 tasks × 5.5K tokens × 5 agents = ~27.5B tokens/day |
| State storage | 50 KB/session × 10K concurrent = 500 MB (in-memory) |
| Task queue | 100K tasks/min peak |
| Tool call latency P99 | 2s (external API) |

## 3. High-Level Architecture

```
                         ┌──────────────────────────────┐
                         │      API Gateway / Load Balancer   │
                         │   Auth, Rate Limit, Session Init   │
                         └──────────────┬───────────────────┘
                                        │
                                        ▼
┌──────────────────────────────────────────────────────────────────┐
│                     Orchestration Layer                           │
│                                                                  │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │             Supervisor Agent (Router)                    │    │
│  │   - Intent classification from user input                │    │
│  │   - Task decomposition → DAG of sub-tasks                │    │
│  │   - Agent selection from registry                        │    │
│  │   - Result aggregation and final response                │    │
│  └────────────┬────────────────────┬────────────────────────┘    │
│               │                    │                              │
│        ┌──────▼──────┐     ┌───────▼───────┐                    │
│        │ Planner      │     │  Executor     │                    │
│        │ (Task DAG)   │     │ (DAG Runner)  │                    │
│        └──────────────┘     └───────────────┘                    │
└──────────────────────────────────────────────────────────────────┘
               │                    │
               ▼                    ▼
┌──────────────────────────────────────────────────────────────────┐
│                      Agent Runtime Layer                          │
│                                                                  │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────────┐    │
│  │ Research │  │ Code     │  │ Data     │  │ Customer     │    │
│  │ Agent    │  │ Agent    │  │ Analyst  │  │ Support Agent│    │
│  │ (Web+KB) │  │ (Sandbox)│  │ (SQL+DB) │  │ (CRM+Ticket) │    │
│  └──────────┘  └──────────┘  └──────────┘  └──────────────┘    │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐                       │
│  │ Creative │  │ Critic   │  │ Observer │                       │
│  │ Agent    │  │ Agent    │  │ (Monitor)│                       │
│  └──────────┘  └──────────┘  └──────────┘                       │
└──────────────────────────────────────────────────────────────────┘
       │              │              │              │
       ▼              ▼              ▼              ▼
┌──────────────────────────────────────────────────────────────────┐
│                      Infrastructure Layer                         │
│                                                                  │
│  ┌─────────┐  ┌──────────┐  ┌──────────┐  ┌───────────────┐    │
│  │ Vector  │  │ Message  │  │ Object   │  │ LLM Provider   │    │
│  │ Store   │  │ Queue    │  │ Store    │  │ (GPT/Claude/   │    │
│  │(Memory) │  │(Kafka/Rabbit)│ (S3)    │  │  Llama)        │    │
│  └─────────┘  └──────────┘  └──────────┘  └───────────────┘    │
│  ┌─────────┐  ┌──────────┐  ┌──────────┐  ┌───────────────┐    │
│  │ Redis   │  │ Postgres │  │ Prometheus│  │ Tool Registry │    │
│  │(State)  │  │(Persistence) │(Metrics) │  │ (OpenAPI/Swagger)│
│  └─────────┘  └──────────┘  └──────────┘  └───────────────┘    │
└──────────────────────────────────────────────────────────────────┘
```

## 4. Low-Level Design

### Agent Abstraction

```python
class Agent(ABC):
    agent_id: str
    name: str
    description: str  # used by supervisor for selection
    system_prompt: str
    tools: list[Tool]
    max_iterations: int = 10
    llm_config: LLMConfig

    @abstractmethod
    async def run(self, task: Task, context: Context) -> Result:
        """Execute the task. Can call tools, spawn sub-agents, return result."""

class Tool:
    name: str
    description: str  # used by LLM for function calling
    parameters: JSONSchema
    requires_approval: bool = False
    max_retries: int = 3
    timeout: int = 30

    async def execute(self, params: dict) -> ToolResult:
        ...

class Task:
    id: UUID
    parent_task_id: UUID | None
    session_id: UUID
    agent_id: str
    input: str
    state: TaskState  # pending | running | completed | failed
    dependencies: list[UUID]  # for DAG ordering
    created_at: datetime
    completed_at: datetime | None

class Context:
    session_id: UUID
    user_id: UUID
    conversation_history: list[Message]
    short_term_memory: dict  # key-value within session
    long_term_memory: VectorMemory
    shared_state: dict  # shared across agents in same session

class Message:
    role: Literal["user", "assistant", "system", "tool", "agent"]
    content: str
    tool_calls: list[ToolCall] | None
    tool_results: list[ToolResult] | None
    agent_id: str | None
    metadata: dict
```

### Supervisor Agent Logic

```
User Input → Supervisor:
  1. Classify intent → determine primary agent(s) needed
  2. Task decomposition:
     - "Analyze Q3 revenue and draft a report"
     → DataAgent(fetch revenue DB) → CodeAgent(compute metrics)
     → CreativeAgent(draft report) → CriticAgent(review)
  3. Build DAG with dependencies
  4. Submit tasks to Executor
  5. Aggregate results, produce final response
```

### Executor — DAG Runner

```python
class DAGExecutor:
    async def execute(tasks: list[Task], dag: dict) -> dict[UUID, Result]:
        completed = {}
        ready_queue = deque(t for t in tasks if not t.dependencies)

        while ready_queue:
            batch = [ready_queue.popleft() for _ in range(parallelism)]
            results = await asyncio.gather(*[
                self._run_task(t, completed) for t in batch
            ])
            for t, r in zip(batch, results):
                completed[t.id] = r

            # Unblock dependent tasks
            for t in tasks:
                if t.id not in completed and all(d in completed for d in t.dependencies):
                    ready_queue.append(t)

        return completed
```

## 5. Database Schema

### PostgreSQL — Persistent State

```sql
CREATE TABLE sessions (
    id UUID PRIMARY KEY,
    user_id UUID NOT NULL,
    tenant_id UUID NOT NULL,
    status TEXT DEFAULT 'active',
    metadata JSONB DEFAULT '{}',
    token_budget INT DEFAULT 100000,
    tokens_used INT DEFAULT 0,
    created_at TIMESTAMPTZ DEFAULT NOW(),
    updated_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE TABLE tasks (
    id UUID PRIMARY KEY,
    session_id UUID REFERENCES sessions(id),
    parent_task_id UUID REFERENCES tasks(id),
    agent_id TEXT NOT NULL,
    input TEXT NOT NULL,
    output TEXT,
    state TEXT DEFAULT 'pending',
    dependencies UUID[] DEFAULT '{}',
    tools_called TEXT[] DEFAULT '{}',
    tokens_used INT DEFAULT 0,
    llm_latency_ms INT,
    error TEXT,
    created_at TIMESTAMPTZ DEFAULT NOW(),
    completed_at TIMESTAMPTZ
);

CREATE TABLE tool_calls (
    id UUID PRIMARY KEY,
    task_id UUID REFERENCES tasks(id),
    tool_name TEXT NOT NULL,
    input JSONB NOT NULL,
    output JSONB,
    status TEXT DEFAULT 'pending',
    latency_ms INT,
    requires_approval BOOLEAN DEFAULT FALSE,
    approved BOOLEAN,
    created_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE TABLE agent_registry (
    agent_id TEXT PRIMARY KEY,
    name TEXT NOT NULL,
    version INT DEFAULT 1,
    description TEXT,
    system_prompt TEXT NOT NULL,
    tools TEXT[] DEFAULT '{}',
    max_iterations INT DEFAULT 10,
    llm_config JSONB,
    status TEXT DEFAULT 'active',
    created_at TIMESTAMPTZ DEFAULT NOW()
);
```

### Redis — Short-Term Memory & State

```
session:{session_id}:state → Hash
  current_agent: "supervisor"
  plan: JSON (current DAG)
  shared_context: JSON
  iteration_count: 5
  ttl: 3600 (1 hour)

session:{session_id}:lock → String (Redlock)
  Used for critical section: human approval, state mutation
```

### Vector Store — Long-Term Memory

```python
# Memory collection in Chroma/Pinecone
collection: "agentic_memory_{tenant_id}"

documents:
  - session_id
  - agent_id
  - task_description
  - result_summary
  - key_insights
  - embedding: text-embedding-3-small (256 dimensions)
  - timestamp
```

## 6. API Contract

### POST /api/v1/agents/execute

```json
// Request
{
  "session_id": "s-123",
  "user_id": "u-456",
  "input": "Analyze our Q3 revenue data and create a summary report",
  "agents": ["research", "data_analyst", "creative", "critic"],
  "options": {
    "max_iterations": 15,
    "parallelism": 2,
    "require_human_approval_for": ["send_email", "modify_db"],
    "token_budget": 50000
  },
  "context": {
    "project_id": "proj-789",
    "priority": "high"
  }
}

// Response (streaming via SSE)
event: task_started
data: {"task_id": "t-1", "agent": "data_analyst", "input": "SELECT * FROM revenue WHERE quarter='Q3'"}

event: tool_call
data: {"task_id": "t-1", "tool": "postgres_query", "status": "running", "latency_ms": 120}

event: human_approval
data: {"task_id": "t-3", "tool": "send_email", "message": "Approve sending report to VP of Sales?"}

event: task_completed
data: {"task_id": "t-1", "agent": "data_analyst", "result_summary": "Q3 revenue: $12.4M, growth 8% YoY"}

event: completed
data: {
  "session_id": "s-123",
  "final_output": "Q3 Revenue Analysis Report...",
  "tasks_completed": 5,
  "tasks_failed": 0,
  "total_tokens": 12500,
  "total_latency_ms": 8400,
  "trace_url": "https://observability.company.com/trace/s-123"
}
```

### POST /api/v1/agents/approve

```json
{
  "session_id": "s-123",
  "tool_call_id": "tc-456",
  "approved": true,
  "reason": "Report is ready for VP review"
}
```

## 7. Sequence Diagram

```
User     Gateway   Supervisor  Planner   Executor   Agent_A   Agent_B   Tool_API   LLM
 │          │          │          │         │          │        │          │         │
 ├─Request──►          │          │         │          │        │          │         │
 │          ├─Route────►          │         │          │        │          │         │
 │          │          ├─Classify─►         │          │        │          │         │
 │          │          │          ├─Plan────►         │        │          │         │
 │          │          │          │         │          │        │          │         │
 │          │          │          │         ├─dispatch─►│        │          │         │
 │          │          │          │         │          ├─prompt─►         │         │
 │          │          │          │         │          │        │         ├─sql────►│
 │stream────┤          │          │         │          │        │         │◄─result─┤
 │          │          │          │         │          │        │         │         │
 │          │          │          │         │          │        ├─dispatch─►         │
 │          │          │          │         │          │        │         ├─search─►│
 │          │          │          │         │          │        │         │◄─result─┤
 │          │          │          │         │          │        │         │         │
 │          │          │          │         │          │        │         │         │
 │◄─Result──┤          │          │         │          │        │         │         │
```

## 8. Scaling Strategy

| Component | Strategy |
|-----------|----------|
| Supervisor | Stateless, horizontally scalable; distribute via consistent hashing on session_id |
| Executor | Work-stealing queue per node; backpressure via semaphore |
| Agents | Pool of pre-warmed agent containers; cold-start < 100ms via snapshot |
| Tool executions | Rate-limited per tool; circuit breaker on 5xx |
| LLM calls | Request batching (if same model), priority queue (user-facing > background) |
| State store | Redis Cluster with read replicas; session affinity for consistent reads |
| Message streaming | Kafka per region; consumer groups for trace log aggregation |

## 9. Failure Handling

| Failure | Mitigation |
|---------|-----------|
| Agent crash (OOM) | Supervisor detects heartbeat failure; retry task on new agent instance |
| LLM timeout | Retry with exponential backoff (max 3); fallback to simpler prompt |
| Tool API unavailable | Circuit breaker; return error to agent which can adapt its plan |
| Deadlock (circular deps) | DAG cycle detection before execution; max depth = 10 |
| Token budget exceeded | Preemptively checkpoint state; return partial results |
| Supervisor failure | Stateless; new supervisor reconstructs from persisted task DAG |
| Human approval timeout | Default deny after 5 min; log and notify admin |
| Cascading agent failure | Bulkhead pattern — each agent type in its own thread pool/container |

## 10. Monitoring

| Metric | Alert | Action |
|--------|-------|--------|
| Supervisor decision latency > 500ms | PagerDuty | Scale supervisor pods |
| Task failure rate > 5% | Slack | Investigate agent/tool health |
| Agent loop count > max_iterations | Log + alert | Adjust system prompt constraints |
| Token usage per session > 80% budget | Log | Warn user or throttle |
| Human approval queue > 10 items | Slack | Notify on-call admin |
| Tool error rate > 10% | PagerDuty | Circuit breaker + OpsGenie |
| Memory growth per session > 100 KB | Log | Check for memory leak |

## 11. Security

- **Tool approval gates**: Destructive tools (DB write, email, deploy) require human approval
- **Agent isolation**: Each agent type runs in container with least-privilege IAM role
- **Context isolation**: Shared context is immutable by child agents; only supervisor writes
- **PII scanning**: All agent outputs pass through PII detection before returning to user
- **Prompt injection**: Hardened system prompts; input/output guardrails on supervisor
- **Token budget enforcement**: Hard cap per session; agent cannot exceed without approval
- **Audit trail**: Every agent decision, tool call, and LLM prompt-response logged (immutable)

## 12. Trade-offs

| Decision | Pros | Cons |
|----------|------|------|
| Supervisor → Sub-agent (hierarchical) | Clear delegation, explainable | Single point of coordination; supervisor can be bottleneck |
| Flat (peer agents) vs Hierarchical | Peer = no bottleneck | Peer = harder to orchestrate complex workflows |
| Pre-defined DAG vs Dynamic planning | Dynamic adapts to input | Dynamic adds latency and token cost |
| In-process tool execution | Low latency, simple | Memory isolation weaker |
| Deterministic retry | Reproducible debugging | May repeat expensive tool calls |
| Streaming vs Batch response | Better UX (partial results) | More connection overhead |

## 13. Interview Tips

- **Start with the supervisor pattern**: It's the most common in production (Microsoft AutoGen, LangGraph, CrewAI). Explain why a single supervisor is simpler than fully decentralized agents
- **DAG orchestration is the key insight**: Draw the task dependency graph. Show you understand topological ordering, parallel execution, and cycle detection
- **Human-in-the-loop**: Mention this explicitly — it's what separates toy demos from production systems. Show the approval API design
- **Failures are expected at scale**: Discuss partial failures — if Code Agent fails but Research Agent succeeded, should the system retry, adapt, or fail?
- **Memory types**: Short-term (Redis), long-term (Vector DB), shared (PostgreSQL). Show you understand the distinction and when each is appropriate
- **Cost management**: Multi-agent systems are expensive. Discuss token budgets, caching intermediate results, and model tiering (cheap model for planning, expensive for generation)
- **Common variants**: "Design a coding agent system", "Design an AI research assistant with multi-agent debate", "Design a content moderation multi-agent pipeline"
