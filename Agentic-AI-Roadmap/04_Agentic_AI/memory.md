# Agent Memory

> Author: **Tamilselvan** · ✉️ tamilselvan.sde@gmail.com · 🔗 [LinkedIn](https://www.linkedin.com/in/tamilselvan-ai/)
>

## 1. What is it?
Agent memory is the system that stores, retrieves, and manages information across an agent's lifecycle. It mirrors human memory: short-term (working memory for current session), long-term (persistent knowledge), and episodic (past experiences and outcomes). RAG-based memory augments this with external knowledge bases.

## 2. Why do we need it?
Without memory, agents are amnesiac — each interaction starts from zero. Memory enables: (1) continuity across turns, (2) learning from past mistakes, (3) personalization per user, (4) knowledge retention beyond LLM context limits, (5) comparison of current state against historical patterns.

## 3. Real-world Example
**Meta's recommendation agent**: Each user session has short-term memory (clicks this session), episodic memory (past sessions' behavior patterns), and long-term memory (user demographics, preferences). The agent retrieves relevant memories to decide what to recommend. RAG augments with product catalog embeddings.

## 4. Architecture Diagram (ASCII)
```
+----------------------------------------------------------+
|                    MEMORY LAYER                            |
|                                                            |
|  +------------------+  +------------------+               |
|  | Short-Term (ST)  |  | Long-Term (LT)   |               |
|  | Redis / Buffer   |  | PostgreSQL       |               |
|  | TTL: 1 hour      |  | Persistent       |               |
|  | Session-scoped   |  | User-scoped      |               |
|  +------------------+  +------------------+               |
|                                                            |
|  +------------------+  +------------------+               |
|  | Episodic         |  | RAG Store        |               |
|  | Vector Store     |  | Vector DB        |               |
|  | Past Experiences |  | Knowledge Base   |               |
|  | Similarity Search|  | Document Chunks  |               |
|  +------------------+  +------------------+               |
|                                                            |
|  +------------------------------------------------------+ |
|  |              Memory Controller                         | |
|  |  - Write: consolidate ST -> LT -> Episodic            | |
|  |  - Read: retrieve from all layers, rank, merge        | |
|  |  - Forget: TTL expiry, importance decay, summarization| |
|  +------------------------------------------------------+ |
+----------------------------------------------------------+
```

## 5. Internal Working
Short-term memory holds the current conversation (sliding window of last N messages, summarized when exceeded). Long-term memory stores user facts, preferences, and learned patterns as key-value pairs or structured records. Episodic memory records past agent runs as vector embeddings — when a similar situation arises, the agent retrieves what worked before. RAG memory chunks documents, embeds them, and retrieves relevant context on demand.

## 6. Production Flow
```
Agent receives input
    -> Memory Controller retrieves relevant context:
        -> ST: last N messages from session buffer
        -> LT: user facts from PostgreSQL
        -> Episodic: top-K similar past sessions from vector DB
        -> RAG: top-K relevant chunks from knowledge base
    -> Merge & rank: recency * importance * relevance score
    -> Inject into LLM context
    -> Agent processes, generates response
    -> Memory Controller writes:
        -> ST: append to session buffer
        -> LT: update facts if new info discovered
        -> Episodic: store experience vector
```

## 7. HLD
```
+------------------+
|   API Gateway     |
+------------------+
         |
         v
+------------------------+
|   Agent Service         |
|  +------------------+  |
|  | Memory Controller |  |
|  +------------------+  |
+------------------------+
    |     |     |     |
    v     v     v     v
+------+ +------+ +------+ +------+
| Redis | | PG   | |Vector| |Obj   |
| (ST)  | | (LT) | | (Epi)| |Store |
+------+ +------+ +------+ +------+
```

