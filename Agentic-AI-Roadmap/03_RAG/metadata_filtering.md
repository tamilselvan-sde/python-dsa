# Metadata Filtering

> Author: **Tamilselvan** · ✉️ tamilselvan.sde@gmail.com · 🔗 [LinkedIn](https://www.linkedin.com/in/tamilselvan-ai/)
>

## 1. What is it?
**ELI5:** Imagine you're searching for a book in a massive library. Instead of searching every shelf, you first filter by floor "2nd floor → Non-fiction section → Computer Science aisle." That's metadata filtering — you narrow down where to search before you even start looking. In a RAG system, metadata filtering lets you attach labels to documents (date, author, category, tags) and then use those labels to pre-filter which documents get searched.

**Technical Definition:** Metadata filtering is a retrieval optimization technique where structured attributes (metadata) associated with document chunks are used to pre-filter or post-filter search results. Common metadata fields include document source, date range, author, category, version, access tier, and tenant ID. Filters are applied either before vector search (pre-filtering, reducing the search space) or after vector search (post-filtering, removing unwanted results from candidates).

## 2. Why do we need it?
- **Precision improvement:** If the user asks "Q3 2024 financials," filtering by `date >= 2024-07-01 AND date <= 2024-09-30` ensures only Q3 docs are considered, regardless of embedding similarity noise.
- **Search space reduction:** On a 10M-document vector DB, filtering by tenant_id reduces the search space to 100K documents — 100x faster, more accurate.
- **Multi-tenancy:** Each customer's data is isolated by `tenant_id` filter. No cross-tenant leakage, no per-tenant vector DB instances.
- **Access control:** Filter by `access_level >= user.role` to ensure users only see documents they're authorized to view.
- **Time-based queries:** "Latest news about AI" — filter by `date >= last_week` before semantic search.
- **Regulatory compliance:** GDPR right-to-be-forgotten — filter out documents the user has deleted.

## 3. Real-world Example
- **Elasticsearch metadata search:** Every document has fields like `author`, `timestamp`, `filetype`. Searches use boolean filters (`author:john AND filetype:pdf`) before full-text scoring.
- **Pinecone metadata filtering:** Vector DB with built-in metadata filter support. `{ "filter": { "category": { "$eq": "support" }, "date": { "$gte": 1704067200 } } }`. Used by Jasper for filtered document retrieval.
- **Weaviate's where filter:** `where: { path: ["tenant"], operator: Equal, valueString: "acme_corp" }` applied before vector search.
- **Qdrant's payload filtering:** Filter conditions on document payload (metadata) combined with vector search in a single query.
- **Airtable's search:** Each base has fields (metadata) that can filter records before full-text or semantic search.

## 4. Architecture Diagram (ASCII)
```
           Without Metadata Filtering
           ==========================
Query ──> Vector DB ──> ANN Search (all 10M) ──> Top-K ──> User
           (Slow, may return docs from wrong category)

           With Metadata Filtering
           =======================
Query ──> Vector DB
            |
            v
        [Metadata Filter: tenant_id=123, category="support"]
            |
            v
        [ANN Search on filtered subset (100K)]
            |
            v
        Top-K Results (only support docs for tenant 123) ──> User
```

## 5. Internal Working
1. **Metadata attachment during ingestion:** When a chunk is indexed, metadata fields are stored alongside the vector in the vector DB: `{vector: [0.1, 0.2, ...], payload: {tenant_id: "acme", category: "support", date: 1704067200, author: "john"}}`.
2. **Filter specification at query time:** The user or application specifies filter conditions: `{"category": {"$eq": "support"}}`.
3. **Pre-filtering (most common):** The vector DB applies the filter to its index, restricting ANN search to only vectors whose metadata matches. The distance computation only runs on filtered candidates.
4. **Post-filtering (less common):** ANN search runs on the full index, then metadata filters are applied to the top-K results. Bad: you may get zero results if the top-K are all from the wrong category.
5. **Hybrid filtering:** Use pre-filtering for high-selectivity filters (tenant, access control) and post-filtering for low-selectivity filters (tags, categories).

## 6. Production Flow
```
Ingestion:
  Document → Chunk → Embed → Store(vector + metadata)
    metadata = {
      "source": "knowledge_base.pdf",
      "section": "API Reference",
      "date_added": 1704067200,
      "tenant_id": "acme",
      "access_level": "admin"
    }

Query:
  User query: "How do I use the REST API?"
  Filters: {"tenant_id": "acme", "section": "API Reference"}
    |
    v
  Vector DB applies filters → searches only "acme" docs in "API Reference" section
    |
    v
  Returns top-K from filtered space → LLM generates answer
```

## 7. HLD
```
                    +------------------+
                    | Query            |
                    +--------+---------+
                             |
              +--------------+--------------+
              |                             |
    +---------v--------+          +---------v--------+
    | Query Text       |          | Metadata Filter  |
    | (embedding)      |          | (tenant, date,   |
    +---------+--------+          |  category, tags)  |
              |                   +---------+--------+
              |                             |
    +---------v-----------------------------v--+
    |      Vector DB Query                     |
    |      {                                    |
    |        "vector": [0.1, 0.2, ...],        |
    |        "filter": {                        |
    |          "tenant_id": "acme",             |
    |          "date": {"$gte": 1704067200}     |
    |        },                                 |
    |        "top_k": 10                        |
    |      }                                    |
    +-------------------+----------------------+
                        |
              +---------v---------+
              | Filtered Results  |
              +-------------------+
```

## 8. LLD
```python
from dataclasses import dataclass, field
from enum import Enum
from typing import Any, Dict, List, Optional

class FilterOperator(str, Enum):
    EQ = "$eq"
    NE = "$ne"
    GT = "$gt"
    GTE = "$gte"
    LT = "$lt"
    LTE = "$lte"
    IN = "$in"
    NIN = "$nin"
    AND = "$and"
    OR = "$or"

@dataclass
class MetadataFilter:
    field: str
    operator: FilterOperator
    value: Any

    def to_dict(self) -> dict:
        return {self.field: {self.operator: self.value}}

    @staticmethod
    def and_(*filters: "MetadataFilter") -> dict:
        return {"$and": [f.to_dict() for f in filters]}

    @staticmethod
    def or_(*filters: "MetadataFilter") -> dict:
        return {"$or": [f.to_dict() for f in filters]}

@dataclass
class MetadataFilterConfig:
    enabled: bool = True
    default_filters: dict = field(default_factory=dict)
    pre_filter: bool = True  # True = pre-filter, False = post-filter

class FilteredRetriever:
    def __init__(self, vector_db_client, config: MetadataFilterConfig):
        self.client = vector_db_client
        self.config = config

    async def search(
        self,
        query_vector: List[float],
        filters: Optional[Dict[str, Any]] = None,
        top_k: int = 10,
    ) -> List[dict]:
        # Merge default filters with request filters
        merged = {**self.config.default_filters, **(filters or {})}

        if not merged or not self.config.enabled:
            return await self.client.search(query_vector, top_k=top_k)

        if self.config.pre_filter:
            return await self.client.search(
                query_vector,
                filter=merged,
                top_k=top_k,
            )
        else:
            # Post-filter: search without filter, then apply
            results = await self.client.search(query_vector, top_k=top_k * 3)
            return self._apply_filter(results, merged)[:top_k]

    def _apply_filter(self, results: List[dict], filters: dict) -> List[dict]:
        filtered = []
        for r in results:
            if self._matches(r.get("metadata", {}), filters):
                filtered.append(r)
        return filtered

    def _matches(self, metadata: dict, filters: dict) -> bool:
        for key, condition in filters.items():
            if key == "$and":
                if not all(self._matches(metadata, c) for c in condition):
                    return False
            elif key == "$or":
                if not any(self._matches(metadata, c) for c in condition):
                    return False
            else:
                val = metadata.get(key)
                for op, target in condition.items():
                    if op == "$eq" and val != target:
                        return False
                    elif op == "$ne" and val == target:
                        return False
                    elif op == "$gte" and not (val >= target):
                        return False
                    elif op == "$lte" and not (val <= target):
                        return False
                    elif op == "$in" and val not in target:
                        return False
        return True
```

## 9. Python Implementation
```python
from enum import Enum
from typing import Any, Dict, List, Optional

from fastapi import FastAPI
from pydantic import BaseModel, Field

app = FastAPI(title="Metadata Filtering Service")

class FilterCondition(BaseModel):
    field: str
    operator: str = Field(pattern=r"^\$(eq|ne|gt|gte|lt|lte|in|nin)$")
    value: Any

class SearchRequest(BaseModel):
    query_vector: List[float] = Field(..., min_length=128, max_length=4096)
    filters: Optional[Dict[str, Any]] = None
    top_k: int = Field(default=10, ge=1, le=100)
    pre_filter: bool = Field(default=True)

class SearchResult(BaseModel):
    doc_id: str
    score: float
    text: str
    metadata: Dict[str, Any]

class SearchResponse(BaseModel):
    results: List[SearchResult]
    filter_applied: bool
    total_candidates_before_filter: Optional[int]

class MetadataFilterService:
    def __init__(self):
        self.docs = [
            {
                "id": "doc_1", "text": "Q3 earnings report",
                "vector": [0.1] * 384,
                "metadata": {"tenant": "acme", "date": 20240901, "category": "financial"},
            },
            {
                "id": "doc_2", "text": "API authentication guide",
                "vector": [0.2] * 384,
                "metadata": {"tenant": "acme", "date": 20240815, "category": "technical"},
            },
            {
                "id": "doc_3", "text": "Employee onboarding checklist",
                "vector": [0.3] * 384,
                "metadata": {"tenant": "globex", "date": 20240910, "category": "hr"},
            },
        ]

    async def search(self, req: SearchRequest) -> SearchResponse:
        import time
        start = time.time()

        if req.pre_filter and req.filters:
            candidates = self._pre_filter(req.filters)
        else:
            candidates = self.docs

        # "Vector search" (simulated: cosine sim with random)
        scored = []
        for doc in candidates:
            sim = self._cosine_sim(req.query_vector, doc["vector"])
            scored.append((sim, doc))

        scored.sort(key=lambda x: x[0], reverse=True)
        top = scored[:req.top_k]

        if not req.pre_filter and req.filters:
            top = self._post_filter(top, req.filters)[:req.top_k]

        return SearchResponse(
            results=[
                SearchResult(
                    doc_id=doc["id"],
                    score=score,
                    text=doc["text"],
                    metadata=doc["metadata"],
                )
                for score, doc in top
            ],
            filter_applied=req.filters is not None,
            total_candidates_before_filter=len(candidates),
        )

    def _pre_filter(self, filters: dict) -> list:
        return [d for d in self.docs if self._matches(d["metadata"], filters)]

    def _post_filter(self, scored: list, filters: dict) -> list:
        return [(s, d) for s, d in scored if self._matches(d["metadata"], filters)]

    def _matches(self, metadata: dict, filters: dict) -> bool:
        for key, condition in filters.items():
            if key == "$and":
                if not all(self._matches(metadata, c) for c in condition):
                    return False
            elif key == "$or":
                if not any(self._matches(metadata, c) for c in condition):
                    return False
            else:
                val = metadata.get(key)
                if isinstance(condition, dict):
                    for op, target in condition.items():
                        if op == "$eq" and val != target:
                            return False
                        elif op == "$ne" and val == target:
                            return False
                        elif op == "$gte" and not (val >= target):
                            return False
                        elif op == "$lte" and not (val <= target):
                            return False
                        elif op == "$in" and val not in target:
                            return False
                else:
                    if val != condition:
                        return False
        return True

    def _cosine_sim(self, a: list, b: list) -> float:
        import math
        dot = sum(x * y for x, y in zip(a, b))
        na = math.sqrt(sum(x * x for x in a))
        nb = math.sqrt(sum(x * x for x in b))
        return dot / (na * nb + 1e-8)

service = MetadataFilterService()

@app.post("/search", response_model=SearchResponse)
async def search(req: SearchRequest):
    return await service.search(req)
```

## 10. Folder Structure
```
metadata-filtering/
├── src/
│   ├── __init__.py
│   ├── main.py
│   ├── filter.py
│   ├── filter_engine.py
│   ├── models/
│   │   ├── __init__.py
│   │   └── schemas.py
│   └── utils.py
├── tests/
├── deploy/
└── pyproject.toml
```

## 11. Configuration
```yaml
metadata_filtering:
  enabled: true
  pre_filter: true  # true = pre-filter, false = post-filter
  default_filters: {}
  allowed_fields:
    - tenant_id
    - category
    - date
    - author
    - access_level
  operators:
    - $eq
    - $ne
    - $gt
    - $gte
    - $lt
    - $lte
    - $in
    - $nin
  max_filter_depth: 3  # nested $and/$or depth limit
```

## 12. Flowchart
```
Start
  |
  v
[Receive Query + Filters]
  |
  v
{Pre-filter or Post-filter?}
  |
  |--Pre-filter:
  |     |
  |     v
  |   [Apply Filter on Index]
  |     |
  |     v
  |   [ANN Search on Filtered Set]
  |     |
  |     v
  |   [Return Top-K]
  |
  |--Post-filter:
  |     |
  |     v
  |   [ANN Search on Full Index]
  |     |
  |     v
  |   [Apply Filter on Results]
  |     |
  |     v
  |   [Return Top-K from Filtered]
  |
  v
  End
```

## 13. Sequence Diagram
```
Client      VectorDB      FilterEngine     Index
  |            |              |              |
  |--Query---->|              |              |
  | +filters   |              |              |
  |            |--pre-filter->|              |
  |            |<--subset-----|              |
  |            |              |              |
  |            |--ANN-------->|              |
  |            |              |--search----->|
  |            |              |<--candidates-|
  |            |<--top-k------|              |
  |            |              |              |
  |<--Results--|              |              |
```

## 14. Pros
- Dramatically improves retrieval precision when filters are relevant
- Reduces ANN search space → faster queries, lower cost
- Enables multi-tenant isolation with a single vector DB
- Supports access control at the document level
- No additional model training — metadata is set at ingestion time

## 15. Cons
- Pre-filtering can exclude relevant documents if the filter is too restrictive
- Post-filtering can return zero results if top-K are all filtered out
- Adds metadata management overhead during ingestion
- Vector DB support for complex filters varies widely (nested $and/$or)
- Filter performance depends on the vector DB's indexing strategy

## 16. Alternatives
| Method | Description |
|--------|-------------|
| No filtering | Search all docs, rely on embedding similarity alone |
| Per-tenant vector DB | Separate DB instance per tenant (costly but simple) |
| Tag-based partitioning | Pre-partition index by metadata key |
| Hybrid approach | Pre-filter on high-selectivity fields, post-filter on low-selectivity |
| Document reranking with metadata | Rank by combined embedding score + metadata score |

## 17. Performance Considerations
- **Pre-filter selectivity:** High-selectivity filters (e.g., tenant_id = "acme") reduce 10M to 100K (100x speedup). Low-selectivity filters (e.g., category = "news") only reduce 10M to 9M.
- **Filter index type:** Vector DBs use different filter indexing. Qdrant uses a payload index. Pinecone uses a metadata index. Both add O(log N) per filter.
- **Post-filter overhead:** Requires retrieving 3-5x more candidates to avoid empty results after filtering.
- **AND vs OR:** AND filters are faster (intersection of sets). OR filters are slower (union).
- **Filter cardinality:** High-cardinality fields (date, timestamps) benefit from range indexes.

## 18. Scaling to Millions
- **Partition by tenant_id** — if one metadata field is always present, use it as a partition key.
- **Composite indexes** — create indexes on (tenant_id + date) for common filter combinations.
- **Filter pushdown** — ensure filters are pushed to the storage layer, not applied in application code.
- **Materialized views** — pre-filtered subsets for common filter combinations (e.g., "public docs").
- **Limit filter depth** — nested $and/$or with depth > 5 is exponentially expensive.

## 19. Failure Scenarios
| Failure | Impact | Mitigation |
|---------|--------|------------|
| Overly restrictive filter | Zero results | Fall back to no filter + rerank |
| Filter field missing | Filter ignored silently | Validate filter fields against schema |
| Deeply nested filter | Query timeout | Enforce max filter depth |
| High-cardinality filter without index | Full scan | Index all filterable fields |

## 20. Security
- Never allow users to filter by internal metadata fields (e.g., `access_level` could expose admin docs).
- Validate filter field allow-lists — reject filters on unknown fields.
- Apply tenant filter server-side — never trust the client's tenant_id.
- Sanitize filter values to prevent injection in string-based filters.
- Rate-limit queries with expensive filters (range scans on unindexed fields).

## 21. Monitoring
- **Filter selectivity:** Average % of documents filtered out
- **Pre-filter vs post-filter latency:** Pre-filters should be faster
- **Zero-result rate:** % of filtered queries returning 0 docs
- **Filter field usage:** Distribution of which metadata fields are most filtered
- **Filter complexity:** Average nesting depth of $and/$or filters
- **Index size:** Metadata index size vs vector index size

## 22. Interview Questions
| Level | Question | Key Points |
|-------|----------|------------|
| L3 | What is metadata filtering and why use it? | Pre-filter search space, enforce access control, multi-tenancy |
| L4 | Pre-filtering vs post-filtering — pros and cons? | Pre: faster, can miss relevant. Post: no misses, might return zero |
| L4 | How would you implement multi-tenant RAG with metadata filtering? | tenant_id field on every chunk, filter by it on every query |
| L5 | Design a metadata filtering system for 100M documents with 50 tenants | Partition by tenant, composite indexes on (tenant, date), pre-filter |
| L5 | How do you handle filter-only queries (no vector component)? | Vector DB should support pure metadata queries with inverted index |

## 23. Cheat Sheet
```python
# Metadata filtering in 5 lines
filters = {"tenant_id": "acme", "date": {"$gte": 20240101}}
results = vector_db.search(
    vector=query_emb,
    filter=filters,  # pre-filter
    top_k=10
)
```

## 24. Common Mistakes
- **Not filtering at all** — searching every document when 99% are irrelevant to the current user.
- **Post-filtering as default** — misses relevant docs because they weren't in the top-K before filter.
- **Unvalidated filter fields** — allowing users to filter by `access_level` or other security-critical fields.
- **Missing metadata on chunks** — if a chunk doesn't have the filter field, it's invisible to queries using that filter.
- **Too many filter fields** — 30 metadata fields per chunk creates excessive storage and query overhead.
- **Filtering on unindexed fields** — causes full table scans at query time.

## 25. Production Best Practices
1. **Always use pre-filtering** for high-selectivity filters (tenant, date range, access level).
2. **Index all filterable fields** — every field that can appear in a filter should have a database index.
3. **Validate filter fields against an allow-list** — reject unknown or security-sensitive fields.
4. **Apply tenant isolation server-side** — the tenant_id filter must come from authentication, not user input.
5. **Set a fallback for zero-result filters** — if a filter returns nothing, retry with a relaxed filter.
6. **Use AND over OR** — AND filters are more efficient and the vector DB can skip more candidates.
7. **Limit filter nesting depth** — prevent exponential query complexity from deeply nested $and/$or.
8. **Attach metadata at chunk time, not query time** — every chunk should carry its metadata from ingestion.
9. **Monitor filter selectivity** — if a filter matches 100% of documents, it's doing nothing.
10. **Pre-filter by access control before any other filter** — it's the highest-selectivity and most security-critical.
