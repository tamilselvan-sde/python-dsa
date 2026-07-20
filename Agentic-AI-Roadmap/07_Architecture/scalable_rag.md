# Scalable RAG Systems

## 1. What is it?
Scalable RAG (Retrieval-Augmented Generation) systems are architectures that maintain sub-second query latency and high recall as the knowledge base grows from thousands to billions of documents, serving millions of daily queries.

## 2. Why do we need it?
Standard RAG fails at scale: sequential retrieval + generation creates bottlenecks, vector indexes saturate memory, metadata filtering slows down, and LLM costs explode. A scalable RAG architecture addresses all dimensions — data volume, query throughput, latency, and cost.

## 3. Real-world Example
**Notion AI** powers Q&A over 100M+ pages across millions of workspaces. Their scalable RAG system shards indexes per workspace (tenant isolation), uses tiered retrieval (keyword pre-filter → vector search → re-ranking), and batches embedding generation asynchronously. p50 latency stays under 800ms even during peak hours.

## 4. Architecture Diagram (ASCII)

```
                     +---------------------+
                     |   Query Router      |
                     | (Intent Classifier) |
                     +----------+----------+
                                |
              +-----------------+-----------------+
              |                                   |
      +-------v-------+                  +--------v--------+
      |   Fast Path    |                  |   Deep Path     |
      | (Keyword/BM25) |                  | (Vector Search) |
      +-------+-------+                  +--------+--------+
              |                                   |
              v                                   v
      +-------+--------+                +---------+--------+
      |  Cache Layer    |                |  Tiered Retrieval|
      | (Redis Cluster) |                |  (Shard + Filter)|
      +-------+--------+                +---------+--------+
              |                                   |
              +-----------------+-----------------+
                                |
                       +--------v--------+
                       |  Reranker (List) |
                       |  (Cross-Encoder) |
                       +--------+--------+
                                |
                       +--------v--------+
                       |  LLM Generation  |
                       |  (Streaming)     |
                       +--------+--------+
                                |
                       +--------v--------+
                       | Response Builder |
                       +-----------------+
```

## 5. Internal Working
Scalable RAG decomposes retrieval into stages: **Pre-retrieval** (query expansion, HyDE, query rewriting), **Multi-stage retrieval** (fast BM25 → ANN vector search → learned re-ranking), **Fusion** (RRF/CC for multi-source results), and **Post-retrieval** (compression, extraction). Each stage is independently scalable and cacheable.

## 6. Production Flow

```
Query → Query Understanding (rewrite, expand, HyDE)
→ Stage 1: Lightweight retrieval (BM25/Elasticsearch) — 10ms
→ Stage 2: Dense retrieval (Vector DB) — 50ms
→ Stage 3: Rerank top-100 to top-5 (Cross-encoder) — 100ms
→ Compress context (pick relevant spans, truncate) — 10ms
→ LLM generation (streaming response) — 500ms-2s
→ Response
```

## 7. HLD

### Components
1. **Query Processor**: Rewriting, HyDE (Hypothetical Document Embedding), query expansion
2. **Index Tiering**: Hot (SSD/vector), Warm (S3 + DiskANN), Cold (S3 + re-index on demand)
3. **Multi-Stage Retrieval Pipeline**: BM25 → ANN → Reranker
4. **Context Manager**: Sliding window, relevance filtering, compression
5. **LLM Serving**: Distilled model for simple queries, full model for complex

### Data Flow
```
Document Ingestion:
  Documents → Chunking (recursive, semantic) → Embedding (batch)
  → Multi-vector representation (summary + chunk + metadata)
  → Index (hot/warm/cold per access pattern)

Query Serving:
  Query → Cache check → Fast retrieval → LLM classify difficulty
  → Simple: cached/distilled
  → Complex: full multi-stage retrieval → full LLM
```

## 8. LLD

### Multi-Stage Retriever

```python
class MultiStageRetriever:
    def __init__(self):
        self.bm25 = BM25Retriever()
        self.vector = VectorRetriever()
        self.reranker = CrossEncoderReranker(model="ms-marco-MiniLM-L6-v2")
        self.query_rewriter = QueryRewriter()

    async def retrieve(self, query: str, k: int = 5) -> list[Chunk]:
        rewritten = self.query_rewriter.rewrite(query)

        # Stage 1: BM25 (fast, cheap)
        bm25_results = await self.bm25.search(rewritten, top_k=200)

        # Stage 2: Vector search (expensive but semantic)
        vector_results = await self.vector.search(rewritten, top_k=200)

        # Merge (RRF)
        merged = self.rrf_fusion([bm25_results, vector_results], k=60)

        # Stage 3: Rerank top 100 with cross-encoder
        candidates = merged[:100]
        reranked = await self.reranker.rerank(rewritten, candidates)

        return reranked[:k]

    def rrf_fusion(self, result_lists: list[list], k: int = 60) -> list:
        scores = defaultdict(float)
        for rank, result_list in enumerate(result_lists):
            for i, doc in enumerate(result_list):
                scores[doc.id] += 1.0 / (k + i)
        return sorted(merged, key=lambda x: scores[x.id], reverse=True)
```

