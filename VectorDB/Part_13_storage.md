# Part 13: Storage

> Author: **Tamilselvan** · ✉️ tamilselvan.sde@gmail.com · 🔗 [LinkedIn](https://www.linkedin.com/in/tamilselvan-ai/)
>

## RAM (In-Memory Storage)

Most vector databases store indexes and vectors primarily in RAM for maximum performance.

```mermaid
graph TB
    subgraph "In-Memory Architecture"
        A[RAM] --> B[Vector Data<br/>float32 arrays]
        A --> C[Graph Index<br/>HNSW edges]
        A --> D[Metadata<br/>Filterable fields]
        A --> E[Cache<br/>Hot paths]
    end
    subgraph "Typical Memory Usage (1M × 768d)"
        F["Vectors: 3.07 GB<br/>HNSW graph: 0.5-1 GB<br/>Metadata: 0.2-0.5 GB<br/>Total: ~4 GB"]
    end
```

### Memory Mapping (mmap)

Many vector DBs use **memory-mapped files** to extend effective RAM:

```python
import numpy as np
import mmap

# Memory-map a large vector file
with open('vectors.bin', 'r+b') as f:
    # Map the entire file
    mmapped = mmap.mmap(f.fileno(), 0)
    
    # Access as numpy array without loading all into RAM
    vectors = np.frombuffer(
        mmapped, dtype=np.float32
    ).reshape(-1, 768)
    
    # Access individual vectors on demand
    vector_42 = vectors[42]  # OS loads this page on demand
```

**Benefits:**
- OS handles caching (hot vectors stay in RAM, cold evicted)
- Larger-than-RAM datasets possible
- Fast restart (no rebuild needed)

---

## SSD & DiskANN

For billion-scale datasets, SSD-based approaches become necessary.

```mermaid
graph TD
    subgraph "Memory Hierarchy for Vector Search"
        A["L1/L2 Cache<br/>~1 MB<br/>0.5 ns"] --> B["RAM<br/>~100 GB<br/>50 ns"]
        B --> C["SSD (NVMe)<br/>~10 TB<br/>10 μs"]
        C --> D["HDD<br/>~100 TB<br/>10 ms"]
    end
    subgraph "Vector DB Placement"
        E["HNSW: RAM only"]
        F["DiskANN: SSD + RAM cache"]
        G["SPANN: SSD + RAM cache"]
    end
```

### DiskANN Architecture

```mermaid
graph TB
    subgraph "DiskANN Design"
        A[SSD Storage] --> B["Vamana Graph<br/>100 GB for 1B vectors"]
        A --> C["Vectors<br/>Compressed (PQ)"]
        D[RAM Cache] --> E["Warm nodes<br/>~2-10 GB"]
        D --> F["Entry points<br/>Metadata"]
    end
    subgraph "Search Flow"
        G[Query] --> H[Load starting nodes from cache]
        H --> I[Traverse graph, fetch from SSD]
        I --> J[Cache hot nodes]
        J --> K[Return top-K]
    end
```

---

## Persistence & Durability

### Write-Ahead Log (WAL)

```mermaid
sequenceDiagram
    participant Client
    participant DB as Vector DB
    participant WAL as Write-Ahead Log
    participant Index
    
    Client->>DB: Insert(v1, meta1)
    DB->>WAL: Append to WAL on disk
    WAL-->>DB: Ack
    DB->>Index: Add to memory buffer
    DB-->>Client: Confirmed
    
    Note over DB,Index: Periodic checkpoint
    DB->>Index: Flush buffer to main index
    DB->>WAL: Truncate checkpointed entries
```

### Persistence Strategies

| Strategy | Durability | Write Speed | Read Speed | Recovery Time |
|----------|-----------|-------------|------------|---------------|
| **mmap only** | None (OS dependent) | Fast | Fast | Instant |
| **WAL + periodic flush** | Crash-safe | Medium | Fast | Seconds |
| **Synchronous write** | Full durability | Slow | Fast | None |
| **Replication** | Network-durable | Medium | Fast | Instant via failover |

---

### Production Tip

> **Memory planning formula:**
> - Vectors: `N × D × 4 bytes` (float32)
> - HNSW edges: `N × M × 2 × 4 bytes` (≈50% of vectors)
> - PQ codes: `N × (D/8) × 1 byte` (≈3% of vectors)
> - Metadata: `N × ~100 bytes` (approximate)
> - Total ≈ `N × (4×D + 32 + 100 + overhead)`

---

