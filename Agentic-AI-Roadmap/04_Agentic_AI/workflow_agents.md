# Workflow Agents

## 1. What is it?
Workflow agents are structured execution patterns that chain, route, or parallelize agent steps. Unlike free-form agents, workflow agents follow predefined topologies: chains (sequential), routers (conditional branching), parallel (fan-out/fan-in), and orchestrator-subworkflow (nested). These patterns mirror enterprise workflow engines but with LLM-powered decision nodes.

## 2. Why do we need it?
Unstructured agents are unpredictable — they might skip steps, loop, or take wrong paths. Workflow agents impose structure: guaranteed execution order, clear failure handling, auditability, and parallel execution where possible. They combine the flexibility of LLMs with the reliability of deterministic workflows.

## 3. Real-world Example
**Amazon's customer service bot**: Uses a routing workflow. (1) Chain step: classify intent (return, refund, complaint). (2) Router: based on intent, route to sub-workflow. (3) Parallel: for refund, simultaneously check order DB + refund policy + customer history. (4) Chain: generate response. If any step fails, error handler routes to human agent.

## 4. Architecture Diagram (ASCII)
```
                 +---------+
                 |  Start   |
                 +---------+
                     |
              +------v------+
              |  Classifier  |   (Router)
              | (LLM Node)   |
              +------+------+
                     |
       +--------+----+----+--------+
       |        |         |        |
  [Return]  [Refund]  [Complaint]  [Other]
       |        |         |        |
  +----v--+ +--v---+ +---v--+ +---v--+
  | Chain | |Paral| | Chain| |Chain |
  | Steps | |lel  | | Steps| |Steps |
  +-------+ +--+---+ +------+ +------+
                |
          +-----+------+
          |  Merge      |
          +-----+-------+
                |
          +-----v------+
          |  Generate   |
          |  Response   |
          +-----+------+
                |
          +-----v------+
          |  End        |
          +-------------+
```

## 5. Internal Working
Each workflow pattern has a distinct execution model:
- **Chain**: Steps execute in sequence. Output of step N is input to step N+1. Simple, predictable, debuggable.
- **Router**: A classifier step evaluates input and sends it to one of N branches. Each branch is a sub-workflow.
- **Parallel**: Fan-out: split input into N chunks, process each in parallel. Fan-in: merge results.
- **Orchestrator**: A central agent decomposes work, assigns to sub-agents, monitors progress, merges results. Unlike simple routing, the orchestrator dynamically decides the plan.

## 6. Production Flow
```
Chain:   A -> B -> C -> D
Router:  Input -> Classify -> Branch A | Branch B | Branch C
Parallel:Input -> Split -> [P1, P2, P3] (parallel) -> Merge -> Output
Orch:    Goal -> Orchestrator -> [Sub1, Sub2, Sub3] -> Merge -> Output
```

## 7. HLD
```
+----------------------+
|  Workflow Engine      |
|  (State machine +     |
|   step executor)      |
+----------------------+
    |     |     |     |
    v     v     v     v
+----+ +----+ +----+ +----+
|Step| |Step| |Step| |Step|
|1   | |2   | |3(N)| |4   |
+----+ +----+ +----+ +----+
    |     |     |     |
    v     v     v     v
+----------------------+
|  Step Executor Pool   |
|  (LLM calls, tools,   |
|   API integrations)   |
+----------------------+
```

