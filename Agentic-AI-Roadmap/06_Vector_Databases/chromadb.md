# ChromaDB

## 1. What is it?
ChromaDB is an open-source, embeddable vector database designed specifically for AI applications. It provides a lightweight, developer-friendly way to store, manage, and query vector embeddings with built-in metadata filtering and full-text search capabilities.

## 2. Why do we need it?
Building LLM applications requires semantic search over embeddings. ChromaDB eliminates the operational overhead of running a separate database service — it embeds directly into your Python application, making it ideal for prototyping, development, and small-to-medium production workloads.

## 3. Real-world Example
A startup building a legal document Q&A system uses ChromaDB to index 50,000 legal contracts. Each contract is chunked, embedded via `text-embedding-ada-002`, and stored with metadata (jurisdiction, date, parties involved). Queries like "find all force majeure clauses in California contracts" run in under 200ms.

## 4. Architecture Diagram (ASCII)

```
+------------------+       +------------------+
|   Application    |       |   Embedding API  |
|   (Python/Django)|       |   (OpenAI/Ollama)|
+--------+---------+       +--------+---------+
         |                          |
         v                          v
+--------+---------+                |
|   ChromaDB SDK   |<---------------+
|   (In-process)   |
+--------+---------+
         |
         v
+--------+---------+
|   DuckDB/LMDB    |
|   (Storage Layer)|
+--------+---------+
         |
         v
+--------+---------+
|   Local Disk      |
|   (./chroma/*)    |
+------------------+
```

## 5. Internal Working
ChromaDB uses DuckDB as its metadata store and LMDB for vector index storage. On write, embeddings and metadata are persisted to disk. On query, HNSW (Hierarchical Navigable Small World) index performs approximate nearest neighbor search. The client-server mode wraps these operations behind a FastAPI HTTP server.

## 6. Production Flow

```
User Query -> Embed -> ChromaDB query() -> HNSW search -> Metadata filter -> Ranked results -> LLM context
```

## 7. HLD
- **Client Layer**: Python/JS SDK — CRUD operations on collections
- **Index Layer**: HNSW for ANN, Brute-force for exact search (small datasets)
- **Storage Layer**: DuckDB (metadata), LMDB (vectors), Parquet (durability)
- **API Layer** (optional): FastAPI server for remote access

## 8. LLD
- Collection = namespace with embedding function + distance metric (cosine, l2, ip)
- Document → `{id, embedding, metadata, document}`
- Query → `{n_results, where, where_document, include}`
- Index: In-memory HNSW graph with LMDB checkpointing every 1000 writes

## 9. Python Implementation

```python
import chromadb
from chromadb.config import Settings

client = chromadb.PersistentClient(
    path="./chroma_db",
    settings=Settings(anonymized_telemetry=False)
)

collection = client.create_collection(
    name="contracts",
    metadata={"hnsw:space": "cosine"}
)

collection.add(
    documents=["Force majeure clause...", "Indemnification..."],
    metadatas=[
        {"jurisdiction": "California", "date": "2024-01-01"},
        {"jurisdiction": "New York", "date": "2024-02-15"},
    ],
    ids=["doc1", "doc2"]
)

results = collection.query(
    query_texts=["unexpected events liability"],
    n_results=5,
    where={"jurisdiction": "California"}
)

print(results["documents"][0])
```

## 10. Folder Structure

```
chroma_db/
├── collections/
│   └── contracts/
│       ├── index/          # HNSW index files
│       ├── metadata.bin    # DuckDB metadata store
│       └── vectors.bin     # LMDB vector storage
├── ts/                     # Tenant separation
└── system/                 # System metadata
```

## 11. Configuration

```python
Settings(
    chroma_db_impl="duckdb+parquet",
    persist_directory="./chroma_db",
    anonymized_telemetry=False,
    allow_reset=True,
    is_persistent=True,
)
```

## 12. Flowchart

```
Start
  |
  v
Create/Get Collection
  |
  v
Add Documents + Embeddings
  |
  v
HNSW Index Built
  |
  v
User Query (text/embedding)
  |
  v
Embed (if text provided)
  |
  v
ANN Search + Metadata Filter
  |
  v
Return Ranked Results
  |
  v
LLM Generates Answer
```

## 13. Sequence Diagram

