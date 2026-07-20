# FAISS

## 1. What is it?
FAISS (Facebook AI Similarity Search) is a highly optimized C++ library for efficient similarity search and clustering of dense vectors, with Python bindings. Developed by Meta AI, it provides GPU-accelerated indexing, compression, and search algorithms.

## 2. Why do we need it?
When building custom retrieval systems at scale, pre-built vector databases add overhead and cost. FAISS gives you the raw building blocks for vector search — it's the engine inside most vector databases. Use it when you need maximum performance, custom index strategies, or lowest-cost deployment at billion-scale.

## 3. Real-world Example
**Meta** uses FAISS to power search across its knowledge graph of 10B+ entities. Facebook's "People You May Know" and Instagram's "Explore" feed both use FAISS for real-time similarity search over user embeddings. Spotify uses FAISS for music recommendation with 100M+ tracks, returning candidates in under 10ms.

## 4. Architecture Diagram (ASCII)

```
+------------------+
|   Application    |
+--------+---------+
         |
         v
+--------+---------+
|   FAISS Index     |
|   (In-memory)     |
|                    |
|   +------------+  |
|   | Coarse     |  |  (IVF: Inverted File)
|   | Quantizer  |  |
|   +-----+------+  |
|         |         |
|   +-----+------+  |
|   | Inverted   |  |  (Lists of vector IDs)
|   | Lists      |  |
|   +-----+------+  |
|         |         |
|   +-----+------+  |
|   | PQ/SQ      |  |  (Product/Scalar Quantization)
|   | Codec      |  |
|   +------------+  |
+--------+---------+
         |
         v
+--------+---------+
|   GPU (Optional)  |
|   (CUDA Kernels   |
|    for brute-     |
|    force/search)  |
+------------------+
```

## 5. Internal Working
FAISS builds compressed representations of vectors using quantization techniques. IVF (Inverted File) partitions vectors into clusters via k-means. At query time, only the closest clusters are searched. Product Quantization (PQ) compresses vectors into short codes (e.g., 64 bytes from 768-dim float) enabling in-memory billion-scale search. The GPU implementation uses CUDA kernels for massively parallel brute-force or IVF search.

## 6. Production Flow

```
Index training: Sample vectors -> k-means clustering -> Train PQ codec
-> Add vectors to IVF lists -> Write index to disk

Query: Load index to RAM (or GPU memory) -> Compute query IVF assignment
-> PQ distance computation -> Select top-K -> Re-rank with original vectors (optional)
```

## 7. HLD
- **Training Phase**: Index factory string specifies the pipeline (e.g., `IVF4096, PQ64`)
- **Index Types**: Flat (exact), IVF (approximate with coarse quantizer), HNSW (graph-based), LSH (hash-based), GpuIndexFlat (brute-force GPU)
- **Compression**: PQ (8-64 byte codes), SQ (scalar quantization to 8/6/4-bit), AQ (additive quantization)
- **Search**: `index.search(query_vector, k)` -> returns distances and IDs
- **Storage**: Index file on disk (`.faiss`), load into RAM at startup

## 8. LLD
- Index creation: `index = faiss.index_factory(dimension, "IVF4096, PQ64", faiss.METRIC_INNER_PRODUCT)`
- Training: `index.train(training_vectors)` (requires `ntotal >= nlist * 39`)
- Adding: `index.add(vectors)` (must be trained first)
- GPU: `res = faiss.StandardGpuResources(); gpu_index = faiss.index_cpu_to_gpu(res, 0, cpu_index)`
- ID mapping: `index = faiss.IndexIDMap(quantizer)` for custom vector IDs

## 9. Python Implementation

```python
import faiss
import numpy as np

dimension = 768
nlist = 4096

quantizer = faiss.IndexFlatIP(dimension)
index = faiss.IndexIVFPQ(quantizer, dimension, nlist, 64, 8)
index.metric_type = faiss.METRIC_INNER_PRODUCT
index.nprobe = 64

training_vectors = np.random.random((50000, dimension)).astype(np.float32)
index.train(training_vectors)

vectors = np.random.random((100000, dimension)).astype(np.float32)
ids = np.arange(100000).astype(np.int64)
index.add_with_ids(vectors, ids)

query = np.random.random((1, dimension)).astype(np.float32)
distances, indices = index.search(query, k=10)

print(f"Top-K indices: {indices[0]}")
print(f"Top-K distances: {distances[0]}")

faiss.write_index(index, "product_index.faiss")

# GPU acceleration
res = faiss.StandardGpuResources()
gpu_index = faiss.index_cpu_to_gpu(res, 0, index)
distances_gpu, indices_gpu = gpu_index.search(query, k=10)
```

