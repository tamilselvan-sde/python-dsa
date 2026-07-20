# Semantic Cache

## 1. What is it?
**ELI5:** Imagine asking the same question in different ways: "What's the weather?" vs "How's the weather outside?" vs "Tell me the forecast." A regular cache needs the exact same words to match. Semantic cache understands that these all mean the same thing — so if you asked "What's the weather?" earlier, and now ask "How's the weather outside?", the cache recognizes they're semantically identical and returns the same answer without doing any real work.

**Technical Definition:** Semantic caching stores query-response pairs from a RAG system and retrieves them not by exact string match, but by embedding similarity. When a new query arrives, it's embedded and compared against cached query embeddings. If the cosine similarity exceeds a threshold (e.g., 0.92), the cached response is returned. This avoids redundant retrieval, embedding, reranking, and LLM generation for semantically equivalent queries.

## 2. Why do we need it?
- **Latency reduction:** Cache hit = 5-20ms vs 2-5s for a full RAG pipeline. 100-1000x faster.
- **Cost savings:** Each cache miss costs: embedding ($0.0001) + vector DB ($0.0001) + reranker ($0.001) + LLM generation ($0.01-0.10). A cache hit costs only a similarity lookup (~$0.00001).
- **Real-world query distribution:** 60-80% of queries are near-duplicates of previously seen queries in production. "How do I reset my password?", "How to change password?", "Forgot password reset" — same intent.
- **Rate limit protection:** Cache absorbs traffic spikes that would otherwise hit LLM API rate limits.
- **Consistent responses:** Users get the same answer for the same question — no LLM variance.

## 3. Real-world Example
- **Github Copilot Chat:** Caches responses for common coding questions. "What's a Python decorator?" answered from cache for most users.
- **Zendesk Answer Bot:** Semantic cache stores answers for FAQ-type support queries. ~60% hit rate, saving millions in LLM API costs.
- **Notion AI Q&A:** Semantic cache for workspace queries — "Who's working on project X?" cached until workspace changes.
- **Internal enterprise chatbots:** Kafka topics documentation, employee handbook queries — these change infrequently, so cache hit rates are >80%.

## 4. Architecture Diagram (ASCII)
```
+---------+     +----------+
| Query   | --> | Embedder |
+---------+     +----------+
                     |
                     v
            +----------------+
            | Cache Lookup   |
            | (cosine sim    |
            |  to cached     |
            |  queries)      |
            +----------------+
               |         |
            hit         miss
               |         |
               v         v
        +----------+  +----------------+
        | Return   |  | Full RAG       |
        | Cached   |  | Pipeline       |
        | Response |  | (retrieve +    |
        +----------+  |  rerank + gen) |
                      +----------------+
                           |
                           v
                      +----------------+
                      | Store in Cache |
                      | (query_emb,    |
                      |  response)     |
                      +----------------+
```

## 5. Internal Working
1. **Cache structure:** Key-value store where key = query embedding (vector), value = (response, metadata, ttl).
2. **Lookup:** Incoming query is embedded. All cached query embeddings are compared via cosine similarity. If max similarity > threshold, return cached response.
3. **Threshold selection:** Typical thresholds: 0.85 (aggressive cache, more hits, risk of false positives) to 0.95 (conservative, fewer hits but safer). Tune on your data.
4. **Eviction:** LRU (Least Recently Used) or TTL-based (time-to-live). TTL depends on data freshness requirements — support KB: 24h. Stock prices: 1min. Legal docs: 7d.
5. **Invalidation:** Cache entries must be invalidated when the underlying knowledge changes (documents re-indexed). Use collection-level version stamps.

## 6. Production Flow
```
User Query: "How do I reset my password?"
    |
    v
1. Embed query → q_vec (5ms)
    |
    v
2. Cosine similarity against cache (1-10ms)
   (faiss index over cached query embeddings)
    |
    +---> Similarity 0.97 > threshold 0.92 → CACHE HIT
    |       ↓
    |     Return cached response (2ms)
    |
    +---> Similarity 0.71 < threshold 0.92 → CACHE MISS
            ↓
          Full RAG pipeline (2-5s)
            ↓
          Store (query_emb, response) in cache
            ↓
          Return response
```

