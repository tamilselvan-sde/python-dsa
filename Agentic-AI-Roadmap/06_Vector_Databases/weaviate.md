# Weaviate

## 1. What is it?
Weaviate is an open-source vector database that combines vector search with object storage, GraphQL API, and built-in AI modules for vectorization and generation. It stores both vectors and objects together, eliminating the need for a separate database.

## 2. Why do we need it?
Most RAG systems require synchronizing data between a vector store and a primary database. Weaviate eliminates this dual-write problem by storing objects and vectors in a single system, offering native GraphQL, multi-modal search (text, image, audio), and out-of-the-box AI modules.

## 3. Real-world Example
**Medium** uses Weaviate for its article recommendation engine. With 100M+ articles, each stored with its embedding, metadata (tags, author, read time), and full content, Weaviate serves personalized recommendations by combining vector similarity with GraphQL filters like "articles from authors I follow, published in the last week, tagged with AI."

## 4. Architecture Diagram (ASCII)

```
+---------------------------+
|       Application         |
|   (GraphQL/REST/gRPC)     |
+------------+--------------+
             |
             v
+------------+--------------+
|    Weaviate Cluster        |
|                            |
|  +----------------------+  |
|  |   API Layer           |  |
|  | (GraphQL, REST, gRPC) |  |
|  +-----------+----------+  |
|              |              |
|  +-----------+----------+  |
|  |   Module System       |  |
|  | (text2vec, img2vec,   |  |
|  |  generative, ner, qna)|  |
|  +-----------+----------+  |
|              |              |
|  +-----------+----------+  |
|  |   Vector Index        |  |
|  |   (HNSW, multi-ten.)  |  |
|  +-----------+----------+  |
|              |              |
|  +-----------+----------+  |
|  |   Object Store        |  |
|  |   (LSM-Tree, RocksDB) |  |
|  +-----------+----------+  |
|              |              |
|  +-----------+----------+  |
|  |   Replication/Shard   |  |
|  |   (Raft-based)        |  |
|  +-----------+----------+  |
+---------------------------+
```

## 5. Internal Working
Weaviate stores each object as a JSON document with an associated vector. The vector index uses HNSW or flat indexing. The object store is LSM-tree backed by RocksDB. Writes go to a WAL, then to the mutable MemTable, then flushed to SSTables. Modules (text2vec, generative) run in-process, calling external AI APIs to vectorize or generate content automatically.

## 6. Production Flow

```
Import document -> Weaviate module auto-vectorizes -> Object + vector stored
-> HNSW index updated

User query -> GraphQL query -> Module vectorizes query -> HNSW search
-> Object retrieval -> Optional generative module -> Results returned
```

## 7. HLD
- **API Layer**: GraphQL (primary), REST (admin), gRPC (bulk); TLS termination via reverse proxy
- **Module Layer**: Pluggable AI modules — text2vec-openai, text2vec-cohere, img2vec-neural, generative-openai, ner-transformers, qna-transformers
- **Index Layer**: HNSW with configurable efConstruction, maxConnections, ef
- **Storage Layer**: LSM-tree (RocksDB) for objects, inverted index for BM25, HNSW for vectors
- **Cluster Layer**: Raft-based replication, automatic sharding via hash ring

## 8. LLD
- Class = schema definition (like a DB table): `{class: "Article", properties: [{name: "title", dataType: ["text"]}]}`
- Object = instance with properties and vector: `{class: "Article", properties: {title: "..."}, vector: [...]}`
- Modules configured at class level: `{class: "Article", vectorizer: "text2vec-openai", moduleConfig: {}}`
- Consistency: tunable per request (ONE, QUORUM, ALL)

## 9. Python Implementation

