# CrewAI Framework

## 1. What is it?

CrewAI is an open-source multi-agent orchestration framework created by João Moura in 2023. It enables developers to define AI agents with specific roles, goals, backstories, and tools, then assemble them into "crews" that collaborate to accomplish complex tasks. Agents communicate, delegate, and work in sequence or parallel, mimicking human team dynamics. CrewAI has gained rapid adoption for its intuitive role-based abstraction and built-in support for sequential and hierarchical workflows.

## 2. Why do we need it?

Single-agent systems hit fundamental limitations on complex work:
- **Specialisation:** One LLM cannot excel at research, analysis, writing, and code generation simultaneously
- **Parallel work:** Agents must work concurrently on independent sub-tasks
- **Delegation:** An agent needs to hand off work to specialised peers
- **Validation:** One agent should review another's output for quality
- **Complex workflows:** Tasks with dependencies require coordination

CrewAI provides a **team metaphor** — agents with roles, crews with processes, tasks with dependencies — making multi-agent orchestration intuitive. Without it, teams would hand-roll complex async coordination logic.

## 3. Real-world Example

**Automated market research report generation:**
1. **Researcher Agent:** Searches the web for competitor data and market trends
2. **Data Analyst Agent:** Analyses collected data, creates charts, extracts patterns
3. **Writer Agent:** Synthesises findings into a structured report
4. **Reviewer Agent:** Checks quality, consistency, and completeness
5. **Manager Agent** (hierarchical): Coordinates work, delegates sub-tasks, resolves conflicts

The crew runs asynchronously, producing a polished executive report in under 5 minutes.

## 4. Architecture Diagram (ASCII)

```
                         CREW
    ┌──────────────────────────────────────────────────┐
    │  +----------------+  +-------------------+       │
    │  |  Agent A       |  |  Agent B          |       │
    │  |  Role: Researcher|  |  Role: Analyst   |       │
    │  |  Goal: Gather  |  |  Goal: Analyze   |       │
    │  |  Tools: [Search]|  |  Tools: [Python] |       │
    │  +-------+--------+  +--------+----------+       │
    │          |                     |                  │
    │  +-------v--------------------v----------+       │
    │  |           Task Pipeline               |       │
    │  |  Task1 -> Task2 -> Task3 -> Task4     |       │
    │  |  (Researcher) (Analyst) (Writer) (Reviewer)   │
    │  +--------------------------------------+       │
    │                                                  │
    │  Process: Sequential or Hierarchical             │
    └──────────────────────────────────────────────────┘
```

## 5. Internal Working

CrewAI's execution model has four layers:

1. **Agent Definition:** Each agent is a `BaseAgent` subclass with `role`, `goal`, `backstory`, `tools`, `llm`, `allow_delegation`, and `verbose` properties. Agents use their assigned LLM for all decisions and tool calls.
2. **Task Definition:** A `Task` has a `description`, `expected_output`, `agent` assignment, `context` (dependencies on other tasks), and optional `output_file` and `callback`. Tasks are not functions — they are executed by their assigned agent.
3. **Crew Assembly:** A `Crew` collects agents and tasks. It defines the `process` (sequential or hierarchical), `verbose` mode, `memory` (entity and short-term), `cache`, and `max_rpm`.
4. **Execution:**
   - **Sequential:** Tasks execute in order. Each task invokes its assigned agent, which uses its tools to produce output. Task output is passed as context to dependent tasks.
   - **Hierarchical:** A manager agent is automatically created. The manager decomposes work, assigns tasks to worker agents, monitors progress, and aggregates results.

## 6. Production Flow

```
Define Agents → Define Tasks → Create Crew → Kickoff
                                              │
                                     Sequential or Hierarchical
                                              │
                                     Task Execution Loop
                                              │
                                     +--------+--------+
                                     |   Agent Decides  |
                                     +--------+--------+
                                              |
                                     +--------+--------+
                                     |   Tool Call(s)  |
                                     +--------+--------+
                                              |
                                     +--------+--------+
                                     |   Task Complete  |
                                     +--------+--------+
                                              |
                                     Next Task or Done
                                              |
                                     Return Final Output
```

