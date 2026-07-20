# Qdrant

## 1. What is it?
Qdrant is a high-performance, open-source vector database written in Rust. It provides advanced filtering, payload indexing, and a gRPC/REST API optimized for production AI workloads at scale.

## 2. Why do we need it?
As RAG systems grow beyond prototypes, they need sub-50ms queries over 100M+ vectors with complex filtering. Qdrant delivers this with a focus on reliability — no external dependencies, configurable consistency, and horizontal scaling via sharding.

## 3. Real-world Example
**Discord** uses Qdrant to power its AI-powered search and moderation features. With 150M+ monthly users, Discord indexes millions of messages per day, running vector similarity searches across conversations with complex metadata filters (channel, timestamp, author role) all under 100ms latency.

## 4. Architecture Diagram (ASCII)

```
+------------------+     +------------------+
|   Client SDK     |     |   HTTP/gRPC API  |
| Python/JS/Go/Rust|     |   + Raft Cluster |
+--------+---------+     +--------+---------+
         |                          |
         v                          v
+--------+---------+     +--------+---------+
|   Router Node    |---->|   Shard 1 (Leader)|
|   (Proxy/Coord)  |     +--------+---------+
+--------+---------+              |
         |                        v
         |              +--------+---------+
         +------------->|   Shard 2 (Replica)|
         |              +--------+---------+
         |                        |
         v                        v
+--------+---------+     +--------+---------+
|   Payload Store   |     |  Vector Index    |
|   (RocksDB)       |     |  (HNSW)          |
+------------------+     +------------------+
```

## 5. Internal Working
Qdrant stores vectors in collections. Each collection is split into shards. Each shard maintains an HNSW graph for ANN search and RocksDB for payload metadata. The Raft consensus protocol ensures data consistency across replicas. Queries fan out to all relevant shards, results are merged at the router, and the top-K hits are returned.

## 6. Production Flow

```
Client -> gRPC query -> Router identifies shards -> Parallel HNSW search per shard
-> Payload filter (RocksDB) -> Merge & rank -> Return top-K
```

## 7. HLD
- **API Layer**: gRPC (primary) + REST (fallback) — TLS termination via reverse proxy
- **Router Layer**: Stateless proxy distributing requests to shards
- **Data Layer**: Shared-nothing shards — each owns a subset of vectors
- **Replication**: Raft-based — 3 replicas for fault tolerance
- **Storage**: HNSW (vectors) + RocksDB (payloads) — both local NVMe SSD

## 8. LLD
- Collection config: `{vectors: {size: 1536, distance: Cosine}, shard_number: 6, replication_factor: 3}`
- Point structure: `{id: UUID, vector: [f32; 1536], payload: {key: value}}`
- Query: `{collection: "docs", vector: [...], filter: Must{HasKeyword("status": "active")}, limit: 10}`
- Optimizer: Automatically segments shards for balanced data distribution

## 9. Python Implementation

```python
from qdrant_client import QdrantClient, models

client = QdrantClient(
    url="https://qdrant-cluster.company.com",
    api_key="sk-qdrant-xxx",
    prefer_grpc=True,
)

client.create_collection(
    collection_name="products",
    vectors_config=models.VectorParams(
        size=1536, distance=models.Distance.COSINE
    ),
    shard_number=4,
    replication_factor=3,
)

client.upsert(
    collection_name="products",
    points=[
        models.PointStruct(
            id=1,
            vector=[0.1] * 1536,
            payload={"name": "Ergonomic Chair", "price": 299, "category": "furniture"},
        )
    ],
)

results = client.query_points(
    collection_name="products",
    query=[0.1] * 1536,
    query_filter=models.Filter(
        must=[
            models.FieldCondition(
                key="category",
                match=models.MatchValue(value="furniture"),
            ),
            models.FieldCondition(
                key="price",
                range=models.Range(gte=100, lte=500),
            ),
        ]
    ),
    limit=10,
).points
```

## 10. Folder Structure

```
qdrant_storage/
├── collections/
│   └── products/
│       ├── segments/       # Data segments (optimized periodically)
│       │   ├── segment_0/
│       │   │   ├── vector_index  # HNSW graph
│       │   │   ├── payload_store # RocksDB
│       │   │   └── id_tracker    # Point ID -> segment map
│       │   └── segment_1/
│       └── config.json     # HNSW params, optimizer config
├── raft_state/             # Raft consensus log
├── snapshots/              # Point-in-time snapshots
└── aliases/                # Collection alias mappings
```

## 11. Configuration

```yaml
# config/production.yaml
cluster:
  enabled: true
  node_name: qdrant-node-1
  uri: "https://qdrant-1.company.com:6335"
  p2p:
    port: 6335
storage:
  optimizers:
    default_segment_number: 2
    memmap_threshold_kb: 20000
  hnsw:
    m: 16
    ef_construct: 100
    full_scan_threshold: 10000
service:
  grpc_port: 6334
  http_port: 6333
  max_request_size_mb: 64
```

## 12. Flowchart

```
Client Upsert
  |
  v
Router receives, hashes ID -> Shard assignment
  |
  v
Leader shard writes WAL -> RocksDB (payload) -> HNSW (vector)
  |
  v
Raft replicates to 2 followers
  |
  v
Ack to client (configurable write_consistency_factor)
  |
  ...
Client Query
  |
  v
Router fans out to all shards
  |
  v
Each shard: HNSW search + payload filter + top-K
  |
  v
Router merges, sorts by score, returns global top-K
```

