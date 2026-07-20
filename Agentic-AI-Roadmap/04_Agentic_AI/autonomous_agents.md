# Fully Autonomous Agents (AutoGPT-style)

> Author: **Tamilselvan** · ✉️ tamilselvan.sde@gmail.com · 🔗 [LinkedIn](https://www.linkedin.com/in/tamilselvan-ai/)
>

## 1. What is it?
A fully autonomous agent operates independently over extended periods — hours or days — to achieve complex, open-ended goals. It maintains its own task list, prioritizes work, manages its own context window, stores long-term memory, and self-corrects without human intervention. The canonical implementation is AutoGPT, which broke new ground in 2023.

## 2. Why do we need it?
Most agents require human prompting every few minutes. Autonomous agents work on goals that take hours: writing entire codebases, conducting multi-source research, managing social media campaigns, or running business processes. They handle ambiguity, recover from failures, and persist across sessions — operating more like a junior employee than a chatbot.

## 3. Real-world Example
**Microsoft Copilot (autonomous mode)**: Given "plan our Q2 marketing campaign," the agent: researches competitors, outlines strategy, drafts content for 5 channels, creates a Gantt chart, schedules social posts, and monitors performance — working autonomously for 8+ hours. It asks for human input only when decisions exceed its authority (budget approval, brand voice conflicts).

## 4. Architecture Diagram (ASCII)
```
+------------------------------------------------------------------+
|                     AUTONOMOUS AGENT                                 |
|                                                                      |
|  +----------------+     +----------------+     +-----------------+  |
|  | Goal Manager   |---->| Task Queue     |---->| Execution Loop  |  |
|  | (long-term     |     | (prioritized)  |     | (perceive, act, |  |
|  |  objectives)   |     |                |     |  evaluate)      |  |
|  +----------------+     +----------------+     +-----------------+  |
|        |                                                   |        |
|        v                                                   v        |
|  +----------------+     +----------------+     +-----------------+  |
|  | Memory System  |<--->| Context Manager|<--->| Tool Executor   |  |
|  | (ST, LT, Epi)  |     | (token budget) |     | (parallel)      |  |
|  +----------------+     +----------------+     +-----------------+  |
|        |                        |                        |          |
|        v                        v                        v          |
|  +----------------+     +----------------+     +-----------------+  |
|  | Vector Store   |     | LLM (GPT-4o)  |     | Sandbox (code)  |  |
|  +----------------+     +----------------+     +-----------------+  |
|                                                                      |
|  +------------------------------------------------------------+    |
|  | Self-Reflection: Monitor progress, re-prioritize, detect  |    |
|  |                 loops, request human if stuck              |    |
|  +------------------------------------------------------------+    |
+------------------------------------------------------------------+
```

## 5. Internal Working
The agent starts with a high-level goal. The Goal Manager decomposes this into concrete tasks (JSON objects with description, priority, dependencies). Tasks go into a priority queue. The Execution Loop pops the highest-priority task, processes it (ReAct-style: think, act, observe), stores results in memory, marks task complete, and repeats. The Context Manager ensures the LLM's context window doesn't overflow — it summarizes old conversation turns. Self-reflection runs every N steps: "Am I making progress toward the goal?" If not, it re-prioritizes tasks or asks for human guidance.

## 6. Production Flow
```
1. User sets goal (e.g., "Create a React dashboard")
2. Goal Manager decomposes: [design DB, create API, build UI, write tests]
3. Tasks enter priority queue with dependencies
4. Loop:
   a. Pop next task
   b. Load relevant memory context
   c. LLM reasons about how to complete task
   d. Execute tool calls (code, search, file I/O)
   e. Evaluate result
   f. Store in memory (short + episodic)
   g. Check: is task complete?
   h. If stuck: re-prioritize, try alternative approach
   i. Every 5 steps: self-reflection ("on track?")
5. All tasks complete -> final summary to user
```

## 7. HLD
```
+------------------+
|  User Interface   |
|  (WebSocket)      |
+------------------+
         |
         v
+---------------------------+
|   Autonomous Agent Service |
|  +---------------------+  |
|  | Goal Manager        |  |
|  +---------------------+  |
|  +---------------------+  |
|  | Task Scheduler      |  |
|  +---------------------+  |
|  +---------------------+  |
|  | Execution Engine    |  |
|  +---------------------+  |
|  +---------------------+  |
|  | Context Manager     |  |
|  +---------------------+  |
|  +---------------------+  |
|  | Reflection Engine   |  |
|  +---------------------+  |
+---------------------------+
    |     |     |     |     |
    v     v     v     v     v
+-----+ +---+ +---+ +---+ +---+
| LLM | |RAG| |Code| |File| |Web|
|     | |   | |Exec| |Sys | |   |
+-----+ +---+ +---+ +---+ +---+
```

## 8. LLD
```python
from pydantic import BaseModel, Field
from enum import Enum
from typing import Any
import asyncio
import json
from datetime import datetime
import uuid

class TaskPriority(Enum):
    CRITICAL = 0
    HIGH = 1
    MEDIUM = 2
    LOW = 3

class TaskStatus(Enum):
    PENDING = "pending"
    IN_PROGRESS = "in_progress"
    COMPLETED = "completed"
    FAILED = "failed"
    BLOCKED = "blocked"
    SKIPPED = "skipped"

class Task(BaseModel):
    task_id: str = Field(default_factory=lambda: uuid.uuid4().hex)
    description: str
    priority: TaskPriority = TaskPriority.MEDIUM
    status: TaskStatus = TaskStatus.PENDING
    depends_on: list[str] = Field(default_factory=list)
    created_at: datetime = Field(default_factory=datetime.utcnow)
    result: Any = None
    error: str | None = None
    attempts: int = 0
    max_attempts: int = 3

class Goal(BaseModel):
    goal_id: str = Field(default_factory=lambda: uuid.uuid4().hex)
    description: str
    tasks: list[Task] = Field(default_factory=list)
    created_at: datetime = Field(default_factory=datetime.utcnow)
    status: str = "active"
    progress: float = 0.0  # 0.0 to 1.0

class AutonomousAgent:
    def __init__(self, llm, memory, tool_registry, max_consecutive_failures=3):
        self.llm = llm
        self.memory = memory
        self.tools = tool_registry
        self.max_consecutive_failures = max_consecutive_failures
        self.reflection_interval = 5

    async def run(self, goal_description: str, max_steps: int = 100) -> Goal:
        goal = await self._initialize_goal(goal_description)
        consecutive_failures = 0

        for step in range(max_steps):
            if goal.status != "active":
                break

            ready_tasks = [t for t in goal.tasks 
                          if t.status == TaskStatus.PENDING
                          and all(self._is_task_done(tid, goal.tasks) for tid in t.depends_on)]
            
            if not ready_tasks:
                if all(t.status == TaskStatus.COMPLETED for t in goal.tasks):
                    goal.status = "completed"
                    break
                elif all(t.status in (TaskStatus.COMPLETED, TaskStatus.FAILED, TaskStatus.SKIPPED) 
                        for t in goal.tasks):
                    goal.status = "completed_with_failures"
                    break
                break

            task = ready_tasks[0]
            task.status = TaskStatus.IN_PROGRESS
            
            context = await self._build_context(goal, task)
            
            result = await self._execute_task(task, context)
            
            if result["success"]:
                task.status = TaskStatus.COMPLETED
                task.result = result["output"]
                consecutive_failures = 0
            else:
                task.attempts += 1
                if task.attempts >= task.max_attempts:
                    task.status = TaskStatus.FAILED
                    task.error = result["error"]
                else:
                    task.status = TaskStatus.PENDING  # Re-queue
                consecutive_failures += 1

            await self._store_memory(goal, task, result)
            
            if step % self.reflection_interval == 0:
                await self._self_reflect(goal)

            goal.progress = sum(1 for t in goal.tasks if t.status == TaskStatus.COMPLETED) / len(goal.tasks)

        return goal

    async def _initialize_goal(self, description: str) -> Goal:
        response = await self.llm.chat.completions.create(
            model="gpt-4o",
            messages=[{"role": "user", "content": 
                f"Decompose this goal into concrete, actionable tasks. "
                f"Each task should be atomic and achievable in one step.\n"
                f"Return as JSON array with: description, priority (0-3), dependencies (task indices).\n"
                f"Goal: {description}"}],
            response_format={"type": "json_object"}
        )
        tasks_data = json.loads(response.choices[0].message.content)
        
        goal = Goal(description=description)
        for t in tasks_data.get("tasks", []):
            goal.tasks.append(Task(
                description=t["description"],
                priority=TaskPriority(t.get("priority", 2)),
                depends_on=[goal.tasks[i].task_id for i in t.get("depends_on", [])]
            ))
        return goal

    async def _execute_task(self, task: Task, context: str) -> dict:
        response = await self.llm.chat.completions.create(
            model="gpt-4o",
            messages=[
                {"role": "system", "content": f"Complete this task. Available tools: {list(self.tools.keys())}"},
                {"role": "user", "content": f"Task: {task.description}\n\nContext:\n{context}"}
            ],
            tools=self.tools.get_openai_tools(),
            tool_choice="auto",
            parallel_tool_calls=True
        )
        
        msg = response.choices[0].message
        output = msg.content or ""
        tool_results = []

        if msg.tool_calls:
            for tc in msg.tool_calls:
                result = await self.tools.execute_tool({
                    "id": tc.id,
                    "function": {"name": tc.function.name, "arguments": tc.function.arguments}
                })
                tool_results.append(result.model_dump() if hasattr(result, 'model_dump') else result)
        
        return {
            "success": True,  # Basic heuristic
            "output": {"text": output, "tool_results": tool_results},
            "error": None
        }

    async def _self_reflect(self, goal: Goal):
        progress_summary = f"Progress: {goal.progress:.0%}\n"
        for t in goal.tasks:
            progress_summary += f"- {t.description}: {t.status.value}\n"
        if t.error:
            progress_summary += f"  Error: {t.error}\n"

        response = await self.llm.chat.completions.create(
            model="gpt-4o",
            messages=[{"role": "user", "content": 
                f"Reflect on progress and suggest adjustments.\n{progress_summary}\n"
                f"Should any tasks be re-prioritized? Added? Removed? "
                f"Return JSON with: adjustments, new_tasks (if any)."}],
            response_format={"type": "json_object"}
        )
        reflection = json.loads(response.choices[0].message.content)
        
        if reflection.get("adjustments"):
            for t in goal.tasks:
                if t.status == TaskStatus.FAILED and t.attempts < t.max_attempts:
                    t.status = TaskStatus.PENDING
        
        for new_task in reflection.get("new_tasks", []):
            goal.tasks.append(Task(description=new_task["description"]))

    async def _build_context(self, goal: Goal, task: Task) -> str:
        recent = await self.memory.get_recent(5)
        similar = await self.memory.recall_similar(task.description)
        
        return json.dumps({
            "goal": goal.description,
            "task": task.description,
            "recent_actions": recent,
            "similar_past": similar
        })

    def _is_task_done(self, task_id: str, tasks: list[Task]) -> bool:
        for t in tasks:
            if t.task_id == task_id:
                return t.status in (TaskStatus.COMPLETED, TaskStatus.SKIPPED)
        return True
```

## 9. Python Implementation
```python
from fastapi import FastAPI, WebSocket, BackgroundTasks
from pydantic import BaseModel
import asyncio

app = FastAPI()

class AutoGPTAgent(AutonomousAgent):
    def __init__(self, llm, memory, tools, workspace_dir: str = "/tmp/workspace"):
        super().__init__(llm, memory, tools)
        self.workspace = workspace_dir
        self.status = "idle"

    async def long_running_goal(self, goal: str, max_hours: int = 24) -> Goal:
        self.status = "running"
        start = datetime.utcnow()
        
        while (datetime.utcnow() - start).total_seconds() < max_hours * 3600:
            result = await self.run(goal, max_steps=50)
            self.status = "checkpoint"
            
            await self._save_checkpoint(result)
            
            if result.status in ("completed", "completed_with_failures"):
                self.status = "idle"
                return result
            
            # Re-prioritize remaining tasks
            await self._self_reflect(result)
            self.status = "running"
        
        self.status = "timeout"
        return result

    async def _save_checkpoint(self, goal: Goal):
        filename = f"/tmp/checkpoint_{goal.goal_id}.json"
        async with aiofiles.open(filename, "w") as f:
            await f.write(goal.model_dump_json(indent=2))

    async def get_status(self) -> dict:
        return {
            "status": self.status,
            "workspace_files": await self._list_workspace()
        }

    async def _list_workspace(self) -> list[str]:
        import os
        return os.listdir(self.workspace) if os.path.exists(self.workspace) else []


@app.websocket("/agent/ws")
async def agent_websocket(websocket: WebSocket):
    await websocket.accept()
    
    agent = AutoGPTAgent(
        llm=AsyncOpenAI(),
        memory=MemoryService(...),
        tools=ToolRegistry()
    )
    
    async def send_updates():
        while True:
            await asyncio.sleep(5)
            status = await agent.get_status()
            await websocket.send_json(status)
    
    update_task = asyncio.create_task(send_updates())
    
    try:
        data = await websocket.receive_json()
        goal = data.get("goal", "")
        
        result = await agent.long_running_goal(goal)
        await websocket.send_json({"type": "complete", "result": result.model_dump()})
    except Exception as e:
        await websocket.send_json({"type": "error", "message": str(e)})
    finally:
        update_task.cancel()
        await websocket.close()


# Simple AutoGPT-style CLI loop
async def autogpt_cli():
    agent = AutoGPTAgent(llm=AsyncOpenAI(), memory=..., tools=ToolRegistry())
    
    print("AutoGPT Agent ready. Enter your goal:")
    goal = input("> ")
    
    result = await agent.run(goal, max_steps=50)
    print(f"\nGoal completed: {result.status}")
    print(f"Progress: {result.progress:.0%}")
    
    for task in result.tasks:
        icon = "✓" if task.status == TaskStatus.COMPLETED else "✗"
        print(f"  {icon} {task.description}: {task.status.value}")
```

## 10. Folder Structure
```
autonomous_agents/
├── core/
│   ├── agent.py             # AutonomousAgent base
│   ├── goal_manager.py      # Goal decomposition, tracking
│   ├── task_scheduler.py    # Priority queue, dependency resolution
│   ├── execution_loop.py    # Main loop with self-reflection
│   └── context_manager.py   # Sliding window, summarization
├── memory/
│   ├── checkpoint.py        # Session persistence
│   ├── workspace.py         # File system management
│   └── progress_tracker.py  # Goal progress monitoring
├── reflection/
│   ├── self_reflection.py   # Progress assessment
│   ├── task_evaluator.py    # Task completion verification
│   └── re_planner.py        # Course correction
├── api/
│   ├── websocket.py         # Real-time status streaming
│   ├── http_routes.py       # REST endpoints
│   └── cli.py               # CLI interface
├── web/
│   ├── dashboard.py         # Streamlit/Gradio dashboard
│   └── log_viewer.py        # Real-time log viewer
└── main.py                  # Entry point
```

## 11. Configuration
```yaml
autonomous_agent:
  max_steps_per_run: 50
  max_runtime_hours: 24
  max_consecutive_failures: 3
  reflection_interval: 5
  
  goal:
    decomposition_model: gpt-4o
    max_tasks_per_goal: 50
  
  execution:
    model: gpt-4o
    temperature: 0.7
    parallel_tool_calls: true
    max_tokens_per_step: 4096
  
  context:
    type: sliding_window
    max_messages: 20
    summarization_model: gpt-4o-mini
    summary_interval: 10
  
  checkpoint:
    enabled: true
    interval_minutes: 15
    storage: s3
  
  workspace:
    dir: /tmp/workspace
    max_files: 100
    max_storage_mb: 500
```

## 12. Flowchart
```
[User sets Goal]
       |
[Goal Manager decomposes]
       |
[Task Queue built with deps]
       |
+------v------+
| Execution   |<-------------------+
| Loop        |                    |
+------+------+                    |
       |                           |
[Pop highest priority task]        |
       |                           |
[Load memory context]              |
       |                           |
[LLM reasons & executes]           |
       |                           |
[Success?] --Yes--> [Store result] |
       |                           |
       No                          |
       |                           |
[Retry?] --No--> [Mark Failed]     |
       |                           |
       Yes                         |
       |                           |
[Re-queue] ---------------------->+
       |
[Reflection interval?] --Yes--> [Self-Reflect] --Adjustments-->+>
       |
       No
       |
[More tasks?] --Yes--> [Continue]------>+
       |                                  |
       No                                 |
       v                                  |
[Goal complete]                           |
```

## 13. Sequence Diagram
```
User       Agent         LLM          Tools         Memory       FileSys
 |           |            |             |             |            |
 |--goal---->|            |             |             |            |
 |           |--decompose>|             |             |            |
 |           |<--tasks----|             |             |            |
 |           |            |             |             |            |
 |           |            |             |             |            |
 |           |--task_1--->|             |             |            |
 |           |            |--think----->|             |            |
 |           |            |<--actions---|             |            |
 |           |            |             |             |            |
 |           |            |--search---->|             |            |
 |           |            |<--results---|             |            |
 |           |            |             |             |            |
 |           |            |--write------|------------>|            |
 |           |            |             |             |            |
 |           |<--result---|             |             |            |
 |           |            |             |             |            |
 |           |--store---->|             |             |            |
 |           |            |             |             |            |
 |           |            |             |             |            |
 |           |--task_2--->| (repeat)    |             |            |
 |           |            |             |             |            |
 |           |            |             |             |            |
 |           |[checkpoint]|             |             |            |
 |           |--save------|------------>|------------>|            |
 |           |            |             |             |            |
 |<--progress|            |             |             |            |
 |  (WS)     |            |             |             |            |
 |           |            |             |             |            |
 |           |[self-reflect]           |             |            |
 |           |--reflect-->|             |             |            |
 |           |<--adjust---|             |             |            |
 |           |            |             |             |            |
 |<--complete-|            |             |             |            |
```

## 14. Pros
- Works on tasks that take hours or days without supervision
- Self-healing: detects and recovers from failures
- Checkpointing prevents total loss on crash
- Progress tracking visible to user in real-time
- Can handle ambiguity through reflection and re-planning

## 15. Cons
- Very expensive (hundreds to thousands of LLM calls)
- Can drift from original goal over long runs
- Debugging is extremely difficult (long traces)
- Task decomposition quality varies — wrong tasks = wasted steps
- Context management is fragile — poor summarization loses critical info

## 16. Alternatives
- **Human-supervised agent** — user confirms each step
- **BabyAGI** — simpler task-driven autonomous agent
- **TaskWeaver** — Microsoft's plugin-based autonomous agent
- **CrewAI** — multi-agent autonomous collaboration
- **Claude Code (agent mode)** — simpler but less autonomous

## 17. Performance Considerations
- Long runs accumulate context — summarization is essential but lossy
- Checkpoint persistence is I/O bound — batch writes
- Tool execution dominates latency (not LLM)
- Reflection adds overhead but prevents waste from wrong direction
- Parallel task execution reduces total run time but increases cost

## 18. Scaling to Millions
```
Infrastructure:
  - Each user gets an isolated agent pod
  - Pod lifecycle: max 24 hours, auto-archived
  - Checkpoints stored in S3 (not local disk)
  - Agent execution as K8s job (not persistent pod)

Resource Management:
  - Token budgets per session: hard cap at 1M tokens
  - Cost tracking per agent run: alert if >$50
  - GPU scheduling: pool shared across agent pods
  - Queue system: limit concurrent agents to 1000

State Management:
  - Redis for active agent short-term state
  - PostgreSQL for long-term agent memory
  - S3 for file outputs and checkpoints
  - TTL: 7 days for completed agent data
```

## 19. Failure Scenarios
| Scenario | Impact | Mitigation |
|----------|--------|------------|
| Goal drift | Agent does wrong thing | Re-align on reflection step: re-state original goal |
| Infinite loop | Token drain | Max steps, similarity detection on consecutive tasks |
| Task explosion | Goal decomposed into 1000+ micro-tasks | Max task limit, merge similar tasks |
| Context collapse | Forget critical info | Hierarchical summarization: recent verbatim, old compressed |
| Tool API changes | All tool calls fail | Version pinned tool definitions, graceful fallback |
| Agent goes off-rails | Unpredictable behavior | Human interrupt: "pause" command, manual override |

## 20. Security
- Workspace is a sandboxed directory — no access outside it
- Code execution in gVisor containers with network disabled
- Web browsing via proxy that blocks dangerous sites/actions
- API keys injected at runtime, never stored in agent context
- Maximum file write size limit (10MB per file)
- Agent cannot spend money without explicit approval (payment APIs)
- Human-in-the-loop required for: file deletion, external sharing, large data export

## 21. Monitoring
```prometheus
autonomous_runs_active
autonomous_runs_completed_total{status="success|failure|timeout"}
autonomous_steps_executed_total
autonomous_tokens_consumed_per_run
autonomous_cost_per_run_usd
autonomous_task_completion_rate
autonomous_context_summaries_per_run
autonomous_self_reflection_count
autonomous_failure_recovery_rate
```

## 22. Interview Questions
**Q1**: How do you prevent an autonomous agent from going in circles? (Consecutive same-task detection, diversity penalty in task selection, max retries per task, human interrupt as last resort)

**Q2**: How does the agent handle a goal that changes mid-execution? (Goal versioning: checkpoint current state, create new goal trajectory, merge remaining tasks from old goal)

**Q3**: How do you bound the cost of an autonomous agent run? (Max steps, max tokens, max runtime, cost-per-run alert, daily budget per user)

## 23. Cheat Sheet
```
Lifecycle: Goal -> Decompose -> Task Queue -> Execute -> Reflect -> Complete
Key guardrails: max_steps, max_time, max_failures, max_cost
Context strategy: recent (verbatim) + old (summarized) + relevant (RAG)
Checkpoint: every N steps or M minutes (whichever comes first)
Self-reflection: every K steps: "Am I making progress? What should change?"
Human escalation triggers: stuck loop, out-of-scope request, budget exceeded
```

## 24. Common Mistakes
- No max step limit — agent runs until tokens run out
- Task decomposition too granular — 200 micro-tasks for simple goal
- No context window management — agent loses track after 20 steps
- Reflecting too often (>50% of calls are reflection = waste)
- Not checkpointing frequently enough — losing 2 hours of work on crash
- Trusting the agent with tools that cost money (no payment confirmation)
- No user-facing progress updates — feels like a black box

## 25. Production Best Practices
1. Always start with a human-in-the-loop pilot before full autonomy
2. Set hard bounds: max steps, max runtime, max cost per run
3. Implement heartbeat monitoring — if agent goes silent N minutes, investigate
4. Stream logs to user in real-time via WebSocket
5. Checkpoint after every task completion (not every step — too expensive)
6. Use cheaper model for task decomposition, expensive model for execution
7. Version-lock tool definitions — agent shouldn't discover API changes mid-run
8. Implement kill switch: user can pause/stop/resume any running agent
9. Test on short goals first (5 min), then medium (30 min), then long (hours)
10. Track cost-per-goal-completed as the key metric — not per-task
