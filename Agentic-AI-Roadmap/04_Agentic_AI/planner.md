# Planning in AI Agents

> Author: **Tamilselvan** · ✉️ tamilselvan.sde@gmail.com · 🔗 [LinkedIn](https://www.linkedin.com/in/tamilselvan-ai/)
>

## 1. What is it?
Planning in AI agents refers to the reasoning process of decomposing a goal into a sequence of actions. It transforms "what needs to be done" into "how to do it step by step." Major paradigms include ReAct (Reasoning + Acting), Plan-and-Execute (hierarchical decomposition), and Tree-of-Thought (parallel exploration of reasoning paths).

## 2. Why do we need it?
Without planning, agents react blindly — they pick the first tool that looks applicable and often waste steps or get stuck. Planning provides structure: it reduces token waste, allows error recovery at the plan level (not just step level), and enables agents to handle complex goals that require dozens of coordinated actions.

## 3. Real-world Example
**Google DeepMind's AlphaGo**: AlphaGo uses Monte Carlo Tree Search (MCTS) to plan board game moves. It simulates thousands of possible futures, evaluates each branch, and selects optimal play. For LLM agents, ReAct planning works similarly — the agent thinks about what to do, acts, observes the result, and re-plans.

## 4. Architecture Diagram (ASCII)
```
                    GOAL
                     |
                     v
+---------------------------------------------------+
|                  PLANNER                            |
|                                                      |
|  +---------------+  +----------------------------+  |
|  | Decomposer    |  | Strategy Selector          |  |
|  | (break goal   |  | (choose: ReAct / PlanEx / |  |
|  |  into steps)  |  |  ToT / MCTS)              |  |
|  +---------------+  +----------------------------+  |
|         |                      |                     |
|         v                      v                     |
|  +------------------------------------------------+ |
|  |              Plan Execution Engine              | |
|  |  Step 1 -> Step 2 -> Step 3 -> ... -> Step N   | |
|  +------------------------------------------------+ |
|         |                      |                     |
|         v                      v                     |
|  +----------+          +--------------+             |
|  | Executor |          | Re-planner   |             |
|  | (act)    |          | (on failure) |             |
|  +----------+          +--------------+             |
+---------------------------------------------------+
```

## 5. Internal Working
- **ReAct**: Interleaves reasoning traces with actions. The LLM outputs "Thought: I need to search for X" then "Action: search[X]" then "Observation: result" then "Thought: based on this, I should..."
- **Plan-and-Execute**: Two agents. Planner creates a high-level plan (list of steps). Executor handles each step, which may itself invoke ReAct. On step failure, planner revises remaining steps.
- **Tree-of-Thought**: At each decision point, generates K candidate thoughts, evaluates each, continues with best B (beam search). Explores multiple reasoning paths in parallel.

## 6. Production Flow
```
Goal -> Validate Goal -> Select Planning Strategy
    -> Decompose into Steps
    -> For each Step:
        -> Execute Step (ReAct loop)
        -> Evaluate Result
        -> If Failure: Re-plan remaining steps
    -> Combine results -> Return
```

## 7. HLD
```
+------------------+
|   Goal Queue      |
|  (Redis Stream)   |
+------------------+
         |
         v
+---------------------------+
|    Planning Orchestrator   |
|  +--------+ +-----------+ |
|  |ReAct   | |PlanEx     | |
|  |Engine  | |Engine     | |
|  +--------+ +-----------+ |
|  +--------+ +-----------+ |
|  |ToT     | |MCTS       | |
|  |Engine  | |Engine     | |
|  +--------+ +-----------+ |
+---------------------------+
    |     |     |     |
    v     v     v     v
+----------------------------+
|  Step Execution Workers     |
|  (K8s job per step group)  |
+----------------------------+
```

## 8. LLD
```python
@dataclass
class PlanStep:
    step_id: str
    description: str
    status: StepStatus = StepStatus.PENDING
    depends_on: list[str] = field(default_factory=list)
    tool_required: str | None = None
    result: Any = None
    error: str | None = None

class Plan(BaseModel):
    plan_id: str = Field(default_factory=lambda: uuid.uuid4().hex)
    goal: str
    steps: list[PlanStep]
    current_step_index: int = 0
    strategy: str  # "react", "plan_exe", "tot", "mcts"

class HierarchicalPlanner:
    def __init__(self, llm, max_depth=5):
        self.llm = llm
        self.max_depth = max_depth

    async def create_plan(self, goal: str, context: dict) -> Plan:
        prompt = f"""Given the goal: {goal}
        And context: {json.dumps(context)}
        Create a step-by-step plan. Each step should be:
        - atomic (one action)
        - actionable (maps to a tool)
        - ordered by dependency
        
        Format as JSON list of {{"description": ..., "tool_required": ...}}"""
        
        response = await self.llm.complete(prompt, response_format={"type": "json_object"})
        steps = [PlanStep(**s) for s in json.loads(response)]
        return Plan(goal=goal, steps=steps, strategy="plan_exe")
```

## 9. Python Implementation
```python
import json
from openai import AsyncOpenAI
from pydantic import BaseModel, Field
from enum import Enum
import asyncio

class StepStatus(str, Enum):
    PENDING = "pending"
    RUNNING = "running"
    SUCCESS = "success"
    FAILED = "failed"
    SKIPPED = "skipped"

class ReActStep(BaseModel):
    thought: str
    action: str
    action_input: dict
    observation: str | None = None

class ReActPlanner:
    def __init__(self, llm: AsyncOpenAI, tools: dict, model: str = "gpt-4o"):
        self.llm = llm
        self.tools = tools
        self.model = model
        self.max_iterations = 15

    async def run(self, goal: str) -> str:
        messages = [{"role": "system", "content": self._system_prompt()}]
        messages.append({"role": "user", "content": goal})
        
        for i in range(self.max_iterations):
            response = await self.llm.chat.completions.create(
                model=self.model,
                messages=messages,
                tools=self._tool_definitions(),
                tool_choice="auto"
            )
            
            msg = response.choices[0].message
            
            if not msg.tool_calls:
                return msg.content or "Goal achieved."
            
            messages.append(msg)
            
            for tc in msg.tool_calls:
                tool_name = tc.function.name
                args = json.loads(tc.function.arguments)
                
                tool = self.tools.get(tool_name)
                if tool:
                    result = await tool(**args)
                else:
                    result = f"Error: unknown tool {tool_name}"
                
                messages.append({
                    "role": "tool",
                    "tool_call_id": tc.id,
                    "content": str(result)
                })
        
        return "Max iterations reached without achieving goal."

    def _system_prompt(self):
        return """You are a ReAct agent. Follow this pattern exactly:
Thought: reason about what to do next
Action: choose a tool from the available functions
Wait for the observation, then repeat.
When the goal is achieved, respond directly without calling tools."""

    def _tool_definitions(self) -> list:
        return [{
            "type": "function",
            "function": {
                "name": name,
                "description": tool.__doc__,
                "parameters": {"type": "object", "properties": {}}
            }
        } for name, tool in self.tools.items()]


class TreeOfThought:
    def __init__(self, llm, branches=3, depth=5, beam_width=2):
        self.llm = llm
        self.branches = branches
        self.depth = depth
        self.beam_width = beam_width

    async def search(self, problem: str) -> list[str]:
        candidates = [{"path": [], "thought": problem}]

        for level in range(self.depth):
            all_branches = []
            for c in candidates:
                branches = await self._generate_branches(c["thought"])
                for b in branches:
                    evaluated = await self._evaluate(b)
                    all_branches.append({
                        "path": c["path"] + [b],
                        "thought": b,
                        "score": evaluated
                    })
            
            all_branches.sort(key=lambda x: x["score"], reverse=True)
            candidates = all_branches[:self.beam_width]

        return candidates[0]["path"] if candidates else []

    async def _generate_branches(self, thought: str) -> list[str]:
        response = await self.llm.chat.completions.create(
            model="gpt-4o",
            messages=[{"role": "user", "content": 
                f"Given the current reasoning: {thought}\n"
                f"Generate {self.branches} distinct next steps. "
                f"Return as JSON array of strings."}],
            response_format={"type": "json_object"}
        )
        return json.loads(response.choices[0].message.content)

    async def _evaluate(self, thought: str) -> float:
        response = await self.llm.chat.completions.create(
            model="gpt-4o",
            messages=[{"role": "user", "content": 
                f"Rate this reasoning step's promise (0.0 to 1.0): {thought}\n"
                f"Return only a number."}]
        )
        return float(response.choices[0].message.content.strip())
```

## 10. Folder Structure
```
planner/
├── strategies/
│   ├── react.py           # ReAct implementation
│   ├── plan_exe.py        # Plan-and-Execute
│   ├── tree_of_thought.py # ToT implementation
│   └── mcts.py            # Monte Carlo Tree Search
├── core/
│   ├── planner_base.py    # Abstract planner
│   ├── plan.py            # Plan/Step data models
│   └── executor.py        # Step execution engine
├── evaluation/
│   ├── step_evaluator.py  # Judge step success
│   └── re_planner.py      # Revise plan on failure
├── api/
│   └── routes.py          # FastAPI planner endpoints
└── main.py
```

## 11. Configuration
```yaml
planner:
  default_strategy: react
  max_depth: 10
  max_branches: 3
  beam_width: 2
  
react:
  max_iterations: 15
  stop_tokens: ["Final Answer:"]
  
plan_exe:
  max_retries_per_step: 3
  parallelism: True
  
tree_of_thought:
  branches_per_node: 3
  max_depth: 5
  beam_width: 2
  evaluation_prompt: "default"
```

## 12. Flowchart
```
[Goal] -> [Planner]
             |
     +-------v--------+
     |  Strategy?      |
     +--+----+----+----+
        |    |    |
   [ReAct] [P&E] [ToT]
        |    |    |
   +----v----v----v----+
   |   Execute Steps    |
   +-------------------+
        |        |
    [Success]  [Failure]
        |        |
   [Return]  [Re-plan]
```

## 13. Sequence Diagram
```
User     Planner      Executor      Tool_1      Tool_2      LLM
 |          |            |            |           |          |
 |--goal--->|            |            |           |          |
 |          |--create---->|            |           |          |
 |          |  plan       |            |           |          |
 |          |<--steps-----|            |           |          |
 |          |            |            |           |          |
 |          |--step_1---->|--call----->|           |          |
 |          |            |<--result---|           |          |
 |          |            |            |           |          |
 |          |--step_2---->|--call---------------->|          |
 |          |            |<--result---------------|          |
 |          |            |            |           |          |
 |          |<--final ----|            |           |          |
 |<--output-|            |            |           |          |
```

## 14. Pros
- ReAct: Simple, well-understood, easy to debug
- Plan-and-Execute: Graceful error recovery, parallel execution
- Tree-of-Thought: Higher quality for complex reasoning, explores alternatives
- MCTS: Optimal for search/simulation domains

## 15. Cons
- ReAct: Can loop, no global plan view
- Plan-and-Execute: Two LLM calls per step (more latency, more cost)
- Tree-of-Thought: 3-5x more tokens per problem
- MCTS: Computationally expensive for LLM contexts

## 16. Alternatives
- **Chain-of-Thought (CoT)** — simple prompting, no tool execution
- **Self-Consistency** — run CoT N times, majority vote
- **Least-to-Most** — solve subproblems in increasing difficulty
- **LLM Compiler** — joint reasoning + action planning

## 17. Performance Considerations
- ReAct is fastest (~1 LLM call per step)
- Plan-and-Execute adds ~1 extra LLM call for plan creation
- ToT latency = branches × depth × LLM_call_time
- Cache intermediate results for identical subproblems
- Use streaming for step status updates

## 18. Scaling to Millions
- Planner pool: dedicated GPU instances for LLM inference
- Plan caching: hash the goal, reuse plan for identical goals
- Parallel ToT: distribute branch evaluation across workers
- Step queue: Redis Stream with consumer groups per strategy type
- Re-planning rate limit: max 3 re-plans per plan to avoid explosion

## 19. Failure Scenarios
| Scenario | Impact | Mitigation |
|----------|--------|------------|
| Plan too granular | Token waste | Max step limit with merge heuristic |
| Plan too abstract | Missing details | Auto-expand vague steps |
| Cyclic plan | Infinite loop | Cycle detection on step graph |
| Wrong tool chosen | Incorrect result | Validate preconditions per step |
| Plan divergence from goal | Drift | Re-evaluate goal alignment every N steps |

## 20. Security
- Plans cannot include raw code execution — sandboxed
- Planning prompts sanitized to prevent prompt injection
- Tool selection whitelist — agent cannot call unregistered tools
- Plan output validation enforces JSON schema
- Rate limit per-plan: max steps, max tokens, max depth

## 21. Monitoring
```prometheus
planner_plans_created_total{strategy="react|plan_exe|tot"}
planner_steps_executed_total{status="success|failure"}
planner_plan_creation_latency_seconds
planner_step_execution_latency_seconds
planner_replan_count_per_session
planner_branch_factor_distribution
planner_goal_completion_rate
```

## 22. Interview Questions
**Q1**: When would you choose ReAct over Plan-and-Execute? (Simple tasks with clear next action vs complex multi-step with dependencies)

**Q2**: How do you detect and break infinite planning loops? (Similarity threshold on consecutive thoughts, step count limit, diversity bonus)

**Q3**: How does Tree-of-Thought differ from beam search? (ToT uses LLM to generate and evaluate branches; beam search uses fixed scoring function)

## 23. Cheat Sheet
```
ReAct = Think -> Act -> Observe -> Repeat
PlanEx = Plan -> Execute steps -> Re-plan if failed
ToT = Generate K branches -> Evaluate -> Keep best B -> Repeat
Re-plan triggers: step failure, N consecutive failures, goal drift detected
Step dependency = DAG, not linear chain (parallelize independent steps)
```

## 24. Common Mistakes
- Using Plan-and-Execute for trivial 1-step tasks (wasteful)
- Not setting beam width limit → exponential branching
- ReAct without max iterations → infinite loop
- Generating plans that are too vague — each step should map to a tool
- Not re-planning when context changes mid-execution

## 25. Production Best Practices
1. Start with ReAct for simple agents, upgrade to PlanEx for complex
2. Always serialize plan state to Redis for crash recovery
3. Set aggressive step timeouts (30s per step, not per plan)
4. Log the full reasoning trace for debugging
5. Use separate LLM calls for planning vs execution (different temperatures)
6. Implement plan validation — does each step have a clear successor?
7. A/B test strategy selection: route 10% of traffic to ToT, measure quality