## 7. HLD

```
                   +-----------------------+
                   |   API Gateway         |
                   +----------+------------+
                              |
                   +----------+------------+
                   |   Crew Service        |
                   |   (Crew Execution)    |
                   +----------+------------+
                              |
              +---------------+---------------+
              |               |               |
      +-------v----+   +-----v------+  +-----v------+
      |  Agent     |   |  Agent     |  |  Agent     |
      |  (LLM)     |   |  (LLM)     |  |  (LLM)     |
      +-----+------+   +-----+------+  +-----+------+
            |                 |                |
      +-----v------+   +-----v------+  +-----v------+
      |  Tools     |   |  Tools     |  |  Tools     |
      | (Sandbox)  |   | (Sandbox)  |  | (Sandbox)  |
      +------------+   +------------+  +------------+

    +----------------+       +----------------+
    | Memory Store   |       | Cache Store    |
    | (Redis/DB)     |       | (Redis)        |
    +----------------+       +----------------+
```

## 8. LLD

```
BaseAgent
  ├── role: str
  ├── goal: str
  ├── backstory: str
  ├── tools: List[BaseTool]
  ├── llm: BaseLLM
  ├── allow_delegation: bool
  ├── verbose: bool
  └── max_iter: int

BaseTask
  ├── description: str
  ├── expected_output: str
  ├── agent: BaseAgent
  ├── context: List[BaseTask]   # dependencies
  ├── async_exec: bool
  └── callback: Callable

Crew
  ├── agents: List[BaseAgent]
  ├── tasks: List[BaseTask]
  ├── process: Sequential | Hierarchical
  ├── memory: bool
  ├── cache: bool
  ├── max_rpm: int
  ├── share_crew: bool  # share memory across tasks
  └── kickoff() -> CrewOutput

BaseTool
  ├── name: str
  ├── description: str
  ├── args_schema: Type[BaseModel]
  └── _run(*args, **kwargs)
```

## 9. Python Implementation

```python
from crewai import Agent, Task, Crew, Process
from crewai_tools import SerperDevTool, ScrapeWebsiteTool, FileWriterTool
from langchain_openai import ChatOpenAI

# LLMs
manager_llm = ChatOpenAI(model="gpt-4o", temperature=0)
worker_llm = ChatOpenAI(model="gpt-4o-mini", temperature=0.3)

# Tools
search_tool = SerperDevTool()
scrape_tool = ScrapeWebsiteTool()
write_tool = FileWriterTool()

# Agents
researcher = Agent(
    role="Senior Market Researcher",
    goal="Find comprehensive market data and competitive intelligence",
    backstory="Expert researcher with 15 years in tech market analysis",
    tools=[search_tool, scrape_tool],
    llm=worker_llm,
    allow_delegation=False,
    verbose=True,
    max_iter=10
)

analyst = Agent(
    role="Data Analyst",
    goal="Transform raw data into actionable insights",
    backstory="Data scientist specialised in market analysis",
    tools=[search_tool],
    llm=worker_llm,
    allow_delegation=False,
    verbose=True,
    max_iter=10
)

writer = Agent(
    role="Executive Report Writer",
    goal="Create compelling, well-structured reports",
    backstory="Former McKinsey consultant with exceptional writing skills",
    tools=[write_tool],
    llm=worker_llm,
    allow_delegation=False,
    verbose=True,
    max_iter=5
)

# Tasks
research_task = Task(
    description="Research the top 5 competitors in the AI coding assistant market. "
                "Include their funding, market share, key features, and pricing.",
    expected_output="Comprehensive competitor analysis with structured data",
    agent=researcher
)

analysis_task = Task(
    description="Analyse the competitive landscape data. Identify gaps, opportunities, "
                "and threats. Create a SWOT analysis.",
    expected_output="SWOT analysis and opportunity matrix",
    agent=analyst,
    context=[research_task]
)

report_task = Task(
    description="Write a 3-page executive report synthesising all findings. "
                "Include executive summary, competitive analysis, SWOT, and recommendations.",
    expected_output="Professional markdown report saved to file",
    agent=writer,
    context=[research_task, analysis_task],
    output_file="market_report.md"
)

# Crew
crew = Crew(
    agents=[researcher, analyst, writer],
    tasks=[research_task, analysis_task, report_task],
    process=Process.sequential,
    verbose=True,
    memory=True,
    cache=True,
    max_rpm=20,
    share_crew=True
)

result = crew.kickoff()
print(f"Report written to market_report.md")
print(f"Usage: {result.token_usage}")
```

