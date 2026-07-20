# Qdrant: Advanced Filter Strategies

> Author: **Tamilselvan** · ✉️ tamilselvan.sde@gmail.com · 🔗 [LinkedIn](https://www.linkedin.com/in/tamilselvan-ai/)
>


### Custom Payload Indexes

```python
from qdrant_client import QdrantClient, models

client = QdrantClient("localhost:6333")

# Create collection with optimized payload indexes
client.create_collection(
    collection_name="advanced_docs",
    vectors_config=models.VectorParams(
        size=768,
        distance=models.Distance.COSINE
    ),
    optimizers_config=models.OptimizersConfigDiff(
        default_segment_number=8,
        memmap_threshold_kb=50000,
        indexing_threshold=10000,
    ),
    # Pre-create payload indexes for common filter fields
    payload_schema={
        "tenant_id": models.PayloadSchemaType.KEYWORD,
        "date": models.PayloadSchemaType.DATETIME,
        "price": models.PayloadSchemaType.FLOAT,
        "is_available": models.PayloadSchemaType.BOOL,
        "category": models.PayloadSchemaType.KEYWORD,
        "view_count": models.PayloadSchemaType.INTEGER,
        "tags": models.PayloadSchemaType.KEYWORD,
    }
)
```

### Complex Filtering Examples

```python
# Range filter with nested conditions
filter_complex = models.Filter(
    must=[
        models.FieldCondition(
            key="tenant_id",
            match=models.MatchValue(value="acme_corp")
        ),
        models.FieldCondition(
            key="date",
            range=models.Range(
                gte="2024-01-01T00:00:00Z",
                lte="2024-12-31T23:59:59Z"
            )
        ),
    ],
    should=[
        models.FieldCondition(
            key="category",
            match=models.MatchAny(any=["engineering", "research"])
        ),
        models.FieldCondition(
            key="tags",
            match=models.MatchAny(
                any=["vector-search", "machine-learning"]
            )
        ),
    ],
    must_not=[
        models.FieldCondition(
            key="status",
            match=models.MatchValue(value="archived")
        ),
        models.FieldCondition(
            key="view_count",
            range=models.Range(lte=10)
        ),
    ]
)

# Execute with complex filter
results = client.search(
    collection_name="advanced_docs",
    query_vector=query_vector,
    query_filter=filter_complex,
    limit=20,
    with_payload=True,
    score_threshold=0.7,  # Minimum similarity threshold
)
```

### Geo-Filtering with Vector Search

```python
# Geo-radius filter
geo_filter = models.Filter(
    must=[
        models.FieldCondition(
            key="location",
            geo_radius=models.GeoRadius(
                center=models.GeoPoint(
                    lon=-73.9857,
                    lat=40.7484
                ),
                radius=5000  # 5km radius
            )
        ),
        models.FieldCondition(
            key="rating",
            range=models.Range(gte=4.0)
        )
    ]
)

# Geo-polygon filter (custom geographic region)
geo_polygon_filter = models.Filter(
    must=[
        models.FieldCondition(
            key="location",
            geo_polygon=models.GeoPolygon(
                exterior=models.GeoLineString(
                    points=[
                        models.GeoPoint(lon=-74.01, lat=40.70),
                        models.GeoPoint(lon=-73.95, lat=40.70),
                        models.GeoPoint(lon=-73.95, lat=40.80),
                        models.GeoPoint(lon=-74.01, lat=40.80),
                        models.GeoPoint(lon=-74.01, lat=40.70),
                    ]
                )
            )
        )
    ]
)
```

---

## Hybrid Search Implementation (Qdrant + BM25)

```python
import json
import numpy as np
from rank_bm25 import BM25Okapi
from qdrant_client import QdrantClient, models
from typing import List, Tuple, Dict

class ProductionHybridSearcher:
    def __init__(self, 
                 qdrant_host: str = "localhost",
                 collection: str = "documents",
                 embedder_model: str = "all-MiniLM-L6-v2"):
        
        self.qdrant = QdrantClient(host=qdrant_host)
        self.collection = collection
        from sentence_transformers import SentenceTransformer
        self.embedder = SentenceTransformer(embedder_model)
        
        # BM25 index (loaded from Qdrant payloads)
        self.bm25_index = None
        self.doc_texts = []
        self.doc_ids = []
        
    def load_bm25_index(self):
        """Build BM25 index from stored documents."""
        # Scroll through all documents
        next_offset = None
        while True:
            records, next_offset = self.qdrant.scroll(
                collection_name=self.collection,
                limit=100,
                offset=next_offset,
                with_payload=["text"]
            )
            if not records:
                break
            for record in records:
                self.doc_texts.append(record.payload["text"])
                self.doc_ids.append(record.id)
        
        tokenized = [doc.split() for doc in self.doc_texts]
        self.bm25_index = BM25Okapi(tokenized)
    
    def dense_search(self, query: str, k: int = 50) -> List[Tuple[int, float]]:
        """Pure vector search."""
        query_vec = self.embedder.encode(query)
        results = self.qdrant.search(
            collection_name=self.collection,
            query_vector=query_vec,
            limit=k,
        )
        return [(r.id, r.score) for r in results]
    
    def sparse_search(self, query: str, k: int = 50) -> List[Tuple[int, float]]:
        """Pure BM25 search."""
        tokenized_query = query.split()
        scores = self.bm25_index.get_scores(tokenized_query)
        
        # Get top-k
        top_indices = np.argsort(scores)[-k:][::-1]
        return [
            (self.doc_ids[i], scores[i])
            for i in top_indices
        ]
    
    def hybrid_search(self, 
                      query: str, 
                      k: int = 10,
                      alpha: float = 0.7,
                      rrf_k: int = 60) -> List[Dict]:
        """Hybrid search with RRF fusion."""
        
        # Run both searches in parallel
        dense_results = self.dense_search(query, k=50)
        sparse_results = self.sparse_search(query, k=50)
        
        # RRF fusion
        combined_scores = {}
        
        for rank, (doc_id, _) in enumerate(dense_results):
            combined_scores[doc_id] = combined_scores.get(doc_id, 0) + \
                (1 - alpha) * (1 / (rrf_k + rank + 1))
        
        for rank, (doc_id, _) in enumerate(sparse_results):
            combined_scores[doc_id] = combined_scores.get(doc_id, 0) + \
                alpha * (1 / (rrf_k + rank + 1))
        
        # Sort and get top-k
        ranked = sorted(
            combined_scores.items(),
            key=lambda x: -x[1]
        )[:k]
        
        # Retrieve full payloads
        final_results = []
        for doc_id, score in ranked:
            point = self.qdrant.retrieve(
                collection_name=self.collection,
                ids=[doc_id],
                with_payload=True
            )[0]
            final_results.append({
                "id": doc_id,
                "score": score,
                "payload": point.payload
            })
        
        return final_results
    
    def search_with_filters(self,
                            query: str,
                            filter_conditions: dict,
                            k: int = 10) -> List[Dict]:
        """Hybrid search with metadata filtering."""
        query_vec = self.embedder.encode(query)
        
        # Build Qdrant filter
        qdrant_filter = models.Filter(
            must=[
                models.FieldCondition(
                    key=key,
                    match=models.MatchValue(value=value)
                )
                for key, value in filter_conditions.items()
            ]
        )
        
        # Vector search with filter (faster than post-filtering)
        results = self.qdrant.search(
            collection_name=self.collection,
            query_vector=query_vec,
            query_filter=qdrant_filter,
            limit=k
        )
        
        return [
            {"id": r.id, "score": r.score, "payload": r.payload}
            for r in results
        ]
```

---

## RAG with Self-Correction

```python
class SelfCorrectingRAG:
    """RAG that detects and fixes bad retrievals."""
    
    def __init__(self, retriever, llm):
        self.retriever = retriever
        self.llm = llm
        self.max_retries = 3
    
    def answer(self, question: str) -> dict:
        history = []
        
        for attempt in range(self.max_retries):
            # Retrieve
            docs = self.retriever.retrieve(question, k=5)
            
            # Check retrieval quality
            quality = self._check_retrieval_quality(question, docs)
            
            if quality["adequate"]:
                # Generate answer
                answer = self._generate(question, docs)
                
                # Verify answer
                verification = self._verify_answer(question, answer, docs)
                
                if verification["correct"]:
                    return {
                        "answer": answer,
                        "sources": docs,
                        "attempts": attempt + 1,
                        "confidence": verification["confidence"]
                    }
                else:
                    # Signal: refine retrieval based on what's missing
                    question = self._refine_query(
                        question, verification["missing_info"]
                    )
                    history.append({
                        "attempt": attempt,
                        "issue": verification["issue"]
                    })
            else:
                # Retrieval failed, try broader search
                question = self._broaden_query(question)
                history.append({
                    "attempt": attempt,
                    "issue": quality["issue"]
                })
        
        # Final fallback
        return {
            "answer": self.llm.invoke(
                f"Answer based on general knowledge: {question}"
            ),
            "sources": [],
            "attempts": self.max_retries,
            "confidence": "low"
        }
    
    def _check_retrieval_quality(self, question: str, docs: list) -> dict:
        prompt = f"""
Question: {question}
Retrieved documents:
{chr(10).join(f'- {d}' for d in docs[:3])}

Are these documents ADEQUATE to answer the question?
Respond in JSON: {{"adequate": bool, "issue": "str or null"}}
"""
        return json.loads(self.llm.invoke(prompt))
    
    def _verify_answer(self, question: str, answer: str, docs: list) -> dict:
        prompt = f"""
Question: {question}
Answer: {answer}
Source documents: {docs}

Is the answer fully supported by the sources?
Respond in JSON: 
{{"correct": bool, "confidence": 0.0-1.0, 
  "issue": "str or null", "missing_info": "str or null"}}
"""
        return json.loads(self.llm.invoke(prompt))
    
    def _refine_query(self, question: str, missing: str) -> str:
        return self.llm.invoke(f"""
Original query: {question}
Missing information needed: {missing}
Generate a refined search query to find this information:""")
    
    def _broaden_query(self, question: str) -> str:
        return self.llm.invoke(f"""
Generate a broader version of this query to get more results:
Original: {question}
Broader:""")
```

---

## Vector DB Performance Benchmark Script

```python
import time
import numpy as np
from typing import Callable, List, Tuple
from dataclasses import dataclass

@dataclass
class BenchmarkResult:
    name: str
    build_time: float
    memory_mb: float
    latency_p50: float
    latency_p95: float
    latency_p99: float
    recall_at_10: float
    qps: float
    index_size_mb: float

class VectorDBBenchmark:
    def __init__(self, 
                 dim: int = 768,
                 n_train: int = 100000,
                 n_test: int = 1000,
                 k: int = 10):
        
        self.dim = dim
        self.n_train = n_train
        self.n_test = n_test
        self.k = k
        
        # Generate data
        np.random.seed(42)
        self.train_vectors = np.random.random(
            (n_train, dim)
        ).astype('float32')
        self.test_vectors = np.random.random(
            (n_test, dim)
        ).astype('float32')
        
        # Normalize
        self.train_vectors /= np.linalg.norm(
            self.train_vectors, axis=1, keepdims=True
        )
        self.test_vectors /= np.linalg.norm(
            self.test_vectors, axis=1, keepdims=True
        )
        
        # Ground truth (exact KNN)
        print("Computing ground truth...")
        self.ground_truth = self._compute_ground_truth()
    
    def _compute_ground_truth(self) -> List[List[int]]:
        """Exact KNN for ground truth."""
        gt = []
        for q in self.test_vectors:
            similarities = self.train_vectors @ q
            top_k = np.argsort(-similarities)[:self.k]
            gt.append(top_k.tolist())
        return gt
    
    def benchmark(self,
                  name: str,
                  build_fn: Callable,
                  search_fn: Callable,
                  batch_size: int = 100) -> BenchmarkResult:
        
        print(f"Benchmarking {name}...")
        
        # Build
        t0 = time.time()
        index = build_fn(self.train_vectors)
        build_time = time.time() - t0
        
        # Memory
        import psutil
        process = psutil.Process()
        memory_mb = process.memory_info().rss / 1024 / 1024
        
        # Warmup
        for q in self.test_vectors[:10]:
            search_fn(index, q, self.k)
        
        # Search latency
        latencies = []
        for batch_start in range(0, self.n_test, batch_size):
            batch = self.test_vectors[
                batch_start:batch_start + batch_size
            ]
            t0 = time.time()
            for q in batch:
                search_fn(index, q, self.k)
            latencies.append(
                (time.time() - t0) / len(batch)
            )
        
        latencies = np.array(latencies)
        
        # Recall
        recalls = []
        for i, q in enumerate(self.test_vectors):
            results = search_fn(index, q, self.k)
            found = set(r[0] if isinstance(r, tuple) else r 
                       for r in results)
            gt_set = set(self.ground_truth[i])
            recalls.append(len(found & gt_set) / self.k)
        
        recall_mean = np.mean(recalls)
        
        return BenchmarkResult(
            name=name,
            build_time=build_time,
            memory_mb=memory_mb,
            latency_p50=np.percentile(latencies, 50) * 1000,
            latency_p95=np.percentile(latencies, 95) * 1000,
            latency_p99=np.percentile(latencies, 99) * 1000,
            recall_at_10=recall_mean,
            qps=1.0 / np.mean(latencies),
            index_size_mb=memory_mb - base_memory,
        )
    
    def print_report(self, results: List[BenchmarkResult]):
        """Print formatted benchmark report."""
        header = f"{'Algorithm':<25} {'Build(s)':<10} {'p50(ms)':<10} "
        header += f"{'p99(ms)':<10} {'Recall':<10} {'QPS':<10} {'Mem(MB)':<10}"
        print(header)
        print("=" * 85)
        
        for r in sorted(results, key=lambda x: -x.recall_at_10):
            print(
                f"{r.name:<25} {r.build_time:<10.1f} "
                f"{r.latency_p50:<10.1f} {r.latency_p99:<10.1f} "
                f"{r.recall_at_10:<10.3f} {r.qps:<10.0f} "
                f"{r.memory_mb:<10.0f}"
            )

# Usage
def bench_all():
    benchmark = VectorDBBenchmark(dim=768, n_train=100000)
    
    results = []
    
    # FAISS Flat (baseline)
    def build_flat(vectors):
        index = faiss.IndexFlatIP(768)
        index.add(vectors)
        return index
    
    def search_flat(index, query, k):
        return index.search(query.reshape(1, -1), k)
    
    results.append(
        benchmark.benchmark("FAISS Flat", build_flat, search_flat)
    )
    
    # FAISS HNSW
    def build_hnsw(vectors):
        index = faiss.IndexHNSWFlat(768, 32)
        index.hnsw.efConstruction = 200
        index.add(vectors)
        return index
    
    def search_hnsw(index, query, k):
        index.hnsw.efSearch = 100
        return index.search(query.reshape(1, -1), k)
    
    results.append(
        benchmark.benchmark("FAISS HNSW (M=32)", build_hnsw, search_hnsw)
    )
    
    # FAISS IVF
    def build_ivf(vectors):
        quantizer = faiss.IndexFlatIP(768)
        index = faiss.IndexIVFFlat(quantizer, 768, 316)
        index.train(vectors)
        index.add(vectors)
        return index
    
    def search_ivf(index, query, k):
        index.nprobe = 10
        return index.search(query.reshape(1, -1), k)
    
    results.append(
        benchmark.benchmark("FAISS IVF (nlist=316)", build_ivf, search_ivf)
    )
    
    benchmark.print_report(results)
```

---

## Vector DB Integration Patterns

### Pattern 1: Write-Through Cache

```python
class VectorWriteThroughCache:
    """Synchronous write to both cache and vector DB."""
    
    def __init__(self, vector_db, cache, ttl=300):
        self.vdb = vector_db
        self.cache = cache
        self.ttl = ttl
    
    def upsert(self, vectors, payloads):
        # Write to vector DB
        ids = self.vdb.upsert(vectors, payloads)
        
        # Write to cache
        for vid, vec, payload in zip(ids, vectors, payloads):
            cache_key = f"vec:{vid}"
            self.cache.setex(
                cache_key,
                self.ttl,
                json.dumps({
                    "vector": vec.tolist(),
                    "payload": payload
                })
            )
        
        return ids
    
    def search(self, query_vec, k=10):
        # Try cache first
        cache_key = f"query:{hash(query_vec.tobytes())}"
        cached = self.cache.get(cache_key)
        if cached:
            return json.loads(cached)
        
        # Fallback to vector DB
        results = self.vdb.search(query_vec, k)
        
        # Cache for future
        self.cache.setex(cache_key, self.ttl, json.dumps(results))
        return results
```

### Pattern 2: Event-Driven Ingestion

```python
import asyncio
from kafka import KafkaConsumer, KafkaProducer
import json

class AsyncVectorIngestion:
    """Event-driven ingestion pipeline."""
    
    def __init__(self, vector_db, 
                 bootstrap_servers=['localhost:9092']):
        self.vdb = vector_db
        self.consumer = KafkaConsumer(
            'document-ingest',
            bootstrap_servers=bootstrap_servers,
            value_deserializer=lambda m: json.loads(m.decode()),
            enable_auto_commit=False,
            max_poll_records=500,
        )
        self.batch_buffer = []
        self.batch_size = 100
    
    async def process_message(self, message):
        self.batch_buffer.append(message)
        
        if len(self.batch_buffer) >= self.batch_size:
            await self.flush()
    
    async def flush(self):
        if not self.batch_buffer:
            return
        
        # Process batch
        vectors = []
        payloads = []
        
        for msg in self.batch_buffer:
            embeddings = await self.embed(msg['text'])
            vectors.append(embeddings)
            payloads.append({
                'id': msg['id'],
                'text': msg['text'],
                'source': msg.get('source'),
                'timestamp': msg.get('timestamp'),
            })
        
        # Batch insert
        ids = self.vdb.upsert(vectors, payloads)
        
        # Commit Kafka offset
        self.consumer.commit()
        self.batch_buffer = []
        
        print(f"Indexed {len(ids)} documents")
    
    async def run(self):
        for message in self.consumer:
            await self.process_message(message.value)
    
    async def embed(self, text):
        # Async embedding API call
        async with aiohttp.ClientSession() as session:
            async with session.post(
                "http://embedding-service:8001/embed",
                json={"text": text}
            ) as resp:
                result = await resp.json()
                return result['embedding']
```

### Pattern 3: CQRS (Command Query Responsibility Segregation)

```python
class VectorCQRS:
    """Separate write and read paths."""
    
    def __init__(self):
        # Write path
        self.write_queue = asyncio.Queue()
        self.write_db = VectorDatabase(write_optimized=True)
        self.index_builder = IndexBuilder()
        
        # Read path
        self.read_db = VectorDatabase(read_optimized=True)
        self.read_replicas = [
            VectorDatabase(read_optimized=True)
            for _ in range(3)
        ]
        
        # Sync
        self.sync_interval = 60  # seconds
    
    async def write(self, vectors, payloads):
        """Write path - accepts and queues."""
        await self.write_queue.put((vectors, payloads))
        
        # Acknowledge immediately
        return {"status": "queued", "count": len(vectors)}
    
    async def read(self, query_vec, k=10):
        """Read path - balanced across replicas."""
        import random
        db = random.choice(self.read_replicas)
        return db.search(query_vec, k)
    
    async def background_worker(self):
        """Process write queue and sync to read replicas."""
        while True:
            vectors, payloads = await self.write_queue.get()
            
            # Write to primary
            ids = self.write_db.upsert(vectors, payloads)
            
            # Rebuild index periodically
            if self.write_queue.qsize() == 0:
                self.index_builder.rebuild(self.write_db)
                
                # Sync to read replicas
                snapshot = self.write_db.snapshot()
                for replica in self.read_replicas:
                    replica.load_snapshot(snapshot)
```

---

## Production Alerting Rules

```yaml
# prometheus-alerts.yml
groups:
  - name: vector_db
    rules:
      - alert: HighVectorSearchLatency
        expr: histogram_quantile(0.99, vector_search_latency_seconds) > 0.1
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "p99 latency above 100ms"
          
      - alert: CriticalVectorSearchLatency
        expr: histogram_quantile(0.99, vector_search_latency_seconds) > 0.5
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: "p99 latency above 500ms"
          
      - alert: LowRecall
        expr: vector_search_recall < 0.90
        for: 10m
        labels:
          severity: warning
        annotations:
          summary: "Recall below 90%"
          
      - alert: CriticalRecallDrop
        expr: vector_search_recall < 0.80
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: "Recall dropped below 80%"
          
      - alert: HighMemoryUsage
        expr: vector_index_memory_bytes / 1e9 > 80
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Vector index using >80GB RAM"
          
      - alert: OOMRisk
        expr: vector_index_memory_bytes / 1e9 > 90
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: "Memory >90%, OOM risk imminent"
          
      - alert: HighErrorRate
        expr: rate(vector_search_queries_total{status="error"}[5m]) > 0.01
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: "Error rate >1%"
          
      - alert: IndexBuildFailure
        expr: vector_index_build_status == 0
        for: 0m
        labels:
          severity: critical
        annotations:
          summary: "Index build failed"
```

---

## Common Migration Patterns

### Pattern 1: Blue-Green Migration

```mermaid
graph TB
    subgraph "Phase 1: Setup Green"
        A[Blue (current)] --> B[Production traffic]
        C[Green (new)] --> D[Building index]
    end
    
    subgraph "Phase 2: Dual Write"
        A --> E[Write to both]
        C --> E
        E --> B
    end
    
    subgraph "Phase 3: Cutover"
        B -.-x A
        C --> B
    end
```

```python
def blue_green_migration(source_db, target_db, chunk_size=1000):
    """Migrate vectors from source to target DB."""
    
    # Phase 2: Dual write (enable on both)
    def dual_write(vectors, payloads):
        source_ids = source_db.upsert(vectors, payloads)
        target_ids = target_db.upsert(vectors, payloads)
        return source_ids  # Keep source as primary
    
    # Phase 3: Backfill
    offset = 0
    while True:
        batch = source_db.scroll(limit=chunk_size, offset=offset)
        if not batch:
            break
        target_db.upsert(batch.vectors, batch.payloads)
        offset += chunk_size
    
    print(f"Migration complete: {offset} vectors")
```

### Pattern 2: Zero-Downtime Embedding Model Change

```python
def migrate_embedding_model(old_model, new_model, vector_db):
    """Change embedding model without downtime."""
    
    # 1. Create new collection with new dimension
    new_dim = len(new_model.encode("test"))
    vector_db.create_collection(
        "documents_v2",
        dimension=new_dim
    )
    
    # 2. Background re-indexing
    def reindex_documents():
        offset = 0
        while True:
            docs = vector_db.scroll(
                collection="documents",  # old collection
                limit=100,
                offset=offset
            )
            if not docs:
                break
            
            # Re-embed with new model
            new_vectors = new_model.encode([
                d.payload["text"] for d in docs
            ])
            
            # Insert to new collection
            vector_db.upsert(
                collection="documents_v2",
                vectors=new_vectors,
                payloads=[d.payload for d in docs]
            )
            offset += 100
    
    # 3. Update application to query both collections
    #    and merge results during transition
    def hybrid_query(query_vec_old, query_vec_new, k=10):
        old_results = vector_db.search(
            "documents", query_vec_old, k
        )
        new_results = vector_db.search(
            "documents_v2", query_vec_new, k
        )
        return merge_by_rrf(old_results, new_results)
    
    # 4. After re-indexing complete, switch to new collection
    import threading
    thread = threading.Thread(target=reindex_documents)
    thread.start()
    
    return hybrid_query  # Use this during transition
```

---

