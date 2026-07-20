# Document Search Engine System Design

## 1. Requirements

### Functional Requirements
- Full-text search across 100M+ documents
- Semantic (vector) search for meaning-based queries
- Hybrid search combining text, metadata, and embeddings
- Faceted filtering (date, author, category, tenant)
- Relevance ranking with learn-to-rank
- Multi-language support (50+ languages)
- Document preview — highlight matched terms in snippets
- Spell correction and query suggestions ("Did you mean...?")
- Boolean search (AND, OR, NOT, wildcards, proximity)
- Document-level access control (RBAC)
- Batch indexing and real-time incremental indexing
- Personalization — boost results based on user history
- Synonyms and custom dictionaries per tenant

### Non-Functional Requirements
- **Latency**: P99 < 300ms for search
- **Availability**: 99.99%
- **Throughput**: 20,000 QPS peak
- **Index freshness**: < 60s from document change to searchable
- **Consistency**: Near real-time (NRT) — eventual consistency acceptable
- **Scale**: 100M documents, 500M indexed chunks
- **Storage**: 10 TB primary, 20 TB with replicas
- **Cost**: < $0.001 per search query

## 2. Capacity Estimation

| Metric | Value |
|--------|-------|
| Total documents | 100M |
| Avg document size | 50 KB (extracted text) |
| Chunks per document | ~20 (avg 512 tokens) |
| Total indexed chunks | 2B |
| Avg chunk size | 1 KB |
| Index storage (inverted) | 2B × 500 bytes ≈ 1 TB |
| Vector storage (1536d float32) | 2B × 1536 × 4 = ~12 TB |
| Metadata store | 100M × 500 bytes = 50 GB |
| Daily new docs | 500K |
| Query QPS | 20,000 peak |
| Avg query size | 40 bytes |
| Total query throughput | 800 KB/s ingress, 40 MB/s egress (results) |
| Log storage (query logs) | 20K × 86400 × 200 bytes = ~345 GB/day |

## 3. High-Level Architecture

```
                         ┌──────────────────────────────┐
                         │      API Gateway / LB          │
                         │  TLS Term, Auth, Rate Limit    │
                         └─────────────┬────────────────┘
                                       │
                                       ▼
┌──────────────────────────────────────────────────────────────────┐
│                       Query Pipeline                              │
│                                                                  │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌───────────────┐   │
│  │ Query    │  │ Query    │  │ Query    │  │ Spell         │   │
│  │ Parser   │─►│ Rewriter │─►│ Expander │─►│ Corrector     │   │
│  │ (Tokens) │  │ (Synonym)│  │ (Related)│  │ (Edit Dist)   │   │
│  └──────────┘  └──────────┘  └──────────┘  └───────────────┘   │
│                      │                                           │
│                      ▼                                           │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │               Query Federation                            │   │
│  │                                                           │   │
│  │  ┌─────────────┐  ┌─────────────┐  ┌────────────────┐    │   │
│  │  │ Inverted    │  │ Vector      │  │ Metadata       │    │   │
│  │  │ Index Search│  │ Search      │  │ Filter         │    │   │
│  │  │ (BM25/ES)    │  │ (ANN/HNSW)  │  │ (SQL/Postgres) │    │   │
│  │  └─────────────┘  └─────────────┘  └────────────────┘    │   │
│  └──────────────────────────────────────────────────────────┘   │
│                      │                                           │
│                      ▼                                           │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │               Fusion & Ranking                            │   │
│  │  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐ │   │
│  │  │ RRF Fuse │─►│ Feature  │─►│ LTR      │─►│ Personal │ │   │
│  │  │ (Merge)  │  │ Extractor│  │ (LambdaMART)│ Boost    │ │   │
│  │  └──────────┘  └──────────┘  └──────────┘  └──────────┘ │   │
│  └──────────────────────────────────────────────────────────┘   │
│                      │                                           │
│                      ▼                                           │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │               Response Formatter                          │   │
│  │  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐ │   │
│  │  │ Snippet  │─►│ Highlight│─►| Facets   │─►│ Suggest  │ │   │
│  │  │ Generator│  │ Generator│  │ Counts   │  │ Builder  │ │   │
│  │  └──────────┘  └──────────┘  └──────────┘  └──────────┘ │   │
│  └──────────────────────────────────────────────────────────┘   │
└──────────────────────────────────────────────────────────────────┘
                           │
                           ▼
┌──────────────────────────────────────────────────────────────────┐
│                       Indexing Pipeline                           │
│                                                                  │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌───────────────┐   │
│  │ Document │  │ Document │  │ Chunk &  │  │ Embedding      │   │
│  │ Fetcher  │─►│ Parser   │─►│ Splitter │─►│ Generator      │   │
│  │ (Poll/   │  │ (Tika/   │  │ (LangChain) │ (Batch/Async)  │   │
│  │ Webhook) │  │ Unstruct)│  │          │  │               │   │
│  └──────────┘  └──────────┘  └──────────┘  └───────┬───────┘   │
│                                                     │           │
│                                                     ▼           │
│  ┌──────────┐  ┌──────────┐  ┌────────────────────────────┐    │
│  │ Index    │  │ Index    │  │ Metadata Indexer            │    │
│  │ Writer   │  │ Writer   │  │ (PostgreSQL)                 │    │
│  │ (ES)     │  │ (Vector) │  │                              │    │
│  └──────────┘  └──────────┘  └──────────────────────────────┘   │
└──────────────────────────────────────────────────────────────────┘
                           │
                           ▼
┌──────────────────────────────────────────────────────────────────┐
│                       Storage Layer                               │
│  ┌────────────────┐  ┌────────────────┐  ┌────────────────────┐ │
│  │ Elasticsearch  │  │ Vector DB      │  │ PostgreSQL         │ │
│  │ 15 data nodes  │  │ (Pinecone/     │  │ (Metadata, Users,  │ │
│  │ 8TB total      │  │  Weaviate)     │  │  Access Control)   │ │
│  │                │  │ 24 pods p2x7   │  │ 3x read replicas   │ │
│  └────────────────┘  └────────────────┘  └────────────────────┘ │
│  ┌────────────────┐  ┌────────────────┐                         │
│  │ Redis          │  │ S3             │                         │
│  │ (Query Cache)  │  │ (Raw Docs)     │                         │
│  │ 50 GB cluster  │  │ 10 TB bucket   │                         │
│  └────────────────┘  └────────────────┘                         │
└──────────────────────────────────────────────────────────────────┘
```

