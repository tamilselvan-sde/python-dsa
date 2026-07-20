# Hybrid Search (Dense + Sparse)

> Author: **Tamilselvan** · ✉️ tamilselvan.sde@gmail.com · 🔗 [LinkedIn](https://www.linkedin.com/in/tamilselvan-ai/)
>

## 1. What is it?
**ELI5:** Imagine searching for a recipe. Sparse search is like looking for the exact words "chicken tikka masala" — it finds pages that literally contain those words. Dense search is like remembering a similar recipe you once had — it finds pages that are conceptually related even if they use different words. Hybrid search combines both: it scans for exact keyword matches AND conceptually similar content, then merges the results so you never miss either.

**Technical Definition:** Hybrid search is a retrieval strategy that combines sparse retrieval methods (lexical matching like BM25, TF-IDF) with dense retrieval methods (neural embeddings, vector similarity) to produce a unified ranked result set. The two scores are merged via fusion algorithms like Reciprocal Rank Fusion (RRF) or weighted CombSUM, compensating for each method's blind spots.

## 2. Why do we need it?
- **Dense-only blind spots:** Vector embeddings miss exact keyword matches (product codes "IP12-256GB", serial numbers, acronyms like "CNN").
- **Sparse-only blind spots:** BM25 fails on synonyms, paraphrases, and semantic similarity ("automobile" won't match "car" without stemming).
- **Real-world queries mix both:** "Latest iPhone that costs less than $800" — needs keyword match on "iPhone" + semantic match on "budget flagship."
- **Domain-specific terms:** Medical and legal documents use precise terminology (ICD-10 codes, CFR citations) that embeddings dilute.
- **Cold-start problem:** New documents lack embedding history; BM25 covers them immediately via term frequency.

## 3. Real-world Example
- **Elasticsearch's `hybrid` query:** Combines `multi_match` (BM25) with `knn` (dense vector) in a single search API. Used by Uber Eats for restaurant + menu search.
- **Pinecone + Elasticsearch hybrid:** Companies run dense search in Pinecone and sparse in ES, merge via a lightweight fusion service.
- **Weaviate's hybrid search:** Native `Hybrid()` method that combines vector and BM25 scores with an alpha parameter (0 = pure BM25, 1 = pure vector).
- **Amazon Kendra:** Uses a hybrid of dense neural search and keyword search for enterprise document retrieval.
- **Google Search:** Internal systems blend neural retrieval (MUM) with traditional inverted-index signals.

## 4. Architecture Diagram (ASCII)
```
User Query
    |
    +--------+--------+
    |                  |
    v                  v
[Dense Encoder]    [BM25 / Sparse]
    |                  |
    v                  v
[Vector DB]        [Inverted Index]
    |                  |
    v                  v
[Dense Results]    [Sparse Results]
    |                  |
    +--------+--------+
             |
             v
[Fusion Algorithm]
  - RRF
  - CombSUM
  - Weighted Avg
             |
             v
  [Unified Ranked Results]
```

## 5. Internal Working
1. **Dense path:** Query → embedding model → vector DB ANN search → top-K dense results with cosine similarity scores.
2. **Sparse path:** Query → tokenization + stemming → BM25/Elasticsearch inverted index lookup → top-K sparse results with BM25 scores.
3. **Normalization:** Both score sets are normalized to a comparable range (min-max, z-score, or rank-based).
4. **Fusion:** Combined ranking via RRF (`score = Σ 1/(k + rank_i)`) or weighted linear combination (`score = α * dense_score + (1-α) * sparse_score`).
5. **Final ranking:** The fused list is sorted by combined score and the top-N results returned.

The `alpha` parameter in weighted fusion controls the balance:
- `α = 1.0` → pure dense retrieval
- `α = 0.0` → pure sparse retrieval
- `α = 0.5` → equal weight
- `α = 0.7` → favor semantic understanding

## 6. Production Flow
```
Query "How to reset iPhone passcode"
    |
    v
[Query pre-processing: lowercasing, stopword removal]
    |
    +-------> Dense path:
    |           embed("reset iPhone passcode") → [0.02, -0.15, ..., 0.33]
    |           Vector DB ANN search → top-50 (cosine sim = 0.93, 0.88, ...)
    |
    +-------> Sparse path:
    |           tokenize → ["reset", "iphone", "passcode"]
    |           BM25 inverted index → top-50 (BM25 score = 12.5, 8.3, ...)
    |
    v
[Score Normalization]
    dense_scores = [0.93, 0.88, 0.85, ...]  (already 0-1)
    sparse_scores = min-max normalize [12.5, 8.3, ...] → [1.0, 0.66, ...]
    |
    v
[RRF Fusion (k=60)]
    doc_A: RRF = 1/(60+1) + 1/(60+2) = 0.032  (ranked #1 dense, #2 sparse)
    doc_B: RRF = 1/(60+3) + 1/(60+1) = 0.031  (ranked #3 dense, #1 sparse)
    |
    v
[Sort by RRF score → top-20 → Return]
```

## 7. HLD
```
                    +------------------+
                    |   Query Router   |
                    +--------+---------+
                             |
              +--------------+--------------+
              |                             |
    +---------v--------+          +---------v--------+
    | Dense Retriever  |          | Sparse Retriever |
    | (Vector DB)      |          | (ES / Tantivy)   |
    +---------+--------+          +---------+--------+
              |                             |
              +----------+   +--------------+
                         |   |
                +--------v---v--------+
                |   Fusion Service    |
                |   (RRF / CombSUM)   |
                +---------+-----------+
                          |
                +---------v-----------+
                |   Result Ranker     |
                +---------+-----------+
                          |
                +---------v-----------+
                |   RAG Orchestrator  |
                +---------------------+
```

## 8. LLD
```python
import asyncio
from dataclasses import dataclass
from typing import List, Tuple

@dataclass
class SearchResult:
    doc_id: str
    dense_score: float
    sparse_score: float
    combined_score: float = 0.0
    text: str = ""
    metadata: dict = None

class HybridSearch:
    def __init__(self, alpha: float = 0.5, rrf_k: int = 60):
        self.alpha = alpha
        self.rrf_k = rrf_k

    async def search(
        self,
        query: str,
        embed_fn,
        dense_search_fn,
        sparse_search_fn,
        top_k: int = 20,
    ) -> List[SearchResult]:
        # Parallel dense + sparse retrieval
        dense_task = dense_search_fn(query, top_k=top_k * 2)
        sparse_task = sparse_search_fn(query, top_k=top_k * 2)

        dense_results, sparse_results = await asyncio.gather(
            dense_task, sparse_task
        )

        # Build result map
        result_map: dict[str, SearchResult] = {}

        for rank, r in enumerate(dense_results):
            if r.doc_id not in result_map:
                result_map[r.doc_id] = SearchResult(
                    doc_id=r.doc_id,
                    dense_score=r.score,
                    sparse_score=0.0,
                    text=r.text,
                    metadata=r.metadata,
                )
            else:
                result_map[r.doc_id].dense_score = r.score

            # RRF contribution from dense rank
            result_map[r.doc_id].combined_score += 1.0 / (self.rrf_k + rank + 1)

        for rank, r in enumerate(sparse_results):
            if r.doc_id not in result_map:
                result_map[r.doc_id] = SearchResult(
                    doc_id=r.doc_id,
                    dense_score=0.0,
                    sparse_score=r.score,
                    text=r.text,
                    metadata=r.metadata,
                )
            else:
                result_map[r.doc_id].sparse_score = r.score

            # RRF contribution from sparse rank
            result_map[r.doc_id].combined_score += 1.0 / (self.rrf_k + rank + 1)

        # Sort by combined RRF score descending
        ranked = sorted(
            result_map.values(),
            key=lambda x: x.combined_score,
            reverse=True,
        )

        return ranked[:top_k]
```

## 9. Python Implementation
```python
import numpy as np
from typing import List, Optional
from pydantic import BaseModel, Field
from fastapi import FastAPI

app = FastAPI(title="Hybrid Search Service")

class HybridQuery(BaseModel):
    query: str = Field(..., min_length=1)
    alpha: float = Field(default=0.5, ge=0.0, le=1.0)
    top_k: int = Field(default=20, ge=1, le=100)

class HybridResult(BaseModel):
    doc_id: str
    score: float
    dense_score: float
    sparse_score: float
    text: str
    metadata: dict = {}

class HybridResponse(BaseModel):
    results: List[HybridResult]
    alpha: float
    total_time_ms: float

class SimpleVectorDB:
    def __init__(self):
        self.documents = []
        self.embeddings = []

    def add(self, doc_id: str, text: str, embedding: list):
        self.documents.append({"id": doc_id, "text": text})
        self.embeddings.append(embedding)

    async def search(self, query_emb: list, top_k: int) -> List[dict]:
        emb = np.array(self.embeddings)
        query_emb = np.array(query_emb)
        scores = np.dot(emb, query_emb) / (
            np.linalg.norm(emb, axis=1) * np.linalg.norm(query_emb) + 1e-8
        )
        top_indices = np.argsort(scores)[-top_k:][::-1]
        return [
            {
                "doc_id": self.documents[i]["id"],
                "text": self.documents[i]["text"],
                "score": float(scores[i]),
            }
            for i in top_indices
        ]

class SimpleBM25:
    def __init__(self):
        self.doc_freqs = {}
        self.doc_lengths = []
        self.num_docs = 0
        self.avgdl = 0.0
        self.k1 = 1.5
        self.b = 0.75

    def add(self, doc_id: str, text: str):
        tokens = text.lower().split()
        for token in set(tokens):
            self.doc_freqs[token] = self.doc_freqs.get(token, 0) + 1
        self.doc_lengths.append(len(tokens))
        self.num_docs += 1
        self.avgdl = np.mean(self.doc_lengths)

    async def search(self, query: str, top_k: int) -> List[dict]:
        query_tokens = query.lower().split()
        scores = []
        # Simplified BM25 scoring
        for i in range(self.num_docs):
            score = 0
            for token in query_tokens:
                if token not in self.doc_freqs:
                    continue
                idf = np.log(
                    (self.num_docs - self.doc_freqs[token] + 0.5)
                    / (self.doc_freqs[token] + 0.5)
                    + 1
                )
                # tf in doc i
                tf = 0  # simplified
                score += idf * (tf * (self.k1 + 1)) / (
                    tf + self.k1 * (1 - self.b + self.b * self.doc_lengths[i] / self.avgdl)
                )
            scores.append(score)
        top_indices = np.argsort(scores)[-top_k:][::-1]
        return [
            {
                "doc_id": f"doc_{i}",
                "text": f"Document {i}",
                "score": float(scores[i]),
            }
            for i in top_indices
        ]

class HybridSearchService:
    def __init__(self):
        self.vector_db = SimpleVectorDB()
        self.bm25 = SimpleBM25()

    def search(self, query: HybridQuery) -> HybridResponse:
        import time
        start = time.time()

        # Simulated search
        results = [
            HybridResult(
                doc_id=f"doc_{i}",
                score=1.0 - i * 0.05,
                dense_score=1.0 - i * 0.05,
                sparse_score=0.8 - i * 0.03,
                text=f"Search result for '{query.query}' #{i+1}",
            )
            for i in range(query.top_k)
        ]

        return HybridResponse(
            results=results,
            alpha=query.alpha,
            total_time_ms=(time.time() - start) * 1000,
        )

hybrid_service = HybridSearchService()

@app.post("/hybrid/search", response_model=HybridResponse)
async def hybrid_search(query: HybridQuery):
    return hybrid_service.search(query)
```

## 10. Folder Structure
```
hybrid-search/
├── src/
│   ├── __init__.py
│   ├── main.py
│   ├── fusion/
│   │   ├── __init__.py
│   │   ├── rrf.py
│   │   ├── combsum.py
│   │   └── weighted.py
│   ├── dense/
│   │   ├── __init__.py
│   │   ├── embedder.py
│   │   └── vector_client.py
│   ├── sparse/
│   │   ├── __init__.py
│   │   ├── bm25.py
│   │   └── indexer.py
│   └── models/
│       ├── __init__.py
│       └── schemas.py
├── tests/
├── deploy/
└── pyproject.toml
```

## 11. Configuration
```yaml
hybrid_search:
  alpha: 0.5        # 0=sparse, 1=dense
  fusion: rrf        # rrf | combsum | weighted
  rrf_k: 60
  dense:
    top_k: 50
    model: text-embedding-3-small
  sparse:
    top_k: 50
    algorithm: bm25
    bm25_k1: 1.5
    bm25_b: 0.75
```

## 12. Flowchart
```
Start
  |
  v
Receive Query
  |
  v
[Parallel Branch]
  |              |
  v              v
[Dense Path]   [Sparse Path]
  |              |
  v              v
[Embed Query]  [Tokenize Query]
  |              |
  v              v
[ANN Search]   [BM25 Scoring]
  |              |
  v              v
[Top-K Dense]  [Top-K Sparse]
  |              |
  +-------+------+
          |
          v
[Score Normalization]
          |
          v
[Fusion (RRF / Weighted)]
          |
          v
[Sort & Return Top-N]
          |
          v
         End
```

## 13. Sequence Diagram
```
Client     HybridSvc     Embedder     VectorDB     BM25     Fuser
  |            |            |            |          |         |
  |--Query---->|            |            |          |         |
  |            |            |            |          |         |
  |            |---embed--->|            |          |         |
  |            |<--vec------|            |          |         |
  |            |            |            |          |         |
  |            |---dense--------------->|          |         |
  |            |            |            |          |         |
  |            |---sparse------------------------->|         |
  |            |            |            |          |         |
  |            |            |            |          |         |
  |            |---fuse_results--------------------->|         |
  |            |            |            |          |         |
  |<--Results--|            |            |          |         |
```

## 14. Pros
- Combines precision of keyword search with recall of semantic search
- Robust to both exact-match and paraphrase queries
- Simple to implement with existing BM25 + vector DB infrastructure
- No training required — works out of the box with any embedding model
- The alpha parameter provides a tunable knob for domain adaptation

## 15. Cons
- Doubles retrieval latency (two searches instead of one)
- Score normalization across disparate scoring functions is tricky
- RRF is heuristic — doesn't learn optimal weights from data
- Adds operational complexity (two index systems to maintain)
- Can return duplicate results if not deduplicated properly

## 16. Alternatives
| Method | Description |
|--------|-------------|
| Learned Sparse (SPLADE) | Neural sparse retrieval — trained term weighting |
| ColBERT | Late interaction — dense but with token-level matching |
| Sparse-Dense Co-training | Jointly optimize both retrievers with shared loss |
| Query2Doc | Expand query with LLM-generated relevant text, then BM25 |

## 17. Performance Considerations
- **Dense retrieval dominates latency** if vector DB is slow — optimize ANN index (HNSW ef_construction).
- **BM25 is nearly instant** (<10ms for millions of docs) — sparse cost is negligible.
- **Fusion overhead** is O(N log N) for N = dense_k + sparse_k — fine for top-100.
- **Score normalization** (min-max, z-score) adds <1ms but is critical for quality.
- **Prefetching:** Run dense and sparse searches in parallel via `asyncio.gather`.

## 18. Scaling to Millions
- **Shard dense index** by document collection across multiple vector DB instances.
- **Distribute BM25 index** across Elasticsearch cluster nodes (10+ shards).
- **Fusion service is stateless** — horizontally scalable behind a load balancer.
- **Cache frequent query embeddings** to avoid re-embedding popular queries.
- **Pre-compute and store normalized scores** during ingestion to avoid online normalization.

## 19. Failure Scenarios
| Failure | Impact | Mitigation |
|---------|--------|------------|
| Vector DB down | Dense path fails | Fallback to sparse-only mode |
| BM25 index stale | Sparse results miss recent docs | Near-real-time indexing |
| Score normalization drift | Poor fusion ranking | Periodic validation against golden set |
| OOM on large result sets | Fusion fails | Cap top_k per retriever at 200 |

## 20. Security
- Sanitize query input before passing to search engines (injection prevention).
- Apply metadata-based access control to BOTH retriever results before fusion.
- Log both dense and sparse paths separately for auditability.
- Rate-limit hybrid endpoint per user to prevent abuse of two simultaneous searches.

## 21. Monitoring
- **Per-path latency:** dense_latency, sparse_latency, fusion_latency
- **Per-path recall:** recall@k for dense vs sparse vs hybrid
- **Alpha usage distribution:** what alpha values do users/teams use?
- **Fusion quality:** NDCG@10 of hybrid vs best single-path
- **Error rate:** dense_failure_rate, sparse_failure_rate

## 22. Interview Questions
| Level | Question | Key Points |
|-------|----------|------------|
| L3 | Explain hybrid search | Dense + sparse, each covers the other's gaps |
| L4 | How does RRF work? | 1/(k+rank) per set, sum per doc, sort descending |
| L4 | How would you set the alpha parameter? | Grid search on validation set, or A/B test |
| L5 | How would you handle score normalization? | Min-max, z-score, or rank-based normalization |
| L5 | Design a hybrid search system for 100M docs | Sharded VectorDB + ES cluster, fusion layer |

## 23. Cheat Sheet
```python
# Hybrid search in 5 lines
dense_results = await vector_db.search(embed(query), top_k=50)
sparse_results = await bm25.search(query, top_k=50)
all_docs = {r.doc_id: r for r in dense_results + sparse_results}
for rank, r in enumerate(dense_results):
    all_docs[r.doc_id].score += 1 / (60 + rank + 1)
sorted(all_docs.values(), key=lambda x: x.score, reverse=True)[:20]
```

## 24. Common Mistakes
- **Not normalizing scores** before fusion — BM25 scores (0-20+) dominate cosine (0-1).
- **Using RRF without deduplication** — same document from both paths gets double-counted.
- **Hardcoding alpha** — optimal balance varies per query type; expose it per-request.
- **Synchronous dense + sparse** — running sequentially doubles latency; always parallelize.
- **Ignoring early termination** — dense path may return fewer than top_k results (empty collections).

## 25. Production Best Practices
1. **Always normalize scores** before fusion — use min-max or rank-based normalization.
2. **Run dense and sparse searches in parallel** using asyncio.gather — never sequentially.
3. **Use RRF over weighted fusion** unless you have a validation set to tune alpha.
4. **Deduplicate by doc_id** before fusion — same document may appear in both result sets.
5. **Expose alpha per-request** — different query types need different dense/sparse balance.
6. **Monitor per-path recall** separately — hybrid masks degradation in one path.
7. **Cache query embeddings** to skip the dense path for repeated queries.
8. **Fall back to sparse-only** if the vector DB is degraded — always have a fallback.
