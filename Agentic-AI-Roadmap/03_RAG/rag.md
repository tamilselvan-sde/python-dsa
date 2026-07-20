# Retrieval Augmented Generation (RAG)

## 1. What is it?
**ELI5:** Imagine you're taking an open-book exam. Instead of memorizing everything (like a regular LLM), you're allowed to look up relevant pages from a textbook before answering each question. RAG is exactly that — it gives an LLM access to a knowledge base so it can look up facts before generating an answer.

**Technical Definition:** RAG is an AI framework that combines a retrieval system with a generative language model. Given a user query, the system first retrieves relevant documents from a knowledge base (vector DB, search index, etc.), then feeds those documents as context to an LLM to generate a grounded, factual response. This decouples knowledge storage from reasoning.

## 2. Why do we need it?
- **Hallucination reduction:** LLMs invent facts. RAG grounds responses in retrieved evidence, slashing hallucination rates from ~15-30% to <2%.
- **Outdated knowledge:** LLMs have a training cutoff. RAG lets you inject fresh data (news, internal docs, live APIs) without retraining.
- **Attribution & auditability:** Every answer can cite its source document, enabling compliance in regulated industries (healthcare, finance, legal).
- **Cost efficiency:** Training a model on new knowledge costs millions. RAG costs pennies per query in retrieval + generation.
- **Domain specificity:** Off-the-shelf LLMs don't know your internal APIs, product catalogs, or proprietary research. RAG bridges that gap.

## 3. Real-world Example
- **Google Search + Gemini:** Google's SGE (Search Generative Experience) retrieves web results, then Gemini summarizes them with citations.
- **Github Copilot Chat:** Retrieves relevant code snippets from the user's codebase before generating completions.
- **Morgan Stanley's DOCI:** Wealth management advisors query 100k+ internal documents via RAG — response grounded in firm-approved research.
- **Notion AI Q&A:** Retrieves from your workspace pages and databases before answering workspace-specific questions.
- **Amazon Bedrock Knowledge Bases:** Enterprise customers index their S3 data lakes into vector stores, then query via RAG for customer support, compliance, and HR.

## 4. Architecture Diagram (ASCII)
```
+--------+     +----------+     +-----------+
| User   | -->| Query    | -->| Retriever |
| Query  |    | Processor|    |           |
+--------+    +----------+    +-----------+
                                   |
                          +--------v--------+
                          |  Vector /       |
                          |  Search Index   |
                          |  (Knowledge     |
                          |   Base)         |
                          +-----------------+
                                   |
                          +--------v--------+
                          |  Retrieved Docs |
                          +-----------------+
                                   |
                          +--------v--------+
                          |  LLM + Context  |
                          |  (Generator)    |
                          +-----------------+
                                   |
                          +--------v--------+
                          |  Grounded       |
                          |  Response       |
                          +-----------------+
```

## 5. Internal Working
1. **Ingestion pipeline:** Documents are chunked → embedded via an embedding model (e.g., `text-embedding-3-small`) → stored in a vector DB with metadata.
2. **Query processing:** User input may be rewritten, expanded, or decomposed for better retrieval quality.
3. **Retrieval:** The query is embedded to the same vector space → Approximate Nearest Neighbor (ANN) search finds top-K chunks by cosine similarity.
4. **Augmentation:** Retrieved chunks are inserted into a prompt template as context: `Context: {chunks}\n\nQuestion: {query}\n\nAnswer:`
5. **Generation:** The LLM produces an answer grounded in the provided context — optionally with citations to source chunks.
6. **Post-processing:** Response may be reranked, filtered for safety, or checked for faithfulness to retrieved context.

## 6. Production Flow
```
User Query
    |
    v
[Gateway / Load Balancer]
    |
    v
[RAG Orchestrator Service]
    |
    +---> Query Rewriting (LLM)
    |
    +---> Dense Retrieval (Vector DB top-50)
    |
    +---> Sparse Retrieval (BM25 top-50)
    |
    +---> Hybrid Fusion (RRF merge → top-20)
    |
    +---> Reranking (Cross-encoder → top-5)
    |
    +---> Context Assembly (token budget management)
    |
    +---> LLM Generation (streaming)
    |
    +---> Citation Check & Safety Filter
    |
    v
[Response to User]
```

