# Microservices for AI

> Author: **Tamilselvan** · ✉️ tamilselvan.sde@gmail.com · 🔗 [LinkedIn](https://www.linkedin.com/in/tamilselvan-ai/)
>

## 1. What is it?
Microservices for AI decomposes AI systems into independently deployable, loosely coupled services — each owning a single capability (embedding, retrieval, inference, orchestration) and communicating via well-defined APIs.

## 2. Why do we need it?
Monolithic AI apps become unmanageable: a prompt change requires redeploying everything, GPU-bound inference competes with CPU-bound ingestion, and team scaling is constrained. Microservices enable independent scaling, deployment, and team ownership per capability.

## 3. Real-world Example
**Shopify's Merlin** serves 50+ ML features via microservices: Embedding Service (GPU), Retrieval Service (Qdrant), Ranker (CPU), LLM Gateway (OpenAI + self-hosted), Orchestrator (FastAPI). Each scales independently — Black Friday spikes retrieval but not embedding.

## 4. Architecture Diagram (ASCII)

```
                     +---------------------+
                     |   API Gateway        |
                     +----------+----------+
                                |
              +-----------------+-----------------+
              |                                   |
      +-------v-------+                  +--------v--------+
      |  Orchestrator  |                  |  Auth Service   |
      +-------+-------+                  +--------+--------+
              |                                   |
      +-------v-------+                  +--------v--------+
      |  LLM Gateway   |                  |  Embedding       |
      |  (OpenAI)      |                  |  Service (GPU)   |
      +-------+-------+                  +--------+--------+
              |                                   |
      +-------v-------+                  +--------v--------+
      |  Retrieval     |                  |  Ranking         |
      |  Service       |                  |  Service         |
      |  (Qdrant)      |                  |  (Cross-encoder) |
      +-------+-------+                  +--------+--------+
              |                                   |
      +-------v-------+                  +--------v--------+
      |  Ingestion     |                  |  Monitoring      |
      |  Pipeline      |                  |  (Prom/Grafana)  |
      |  (Kafka -> S3) |                  +-----------------+
      +---------------+
```

## 5. Internal Working
Each microservice owns its data and runs in its own container. Services communicate via gRPC (internal) or REST (external). The Orchestrator coordinates workflows by calling services in parallel. Each service has its own datastore, scaling policy, and deployment cadence. Circuit breakers prevent cascading failures.

## 6. Production Flow
```
API Gateway -> Orchestrator -> LLM Gateway + Embedding + Retrieval (parallel)
-> Orchestrator merges -> Ranking -> Response -> Monitoring event
```

## 7. HLD
| Service | Tech | Data Store | Scaling |
|---------|------|------------|---------|
| Orchestrator | FastAPI, Celery | None | HPA (CPU) |
| LLM Gateway | FastAPI, OpenAI | Redis cache | HPA (req/s) |
| Embedding | ONNX, GPU | None | GPU HPA |
| Retrieval | Qdrant gRPC | Qdrant cluster | Shard count |
| Ranking | Cross-encoder | None | HPA (CPU) |
| Ingestion | Kafka, Spark | S3 + Qdrant | Partition count |

## 8. LLD
```python
class Orchestrator:
    def __init__(self):
        self.llm = LLMGatewayClient()
        self.embed = EmbeddingClient()
        self.retrieve = RetrievalClient()
        self.rank = RankingClient()

    async def process_query(self, query: str):
        embedded, cached = await asyncio.gather(
            self.embed.embed(query), self.cache.lookup(query)
        )
        if cached:
            return cached
        docs = await self.retrieve.search(embedded, top_k=50)
        ranked = await self.rank.rerank(query, docs)
        response = await self.llm.generate(query, ranked[:5])
        await self.cache.store(query, response)
        return response
```

## 9. Python Implementation
```python
from fastapi import FastAPI
from pydantic import BaseModel
import aiohttp, asyncio

app = FastAPI()

class QueryRequest(BaseModel):
    text: str
    user_id: str

class ServiceClient:
    def __init__(self, base_url: str, timeout: int = 5):
        self.base_url = base_url
        self.timeout = timeout
        self.session = aiohttp.ClientSession()

    async def post(self, path: str, data: dict) -> dict:
        async with self.session.post(
            f"{self.base_url}{path}", json=data,
            timeout=aiohttp.ClientTimeout(total=self.timeout)
        ) as resp:
            return await resp.json()

embed = ServiceClient("http://embedding:8001")
retrieve = ServiceClient("http://retrieval:8002")
llm = ServiceClient("http://llm-gateway:8003")

@app.post("/query")
async def query(request: QueryRequest):
    embedding = await embed.post("/embed", {"text": request.text})
    docs = await retrieve.post("/search", {"vector": embedding["vector"], "top_k": 5})
    response = await llm.post("/generate", {
        "messages": [
            {"role": "system", "content": "You are an assistant..."},
            {"role": "user", "content": f"Context: {docs}\nQuery: {request.text}"},
        ]
    })
    return {"answer": response["text"], "sources": docs["results"]}
```

## 10. Folder Structure
```
ai-platform/
  orchestrator/        # FastAPI, Celery tasks
  llm-gateway/         # OpenAI client, circuit breaker
  embedding-service/   # GPU-accelerated embedding
  retrieval-service/   # Qdrant wrapper
  ranking-service/     # Cross-encoder reranker
  ingestion-pipeline/  # Kafka, Spark
  auth-service/        # OAuth2, API keys
  monitoring/          # Prometheus, Grafana
  shared/              # Protobuf definitions
  deploy/              # Helm charts
```

## 11. Configuration
```yaml
services:
  orchestrator:
    replicas: 5
    dependencies:
      embedding: { timeout: 5s, retries: 2 }
      retrieval: { timeout: 3s, retries: 1 }
      llm: { timeout: 30s, retries: 3 }
  embedding:
    gpu: { type: T4, count: 2 }
    autoscaling: { metric: gpu_utilization, target: 70 }
  retrieval:
    qdrant: { shards: 8, replicas: 3 }
```

## 12. Flowchart
```
Request -> Gateway -> Orchestrator
  -> Embedding Service (GPU) ----+
  -> Cache Check (Redis) --------+--- parallel
  -> (Cache miss)
  -> Retrieval Service (Qdrant)
  -> Ranking Service (Cross-encoder)
  -> LLM Gateway (OpenAI)
  -> Cache Store (async)
  -> Response -> Gateway -> Client
```

## 13. Sequence Diagram
```
Client  Gateway  Orchestrator  Embed   Retrieve   Rank   LLM   Cache
 |        |          |           |        |         |      |      |
 |--req-->|          |           |        |         |      |      |
 |        |--query-->|           |        |         |      |      |
 |        |          |---embed-->|        |         |      |      |
 |        |          |---check----------->|         |      |      |
 |        |          |<--miss----|        |         |      |      |
 |        |          |<--vec-----|        |         |      |      |
 |        |          |           |---search------->|      |      |
 |        |          |           |<--docs----------|      |      |
 |        |          |           |        |--rerank->|      |      |
 |        |          |           |        |         |--llm->|      |
 |        |          |------------------store----->|      |      |
 |<--resp-|          |           |        |         |      |      |
```

## 14. Pros
- Independent scaling; Team ownership; Isolated failures; Independent deploy cycles; Technology flexibility per service; Reusable across products.

## 15. Cons
- Network latency between services; Operational complexity; Data consistency challenges; Duplicated infrastructure; Harder end-to-end testing.

## 16. Alternatives
- **Modular monolith**: Simpler ops; **Serverless Lambda**: No infra management; **Event-driven with Kafka**: Async decoupling; **Service mesh**: Better traffic management.

## 17. Performance Considerations
- Each hop adds 1-5ms (gRPC) or 5-15ms (REST); Batch requests before downstream calls; Cache aggressively to skip calls; gRPC is 3-5x faster than REST for internal; LLM dominates latency (500ms-5s).

## 18. Scaling to Millions
- **10K req/s**: 5-10 pods per service, Qdrant 3 shards; **100K**: 50 pods, 16 Qdrant shards, Redis cluster; **1M**: Multi-region, global load balancer; Auto-scale each service on its own metrics.

## 19. Failure Scenarios
- **Orchestrator down**: Gateway returns 503; **LLM timeout**: Retry with fallback model; **Cache storm**: Cache stampede protection; **Embedding GPU OOM**: Queue requests; **Cascading failure**: Bulkhead pattern per dependency.

## 20. Security
- mTLS for service-to-service; API Gateway terminates external TLS; OAuth2 for users; Network policies for pod isolation; Secrets via Vault; Audit logs per service.

## 21. Monitoring
- Per-service: request rate, latency p50/p99, error rate; Dependency health: circuit breaker state; SLA compliance per endpoint; Cost per service; OpenTelemetry distributed tracing.

## 22. Interview Questions
1. *Design AI platform as microservices.* — Embedding, retrieval, ranking, LLM, orchestration.
2. *How do services communicate?* — gRPC for sync, Kafka for async, REST for external.
3. *How to handle LLM key management across services?* — Dedicated LLM Gateway owns keys.
4. *How to ensure consistency?* — Eventual via Kafka, idempotent ops, saga pattern.

## 23. Cheat Sheet
```
Service Boundaries: Embedding | Retrieval | Ranking | LLM | Orchestration | Ingestion
Communication: gRPC (internal) | REST (external) | Kafka (async)
Resilience: Circuit breaker | Retry | Bulkhead | Timeout | Fallback
```

## 24. Common Mistakes
- Too many small services; Shared databases; Sync-only communication; Missing circuit breakers; Inconsistent error handling; No service mesh; Over-abstracting shared libraries.

## 25. Production Best Practices
- Start modular monolith, extract as needed; Define contracts via Protobuf/OpenAPI; Health checks + readiness probes; Structured JSON logging with correlation IDs; GitOps deployment (ArgoCD); Each service owns its database; Document dependency graph; Monthly chaos engineering.