## 8. LLD
```python
from pydantic import BaseModel, Field
from datetime import datetime, timedelta
from typing import Any
import numpy as np
import json

class Memory(BaseModel):
    memory_id: str
    agent_id: str
    session_id: str
    user_id: str
    content: str
    memory_type: str  # "short_term", "long_term", "episodic", "rag"
    importance: float = 0.5  # 0.0 to 1.0
    embedding: list[float] | None = None
    metadata: dict[str, Any] = Field(default_factory=dict)
    created_at: datetime = Field(default_factory=datetime.utcnow)
    expires_at: datetime | None = None
    access_count: int = 0
    last_accessed: datetime = Field(default_factory=datetime.utcnow)

class MemoryController:
    def __init__(self, redis_client, pg_pool, vector_client):
        self.redis = redis_client
        self.pg = pg_pool
        self.vector = vector_client
        self.max_short_term = 20  # messages
        self.consolidation_interval = timedelta(minutes=30)

    async def retrieve(self, agent_id: str, session_id: str, user_id: str, query: str) -> str:
        st = await self._get_short_term(session_id)
        lt = await self._get_long_term(user_id)
        epi = await self._search_episodic(query, user_id, k=3)
        rag = await self._search_rag(query, k=5)
        
        memories = self._rank_merge(st, lt, epi, rag, query)
        return self._format_context(memories)

    async def store(self, memory: Memory):
        memory.last_accessed = datetime.utcnow()
        
        if memory.memory_type == "short_term":
            await self._store_short_term(memory)
            if memory.importance > 0.7:
                await self._consolidate_to_long_term(memory)
        elif memory.memory_type == "long_term":
            await self._store_long_term(memory)
        elif memory.memory_type == "episodic":
            await self._store_episodic(memory)
        
        if self._should_consolidate():
            await self._consolidate()

    async def _get_short_term(self, session_id: str) -> list[Memory]:
        raw = await self.redis.lrange(f"st:{session_id}", 0, self.max_short_term - 1)
        return [Memory(**json.loads(m)) for m in raw]

    async def _search_episodic(self, query: str, user_id: str, k: int) -> list[Memory]:
        q_emb = await self._embed(query)
        results = await self.vector.search(
            collection=f"episodic:{user_id}",
            query_vector=q_emb,
            limit=k
        )
        return [Memory(**r) for r in results]

    async def _search_rag(self, query: str, k: int) -> list[Memory]:
        q_emb = await self._embed(query)
        results = await self.vector.search(
            collection="knowledge_base",
            query_vector=q_emb,
            limit=k
        )
        return [Memory(**r) for r in results]

    def _rank_merge(self, *memory_lists: list[Memory], query: str) -> list[Memory]:
        scored = []
        for mem_list in memory_lists:
            for mem in mem_list:
                recency = (datetime.utcnow() - mem.created_at).total_seconds()
                recency_score = np.exp(-recency / 3600)  # decay over 1 hour
                access_score = min(mem.access_count / 10, 1.0)
                total = (mem.importance * 0.4 + recency_score * 0.3 + access_score * 0.3)
                scored.append((total, mem))
        
        scored.sort(key=lambda x: x[0], reverse=True)
        return [m for _, m in scored[:10]]  # Top 10

    async def _consolidate(self):
        """Summarize and move short-term to episodic"""
        # Read recent ST memories
        # Summarize with LLM
        # Store as episodic vector
        pass

    async def _embed(self, text: str) -> list[float]:
        # Call embedding API (OpenAI, etc.)
        return [0.0] * 1536  # placeholder
```

## 9. Python Implementation
```python
import openai
from pydantic import BaseModel
from datetime import datetime
import uuid

class RAGMemory:
    def __init__(self, embedding_client, vector_db, chunk_size=512, overlap=50):
        self.embedder = embedding_client
        self.vector_db = vector_db
        self.chunk_size = chunk_size
        self.overlap = overlap

    async def ingest_document(self, doc_id: str, content: str, metadata: dict = None):
        chunks = self._chunk_text(content)
        for i, chunk in enumerate(chunks):
            embedding = await self.embedder.create(input=chunk).embeddings[0].vector
            await self.vector_db.upsert(
                collection="rag",
                vectors=[{
                    "id": f"{doc_id}:{i}",
                    "values": embedding,
                    "metadata": {
                        "doc_id": doc_id,
                        "chunk_index": i,
                        "text": chunk,
                        **(metadata or {})
                    }
                }]
            )

    def _chunk_text(self, text: str) -> list[str]:
        chunks = []
        start = 0
        while start < len(text):
            end = min(start + self.chunk_size, len(text))
            chunks.append(text[start:end])
            start += self.chunk_size - self.overlap
        return chunks

    async def retrieve(self, query: str, k: int = 5, min_score: float = 0.7) -> list[dict]:
        q_emb = await self.embedder.create(input=query).embeddings[0].vector
        results = await self.vector_db.query(
            collection="rag",
            query_vector=q_emb,
            limit=k
        )
        return [
            r.metadata for r in results 
            if r.score >= min_score
        ]


class EpisodicMemory:
    def __init__(self, vector_db, llm_client):
        self.vector_db = vector_db
        self.llm = llm_client

    async def record_experience(self, agent_id: str, goal: str, steps: list, outcome: str):
        summary = await self._summarize_experience(goal, steps, outcome)
        embedding = await self._embed(summary)
        
        await self.vector_db.upsert(
            collection=f"episodic:{agent_id}",
            vectors=[{
                "id": uuid.uuid4().hex,
                "values": embedding,
                "metadata": {
                    "goal": goal,
                    "summary": summary,
                    "outcome": outcome,
                    "timestamp": datetime.utcnow().isoformat()
                }
            }]
        )

    async def recall_similar(self, agent_id: str, current_goal: str, k: int = 3):
        q_emb = await self._embed(current_goal)
        results = await self.vector_db.query(
            collection=f"episodic:{agent_id}",
            query_vector=q_emb,
            limit=k
        )
        return [(r.metadata["goal"], r.metadata["summary"], r.metadata["outcome"]) 
                for r in results]

    async def _summarize_experience(self, goal: str, steps: list, outcome: str) -> str:
        response = await self.llm.chat.completions.create(
            model="gpt-4o",
            messages=[{"role": "user", "content": 
                f"Summarize this agent experience:\nGoal: {goal}\n"
                f"Steps: {steps}\nOutcome: {outcome}\n"
                f"One paragraph."}]
        )
        return response.choices[0].message.content
```

