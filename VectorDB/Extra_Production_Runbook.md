# Production Runbook: Common Incidents

> Author: **Tamilselvan** · ✉️ tamilselvan.sde@gmail.com · 🔗 [LinkedIn](https://www.linkedin.com/in/tamilselvan-ai/)
>


### Incident 1: Search Latency Spike

```markdown
## Incident: Search Latency Spike above 500ms p99

### Symptoms
- P99 latency > 500ms (baseline: 50ms)
- QPS stable or decreasing
- No deployment changes

### Immediate Actions
1. Check if index was rebuilt (fresh build often has high latency)
2. Check memory usage (OOM pressure causes swapping)
3. Check for slow queries in logs
4. Verify index type and parameters

### Diagnosis
```bash
# Qdrant: check segment info
curl http://localhost:6333/collections/my_docs | jq '.result.segments_count'

# Check for unoptimized segments
curl http://localhost:6333/collections/my_docs | jq '.result.optimizer_status'

# Check memory
free -h

# Check swap usage
vmstat 1 5
```

### Resolution
1. If index rebuild: wait for completion (usually minutes)
2. If memory pressure: add RAM or enable PQ compression
3. If slow queries: identify and optimize
4. If index needs rebuild: trigger manual optimization

### Prevention
- Set up latency alert at 2x baseline
- Pre-warm indexes before traffic
- Schedule rebuilds during low traffic
```

### Incident 2: Recall Degradation

```markdown
## Incident: Recall Dropped Below 90%

### Symptoms
- User complaints about irrelevant results
- Dashboard shows recall < 90%
- No change in query patterns

### Immediate Actions
1. Compute recall against ground truth
2. Check if embedding model changed
3. Check if new vectors are being added with wrong model
4. Verify metadata filters aren't too restrictive

### Diagnosis
```python
# Quick recall check
def quick_recall_check(index, test_queries, test_gt):
    total_recall = 0
    for q, gt in zip(test_queries, test_gt):
        D, I = index.search(q.reshape(1, -1), 10)
        recall = len(set(I[0]) & set(gt)) / 10
        total_recall += recall
    return total_recall / len(test_queries)

recall = quick_recall_check(index, queries, ground_truth)
print(f"Current recall: {recall:.2%}")
```

### Resolution
1. If model changed: rebuild index with correct model
2. If vectors drifted: retrain index or rebuild
3. If filters too restrictive: adjust filter logic
4. Increase efSearch from 100 to 200-500

### Prevention
- Monitor recall hourly with ground-truth queries
- Validate all vectors have the correct dimension and model version
- Automate recall alerting
```

---

## Choosing the Right Embedding Model: Decision Flow

```mermaid
graph TD
    A[Need Embeddings?] --> B{Language?}
    B -->|English only| C{Accuracy needed?}
    B -->|Multilingual| D[BGE-M3<br/>Cohere Multilingual<br/>Jina v3]
    
    C -->|High| E{Budget?}
    C -->|Moderate| F{Latency?}
    C -->|Low| G["all-MiniLM-L6-v2<br/>(384d, fast)"]
    
    E -->|Free| H[BGE-large<br/>E5-large<br/>Instructor-XL]
    E -->|Paid| I[OpenAI text-3-large<br/>Cohere Embed v3]
    
    F -->|Low latency| J[BGE-base<br/>(768d, <10ms)]
    F -->|Can wait| K[BGE-large<br/>(1024d, <50ms)]
    
    H --> L{Dimensions?}
    I --> L
    
    L -->|384| M["all-MiniLM-L6-v2"]
    L -->|768| N["BGE-base<br/>GTE-base<br/>Instructor-base"]
    L -->|1024| O["BGE-large<br/>E5-large"]
    L -->|1536| P["OpenAI ada-002<br/>text-3-small"]
    L -->|3072| Q["OpenAI text-3-large"]
    
    M --> R{Domain?}
    N --> R
    O --> R
    P --> R
    Q --> R
    
    R -->|General| S[✓ Use selected]
    R -->|Code| T["CodeBERT<br/>Starcoder"]
    R -->|Medical| U["BioBERT<br/>PubMedBERT"]
    R -->|Legal| V["Legal-BERT<br/>CaseLaw"]
    R -->|Finance| W["FinBERT<br/>Sentence-BERT finetuned"]
```

---

## Benchmarks: Detailed Performance Data

### ANN Benchmarks on Standard Datasets

| Dataset | N | Dim | Algorithm | Recall@10 | QPS | Build Time |
|---------|---|-----|-----------|-----------|-----|------------|
| **SIFT1M** | 1M | 128 | HNSW (M=16) | 99.2% | 12,500 | 42s |
| **SIFT1M** | 1M | 128 | IVF (nlist=1000) | 95.1% | 8,200 | 28s |
| **SIFT1M** | 1M | 128 | IVF_PQ (M=64) | 90.3% | 15,000 | 35s |
| **GIST1M** | 1M | 960 | HNSW (M=32) | 98.7% | 3,200 | 180s |
| **GIST1M** | 1M | 960 | IVF_PQ (M=64) | 88.5% | 8,500 | 95s |
| **GLOVE-100** | 1.2M | 100 | HNSW (M=16) | 99.5% | 18,000 | 55s |
| **DEEP1M** | 1M | 256 | HNSW (M=16) | 99.0% | 8,900 | 65s |
| **DEEP1M** | 1M | 256 | ScaNN | 98.5% | 22,000 | 120s |
| **Yandex DEEP** | 10M | 96 | HNSW | 98.5% | 6,500 | 380s |
| **Yandex Text-to-Image** | 10M | 200 | DiskANN | 95.2% | 2,100 | 900s |

> **Note:** Benchmarks vary by hardware. These were measured on an AWS r6i.8xlarge (32 vCPU, 256GB RAM). Always benchmark on your own hardware and data.

---

### Scalability Test Results

Using FAISS HNSW with varying dataset sizes:

```python
import time
import numpy as np
import faiss

def scalability_test(sizes=[100_000, 500_000, 1_000_000, 5_000_000]):
    results = []
    
    for n in sizes:
        # Generate data
        vectors = np.random.random((n, 768)).astype('float32')
        query = np.random.random(768).astype('float32')
        
        # Build index
        t0 = time.time()
        index = faiss.IndexHNSWFlat(768, 16)
        index.hnsw.efConstruction = 200
        index.add(vectors)
        build_time = time.time() - t0
        
        # Search
        index.hnsw.efSearch = 100
        t0 = time.time()
        for _ in range(100):
            index.search(query.reshape(1, -1), 10)
        search_time = (time.time() - t0) / 100 * 1000  # ms
        
        results.append({
            "n": n,
            "build_time_s": round(build_time, 1),
            "search_ms": round(search_time, 2),
            "memory_gb": round(
                (n * 768 * 4 + n * 16 * 2 * 4) / 1e9, 1
            ),
        })
        
        print(f"N={n:>8}: build={build_time:6.1f}s  "
              f"search={search_time:5.2f}ms  "
              f"mem={results[-1]['memory_gb']:.1f}GB")
    
    return results

# Expected output:
# N=  100000: build=   5.2s  search= 0.85ms  mem= 0.4GB
# N=  500000: build=  28.5s  search= 1.20ms  mem= 2.1GB
# N= 1000000: build=  58.0s  search= 1.50ms  mem= 4.3GB
# N= 5000000: build= 312.0s  search= 2.10ms  mem=21.4GB
```

---

## Complete Docker Compose for Vector DB Stack

```yaml
version: '3.8'

services:
  # Vector Database (Qdrant)
  qdrant:
    image: qdrant/qdrant:v1.8.0
    restart: always
    ports:
      - "6333:6333"  # HTTP
      - "6334:6334"  # gRPC
    volumes:
      - ./qdrant_storage:/qdrant/storage
      - ./qdrant_config.yaml:/qdrant/config/production.yaml
    environment:
      - QDRANT__SERVICE__GRPC_PORT=6334
      - QDRANT__SERVICE__HTTP_PORT=6333
      - QDRANT__LOG__LEVEL=info
    deploy:
      resources:
        limits:
          memory: 32G
          cpus: '8'
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:6333/healthz"]
      interval: 30s
      timeout: 10s
      retries: 3
  
  # Embedding Service
  embedding:
    build:
      context: .
      dockerfile: Dockerfile.embedding
    ports:
      - "8001:8001"
    environment:
      - MODEL_NAME=BAAI/bge-base-en-v1.5
      - CUDA_VISIBLE_DEVICES=0
    deploy:
      resources:
        limits:
          memory: 4G
          cpus: '4'
          reservations:
            devices:
              - driver: nvidia
                count: 1
                capabilities: [gpu]
  
  # Reranker Service
  reranker:
    build:
      context: .
      dockerfile: Dockerfile.reranker
    ports:
      - "8002:8002"
    environment:
      - MODEL_NAME=cross-encoder/ms-marco-MiniLM-L-6-v2
    deploy:
      resources:
        limits:
          memory: 2G
          cpus: '2'
  
  # LLM Service
  llm:
    image: vllm/vllm-openai:latest
    ports:
      - "8000:8000"
    command: >
      --model mistralai/Mistral-7B-Instruct-v0.2
      --tensor-parallel-size 1
      --max-model-len 8192
      --gpu-memory-utilization 0.9
    deploy:
      resources:
        limits:
          memory: 32G
          cpus: '8'
          reservations:
            devices:
              - driver: nvidia
                count: 1
                capabilities: [gpu]
  
  # API Gateway
  api:
    build: ./api
    ports:
      - "8080:8080"
    environment:
      - QDRANT_HOST=qdrant:6334
      - EMBEDDING_URL=http://embedding:8001
      - RERANKER_URL=http://reranker:8002
      - LLM_URL=http://llm:8000/v1
      - REDIS_URL=redis://redis:6379
    depends_on:
      - qdrant
      - embedding
      - reranker
      - redis
    deploy:
      resources:
        limits:
          memory: 1G
          cpus: '2'
  
  # Cache
  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"
    volumes:
      - ./redis_data:/data
    command: redis-server --appendonly yes --maxmemory 4gb --maxmemory-policy allkeys-lru
  
  # Message Queue for async ingestion
  rabbitmq:
    image: rabbitmq:3-management
    ports:
      - "5672:5672"
      - "15672:15672"
    environment:
      - RABBITMQ_DEFAULT_USER=admin
      - RABBITMQ_DEFAULT_PASS=password
  
  # Ingester (async document processing)
  ingester:
    build: ./ingester
    environment:
      - RABBITMQ_HOST=rabbitmq
      - QDRANT_HOST=qdrant:6334
      - EMBEDDING_URL=http://embedding:8001
    depends_on:
      - rabbitmq
      - qdrant
      - embedding
    deploy:
      replicas: 3
      resources:
        limits:
          memory: 2G
          cpus: '2'
  
  # Monitoring
  prometheus:
    image: prom/prometheus:latest
    ports:
      - "9090:9090"
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml
      - prometheus_data:/prometheus
  
  grafana:
    image: grafana/grafana:latest
    ports:
      - "3000:3000"
    environment:
      - GF_SECURITY_ADMIN_PASSWORD=admin
    volumes:
      - ./grafana/dashboards:/etc/grafana/provisioning/dashboards
      - grafana_data:/var/lib/grafana
    depends_on:
      - prometheus

volumes:
  prometheus_data:
  grafana_data:
```

This completes the most comprehensive guide to vector databases available — covering every aspect from fundamental concepts through production deployment, with code examples, REST APIs, interview questions, troubleshooting guides, and best practices.
