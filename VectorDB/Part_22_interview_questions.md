# Part 22: Interview Questions

> Author: **Tamilselvan** · ✉️ tamilselvan.sde@gmail.com · 🔗 [LinkedIn](https://www.linkedin.com/in/tamilselvan-ai/)
>

## 100 Vector DB Interview Questions with Answers

### Beginner Level (1-25)

**Q1: What is a vector database?**

> **A:** A vector database is a specialized database system designed to store, index, and search high-dimensional vectors (embeddings). It enables similarity search — finding vectors that are "closest" to a given query vector using distance metrics like cosine similarity or Euclidean distance.

**Q2: How is a vector database different from a traditional SQL database?**

> **A:** SQL databases store structured data in tables and support exact match queries. Vector databases store embeddings (lists of floats) and support similarity search. SQL answers "equals," vector DBs answer "similar to."

**Q3: What is an embedding?**

> **A:** An embedding is a dense vector representation of data (text, image, audio) that captures semantic meaning in a high-dimensional space. Similar content produces similar vectors.

**Q4: What is cosine similarity?**

> **A:** Cosine similarity measures the cosine of the angle between two vectors, ranges from -1 to 1, with 1 meaning identical direction (similar) and -1 meaning opposite direction (dissimilar).

**Q5: What is the curse of dimensionality?**

> **A:** In high-dimensional spaces (>100 dimensions), all vectors appear roughly equidistant, making distance metrics less discriminative. This is why specialized ANN algorithms are needed.

**Q6: What is ANN search?**

> **A:** Approximate Nearest Neighbor search — finds "close enough" vectors without comparing against every vector in the database. Sacrifices a small amount of recall for massive speed gains.

**Q7: What is recall in vector search?**

> **A:** Recall = (number of true nearest neighbors found) / (total true nearest neighbors). 100% recall means all exact nearest neighbors were found. ANN typically targets 90-99% recall.

**Q8: What is HNSW?**

> **A:** Hierarchical Navigable Small World — the most popular ANN algorithm. It uses a multi-layer graph structure for extremely fast approximate nearest neighbor search with high recall.

**Q9: What is IVF?**

> **A:** Inverted File Index — clusters vectors using k-means, then searches only the nearest clusters. Balances speed and memory usage.

**Q10: What is Product Quantization (PQ)?**

> **A:** A compression technique that splits vectors into sub-vectors and quantizes each using a learned codebook. Reduces memory by 10-50x with minimal accuracy loss.

**Q11: What is RAG?**

> **A:** Retrieval-Augmented Generation — a technique where relevant documents are retrieved from a vector database and provided as context to an LLM to generate accurate, grounded answers.

**Q12: What is the difference between dense and sparse retrieval?**

> **A:** Dense retrieval uses embeddings (semantic matching, understands synonyms). Sparse retrieval (BM25) uses exact keyword matching (captures rare terms, no semantic understanding).

**Q13: What is chunking and why is it important?**

> **A:** Chunking splits documents into smaller pieces for embedding. It's important because models have token limits, smaller chunks enable more precise retrieval, and LLM context windows are limited.

**Q14: What is metadata filtering?**

> **A:** Attaching structured fields (date, author, tenant, category) to vectors to enable filtered search — only returning vectors matching certain criteria in addition to similarity.

**Q15: What is pre-filtering vs post-filtering?**

> **A:** Pre-filtering: filter vectors first, then search. Post-filtering: search first, then filter results. Most modern DBs use integrated filtering for both efficiency and correctness.

**Q16: What is a vector index?**

> **A:** A data structure that enables fast approximate nearest neighbor search by organizing vectors for efficient traversal instead of linear scan.

**Q17: What is `efSearch` in HNSW?**

> **A:** A query-time parameter controlling search breadth. Higher values = more candidates considered = better recall but slower search. Typically set between 100-500.

**Q18: What is `nprobe` in IVF?**

> **A:** The number of nearest clusters to search. Higher values = better recall but slower search. Set based on recall requirements.

**Q19: What are the advantages of HNSW over IVF?**

> **A:** HNSW offers faster search, higher recall at the same speed, and no training phase. IVF uses less memory and builds faster.

**Q20: What is the purpose of normalization in vector search?**

> **A:** Normalization makes vectors unit-length so that cosine similarity = dot product. This simplifies distance computations and ensures only direction matters.

**Q21: What embedding dimensions does OpenAI's text-embedding-3-small use?**

> **A:** It supports 512-1536 dimensions (configurable). Default is 1536.

**Q22: What is BM25?**

> **A:** A ranking function for keyword search that scores documents based on term frequency and inverse document frequency, with saturation and length normalization.

**Q23: What is the role of a reranker in a RAG pipeline?**

> **A:** A reranker (typically a cross-encoder) takes the top-k results from the first-stage retrieval and scores them more accurately, improving result quality before sending to the LLM.

**Q24: What is RRF (Reciprocal Rank Fusion)?**

> **A:** A method for combining multiple ranked lists by averaging reciprocal ranks. Robust to differences in score scales between different retrieval methods.

**Q25: What is MMR (Maximum Marginal Relevance)?**

> **A:** A diversity-promoting algorithm that selects results balancing relevance to query and dissimilarity to already-selected results.

---

### Intermediate Level (26-50)

**Q26: Explain how HNSW search works step by step.**

> **A:** (1) Start at entry point at the top layer. (2) Greedily traverse to the closest neighbor at the current layer. (3) Repeat until no closer neighbor found. (4) Descend to the next layer. (5) Repeat from the nearest node. (6) At the bottom layer, expand search with ef candidates. (7) Return top-k results.

**Q27: How does IVFPQ combine IVF and PQ?**

> **A:** IVF first assigns each vector to its nearest centroid. Within each cluster, PQ compresses the residual (vector - centroid). Search is done by computing distances to centroids, then ADC (asymmetric distance computation) within selected clusters.

**Q28: What is the difference between symmetric and asymmetric distance computation?**

> **A:** Symmetric: both query and DB vectors are quantized. Asymmetric: query remains uncompressed, only DB vectors are quantized. Asymmetric gives better accuracy.

**Q29: How do you choose the number of clusters for IVF?**

> **A:** Common heuristic: `nlist = sqrt(N)`. For N=1M, nlist=1000. Adjust based on data distribution and query requirements.

**Q30: What is the memory calculation for storing 10M vectors of 768 dimensions?**

> **A:** Raw: 10M × 768 × 4 bytes = 30.7 GB. HNSW: add ~15 GB for edges. Total ~46 GB. With PQ (M=96): 10M × 96 × 1 = 0.96 GB + centroids ~ negligible.

**Q31: Explain the tradeoff in choosing M parameter for HNSW.**

> **A:** Higher M (>16): faster search, better recall, more memory. Lower M (<16): slower search, lower recall, less memory. M=16 is the recommended default for most use cases.

**Q32: How does quantization affect search quality?**

> **A:** Quantization introduces approximation error. float32→float16: negligible loss (<0.5%). float32→int8: 1-3% recall drop. PQ (M=96): 5-10% drop. Binary quantization: 10-20% drop.

**Q33: What is sharding and why is it needed?**

> **A:** Sharding splits a dataset across multiple machines when it doesn't fit on one. Each shard holds a subset of vectors and searches independently. Results are merged by a coordinator.

**Q34: What is consistent hashing?**

> **A:** A hash-ring-based technique for distributing data across nodes where adding/removing nodes only requires relocating ~1/N of data, minimizing disruption.

**Q35: How does replication affect read and write throughput?**

> **A:** Write throughput decreases (each write goes to N replicas). Read throughput increases (reads can be served by any replica). Replication factor = R → R-1 node fault tolerance.

**Q36: What is DiskANN and when would you use it?**

> **A:** Microsoft's SSD-based ANN algorithm. Use when your dataset is too large for available RAM. Vectors are stored on SSD with a RAM cache for hot nodes.

**Q37: How does GPU acceleration help vector search?**

> **A:** GPUs excel at matrix operations (batch distance computation, k-means training). Speedup: 50x for brute force, 30x for index training, 2x for HNSW search.

**Q38: What is the role of a cross-encoder in retrieval?**

> **A:** Cross-encoders jointly process query + document for accurate relevance scoring. Much more accurate than bi-encoders but too slow for initial retrieval (used for reranking top-50/-100).

**Q39: How do you measure recall in production?**

> **A:** Sample queries → run exact KNN → compare ANN results → compute recall@k. Monitor over time to detect index degradation.

**Q40: What is the difference between vector database and vector indexing library (FAISS)?**

> **A:** FAISS is a library for vector index construction and search. A vector database adds persistence, CRUD operations, filtering, distributed architecture, replication, and a query API.

**Q41: How would you design a multi-tenant vector database?**

> **A:** Options: (1) Tenant ID as metadata filter. (2) Separate collection per tenant. (3) Separate DB instance per tenant. Choice depends on number of tenants, data per tenant, isolation requirements.

**Q42: What is HyDE (Hypothetical Document Embedding)?**

> **A:** A technique where the LLM generates a hypothetical document answering the query, then the hypothetical document's embedding is used for retrieval. Improves retrieval for complex queries.

**Q43: What is the optimal chunk size for RAG?**

> **A:** Typically 300-1000 tokens. Depends on content type: short chunks for precise retrieval, larger chunks for more context. Use parent-child chunking for best results.

**Q44: How do you handle vector updates in production?**

> **A:** Options: (1) Delete + re-insert (standard). (2) Update in place (if supported). (3) Soft delete with tombstone. (4) Version-based with periodic compaction.

**Q45: What is the difference between dot product and cosine similarity?**

> **A:** Dot product = Σ(ai × bi). Cosine similarity = dot(A,B) / (||A|| × ||B||). For normalized vectors, they're equivalent. Dot product preserves magnitude info, cosine doesn't.

**Q46: How would you benchmark a vector database?**

> **A:** Use ANN benchmarks library. Measure: QPS (queries per second), latency (p50/p95/p99), recall@k, build time, memory usage. Test at different scales and concurrency levels.

**Q47: What is the impact of increasing dimensions on vector search?**

> **A:** Higher dimensions = more compute per distance calculation, stronger curse of dimensionality, more memory. Benefits: better semantic representation, higher accuracy potential.

**Q48: How does sliding window chunking work?**

> **A:** Create overlapping chunks by sliding a window across text by a fixed stride. Each token appears in multiple chunks. Maximizes context preservation at the cost of more storage.

**Q49: What is adaptive chunking?**

> **A:** ML-based chunking that determines optimal chunk boundaries based on content. Can split at topic changes or semantic boundaries rather than fixed token counts.

**Q50: How do you handle out-of-vocabulary words in sparse retrieval?**

> **A:** Sparse retrieval (BM25) can only match exact tokens. For OOV terms, there's no match. Dense retrieval handles OOV through sub-word tokenization in the embedding model.

---

### Advanced Level (51-75)

**Q51: Explain the HNSW insertion algorithm in detail.**

> **A:** (1) Assign a random layer L using exponential distribution. (2) Find entry point at top layer. (3) Greedy traverse to nearest at each layer down to L+1. (4) At layer L and below: find efConstruction nearest neighbors. (5) Select M neighbors using heuristic. (6) Connect bidirectionally. (7) Prune existing connections if needed.

**Q52: How does HNSW's heuristic neighbor selection work?**

> **A:** The heuristic prioritizes diverse coverage: select closest candidate first, then add subsequent candidates only if they're closer to the query than to any already-selected neighbor. This prevents all neighbors from being in one direction.

**Q53: Explain the concept of "small world" graphs in ANN.**

> **A:** Small world graphs have O(log N) diameter — any node can reach any other in few hops. This makes them ideal for search: you can traverse from a random node to the query's nearest neighbor in O(log N) steps.

**Q54: What is the complexity of HNSW search and build?**

> **A:** Search: O(log N) greedy traversals × O(M) distance computations per hop. Build: O(N log N) — each insertion traverses the graph. Memory: O(N × M) edges.

**Q55: How does PQ achieve better recall than Scalar Quantization at the same compression ratio?**

> **A:** PQ learns cluster centroids per sub-space, adapting to data distribution. SQ applies uniform quantization per dimension regardless of distribution. PQ's learned codebooks better preserve relative distances.

**Q56: What is OPQ and how does it improve PQ?**

> **A:** Optimized Product Quantization adds a rotation matrix to align data variance with sub-vector boundaries. This reduces quantization error, improving recall by 1-5% at the same compression ratio.

**Q57: Explain the asymmetry in PQ distance computation.**

> **A:** ADC (Asymmetric Distance Computation): query is kept as float32, DB vectors are codes. For each sub-vector, pre-compute distance from query sub-vector to all codebook centroids. Lookup in O(1) per sub-vector. The query isn't quantized, preserving accuracy.

**Q58: How would you implement a hybrid search system?**

> **A:** (1) Index documents in both vector DB (dense) and inverted index (sparse). (2) For each query, run both searches in parallel. (3) Merge using RRF (reciprocal rank fusion). (4) Rerank with cross-encoder. (5) Return top-k.

**Q59: What is the CAP theorem tradeoff in vector databases?**

> **A:** Vector DBs typically prioritize availability and partition tolerance over strong consistency. Most use eventual consistency for vector indexes (which are "always overwritable") with strong consistency for metadata.

**Q60: How do you handle vector database backup and restore?**

> **A:** (1) WAL-based continuous archiving for durability. (2) Periodic snapshot of index files. (3) For restore: replay WAL from last snapshot. (4) For large-scale: use replication for high availability, skip traditional backup.

**Q61: What is SPANN and how does it differ from DiskANN?**

> **A:** SPANN uses a partitioned inverted index on SSD (cluster-based) while DiskANN uses a Vamana graph. SPANN generally has better recall at the cost of more RAM. Both are Microsoft algorithms for billion-scale disk-based search.

**Q62: How does LSH work for nearest neighbor search?**

> **A:** LSH uses random projections to hash similar vectors to the same bucket with high probability. Multiple hash tables increase recall. Query hashes to a bucket and only compares against vectors in the same bucket.

**Q63: What is the theory behind LSH?**

> **A:** LSH families have the property: if d(p,q) < r1 → P(h(p)=h(q)) ≥ p1, and if d(p,q) > r2 → P(h(p)=h(q)) ≤ p2. Where r1 < r2 and p1 > p2. This guarantees similar vectors collide more often.

**Q64: How do you choose the number of shards?**

> **A:** shards = ceil(total_vectors / vectors_per_shard). Target 50-80% memory per shard for headroom. Consider: query concurrency (more shards = more parallelism), rebalancing overhead.

**Q65: Explain the tradeoffs in PQ parameter M selection.**

> **A:** Higher M (more sub-vectors) = more compression = less memory = lower recall. M=dim/8 is common (M=96 for 768d). M=dim/4 gives better recall but 2x memory. M=dim/16 is aggressive compression with notable recall loss.

**Q66: What is the relationship between efConstruction and recall?**

> **A:** efConstruction determines search width during HNSW index building. Higher efConstruction = more thorough neighbor selection = better graph quality = higher recall. Returns diminish after efConstruction ≈ 2-3x M.

**Q67: How do you detect and handle index drift over time?**

> **A:** Index drift: as vectors are added/deleted, index quality degrades. Detection: monitor recall@k against ground-truth queries. Handling: periodic full rebuild, or incremental maintenance (harder for HNSW).

**Q68: What is the role of WAL in vector databases?**

> **A:** Write-Ahead Log ensures durability. All writes are appended to WAL before being applied to the index. On crash, the WAL is replayed. WAL can be truncated after checkpointed.

**Q69: How does memory-mapped I/O work in vector search?**

> **A:** mmap maps a file directly into virtual memory. The OS handles page loading — accessed pages are pulled from disk to RAM on demand. Hot pages stay in RAM (page cache), cold pages are evicted. Allows >RAM datasets with minimal code.

**Q70: Design a system that needs to search 1B vectors with <100ms latency.**

> **A:** Use DiskANN or SPANN for SSD-based approach. Shard across 4-8 machines. Use PQ for compression. Implement read replicas for query scaling. Cache frequent queries in Redis. Monitor recall vs latency tradeoff.

**Q71: How would you implement sliding window chunking efficiently?**

> **A:** Use a deque to maintain the window. Slide by stride tokens, yield window tokens. For overlap: stride < window_size. For zero overlap: stride = window_size. Can parallelize with token-level offsets.

**Q72: What is the impact of embedding model choice on vector DB performance?**

> **A:** Model determines: dimension (affects memory/speed), vector distribution (affects index quality), language coverage, token limit (affects chunking strategy). Switching models invalidates existing indexes.

**Q73: How do you version embeddings in production?**

> **A:** Include model version + dimension in collection name (e.g., "docs_v3_768"). Store version as metadata. Run side-by-side during migration. Gradually rebuild indexes with new model. A/B test before cutover.

**Q74: What is the optimal batch size for vector insertion and why?**

> **A:** 1000-10000 vectors per batch. Too small: high overhead per operation. Too large: memory spike during indexing, potential timeouts. Tune based on vector dimension and hardware.

**Q75: Explain how k-means initialization affects IVF quality.**

> **A:** Poor initialization (random) can lead to empty clusters and unbalanced assignments. Better: k-means++ (spreads initial centroids) or PCA-based initialization (uses principal components). Balanced clusters = better recall.

---

### Expert Level (76-100)

**Q76: Prove that HNSW search has O(log N) complexity.**

> **A:** Each layer reduces the search space exponentially. Top layer has ~1 node, next ~M, next ~M², etc. The number of layers is proportional to log(N) / log(M). Each layer requires O(1) hops. Total: O(log N / log M) = O(log N).

**Q77: How does Vamana (DiskANN's graph structure) differ from HNSW?**

> **A:** Vamana uses a single-layer graph with a "robust" pruning condition. It's designed for SSD: minimizes random I/O by keeping graph structure compact, uses a single-level graph with higher degree rather than multi-level hierarchy.

**Q78: Explain the mathematical relationship between cosine similarity and Euclidean distance for normalized vectors.**

> **A:** For ||A|| = ||B|| = 1: ||A - B||² = 2 - 2cos(A,B). So Euclidean² = 2(1 - cosine_sim). This means cosine similarity ranks are equivalent to Euclidean distance ranks for normalized vectors.

**Q79: How does the ADSampling algorithm in ScaNN work?**

> **A:** Anisotropic Distance Sampling: instead of minimizing reconstruction error (standard PQ), it optimizes quantization to preserve ranking order. It up-weights directions where small differences matter more for search quality.

**Q80: What is the theoretical lower bound for ANN search complexity?**

> **A:** The "curse of dimensionality" lower bound suggests any data structure for exact NN requires O(exp(d)) for worst-case queries in d dimensions. ANN relaxes this, with practical algorithms achieving O(poly(log N, 1/ε)) where ε is approximation error.

**Q81: Explain the role of multi-vector retrieval in RAG.**

> **A:** Instead of one vector per document, use multiple vectors (one per sentence, or late interaction like ColBERT). Enables finer-grained matching: query tokens match relevant document tokens independently, then aggregate scores.

**Q82: How does contrastive learning improve embeddings?**

> **A:** Contrastive learning pulls similar pairs together and pushes dissimilar pairs apart in embedding space. This creates more structured embedding spaces with clearer cluster separation, improving retrieval quality.

**Q83: What is late interaction (ColBERT-style) retrieval?**

> **A:** Both query and document are embedded as multiple token-level vectors. Similarity = max over query tokens of max over doc tokens of cosine similarity. Enables fine-grained matching without a cross-encoder's quadratic cost.

**Q84: How would you handle real-time updates in HNSW?**

> **A:** HNSW supports online insertion by design. Deletion is harder: use tombstones (mark as deleted, skip in search). Periodic rebuild cleans tombstones. Some implementations (Qdrant) rebuild affected segments.

**Q85: What is the impact of data distribution on ANN index quality?**

> **A:** Uniform data → easy for all indexes. Clustered data → IVF works well (clusters match data clusters). Adversarial data → hard for all (worst-case). Real-world data is usually clustered, which ANN algorithms exploit.

**Q86: Design a vector database from scratch. What components would you build?**

> **A:** (1) Storage engine (WAL + segment files). (2) Index module (HNSW + IVF implementations). (3) Query planner (filter optimization). (4) Distributed coordinator (sharding + replication). (5) API layer (gRPC + REST). (6) Background tasks (compaction, optimization).

**Q87: How does RAFT consensus work in vector database replication?**

> **A:** Leader election → all writes go to leader → leader replicates to followers via log entries → on leader failure, followers elect new leader via majority vote. Ensures linearizable writes, eventually consistent reads.

**Q88: Explain the tradeoff between write-ahead logging and index performance.**

> **A:** WAL ensures durability but adds write latency (fsync per write). Batching WAL flushes improves throughput at cost of durability window. Some systems use group commit to amortize fsync over multiple writes.

**Q89: How do you debug a sudden drop in recall?**

> **A:** (1) Check if new vectors use the same embedding model. (2) Check for corrupted index files. (3) Verify filter predicates aren't accidentally excluding results. (4) Check if efSearch was changed. (5) Compare against ground-truth query set.

**Q90: What is the optimal strategy for index rebuild frequency?**

> **A:** Depends on: (1) Write rate → rebuild when 20%+ vectors are new. (2) Delete rate → rebuild when 10%+ are tombstones. (3) Time-based → weekly rebuild for moderate churn. (4) Event-based → rebuild after bulk loads.

**Q91: Explain the memory vs. accuracy tradeoff in ANN indexes mathematically.**

> **A:** Let M be memory budget. HNSW can store ~M/(d×4 + 2M×4) vectors. PQ can store ~M/(d/8) vectors. For same M, PQ handles 32x more vectors at 5-10% lower recall. The "Pareto frontier" of accuracy vs memory is the key design choice.

**Q92: How does the choice of distance metric affect index performance?**

> **A:** Cosine: requires normalized vectors, efficient with dot product indexes. Euclidean: naturally handles magnitude, wider index support. Inner product: non-metric (breaks triangle inequality), fewer index types support it natively.

**Q93: What techniques exist for billion-scale vector search on a single machine?**

> **A:** (1) DiskANN (SSD-based Vamana graph). (2) SPANN (SSD-based cluster index). (3) PQ compression (fit vectors in <50 GB). (4) Hierarchical quantization: coarse + fine. (5) Hybrid: GPU-accelerated batch processing + disk-based search.

**Q94: How would you implement vector search on streaming data?**

> **A:** (1) Micro-batch approach: buffer, batch insert, apply index updates. (2) Window-based: only index recent data. (3) Two-tier: hot (RAM, recent) + cold (SSD, historical). (4) Incremental HNSW: insert without rebuild.

**Q95: Explain the Bloom filter-like technique for approximate membership testing in vectors.**

> **A:** LSH-based: hash vectors into buckets, maintain a bloom filter per bucket. Query-only search buckets where the query's LSH hash matches. Not exact but can prune 99%+ of candidates.

**Q96: How does the DRAM latency gap (CPU vs RAM vs SSD) affect ANN architecture?**

> **A:** L1: 1ns, RAM: 100ns, SSD: 10μs. This 10,000x gap means DiskANN must minimize random SSD reads by batching, prefetching, and keeping hot nodes in RAM. HNSW avoids this entirely by keeping everything in RAM.

**Q97: What is the role of SIMD (AVX-512, SVE) in vector search?**

> **A:** SIMD processes 8-16 floats per instruction vs 1 for scalar. Distance computation (the bottleneck) benefits massively: 8-16x speedup. Modern implementations (FAISS, ScaNN) are heavily SIMD-optimized.

**Q98: How would you design a cross-modal search system (text→image)?**

> **A:** Use CLIP or similar multimodal model that embeds text and images in the same space. Index all images. Query with text embedding. Also supports image→text and text→text search within the same index.

**Q99: Compare the theoretical foundations of tree-based and graph-based ANN.**

> **A:** Trees (kd-tree, rp-tree, Annoy) partition space recursively. Search descends one path per level. Graphs (HNSW, NSG) connect nearby points. Search follows multiple paths in parallel. Graphs generally outperform because they handle high dimensions better (space partitioning becomes inefficient above ~20 dimensions).

**Q100: What do you see as the future of vector databases?**

> **A:** (1) Convergence with traditional databases (all DBs will have vector indexes). (2) Streaming vector search for real-time AI. (3) Multimodal-native (text/image/audio/3D all in one space). (4) Learned indexes (ML to optimize the index structure itself). (5) Hardware-optimized (DPU, TPU, neuromorphic).

---