## 7. HLD
```
                    +------------------+
                    |    Redis Cache   |
                    | (embedding →     |
                    |  response KV)    |
                    +--------+---------+
                             |
                    +--------v---------+
                    | Faiss Index      |
                    | (cached query    |
                    |  embeddings for  |
                    |  ANN search)     |
                    +--------+---------+
                             |
                    +--------v---------+
                    | Cache Service    |
                    | (lookup, store,  |
                    |  evict, version) |
                    +--------+---------+
                             |
               +-------------+-------------+
               |             |             |
        +------v----+ +-----v-----+ +-----v-----+
        | Embedder  | | RAG       | | TTL       |
        | Service   | | Pipeline  | | Scheduler |
        +-----------+ +-----------+ +-----------+
```

## 8. LLD
```python
import asyncio
import time
from dataclasses import dataclass
from typing import List, Optional, Tuple
import numpy as np
import redis.asynced as redis

@dataclass
class CacheEntry:
    query_embedding: List[float]
    response: str
    citations: List[dict]
    timestamp: float
    ttl: int  # seconds

class SemanticCache:
    def __init__(
        self,
        redis_url: str = "redis://localhost:6379",
        threshold: float = 0.92,
        index_path: str = "/data/cache_index",
    ):
        self.redis = None
        self.threshold = threshold
        self.entries: List[CacheEntry] = []
        self.embedding_dim = 1536  # text-embedding-3-small

    async def connect(self):
        self.redis = await redis.from_url(self.redis_url)

    async def lookup(self, query_emb: List[float]) -> Optional[dict]:
        if not self.entries:
            return None

        # Brute force (small cache) or ANN (large cache)
        query_emb = np.array(query_emb)
        best_score = -1.0
        best_idx = -1

        for i, entry in enumerate(self.entries):
            cached_emb = np.array(entry.query_embedding)
            score = np.dot(query_emb, cached_emb) / (
                np.linalg.norm(query_emb) * np.linalg.norm(cached_emb) + 1e-8
            )

            # Check TTL
            if time.time() - entry.timestamp > entry.ttl:
                continue

            if score > best_score:
                best_score = score
                best_idx = i

        if best_score >= self.threshold and best_idx >= 0:
            entry = self.entries[best_idx]
            return {
                "response": entry.response,
                "citations": entry.citations,
                "similarity": float(best_score),
            }

        return None

    async def store(
        self, query_emb: List[float], response: str, citations: List[dict], ttl: int = 3600
    ):
        entry = CacheEntry(
            query_embedding=query_emb,
            response=response,
            citations=citations,
            timestamp=time.time(),
            ttl=ttl,
        )
        self.entries.append(entry)

    async def invalidate_collection(self, collection_id: str):
        # Remove entries tied to a specific collection
        pass
```

## 9. Python Implementation
```python
import hashlib
import json
import time
from typing import Optional

import numpy as np
from fastapi import FastAPI
from pydantic import BaseModel, Field
import redis.asynced as redis

app = FastAPI(title="Semantic Cache Service")

class CacheQuery(BaseModel):
    query: str = Field(..., min_length=1)
    threshold: float = Field(default=0.92, ge=0.0, le=1.0)

class CacheResponse(BaseModel):
    cached: bool
    response: Optional[str] = None
    similarity: Optional[float] = None
    latency_ms: float

class StoreRequest(BaseModel):
    query: str
    response: str
    ttl: int = 3600

class SemanticCacheService:
    def __init__(self, redis_url: str = "redis://localhost:6379", dim: int = 1536):
        self.redis = None
        self.dim = dim
        self.threshold = 0.92

    async def connect(self):
        self.redis = await redis.from_url("redis://localhost:6379")

    def _embed(self, text: str) -> list:
        # In production: call embedding API
        rng = np.random.default_rng(hashlib.sha256(text.encode()).digest())
        vec = rng.standard_normal(self.dim)
        vec = vec / np.linalg.norm(vec)
        return vec.tolist()

    def _cosine_sim(self, a: list, b: list) -> float:
        a_np = np.array(a)
        b_np = np.array(b)
        return float(np.dot(a_np, b_np) / (np.linalg.norm(a_np) * np.linalg.norm(b_np) + 1e-8))

    async def lookup(self, query: str, threshold: float = 0.92) -> Optional[dict]:
        q_vec = self._embed(query)
        keys = await self.redis.keys("cache:*")
        best_score = -1.0
        best_key = None

        for key in keys:
            data = json.loads(await self.redis.get(key))
            score = self._cosine_sim(q_vec, data["embedding"])
            if score > best_score and score >= threshold:
                best_score = score
                best_key = key

        if best_key:
            data = json.loads(await self.redis.get(best_key))
            return {
                "response": data["response"],
                "similarity": best_score,
            }
        return None

    async def store(self, query: str, response: str, ttl: int = 3600):
        q_vec = self._embed(query)
        key = f"cache:{hashlib.md5(query.encode()).hexdigest()}"
        await self.redis.setex(
            key,
            ttl,
            json.dumps({
                "query": query,
                "embedding": q_vec,
                "response": response,
                "timestamp": time.time(),
            }),
        )

cache_service = SemanticCacheService()

@app.on_event("startup")
async def startup():
    await cache_service.connect()

@app.post("/cache/lookup", response_model=CacheResponse)
async def lookup(req: CacheQuery):
    import time as t
    start = t.time()

    result = await cache_service.lookup(req.query, req.threshold)

    return CacheResponse(
        cached=result is not None,
        response=result["response"] if result else None,
        similarity=result["similarity"] if result else None,
        latency_ms=(t.time() - start) * 1000,
    )

@app.post("/cache/store")
async def store(req: StoreRequest):
    await cache_service.store(req.query, req.response, req.ttl)
    return {"status": "stored", "ttl": req.ttl}

@app.delete("/cache/invalidate")
async def invalidate():
    keys = await cache_service.redis.keys("cache:*")
    if keys:
        await cache_service.redis.delete(*keys)
    return {"status": "invalidated", "count": len(keys)}
```

