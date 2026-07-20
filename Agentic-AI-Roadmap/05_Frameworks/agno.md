# Agno (Phidata)

## 1. What is it?

Agno (formerly Phidata) is an open-source framework for building multimodal AI agents with memory, knowledge, and tool-use capabilities. It provides a full platform for creating agents that can process text, images, audio, and video; store and retrieve structured memories; access curated knowledge bases; and call external tools. Agno emphasises a **unified data layer** — agents have built-in persistent memory, structured knowledge retrieval, and a rich tool ecosystem.

## 2. Why do we need it?

Most agent frameworks treat memory as a simple chat buffer and knowledge as an afterthought. Agno solves:
- **Structured memory:** Session, user, and agent-level memory stored in PostgreSQL (persistent, queryable)
- **Multimodal agents:** Process and generate text, images, audio, and video through a unified API
- **Knowledge management:** Curated knowledge bases with automatic retrieval augmentation
- **Agent teams:** Coordinated multi-agent systems with shared memory
- **Production infrastructure:** Built-in monitoring, evaluation, and deployment tools

## 3. Real-world Example

**Multimodal customer support agent:** A customer uploads a screenshot of a product defect. The Agno agent: 1) Processes the image with GPT-4 Vision, 2) Retrieves product specs from its knowledge base (PDFs in S3), 3) Checks the customer's order history from the session memory, 4) Calls the inventory API to check warranty status, 5) Generates a response with an attached replacement label. All interactions and memories are persisted in PostgreSQL.

## 4. Architecture Diagram (ASCII)

```
                    +---------------------------+
                    |      Agno Agent           |
                    +---------------------------+
                    |                           |
    +---------------+    +--------------------+-----------------+
    |  Memory       |    |    Knowledge       |    Tools       |
    |  (PostgreSQL) |    |  (Vector / SQL)    |  (API, Code)  |
    +-------+-------+    +---------+----------+-------+-------+
            |                       |                  |
    +-------v-------+     +---------v----------+      |
    |  Session      |     |  Knowledge Base    |      |
    |  Memory       |     |  (PDFs, Web, DB)   |      |
    +---------------+     +--------------------+      |
                                              +------v-------+
                                              |  Tool Runner  |
                                              |  (Sandboxed)  |
                                              +--------------+
```

## 5. Internal Working

Agno's architecture centres on the **Agent** object:

1. **Agent creation:** Define agent with `model` (multimodal LLM), `memory` (PostgreSQL-backed), `knowledge` (vector store + document set), `tools`, and `instructions`.
2. **Execution loop:** User input → tool selection → tool execution → memory update → knowledge retrieval → response generation.
3. **Memory persistence:** After each interaction, the agent saves the conversation, entities extracted, and summary to PostgreSQL.
4. **Knowledge retrieval:** On each query, the agent retrieves relevant chunks from the knowledge base (PDFs, URLs, databases) and appends them to the context.
5. **Agent teams:** Multiple agents share a coordinator agent. The coordinator routes tasks to specialist agents and aggregates results.

## 6. Production Flow

```
User Input → Memory retrieval (session context) →
Knowledge retrieval (relevant documents) →
LLM reasoning (with tools) → Tool execution →
Memory update (save new context) → Response generation
```

## 7. HLD

```
                   +-------------------------+
                   |   API Gateway           |
                   +-----------+-------------+
                               |
                   +-----------+-------------+
                   |   Agno Agent Service    |
                   |   (FastAPI + Workers)   |
                   +-----------+-------------+
                               |
    +--------------------------+--------------------------+
    |                          |                          |
+---v------------+   +---------v----------+   +-----------v----+
| PostgreSQL     |   | Vector Store      |   | LLM Provider   |
| (Memory +      |   | (Pinecone/Qdrant) |   | (OpenAI, etc)  |
|  Knowledge)    |   | + Knowledge Docs   |   +----------------+
+----------------+   +-------------------+
```

## 8. LLD