### Context Compression

```python
class ContextCompressor:
    def __init__(self, max_tokens: int = 3000):
        self.max_tokens = max_tokens
        self.extractor = RelevantSpanExtractor()

    def compress(self, chunks: list[Chunk], query: str) -> str:
        # Extract only relevant spans (not whole chunks)
        spans = []
        for chunk in chunks:
            relevant = self.extractor.extract(chunk.text, query)
            spans.extend(relevant)

        # Truncate by token count
        sorted_spans = sorted(spans, key=lambda x: x.relevance, reverse=True)
        result = []
        total_tokens = 0

        for span in sorted_spans:
            tokens = count_tokens(span.text)
            if total_tokens + tokens > self.max_tokens:
                break
            result.append(span.text)
            total_tokens += tokens

        return "\n".join(result)
```

## 9. Python Implementation

```python
from abc import ABC, abstractmethod
from dataclasses import dataclass
import asyncio

class RetrievalStrategy(ABC):
    @abstractmethod
    async def search(self, query: str, top_k: int) -> list[Chunk]:
        ...

@dataclass
class QueryResult:
    answer: str
    sources: list[Chunk]
    latency_ms: dict
    model: str

class ScalableRAG:
    def __init__(self):
        self.fast_pipeline = FastRAGPipeline()
        self.deep_pipeline = DeepRAGPipeline()
        self.difficulty_classifier = DifficultyClassifier()

    async def query(self, user_query: str) -> QueryResult:
        difficulty = await self.difficulty_classifier.classify(user_query)

        if difficulty == "simple":
            return await self.fast_pipeline.run(user_query)
        else:
            return await self.deep_pipeline.run(user_query)

class FastRAGPipeline:
    def __init__(self):
        self.cache = SemanticCache()
        self.retriever = BM25Retriever()
        self.llm = DistilledLLM()

    async def run(self, query: str) -> QueryResult:
        cached = await self.cache.lookup(query)
        if cached:
            return cached

        context = await self.retriever.search(query, top_k=3)
        answer = await self.llm.generate(query, context)
        result = QueryResult(answer=answer, sources=context, ...)
        await self.cache.store(query, result)
        return result

class DeepRAGPipeline:
    def __init__(self):
        self.rewriter = QueryRewriter()
        self.retriever = MultiStageRetriever()
        self.compressor = ContextCompressor()
        self.llm = FrontierLLM()

    async def run(self, query: str) -> QueryResult:
        rewritten = self.rewriter.rewrite(query)
        context = await self.retriever.retrieve(rewritten, top_k=10)
        compressed = self.compressor.compress(context, rewritten)
        answer = await self.llm.generate(rewritten, compressed)
        return QueryResult(answer=answer, sources=context, ...)
```

## 10. Folder Structure

```
scalable_rag/
├── ingestion/
│   ├── chunker.py
│   ├── embedder.py
│   └── indexer.py
├── retrieval/
│   ├── bm25.py
│   ├── vector.py
│   ├── hybrid.py
│   └── reranker.py
├── query/
│   ├── rewriter.py
│   ├── classifier.py
│   └── hyde.py
├── context/
│   ├── compressor.py
│   └── extractor.py
├── cache/
│   ├── semantic.py
│   └── response.py
├── llm/
│   ├── client.py
│   └── fallback.py
└── monitors/
    ├── latency.py
    └── recall.py
```

## 11. Configuration

```yaml
pipeline:
  fast_path:
    retriever: bm25
    top_k: 3
    llm: gpt-4o-mini
    cache_ttl: 3600
  deep_path:
    stages:
      - type: bm25
        top_k: 200
      - type: vector
        top_k: 200
        index: ivf_pq
      - type: reranker
        model: ms-marco-MiniLM-L6-v2
        top_k: 5
    context:
      max_tokens: 3000
      strategy: relevant_span
    llm: gpt-4o
```

## 12. Flowchart

```
Query In
  |
  v
Query Understanding (rewrite, expand, HyDE)
  |
  v
Check Cache
  |      \
  |      (miss)
  v       v
Route by Difficulty
  |         \
simple     complex
  |          |
  v          v
BM25 → 3 docs    BM25 + ANN → 200 docs
  |              |
  v              v
Distilled LLM    Rerank 200 → 5
  |              |
  v              v
Response    Frontier LLM
  |              |
  +-------+------+
          |
          v
     Cache Store
          |
          v
     Response Out
```

## 13. Sequence Diagram

