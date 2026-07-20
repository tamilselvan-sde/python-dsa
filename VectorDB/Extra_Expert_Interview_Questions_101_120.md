# Additional Expert Interview Questions

> Author: **Tamilselvan** · ✉️ tamilselvan.sde@gmail.com · 🔗 [LinkedIn](https://www.linkedin.com/in/tamilselvan-ai/)
>


**Q101: How does the choice of `M` in PQ compression affect retrieval quality mathematically?**

> **A:** PQ splits a D-dimensional vector into M sub-vectors of D/M dimensions. Each sub-vector is quantized to k centroids (typically k=256). The total number of possible centroids is k^M, but the effective codebook size is M × k. Higher M means each sub-vector has fewer dimensions, making centroid learning more stable, but the total centroids are fewer. The theoretical distortion is O(1/M²) — diminishing returns beyond M=96 for 768d vectors.

**Q102: Explain the mathematical foundation of RaBitQ (RABit Quantization).**

> **A:** RaBitQ uses random binary trees to partition the space. Each vector is assigned a binary code based on which side of each hyperplane it falls on. Distance is approximated by counting different bits (Hamming distance) weighted by tree depth. This achieves O(log N) search with extremely compact codes (64-128 bits per vector).

**Q103: What is the role of the "reorder" step in ScaNN?**

> **A:** ScaNN's reorder step: after first-stage approximate search with PQ, re-compute exact distances for the top candidates using the original uncompressed vectors. This corrects ranking errors from PQ compression while keeping most of the speed. Typically reorder top 100 candidates out of 1000 retrieved.

**Q104: How does the dimensionality of embeddings affect the Pareto frontier of speed vs accuracy in ANN?**

> **A:** Higher dimensions shift the Pareto frontier: each distance computation is more expensive (O(d)), and the curse of dimensionality makes approximate methods less effective. At 768d, HNSW achieves 99% recall at 2ms. At 3072d, the same recall requires 8ms. The tradeoff is linear in dimension for computation but super-linear in accuracy.

**Q105: What is the theoretical connection between HNSW and the Delaunay graph?**

> **A:** HNSW approximates the Delaunay graph of the point set. A Delaunay graph connects points whose Voronoi cells share a boundary. HNSW's neighbor selection heuristic approximates this — each node connects to diverse neighbors that "cover" different angular regions, mimicking Delaunay connectivity.

**Q106: Compare and contrast the write amplification of LSM-tree based vector DBs vs segment-based approaches.**

> **A:** LSM-tree approach (used by Qdrant): writes go to WAL -> in-memory buffer -> periodic compaction. Write amplification: 10-50x (each vector written and rewritten during compaction). Segment-based (used by Milvus): append-only segments, periodically merged. Write amplification: 2-5x. LSM gives better read performance, segments give better write performance.

**Q107: How would you implement exact nearest neighbor search using only the HNSW index structure?**

> **A:** Set efSearch = N (total number of vectors). This forces the search to explore all nodes in the bottom layer, effectively performing an exact search. However, this is less efficient than brute force because graph traversal overhead. Better approach: use HNSW as a filtered candidate generator, then rerank all candidates with exact computation.

**Q108: Explain the non-determinism in HNSW search and how to mitigate it.**

> **A:** HNSW search is non-deterministic because it depends on: (1) insertion order, (2) random level assignment, (3) efSearch ordering. Results can vary across runs. Mitigation: (1) fix random seed during build, (2) use deterministic neighbor selection, (3) increase efSearch beyond needed recall to reduce variance, (4) for production, use majority voting across multiple searches.

**Q109: How does the "Ablation Study" in the HNSW paper demonstrate the importance of hierarchy?**

> **A:** The HNSW paper shows that without hierarchy (single layer NSW), search complexity is O(N^c) where c < 1, versus O(log N) with hierarchy. The multi-layer structure provides "express lanes" that allow the search to cover large distances in few hops at the top layer, then refine at lower layers.

**Q110: Design a vector search system for a real-time fraud detection pipeline processing 100K transactions/second.**

> **A:** Key components: (1) Streaming embeddings: real-time feature extraction from transactions. (2) In-memory buffer: 1-minute sliding window of recent transactions. (3) FAISS GPU index: rebuilt every minute from buffer. (4) HNSW index: <1ms search time. (5) Query: check if current transaction is similar to known fraud patterns. (6) Threshold-based alerting. Architecture: Kafka -> Flink -> GPU cluster -> Alert system.

**Q111: What is the relationship between recall@k and precision@k in vector search, and when would you optimize for each?**

> **A:** recall@k = relevant retrieved / total relevant; precision@k = relevant retrieved / k. In vector search, recall@k matters when the user needs to find ALL relevant items (e-discovery, research). precision@k matters when the user only sees the top results and irrelevant ones are costly (search engines, recommendations).

**Q112: Explain the role of the "filtered search" algorithm in Qdrant's HNSW implementation.**

> **A:** Qdrant's filtered HNSW integrates filters into graph traversal: each node stores a bitmask of filter conditions. During search, nodes that don't match the filter are skipped without expanding their neighbors. This avoids post-filtering (which may lose results) and pre-filtering (which may be slow). The filter index is built as an inverted index on top of the graph.

**Q113: How would you implement a vector database from scratch in Python (educational)?**

> **A:** Core components: (1) Vector storage: numpy array (memory) or memmap (disk). (2) Index: simple HNSW implementation with random level assignment, greedy search, efSearch. (3) Metadata: dict mapping IDs to payloads. (4) Filter: Python expression evaluation on payloads. (5) Distance: numpy vectorized cosine/dot/L2. (~500 lines for basic version)

**Q114: What is the impact of batch normalization on embedding quality and search performance?**

> **A:** Batch normalization during embedding model training helps stabilize training and produce more uniformly distributed embeddings. This improves search because: (1) all dimensions have similar scale (no dominant dimensions), (2) vectors are more spread out (less collapse), (3) distance metrics become more discriminative. Poorly normalized embeddings can lead to 5-15% recall degradation.

**Q115: How does the number of clusters (nlist) in IVF interact with the data dimensionality?**

> **A:** For low dimensions (<100), nlist can be large (sqrt(N) × 2-3x) because clusters are well-separated. For high dimensions (768+), clusters overlap more (curse of dimensionality), so larger nlist doesn't help as much. Heuristic: nlist = sqrt(N) × max(1, 100/d). So for 768d, nlist = sqrt(N) × 0.13.

**Q116: Design an A/B testing framework for evaluating vector search quality in production.**

> **A:** (1) Shadow mode: new index runs in parallel, results logged but not served. (2) Offline evaluation: compare shadow results to production results using recall@k and NDCG. (3) Live A/B: 1-5-10% traffic to new index, track click-through rate, engagement, user satisfaction. (4) Rollback: automated if any metric drops >2% with statistical significance.

**Q117: What is the theoretical maximum speedup of ANN over exact search for a given dataset?**

> **A:** Maximum speedup = O(N / log N) for graph-based (HNSW), O(N / sqrt(N)) for cluster-based (IVF). For N=1M: HNSW speedup ~50,000x, IVF speedup ~1,000x. In practice, constant factors reduce this: HNSW achieves ~1,000-10,000x speedup at 99% recall; IVF achieves ~100-1,000x at 95% recall.

**Q118: How does the Johnson-Lindenstrauss lemma apply to vector search?**

> **A:** The JL lemma states that any N points in high-dimensional space can be embedded into O(log N / ε²) dimensions while approximately preserving pairwise distances (within 1±ε). This means we can theoretically reduce 768d to ~200d for 1M points with minimal distortion. In practice, learned embeddings already capture the "intrinsic dimension" which may be much lower than the raw dimension.

**Q119: Explain the concept of "intrinsic dimension" of embeddings and its impact on ANN.**

> **A:** The intrinsic dimension is the minimum number of parameters needed to represent the data without significant information loss. Text embeddings often have intrinsic dimension 64-128 even if the extrinsic dimension is 768. ANN algorithms that exploit this (like PCA-preprocessing, or RaBitQ) can achieve 5-10x better speed/memory tradeoffs than those that don't.

**Q120: What novel ANN algorithms or techniques do you expect to see in the next 5 years?**

> **A:** (1) Neural indexes: learned data structures replacing hand-crafted algorithms. (2) Quantum-inspired ANN: using quantum computing principles on classical hardware. (3) Self-supervised index tuning: indexes that learn from query patterns. (4) Biological-inspired: neuromorphic computing for ultra-low-power vector search. (5) Federated ANN: privacy-preserving distributed vector search. (6) Differentiable indexes: end-to-end learned retrieval systems.