## 10. Folder Structure
```
semantic-cache/
├── src/
│   ├── __init__.py
│   ├── main.py
│   ├── cache.py
│   ├── embedder.py
│   ├── index.py
│   ├── config.py
│   └── models.py
├── tests/
├── deploy/
└── pyproject.toml
```

## 11. Configuration
```yaml
semantic_cache:
  backend: redis
  redis_url: redis://redis:6379
  embedding:
    model: text-embedding-3-small
    dimension: 1536
  similarity:
    threshold: 0.92
    metric: cosine
  ttl:
    default: 3600
    min: 60
    max: 86400
  eviction: lru
  max_entries: 100000
```

## 12. Flowchart
```
Start
  |
  v
[Receive Query]
  |
  v
[Embed Query → q_vec]
  |
  v
[ANN Search on Cache Index]
  |
  +---> Max sim >= threshold?
  |       |
  |    YES --> [Return Cached Response]
  |       |
  |    NO  --> [Run Full RAG Pipeline]
  |            |
  |            v
  |       [Store (q_vec, response) in Cache]
  |            |
  |            v
  +---> [Return Response]
```

## 13. Sequence Diagram
```
User      CacheSvc     Embedder    Redis       RAG Pipeline
  |          |            |          |              |
  |--Query-->|            |          |              |
  |          |--embed---->|          |              |
  |          |<--q_vec----|          |              |
  |          |            |          |              |
  |          |--ANN search---------->|              |
  |          |<--candidates----------|              |
  |          |            |          |              |
  |          |--score---->|          |              |
  |          |<--0.94-----|          |              |
  |          |            |          |              |
  |          v (cache hit)           |              |
  |<--Response|            |          |              |
```

## 14. Pros
- 100-1000x faster than full RAG on cache hits
- Reduces LLM API costs by 40-80%
- Absorbs traffic spikes (LLM rate limit buffer)
- Simple to implement on top of Redis
- Users get consistent answers for the same question

## 15. Cons
- Stale responses for fast-changing knowledge bases
- Memory cost: storing query embeddings for millions of queries
- False positive risk: overly aggressive threshold returns wrong response
- Embedding lookup itself adds 5-10ms latency
- Cache invalidation is complex when underlying data changes

## 16. Alternatives
| Method | Description |
|--------|-------------|
| Exact cache (Redis string) | Key = exact query string. No semantic matching |
| TTL-based cache | Evict after fixed time. Simple but wastes hits |
| LRU cache | Evict least recently used. Good for hot queries |
| Contextual cache | Cache entire (query + retrieved chunk hashes) for more precision |
| Probabilistic cache | Early expiration to prevent cache stampede |