```
Agent
  ├── model: MultimodalModel (OpenAI, Gemini, Anthropic)
  ├── memory: Memory (PostgresMemory, InMemory)
  ├── knowledge: KnowledgeBase (DocSet, WebSet, VectorDB)
  ├── tools: List[Tool]
  ├── instructions: List[str]
  ├── add_knowledge(documents)
  ├── run(message) -> RunResponse
  ├── stream(message) -> Generator[RunResponse]
  ├── print_response(message)  # CLI pretty print
  └── memory.memories()  # query stored memories

Memory
  ├── create(user_id, session_id)
  ├── add_memory(memory)
  ├── get_memories(query) -> List[Memory]
  └── clear()

KnowledgeBase
  ├── load(recreate=False)
  ├── search(query) -> List[Document]
  └── document_count: int

Tool
  ├── name, description, entrypoint
  └── run(args) -> str
```

## 9. Python Implementation

```python
from agno.agent import Agent, RunResponse
from agno.models.openai import OpenAIChat
from agno.memory import AgentMemory, MemoryRetrieval
from agno.knowledge import AgentKnowledge
from agno.vectordb import PgVector
from agno.tools import Toolkit
from agno.embedder import OpenAIEmbedder
import json

# Knowledge base (vector + PostgreSQL)
vector_db = PgVector(
    table_name="agent_knowledge",
    db_url="postgresql://user:pass@localhost:5532/agno",
    embedder=OpenAIEmbedder(model="text-embedding-3-small"),
)

knowledge = AgentKnowledge(vector_db=vector_db)
knowledge.load_text("Agno is a framework for building multimodal AI agents.")
knowledge.load_text(
    "Agno supports memory, knowledge, tools, and agent teams."
)
knowledge.load_pdf("documentation.pdf")

# Memory with retrieval
memory = AgentMemory(
    db_url="postgresql://user:pass@localhost:5532/agno",
    table="agent_sessions",
    retrieval=MemoryRetriery(
        search_type="hybrid",
        similarity_top_k=5
    ),
    create_user_memories=True,
    update_user_memories_after_run=True,
)

# Tools
class WebSearchTool(Toolkit):
    def __init__(self):
        super().__init__(name="web_search")

    def search(self, query: str) -> str:
        # Implement web search
        return f"Results for: {query}"

# Agent
agent = Agent(
    model=OpenAIChat(model="gpt-4o", temperature=0.3),
    memory=memory,
    knowledge=knowledge,
    tools=[WebSearchTool()],
    instructions=[
        "You are a helpful assistant with knowledge about Agno.",
        "Use your knowledge base to answer questions.",
        "Search the web if the knowledge base doesn't have the answer.",
        "Remember user preferences across sessions."
    ],
    add_history_to_messages=True,
    num_history_responses=5,
    markdown=True,
)

# Run
response: RunResponse = agent.run("What is Agno and how does memory work?")
print(response.content)

# Access memories
memories = agent.memory.get_memories("user preferences")
for m in memories:
    print(f"Memory: {m.memory} (relevance: {m.relevance})")

# Stream
for chunk in agent.stream("Tell me more about Agno tools"):
    print(chunk.content, end="", flush=True)

# Multi-turn with memory
agent.print_response("My name is Alice and I prefer short answers.")
agent.print_response("What is my name?")  # Uses memory
```

### Agent Teams

```python
from agno.agent import Team

# Specialist agents
research_agent = Agent(
    name="Researcher",
    model=OpenAIChat("gpt-4o"),
    tools=[WebSearchTool()],
    instructions=["Research thoroughly."]
)

writer_agent = Agent(
    name="Writer",
    model=OpenAIChat("gpt-4o"),
    instructions=["Write compelling content."]
)

# Team with coordinator
team = Team(
    name="Content Team",
    mode="coordinate",  # or "route" or "collaborate"
    members=[research_agent, writer_agent],
    model=OpenAIChat("gpt-4o"),
    instructions=["Coordinate research and writing tasks."],
    enable_agentic_context=True,
    success_criteria="Produce a well-researched, well-written article."
)

team.print_response("Write an article about AI agent frameworks.")
```

## 10. Folder Structure