## 7. HLD
```
                    +-----------------------------+
                    |     Load Balancer (ALB)      |
                    +--------------+--------------+
                                   |
                    +--------------v--------------+
                    |     RAG API Gateway          |
                    |  (FastAPI / Kong)            |
                    +--+--------+--------+--------+
                       |        |        |
            +----------v-+  +---v----+  +-v----------+
            | Query      |  | Auth   |  | Rate       |
            | Rewriter   |  | Svc    |  | Limiter    |
            +----------+-+  +--------+  +-+----------+
                       |                    |
            +----------v-+       +----------v-+
            | Retriever  |       | Redis      |
            | Service    |       | Cache      |
            +---+--------+       +----------+-+
                |    |                        |
       +--------v+ +-v--------+    +---------v+
       | Vector  | | BM25     |    | Semantic |
       | DB      | | Index    |    | Cache    |
       +--------+ +----------+    +----------+
                |    |
       +--------v+ +-v--------+
       | Reranker | | Context  |
       | Svc      | | Assembly |
       +--------+ +----------+
                |
       +--------v+
       | LLM      |
       | (Gen Svc)|
       +--------+
```

## 8. LLD
```
class RAGOrchestrator:
    def __init__(self, config: RAGConfig):
        self.embedder = EmbeddingService(config.embedding_model)
        self.vector_db = VectorDBClient(config.vector_db_url)
        self.bm25_index = BM25Index(config.bm25_index_path)
        self.reranker = CrossEncoderReranker(config.reranker_model)
        self.llm = LLMClient(config.llm_endpoint, config.llm_api_key)
        self.cache = SemanticCache(config.redis_url)
        self.monitor = RAGMonitor()

    async def answer(self, query: str, user_id: str) -> RAGResponse:
        # 1. Check semantic cache
        cached = await self.cache.lookup(query)
        if cached:
            self.monitor.record_cache_hit(query)
            return cached

        # 2. Query rewriting
        rewritten = await self.rewriter.rewrite(query)

        # 3. Parallel retrieval
        dense_chunks = await self.vector_db.search(
            await self.embedder.embed(rewritten), top_k=50
        )
        sparse_chunks = await self.bm25_index.search(rewritten, top_k=50)

        # 4. Hybrid fusion (RRF)
        fused = reciprocal_rank_fusion(dense_chunks, sparse_chunks, k=60)

        # 5. Rerank top-60 → top-5
        reranked = await self.reranker.rerank(query, fused[:60], top_k=5)

        # 6. Context assembly (fit token budget)
        context = self.assemble_context(reranked, max_tokens=4096)

        # 7. Generate
        response = await self.llm.generate(
            prompt=self.build_prompt(query, context),
            stream=True
        )

        # 8. Cache and return
        await self.cache.store(query, response)
        self.monitor.record_query(query, len(context))
        return response
```