## 10. Folder Structure

```
crewai-app/
├── src/
│   ├── crew/
│   │   ├── __init__.py
│   │   ├── agents.py          # Agent definitions
│   │   ├── tasks.py           # Task definitions
│   │   ├── crew.py            # Crew assembly
│   │   └── tools/
│   │       ├── __init__.py
│   │       ├── search_tools.py
│   │       ├── analysis_tools.py
│   │       └── custom_tool.py
│   ├── main.py                # Entry point (CLI or API)
│   ├── api.py                 # FastAPI server
│   └── config.py
├── outputs/                   # Generated files
├── tests/
│   ├── test_agents.py
│   ├── test_tasks.py
│   └── test_crew.py
├── pyproject.toml
├── Dockerfile
└── docker-compose.yml
```

## 11. Configuration

```python
from pydantic_settings import BaseSettings

class CrewAIConfig(BaseSettings):
    verbose: bool = True
    memory: bool = True
    cache: bool = True
    max_rpm: int = 10
    manager_llm_model: str = "gpt-4o"
    worker_llm_model: str = "gpt-4o-mini"
    embedder_provider: str = "openai"
    embedder_model: str = "text-embedding-3-small"
    cache_backend: str = "redis"
    redis_url: str = "redis://localhost:6379"

    class Config:
        env_prefix = "CREWAI_"
        env_file = ".env"
```

## 12. Flowchart

```
       Kickoff
          |
    +-----v------+
    | Process?   |
    +-----+------+
          |
    +-----+------+----+
    |            |    |
Sequential  Hierarchical
    |            |
    v            v
Task1 ->    Manager
  Agent1     creates
    |        sub-tasks
    v            |
Task2 ->      Worker1
  Agent2         |
    |          Worker2
    v            |
Task3 ->      Results
  Agent3      merged
    |            |
    v            v
  Output      Output
```

## 13. Sequence Diagram

```
User      Crew       Manager     Worker1     Worker2     Tool
 |         |           |           |           |          |
 |kickoff->|           |           |           |          |
 |         |--plan---->|           |           |          |
 |         |           |--assign-->|           |          |
 |         |           |           |--tool---->|          |
 |         |           |           |<--result--|          |
 |         |           |<--output--|           |          |
 |         |           |--assign-->|           |          |
 |         |           |           |--tool---->|          |
 |         |           |<--output--|           |          |
 |         |<--result--|           |           |          |
 |<--resp--|           |           |           |          |
```

## 14. Pros

- **Intuitive role-based model** — easy for non-engineers to understand
- **Built-in delegation and process management** — no manual orchestration code
- **CrewAI Enterprise** — managed hosting, monitoring, audit logs
- **Active community** — 20k+ GitHub stars, frequent releases
- **Integration with LangChain tools** — use existing LangChain tool ecosystem
- **Memory and caching built-in** — entity memory, short-term memory, Redis cache

## 15. Cons

- **Limited flexibility** — hard to express custom graph-based orchestration (LangGraph is better)
- **Hierarchical overhead** — manager LLM call adds latency and cost for every delegation
- **Smaller tool ecosystem** — fewer native tools than LangChain
- **Debugging complexity** — multi-agent traces are harder to parse than single-agent
- **State management** — basic compared to LangGraph's checkpointer system

## 16. Alternatives

| Framework | When to Choose |
|-----------|---------------|
| AutoGen | Deep conversational multi-agent, Microsoft ecosystem |
| LangGraph | Stateful, cyclic graph orchestration |
| Semantic Kernel | .NET/enterprise, Azure ecosystem |
| OpenAI Assistants API | Managed agents, no infra |
| Microsoft Copilot Studio | No-code agent builder |