## 8. LLD
```python
from pydantic import BaseModel, Field
from enum import Enum
from typing import Any, Callable, Awaitable
import asyncio
from datetime import datetime
import uuid

class StepType(str, Enum):
    LLM = "llm"
    TOOL = "tool"
    CODE = "code"
    ROUTER = "router"
    PARALLEL = "parallel"
    MERGE = "merge"
    HUMAN = "human"

class WorkflowStatus(str, Enum):
    PENDING = "pending"
    RUNNING = "running"
    PAUSED = "paused"
    COMPLETED = "completed"
    FAILED = "failed"

class WorkflowStep(BaseModel):
    step_id: str
    name: str
    step_type: StepType
    config: dict[str, Any] = Field(default_factory=dict)
    depends_on: list[str] = Field(default_factory=list)
    retry_count: int = 0
    max_retries: int = 2
    timeout: int = 60
    input_mapping: dict[str, str] = Field(
        default_factory=dict
    )  # maps "step_id.field" -> "this_step.input_field"

class Workflow(BaseModel):
    workflow_id: str = Field(default_factory=lambda: uuid.uuid4().hex)
    name: str
    steps: list[WorkflowStep]
    status: WorkflowStatus = WorkflowStatus.PENDING
    context: dict[str, Any] = Field(default_factory=dict)
    created_at: datetime = Field(default_factory=datetime.utcnow)
    execution_history: list[dict] = Field(default_factory=list)

class WorkflowEngine:
    def __init__(self):
        self.executors: dict[StepType, Callable] = {}

    def register_executor(self, step_type: StepType, executor: Callable):
        self.executors[step_type] = executor

    async def execute(self, workflow: Workflow) -> dict:
        workflow.status = WorkflowStatus.RUNNING
        completed = set()

        while len(completed) < len(workflow.steps):
            ready = [s for s in workflow.steps 
                     if s.step_id not in completed 
                     and all(d in completed for d in s.depends_on)]
            
            results = await asyncio.gather(*[
                self._execute_step(workflow, step) for step in ready
            ], return_exceptions=True)
            
            for step, result in zip(ready, results):
                if isinstance(result, Exception):
                    workflow.execution_history.append({
                        "step_id": step.step_id,
                        "error": str(result),
                        "timestamp": datetime.utcnow().isoformat()
                    })
                    if step.retry_count >= step.max_retries:
                        workflow.status = WorkflowStatus.FAILED
                        return {"error": f"Step {step.name} failed: {result}"}
                    step.retry_count += 1
                else:
                    workflow.context[step.step_id] = result
                    completed.add(step.step_id)

        workflow.status = WorkflowStatus.COMPLETED
        return workflow.context

    async def _execute_step(self, workflow: Workflow, step: WorkflowStep) -> Any:
        executor = self.executors.get(step.step_type)
        if not executor:
            raise ValueError(f"No executor for {step.step_type}")
        
        input_data = self._resolve_inputs(workflow, step)
        return await asyncio.wait_for(
            executor(input_data, step.config),
            timeout=step.timeout
        )

    def _resolve_inputs(self, workflow: Workflow, step: WorkflowStep) -> dict:
        inputs = {}
        for target_field, source in step.input_mapping.items():
            if "." in source:
                step_id, field = source.split(".")
                inputs[target_field] = workflow.context.get(step_id, {}).get(field)
            else:
                inputs[target_field] = workflow.context.get(source)
        return inputs


class ChainAgent:
    def __init__(self, llm, steps: list[dict]):
        self.llm = llm
        self.steps = steps

    async def run(self, initial_input: str) -> str:
        current = initial_input
        for i, step in enumerate(self.steps):
            result = await self.llm.chat.completions.create(
                model=step.get("model", "gpt-4o"),
                messages=[
                    {"role": "system", "content": step["prompt"]},
                    {"role": "user", "content": current}
                ]
            )
            current = result.choices[0].message.content
        return current


class RouterAgent:
    def __init__(self, llm, routes: dict[str, callable]):
        self.llm = llm
        self.routes = routes

    async def route(self, input_text: str) -> Any:
        response = await self.llm.chat.completions.create(
            model="gpt-4o",
            messages=[{"role": "user", "content": 
                f"Classify this input into one of: {list(self.routes.keys())}\n"
                f"Return only the route name.\n\nInput: {input_text}"}]
        )
        route_name = response.choices[0].message.content.strip()
        handler = self.routes.get(route_name)
        if handler:
            return await handler(input_text)
        raise ValueError(f"No route for: {route_name}")
```