```python
import weaviate
import weaviate.classes as wvc

client = weaviate.connect_to_wcs(
    cluster_url="https://my-cluster.weaviate.cloud",
    auth_credentials=weaviate.auth.AuthApiKey("wvc-xxx"),
)

client.collections.create(
    name="Article",
    vectorizer_config=wvc.config.Configure.Vectorizer.text2vec_openai(
        model="text-embedding-3-small",
    ),
    generative_config=wvc.config.Configure.Generative.openai(
        model="gpt-4o",
    ),
    properties=[
        wvc.config.Property(name="title", data_type=wvc.config.DataType.TEXT),
        wvc.config.Property(name="content", data_type=wvc.config.DataType.TEXT),
        wvc.config.Property(name="author", data_type=wvc.config.DataType.TEXT),
        wvc.config.Property(name="published_at", data_type=wvc.config.DataType.DATE),
    ],
)

collection = client.collections.get("Article")
collection.data.insert({
    "title": "RAG in Production",
    "content": "How to deploy RAG systems at scale...",
    "author": "Jane Doe",
    "published_at": "2024-06-15",
})

response = collection.query.near_text(
    query="production RAG deployment",
    limit=10,
    return_metadata=wvc.query.MetadataQuery(certainty=True),
    filters=wvc.query.Filter.by_property("author").equal("Jane Doe"),
)

for obj in response.objects:
    print(obj.properties["title"], obj.metadata.certainty)
```

## 11. Configuration

```yaml
# docker-compose.yml
services:
  weaviate:
    image: cr.weaviate.io/semitechnologies/weaviate:1.28
    environment:
      QUERY_DEFAULTS_LIMIT: 100
      AUTHENTICATION_APIKEY_ENABLED: "true"
      AUTHENTICATION_APIKEY_ALLOWED_KEYS: "wvc-readonly,wvc-admin"
      AUTHENTICATION_APIKEY_USERS: "reader,admin"
      AUTHORIZATION_ADMINLIST_ENABLED: "true"
      AUTHORIZATION_ADMINLIST_USERS: "admin"
      PERSISTENCE_DATA_PATH: "/var/lib/weaviate"
      CLUSTER_HOSTNAME: "node-1"
      CLUSTER_GOSSIP_BIND_PORT: "7100"
      CLUSTER_DATA_BIND_PORT: "7101"
      ENABLE_MODULES: "text2vec-openai,generative-openai"
      DEFAULT_VECTORIZER_MODULE: "text2vec-openai"
```

## 12. Flowchart

```
Define Schema (Class with properties + vectorizer)
  |
  v
Insert Object (Weaviate calls text2vec module automatically)
  |
  v
Module generates embedding -> Object + vector stored in LSM-tree
  |
  v
HNSW index updated asynchronously
  |
  v
Query: nearText / nearVector / hybrid
  |
  v
If nearText: module vectorizes query
  |
  v
HNSW search -> Optional BM25 (hybrid) -> Object retrieval
  |
  v
Optional generative module for answer generation
  |
  v
Return results (objects + certainty scores)
```

## 13. Sequence Diagram

```
Client            Weaviate API      Module(HuggingFace/OpenAI)    HNSW    RocksDB
  |                    |                       |                    |        |
  |--GraphQL query---->|                       |                    |        |
  |                    |                       |                    |        |
  |--nearText--------->|                       |                    |        |
  |                    |--vectorize query------|                    |        |
  |                    |<--embedding-----------|                    |        |
  |                    |                                            |        |
  |                    |--HNSW search------------------------------>|        |
  |                    |<--candidate IDs----------------------------|        |
  |                    |                                            |        |
  |                    |--get objects by ID--------------------------------->|
  |                    |<--objects and vectors-------------------------------|
  |                    |                                            |        |
  |                    |--generative (optional)--------|            |        |
  |                    |<--generated answer------------|            |        |
  |                    |                                            |        |
  |<--GraphQL response|                                            |        |
```

## 14. Pros
- Single system for objects + vectors (no dual-write); Built-in AI modules (no `text-embedding-ada-002` SDK needed); GraphQL-native (elegant for frontend teams); Hybrid search (vector + BM25 keyword); Multi-modal vectorization (text, image, audio); Automatic schema inference; Multi-tenancy support; Strong consistency option.

## 15. Cons
- Higher per-vector storage overhead (~2x vs raw HNSW); Module calls increase p99 latency (external API dependency); GraphQL query complexity can hurt performance; Limited GPU acceleration (CPU-only indexing); Smaller community than Milvus/Qdrant; Schema-on-write means migration is explicit.

## 16. Alternatives
- **Qdrant**: Faster raw vector search, Rust-based; **Milvus**: Billion-scale with GPU; **ChromaDB**: Simpler, lighter for dev; **Pinecone**: Fully managed alternative; **Supabase pgvector**: If you're already on Postgres.