```
Client   Gateway  Cache   Retriever  Reranker   LLM
  |        |        |        |          |        |
  |--req-->|        |        |          |        |
  |        |--check->|        |          |        |
  |        |<--miss--|        |          |        |
  |        |        |        |          |        |
  |        |--classify        |          |        |
  |        |---"complex"      |          |        |
  |        |        |        |          |        |
  |        |--rewrite-------->|          |        |
  |        |        |        |          |        |
  |        |--bm25 search---->|          |        |
  |        |<--200 docs-------|          |        |
  |        |        |        |          |        |
  |        |--vector search-->|          |        |
  |        |<--200 docs-------|          |        |
  |        |        |        |          |        |
  |        |--rerank(200)-------------->|        |
  |        |<--5 docs-------------------|        |
  |        |        |        |          |        |
  |        |--context compress          |        |
  |        |        |        |          |        |
  |        |--generate------------------------->|
  |        |<--stream---------------------------|
  |<--resp-|        |        |          |        |
```

## 14. Pros
- Sub-second latency at 100M+ document scale; Graceful degradation (fast path always works); Cost-efficient (distilled model for 60%+ queries); High recall via multi-stage retrieval; Cache reduces LLM costs by 30-50%; Tenants isolated via partition key.

## 15. Cons
- Infrastructure complexity (multiple retrieval backends); Query routing adds 20-50ms overhead; Reranker is an extra dependency to maintain; Context compression may lose information; Tiered storage adds ingestion complexity; HyDE can hallucinate query expansions.

## 16. Alternatives
- **Single-stage with better embedding**: Simpler but lower recall; **No-rerank direct LLM**: Faster but worse quality; **Extractive only (no generation)**: Lower cost, limited use cases; **Agent-based retrieval**: More flexible but higher latency.

## 17. Performance Considerations
- BM25 pre-filter before vector search reduces candidates 10x; Reranker is the most expensive stage (100ms for 100 docs); Batch embeddings at ingestion, not query time; Context compression reduces LLM cost by 40% (fewer tokens); Cache hit ratio target > 50%; Fast path handles 70% of queries with 100ms p50.

## 18. Scaling to Millions
- **10M docs**: Single shard, BM25 + IVF vector; **100M**: 8-16 shards, multi-stage retrieval; **1B**: 64+ shards, DiskANN for warm tier, S3 for cold; Read replicas for cache-avoidant queries; Query router as shared-nothing stateless service; Tiered storage: hot (NVMe 1TB), warm (SSD 10TB), cold (S3 unlimited).

## 19. Failure Scenarios
- **Reranker throughput exhaustion**: Degrade to BM25+vector only; **LLM API rate limit**: Queue + slower fallback model; **Vector DB slow (>200ms)**: BM25-only retrieval path; **Cache invalidation storm**: Stale cache serves until TTL; **Embedding service down**: Use cached embeddings or fallback to BM25 only.

## 20. Security
- Query injection detection at gateway; Tenant isolation via separate indexes; PII redaction before context injection; Content filter on LLM output; Rate limiting per tenant; Audit log of all queries and retrievals; Context window monitoring for prompt injection; TLS for all services.

## 21. Monitoring
- **Recall@K**: Track against labeled test set (weekly); **Latency**: p50/p99 per pipeline stage; **Cache**: hit ratio, eviction rate, size; **Cost**: tokens saved by cache, per-query cost breakdown; **Stage utilization**: BM25 vs vector vs reranker split; **Quality**: User satisfaction survey correlation with pipeline choice.

## 22. Interview Questions
1. *Design a RAG system that serves 100M queries/day.* — Multi-stage retrieval, tiered cache, difficulty-based routing, auto-scaling.
2. *How do you reduce LLM costs in a high-volume RAG system?* — Semantic cache, distilled model for easy queries, context compression, batching.
3. *What's the tradeoff between BM25 and vector search?* — BM25: fast, keyword-precision, no training; Vector: semantic understanding, slower, needs embeddings.
4. *How would you scale document ingestion for a RAG system?* — Async pipeline with event-driven architecture, batch processing, incremental indexing.

## 23. Cheat Sheet

```
Scalable RAG = Query Routing + Multi-Stage Retrieval + Cache + Tiered Storage

Key Metrics:
  p50 latency < 500ms  |  p99 latency < 3s
  Recall@5 > 0.85      |  Cache hit rate > 50%
  LLM cost < $0.001/query  |  Availability 99.9%
```

## 24. Common Mistakes
- No fast path (every query goes through full pipeline); Using single-stage retrieval at scale; Not compressing context (blowing LLM context window); Ignoring cache effectiveness (thrashing cache); Over-relying on reranker (it's expensive); No query understanding (bad retrieval for typos/paraphrases); Monolithic deployment (all stages in one service).

## 25. Production Best Practices
- Implement fast/slow paths from day one; Cache aggressively (semantic + exact-match); Monitor recall quarterly against ground truth; Graceful degradation with explicit fallbacks; Cost-budget per query with alerts; A/B test pipeline changes on 5% traffic; Regular index rebuilds (weekly for hot, monthly for warm); Document emergency operational procedures (runbook); Train difficulty classifier on real traffic patterns.
