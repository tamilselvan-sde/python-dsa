# Part 19: Scaling

> Author: **Tamilselvan** · ✉️ tamilselvan.sde@gmail.com · 🔗 [LinkedIn](https://www.linkedin.com/in/tamilselvan-ai/)
>

## Millions to Billions

```mermaid
graph LR
    subgraph "Scale Levels"
        A["<1M vectors<br/>Single machine<br/>HNSW"] --> B["1M-10M vectors<br/>Single machine<br/>HNSW/IVF_PQ"]
        B --> C["10M-100M vectors<br/>Sharded cluster<br/>IVF_PQ/HNSW"]
        C --> D["100M-1B vectors<br/>Distributed cluster<br/>DiskANN/PQ"]
        D --> E["1B+ vectors<br/>Multi-region<br/>DiskANN/SPANN"]
    end
```

### Scaling Dimensions

```mermaid
graph TB
    A[Scaling Vector DB] --> B[Vertical Scaling<br/>More RAM, Better CPU]
    A --> C[Horizontal Scaling<br/>More machines]
    A --> D[Compression<br/>PQ, SQ, Binary]
    
    B --> E["1 machine<br/>64-512 GB RAM<br/>50M-500M vectors"]
    C --> F["N machines<br/>TBs of RAM<br/>1B+ vectors"]
    D --> G["Same RAM<br/>3-20x more vectors<br/>5-15% accuracy loss"]
```

---

## Sharding

**Sharding** splits data across multiple machines (shards).

```mermaid
graph TB
    subgraph "Sharded Cluster"
        A[Router / Proxy] --> B[Shard 0<br/>Vectors 0-9999]
        A --> C[Shard 1<br/>Vectors 10000-19999]
        A --> D[Shard 2<br/>Vectors 20000-29999]
        A --> E[Shard N<br/>...]
    end
    subgraph "Query Flow"
        F[Query] --> A
        A --> G[Broadcast to all shards]
        G --> H[Aggregate results]
        H --> I[Return top-K overall]
    end
```

### Sharding Strategies

| Strategy | Description | Pros | Cons |
|----------|-------------|------|------|
| **Hash-based** | hash(vector_id) → shard | Even distribution | Hard to rebalance |
| **Range-based** | ID range → shard | Good for sequential access | Skew possible |
| **Location-based** | geo_hash → shard | Locality-preserving | Complex routing |
| **Tenant-based** | tenant_id → shard | Tenant isolation | Variable shard sizes |

### Consistent Hashing

```mermaid
graph TB
    subgraph "Consistent Hash Ring"
        A["Node A<br/>>>-  <"]
        B["Node B"]
        C["Node C"]
        D["Node D"]
    end
    E["Vector ID 42"] -.->|"hash→ring position"| C
    F["Vector ID 99"] -.->|"hash→ring position"| A
```

**Benefits:** Adding/removing nodes only relocates ~1/N of data.

---

## Replication

**Replication** creates copies of data for fault tolerance and read scaling.

```mermaid
graph TB
    subgraph "Replication Factor = 3"
        A[Leader Shard 0] --> B[Replica 0-1]
        A --> C[Replica 0-2]
        
        D[Leader Shard 1] --> E[Replica 1-1]
        D --> F[Replica 1-2]
    end
    subgraph "Read/Write Flow"
        G[Write] --> A
        G --> D
        H[Read] --> B
        H --> C
        H --> E
        H --> F
    end
```

| Replication Factor | Fault Tolerance | Write Cost | Read Throughput |
|-------------------|----------------|------------|-----------------|
| 1 | None | 1x | 1x |
| 2 | 1 node | 2x | 2x |
| 3 | 2 nodes | 3x | 3x |

**Raft/PAXOS consensus:** Used by Milvus, Qdrant for consistent replication.

---

## Partitioning

**Partitioning** splits a collection into smaller physical segments for easier management.

```mermaid
graph TB
    subgraph "Collection"
        A[Partition 0<br/>date: 2024-Q1]
        B[Partition 1<br/>date: 2024-Q2]
        C[Partition 2<br/>date: 2024-Q3]
    end
    subgraph "Query with filter"
        D[Query: Q2 2024 data]
        D --> E["Only search Partition 1"]
        E --> F[Faster search + lower cost]
    end
```

**Use cases:**
- Time-based partitioning (search recent data only)
- Geo-based partitioning (search data in region only)
- Category-based partitioning (search specific category only)

---

## GPU vs CPU

```mermaid
graph TB
    subgraph "Vector Search Hardware"
        A[CPU] --> B["Good for:<br/>HNSW traversal<br/>Filtering<br/>Metadata indexing"]
        C[GPU] --> D["Good for:<br/>Batch similarity<br/>K-means training<br/>Large matrix ops"]
    end
```

### Performance Comparison

| Operation | CPU (32 cores) | GPU (A100) | GPU Speedup |
|-----------|---------------|------------|-------------|
| Brute force (1M × 768) | 50ms | 1ms | 50x |
| IVF training (10M) | 5 min | 10 sec | 30x |
| HNSW build (10M) | 30 min | 5 min | 6x |
| Search (1M, HNSW) | 2ms | 1ms | 2x |

**When GPU helps:**
- Large batch searches
- Frequent index rebuilding
- Real-time training
- High-throughput scenarios

**When CPU is fine:**
- Low-latency single queries
- Small-medium indexes
- Filter-heavy workloads

---

### Production Tip

> **Scaling checklist:**
> 1. Start with single node + HNSW
> 2. At ~60% memory usage → add PQ compression
> 3. At ~80% memory usage → add sharding
> 4. At 5+ shards → add replication
> 5. At 50+ shards → consider multi-region

---

