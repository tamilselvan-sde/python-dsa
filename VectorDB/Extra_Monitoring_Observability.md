# Monitoring & Observability Deep Dive

> Author: **Tamilselvan** · ✉️ tamilselvan.sde@gmail.com · 🔗 [LinkedIn](https://www.linkedin.com/in/tamilselvan-ai/)
>


### Key Metrics to Monitor

```python
import prometheus_client as prom

# Define metrics
vector_search_latency = prom.Histogram(
    'vector_search_latency_seconds',
    'Vector search latency',
    buckets=[0.001, 0.005, 0.01, 0.025, 0.05, 0.1, 0.25, 0.5, 1.0],
    labelnames=['index_type', 'shard']
)

recall_gauge = prom.Gauge(
    'vector_search_recall',
    'Recall@10 compared to ground truth',
    labelnames=['collection']
)

memory_usage = prom.Gauge(
    'vector_index_memory_bytes',
    'Vector index memory usage',
    labelnames=['collection', 'component']
)

qps_counter = prom.Counter(
    'vector_search_queries_total',
    'Total vector search queries',
    labelnames=['status']  # 'success', 'error', 'timeout'
)

# Usage
def search_with_monitoring(query, db):
    start = time.time()
    try:
        results = db.search(query)
        status = 'success'
    except Exception as e:
        status = 'error'
        raise
    finally:
        latency = time.time() - start
        vector_search_latency.labels(
            index_type=db.index_type, shard=db.shard_id
        ).observe(latency)
        qps_counter.labels(status=status).inc()
    
    return results
```

### Monitoring Dashboard (Grafana)

```yaml
# Grafana dashboard annotations
panels:
  - title: "Vector Search p99 Latency"
    type: "heatmap"
    datasource: "Prometheus"
    targets:
      - expr: "histogram_quantile(0.99, vector_search_latency_seconds)"
        
  - title: "Recall@10"
    type: "stat"
    targets:
      - expr: "vector_search_recall"
    thresholds:
      - value: 95
        color: "green"
      - value: 90
        color: "yellow"
      - value: 0
        color: "red"
        
  - title: "QPS by Index Type"
    type: "barchart"
    targets:
      - expr: "rate(vector_search_queries_total[5m])"
```

---

## Benchmarking Methodology

### How to Benchmark ANN Algorithms

```python
import time
import numpy as np
from sklearn.model_selection import train_test_split
from typing import List, Tuple

class ANNBenchmark:
    def __init__(self, dim=768, n_queries=1000):
        self.dim = dim
        self.n_queries = n_queries
        self.results = {}
        
    def generate_ground_truth(self, queries, vectors, k=10):
        """Compute exact KNN for queries."""
        from sklearn.metrics.pairwise import cosine_similarity
        ground_truth = []
        for q in queries:
            sims = cosine_similarity([q], vectors)[0]
            top_k = np.argsort(-sims)[:k]
            ground_truth.append(set(top_k))
        return ground_truth
    
    def measure(self, 
                name: str,
                build_fn,
                search_fn,
                train_vectors: np.ndarray,
                test_queries: np.ndarray,
                k: int = 10):
        
        # Build index
        t0 = time.time()
        index = build_fn(train_vectors)
        build_time = time.time() - t0
        
        # Get ground truth
        ground_truth = self.generate_ground_truth(
            test_queries, train_vectors, k
        )
        
        # Measure search
        latencies = []
        recalls = []
        
        for query, gt in zip(test_queries, ground_truth):
            t0 = time.time()
            results = search_fn(query, k)
            latency = time.time() - t0
            
            found = set([idx for idx, _ in results])
            recall = len(found & gt) / len(gt)
            
            latencies.append(latency)
            recalls.append(recall)
        
        # Store results
        self.results[name] = {
            "build_time": build_time,
            "latency_p50": np.percentile(latencies, 50),
            "latency_p95": np.percentile(latencies, 95),
            "latency_p99": np.percentile(latencies, 99),
            "mean_recall": np.mean(recalls),
            "min_recall": np.min(recalls),
            "qps": 1.0 / np.mean(latencies),
        }
        
        return self.results[name]
    
    def print_report(self):
        """Print comparison table."""
        print(f"{'Algorithm':<20} {'Build(s)':<10} {'p50(ms)':<10} "
              f"{'p99(ms)':<10} {'Recall':<10} {'QPS':<10}")
        print("-" * 70)
        for name, metrics in self.results.items():
            print(f"{name:<20} {metrics['build_time']:<10.1f} "
                  f"{metrics['latency_p50']*1000:<10.1f} "
                  f"{metrics['latency_p99']*1000:<10.1f} "
                  f"{metrics['mean_recall']:<10.3f} "
                  f"{metrics['qps']:<10.0f}")

# Usage
benchmark = ANNBenchmark(dim=768)
vectors = np.random.random((100000, 768)).astype('float32')
queries = np.random.random((1000, 768)).astype('float32')

benchmark.measure(
    "HNSW-M16",
    lambda v: faiss.IndexHNSWFlat(768, 16),
    lambda q, k: index.search(q.reshape(1, -1), k),
    vectors, queries
)
benchmark.print_report()
```

---

