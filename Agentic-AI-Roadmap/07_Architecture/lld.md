# Low-Level Design (LLD) for AI Systems

> Author: **Tamilselvan** · ✉️ tamilselvan.sde@gmail.com · 🔗 [LinkedIn](https://www.linkedin.com/in/tamilselvan-ai/)
>

## 1. What is it?
Low-Level Design for AI systems provides detailed specifications for each component — class hierarchies, data structures, algorithms, API contracts, error handling, and implementation patterns. It translates the HLD's component boundaries into concrete, implementable designs.

## 2. Why do we need it?
HLD tells you WHAT to build; LLD tells you HOW. Without LLD, developers make inconsistent implementation decisions, leading to integration issues, duplicated logic, and technical debt. LLD ensures every developer implements their component the same way.

## 3. Real-world Example
**Uber's Michelangelo ML Platform** — LLD specifies exact feature computation logic: `lookup_feature(feature_name, entity_key, timestamp)` with type coercion, null handling, and staleness checks. Every team implements features using the same `FeatureStoreClient` class with identical retry, timeout, and caching behavior.

## 4. Architecture Diagram (ASCII)

```
+------------------------------------------------------------------+
|                     RAGWorkflow Class                              |
|  +------------------------------------------------------------+  |
|  | Components:                                                  |  |
|  |  +------------+  +------------+  +------------+             |  |
|  |  | QueryParser |  | Context    |  | Prompt      |           |  |
|  |  | (Intent,   |  | Retriever  |  | Assembler   |           |  |
|  |  | Entities,  |  | (Hybrid    |  | (Template,  |           |  |
|  |  | Filters)   |  |  Search)   |  | History)    |           |  |
|  |  +------------+  +------------+  +------------+             |  |
|  |  +------------+  +------------+  +------------+             |  |
|  |  | LLM Client |  | Response   |  | Validator   |           |  |
|  |  | (Retry,    |  | Formatter  |  | (JSON,      |           |  |
|  |  | Streaming) |  | (Markdown) |  |  Safety)    |           |  |
|  |  +------------+  +------------+  +------------+             |  |
|  +------------------------------------------------------------+  |
+------------------------------------------------------------------+
```

## 5. Internal Working
LLD specifies every component's internals: the Retriever uses a `HybridSearchEngine` combining vector similarity (FAISS) and keyword match (BM25) with configurable `alpha` weight. The LLM Client wraps the OpenAI API with exponential backoff, circuit breaker (3 failures → open for 30s), and response streaming via `asyncio`. The Prompt Assembler constructs a message list from templates with token counting to stay under context limits.

## 6. Production Flow

```python
async def process_query(user_query: str, user_id: str) -> Response:
    # Step 1: Parse
    parsed = QueryParser.parse(user_query)
    # Step 2: Retrieve context
    context = await Retriever.retrieve(parsed.query, user_id, k=5)
    # Step 3: Build prompt
    messages = PromptAssembler.build(context, parsed.query)
    # Step 4: Call LLM
    llm_response = await LLMClient.generate(messages)
    # Step 5: Validate and format
    validated = ResponseValidator.validate(llm_response)
    return ResponseFormatter.format(validated)
```

## 7. HLD
(Detailed in `hld.md`) — LLD focuses on implementation details.

## 8. LLD

### QueryParser Class

```python
@dataclass
class ParsedQuery:
    query: str
    intent: str          # "qa", "summarize", "recommend"
    entities: list[str]  # ["California", "contract law"]
    filters: dict        # {"date_range": ["2024-01-01", "2024-12-31"]}
    language: str        # "en"

class QueryParser:
    @staticmethod
    def parse(raw: str) -> ParsedQuery:
        # 1. Language detection
        # 2. Intent classification (small classifier or LLM call)
        # 3. NER for entity extraction
        # 4. Filter extraction via regex patterns
        # 5. Return structured ParsedQuery
```

### Retriever Class

```python
class Retriever:
    def __init__(self, config: RetrieverConfig):
        self.vector_db = VectorDBClient(config.vector_db)
        self.embedder = EmbeddingClient(config.embedder)
        self.bm25_index = BM25Index(config.bm25_path)
        self.alpha = config.alpha  # 0.0 = pure vector, 1.0 = pure bm25

    async def retrieve(
        self, query: str, top_k: int = 5, filters: dict = {}
    ) -> list[Chunk]:
        query_vec = await self.embedder.embed(query)

        # Parallel search
        vector_results = await self.vector_db.search(query_vec, top_k * 2, filters)
        bm25_results = self.bm25_search(query, top_k * 2)

        # Reciprocal Rank Fusion
        fused = self.rrf_merge(vector_results, bm25_results, k=60)

        return fused[:top_k]

    def bm25_search(self, query: str, k: int) -> list[Chunk]:
        tokenized = self.tokenizer.tokenize(query.lower())
        scores = self.bm25_index.get_scores(tokenized)
        top_indices = np.argsort(scores)[-k:][::-1]
        return [self.chunks[i] for i in top_indices]

    def rrf_merge(self, lists: list[list], k: int = 60) -> list:
        scores = {}
        for rank, item in enumerate(
            itertools.chain.from_iterable(lists)
        ):
            scores[item.id] = scores.get(item.id, 0) + 1 / (k + rank)
        return sorted(scores, key=scores.get, reverse=True)
```

### LLM Client Class

```python
class LLMClient:
    def __init__(self, config: LLMConfig):
        self.client = AsyncOpenAI(api_key=config.api_key)
        self.model = config.model
        self.circuit = CircuitBreaker(
            failure_threshold=3,
            recovery_timeout=30,
        )
        self.semaphore = asyncio.Semaphore(config.max_concurrent)

    async def generate(
        self, messages: list[dict], stream: bool = False, **kwargs
    ) -> LLMResponse:
        if self.circuit.is_open():
            raise CircuitBreakerOpen("LLM circuit breaker is open")

        async with self.semaphore:
            try:
                response = await self.client.chat.completions.create(
                    model=self.model,
                    messages=messages,
                    stream=stream,
                    temperature=kwargs.get("temperature", 0.3),
                    max_tokens=kwargs.get("max_tokens", 4096),
                )
                self.circuit.record_success()
                if stream:
                    return self._handle_stream(response)
                return LLMResponse(
                    text=response.choices[0].message.content,
                    usage=response.usage,
                    model=response.model,
                )
            except Exception as e:
                self.circuit.record_failure()
                raise LLMException(f"LLM call failed: {e}") from e
```

### Data Structures

```python
@dataclass
class Chunk:
    id: str
    text: str
    embedding: list[float]
    metadata: dict
    token_count: int

@dataclass
class LLMResponse:
    text: str
    usage: UsageInfo
    model: str

@dataclass
class UsageInfo:
    prompt_tokens: int
    completion_tokens: int
    total_tokens: int

@dataclass
class CacheEntry:
    response: LLMResponse
    created_at: datetime
    ttl_seconds: int
    hit_count: int = 0
```

## 9. Python Implementation

```python
# Complete RAG workflow LLD
import asyncio
from dataclasses import dataclass, field
from datetime import datetime
from typing import Optional
import numpy as np

@dataclass
class RAGConfig:
    vector_db_url: str = "http://localhost:6333"
    embedding_model: str = "text-embedding-ada-002"
    llm_model: str = "gpt-4o"
    top_k: int = 5
    cache_ttl: int = 3600
    max_retries: int = 3

class RAGEngine:
    def __init__(self, config: RAGConfig):
        self.config = config
        self.retriever = Retriever(config)
        self.llm = LLMClient(config)
        self.cache = SemanticCache(config)
        self.parser = QueryParser()

    async def query(self, user_input: str) -> dict:
        parsed = self.parser.parse(user_input)
        cached = await self.cache.lookup(parsed.query)
        if cached:
            return cached

        context = await self.retriever.retrieve(
            parsed.query, self.config.top_k, parsed.filters
        )
        prompt = self._build_prompt(context, parsed.query)
        response = await self.llm.generate(prompt)
        formatted = self._format_response(response, context)

        await self.cache.store(parsed.query, formatted)
        return formatted

    def _build_prompt(
        self, context: list[Chunk], query: str
    ) -> list[dict]:
        system_prompt = "You are an AI assistant..."
        context_text = "\n\n".join(c.text for c in context)
        return [
            {"role": "system", "content": system_prompt},
            {"role": "user", "content": context_text + "\n\n" + query},
        ]

    def _format_response(self, response, context):
        return {
            "answer": response.text,
            "sources": [c.metadata for c in context],
            "usage": response.usage.__dict__,
            "timestamp": datetime.now().isoformat(),
        }
```

## 10. Folder Structure

```
src/
├── core/
│   ├── __init__.py
│   ├── query_parser.py
│   ├── retriever.py
│   ├── llm_client.py
│   ├── cache.py
│   ├── prompt_assembler.py
│   ├── response_validator.py
│   └── response_formatter.py
├── models/
│   ├── chunk.py
│   ├── query.py
│   └── response.py
├── utils/
│   ├── circuit_breaker.py
│   ├── token_counter.py
│   └── rrf.py
├── config/
│   └── settings.py
└── tests/
    ├── test_retriever.py
    ├── test_llm_client.py
    └── test_workflow.py
```

## 11. Configuration

```yaml
# config/rag_config.yaml
retriever:
  alpha: 0.3
  top_k: 5
  vector_db:
    type: qdrant
    url: http://localhost:6333
    collection: docs
    grpc: true
  embedder:
    model: text-embedding-ada-002
    batch_size: 16
    max_retries: 3

llm:
  model: gpt-4o
  temperature: 0.3
  max_tokens: 4096
  max_concurrent: 10
  timeout: 30
  circuit_breaker:
    failure_threshold: 3
    recovery_timeout: 30
  fallback:
    model: gpt-4o-mini
    temperature: 0.5

cache:
  type: redis
  ttl: 3600
  max_size: 10000
  similarity_threshold: 0.85
```

## 12. Flowchart

```
query(input)
  |
  v
Parse query (intent, entities, filters)
  |
  v
Check semantic cache
  |         \
  |         (miss)
  v          v
Retrieve context (vector BM25 hybrid)
  |
  v
Build prompt (system + context + query)
  |
  v
LLM generate (with retry + circuit breaker)
  |
  v
Validate output (JSON, length, safety)
  |
  v
Format response + store in cache
  |
  v
Return
```

## 13. Sequence Diagram

```
User       RAGEngine    QueryParser   Retriever   LLMClient   Cache
 |            |              |            |            |        |
 |--query---->|              |            |            |        |
 |            |--parse------>|            |            |        |
 |            |<--parsed-----|            |            |        |
 |            |                         |            |        |
 |            |--cache lookup----------------------->|        |
 |            |<--miss-------------------------------|        |
 |            |              |            |            |        |
 |            |--retrieve--------------->|            |        |
 |            |<--context----------------|            |        |
 |            |              |            |            |        |
 |            |--generate--------------------------->|        |
 |            |<--response----------------------------|        |
 |            |                         |            |        |
 |            |--cache store------------------------->|        |
 |<--response |              |            |            |        |
```

## 14. Pros
- Eliminates ambiguity — every developer knows exactly how to implement; Enables parallel development on independent components; Easy to unit test (clear interfaces); Identifies missing error handling; Foundation for accurate estimation.

## 15. Cons
- Time-consuming to create; Rigid if requirements change; Can over-specify non-critical components; Risk of cargo-cult implementation without understanding; May not capture all edge cases.

## 16. Alternatives
- **TDD**: Let tests define the design; **ADR**: Document only key decisions; **Spec-first (OpenAPI)**: Define API contracts only; **RFC process**: Collaborative design documents.

## 17. Performance Considerations
- Every database call adds 5-20ms; Vector search is I/O bound (RAM vs SSD); LLM generation dominates (500ms-5s); Serialize parallelizable operations (retrieve + bm25 concurrently); Token counting prevents context window overflow; Cache hit ratio directly impacts p50 latency.

## 18. Scaling to Millions
- Async everywhere (no blocking calls); Connection pooling for all external services; Batch processing where possible (embedding, indexing); Stateless services for horizontal scaling; Queue-based load leveling (SQS/RabbitMQ); Read replicas for cache-avoidant workloads; Pre-compute embeddings during ingestion.

## 19. Failure Scenarios
- **LLM timeout**: Retry with exponential backoff (1s, 2s, 4s), then fallback model; **Cache failure**: Degrade to direct fetch (no semantic caching); **Vector DB down**: BM25-only fallback (lower quality but functional); **Circuit breaker open**: Queue requests and return stale cache; **Token overflow**: Implement truncation with sliding window.

## 20. Security
- API keys in environment variables (not code); Input sanitization (PII detection); Output validation (toxicity check); Rate limiting per user; Prompt injection detection; Secrets rotation schedule; Encrypted PII fields; Audit logging with correlation IDs.

## 21. Monitoring
- **LLM metrics**: Latency, token usage, error rate, circuit breaker state; **Retrieval metrics**: Recall@K, latency, filter selectivity; **Cache metrics**: Hit rate, size, eviction count; **Workflow metrics**: End-to-end latency per step, error rate; **Business metrics**: Query count per user, satisfaction score.

## 22. Interview Questions
1. *Implement a circuit breaker for LLM API calls.* — Tracks failures in a sliding window, opens circuit on threshold, half-opens after timeout.
2. *Design a hybrid retriever combining vector and keyword search.* — RRF fusion: `score = sum(1/(k+rank))` across result lists.
3. *How do you handle rate limiting for multiple LLM API keys?* — Round-robin with per-key usage tracking.
4. *Design a semantic cache for RAG.* — Embed query, cosine similarity to cached entries, return if above threshold.

## 23. Cheat Sheet

```
LLD Checklist:
☐ Class diagrams for all components
☐ API contracts (request/response)
☐ Error handling for all failure modes
☐ Data structures and types
☐ Configuration schema
☐ Sequence diagrams for key flows
☐ Retry and timeout strategy
☐ Logging and metrics
```

## 24. Common Mistakes
- Making classes too large (SRP violation); No error handling for transient failures; Hardcoded timeouts and retries; Sync blocking calls in async context; Over-abstracting (interface for every tiny class); Forgetting to close connections; No request ID tracing; Ignoring token counting for context management.

## 25. Production Best Practices
- Keep LLD in Git alongside code (not Google Docs); Review LLDs in code review; Update LLD when implementation diverges; Use pydantic for data validation at boundaries; Pin dependency versions; Measure and document expected latencies; Add async everywhere (EAFP, not LBYL); Use structured logging (JSON); Implement healthcheck endpoints; Performance test before production deployment.