## 9. Python Implementation
```python
import asyncio
import hashlib
from dataclasses import dataclass, field
from typing import List, Optional

import numpy as np
from fastapi import FastAPI, HTTPException
from pydantic import BaseModel, Field
from openai import AsyncOpenAI
import redis.asynced as redis

app = FastAPI(title="RAG Service")

# --- Pydantic Models ---

class RAGQuery(BaseModel):
    query: str = Field(..., min_length=1, max_length=4096)
    user_id: str = Field(..., pattern=r"^[a-zA-Z0-9_-]+$")
    top_k: int = Field(default=5, ge=1, le=20)
    stream: bool = Field(default=False)

class Chunk(BaseModel):
    id: str
    text: str
    metadata: dict = {}
    score: float = 0.0

class Citation(BaseModel):
    chunk_id: str
    text_snippet: str
    source: str

class RAGResponse(BaseModel):
    answer: str
    citations: List[Citation]
    latency_ms: float

# --- Configuration ---

@dataclass
class RAGConfig:
    embedding_model: str = "text-embedding-3-small"
    llm_model: str = "gpt-4o"
    vector_db_url: str = "http://qdrant:6333"
    redis_url: str = "redis://redis:6379"
    max_context_tokens: int = 4096
    retrieval_top_k: int = 50
    rerank_top_k: int = 5

# --- Embedding Service ---

class EmbeddingService:
    def __init__(self, model: str):
        self.client = AsyncOpenAI()
        self.model = model

    async def embed(self, text: str) -> List[float]:
        resp = await self.client.embeddings.create(
            model=self.model, input=text
        )
        return resp.data[0].embedding

    async def embed_batch(self, texts: List[str]) -> List[List[float]]:
        resp = await self.client.embeddings.create(
            model=self.model, input=texts
        )
        return [d.embedding for d in resp.data]

# --- Vector DB Client ---

class VectorDBClient:
    def __init__(self, url: str):
        self.url = url

    async def search(
        self, vector: List[float], top_k: int = 50
    ) -> List[Chunk]:
        # Qdrant / Pinecone / Weaviate client call
        # Returns top_k nearest neighbors
        return [Chunk(id="1", text="sample", score=0.95)]

    async def upsert(self, chunks: List[dict]):
        pass

# --- BM25 Client ---

class BM25Index:
    def __init__(self, index_path: str):
        self.index_path = index_path

    async def search(self, query: str, top_k: int = 50) -> List[Chunk]:
        # BM25 search via Tantivy / Elasticsearch
        return [Chunk(id="2", text="bm25 result", score=0.8)]

# --- Reranker ---

class CrossEncoderReranker:
    def __init__(self, model: str = "cross-encoder/ms-marco-MiniLM-L-6-v2"):
        self.model = model

    async def rerank(
        self, query: str, chunks: List[Chunk], top_k: int = 5
    ) -> List[Chunk]:
        # Sort by cross-encoder relevance score
        scored = sorted(chunks, key=lambda c: c.score, reverse=True)
        return scored[:top_k]

# --- Semantic Cache ---

class SemanticCache:
    def __init__(self, redis_url: str, threshold: float = 0.92):
        self.redis = None
        self.threshold = threshold

    async def connect(self):
        self.redis = await redis.from_url(self.redis_url)

    async def lookup(self, query: str) -> Optional[RAGResponse]:
        key = hashlib.sha256(query.encode()).hexdigest()
        cached = await self.redis.get(key)
        return None  # simplified

    async def store(self, query: str, response: RAGResponse):
        key = hashlib.sha256(query.encode()).hexdigest()
        await self.redis.setex(key, 3600, response.model_dump_json())

# --- Context Assembly ---

def reciprocal_rank_fusion(
    dense: List[Chunk], sparse: List[Chunk], k: int = 60
) -> List[Chunk]:
    scores = {}
    for rank, chunk in enumerate(dense + sparse):
        chunk_id = chunk.id
        score = 1.0 / (k + rank + 1)
        scores[chunk_id] = scores.get(chunk_id, 0) + score
    ranked_ids = sorted(scores, key=scores.get, reverse=True)
    all_chunks = {c.id: c for c in dense + sparse}
    return [all_chunks[cid] for cid in ranked_ids]

# --- Orchestrator ---

class RAGOrchestrator:
    def __init__(self, config: RAGConfig):
        self.config = config
        self.embedder = EmbeddingService(config.embedding_model)
        self.vector_db = VectorDBClient(config.vector_db_url)
        self.bm25 = BM25Index("/data/bm25_index")
        self.reranker = CrossEncoderReranker()
        self.cache = SemanticCache(config.redis_url)
        self.llm_client = AsyncOpenAI()

    async def answer(self, query: RAGQuery) -> RAGResponse:
        import time
        start = time.time()

        # Retrieve
        query_vec = await self.embedder.embed(query.query)
        dense_results = await self.vector_db.search(query_vec, top_k=50)
        sparse_results = await self.bm25.search(query.query, top_k=50)

        # Fuse + Rerank
        fused = reciprocal_rank_fusion(dense_results, sparse_results)
        reranked = await self.reranker.rerank(
            query.query, fused[:60], top_k=query.top_k
        )

        # Build context
        context = "\n\n".join(
            f"[{i+1}] {c.text}" for i, c in enumerate(reranked)
        )

        # Generate
        system_prompt = (
            "Answer using ONLY the provided context. "
            "Cite sources as [1], [2], etc. If unsure, say so."
        )
        user_prompt = f"Context:\n{context}\n\nQuestion: {query.query}"

        resp = await self.llm_client.chat.completions.create(
            model=self.config.llm_model,
            messages=[
                {"role": "system", "content": system_prompt},
                {"role": "user", "content": user_prompt},
            ],
            temperature=0.1,
            stream=query.stream,
        )

        answer = resp.choices[0].message.content

        citations = [
            Citation(
                chunk_id=c.id,
                text_snippet=c.text[:150],
                source=c.metadata.get("source", ""),
            )
            for c in reranked
        ]

        return RAGResponse(
            answer=answer,
            citations=citations,
            latency_ms=(time.time() - start) * 1000,
        )

orchestrator = RAGOrchestrator(RAGConfig())

@app.post("/rag/answer", response_model=RAGResponse)
async def rag_answer(query: RAGQuery):
    try:
        return await orchestrator.answer(query)
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))
```