## 9. Python Implementation
```python
from fastapi import FastAPI, HTTPException
from pydantic import BaseModel
import json

app = FastAPI()

class ParallelExecutor:
    def __init__(self, workers: list[callable], merge_fn: callable = None):
        self.workers = workers
        self.merge_fn = merge_fn or (lambda x: "\n".join(x))

    async def execute(self, input_data: Any) -> Any:
        async def wrapped(worker, data):
            return await worker(data)

        tasks = [wrapped(w, input_data) for w in self.workers]
        results = await asyncio.gather(*tasks)
        return self.merge_fn(results)


class WorkflowBuilder:
    """Fluent API for building workflows"""
    
    def __init__(self, name: str):
        self.workflow = Workflow(name=name, steps=[])

    def add_step(self, name: str, step_type: StepType, config: dict = None, 
                 depends_on: list[str] = None, timeout: int = 60):
        self.workflow.steps.append(WorkflowStep(
            step_id=name.lower().replace(" ", "_"),
            name=name,
            step_type=step_type,
            config=config or {},
            depends_on=depends_on or [],
            timeout=timeout,
            input_mapping={"input": "input"}
        ))
        return self

    def parallel(self, name: str, step_names: list[str], merge_config: dict = None):
        self.workflow.steps.append(WorkflowStep(
            step_id=name.lower().replace(" ", "_"),
            name=name,
            step_type=StepType.PARALLEL,
            config={
                "sub_steps": step_names,
                "merge": merge_config or {}
            },
            depends_on=step_names
        ))
        return self

    def build(self) -> Workflow:
        return self.workflow


# Example: Customer Support Workflow

async def classify_intent(input_data: dict, config: dict) -> dict:
    llm = AsyncOpenAI()
    response = await llm.chat.completions.create(
        model="gpt-4o",
        messages=[{"role": "user", "content": 
            f"Classify this support request into: 'refund', 'return', 'complaint', or 'other'.\n"
            f"Request: {input_data.get('message', '')}\nReturn only the category."}]
    )
    return {"intent": response.choices[0].message.content.strip()}

async def check_order(input_data: dict, config: dict) -> dict:
    # Call order DB
    order_id = input_data.get("order_id", "")
    return {"order_status": "delivered", "order_date": "2024-01-15"}

async def check_refund_policy(input_data: dict, config: dict) -> dict:
    return {"eligible": True, "max_refund": 100.0}

async def generate_response(input_data: dict, config: dict) -> dict:
    llm = AsyncOpenAI()
    parts = []
    for key, value in input_data.items():
        parts.append(f"{key}: {value}")
    context = "\n".join(parts)
    
    response = await llm.chat.completions.create(
        model="gpt-4o",
        messages=[{"role": "user", "content": 
            f"Generate a customer support response based on:\n{context}"}]
    )
    return {"response": response.choices[0].message.content}


@app.post("/workflow/run")
async def run_workflow(request: dict):
    engine = WorkflowEngine()
    engine.register_executor(StepType.LLM, classify_intent)
    engine.register_executor(StepType.TOOL, check_order)
    
    builder = WorkflowBuilder("customer_support")
    builder.add_step("classify", StepType.LLM, timeout=30)
    builder.add_step("check_order", StepType.TOOL, depends_on=["classify"])
    builder.add_step("check_policy", StepType.TOOL, depends_on=["classify"])
    builder.parallel("merge_results", ["check_order", "check_policy"],
                     merge_config={"strategy": "concatenate"})
    builder.add_step("generate", StepType.LLM, depends_on=["merge_results"])
    
    workflow = builder.build()
    result = await engine.execute(workflow)
    return result
```

## 10. Folder Structure
```
workflow_agents/
├── engine/
│   ├── workflow_engine.py  # Core state machine
│   ├── step_executor.py    # Step execution with retry
│   └── context_manager.py  # Shared context across steps
├── patterns/
│   ├── chain.py            # Sequential chain
│   ├── router.py           # Conditional branching
│   ├── parallel.py         # Fan-out/fan-in
│   ├── orchestrator.py     # Dynamic sub-workflows
│   └── map_reduce.py       # Map-reduce pattern
├── nodes/
│   ├── llm_node.py         # LLM call node
│   ├── tool_node.py        # Tool execution node
│   ├── code_node.py        # Code execution node
│   └── human_node.py       # Human-in-the-loop node
├── monitoring/
│   ├── tracer.py           # Step-level tracing
│   └── metrics.py          # Step duration tracking
└── api/
    └── routes.py
```

## 11. Configuration
```yaml
workflow:
  engine:
    max_concurrent_steps: 10
    step_default_timeout: 60
    max_retries: 2
    state_persistence: redis
  
  patterns:
    chain:
      stop_on_failure: true
    router:
      classifier_model: gpt-4o-mini
    parallel:
      max_parallelism: 5
      merge_strategy: concat
    map_reduce:
      map_model: gpt-4o-mini
      reduce_model: gpt-4o
  
  monitoring:
    trace_steps: true
    collect_llm_usage: true
```

## 12. Flowchart
```
[Input]
   |
[Chain: Validate]
   |
[Router: Classify]
   |
   +----[Refund]----+
   |                |
[Parallel:     [Chain:
 CheckOrder +   Generate
 CheckPolicy]   Response]
   |                |
   +----[Merge]-----+
   |
[Chain: Format Output]
   |
[Output]
```

## 13. Sequence Diagram
```
User        WFEngine       Step1:Classify   Step2:CheckOrd   Step3:GenResp
 |             |                |               |               |
 |--request--->|                |               |               |
 |             |--execute------>|               |               |
 |             |<--intent-------|               |               |
 |             |                |               |               |
 |             |--execute(par)----------------->|               |
 |             |<--order_data-------------------|               |
 |             |                |               |               |
 |             |--execute--------------------->|               |
 |             |<--response------------------------------------|
 |             |                |               |               |
 |<--result----|                |               |               |
```