## 4. Low-Level Design

### Query Pipeline Components

```python
class SearchRequest:
    query: str
    filters: SearchFilters  # date, author, category, tenant
    page: int = 1
    page_size: int = 20
    sort: SortOption = SortOption.RELEVANCE
    user_id: UUID | None  # for personalization
    search_type: SearchType = SearchType.HYBRID

class SearchFilters:
    tenant_id: UUID
    date_range: tuple[datetime, datetime] | None
    authors: list[str] | None
    categories: list[str] | None
    file_types: list[str] | None
    access_levels: list[str] | None

class QueryProcessor:
    async def process(request: SearchRequest) -> ProcessedQuery:
        # 1. Parse query into tokens
        tokens = self.tokenizer.tokenize(request.query)

        # 2. Expand with synonyms
        expanded = await self.synonym_expander.expand(tokens, request.filters.tenant_id)

        # 3. Spell correction
        corrected = await self.spell_checker.correct(request.query)

        # 4. Detect query type (textual, semantic, code, etc.)
        query_type = self.query_classifier.classify(corrected or request.query)

        return ProcessedQuery(
            original=request.query,
            corrected=corrected,
            tokens=expanded,
            query_type=query_type,
            embedding=await self.embedder.embed(request.query) if request.search_type != SearchType.TEXT_ONLY else None
        )

class HybridRetriever:
    def __init__(self):
        self.es = ElasticsearchClient()
        self.vector = VectorDBClient()
        self.metadata = MetadataStore()

    async def retrieve(self, query: ProcessedQuery, filters: SearchFilters) -> list[SearchResult]:
        # Parallel queries to all backends
        text_results = self.es.search(query.tokens, filters)
        vector_results = self.vector.search(query.embedding, filters, top_k=100)
        filtered_ids = self.metadata.filter_ids(filters)

        # Reciprocal Rank Fusion
        fused = self.rrf_fuse(text_results, vector_results, k=60)

        # Apply metadata filter
        fused = [r for r in fused if r.id in filtered_ids]

        return fused[:20]  # Reranker will re-rank top 20

    def rrf_fuse(self, lists: list[list[SearchResult]], k: int = 60) -> list[dict]:
        scores = defaultdict(float)
        for rank, result_list in enumerate(lists):
            for idx, result in enumerate(result_list):
                scores[result.id] += 1.0 / (k + idx)
        return sorted(scores.items(), key=lambda x: -x[1])
```