## 10. Folder Structure

```
faiss_repo/
├── faiss/
│   ├── IndexFlat.h          # Exact search (brute-force)
│   ├── IndexIVF.h           # Inverted File index
│   ├── IndexPQ.h            # Product Quantization
│   ├── IndexHNSW.h          # HNSW graph index
│   ├── IndexScalarQuantizer.h
│   ├── GpuIndexFlat.h       # GPU brute-force
│   └── impl/
│       ├── ProductQuantizer.h
│       ├── ScalarQuantizer.h
│       └── PolysemousTraining.h
├── python/
│   └── swigfaiss.py          # Python bindings
├── benchs/
│   ├── bench_index.py
│   └── bench_gpu.py
└── tests/
    └── test_index.py
```

## 11. Configuration

```python
# Index factory strings
"Flat"                           # Exact search, no compression
"IVF4096, Flat"                  # IVF with exact search in cells
"IVF4096, PQ64"                  # IVF + Product Quantization (64 bytes)
"IVF4096, SQ8"                   # IVF + Scalar Quantization (8-bit)
"HNSW32, Flat"                   # HNSW graph
"OPQ64, IVF4096, PQ64x4fs"      # OPQ rotation + IVF + optimized PQ
"PCAR256, IVF4096, PQ64"         # PCA reduction then IVF+PQ

# Key parameters
index.nprobe = 64               # Cells to search during query
index.pq.cp.niter = 25          # PQ training iterations
index.verbose = True             # Print training progress

# GPU config
config = faiss.GpuIndexFlatConfig()
config.device = 0
config.useFloat16CoarseQuantizer = True
```

## 12. Flowchart

```
Start
  |
  v
Choose index type (based on data size, dimension, recall)
  |
  v
Prepare training vectors (sample representative subset)
  |
  v
Train index (k-means+PQ codebooks)
  |
  v
Add full vector set to index
  |
  v
Write index to disk
  |
  v
Load index at application start
  |
  v
Accept query vector
  |
  v
Assign to nearest nprobe IVF clusters
  |
  v
Compute PQ distance to cluster vectors
  |
  v
Re-rank (optional with original vectors)
  |
  v
Return top-K (IDs + distances/scores)
```

## 13. Sequence Diagram

```
Application         FAISS Index        GPU (optional)      Disk
  |                     |                   |                |
  |--train(samples)---->|                   |                |
  |                     |--k-means----------|                |
  |                     |--train PQ --------|                |
  |<--trained-----------|                   |                |
  |                     |                   |                |
  |--add(vectors)------>|                   |                |
  |                     |--enumerate--------|                |
  |<--added-------------|                   |                |
  |                     |                   |                |
  |--write_index()----->|                                    |
  |                     |----------------------------------->|
  |                     |                   |                |
  |--read_index()------>|                                    |
  |<--loaded------------|                   |                |
  |                     |                   |                |
  |--search(query,k)--->|                   |                |
  |                     |--assign IVF----->|                 |
  |                     |--compute PQ----->|                 |
  |                     |--re-rank----------|                |
  |<--distances,ids-----|                   |                |
```

## 14. Pros
- Blazing fast (C++ native, GPU acceleration); Extreme compression (PQ to 8 bytes/vector); Billion-scale on a single node; Full control over index configuration; No external dependencies (standalone library); Free and open-source (MIT); Memory efficient (1B 768-dim vectors ~ 16GB with PQ64); Active Meta AI development.

## 15. Cons
- No built-in persistence (you manage load/save); No metadata filtering; No distributed architecture (single-node only); No client-server protocol; No built-in embedding; Must implement your own query serving infrastructure; No incremental index building (some indexes need retraining); Steep learning curve for index selection.