## 17. Performance Considerations
- Use batch import (1000 objects/batch) for 10x throughput; Set `vectorCacheMaxObjects` for hot data; Hybrid search adds 20-40% latency vs pure vector; Limit `nearText` per-query API calls by pre-vectorizing; HNSW `efConstruction=128`, `maxConnections=32` for production; Shard by tenant for multi-tenant workloads.

## 18. Scaling to Millions
- **10M objects** ~ 30GB (768-dim vectors + objects): 3-5 nodes with 16GB RAM each; **100M**: 10-15 nodes, 32GB RAM, shard by class; Use multi-tenancy instead of separate classes; Tiered storage: keep hot objects in RAM, cold on SSD via RocksDB cache; Compaction runs automatically — monitor `weaviate_compaction` metrics.

## 19. Failure Scenarios
- **Raft split-brain**: Network partition leads to two leaders — resolved by Weaviate's lease-based system; **Module API throttling**: OpenAI rate limits cause timeouts — configure retries with exponential backoff; **RocksDB corruption**: Restore from latest backup snapshot; **OOM**: Set `PERSISTENCE_MEMTABLE_SIZE_MB` and `PERSISTENCE_ROCKSDB_BLOCK_CACHE_SIZE` appropriately; **Disk full**: Weaviate returns 507 (Insufficient Storage) and rejects writes.

## 20. Security
- API key authentication (multiple keys with different roles); Admin list for privilege separation; TLS for all API endpoints; Encryption at rest (filesystem-level); Audit logging via `LOG_LEVEL=debug` to stdout → ELK; Network isolation — Weaviate binds to private IP in production; Object-level permissions can be implemented via query middleware; Regular CVE scanning on Weaviate image.

## 21. Monitoring
- Prometheus metrics: `weaviate_requests_total`, `weaviate_request_duration_ms`, `weaviate_hnsw_latency`; Grafana WCS dashboard available; Track `weaviate_objects_count` per class for growth; Monitor `weaviate_module_inference_latency_ms` for module bottlenecks; Alert on 5xx rate > 1%; Watch RocksDB compaction lag; Set log level to `error` in production.

## 22. Interview Questions
1. *Explain how Weaviate auto-vectorization works.* — The text2vec module intercepts writes, calls an embedding API synchronously, stores the vector alongside the object.
2. *What are the tradeoffs of using Weaviate's built-in modules vs pre-vectorizing?* — Modules simplify code but add latency; pre-vectorizing is faster but requires extra orchestration.
3. *How does hybrid search combine vector and keyword search?* — Using `alpha` parameter (0=vector only, 1=BM25 only), fusion via Reciprocal Rank Fusion (RRF).
4. *What consistency levels does Weaviate support and when to use each?* — ONE (fast, eventual), QUORUM (balanced), ALL (strong, slow).

## 23. Cheat Sheet

```graphql
# Create schema
{
  "class": "Article",
  "vectorizer": "text2vec-openai",
  "properties": [{"name": "title", "dataType": ["text"]}]
}

# Query
{
  Get {
    Article(nearText: {concepts: ["AI production"]},
            limit: 10) {
      title
      _additional { certainty }
    }
  }
}
```

```python
# Python
collection.data.insert({...})
collection.query.near_text(query="...", limit=N)
collection.query.hybrid(query="vector semantic", alpha=0.5)
collection.aggregate.over_all(total_count=True)
```

## 24. Common Mistakes
- Enabling too many modules per class (slows writes); Using `nearText` without caching embeddings; Setting `vectorCacheMaxObjects` too low (increases disk reads); Not enabling multi-tenancy for SaaS workloads; Forgetting GraphQL query complexity limits; Over-using `_additional {certainty}` in every query; Not pinning Weaviate Docker image version.

## 25. Production Best Practices
- Use Weaviate Cloud Services (WCS) for managed SLA; Pin Docker image version; Use batch import for bulk loads; Enable multi-tenancy for SaaS; Configure resource limits for modules (OpenAI API timeouts); Set `BACKUP_FILESYSTEM_PATH` for periodic backups to S3; Use gRPC for bulk operations (10x faster); Monitor module inference latency separately; Implement query timeout middleware; Test module API failovers (rate limiting, outage).
