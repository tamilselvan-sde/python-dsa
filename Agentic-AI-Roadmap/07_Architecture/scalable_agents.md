# Scalable Agent Systems

> Author: **Tamilselvan** · ✉️ tamilselvan.sde@gmail.com · 🔗 [LinkedIn](https://www.linkedin.com/in/tamilselvan-ai/)
>

## 1. What is it?
Scalable agent systems are architectures for deploying AI agents that can handle millions of concurrent conversations, coordinate across multiple tools and data sources, maintain state reliably, and recover from failures — all while keeping latency under user expectations.

## 2. Why do we need it?
Single-agent chat works for demos. Production agent systems must handle: concurrent users, long-running workflows (minutes to hours), tool execution failures, state persistence across crashes, rate limits, and multi-agent coordination. Without scalable architecture, agents stall, lose context, or crash under load.

## 3. Real-world Example
**Intercom's Fin** handles 500M+ conversations with stateless agent workers behind a queue, conversation state in Redis + PostgreSQL, tool executions as idempotent async jobs, and a supervisor agent that escalates to humans when confidence drops below 0.8. Handles 10K concurrent conversations with p50 < 2s.

## 4. Architecture Diagram (ASCII)

```
                      +---------------------+
                      |   API Gateway        |
                      +----------+----------+
                                 |
                      +----------v----------+
                      |   Conversation       |
                      |   Manager            |
                      +----------+----------+
                                 |
              +------------------+------------------+
              |                                     |
      +-------v-------+                     +-------v-------+
      |  Agent Worker  |                    |  Agent Worker  |
      |  Pool (K8s)    |                    |  Pool (K8s)    |
      +-------+-------+                     +-------+-------+
              |                                     |
      +-------v-------+                     +-------v-------+
      |  Tool Executor |                    |  Tool Executor |
      |  (Sandboxed)   |                    |  (Sandboxed)   |
      +-------+-------+                     +-------+-------+
              |                                     |
      +-------v--------+               +------------+------+
      |  State Store    |               |  Message Queue    |
      |  (Redis + PG)   |               |  (SQS/RabbitMQ)   |
      +-----------------+               +-------------------+
```

## 5. Internal Working
Agent workers are stateless — they receive a task + context, execute the agent loop (observe -> think -> act), and write results. State is stored externally in Redis (hot) and PostgreSQL (durable). Workers scale horizontally via queue depth. Long-running agents checkpoint after each step, enabling crash recovery.

## 6. Production Flow
```
User sends message -> API Gateway -> Conversation Manager -> Queue
-> Agent Worker picks up -> Load state -> LLM call -> Tool execution
-> LLM call -> Tool execution -> ... -> Final response -> User
```

## 7. HLD

### Components
1. **Conversation Manager**: Session routing, state lookup, idle timeout (30min)
2. **Agent Workers**: Stateless, auto-scaled, each handling 5-10 concurrent conversations
3. **Tool Executors**: Sandboxed (gVisor/Firecracker), rate-limited, idempotent
4. **State Store**: Redis (hot state, locks, sessions), PostgreSQL (history, checkpoints)
5. **Message Queue**: SQS/RabbitMQ — buffers spikes, enables retry, decouples components
6. **LLM Gateway**: Rate limiting, key rotation, fallback models, cost tracking

## 8. LLD

```python
class AgentWorker:
    def __init__(self):
        self.llm = LLMClient()
        self.state_store = StateStore()
        self.tool_registry = ToolRegistry()
        self.max_steps = 20
        self.timeout = 300

    async def run(self, task: AgentTask) -> AgentResult:
        state = await self.state_store.load(task.conversation_id)
        for step in range(state.current_step, self.max_steps):
            if time.monotonic() - state.start_time > self.timeout:
                raise TimeoutError("Agent exceeded max duration")
            response = await self.llm.generate(self._build_prompt(state))
            action = self._parse_action(response)
            if action.type == "final":
                break
            result = await self.tool_registry.execute(action.tool, action.args)
            state.add_step(action, result)
            await self.state_store.checkpoint(task.conversation_id, state)
        await self.state_store.finalize(task.conversation_id, state)
        return AgentResult(state.final_response, state.steps)
```

## 9. Python Implementation

```python
import asyncio
from dataclasses import dataclass, field
from enum import Enum
import uuid

class StepType(Enum):
    THINK = "think"
    ACT = "act"
    OBSERVE = "observe"
    FINAL = "final"

@dataclass
class ConversationState:
    conversation_id: str
    messages: list = field(default_factory=list)
    steps: list = field(default_factory=list)
    current_step: int = 0
    start_time: float = 0.0
    final_response: str | None = None

class ScalableAgentSystem:
    def __init__(self):
        self.workers = 10
        self.queue = asyncio.Queue()
        self.state_store = StateStore()

    async def start_workers(self):
        workers = [self._worker_loop(i) for i in range(self.workers)]
        await asyncio.gather(*workers)

    async def _worker_loop(self, worker_id: int):
        while True:
            task = await self.queue.get()
            try:
                agent = AgentWorker()
                result = await agent.run(task)
                await self._notify_complete(task.conversation_id, result)
            except Exception as e:
                await self._notify_error(task.conversation_id, e)
            finally:
                self.queue.task_done()

    async def submit(self, user_id: str, message: str) -> str:
        conversation_id = str(uuid.uuid4())
        task = AgentTask(conversation_id, user_id, message)
        await self.queue.put(task)
        return conversation_id
```

## 10. Folder Structure

```
agent_platform/
  core/
    agent.py, state.py, tools.py, llm.py
  workers/
    supervisor_agent.py, research_agent.py, coding_agent.py
  orchestration/
    conversation_manager.py, task_scheduler.py
  storage/
    state_store.py, checkpointer.py, history.py
  sandbox/
    tool_executor.py, sandbox_manager.py
  monitoring/
    metrics.py, tracing.py
  deploy/
    k8s/
```

## 11. Configuration

```yaml
agent_system:
  workers:
    min_replicas: 10
    max_replicas: 100
    scaling_metric: queue_depth
    target_queue_depth: 50
  state_store:
    redis:
      url: redis://redis-cluster:6379
      ttl_seconds: 3600
    postgres:
      url: postgres://...
  agent:
    max_steps: 20
    timeout_seconds: 300
  llm:
    primary:
      model: gpt-4o
      max_concurrent: 50
    fallback:
      model: claude-3-haiku
  tools:
    rate_limit_per_user: 100/minute
    sandbox: gvisor
```

## 12. Flowchart

```
User Message -> Conversation Manager -> Queue -> Agent Worker
  -> Load State -> LLM Think -> Parse Action
  -> [if final] Return Response
  -> [if tool] Execute Tool -> Observe -> Checkpoint -> Loop
  -> [if error/timeout] Fallback Response
```

## 13. Sequence Diagram

```
User    ConvMgr    Queue    Worker    LLM    Tool    StateStore
 |        |          |        |        |      |        |
 |--msg-->|          |        |        |      |        |
 |        |--enq---->|        |        |      |        |
 |        |          |--deq-->|        |      |        |
 |        |          |        |--load---------->|        |
 |        |          |        |<--state--------|        |
 |        |          |        |--think->|      |        |
 |        |          |        |<--act---|      |        |
 |        |          |        |--exec--------->|        |
 |        |          |        |<--result-------|        |
 |        |          |        |--checkpoint----------->|
 |        |          |        | ...repeat...    |        |
 |        |          |        |--finalize------------->|
 |<--resp-|          |        |        |      |        |
```

## 14. Pros
- Horizontal scaling via stateless workers; Fault tolerance with checkpoint/replay; Queue-based load leveling; State persistence survives crashes; Tool sandboxing for security; Easy to add new agent types.

## 15. Cons
- Queue adds 10-50ms latency; State serialization overhead for long conversations; Tool execution is sequential per agent; Distributed tracing required for debugging; LLM rate limiting impacts all agents.

## 16. Alternatives
- **Orchestrator-based (LangGraph)**: Centralized state management; **Event-driven (CrewAI)**: Agents emit events, decoupled execution; **ReAct loop in single process**: Simpler but no isolation; **Serverless (AWS Step Functions)**: Managed state machine.

## 17. Performance Considerations
- Main bottleneck is LLM generation (500ms-5s); Tool execution should be async with timeout; State serialization adds 5-20ms per checkpoint; Queue depth determines worker count scaling; Keep checkpoint frequency configurable.

## 18. Scaling to Millions
- **10K concurrent**: 20 agent workers, single Redis + PG; **100K**: 200 workers, Redis Cluster, read replicas for PG; **1M**: 2000+ workers, partitioned queues, sharded state store; Use HPA on K8s based on queue depth; Pre-warm workers in pool (avoid cold start).

## 19. Failure Scenarios
- **Worker crash**: Another worker picks up from last checkpoint; **State store failure**: Read-only mode, return cached responses; **Tool timeout**: Agent retries, marks tool degraded after 3 failures; **LLM rate limit**: Queue and use fallback model; **Queue backlog**: Auto-scale workers, shed load if backlog > 100K.

## 20. Security
- Tool execution in sandbox (gVisor/Firecracker); LLM output filtering (PII, toxicity); Conversation isolation via tenant IDs; Rate limiting per user; Input validation for tool args; Secrets encryption at rest; Audit logging of all agent actions.

## 21. Monitoring
- **Queue depth**: Alert if > 10K; **Agent latency**: p50/p99 per step; **Tool success rate**: Track per tool; **Checkpoint size**: Alert if > 1MB; **Cost**: Per-conversation LLM cost; **Escalation rate**: Agent -> human handoff ratio; **Dashboard**: Grafana with per-component panels.

## 22. Interview Questions
1. *Design an agent system handling 1M daily conversations.* — Stateless workers, queue-based, state in Redis+PG, checkpointing.
2. *How do you handle tool failures in agent systems?* — Retry with backoff, fallback strategies, mark tool degraded.
3. *What happens when an agent worker crashes mid-execution?* — Last checkpoint is loaded by next available worker.
4. *How do you prevent infinite agent loops?* — Max step counter, timeout, token limit, loop detection.

## 23. Cheat Sheet

```
Agent Scaling = Stateless Workers + Queue + External State + Checkpointing
Key decisions: max_steps, timeout, checkpoint frequency, queue type
State: Redis for hot, PG for durable, TTL cleanup
Tools: sandbox, idempotent, timeout, rate-limited
```

## 24. Common Mistakes
- State stored in worker memory (lost on crash); No max step limit (infinite loops); Synchronous tool calls block the agent loop; Overly large checkpoint payloads (slow persistence); No rate limiting on tool execution; Queue without dead-letter handling; Workers not idempotent (duplicate execution).

## 25. Production Best Practices
- Implement checkpointing from day one; Set conservative timeouts (max_steps=20, 5min); Make tool execution idempotent; Use dead-letter queue for failed tasks; Monitor LLM cost per conversation; Auto-scale workers on queue depth (not CPU); Chaos test worker failures regularly; Pin LLM model versions; Log every agent step with trace ID.