### Indexing Pipeline Components

```python
class IndexingPipeline:
    async def ingest(document: RawDocument) -> DocumentIndexed:
        # 1. Fetch and parse
        parsed = await self.parser.parse(document)

        # 2. Chunk
        chunks = self.chunker.chunk(parsed, strategy=parsed.recommended_strategy)

        # 3. Generate embeddings (batch)
        embeddings = await self.embedder.embed_batch([c.text for c in chunks])

        # 4. Index to all stores
        es_fut = self.es.bulk_index(parsed.id, chunks)
        vec_fut = self.vector.upsert([
            (chunk.id, emb, chunk.metadata) for chunk, emb in zip(chunks, embeddings)
        ])
        meta_fut = self.metadata.index_document(parsed.metadata)

        await asyncio.gather(es_fut, vec_fut, meta_fut)

        return DocumentIndexed(parsed.id, len(chunks), status="active")

class Chunker:
    def chunk(self, doc: ParsedDocument) -> list[Chunk]:
        if doc.type == DocumentType.MARKDOWN:
            return self.markdown_splitter.split(doc.content)
        elif doc.type == DocumentType.CODE:
            return self.code_splitter.split(doc.content, doc.language)
        elif doc.type == DocumentType.TABLE_HEAVY:
            return self.table_aware_splitter.split(doc.content)
        else:
            return self.recursive_splitter.split(doc.content, chunk_size=512, overlap=128)
```

## 5. Database Schema

### PostgreSQL — Metadata & ACL

```sql
CREATE TABLE documents (
    id UUID PRIMARY KEY,
    tenant_id UUID NOT NULL,
    external_id TEXT,
    title TEXT NOT NULL,
    author TEXT,
    source_type TEXT NOT NULL,  -- confluence, sharepoint, s3, upload
    source_uri TEXT NOT NULL,
    file_type TEXT,  -- pdf, docx, md, html, py, js
    language TEXT,
    content_hash TEXT,
    page_count INT,
    chunk_count INT,
    status TEXT DEFAULT 'pending',
    size_bytes BIGINT,
    indexed_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ DEFAULT NOW(),
    updated_at TIMESTAMPTZ DEFAULT NOW()
);
CREATE INDEX idx_docs_tenant_status ON documents(tenant_id, status);
CREATE INDEX idx_docs_author ON documents(author);
CREATE INDEX idx_docs_created ON documents(created_at);

CREATE TABLE document_access (
    document_id UUID REFERENCES documents(id) ON DELETE CASCADE,
    tenant_id UUID NOT NULL,
    group_id TEXT NOT NULL,  -- RBAC group
    permission TEXT DEFAULT 'read',
    PRIMARY KEY (document_id, group_id)
);

CREATE TABLE search_synonyms (
    id UUID PRIMARY KEY,
    tenant_id UUID NOT NULL,
    term TEXT NOT NULL,
    synonyms TEXT[] NOT NULL,  -- ["automobile", "vehicle", "car"]
    created_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE TABLE query_log (
    id UUID PRIMARY KEY,
    tenant_id UUID,
    user_id UUID,
    query_text TEXT,
    corrected_text TEXT,
    result_count INT,
    clicked_docs UUID[],  -- document IDs clicked
    latency_ms INT,
    search_type TEXT,
    created_at TIMESTAMPTZ DEFAULT NOW()
);
CREATE INDEX idx_query_log_tenant ON query_log(tenant_id, created_at);
```

### Elasticsearch Mapping

```json
{
  "settings": {
    "number_of_shards": 24,
    "number_of_replicas": 2,
    "analysis": {
      "analyzer": {
        "code_analyzer": {
          "type": "custom",
          "tokenizer": "standard",
          "filter": ["lowercase", "stop"]
        }
      }
    }
  },
  "mappings": {
    "properties": {
      "content": {
        "type": "text",
        "analyzer": "standard",
        "fields": {
          "english": { "type": "text", "analyzer": "english" },
          "keyword": { "type": "keyword" }
        }
      },
      "title": { "type": "text", "boost": 3.0 },
      "author": { "type": "keyword" },
      "tenant_id": { "type": "keyword" },
      "file_type": { "type": "keyword" },
      "created_at": { "type": "date" },
      "chunk_index": { "type": "integer" },
      "heading": { "type": "text", "boost": 2.0 },
      "embedding": {
        "type": "dense_vector",
        "dims": 1536,
        "index": true,
        "similarity": "cosine"
      }
    }
  }
}
```

