# Multi-Agent Systems

> Author: **Tamilselvan** · ✉️ tamilselvan.sde@gmail.com · 🔗 [LinkedIn](https://www.linkedin.com/in/tamilselvan-ai/)
>

## 1. What is it?
Multi-agent systems (MAS) consist of multiple AI agents that collaborate, coordinate, or compete to solve tasks. Agents have specialized roles (researcher, coder, reviewer, orchestrator), communicate through structured message passing, and collectively achieve goals no single agent could. Patterns include orchestrator-worker, swarm, debate, and pipeline.

## 2. Why do we need it?
Single agents have limitations: context window constraints, single-perspective bias, limited parallelism. Multi-agent systems (1) decompose complex tasks into specialized roles, (2) provide diverse perspectives (reducing hallucination via debate), (3) parallelize independent work, (4) enable error-checking through peer review.

## 3. Real-world Example
**Google's Pathways Architecture**: Multiple specialized agents handle different aspects of a single query. One agent retrieves documents, another summarizes, another cross-references facts, another formats output. The orchestrator routes work, merges results, and handles conflicts. This architecture powers Google's most advanced AI systems.

## 4. Architecture Diagram (ASCII)
```
+--------------------------------------------------+
|                  ORCHESTRATOR                      |
|   (Task decomposition, routing, merge, conflict)  |
+--------------------------------------------------+
    |         |           |          |         |
    v         v           v          v         v
+------+ +--------+ +--------+ +--------+ +--------+
| Agent1 | | Agent2 | | Agent3 | | Agent4 | | Agent5 |
|Research| |  Code  | |Review  | |  Test  | |  Docs  |
+------+ +--------+ +--------+ +--------+ +--------+
    |         |           |          |         |
    +---------+-----------+----------+---------+
                        |
+--------------------------------------------------+
|              Message Bus (Redis/Kafka)             |
+--------------------------------------------------+
```

## 5. Internal Working
Agents communicate via structured messages (not free text). Each message has: sender, recipient(s), message type (request, response, critique, vote), payload, and metadata. The orchestrator maintains a shared state (task graph, dependencies, results). Agents pull tasks from queues, process, and push results back. Conflict resolution happens through voting, ranking, or orchestrator override.

## 6. Production Flow
```
1. Orchestrator receives goal
2. Decomposes goal into sub-tasks (task DAG)
3. Assigns sub-tasks to specialized agents
4. Agents execute in parallel where dependencies allow
5. Results aggregated by orchestrator
6. Conflict resolution if agents disagree
7. Orchestrator produces final output
```

## 7. HLD
```
+------------------+
|  API Gateway     |
+------------------+
         |
         v
+---------------------------+
|   Orchestrator Service     |
|  - Task Planner            |
|  - Dependency Resolver     |
|  - Result Merger           |
+---------------------------+
    |     |     |     |
+-----------+ +-----------+
| Agent Pool| | Shared    |
| (K8s Deploy)| | State     |
+-----------+ | (Redis)   |
    |     |   +-----------+
    v     v
+--------+ +--------+
|Agent A | |Agent B |
|Replicas| |Replicas|
+--------+ +--------+
```

## 8. LLD
```python
from pydantic import BaseModel, Field
from enum import Enum
from typing import Any
import uuid
from datetime import datetime

class TaskStatus(str, Enum):
    PENDING = "pending"
    ASSIGNED = "assigned"
    RUNNING = "running"
    COMPLETED = "completed"
    FAILED = "failed"
    NEEDS_REVIEW = "needs_review"

class AgentRole(str, Enum):
    ORCHESTRATOR = "orchestrator"
    RESEARCHER = "researcher"
    CODER = "coder"
    REVIEWER = "reviewer"
    TESTER = "tester"
    DOCS = "documentation"

class AgentMessage(BaseModel):
    message_id: str = Field(default_factory=lambda: uuid.uuid4().hex)
    sender: str
    recipient: str | list[str] | None = None  # None = broadcast
    msg_type: str  # "task", "result", "critique", "vote", "status"
    payload: dict[str, Any]
    timestamp: datetime = Field(default_factory=datetime.utcnow)
    correlation_id: str  # Links messages in a conversation

class Task(BaseModel):
    task_id: str = Field(default_factory=lambda: uuid.uuid4().hex)
    description: str
    agent_role: AgentRole
    depends_on: list[str] = Field(default_factory=list)
    input_data: dict[str, Any] = Field(default_factory=dict)
    output_data: dict[str, Any] | None = None
    status: TaskStatus = TaskStatus.PENDING
    priority: int = 0
    max_retries: int = 2
    retry_count: int = 0

class AgentRegistration(BaseModel):
    agent_id: str
    role: AgentRole
    capabilities: list[str]
    status: str = "idle"
    last_heartbeat: datetime = Field(default_factory=datetime.utcnow)

class MultiAgentOrchestrator:
    def __init__(self, message_bus, agent_pool, llm):
        self.bus = message_bus
        self.agent_pool = agent_pool
        self.llm = llm
        self.tasks: dict[str, Task] = {}
        self.agents: dict[str, AgentRegistration] = {}

    async def handle_goal(self, goal: str) -> dict:
        task_graph = await self._decompose_goal(goal)
        
        # Execute tasks respecting dependency DAG
        completed = set()
        while len(completed) < len(task_graph):
            ready = [t for t in task_graph if t.task_id not in completed 
                     and all(d in completed for d in t.depends_on)
                     and t.status == TaskStatus.PENDING]
            
            results = await asyncio.gather(*[
                self._execute_task(t) for t in ready
            ])
            
            for t, result in zip(ready, results):
                t.output_data = result
                t.status = TaskStatus.COMPLETED
                completed.add(t.task_id)
        
        merged = await self._merge_results(task_graph)
        return merged

    async def _decompose_goal(self, goal: str) -> list[Task]:
        response = await self.llm.chat.completions.create(
            model="gpt-4o",
            messages=[{"role": "user", "content": 
                f"Decompose this goal into sub-tasks with dependencies. "
                f"Assign each to a role: researcher, coder, reviewer, tester.\n"
                f"Goal: {goal}\nReturn JSON."}],
            response_format={"type": "json_object"}
        )
        tasks_data = json.loads(response.choices[0].message.content)
        return [Task(**t) for t in tasks_data.get("tasks", [])]

    async def _execute_task(self, task: Task) -> dict:
        agent = await self._assign_agent(task.agent_role)
        if not agent:
            return {"error": "No available agent for role", "task_id": task.task_id}
        
        msg = AgentMessage(
            sender="orchestrator",
            recipient=agent.agent_id,
            msg_type="task",
            payload={"task": task.model_dump()},
            correlation_id=task.task_id
        )
        await self.bus.publish(f"agent:{agent.agent_id}", msg)
        
        # Wait for result
        result_msg = await self.bus.subscribe(
            f"result:{task.task_id}", 
            timeout=120
        )
        return result_msg.payload.get("result", {})

    async def _assign_agent(self, role: AgentRole) -> AgentRegistration | None:
        available = [a for a in self.agents.values() 
                     if a.role == role and a.status == "idle"]
        return available[0] if available else None

    async def _merge_results(self, tasks: list[Task]) -> dict:
        response = await self.llm.chat.completions.create(
            model="gpt-4o",
            messages=[{"role": "user", "content": 
                f"Merge these task outputs into a coherent final result:\n"
                + json.dumps({t.task_id: t.output_data for t in tasks})
                + "\nReturn as JSON."}]
        )
        return json.loads(response.choices[0].message.content)
```

## 9. Python Implementation
```python
from fastapi import FastAPI
import asyncio
from enum import Enum

app = FastAPI()

class DebateAgent:
    """Implements multi-agent debate for improved reasoning"""
    
    def __init__(self, llm, num_agents=3, num_rounds=3):
        self.llm = llm
        self.num_agents = num_agents
        self.num_rounds = num_rounds

    async def debate(self, question: str) -> dict:
        positions = await self._generate_initial_positions(question)
        transcript = []
        
        for round_num in range(self.num_rounds):
            round_responses = []
            for i, pos in enumerate(positions):
                critique = await self._critique_other_positions(pos, positions, i)
                rebuttal = await self._generate_rebuttal(pos, critique, question)
                positions[i] = rebuttal
                round_responses.append({
                    "agent": i,
                    "position": rebuttal,
                    "critique": critique
                })
            transcript.append({"round": round_num + 1, "responses": round_responses})
        
        consensus = await self._reach_consensus(positions, question)
        return {
            "question": question,
            "transcript": transcript,
            "consensus": consensus,
            "agreement": await self._measure_agreement(positions)
        }

    async def _generate_initial_positions(self, question: str) -> list[str]:
        positions = []
        for i in range(self.num_agents):
            resp = await self.llm.chat.completions.create(
                model="gpt-4o",
                messages=[{"role": "user", "content": 
                    f"Take a distinct perspective on this question and state your position:\n{question}\n"
                    f"Agent {i} should have a unique viewpoint from agents {[(j) for j in range(self.num_agents) if j != i]}."}]
            )
            positions.append(resp.choices[0].message.content)
        return positions

    async def _critique_other_positions(self, my_pos: str, all_positions: list, my_idx: int) -> str:
        others = [p for i, p in enumerate(all_positions) if i != my_idx]
        resp = await self.llm.chat.completions.create(
            model="gpt-4o",
            messages=[{"role": "user", "content": 
                f"Your position: {my_pos}\n\nOther positions: {others}\n\n"
                f"Critique the flaws in the other positions."}]
        )
        return resp.choices[0].message.content

    async def _generate_rebuttal(self, my_pos: str, critique: str, question: str) -> str:
        resp = await self.llm.chat.completions.create(
            model="gpt-4o",
            messages=[{"role": "user", "content": 
                f"Refine your position considering this critique.\n"
                f"Original: {my_pos}\nCritique: {critique}\n"
                f"Question: {question}\nUpdated position:"}]
        )
        return resp.choices[0].message.content

    async def _reach_consensus(self, positions: list[str], question: str) -> str:
        resp = await self.llm.chat.completions.create(
            model="gpt-4o",
            messages=[{"role": "user", "content": 
                f"Synthesize these positions into a consensus answer.\n"
                f"Question: {question}\nPositions: {json.dumps(positions)}"}]
        )
        return resp.choices[0].message.content

    async def _measure_agreement(self, positions: list[str]) -> float:
        # Simple semantic similarity heuristic
        return 0.0  # Advanced embedding comparison in production


class SwarmAgent:
    """Implements a swarm-style agent system (like BeeJee)"""
    
    def __init__(self, llm, role: str, instructions: str):
        self.llm = llm
        self.role = role
        self.instructions = instructions

    async def process(self, context: dict) -> dict:
        resp = await self.llm.chat.completions.create(
            model="gpt-4o",
            messages=[
                {"role": "system", "content": f"You are a {self.role}. {self.instructions}"},
                {"role": "user", "content": json.dumps(context)}
            ]
        )
        return {
            "role": self.role,
            "output": resp.choices[0].message.content
        }
```

## 10. Folder Structure
```
multi_agent/
├── orchestrator/
│   ├── orchestrator.py       # Central orchestrator
│   ├── task_graph.py         # Task dependency DAG
│   └── conflict_resolver.py  # Dispute resolution
├── agents/
│   ├── base_agent.py         # Abstract agent class
│   ├── researcher.py
│   ├── coder.py
│   ├── reviewer.py
│   ├── tester.py
│   └── documentation.py
├── communication/
│   ├── message_bus.py        # Redis/Kafka pub/sub
│   ├── message_types.py      # Message schemas
│   └── protocol.py           # Agent communication protocol
├── patterns/
│   ├── debate.py             # Multi-agent debate
│   ├── swarm.py              # Swarm execution
│   ├── pipeline.py           # Sequential pipeline
│   └── voting.py             # Majority voting
├── shared/
│   ├── state_manager.py      # Shared state (Redis)
│   └── knowledge_base.py     # Common KB access
└── api/
    └── routes.py
```

## 11. Configuration
```yaml
multi_agent:
  orchestrator:
    max_concurrent_agents: 10
    task_timeout_seconds: 300
    result_merge_strategy: weighted_vote
  
  agents:
    researcher:
      model: gpt-4o
      replicas: 3
      timeout: 120
    coder:
      model: gpt-4o
      replicas: 5
      timeout: 180
    reviewer:
      model: gpt-4o
      replicas: 2
      timeout: 60
  
  debate:
    num_agents: 3
    num_rounds: 3
    consensus_threshold: 0.7
  
  communication:
    backend: redis
    message_ttl_seconds: 3600
    max_message_size: 1048576
```

## 12. Flowchart
```
[Goal] -> [Orchestrator]
             |
        [Decompose Task]
             |
        [Build Dependency DAG]
             |
        [Identify Parallel Tasks]
             |
   +---------+---------+
   |         |         |
[Agent1] [Agent2] [Agent3]
   |         |         |
   +---------+---------+
             |
     [Any Conflicts?] --Yes--> [Debate/Resolve]
             |                        |
             No                  [Resolved?]
             |                   Yes     No
             v                     v     v
     [Merge Results]         [Merge] [Override]
             |
        [Final Output]
```

## 13. Sequence Diagram
```
User      Orchestrator      AgentA(Coder)    AgentB(Reviewer)   AgentC(Tester)
 |            |                  |                |                 |
 |--goal----->|                  |                |                 |
 |            |--task_1--------->|                |                 |
 |            |--task_2-------------------------->|                 |
 |            |--task_3------------------------------------------->|
 |            |                  |                |                 |
 |            |<--code-----------|                |                 |
 |            |                  |                |                 |
 |            |--review----------------------->|                 |
 |            |<--critique----------------------|                 |
 |            |                  |                |                 |
 |            |--revise--------->|                |                 |
 |            |<--fixed_code-----|                |                 |
 |            |                  |                |                 |
 |            |--test---------------------------------------------->|
 |            |<--passed------------------------------------------------|
 |            |                  |                |                 |
 |<--result---|                  |                |                 |
```

## 14. Pros
- Specialization: each agent excels at one thing
- Parallel execution: independent sub-tasks run concurrently
- Error catching: reviewer catches coder's mistakes
- Scalable: add more agent types without changing architecture
- Robust: one agent failing doesn't kill the whole system

## 15. Cons
- Coordination overhead: message passing, serialization, orchestration
- More expensive: N agents = N times the LLM calls
- Complex debugging: failure could be in any agent or in communication
- Orchestrator becomes SPOF (single point of failure)
- Hard to test: combinatorial interactions between agents

## 16. Alternatives
- **Single agent with tool loop** — simpler, same capabilities for many tasks
- **MoE (Mixture of Experts)** — model-level specialization, not agent-level
- **Pipeline (single-pass)** — no feedback loop, faster but less accurate
- **Ensemble (independent)** — run N copies, vote, no coordination

## 17. Performance Considerations
- Parallel agents reduce wall-clock time but increase total compute
- Message bus is the bottleneck — keep messages small
- Orchestrator merge is serial — optimize for large result sets
- Agent warm-up time (model loading) adds to first-request latency
- Use connection pooling for shared resources (DB, vector store)

## 18. Scaling to Millions
```
Agent Pool:
  - K8s with HPA per agent type (coder pool scales independently)
  - Agent pods with resource limits (CPU/memory per role)
  - Pre-warm agent pools (min 1 replica per type always hot)
  - GPU scheduling for agents that need inference

Message Bus:
  - Kafka with partition per agent type (coder: partition 0, reviewer: partition 1)
  - Dead letter queue for failed messages
  - Message compaction for state replay

Shared State:
  - Redis Cluster with persistence
  - State sharded by session_id
  - TTL-based cleanup for completed sessions
```

## 19. Failure Scenarios
| Scenario | Impact | Mitigation |
|----------|--------|------------|
| Agent hangs | Task timeout | asyncio.wait_for per agent call |
| Conflicting results | Wrong output | Voting merge, orchestrator override |
| Orchestrator dies | Entire system down | Hot standby orchestrator (leader election) |
| Message loss | Lost results | Persistent queues (Kafka), exactly-once delivery |
| Agent divergence | Stale results | Checkpoint + rollback to consensus |

## 20. Security
- Agents authenticate via API keys (not message-based auth)
- Message bus access controlled (agent A cannot impersonate agent B)
- Agent outputs validated before merging (schema + safety)
- Orchestrator authorizes all inter-agent communication
- Secrets injected per agent role (coder gets DB creds, docs agent doesn't)

## 21. Monitoring
```prometheus
mas_agents_active{role="coder|reviewer|tester"}
mas_tasks_queued_total{priority="high|medium|low"}
mas_task_latency_seconds{role="*"}
mas_task_status{status="completed|failed|timeout"}
mas_conflicts_total{type="vote|orchestrator"}
mas_message_bus_throughput
mas_agent_utilization_per_role
mas_orchestrator_latency_seconds
```

## 22. Interview Questions
**Q1**: When would you use multi-agent vs single-agent? (Complex tasks needing diverse expertise, tasks with review/iteration cycles, tasks with clearly independent subproblems)

**Q2**: How do you handle disagreements between agents? (Weighted voting by confidence, orchestrator override, escalate to human)

**Q3**: What is the orchestrator's failure mode? (Single point of failure; mitigations: stateless orchestrator + hot standby + idempotent task dispatch)

## 23. Cheat Sheet
```
MAS Roles: Orchestrator, Worker, Reviewer, Specialists
Patterns: Pipeline, Swarm, Debate, Hierarchy, Voting
Communication: Request/Response, Pub/Sub, Broadcast
Conflict resolution: Majority vote, Confidence-weighted, Orchestrator override, Human review
Key metrics: Agent utilization, Task throughput, Conflict rate, Merge latency
```

## 24. Common Mistakes
- Over-decomposing tasks (too many micro-agents, overhead dominates)
- Under-decomposing (agents too general, no specialization benefit)
- Not handling agent failures — orchestrator should reassign
- Synchronous orchestration — destroys parallelism benefits
- No message size limit — giant payloads clog the bus
- Agents sharing too much context — bloat, privacy issues

## 25. Production Best Practices
1. Start with 2-3 agent types, expand based on observed bottlenecks
2. Make orchestrator stateless — store state in Redis, not in memory
3. Implement circuit breakers per agent type (if reviewer fails 5 times, skip review)
4. Log full message traces for debugging agent interactions
5. Use backpressure: if agents can't keep up, throttle submission
6. Agent health checks with heartbeat — restart unresponsive agents
7. Version agent code independently — A/B test new agent versions
8. Measure value-per-agent: track which agents actually improve final quality