```
agno-app/
├── agents/
│   ├── __init__.py
│   ├── customer_support_agent.py
│   ├── research_agent.py
│   └── teams/
│       ├── __init__.py
│       └── content_team.py
├── knowledge/
│   ├── __init__.py
│   ├── docs/
│   │   ├── product_manual.pdf
│   │   └── faq.csv
│   └── load_knowledge.py
├── memory/
│   └── schema.py
├── tools/
│   ├── __init__.py
│   └── custom_tools.py
├── api/
│   └── main.py
├── tests/
├── docker-compose.yml
├── .env
└── pyproject.toml
```

## 11. Configuration

```python
from pydantic_settings import BaseSettings

class AgnoConfig(BaseSettings):
    # Database
    db_url: str = "postgresql://user:pass@localhost:5532/agno"
    vector_db_url: str = "postgresql://user:pass@localhost:5532/agno"

    # LLM
    openai_api_key: str = ""
    model_id: str = "gpt-4o"

    # Memory
    session_table: str = "agent_sessions"
    memory_table: str = "agent_memories"
    similarity_top_k: int = 5
    num_history_responses: int = 5
    enable_user_memories: bool = True

    # Knowledge
    knowledge_table: str = "agent_knowledge"
    knowledge_recreate: bool = False

    class Config:
        env_file = ".env"

config = AgnoConfig()
```

## 12. Flowchart

```
User Input
    |
    v
Memory Retrieval (session context + user memories)
    |
    v
Knowledge Retrieval (relevant docs from vector DB)
    |
    v
Construct Prompt (system + instructions + context + memories + history)
    |
    v
LLM Reasoning (with tool schemas)
    |
    v
Tool Call? --Yes--> Execute Tool --> Add Result to Context --> Loop
    |
    No
    |
    v
Generate Response
    |
    v
Update Memory (save conversation, extract entities)
    |
    v
Return Response
```

## 13. Sequence Diagram

```
User        Agent       Memory      Knowledge     Tool       LLM
 |           |           |            |            |          |
 |--run()--->|           |            |            |          |
 |           |--query--->|            |            |          |
 |           |<--history--|           |            |          |
 |           |--search-->|            |            |          |
 |           |<--docs----|            |            |          |
 |           |                     |              |          |
 |           |--prompt-------------------------------------->|
 |           |<--tool_call-----------------------------------|
 |           |--exec------|---------|----------->|          |
 |           |<--result---|---------|------------|          |
 |           |--prompt-------------------------------------->|
 |           |<--response-----------------------------------|
 |           |--save----->|         |            |          |
 |<--resp----|           |         |            |          |
```

## 14. Pros

- **Built-in persistent memory** — PostgreSQL-backed, hybrid search, multi-level (session, user, agent)
- **Multimodal by design** — text, image, audio, video through a single Agent API
- **Knowledge management** — auto-embedding, vector search, easy document loading
- **Agent teams** — coordinated multi-agent system with shared context
- **Production infrastructure** — monitoring, evaluation, Docker Compose templates
- **Clean API** — fewer concepts than LangChain, easier to get started

## 15. Cons

- **Smaller community** — newer and less adopted than major frameworks
- **PostgreSQL dependency** — requires PostgreSQL for memory (not truly lightweight)
- **Documentation gaps** — some advanced features lack comprehensive docs
- **Model support** — fewer model providers than LangChain
- **API churn** — renamed from Phidata, API still evolving

## 16. Alternatives

| Framework | When to Choose |
|-----------|---------------|
| LangChain | Largest ecosystem, model-agnostic |
| LangGraph | Stateful graphical orchestrat ion |
| Pydantic AI | Type safety, validation-driven |
| CrewAI | Role-based agent teams |

## 17. Performance Considerations

- **Memory queries:** PostgreSQL-backed memory with hybrid search. Query time ~10-50ms depending on index size.
- **Knowledge retrieval:** Vector search in PgVector. 100k documents ~50ms query time.
- **Agent team overhead:** The coordinator LLM call adds latency. Use `mode="route"` for simple routing, `mode="coordinate"` for complex tasks.
- **Multimodal processing:** Image processing can add 1-5s latency. Use smaller images (resize before sending).
- **Memory growth:** Long-running sessions accumulate memory. Set `num_history_responses` to limit context.

## 18. Scaling to Millions