## 17. Performance Considerations

- **LLM call overhead:** Each agent decision = one LLM call. For complex tasks, this adds up. Use `gpt-4o-mini` for worker agents, reserve `gpt-4o` for the manager.
- **max_rpm:** Set `max_rpm` to your API tier limit. Higher values risk rate limiting.
- **Parallel task execution:** Tasks with no dependencies run concurrently. Use `async_exec=True` for independent tasks.
- **Memory scaling:** Entity memory grows with conversation length. Use `share_crew=False` for independent crews to limit memory bloat.
- **Cache hit rate:** Tool call caching reduces redundant LLM+tool calls. Monitor cache hit rate; if low, review task descriptions for uniqueness.

## 18. Scaling to Millions

1. **Per-user crews:** Each user session gets its own Crew instance. Stateless crew definitions, stateful memory in Redis.
2. **Worker pools:** Pre-initialise agent/task definitions. Use a job queue (Celery, RabbitMQ) to distribute kickoff requests.
3. **Manager as a service:** The manager agent can be a separate, higher-capacity service. Worker agents can be lightweight, cheaper models.
4. **Caching:** Redis-backed caching for tool outputs. TTL-based cache invalidation.
5. **Result storage:** Crew outputs stored in PostgreSQL. Audit trail for every task execution.
6. **Monitoring aggregation:** All crew metrics pushed to a central monitoring service (DataDog, Prometheus).

## 19. Failure Scenarios

1. **Agent loops:** `max_iter` prevents infinite loops. If hit, the agent returns "I cannot complete this task" rather than looping forever.
2. **Tool failures:** Failed tool calls return error messages to the LLM. The LLM can retry or adapt. For persistent failures, the task is marked failed.
3. **Rate limiting:** CrewAI throttles internally with `max_rpm`. If exceeded, tasks queue until tokens replenish.
4. **LLM context overflow:** Long task outputs exceed the LLM's context window. Use `context_window` parameter to truncate. Summarise long inputs.
5. **Agent hallucination:** Tools ground the agent in reality. For tools output, the agent cannot easily hallucinate. Review the expected_output in task definitions.

## 20. Security

- **Tool sandboxing:** Use `crewai_tools` with restricted execution. For custom tools, run in Docker containers with no network access.
- **PII in memory:** CrewAI's entity memory stores entities extracted from conversations. Implement PII filtering before memory write.
- **Agent credentials:** Each agent uses its LLM's API key. Use separate keys for different agents to enforce rate limits and billing separation.
- **Task injection:** Never include raw user input in task descriptions without sanitisation. Validate task outputs before using them in subsequent tasks.
- **Audit trail:** Log every tool call, LLM response, and task completion. CrewAI Enterprise provides built-in audit logging.

## 21. Monitoring

- **CrewAI Enterprise:** Built-in dashboard for crew executions, agent activity, token usage, and errors.
- **Self-hosted metrics:**
  - Crew completion rate (% successful kickoffs)
  - Agent-level latency and token usage
  - Tool call success/failure rate
  - Memory usage per crew
  - Cache hit ratio
- **Logging:** Structured JSON logs per agent step. Include `crew_id`, `agent_role`, `task_id`, `duration_ms`.

## 22. Interview Questions

1. Explain the difference between sequential and hierarchical processes in CrewAI.
2. How does agent delegation work and when would you use it?
3. Design a multi-agent crew for customer support with escalation.
4. How does CrewAI's memory system work? What types of memory are available?
5. Compare CrewAI's multi-agent model with AutoGen's conversational model.
6. How would you implement a custom tool in CrewAI?
7. What are the trade-offs of using hierarchical vs. sequential process?
8. How does CrewAI handle task dependencies and context passing?

## 23. Cheat Sheet