## 14. Pros
- Predictable execution order (no agent wandering)
- Easy to debug — each step is a checkpoint
- Retry logic per step without affecting others
- Parallel steps reduce wall-clock time
- Human-in-the-loop possible at any step

## 15. Cons
- Rigid structure — can't adapt to unexpected situations
- Router depends on classifier accuracy
- Parallel fan-out can overwhelm rate limits
- State management grows complex with many steps
- Boilerplate for simple chains

## 16. Alternatives
- **LangGraph** — graph-based state machines for agents
- **Temporal/Airflow** — deterministic workflow engines (no LLM)
- **Durable functions (Azure)** — serverless workflow orchestration
- **Free-form ReAct** — maximum flexibility, minimum structure

## 17. Performance Considerations
- Step serialization/deserialization adds ~5ms per step
- Parallel steps should be independent — no shared state
- Router latency is 1 LLM call (200-500ms)
- Chain of N steps = N sequential LLM calls (N × latency)
- State persists in Redis — keep per-step data small

## 18. Scaling to Millions
```
Horizontal Scaling:
  - Workflow engine stateless (state in Redis)
  - Step executors as K8s jobs (ephemeral per step)
  - Parallel steps distributed across worker pool
  - Router classification via fast classifier (distilbert, not GPT-4o)

Bottlenecks:
  - State persistence: Redis Cluster (80K ops/sec)
  - Router: can be pre-computed offline for known patterns
  - Merge step: O(N) where N = parallel branches

Optimizations:
  - Pre-compile workflows to execution DAGs
  - Cache step results for identical inputs (idempotent steps)
  - Batch small steps into single LLM calls
```

## 19. Failure Scenarios
| Scenario | Impact | Mitigation |
|----------|--------|------------|
| Step timeout | Pipeline stall | Per-step timeout, continue to error handler |
| Router misclassifies | Wrong branch | Confidence threshold: <0.7 = human review |
| Parallel step fails | Partial data | Fan-in with partial results, mark step degraded |
| State corruption | Wrong outputs | Versioned state snapshots |
| Infinite loop in chain | Resource drain | Max chain length guard |

## 20. Security
- Step outputs validated against schema before passing to next step
- Router branch names are whitelisted (not dynamically constructed)
- State context sanitized before LLM injection
- Human-in-the-loop steps require authentication
- Workflow definitions are versioned and immutable after deployment

## 21. Monitoring
```prometheus
workflow_executions_total{status="completed|failed|timeout"}
workflow_step_duration_seconds{step_type="*"}
workflow_router_accuracy
workflow_parallel_efficiency (speedup / num_workers)
workflow_state_size_bytes
workflow_human_loop_requests_total
workflow_active_runs
```

## 22. Interview Questions
**Q1**: When would you use a chain vs a router? (Chain for linear transformations — translate, summarize, format. Router for conditional paths — based on intent, user type, data classification)

**Q2**: How do you handle partial failures in parallel workflows? (Two strategies: strict (all must succeed) or partial (best-effort, mark degraded). Choose based on task criticality)

**Q3**: How would you implement dynamic workflows where the next step isn't known until runtime? (Use orchestrator pattern: an LLM agent decides next step based on current state, adds it to the DAG dynamically)

## 23. Cheat Sheet
```
Chain:     A -> B -> C -> D (sequential, N LLM calls)
Router:    Input -> Classify -> Branch (1 classifier + 1 executor)
Parallel:  Input -> [A, B, C] -> Merge (N parallel + 1 merge)
Orch:      Goal -> Dynamic decomposer -> N sub-tasks -> Merge
Best for:  Routers when branches are known, Chains when order is fixed
```

## 24. Common Mistakes
- Using chain for everything — misses parallelism opportunities
- Router without confidence check — wrong branch silently
- Parallel steps that share state — race conditions
- No timeout per step — one stuck step blocks entire workflow
- Too many steps — workflow becomes unmanageable DAG
- State mutation in parallel steps — output depends on execution order

## 25. Production Best Practices
1. Always set per-step timeout (default 60s, raise if LLM-heavy)
2. Use router with confidence threshold (below threshold = route to human)
3. Implement idempotent steps (re-running should be safe)
4. Monitor step duration trends — degrading step is early failure signal
5. Cache router classifications for identical inputs
6. Version workflow definitions — cannot change a running workflow mid-flight
7. Log step-level traces with input/output for debugging
8. Test each step independently before integrating into workflow