## 16. Alternatives
- **Qdrant/Milvus** (distributed, managed); **ChromaDB** (simpler, persistent); **Elasticsearch** (metadata + vector); **ScaNN** (Google's alternative, tighter integration with TF); **Annoy** (Spotify, simpler but less accurate).

## 17. Performance Considerations
- `index_factory` string determines 100x performance difference; `nprobe` controls speed/recall tradeoff (64 for 0.95 recall); PQ code size = (dimension / subvectors) × 8 bits; IVF training requires `nlist * 39` vectors minimum; GPU index is 5-10x faster than CPU for Flat search; Memory: Flat = dimension × ntotal × 4 bytes; PQ64 = ntotal × 64 bytes + IVF overhead.

## 18. Scaling to Millions
- **1M 768-dim**: `IVF4096, PQ64` — 64MB PQ codes + 16MB IVF = ~80MB RAM; **100M**: same config → 6.4GB RAM, ~50ms per query; **1B**: `IVF65536, PQ64` → 64GB RAM, 100ms per query; Use GPU IndexFlat for 10M exact search (2ms); Multi-GPU with `index = faiss.index_cpu_to_all_gpus(cpu_index)`; For 10B+: shard across multiple machines, each with a segment of the data; Use `IndexShards` for multi-node support.

## 19. Failure Scenarios
- **OOM during training**: Use `faiss.index_factory` with smaller `nlist` or train on fewer vectors; **Index loading crash**: Version mismatch — always use same FAISS version for save/load; **Degraded recall post-add**: PQ centroids not representative — retrain index; **GPU OOM**: Reduce batch size or use `float16` for coarse quantizer; **Corrupted index file**: No built-in recovery — maintain backup `.faiss` files; **Non-contiguous memory**: FAISS requires contiguous float32 arrays — use `np.ascontiguousarray()`.

## 20. Security
- FAISS has no built-in auth — wrap behind API/middleware; Encrypt index files at rest (AES-256-GCM); Memory — vectors in plaintext in RAM; Run in memory-safe process (no JNI); Use gVisor/AppArmor if multi-tenant; Limit query rate at application layer; Input validation (NaN vectors crash FAISS); Regular FAISS version updates for security patches.

## 21. Monitoring
- Track `index.ntotal` for index size growth; Measure `search()` wall time at application level; Log `nprobe` and recall metrics; Monitor GPU memory usage if using GPU index; Track index rebuild frequency; Alert on search latency > 100ms or recall < 0.90; Prometheus metrics: `faiss_search_latency`, `faiss_index_size`, `faiss_recall_at_k`.

## 22. Interview Questions
1. *Explain Product Quantization in FAISS.* — Vectors are split into subvectors; each subvector is quantized using a separate codebook learned via k-means; distances are computed via lookup tables.
2. *How does IVFPQ work end-to-end?* — IVF partitions space into Voronoi cells; PQ compresses vectors within each cell; search: find nearest cells, compute PQ distances, return top-K.
3. *What is the tradeoff between IVF and HNSW?* — IVF: faster build, slower search for high recall; HNSW: slower build, faster search, more memory.
4. *When would you use GPU vs CPU FAISS?* — GPU for Flat (exact) search under 10M vectors; CPU IVFPQ for billion-scale approximate search.

## 23. Cheat Sheet

```python
# Index creation
index = faiss.index_factory(dim, "IVF4096, PQ64", faiss.METRIC_L2)
index.train(x)
index.add_with_ids(x, ids)
index.nprobe = 64

# Search
D, I = index.search(query, k=10)

# Save/Load
faiss.write_index(index, "index.faiss")
index = faiss.read_index("index.faiss")

# GPU
gpu_index = faiss.index_cpu_to_gpu(res, 0, cpu_index)

# ID mapping
index2 = faiss.IndexIDMap(index)
index2.add_with_ids(x, custom_ids)
```

## 24. Common Mistakes
- Training with fewer than `nlist * 39` vectors (k-means fails); Using `METRIC_L2` when vectors are normalized (use `METRIC_INNER_PRODUCT`); Not converting to float32 contiguous (silent crashes); Adding vectors to untrained index; Using `Flat` for billion-scale (RAM explosion); Not calling `index.reset()` before retraining; Assuming PQ64 means 64-dim (it's 64 bytes); Setting `nprobe > nlist` (wasted computation).

## 25. Production Best Practices
- Always pin FAISS version in requirements; Test index on representative sample before full build; Maintain separate training and production vector sets; Implement index versioning in file names; Use `IndexIDMap` for application-internal IDs; Run periodic recall benchmarks; Cache query results with TTL; Pre-filter vectors before FAISS search (application-level metadata filter); Implement graceful degradation (return cached results if search fails); Schedule index rebuilds when data distribution shifts detected.