## 10. Folder Structure
```
memory/
├── controller/
│   ├── memory_controller.py  # Orchestrates read/write across layers
│   ├── consolidation.py      # ST->LT, ST->Episodic promotion
│   └── forget.py             # TTL management, importance decay
├── stores/
│   ├── short_term.py         # Redis-based sliding window
│   ├── long_term.py          # PostgreSQL structured storage
│   ├── episodic.py           # Vector store for experiences
│   └── rag.py                # RAG document ingestion & retrieval
├── embedding/
│   ├── embedder.py           # OpenAI/Cohere/Azure embedding
│   └── chunker.py            # Text splitting strategies
├── ranking/
│   ├── scorer.py             # Recency * importance * relevance
│   └── merger.py             # Merge + dedup across layers
└── api/
    └── routes.py             # CRUD endpoints for memory
```

## 11. Configuration
```yaml
memory:
  short_term:
    backend: redis
    ttl_seconds: 3600
    max_messages: 20
    max_tokens: 4096
  
  long_term:
    backend: postgresql
    connection_pool_size: 20
    table: agent_long_term_memory
  
  episodic:
    backend: pinecone
    index_name: agent_episodic
    dimension: 1536
    metric: cosine
    k_default: 5
  
  rag:
    chunk_size: 512
    chunk_overlap: 50
    top_k: 5
    min_score: 0.7
  
  consolidation:
    interval_minutes: 30
    importance_threshold: 0.7
    max_long_term_per_user: 1000
```

## 12. Flowchart
```
[Input] -> [Memory Controller]
               |
        +-------v--------+
        |  Retrieve from  |
        |  all layers     |
        +-------+--------+
                |
        +-------v--------+
        |  Rank & Merge   |
        +-------+--------+
                |
        +-------v--------+
        |  Inject context |
        |  into LLM       |
        +-------+--------+
                |
        +-------v--------+
        |  Agent processes|
        +-------+--------+
                |
        +-------v--------+
        |  Store results  |
        |  to ST / LT /   |
        |  Episodic       |
        +-------+--------+
                |
        +-------v--------+
        |  Consolidate if |
        |  needed         |
        +----------------+
```

## 13. Sequence Diagram
```
Agent         MemController      Redis(ST)       PG(LT)       VectorDB(Epi)     Embedder
 |                  |               |              |               |              |
 |--store action-->|               |              |               |              |
 |                  |--append------>|               |               |              |
 |                  |               |               |               |              |
 |                  |--if imp >0.7->|               |               |              |
 |                  |  embed--------|-------------->|-------------->|              |
 |                  |  store        |               |               |              |
 |                  |               |               |               |              |
 |--query--------->|               |              |               |              |
 |                  |--get ST------>|               |               |              |
 |                  |<--ST msgs-----|               |               |              |
 |                  |--get LT------>|               |               |              |
 |                  |<--LT facts----|               |               |              |
 |                  |--search Epi-->|              |               |              |
 |                  |<--sim exp-----|               |               |              |
 |                  |--embed query->|              |               |              |
 |                  |               |               |               |--embed---->|
 |                  |               |               |               |<--vec------|
 |                  |               |               |               |              |
 |<--merged ctx-----|               |               |               |              |
```