```
Client          ChromaDB        DuckDB          HNSW
  |                |               |              |
  |--add(docs)---->|               |              |
  |                |--embed()----->|              |
  |                |--store(idx)-->|              |
  |                |--save(vec)------------------>|
  |                |               |              |
  |--query(text)-->|               |              |
  |                |--search()------------------>|
  |                |--filter()---->|              |
  |<---results-----|               |              |
```

## 14. Pros
- Zero operational overhead (embeddable); Fast prototyping; Pythonic API with native type hints; Automatic embedding function integration with OpenAI/Sentence-Transformers; Lightweight (pip install only); Metadata + document filtering.

## 15. Cons
- Not designed for billion-scale (10M+ vectors degrade); Single-node only; No built-in sharding; Limited query language compared to Qdrant/Pinecone; No distributed deployments; Write throughput limited by single process.

## 16. Alternatives
- **Qdrant**: Better for ~100M scale with filtering; **Pinecone**: Fully managed, no ops; **Weaviate**: Object storage + vector search combined; **Milvus**: Billion-scale distributed; **FAISS**: Raw library, no persistence.

## 17. Performance Considerations
- HNSW `ef_construction` (100-200) and `M` (16-48) control index quality vs build speed; Batch writes in 1000-doc chunks; Use cosine distance for normalized embeddings; Metadata filtering before vector search reduces candidate pool; Memory-mapped indexes for large datasets; Prefer `query()` over `get()` for ranked results.

## 18. Scaling to Millions
ChromaDB hits bottlenecks around 10M vectors. Mitigations: Shard manually by collection per tenant; Use client-server mode with read replicas; Offload to DuckDB directly for analytical queries; Accept 200-500ms latency at 80% recall; Migrate to Qdrant/Milvus beyond 10M.

## 19. Failure Scenarios
- **Corrupt index**: LMDB corruption on unclean shutdown → restore from Parquet checkpoint
- **OOM on large insert**: Chroma loads index in memory → batch inserts to 5000 docs
- **Disk full**: DuckDB fails silently → monitor `persist_directory` disk usage via `df -h`
- **Version mismatch**: ChromaDB stores version in metadata → pin version in `requirements.txt`

## 20. Security
- No built-in authentication (add reverse proxy for client-server mode)
- Encrypt persistence directory at filesystem level (LUKS/bitlocker)
- Client-server mode supports HTTPS via reverse proxy
- Sanitize `where` filters to prevent injection (Chroma uses parameterized queries)
- Isolate collections per tenant using separate `PersistentClient` instances

## 21. Monitoring
- Track `collection.count()` over time for growth
- Monitor query latency via `@timed` decorator
- Log `n_results` returned vs requested to detect data drift
- Watch disk IO on `persist_directory` via `iostat`
- Export metrics to Prometheus via custom middleware in client-server mode

## 22. Interview Questions
1. *How does ChromaDB differ from traditional databases?* — It's optimized for vector similarity search, not exact match or range queries.
2. *Explain HNSW index in ChromaDB.* — Multi-layer navigable small world graph for approximate nearest neighbor search.
3. *How to handle multi-tenancy in ChromaDB?* — Separate collections or separate `PersistentClient` instances per tenant.
4. *What happens when ChromaDB runs out of memory?* — Inserts fail; use batch sizes of 1000 and monitor memory.

## 23. Cheat Sheet

```python
# CRUD
c.add(documents=[], metadatas=[], ids=[])
c.update(ids=[], metadatas=[], documents=[])
c.delete(ids=[])
c.get(where={})

# Query
c.query(query_texts=[], n_results=10, where={})

# Collections
client.list_collections()
client.delete_collection("name")
```

## 24. Common Mistakes
- Not batching inserts (insert one-by-one is 100x slower); Forgetting to set `anonymized_telemetry=False` in production; Using `get()` instead of `query()` for semantic search; Running multiple `PersistentClient` instances on the same directory; Storing raw text in metadata instead of documents field; Not handling `where` conditions type-correctly (string vs int comparison).

## 25. Production Best Practices
- Pin ChromaDB version; Use client-server mode with `chroma run --path /data/chroma`; Set `is_persistent=True`; Configure `EF_CONSTRUCTION=200` for high-recall use cases; Pre-compute embeddings client-side (avoid per-request embed call); Use `where` filters to reduce candidate pool; Separate collections by logical domain; Monitor disk usage daily; Backup Parquet checkpoints to S3; Run health checks via `collection.count()` every 30s.
