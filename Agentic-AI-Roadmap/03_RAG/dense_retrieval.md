# Dense Retrieval

> Author: **Tamilselvan** · ✉️ tamilselvan.sde@gmail.com · 🔗 [LinkedIn](https://www.linkedin.com/in/tamilselvan-ai/)
>

## 1. What is it?
**ELI5:** Dense retrieval is like using a really smart librarian who doesn't just look for exact words — they understand the meaning behind your question. If you ask for "vehicles that run on electricity," they'll find documents about "EVs" and "electric cars" even though none of those exact words match. They do this by mentally converting everything into a "thought vector" and finding the closest matches.

**Technical Definition:** Dense retrieval maps queries and documents into a shared dense vector space using neural network encoders (typically BERT-based transformer models). Similarity is measured via cosine distance or dot product between the query embedding and document embeddings. Documents are pre-embedded and stored in a vector database; at query time, the query is embedded and Approximate Nearest Neighbor (ANN) search returns the top-K closest document vectors.

## 2. Why do we need it?
- **Semantic understanding:** Captures meaning, not just keyword overlap. "Renal failure" matches "kidney disease."
- **Multilingual retrieval:** Embeddings capture cross-lingual semantics — English query retrieves relevant French documents without translation.
- **Handles synonyms naturally:** "car," "automobile," "vehicle," "sedan" all map to nearby regions in the embedding space.
- **Beyond exact match:** Works for paraphrases, rephrasings, and typo-tolerant matching.
- **Foundation for hybrid:** Dense retrieval is the semantic half of hybrid search; without it, you're stuck with keyword-only.

## 3. Real-world Example
- **Pinecone / Weaviate / Qdrant:** Every major vector DB implements dense retrieval as their core offering. Used by Jasper AI, Notion, Gong for semantic document search.
- **Google's MUM (Multitask Unified Model):** Powers Google Search's neural retrieval for complex queries — "What do I need to do to prepare for climbing Mt. Fuji?" returns semantically relevant climbing guides.
- **Facebook's Faiss:** Meta's ANN library powers dense retrieval across billions of vectors for Facebook Search and Instagram Reels recommendations.
- **Cohere Embed:** Cohere's embedding API is used by enterprises for semantic search across legal contracts and technical documentation.
- **Azure Cognitive Search:** Built-in dense vector search using Azure OpenAI embeddings.

## 4. Architecture Diagram (ASCII)
```
+-----------+
| Document  |
+-----+-----+
      |
      v
[Embedding Model]  (e.g., BERT, text-embedding-3)
      |
      v
[Vector DB]
  | doc1: [0.1, 0.3, ..., 0.8]
  | doc2: [0.9, 0.2, ..., 0.1]
  | doc3: [0.4, 0.7, ..., 0.2]
  +------------------+
                     |
+--------+           |
| Query  |           |
+---+----+           |
    |                |
    v                |
[Embedding Model]    |
    |                |
    v                v
[Query Vec] ------>[ANN Search]
    |                |
    |         [Top-K Results]
    |                |
    +---cosine-------+
     similarity
```

## 5. Internal Working
1. **Document embedding (offline):** Each document chunk is passed through a transformer encoder (e.g., `sentence-transformers/all-MiniLM-L6-v2`). The output CLS token or mean-pooled token embeddings form a fixed-size vector (384, 768, 1536 dimensions).
2. **Indexing (offline):** All document vectors are stored in a vector DB with an ANN index (HNSW, IVF, PQ). The index organizes vectors for sub-linear search.
3. **Query embedding (online):** The user query is passed through the same embedding model to produce a query vector in the same latent space.
4. **ANN search (online):** The query vector is compared against the index using a distance metric. For cosine similarity, the vectors are normalized and inner product is computed.
5. **Result ranking:** The top-K documents with highest similarity scores are returned.

**Key concept — embedding models produce a distributional representation:** Each dimension captures some latent semantic feature. Two documents are "similar" when their dimension values covary — meaning they activate the same learned patterns in the model.

## 6. Production Flow
```
Ingestion Pipeline:
  Raw Docs → Chunk → Embed (batch, GPU) → Normalize → Index (HNSW) → Persist

Query Pipeline:
  User Query → Embed (GPU/CPU) → Normalize → ANN Search → Top-K Chunks
  
Scoring:
  cosine_sim(q, d) = dot(q/|q|, d/|d|)
  or
  dot_product(q, d) (for normalized vectors, these are equivalent)
```

## 7. HLD
```
                 +----------------------+
                 | Embedding Service    |
                 | (GPU cluster, ONNX)  |
                 +----------+-----------+
                            |
               +------------+------------+
               |                         |
    +----------v-----------+  +----------v-----------+
    | Vector DB Primary    |  | Vector DB Replica     |
    | (HNSW index, writes) |  | (read-only, queries)  |
    +----------+-----------+  +----------+-----------+
               |                         |
               +------------+------------+
                            |
                 +----------v-----------+
                 | ANN Search Service   |
                 | (top-K retrieval)     |
                 +----------+-----------+
                            |
                 +----------v-----------+
                 | RAG Orchestrator     |
                 +----------------------+
```

## 8. LLD
```python
import numpy as np
from typing import List, Optional
from dataclasses import dataclass
from enum import Enum

class DistanceMetric(str, Enum):
    COSINE = "cosine"
    DOT_PRODUCT = "dot_product"
    EUCLIDEAN = "euclidean"

@dataclass
class DenseRetrievalConfig:
    embedding_model: str = "sentence-transformers/all-MiniLM-L6-v2"
    embedding_dim: int = 384
    distance_metric: DistanceMetric = DistanceMetric.COSINE
    index_type: str = "HNSW"
    hnsw_m: int = 16
    hnsw_ef_construction: int = 200
    hnsw_ef_search: int = 50

class EmbeddingService:
    def __init__(self, model_name: str, device: str = "cpu"):
        self.model_name = model_name
        self.device = device
        # In production: load SentenceTransformer or OpenAIEmbedding

    async def embed(self, text: str) -> List[float]:
        # Call embedding model API
        return [0.1] * 384  # placeholder

    async def embed_batch(
        self, texts: List[str], batch_size: int = 32
    ) -> List[List[float]]:
        results = []
        for i in range(0, len(texts), batch_size):
            batch = texts[i : i + batch_size]
            results.extend([
                [0.1] * 384 for _ in batch
            ])
        return results

class ANNIndex:
    def __init__(self, config: DenseRetrievalConfig):
        self.config = config
        self.vectors: np.ndarray = np.empty((0, config.embedding_dim))
        self.ids: List[str] = []

    def add(self, doc_id: str, vector: List[float]):
        self.vectors = np.vstack([self.vectors, np.array(vector)])
        self.ids.append(doc_id)

    def search(self, query_vec: List[float], top_k: int = 10) -> List[dict]:
        q = np.array(query_vec).reshape(1, -1)
        # Cosine similarity
        norms = np.linalg.norm(self.vectors, axis=1)
        q_norm = np.linalg.norm(q)
        if q_norm == 0 or np.any(norms == 0):
            return []
        sims = np.dot(self.vectors, q.T).flatten() / (norms * q_norm + 1e-8)
        top_indices = np.argsort(sims)[-top_k:][::-1]
        return [
            {"id": self.ids[i], "score": float(sims[i])}
            for i in top_indices
        ]
```

## 9. Python Implementation
```python
import asyncio
from typing import List, Optional, Tuple

import numpy as np
from fastapi import FastAPI
from pydantic import BaseModel, Field

app = FastAPI(title="Dense Retrieval Service")

class DenseQuery(BaseModel):
    query: str = Field(..., min_length=1)
    top_k: int = Field(default=10, ge=1, le=100)

class DenseResult(BaseModel):
    doc_id: str
    score: float
    text: str
    metadata: dict = {}

class DenseResponse(BaseModel):
    results: List[DenseResult]
    latency_ms: float

class DenseRetriever:
    def __init__(self, dimension: int = 384):
        self.dimension = dimension
        self.documents: List[dict] = []
        self.embeddings: List[np.ndarray] = []

    def add_document(self, doc_id: str, text: str, metadata: dict = None):
        vec = np.random.randn(self.dimension)
        vec = vec / np.linalg.norm(vec)  # unit vector
        self.documents.append({
            "id": doc_id,
            "text": text,
            "metadata": metadata or {},
            "vector": vec,
        })
        self.embeddings.append(vec)

    async def search(self, query_vec: np.ndarray, top_k: int) -> List[dict]:
        if len(self.embeddings) == 0:
            return []
        stack = np.array(self.embeddings)
        query_vec = query_vec / (np.linalg.norm(query_vec) + 1e-8)
        scores = np.dot(stack, query_vec)
        top_indices = np.argsort(scores)[-top_k:][::-1]
        return [
            {
                **self.documents[i],
                "score": float(scores[i]),
            }
            for i in top_indices
        ]

retriever = DenseRetriever()

# Pre-load some documents
for i in range(1000):
    retriever.add_document(
        doc_id=f"doc_{i}",
        text=f"This is sample document {i} about AI and machine learning.",
    )

@app.post("/dense/search", response_model=DenseResponse)
async def dense_search(query: DenseQuery):
    import time
    start = time.time()

    query_vec = np.random.randn(retriever.dimension)
    query_vec = query_vec / np.linalg.norm(query_vec)

    results = await retriever.search(query_vec, query.top_k)

    return DenseResponse(
        results=[
            DenseResult(
                doc_id=r["id"],
                score=r["score"],
                text=r["text"],
                metadata=r["metadata"],
            )
            for r in results
        ],
        latency_ms=(time.time() - start) * 1000,
    )

@app.post("/dense/bulk_add")
async def bulk_add(docs: List[dict]):
    for doc in docs:
        retriever.add_document(
            doc_id=doc["id"],
            text=doc["text"],
            metadata=doc.get("metadata"),
        )
    return {"status": "ok", "count": len(docs)}
```

## 10. Folder Structure
```
dense-retrieval/
├── src/
│   ├── __init__.py
│   ├── main.py
│   ├── embedder.py
│   ├── index/
│   │   ├── __init__.py
│   │   ├── hnsw.py
│   │   ├── ivf.py
│   │   └── factory.py
│   ├── models/
│   │   └── schemas.py
│   └── client.py
├── tests/
├── deploy/
└── pyproject.toml
```

## 11. Configuration
```yaml
dense_retrieval:
  embedding:
    model: sentence-transformers/all-MiniLM-L6-v2
    dimension: 384
    device: cuda  # cpu | cuda
    batch_size: 64

  index:
    type: hnsw  # hnsw | ivf | pq
    hnsw:
      m: 16
      ef_construction: 200
      ef_search: 50
    ivf:
      nlist: 4096
      nprobe: 10

  search:
    metric: cosine  # cosine | dot | l2
    top_k_default: 10
    top_k_max: 200
```

## 12. Flowchart
```
Start
  |
  v
[Input Query]
  |
  v
[Tokenize + Encode (Transformer)]
  |
  v
[Query Embedding Vector]
  |
  v
[ANN Search on Index]
  |
  +---> HNSW: traverse graph layers
  |
  +---> IVF: search nearest centroids
  |
  v
[Candidate vectors + Distances]
  |
  v
[Sort by Similarity (desc)]
  |
  v
[Return Top-K Results]
  |
  End
```

## 13. Sequence Diagram
```
Client      API         Embedder     VectorDB     ANN Index
  |          |             |            |            |
  |--Query-->|             |            |            |
  |          |--embed----->|            |            |
  |          |<--vector----|            |            |
  |          |             |            |            |
  |          |--search----------------->|            |
  |          |             |            |--ANN------>|
  |          |             |            |<--candidates|
  |          |<--top-k------------------|            |
  |<--Results|             |            |            |
```

## 14. Pros
- Captures semantic meaning beyond keyword overlap
- Handles synonyms, paraphrases, and typos naturally
- Works cross-lingually with multilingual embedding models
- Enables zero-shot retrieval — no task-specific training needed
- Foundation for RAG, hybrid search, and semantic caching

## 15. Cons
- Expensive to embed millions of documents (GPU cost)
- Requires vector DB infrastructure (operational overhead)
- ANN search is approximate — can miss relevant results (recall < 100%)
- Embedding model has its own knowledge cutoff and biases
- Poor at exact-match retrieval (product codes, serial numbers)
- Cold start: new documents not retrievable until indexed and embedded

## 16. Alternatives
| Method | Description |
|--------|-------------|
| Sparse retrieval (BM25) | Keyword-based, exact match, no semantic understanding |
| Learned sparse (SPLADE) | Neural term weighting, combines dense quality with sparse interpretability |
| ColBERT | Late interaction model — more compute at search time but higher accuracy |
| Hybrid (dense + sparse) | Combination of both — best of both worlds |

## 17. Performance Considerations
- **Embedding latency:** ~10-50ms per query via GPU, ~100-500ms on CPU. Use GPU for production.
- **ANN search latency:** HNSW: 1-10ms at 1M vectors. IVF: 10-50ms. PQ: <1ms but lower recall.
- **Memory:** 1M vectors × 384 dimensions × 4 bytes (float32) = ~1.5GB. HNSW graph adds ~2x overhead.
- **Index build time:** HNSW: O(N log N). 10M vectors takes ~30 minutes on a 16-core machine.
- **Recall-speed tradeoff:** HNSW ef_search: 50 → ~95% recall @ 10. ef_search: 500 → ~99% but 5x slower.

## 18. Scaling to Millions
- **Shard the vector index** across multiple nodes by document hash or collection.
- **Use product quantization (PQ)** to compress vectors to 1/4 size (4-bit encoding) — reduces memory 4x with <2% recall loss.
- **Distribute embedding** across a GPU cluster with batch inference.
- **Reader-writer separation:** Dedicated nodes for indexing (writes) vs query serving (reads).
- **Cache query embeddings** in Redis — popular queries don't need re-embedding.
- **Async client** — use connection pooling and batch requests to the vector DB.

## 19. Failure Scenarios
| Failure | Impact | Mitigation |
|---------|--------|------------|
| Embedding model degraded | Poor query representations | Fallback to sparse retrieval |
| Vector DB down | Complete retrieval failure | Circuit breaker, fallback to BM25 |
| Index corruption | Wrong/empty results | Daily index snapshots, quick reload |
| OOM during indexing | Process crashes | Batch documents, monitor memory |

## 20. Security
- Sanitize document text before embedding (remove PII, credentials, secrets).
- Use tenant-specific embedding models or add tenant metadata filters post-retrieval.
- Rate-limit embedding API to prevent resource exhaustion.
- Encrypt vector DB indexes at rest.
- Monitor for data poisoning — adversarial documents can manipulate embedding space.

## 21. Monitoring
- **Query embedding latency:** p50, p95, p99
- **ANN search latency:** By index type and top_k
- **Recall@k:** Against ground-truth relevance judgments
- **Index freshness:** Time since last document indexed
- **Memory usage:** Index size / RAM ratio
- **Cache hit rate:** Query embedding cache
- **Error rate:** Embedding failures, search failures

## 22. Interview Questions
| Level | Question | Key Points |
|-------|----------|------------|
| L3 | How does dense retrieval differ from keyword search? | Semantic vs lexical, embedding vs tokens, ANN vs inverted index |
| L4 | Explain how BERT is used for dense retrieval | CLS token pooling or mean pooling, bi-encoder architecture |
| L4 | What's the difference between HNSW and IVF? | HNSW: graph-based, fast search, more memory. IVF: cluster-based, less memory, slower |
| L5 | How would you improve recall for a dense retrieval system? | Query expansion, multi-vector encoding, hybrid with sparse, ensemble embeddings |
| L5 | Design a dense retrieval system for 100M documents | Sharded HNSW, PQ compression, GPU embedding cluster, async replication |

## 23. Cheat Sheet
```python
# Dense retrieval in 5 lines
from sentence_transformers import SentenceTransformer

model = SentenceTransformer("all-MiniLM-L6-v2")
doc_embeddings = model.encode(documents)
query_emb = model.encode(query)
scores = np.dot(doc_embeddings, query_emb) / (
    np.linalg.norm(doc_embeddings, axis=1) * np.linalg.norm(query_emb)
)
top_k = np.argsort(scores)[-10:][::-1]
```

## 24. Common Mistakes
- **Not normalizing vectors** — cosine similarity requires unit vectors for proper scoring.
- **Using different embedding models for indexing and querying** — must be identical.
- **Ignoring the embedding model's max token length** — truncating documents silently loses information.
- **Applying ANN search when brute force is fine** — for <100K vectors, brute force is simpler and 100% recall.
- **Not batching embeddings during indexing** — single-document embedding is 50x slower than batched.
- **Using the wrong distance metric** — some models are trained with dot product, others with cosine.

## 25. Production Best Practices
1. **Always normalize embedding vectors** to unit length — ensures cosine similarity works correctly.
2. **Use the exact same embedding model** for indexing and querying — even a minor version mismatch degrades quality.
3. **Batch embeddings during ingestion** — GPU utilization is ~50x better with batch_size=64+.
4. **Monitor recall@k offline** — maintain a golden query-doc relevance set to detect embedding drift.
5. **Chunk documents at 256-512 tokens** before embedding — long documents produce averaged-out embeddings.
6. **Use HNSW for search, IVF for indexing speed** — HNSW for read-heavy, IVF for write-heavy workloads.
7. **Set ef_search proportional to top_k** — rule of thumb: ef_search = 10 × top_k.
8. **Implement graceful degradation** — degrade to sparse/BL25 if dense service fails or times out.