## 14. Pros
- Multi-layer retrieval provides rich context
- Episodic memory enables "learning from experience"
- RAG scales knowledge beyond context window
- Consolidation prevents memory explosion
- TTL-based forgetting keeps memory fresh

## 15. Cons
- Vector embedding adds latency (~200ms per query)
- Storage costs (vector DB is expensive at scale)
- Memory pollution from bad experiences
- Consolidation LLM calls add cost
- Privacy concerns with persistent user memory

## 16. Alternatives
- **Infinite context LLMs (Gemini 10M)** — no external memory needed
- **MemGPT** — OS-style virtual memory management for LLMs
- **KV cache reuse** — no explicit memory store, cache attention states
- **Knowledge graphs** — structured memory with relationship traversal

## 17. Performance Considerations
- Embedding lookup: ~200ms per query (batch for throughput)
- Vector search: <100ms for <10M vectors with IVF_PQ
- Redis ST ops: <1ms (single digit microseconds)
- PG LT ops: <5ms with proper indexing
- Consolidation is async — never in the critical path

## 18. Scaling to Millions
```
Short-Term (Redis):
  - Cluster with 6-12 shards
  - Pipeline batch reads for high-throughput sessions
  - TTL ensures automatic eviction

Long-Term (PostgreSQL):
  - Partition by user_id hash (256 partitions)
  - BRIN index on created_at for time-range queries
  - Hot row caching in Redis for active users

Episodic (Vector DB):
  - Pinecone/Weaviate with pod-based sharding
  - Pre-filter by user_id before vector search
  - Quantization (PQ) for 10x memory reduction

RAG:
  - Embedding pre-computation pipeline (async job)
  - CDN for static document chunks
  - Tiered retrieval: small index for hot docs, large for cold
```

## 19. Failure Scenarios
| Scenario | Impact | Mitigation |
|----------|--------|------------|
| Vector DB down | No episodic recall | Fallback to ST + LT only |
| Embedder timeout | No RAG | Cache frequent query embeddings |
| Memory corruption | Wrong agent decisions | Versioned memories, rollback |
| ST overflow | Lost context | Summarize oldest messages |
| Full long-term table | Slow queries | Archival to cold storage |

## 20. Security
- User memories are namespaced — never shared across users
- PII detection before storing any memory
- Retention policies enforced at storage layer (delete after N days)
- Memory access audit log (who read what, when)
- Encryption at rest (AES-256) for LT and vector store
- Embedding of PII data is itself PII — handle accordingly

## 21. Monitoring
```prometheus
memory_read_latency_ms{layer="st|lt|episodic|rag"}
memory_write_throughput_ops
memory_total_entries{type="short_term|long_term|episodic|rag"}
memory_consolidation_count_total
memory_cache_hit_ratio
memory_embedding_latency_ms
memory_forget_events_total
vector_search_recall_rate
```

## 22. Interview Questions
**Q1**: How does agent memory differ from traditional database caching? (Memory is about relevance + importance ranking, not just speed; it involves semantic understanding of what to keep vs discard)

**Q2**: How do you prevent memory from growing unbounded? (TTL, importance threshold, consolidation + summarization, archival policies)

**Q3**: How would you implement cross-session memory without leaking user privacy? (User-namespaced stores, encrypted embeddings, PII redaction before storage)

## 23. Cheat Sheet
```
Read path:     ST -> LT -> Episodic -> RAG -> Merge & Rank -> Context
Write path:    ST -> (if important) LT -> (if successful) Episodic
Forget:        TTL expiry OR importance decay OR explicit delete
Consolidation: Summarize old ST -> Store as episodic -> Delete old ST
Memory budget: Max tokens for memory in context (recommended: 30% of context)
```

## 24. Common Mistakes
- Storing everything in short-term memory (tokens blow up)
- No importance scoring — all memories treated equally
- Retrieving too many memories — context pollution
- Not deduplicating across layers — redundant context
- Forgetting to embed queries for vector search — always embed at query time too
- Giving agent direct write access to long-term memory — needs validation

## 25. Production Best Practices
1. Always set memory budget as fraction of context window (never exceed 50%)
2. Use async consolidation — don't block the agent loop on memory writes
3. Implement importance classifier for automatic memory prioritization
4. Cache embeddings aggressively (identical text = identical embedding)
5. Monitor memory-to-output ratio — high ratio = poor retrieval quality
6. Pre-compute RAG embeddings in a background pipeline, not at query time
7. Use write-behind cache for high-frequency memory writes
8. Test with and without memory to quantify memory's actual contribution