## 13. Sequence Diagram

```
Client          Router          Shard_1(L)    Shard_2(R)    Shard_3(R)
  |               |               |              |              |
  |--upsert------>|               |              |              |
  |               |--hash(id)---->|              |              |
  |               |               |--write WAL-->|              |
  |               |               |--append------|---->---------|
  |               |               |              |              |
  |<---ack--------|               |              |              |
  |               |               |              |              |
  |--query------->|               |              |              |
  |               |--broadcast----|---->---------|---->---------|
  |               |               |--search      |--search      |
  |               |<--top 10------|<-------------|<-------------|
  |               |--merge sort   |              |              |
  |<---results----|               |              |              |
```

## 14. Pros
- Sub-10ms latency for 10M vectors with filters; Written in Rust (memory-safe, fast); No external dependencies (no ZooKeeper, etcd); Built-in payload indexing (B-tree, keyword, geo); Tunable consistency via `write_consistency_factor`; Excellent gRPC performance; Horizontal scaling with zero-downtime rebalancing.

## 15. Cons
- Higher operational complexity than ChromaDB (requires cluster management); Cost scales with shard count; No built-in embedding function (must pre-compute); Python client is a wrapper over gRPC — debugging is harder; Storage cost is higher per vector than FAISS (RocksDB overhead); Requires careful tuning of optimizer segment count.

## 16. Alternatives
- **Pinecone**: Fully managed, simpler to operate; **Weaviate**: GraphQL-native with object storage; **Milvus**: Better for billion-scale with GPU acceleration; **ChromaDB**: Zero-ops prototyping.

## 17. Performance Considerations
- Use gRPC over HTTP (3-5x faster); Set `ef_construct=200` for high recall, `ef=64` for query; Payload index fields used in filters — up to 10x speedup; Batch upserts (256 points per batch) for 5x throughput; Use `memmap_threshold_kb` for datasets > RAM; Optimize HNSW `M` (16 default) — higher = more accuracy, more memory.

## 18. Scaling to Millions
- **100M+ vectors**: 8-16 shards, 3x replication, 16-32 GB RAM per node; **1B+**: 64+ shards, cluster with dedicated router nodes; Use geo-distributed clusters for multi-region; Snapshot-based backup for quick recovery; Re-index periodically to defragment segments.

## 19. Failure Scenarios
- **Node loss**: Raft elects new leader among remaining replicas (10-30s); **Corrupt segment**: Automatic recovery from WAL replay; **Network partition**: Raft minority partition becomes read-only; **OOM**: Set `memmap_threshold_kb` to page vectors to disk; **Disk full**: Qdrant enters read-only mode automatically.

## 20. Security
- TLS for gRPC/HTTP; API key authentication (built-in); Payload-level access control via pre-query filter injection; Network isolation — bind to private VPC; Audit logging of all admin operations; Encryption at rest via filesystem-level encryption.

## 21. Monitoring
- **Prometheus metrics**: `qdrant_points_count`, `qdrant_optimizer_*`, `qdrant_segment_*`; **Latency**: p50/p99 query latency per collection; **Optimizer**: track segment count — high count means need tuning; **Raft**: follower lag in ms; **Alerts**: Write failures, replica lag > 500ms, disk > 80%.

## 22. Interview Questions
1. *How does Qdrant handle vector index updates without blocking reads?* — Segment-based architecture: writes go to new segments, old segments are read-only, background optimizer merges them.
2. *Explain Raft consensus in Qdrant.* — Leader election + log replication across 3+ replicas for fault-tolerant writes.
3. *How to achieve sub-10ms query latency at 100M scale?* — Proper sharding, payload indexing, gRPC, memmap for disk-based storage, batch queries.
4. *What happens during a Qdrant segment optimization?* — Small segments are merged into larger ones, HNSW is rebuilt, old segments are deleted atomically.

## 23. Cheat Sheet

```python
# Collection management
client.create_collection(name, vectors_config, shard_number=N)
client.update_collection(optimizers_config=...)
client.delete_collection(name)

# CRUD
client.upsert(collection_name, points=[PointStruct(...)])
client.retrieve(collection_name, ids=[...])
client.delete(collection_name, points_selector=FilterSelector(...))

# Query
client.query_points(collection_name, query=vec, query_filter=Filter(...), limit=K)
client.search_groups(collection_name, query=vec, group_by="field")
```

## 24. Common Mistakes
- Not creating payload indexes for filtered fields — leads to full scan; Using default `ef_construct` for high-recall production (increase to 200); Not setting `write_consistency_factor=1` for throughput-only workloads; Over-sharding — 1 shard per 25M vectors is a good rule; Ignoring segment optimizer — check `qdrant_optimizer_segments` metric; Mixing gRPC and HTTP clients in same process.

## 25. Production Best Practices
- Run with `RUST_LOG=error` in production; Pin Qdrant version in Docker tags; Use `write_consistency_factor=0` (ack after WAL) for throughput; Pre-warm pages on startup; Schedule snapshots every 6h to S3; Use collection aliases for zero-downtime reindexing; Monitor shard distribution weekly; Test disaster recovery by killing nodes in staging; Run with `--disable-telemetry` for security compliance.