## 10. Folder Structure
```
rag-service/
├── src/
│   ├── __init__.py
│   ├── main.py              # FastAPI entry
│   ├── config.py            # RAGConfig pydantic-settings
│   ├── orchestrator.py      # RAGOrchestrator
│   ├── retrieval/
│   │   ├── __init__.py
│   │   ├── dense.py         # Vector DB client
│   │   ├── sparse.py        # BM25 client
│   │   └── hybrid.py        # Fusion (RRF, CC)
│   ├── reranking/
│   │   ├── __init__.py
│   │   └── cross_encoder.py # Cross-encoder reranker
│   ├── generation/
│   │   ├── __init__.py
│   │   ├── llm_client.py    # OpenAI / Anthropic / local
│   │   └── prompt_templates.py
│   ├── embedding/
│   │   ├── __init__.py
│   │   └── embedder.py      # Embedding service
│   ├── cache/
│   │   ├── __init__.py
│   │   └── semantic_cache.py
│   └── monitoring/
│       ├── __init__.py
│       └── metrics.py       # Prometheus metrics
├── ingestion/
│   ├── __init__.py
│   ├── chunker.py           # Document chunking
│   ├── embed_and_index.py   # Pipeline runner
│   └── parsers/             # PDF, HTML, Markdown parsers
├── tests/
│   ├── test_retrieval.py
│   ├── test_reranking.py
│   ├── test_orchestrator.py
│   └── fixtures/
├── deploy/
│   ├── Dockerfile
│   ├── docker-compose.yml
│   └── k8s/
├── pyproject.toml
└── README.md
```

## 11. Configuration
```yaml
# config.yaml
rag:
  embedding:
    model: text-embedding-3-small
    dimensions: 1536
    batch_size: 100

  retrieval:
    dense_top_k: 50
    sparse_top_k: 50
    fusion_method: rrf  # rrf | cc | weighted
    rrf_k: 60

  reranking:
    enabled: true
    model: cross-encoder/ms-marco-MiniLM-L-6-v2
    top_k: 5

  generation:
    model: gpt-4o
    temperature: 0.1
    max_tokens: 2048
    max_context_tokens: 4096

  cache:
    type: semantic
    backend: redis
    ttl_seconds: 3600
    similarity_threshold: 0.92

  monitoring:
    metrics_port: 9090
    tracing: true
```

## 12. Flowchart
```
Start
  |
  v
Receive Query
  |
  v
[Cache Check] --hit--> Return Cached Response
  |                          
 miss                         
  |                          
  v                          
[Query Rewriting]                           
  |                          
  v                          
[Parallel Retrieval]         
  |            |             
  v            v             
[Dense]      [Sparse]       
  |            |             
  +----+-------+             
       |                     
       v                     
[Hybrid Fusion (RRF)]        
       |                     
       v                     
[Rerank (Cross-Encoder)]    
       |                     
       v                     
[Context Assembly]           
       |                     
       v                     
[LLM Generation]             
       |                     
       v                     
[Caching + Monitoring]       
       |                     
       v                     
[Response with Citations]    
```

## 13. Sequence Diagram
```
User       Gateway     RAG Svc      Vector DB   BM25      Reranker    LLM
 |            |           |            |          |          |          |
 |---Query--->|           |            |          |          |          |
 |            |--Query--->|            |          |          |          |
 |            |           |            |          |          |          |
 |            |           |---Cache----|----------|----------|----------|
 |            |           |<--Miss-----|----------|----------|----------|
 |            |           |            |          |          |          |
 |            |           |---Embed--->|          |          |          |
 |            |           |<--Vector---|          |          |          |
 |            |           |            |          |          |          |
 |            |           |---Dense--->|          |          |          |
 |            |           |            |--ANN---->|          |          |
 |            |           |<--Top-50---|          |          |          |
 |            |           |            |          |          |          |
 |            |           |---Sparse------------->|          |          |
 |            |           |            |          |--BM25--->|          |
 |            |           |<--Top-50--------------|          |          |
 |            |           |            |          |          |          |
 |            |           |---RRF-----------------+----------|          |
 |            |           |            |          |          |          |
 |            |           |---Rerank------------------------->|          |
 |            |           |<--Top-5---------------------------|          |
 |            |           |            |          |          |          |
 |            |           |---Generate-------------------------->|      |
 |            |           |<--Response---------------------------|      |
 |            |           |            |          |          |          |
 |<--Response-|---|       |            |          |          |          |
```