### Vector DB Index Schema

```python
# Pinecone Index
name: "doc-search-{tenant_shard}"
dimension: 1536
metric: cosine
pods: 24 x p2x7
metadata_config:
  indexed:
    - tenant_id
    - doc_id
    - chunk_index
    - file_type
    - language
    - created_at
```

## 6. API Contract

### GET /api/v1/search

```json
// Request
GET /api/v1/search?q=parental+leave+policy&tenant_id=t-123&page=1&size=20&sort=relevance
&filter.date_from=2024-01-01&filter.author=HR&filter.file_type=pdf

// Response
{
  "query": "parental leave policy",
  "corrected_query": null,
  "total_results": 42,
  "page": 1,
  "page_size": 20,
  "results": [
    {
      "id": "doc-uuid-1",
      "title": "Employee Benefits Handbook 2025",
      "snippet": "...offers <em>parental leave</em> of 16 weeks... The <em>policy</em> applies to all full-time...",
      "highlights": ["parental leave", "policy"],
      "author": "HR Team",
      "file_type": "pdf",
      "source_uri": "https://sharepoint.company.com/hr/benefits.pdf",
      "score": 0.94,
      "matched_by": "hybrid",
      "chunks": [
        {"chunk_index": 3, "heading": "Parental Leave Overview", "content": "..."},
        {"chunk_index": 4, "heading": "Eligibility", "content": "..."}
      ],
      "created_at": "2025-01-15T10:00:00Z"
    }
  ],
  "facets": {
    "file_type": {
      "pdf": 20,
      "docx": 12,
      "md": 8,
      "html": 2
    },
    "author": {
      "HR Team": 25,
      "Legal": 10,
      "Engineering": 7
    },
    "created_at": {
      "last_week": 3,
      "last_month": 15,
      "last_year": 42
    }
  },
  "suggestions": ["parental leave", "paternity leave policy", "maternity leave"],
  "latency_ms": 145,
  "search_type": "hybrid"
}
```

### POST /api/v1/documents

```json
// Index a document
POST /api/v1/documents
{
  "tenant_id": "t-123",
  "source_type": "s3",
  "source_uri": "s3://company-docs/hr/policy.pdf",
  "title": "HR Policy 2025",
  "author": "HR Team",
  "metadata": {
    "department": "HR",
    "region": "US"
  },
  "access_groups": ["all_employees", "hr_team"]
}

// Response
{
  "doc_id": "doc-uuid",
  "status": "indexing",
  "estimated_seconds": 15
}
```

### GET /api/v1/suggest?q=parent

```json
{
  "suggestions": [
    {"text": "parental leave policy", "score": 0.95, "type": "popular"},
    {"text": "parental leave", "score": 0.89, "type": "completion"},
    {"text": "parent-teacher conference", "score": 0.72, "type": "fuzzy"}
  ]
}
```

## 7. Sequence Diagram

```
User        Gateway     QueryProc   Retriever     ES     VectorDB    Metadata    Logger
 │            │            │           │           │        │          │          │
 ├─Search─────►            │           │           │        │          │          │
 │            ├─Auth──────►│           │           │        │          │          │
 │            │            ├─Parse────►│           │        │          │          │
 │            │            │           │           │        │          │          │
 │            │            │           ├─BM25──────►         │          │          │
 │            │            │           ├─Vector──────────────►          │          │
 │            │            │           ├─Filter─────────────────────────►          │
 │            │            │           │           │        │          │          │
 │            │            │           ├─RRF─────────────────┤          │          │
 │            │            │           ├─Rerank──►│        │          │          │
 │            │            │           │           │        │          │          │
 │            │            │◄─Results──┤           │        │          │          │
 │            │            ├─Format─────────────────┤          │          │          │
 │            │◄─Response──┤           │           │        │          │          │
 │◄─Display───┤            │           │           │        │          │          │
 │            │            │           │           │        │          ├─Log──────►│
```

## 8. Scaling Strategy

| Component | Strategy |
|-----------|----------|
| Elasticsearch | 24 data nodes (i3.2xlarge), 2 replicas, ILM for time-based indices, cross-cluster search for geo-distribution |
| Vector DB | Index per tenant shard; HNSW with ef_construction=200, ef_search=256; nightly optimization |
| Query Processor | Stateless, auto-scale on CPU > 70%; ONNX runtime for embedder |
| Reranker (LTR) | Cross-encoder distilled model (6 layers); GPU batch inference every 50ms |
| Facet Computation | Pre-computed for common filters; on-demand with cardinality approximation for rare |
| Cache | Redis cluster — TTL = 60s for popular queries, 300s for suggestions |
| Indexing Pipeline | SQS FIFO queue; batch to ES (500 docs/batch); parallel embedding via async |

