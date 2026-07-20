# Milvus

> Author: **Tamilselvan** · ✉️ tamilselvan.sde@gmail.com · 🔗 [LinkedIn](https://www.linkedin.com/in/tamilselvan-ai/)
>

## 1. What is it?
Milvus is an open-source, cloud-native vector database built for billion-scale similarity search. Originally developed by Zilliz, it separates compute from storage using a log-structured architecture, enabling elastic scaling and GPU acceleration.

## 2. Why do we need it?
When your vector dataset exceeds 100M, single-node databases fail. Milvus provides distributed architecture with sharding, replication, and GPU-accelerated indexing. It's built for enterprise AI workloads requiring high throughput, low latency, and multi-tenancy at scale.

## 3. Real-world Example
**Walmart** uses Milvus for its product recommendation system. With 500M+ SKUs, each represented by a 768-dimension embedding, Milvus serves 50,000 QPS with p99 < 50ms. Complex filters (price range, category, availability) combined with vector similarity drive "customers also bought" recommendations across 4,700 stores.

## 4. Architecture Diagram (ASCII)

```
+------------------+     +------------------+
|   Proxy Nodes    |     |   Proxy Nodes    |
|   (Stateless)    |     |   (Stateless)    |
+--------+---------+     +--------+---------+
         |                          |
         v                          v
+--------+---------+     +------------------+
|   Root Coordinator|     |  Data Coordinator|
|   (DDL, DCL)      |     |  (Segment Mgmt) |
+--------+---------+     +--------+---------+
         |                          |
         v                          v
+--------+---------+     +--------+---------+
|   Query Nodes     |     |   Index Nodes    |
|   (CPU/GPU)       |     |   (GPU: NVIDIA)  |
+--------+---------+     +--------+---------+
         |                          |
         v                          v
+--------+---------+     +------------------+
|   Object Store    |     |   Meta Store     |
|   (MinIO/S3/GCS) |     |   (etcd)         |
+------------------+     +------------------+
```

## 5. Internal Working
Milvus uses a log-structured architecture: data flows as streaming logs through message storage (Pulsar/Kafka). The RootCoordinator manages DDL operations. DataCoordinator assigns segment building to DataNodes. IndexNodes build vector indexes (IVF, HNSW, DiskANN) on new segments. QueryNodes load segments into memory for serving. MetaStore (etcd) tracks all component states.

## 6. Production Flow

```
Client insert -> Proxy logs to Pulsar -> DataNode writes to object store
-> IndexNode builds index -> QueryNode loads index -> Ready for search

Client search -> Proxy broadcasts to QueryNodes -> Parallel ANN search
-> Reduce & rank -> Return top-K
```

## 7. HLD
- **Access Layer**: Proxy nodes — stateless, no caching, handles auth and rate limiting
- **Coordinator Layer**: RootCoord (schema), DataCoord (segments), QueryCoord (load balancing), IndexCoord (task assignment)
- **Worker Layer**: DataNodes (write), QueryNodes (read/search), IndexNodes (build indexes on GPU)
- **Storage Layer**: Object store (MinIO/S3/GCS) for data durability, etcd for metadata
- **Log Layer**: Pulsar/Kafka for write-ahead log and streaming

## 8. LLD
- Collection = table with auto_id, shards_num, partition_key
- Entity = `{id: int64, vector: float_vector, field1: varchar, ...}`
- Index types: IVF_FLAT, IVF_SQ8, HNSW, DiskANN, GPU_IVF_FLAT, GPU_CAGRA
- Consistency levels: Strong, Bounded, Session, Eventually — tunable per request

## 9. Python Implementation

```python
from pymilvus import connections, Collection, FieldSchema, CollectionSchema, DataType

connections.connect(host="milvus-proxy.company.com", port=19530)

fields = [
    FieldSchema(name="id", dtype=DataType.INT64, is_primary=True, auto_id=True),
    FieldSchema(name="embedding", dtype=DataType.FLOAT_VECTOR, dim=768),
    FieldSchema(name="category", dtype=DataType.VARCHAR, max_length=64),
    FieldSchema(name="price", dtype=DataType.FLOAT),
]

schema = CollectionSchema(fields, description="Product embeddings")
collection = Collection(name="products", schema=schema)

collection.create_index(
    field_name="embedding",
    index_params={
        "metric_type": "IP",
        "index_type": "IVF_SQ8",
        "params": {"nlist": 4096},
    },
)

collection.load()

search_params = {
    "metric_type": "IP",
    "params": {"nprobe": 64},
}

results = collection.search(
    data=[query_vector],
    anns_field="embedding",
    param=search_params,
    limit=100,
    expr="category == 'electronics' && price < 500",
)
```

## 10. Folder Structure

```
milvus/
├── configs/
│   ├── milvus.yaml        # Main configuration
│   └── chart/              # Helm chart for K8s
├── internal/
│   ├── proxy/              # Query routing and auth
│   ├── rootcoord/          # DDL/DCL coordination
│   ├── datacoord/          # Segment assignment
│   ├── datanode/           # Write path
│   ├── querynode/          # Read path
│   ├── indexnode/          # Index building
│   └── storage/            # MinIO/S3 abstraction
├── scripts/
│   └── migrate/            # Data migration tools
└── deploy/
    └── docker-compose.yml
```

## 11. Configuration

```yaml
# milvus.yaml
proxy:
  port: 19530
  rateLimit: { qps: 5000 }
rootCoord:
  port: 53100
  maxCollectionNum: 65536
dataCoord:
  segment:
    maxSize: 512   # MB per segment
    sealInterval: 600  # seconds
queryNode:
  enableDisk: true
  loadMemoryRatio: 0.4
indexNode:
  enableGPU: true
  gpu: { device: "cuda:0" }
common:
  storageType: "minio"
  pulsar: { address: "pulsar:6650" }
```

## 12. Flowchart

```
Create Collection
  |
  v
Insert Entities (batched)
  |
  v
Proxy -> Pulsar log -> DataNode flush -> MinIO
  |
  v
IndexNode builds index (GPU: IVF/HNSW)
  |
  v
QueryNode loads index into memory/disk
  |
  v
Search Request
  |
  v
Proxy routes to all QueryNodes
  |
  v
Each QueryNode: index search + expression filter
  |
  v
Proxy merges top-K
  |
  v
Return results
```

## 13. Sequence Diagram

```
Client     Proxy    RootCoord   DataCoord   DataNode   IndexNode   QueryNode  MinIO
  |          |          |           |           |           |          |        |
  |--create-->|          |           |           |           |          |        |
  |          |--DDL----->|           |           |           |          |        |
  |          |          |--ok------->|           |           |          |        |
  |<--ok-----|          |           |           |           |          |        |
  |          |          |           |           |           |          |        |
  |--insert-->|          |           |           |           |          |        |
  |          |--log---------------> |           |           |          |        |
  |          |          |           |--flush--->|           |          |        |
  |          |          |           |           |--write-------------->|        |
  |          |          |           |           |           |          |        |
  |          |          |           |--index--->|----------->|         |        |
  |          |          |           |           |           |--read--->|        |
  |<--ok-----|          |           |           |           |          |        |
  |          |          |           |           |           |          |        |
  |--search->|          |           |           |           |          |        |
  |          |--broadcast---------------------------------->|         |        |
  |          |          |           |           |           |--search  |        |
  |          |<--results------------------------------------|         |        |
  |<--ok-----|          |           |           |           |          |        |
```

## 14. Pros
- Billion-scale with GPU-accelerated indexing; Cloud-native (Kubernetes-first); Separate compute from storage (elastic scaling); Multiple index types (IVF, HNSW, DiskANN, GPU-CAGRA); Hybrid search (vector + scalar + full-text); Change data capture via Pulsar log; Active community (CNCF graduated); Multi-language SDKs.

## 15. Cons
- High operational complexity (Pulsar, etcd, MinIO, 6+ components); Resource-heavy — GPU required for fast indexing; Write-then-read eventual consistency can surprise; Segment management adds latency spikes during compaction; Overkill for datasets under 10M; Learning curve: 40+ configuration parameters.

## 16. Alternatives
- **Qdrant**: Simpler ops for 10-500M scale; **Pinecone**: Fully managed, no ops; **Weaviate**: GraphQL + vector search combined; **Elasticsearch**: If you already use ES, its vector search may suffice.

## 17. Performance Considerations
- IVF_SQ8 offers 4x compression vs IVF_FLAT with 2% recall loss; GPU-CAGRA (NVIDIA) indexes 1B vectors in under 2 hours; Use `nprobe=64` for high recall, 8 for high throughput; Segment size 512MB balances compaction vs search speed; DiskANN for >10M vectors where RAM is constrained; 10M vectors = ~6GB RAM for FLOAT_VECTOR(768).

## 18. Scaling to Millions
- **100M**: 4 QueryNodes, 4GB RAM each, IVF_SQ8 index; **1B**: 32 QueryNodes, GPU IndexNode with A100, DiskANN for cold data; **10B**: 128+ node cluster, partition by tenant, tiered storage (hot in RAM, cold in SSD); Use partition key for multi-tenancy — query only relevant partitions.

## 19. Failure Scenarios
- **etcd failure**: Cluster becomes read-only — deploy etcd cluster with 3+ members; **Pulsar backlog**: DataNode falls behind — scale DataNodes or increase Pulsar partitions; **Segment seal timeout**: Large insert triggers stuck segment — tune `segment.sealInterval`; **GPU OOM**: Index building fails — set `indexNode.gpu.memoryLimit` lower; **MinIO degradation**: All writes stall — deploy MinIO in distributed mode with erasure coding.

## 20. Security
- TLS for all inter-component communication; RBAC via Milvus built-in (role-based); Authentication with username/password or token; Network Policies in K8s to isolate components; Audit logging of DDL operations; Encrypt sensitive fields at application level before storing; Regular security updates — subscribe to Milvus releases.

## 21. Monitoring
- Milvus exports Prometheus metrics by default: `milvus_proxy_search_latency`, `milvus_datanode_flush_count`; Grafana dashboards available in repo; Track `querynode.memoryUsage` — should be under 80%; Watch `segment.sealedCount` — growth indicates compaction lag; Alert on `rootcoord.ddl_failure_count` > 0; Log all component logs via Fluentd to Elasticsearch.

## 22. Interview Questions
1. *Explain Milvus's log-structured architecture.* — Data flows as logs through Pulsar, enabling separation of write and read paths, elastic scaling, and point-in-time recovery.
2. *How does Milvus handle index building without blocking writes?* — Streaming and batch indexes coexist; writes go to growing segments (no index), indexed segments are immutable for search.
3. *When would you choose GPU_IVF_FLAT over HNSW?* — GPU_IVF_FLAT for throughput-sensitive GPU-enabled clusters; HNSW for lower memory or CPU-only.
4. *What is the purpose of the consistency level in Milvus?* — Tunable consistency (Strong/Session/Bounded/Eventually) allows performance vs recency tradeoff per query.

## 23. Cheat Sheet

```python
# Connection + Schema
connections.connect(host, port)
collection = Collection("name")
collection.insert([{...}, {...}])

# Index
collection.create_index("embedding", index_params)
collection.load()

# Search
collection.search(query_vectors, anns_field, param, limit, expr)

# Manage
collection.drop_index()
collection.release()
collection.flush()  # force seal current segment
```

## 24. Common Mistakes
- Not calling `collection.load()` before search (returns empty results silently); Using `IVF_FLAT` on billion-scale without compression (use IVF_SQ8 or DiskANN); Running without resource limits in K8s (OOM kills QueryNodes); Setting `shards_num > 16` (degrades performance); Not scheduling compaction during low-traffic windows; Ignoring `expr` index requirements (filtering becomes slow full scan).

## 25. Production Best Practices
- Deploy via Helm on Kubernetes with resource requests/limits; Use 3+ etcd nodes, 3+ Pulsar brokers; Pre-create indexes before bulk load; Batch inserts in 10K-50K per request; Schedule compaction during off-peak hours; Use partition key for multi-tenant isolation; Enable DiskANN for cost-effective billion-scale; Monitor segment count (target < 100 per collection); Run chaos engineering tests (kill nodes, inject latency) in staging; Keep Milvus version pinned and test upgrades in dev first.