## 14. Pros
- Grounds LLM responses in retrievable evidence
- Reduces hallucination by ~80-95% vs. bare LLM
- Knowledge updates via re-indexing — no retraining
- Supports citation and audit trails
- Modular: swap retriever, reranker, or LLM independently
- Scales to millions of documents
- Works with any document format (PDF, HTML, Markdown, code)

## 15. Cons
- Retrieval failure → bad answer even with good LLM
- Latency: retrieval + generation = 2-5x slower than direct LLM
- Chunk quality directly impacts answer quality
- Context window limits how many docs you can inject
- Requires a separate ingestion/indexing pipeline
- Synonym gaps and out-of-vocabulary terms hurt sparse retrieval
- Embedding models have knowledge cutoff too

## 16. Alternatives
| Approach | Description | When to use |
|----------|-------------|-------------|
| Fine-tuning | Train LLM on domain data | Small, stable document corpus |
| Prompt engineering | Few-shot in context window | Simple, small knowledge |
| In-context learning | Provide examples in prompt | No retrieval infra available |
| Tool-use / Function calling | LLM calls APIs dynamically | Real-time data (weather, stock) |
| Graph RAG | Knowledge graph + retrieval | Multi-hop reasoning across entities |
| Agentic RAG | Agent decides retrieval strategy | Complex, multi-step queries |
| Fusion-in-Decoder | Encoder-decoder with retrieved passages | Max accuracy, higher compute |

## 17. Performance Considerations
- **Embedding latency:** Batch embeddings during ingestion; use GPU or ONNX for real-time.
- **Vector DB index type:** HNSW (fast, memory-heavy) vs IVF (slower but smaller). HNSW for production.
- **RRF fusion cost:** O(N log N) where N = dense_top_k + sparse_top_k (typically 100).
- **Cross-encoder reranker:** Much slower than bi-encoders — only rerank top 20-60, not 1000.
- **LLM generation:** Use streaming for UX. Set `max_tokens` to avoid runaway costs.
- **Cache hit ratio:** Aim for >50% with semantic caching (identical + similar queries).
- **P95 latency budget:** 500ms retrieval + 200ms reranking + 1-3s generation = ~2-4s total.

## 18. Scaling to Millions
- **Shard vector DB** across multiple nodes by document collection or hash.
- **Pre-filter with metadata** to reduce search space before ANN search.
- **Use a reader/writer split:** One set of nodes for ingestion indexing, another for query serving.
- **Batch inference for embeddings:** Embed 100 chunks at once on GPU.
- **Async non-blocking I/O:** All retrieval calls in parallel (asyncio.gather).
- **Content-addressable cache:** Deduplicate identical chunks across documents.
- **Distributed BM25:** Elasticsearch shards across 10+ nodes with replication factor 2.
- **LLM load balancing:** Round-robin across multiple model replicas (K8s HPA).

## 19. Failure Scenarios
| Failure | Impact | Mitigation |
|---------|--------|------------|
| No relevant docs retrieved | LLM guesses / hallucinates | Fallback: "I don't have enough information" |
| Vector DB down | Complete retrieval failure | Circuit breaker → fallback to BM25 only |
| Embedding model degraded | Poor retrieval quality | Monitor cosine similarity distribution |
| Context overflow | Truncation loses critical info | Smart truncation (remove lowest-score chunks) |
| Cache stampede | All requests miss simultaneously | Probabilistic early expiration |
| LLM rate limited | Generation delayed | Queue requests; exponential backoff |
| Stale index | Answers based on outdated docs | Incremental indexing every N minutes |