## 9. Failure Handling

| Failure | Mitigation |
|---------|-----------|
| ES node failure | Replica shard promotion; rebalance within 60s |
| Vector DB timeout | Degrade to text-only search; log and alert |
| Embedding service down | Skip vector search; rely on BM25 |
| Index stale | Webhook notification → reindex single doc; full nightly rebuild for consistency |
| High query latency | Fallback to cache (stale results with freshness indicator) |
| Corrupted index | Snapshot-based recovery from S3 (every 6 hours) |
| Thundering herd on fresh content | Bulkhead for indexing vs query; indexing gets dedicated resources |

## 10. Monitoring

| Metric | Tool | Target |
|--------|------|--------|
| P50/P95/P99 latency | Prometheus + Grafana | P99 < 300ms |
| Recall@10 | A/B evaluation pipeline | > 90% |
| Precision@10 | A/B evaluation | > 80% |
| Zero result rate | Kibana | < 2% |
| Click-through rate (CTR) | Elasticsearch → dashboard | > 35% |
| Index lag | Custom metric | < 60s |
| Cache hit ratio | Redis metrics | > 60% |
| Query throughput | Grafana | Per-node < 200 QPS |
| Shard health | ES cluster health | green |
| Embedding queue depth | SQS monitoring | < 10K |

## 11. Security

- **Document-level ACL**: Filter search results by user's group membership. ES query must include `terms` filter on groups.
- **Tenant isolation**: Separate ES index per tenant; tenant_id enforced on every query. Vector DB namespace per tenant.
- **HTTPS only**: TLS 1.3 for all traffic. mTLS between internal services.
- **Query audit**: Every query logged with user_id, tenant_id, timestamp, results (anonymized).
- **Rate limiting**: Per-tenant (10K/min), per-user (100/min), per-IP (500/min).
- **Injection prevention**: Query parser rejects control characters; regex-based guard on query input.
- **Data retention**: Raw documents retained 90 days; search indices rebuilt on retention policy.
- **PII**: Pre-indexing PII removal configurable per tenant.

## 12. Trade-offs

| Decision | Pros | Cons |
|----------|------|------|
| HNSW vs IVF for ANN | HNSW = faster, higher recall | More memory per index |
| ES + Vector DB vs single (Elasticsearch NNS) | NNS = simpler ops | NNS = slower recall, no GPU acceleration |
| RRF fusion vs weighted sum | RRF = parameter-free, robust | Weighted sum can be tuned per query type |
| Real-time indexing vs batch | Real-time = fresh results | More writes, higher resource cost |
| Full reindex vs incremental | Incremental = fast updates | Can miss term frequency changes; periodic full reindex needed |
| Cross-encoder reranker vs Cohere | Self-hosted = no data leaving VPC | 2x infra cost vs API |
| LTR model vs static BM25 | LTR = 15-20% relevance improvement | Requires click feedback data, retraining pipeline |

## 13. Interview Tips

- **Start with search types**: Distinguish keyword search (BM25) vs semantic search (vectors) vs hybrid. Show you know when each is appropriate.
- **Ranking is everything**: Explain the funnel — recall phase (top 1000) → precision phase (top 50) → rerank phase (top 20). Draw the inverted pyramid.
- **Indexing pipeline depth**: Show you understand incremental indexing, partial updates, document deletion handling, and consistency model.
- **Freshness vs latency trade-off**: Real-time indexing requires log-based CDC (Kafka + Debezium) for DB sources. Webhook-based for S3. Polling for SharePoint.
- **Facets are an underrated challenge**: Pre-computed vs on-demand faceting. Cardinality estimation (HyperLogLog) for high-cardinality facets.
- **Scale numbers matter**: 20K QPS, 100M docs, 2B chunks. Be ready with back-of-envelope for ES shard count (rule of thumb: 20-40 GB per shard).
- **Common variants**: "Design Google Docs search", "Design an internal enterprise search like Glean", "Design a legal document search engine", "Design a code search engine (like Sourcegraph)".
- **Evaluation methodology**: Offline (NDCG, MRR on held-out queries) + online (CTR, long-click rate, abandonment rate). A/B test ranking changes.