1. **PostgreSQL connection pooling:** Use PgBond (or PgCat) for connection pooling. Max connections scale with pod count.
2. **Vector index tuning:** PgVector supports HNSW and IVFFlat indexes. HNSW is faster for large datasets (>100k).
3. **Memory sharding:** Partition memory tables by `user_id`. Use Citus or CockroachDB for distributed PostgreSQL.
4. **Knowledge base sharding:** Split knowledge bases by domain. Route queries to the correct knowledge base.
5. **Agent worker pool:** Pre-warm agents in a worker pool. Assign agents to user sessions.
6. **Caching:** Cache knowledge retrieval results for identical queries. Use Redis with TTL.

## 19. Failure Scenarios

1. **PostgreSQL outage:** Memory and knowledge become unavailable. Agent falls back to no-memory/ no-knowledge mode.
2. **Embedding failure:** Knowledge search fails. Agent uses only conversation history.
3. **Tool error:** Error message passed back to the LLM. The LLM retries or apologises.
4. **Memory corruption:** `clear()` resets session memory. User memories survive session resets.
5. **LLM timeout:** Agent retries with exponential backoff. Configure `retry_attempts` on the model.

## 20. Security

- **Memory isolation:** Memories are scoped to `user_id` and `session_id`. Cross-user memory access is prevented at the database level.
- **Tool sandboxing:** Tools run in-process by default. For custom tools, use Docker containers.
- **Knowledge base access control:** Control read/write access to knowledge bases via API keys.
- **PII in memories:** Use `exclude_memory_keywords` to prevent certain patterns from being stored as memories.
- **Database encryption:** Encrypt memory and knowledge tables at rest (PostgreSQL TDE or column-level encryption).

## 21. Monitoring

- **Agno dashboard** (coming): Built-in dashboard for agent runs, memory usage, knowledge queries.
- **Custom metrics:** Track `agent_run_count`, `memory_query_latency`, `knowledge_retrieval_recall`, `tool_success_rate`.
- **Logging:** Structured JSON logs per agent run. Include `session_id`, `user_id`, `agent_name`, `duration_ms`.
- **Traceability:** Agno integrates with OpenTelemetry for distributed tracing.

## 22. Interview Questions

1. How does Agno's memory system differ from LangChain's memory?
2. Explain the knowledge base architecture — how are documents stored and retrieved?
3. How would you implement a multimodal agent that processes images and text?
4. Compare Agno's agent teams with CrewAI's crew model.
5. How does Agno handle memory persistence across sessions?
6. Design a customer support agent with Agno that escalates to humans.
7. How would you implement user-specific knowledge bases in Agno?
8. Explain the trade-offs of the three team modes: coordinate, route, collaborate.

## 23. Cheat Sheet

| Task | Code |
|------|------|
| Create agent | `Agent(model=OpenAIChat("gpt-4o"), memory=memory, knowledge=knowledge)` |
| Run agent | `agent.run("question")` |
| Stream agent | `agent.stream("question")` |
| Add knowledge text | `knowledge.load_text("Content")` |
| Add knowledge PDF | `knowledge.load_pdf("file.pdf")` |
| Query memories | `agent.memory.get_memories("query")` |
| Create team | `Team(name="T", mode="coordinate", members=[a1, a2])` |
| Pretty print | `agent.print_response("question")` |

## History

Agno (formerly Phidata) was created by a team led by Ashpreet Bedi (ex-Google, ex-Amazon) to address the infrastructure gaps in agent frameworks. Initially launched as Phidata in 2023, the framework focused on data-centric agent workflows. In 2024, it was rebranded to Agno (Greek for "knowledge") with a major architecture overhaul centred on memory, knowledge, and multimodality. The framework gained traction for its production-ready memory system (PostgreSQL-backed, hybrid search) and was adopted by several startups for customer service automation and research assistant applications. Agno continues to evolve with a focus on making production-grade multimodal agents accessible to Python developers.

## Core Components Explained