## 20. Security
- **Guardrails:** Plug a guardrail service (e.g., NVIDIA NeMo Guardrails) after generation to filter toxic, PII, or off-topic responses.
- **Access control:** Tag documents with access tiers; filter retrieved chunks by user permissions before generation.
- **Prompt injection:** Sanitize retrieved chunks — they come from documents, not users, but adversarial docs exist. Apply input validation on indexed content.
- **Data isolation:** Multi-tenant vector DB with tenant_id metadata filter on every query.
- **Audit logging:** Log every query, retrieved chunk IDs, and generated response for compliance.
- **Encryption at rest:** Encrypt vector DB indexes and BM25 term dictionaries.
- **Rate limiting:** Per-user and per-tenant rate limits on the RAG endpoint.

## 21. Monitoring
- **Retrieval metrics:** `top_k_recall@5`, `mean_reciprocal_rank`, `chunk_count_per_query`
- **Reranker metrics:** Distribution of reranker scores (drift detection)
- **Generation metrics:** `tokens_generated_per_query`, `generation_latency_p50/p95/p99`
- **Cache metrics:** `cache_hit_ratio`, `cache_lookup_latency`
- **System:** `vector_db_latency`, `embedding_latency`, `rrf_fusion_latency`
- **Quality:** Human feedback thumbs up/down rate, automated answer faithfulness score
- **Alerts:** P95 latency > 5s, cache hit ratio < 20%, error rate > 1%

## 22. Interview Questions
| Level | Question | Key Answer Points |
|-------|----------|-------------------|
| L3 | What is RAG and why use it? | Grounding, hallucination reduction, fresh data |
| L4 | How does hybrid search work in RAG? | Dense (semantic) + sparse (keyword), RRF fusion |
| L4 | How would you handle no results from retrieval? | Fallback responses, query expansion, "I don't know" |
| L5 | How would you debug a RAG system giving bad answers? | Isolate: retrieval quality → context quality → LLM quality |
| L5 | Design a RAG system for 10M documents | Sharding, tiered retrieval, incremental indexing |
| L6 | How would you measure RAG quality end-to-end? | Retrieval metrics + faithfulness score + user satisfaction |
| L6 | Tradeoffs: Fine-tune vs RAG for a customer support bot | RAG: dynamic KB, easy updates. Fine-tune: lower latency, no retrieval infra |

## 23. Cheat Sheet
```python
# RAG in 10 lines
query = "What is RAG?"
query_vec = embed(query)
dense_chunks = vector_db.search(query_vec, top_k=50)
sparse_chunks = bm25.search(query, top_k=50)
fused = rrf(dense_chunks, sparse_chunks, k=60)
reranked = cross_encoder.rerank(query, fused[:60], top_k=5)
context = "\n".join(c.text for c in reranked)
prompt = f"Context: {context}\nQ: {query}\nA:"
answer = llm.generate(prompt)
```

## 24. Common Mistakes
- **Token budget mismanagement:** Context too long = truncated + lost information. Always count tokens before sending to LLM.
- **No metadata filtering:** Retrieving from all documents when user only needs data from Q3 2024. Always apply filters early.
- **Ignoring chunk order:** Chunks returned out of order. Sort by document position before assembling context.
- **Over-embedding:** Embedding entire documents as single vectors. Chunk at 256-512 tokens for granularity.
- **Single retrieval method:** Dense-only misses exact keywords; sparse-only misses semantic matches. Always use hybrid.
- **No caching:** Same question asked 100 times → 100 identical retrievals + generations. Cache semantically.
- **Skipping reranker:** Bi-encoder retrieval is fast but noisy. A cross-encoder reranker dramatically improves final quality.

## 25. Production Best Practices
1. **Always use hybrid search** (dense + sparse) — never rely on one retrieval method alone.
2. **Rerank retrieved results** with a cross-encoder — it's the highest-leverage quality improvement per compute dollar.
3. **Implement semantic caching** — identical and similar queries get served from cache, slashing cost by 40-60%.
4. **Set a fallback response** — when retrieval returns nothing, say "I don't know" rather than letting the LLM guess.
5. **Measure retrieval recall offline** before deploying — build a golden dataset of query → relevant doc pairs.
6. **Use async I/O everywhere** — retrieval, embedding, reranking, and generation should all be concurrent.
7. **Apply metadata filtering before retrieval** — reduces search space and improves relevance dramatically.
8. **Stream LLM responses** — don't make users wait for full generation before seeing output.
9. **Log every query and retrieved chunk** — essential for debugging, auditing, and improving retrieval quality.
10. **Continuously evaluate** — set up a nightly eval pipeline with human-rated or LLM-as-judge scoring.