## 17. Performance Considerations
- **ANN index on embeddings:** Build a Faiss IVF index over cached query embeddings for <1ms search over 100K entries.
- **Threshold selection:** 0.85 = high hit rate (~70%), some false positives. 0.95 = safe, lower hit rate (~30%). Tune per dataset.
- **Memory:** 1M cached queries × 1536-dim float32 embedding = ~6GB. Compress with PQ to ~1.5GB.
- **TTL strategy:** Long TTL for static knowledge (product docs), short TTL for dynamic data (news, stock prices).
- **Cache warmup:** Pre-populate cache with top-10K historical queries on service startup.

## 18. Scaling to Millions
- **Shard cache across Redis clusters** by query embedding hash.
- **Use Faiss for ANN search** over cached embeddings — scales to 10M vectors on a single machine.
- **Segmented cache:** Hot tier (in-memory Redis, <1M entries), warm tier (SSD-based, RocksDB).
- **Write-behind caching:** Store responses asynchronously to avoid blocking the query path.
- **Global + local cache:** Global Redis for all users, local LRU on each app server for ultra-fast hits.

## 19. Failure Scenarios
| Failure | Impact | Mitigation |
|---------|--------|------------|
| Cache poisoning | Wrong response served | Validate cached response signature |
| Cache stampede | All requests miss → overload RAG pipeline | Probabilistic early expiration, request coalescing |
| Redis down | All requests miss → no cache | Circuit breaker, fall through to RAG |
| Stale data | Outdated responses | Publish version events on data change |

## 20. Security
- Sign cached responses to detect tampering (HMAC).
- Don't cache PII or sensitive user-specific responses.
- Tenant isolation: separate cache namespaces per tenant.
- Rate-limit cache store operations to prevent cache poisoning at scale.

## 21. Monitoring
- **Cache hit ratio:** Overall, per-user, per-query-type
- **Lookup latency:** p50, p95, p99
- **False positive rate:** % of served cached responses that users thumbs-down
- **Cache size:** Entry count, memory usage
- **Eviction rate:** LRU evictions per minute
- **Cold start time:** Time to reach target hit ratio after service start

## 22. Interview Questions
| Level | Question | Key Points |
|-------|----------|------------|
| L3 | What is a semantic cache and how does it differ from a regular cache? | Regular: exact match. Semantic: embedding similarity |
| L4 | How would you choose the similarity threshold? | A/B test on historical queries, precision-recall tradeoff |
| L4 | How do you handle cache invalidation when data changes? | Collection-level version stamps, TTL-based, event-driven |
| L5 | Design a semantic cache for a multi-tenant RAG system | Tenant-prefixed keys, per-tenant thresholds, quota limits |
| L5 | How would you prevent cache poisoning? | Input validation, response signing, anomaly detection |

## 23. Cheat Sheet
```python
# Semantic cache in 5 lines
q_vec = embed(query)
best_score = max(cosine_sim(q_vec, e.emb) for e in cache_entries)
if best_score > threshold:
    return cache_entries[best_idx].response
# else: run RAG, store result
```

## 24. Common Mistakes
- **Threshold too low** — serving wrong answers for semantically different queries (precision failure).
- **Threshold too high** — cache never hits, paying full RAG cost anyway (recall failure).
- **No TTL on cache entries** — serving "CEO is John Smith" years after he left.
- **Embedding model mismatch** — using a different embedder for cache lookup than for retrieval.
- **Ignoring query length** — very short queries ("hello") have high cosine similarity to everything.
- **Not invalidating on data changes** — users get stale answers after knowledge base update.

## 25. Production Best Practices
1. **Set a similarity threshold between 0.88-0.95** — tune on a validation set of query pairs.
2. **Always set TTL on cached entries** — default 1 hour for dynamic KB, 24 hours for static.
3. **Use the exact same embedding model** for cache lookup and for retrieval embedding.
4. **Implement cache stampede protection** — probabilistic early expiration (XFetch algorithm).
5. **Monitor false positive rate** — if users thumbs-down cached responses, raise the threshold.
6. **Version your cache** — bump cache version when you re-index the knowledge base, invalidating all entries.
7. **Warm the cache on deploy** — replay top-10K historical queries to pre-populate.
8. **Isolate by tenant** — prefix cache keys with tenant_id for multi-tenant systems.
9. **Don't cache user-specific responses** — cache only queries that produce the same answer for all users.
10. **Log cache hits and misses** — essential for debugging and hit-ratio optimization.