- **Agent:** The core class. Configured with a model, memory, knowledge, tools, and instructions. Handles the full execution loop.
- **Memory:** Persistent storage for conversation history, extracted entities, and user preferences. Backed by PostgreSQL with vector/hybrid search.
- **Knowledge:** Curated knowledge base with auto-embedding and vector search. Supports text, PDFs, URLs, CSV, and databases.
- **Tools:** Capabilities the agent can use. Agno includes built-in tools (web search, code execution, file operations) and supports custom tools.
- **Agent Teams:** Coordinator + specialist agents. Three modes: `coordinate` (delegate), `route` (classify + route), `collaborate` (discuss + agree).
- **Models:** Multimodal LLM wrappers. Support OpenAI, Gemini, Anthropic, and local models (Ollama, vLLM).
- **KnowledgeBase:** Multiple implementations — `DocSet` (documents), `WebSet` (web pages), ` PDFKnowledgeBase`, `CSVKnowledgeBase`.
- **AgentMemory:** Memory types — `SessionMemory` (conversation), `UserMemory` (user preferences), `EntityMemory` (extracted entities).

## Installation

```bash
pip install agno
# With PostgreSQL (required for persistent memory)
pip install agno[postgres]
# With vector support
pip install agno[vector]
# Full
pip install agno[all]
# Docker for PostgreSQL
docker run -d --name agno-postgres -e POSTGRES_USER=user \
  -e POSTGRES_PASSWORD=pass -e POSTGRES_DB=agno \
  -p 5532:5432 postgres:16
```

## Advanced Features

- **Hybrid memory search:** Combines vector similarity with keyword matching for memory retrieval.
- **User memory extraction:** Automatically extracts user facts (name, preferences, facts) from conversations and persists them.
- **Multimodal input/output:** Process images directly in the agent. Generate images via DALL-E integration.
- **Agent evaluation:** Built-in evaluation tools. Run test suites against agents with golden answers.
- **Custom knowledge base:** Implement custom document loaders by subclassing `KnowledgeBase`.
- **Model routing:** Route queries to different models based on complexity.
- **Async execution:** Full async support for concurrent agent runs.
- **Streaming tools:** Display tool execution in real-time in the CLI.
- **Feedback loops:** Collect user feedback on responses for fine-tuning.
- **Session management:** Create, persist, resume sessions with full context.
- **Toolkits:** Pre-built tool collections (web, code, file, database, API).
- **Knowledge versioning:** Track document versions and update embeddings incrementally.

## Performance Tuning

1. **Reduce memory size:** Set `num_history_responses` to 3-5 to limit context. For long sessions, use summary memory.
2. **Vector index tuning:** Use HNSW index with `m=16` and `ef_construction=200` for PgVector. Queries are 10x faster than IVFFlat for >100k records.
3. **Knowledge chunking:** Smaller chunks (256 tokens) improve retrieval precision but increase storage. Start with 512 tokens.
4. **Connection pooling:** Use `psycopg2.pool.ThreadedConnectionPool` for PostgreSQL. Pool size = 2x number of gunicorn workers.
5. **Caching:** Implement a Redis cache layer for knowledge retrieval. Cache key = `knowledge_base_id + query_hash`.

## Debugging Tips

1. Set `agent.print_response("...", markdown=True)` for beautiful CLI output with tool call logs.
2. `agent.memory.get_memories("all")` dumps all stored memories — useful for debugging memory corruption.
3. Enable debug mode: `agent.run("...", debug_mode=True)` prints raw LLM prompts and responses.
4. Check `response.tools_used` to see which tools were called.
5. Common errors: `ModuleNotFoundError: psycopg2` (PostgreSQL driver not installed), `ConnectionError` (PostgreSQL not running).

## Production Deployment

1. **Docker Compose:** Agno provides a `docker-compose.yml` with PostgreSQL + application container.
2. **Database setup:** Run `python -m agno.migrate` to create memory and knowledge tables.
3. **Scaling:** Stateless agent pods behind a load balancer. PostgreSQL as the single source of truth.
4. **Knowledge base updates:** Run knowledge loading as a separate cron job. Do not load knowledge in the request path.
5. **Monitoring:** Prometheus + Grafana for system metrics. Structured logging to stdout for log aggregation.
6. **Secrets management:** Use Docker secrets or HashiCorp Vault for API keys and database credentials.