| Task | Code |
|------|------|
| Create agent | `Agent(role="X", goal="Y", backstory="Z", tools=[t])` |
| Create task | `Task(description="...", expected_output="...", agent=a)` |
| Sequential crew | `Crew(agents=[a,b], tasks=[t1,t2], process=Process.sequential)` |
| Hierarchical crew | `Crew(agents=[a,b], tasks=[t1,t2], process=Process.hierarchical)` |
| Task with context | `Task(..., context=[prior_task])` |
| Async task | `Task(..., async_exec=True)` |
| Output to file | `Task(..., output_file="path.md")` |
| Kickoff | `crew.kickoff()` returns `CrewOutput` |

## History

CrewAI was created by João Moura in 2023 as a response to the complexity of existing multi-agent systems. The first public version was released in early 2024 and quickly gained traction for its developer-friendly role-based metaphor. Version 0.1 supported basic sequential agents. Hierarchical process was added in mid-2024. Memory and caching systems were introduced in v0.3. CrewAI Enterprise launched in late 2024 with managed hosting, monitoring, and team collaboration features. By 2025, CrewAI has over 20k GitHub stars, a growing ecosystem of tools, and is widely adopted in startups and enterprises for automating research, content generation, and business analysis workflows.

## Core Components Explained

- **Agent:** The fundamental building block. Each agent has a `role` (job title), `goal` (objective), `backstory` (personality context), and `tools` (capabilities). Agents use a specific LLM and can be configured to delegate tasks to other agents.
- **Task:** A unit of work assigned to an agent. Tasks can have `context` (dependencies on other tasks), `async_exec` (for independent parallel execution), and `callback` (trigger on completion).
- **Crew:** The orchestration container. A crew defines which agents participate and how tasks are executed. The `process` attribute determines sequential vs. hierarchical execution.
- **Process:** Sequential (agents execute their tasks in order, one after another) or Hierarchical (a manager agent decomposes and delegates work to worker agents, then aggregates results).
- **Tool:** A capability that an agent can use. CrewAI natively supports LangChain tools. Custom tools are defined with the `@tool` decorator.

## Installation

```bash
pip install crewai crewai-tools
# Optional integrations
pip install langchain-openai
# Development
pip install crewai[dev]
```

## Advanced Features

- **Crew memory:** Entity memory (remembers facts about entities mentioned), short-term memory (last few interactions), user memory (per-user context).
- **Caching:** In-memory or Redis-based caching of tool and LLM responses. Reduces cost and latency for repeated queries.
- **Callbacks:** `on_task_start`, `on_task_end`, `on_agent_start`, `on_agent_end` callbacks for custom logging and side effects.
- **Custom tools:** Implement `BaseTool` or use `@tool` decorator to wrap any Python function.
- **Output parsing:** CrewAI supports Pydantic output parsing for structured task outputs.
- **Step callback:** Per-step callback for streaming updates to the UI.

## Performance Tuning

1. **Model tiering:** Use `gpt-4o-mini` for high-volume worker agents. Reserve `gpt-4o` or `claude-opus-4` for the manager and critical tasks.
2. **Reduce max_iter:** Set `max_iter=3-5` for simple tasks. Higher values give the agent more chances to refine but increase latency and cost.
3. **Tool selection:** Give agents only the tools they need. Too many tools confuse the agent and increase LLM token usage (tool descriptions are included in the prompt).
4. **Cache warming:** Pre-warm the cache with common queries during deployment.

## Debugging Tips

1. Set `verbose=True` on the Crew — prints every agent thought, action, and observation.
2. Use `crew.kickoff_for_each()` to inspect intermediate results.
3. Enable `crew.share_crew=False` for isolation debugging — prevents memory cross-contamination between tasks.
4. Inspect `result.token_usage` to identify expensive tasks.

## Production Deployment

1. **Containerise:** Docker container with the crew application. Use `gunicorn` + `uvicorn` for API workers.
2. **Rate limiting:** Set `max_rpm` according to your API plan. CrewAI Enterprise handles automatic rate limiting.
3. **Persistence:** Store crew outputs in PostgreSQL. Redis for cache and memory backend.
4. **Scaling:** Horizontal scaling with Kubernetes. Each pod can run multiple crews depending on `max_rpm`.
5. **Monitoring:** CrewAI Enterprise dashboard. For self-hosted, use Prometheus + Grafana with custom metrics.
