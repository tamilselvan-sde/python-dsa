# Enterprise RAG System Design

> Author: **Tamilselvan** · ✉️ tamilselvan.sde@gmail.com · 🔗 [LinkedIn](https://www.linkedin.com/in/tamilselvan-ai/)
>

## 1. Requirements

### Functional Requirements
- Ingest documents from 50+ formats (PDF, DOCX, HTML, Markdown, Confluence, Notion, SharePoint, email)
- Semantic search across enterprise knowledge corpus
- Answer generation grounded in retrieved documents
- Source citation and provenance tracking
- Multi-tenant isolation with RBAC
- Incremental indexing — no full rebuild
- Document versioning and change detection
- Support for tables, images, code blocks in documents
- Manual curation — allow admins to add/edit/delete chunks
- Feedback loop — thumbs up/down on answers for improvement

### Non-Functional Requirements
- **Indexing latency**: < 5 min for a 100-page document
- **Query latency**: P99 < 1s (retrieval + generation)
- **Recall@5**: > 90%
- **Freshness**: Index updates reflected within 2 minutes of source change
- **Availability**: 99.9%
- **Scale**: 10M+ documents, 10B+ chunks, 1000+ tenants
- **Compliance**: Data never leaves tenant boundary; SOC 2 Type II
- **Accuracy**: No hallucinated sources — answer must cite retrieved chunks

## 2. Capacity Estimation

| Metric | Value |
|--------|-------|
| Documents | 10M total |
| Avg document size | 50 KB (text), 2 MB (scanned PDF) |
| Chunks per doc | ~50 (avg 512 tokens) |
| Total chunks | 500M |
| Daily new docs | 50K |
| Query QPS | 5,000 peak |
| Embedding dimension | 1536 (ada-002) |
| Vector DB size | 500M × 1536 × 4 bytes = ~3 TB |
| Metadata index | ~200 GB (PostgreSQL) |
| Inverted index (BM25) | ~1.5 TB (Elasticsearch) |

## 3. High-Level Architecture

```
                         ┌──────────────────────────────┐
                         │    Document Sources           │
                         │  S3 │ Confluence │ SharePoint │
                         └────────────┬─────────────────┘
                                      │
                                      ▼
┌────────────────────────────────────────────────────────────────┐
│                     Ingestion Pipeline                           │
│                                                                  │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌───────────────┐   │
│  │ Document │  │ Document │  │ Chunk &  │  │ Embedding      │   │
│  │ Fetcher  │─►│ Parser   │─►│ Splitter │─►│ Generator      │   │
│  │ (Poll)   │  │ (Unstructured)│ (LangChain)│ (Batch API)   │   │
│  └──────────┘  └──────────┘  └──────────┘  └───────┬───────┘   │
│                                                     │           │
│  ┌──────────┐  ┌────────────────────────────────────┘           │
│  │ Chunk    │  │                                                │
│  │ Store    │◄─┘                                                │
│  └──────────┘                                                   │
└──────────────────────────────────────────────────────────────────┘
                           │
                           ▼
┌──────────────────────────────────────────────────────────────────┐
│                       Index Layer                                 │
│  ┌──────────────┐  ┌──────────────────┐  ┌────────────────┐     │
│  │ Vector DB    │  │ Elasticsearch    │  │ PostgreSQL     │     │
│  │ (Pinecone/   │  │ (BM25 + Filter)  │  │ (Metadata +    │     │
│  │  Weaviate)   │  │                  │  │  Relationships)│     │
│  └──────────────┘  └──────────────────┘  └────────────────┘     │
└──────────────────────────────────────────────────────────────────┘
                           ▲
                           │
┌──────────────────────────────────────────────────────────────────┐
│                        Query Pipeline                             │
│                                                                  │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌───────────────┐   │
│  │ Query    │  │ Query    │  │ Hybrid   │  │ Reranker      │   │
│  │ Router   │─►│ Embedder │─►│ Retriever│─►│ (Cross-       │   │
│  │          │  │          │  │ (Dense+  │  │  encoder)     │   │
│  │          │  │          │  │ BM25)    │  │               │   │
│  └──────────┘  └──────────┘  └────┬─────┘  └──────┬────────┘   │
│                                    │                │           │
│                                    ▼                ▼           │
│                              ┌─────────────────────────┐        │
│                              │  Generator (LLM)        │        │
│                              │  GPT-4o / Claude 3.5    │        │
│                              │  + Prompt Template      │        │
│                              │  + Citation Formatter   │        │
│                              └──────────┬──────────────┘        │
│                                         │                       │
│                                         ▼                       │
│                              ┌─────────────────────────┐        │
│                              │  Post-Processing         │        │
│                              │  - Guardrails (no PII)   │        │
│                              │  - Citation validation   │        │
│                              │  - Factuality check      │        │
│                              └─────────────────────────┘        │
└──────────────────────────────────────────────────────────────────┘
```

## 4. Low-Level Design

### Chunking Strategy (Critical Design Decision)

| Strategy | Use Case | Chunk Size | Overlap |
|----------|----------|------------|---------|
| RecursiveCharacter | General text | 512 tokens | 128 tokens |
| Semantic Splitter | Well-structured docs | Variable | None |
| Markdown/HTML | Web content | Per section | 1 sentence |
| Code-aware | Code docs | Per function | Docstring |
| Table-aware | Financial/Reports | Per row block | Row header |

### Component Classes

```python
class Document:
    doc_id: UUID
    tenant_id: UUID
    source_type: SourceType  # PDF | CONFLUENCE | SHAREPOINT
    source_uri: str
    title: str
    raw_content: bytes
    parsed_content: str
    content_type: str  # text | table | image
    metadata: dict
    version: int
    ingested_at: datetime

class Chunk:
    chunk_id: UUID
    doc_id: UUID
    tenant_id: UUID
    content: str
    embedding: list[float]
    chunk_index: int
    tokens: int
    heading: str | None
    table_data: dict | None
    source_page: int | None

class Retriever:
    async def retrieve(query: str, filters: RetrievalFilters) -> list[Chunk]:
        dense_results = await self.vector_store.similarity_search(query)
        sparse_results = await self.bm25_search(query)
        hybrid = self.reciprocal_rank_fusion(dense_results, sparse_results)
        return await self.reranker.rerank(query, hybrid)

class RAGPipeline:
    async def answer(query: str, context: QueryContext) -> RAGResponse:
        chunks = await self.retriever.retrieve(query, context.filters)
        prompt = self.template.format(
            query=query,
            chunks=[c.content for c in chunks],
            sources=[c.source_info for c in chunks]
        )
        response = await self.llm.generate(prompt)
        validated = await self.guardrails.validate(response)
        return RAGResponse(
            answer=validated.text,
            citations=[Citation(c.doc_id, c.chunk_id, c.content) for c in chunks[:3]],
            confidence=validated.confidence
        )
```

## 5. Database Schema

### PostgreSQL — Metadata Store

```sql
CREATE TABLE documents (
    id UUID PRIMARY KEY,
    tenant_id UUID NOT NULL,
    external_id TEXT,
    title TEXT NOT NULL,
    source_type TEXT NOT NULL,
    source_uri TEXT NOT NULL,
    content_hash TEXT, -- SHA-256 for dedup
    status TEXT DEFAULT 'pending' CHECK (status IN ('pending','indexing','active','failed')),
    page_count INT,
    file_size_bytes BIGINT,
    version INT DEFAULT 1,
    created_at TIMESTAMPTZ DEFAULT NOW(),
    updated_at TIMESTAMPTZ DEFAULT NOW()
);
CREATE INDEX idx_docs_tenant ON documents(tenant_id, status);

CREATE TABLE chunks (
    id UUID PRIMARY KEY,
    doc_id UUID REFERENCES documents(id) ON DELETE CASCADE,
    tenant_id UUID NOT NULL,
    chunk_index INT NOT NULL,
    content TEXT NOT NULL,
    tokens INT NOT NULL,
    heading TEXT,
    page_number INT,
    embedding_id TEXT, -- reference to vector store external ID
    metadata JSONB DEFAULT '{}',
    created_at TIMESTAMPTZ DEFAULT NOW()
);
CREATE INDEX idx_chunks_doc ON chunks(doc_id);
CREATE INDEX idx_chunks_tenant ON chunks(tenant_id);

CREATE TABLE tenants (
    id UUID PRIMARY KEY,
    name TEXT NOT NULL,
    plan TEXT DEFAULT 'enterprise',
    vector_index_name TEXT, -- per-tenant isolated index
    embedding_model TEXT DEFAULT 'text-embedding-3-large',
    chunk_strategy TEXT DEFAULT 'recursive',
    created_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE TABLE feedback (
    id UUID PRIMARY KEY,
    query_id UUID NOT NULL,
    tenant_id UUID NOT NULL,
    user_id UUID,
    query_text TEXT,
    answer_text TEXT,
    rating INT CHECK (rating BETWEEN 1 AND 5),
    selected_sources UUID[], -- chunk IDs user found useful
    created_at TIMESTAMPTZ DEFAULT NOW()
);
```

### Vector Index — Per-Tenant Namespace

```python
# Pinecone Index Configuration
index:
  name: "enterprise-rag-{tenant_shard}"
  dimension: 1536
  metric: cosine
  spec:
    pod_type: p1x2
    replicas: 2
    metadata_config:
      indexed: ["tenant_id", "doc_id", "chunk_index", "tokens"]

namespaces:
  per_tenant: true  # "tenant_{uuid}" namespace per tenant
```

### Elasticsearch — BM25 + Filtered Search

```python
mappings = {
    "properties": {
        "content": {"type": "text", "analyzer": "english"},
        "heading": {"type": "text"},
        "tenant_id": {"type": "keyword"},
        "doc_id": {"type": "keyword"},
        "source_type": {"type": "keyword"},
        "created_at": {"type": "date"}
    }
}
```

## 6. API Contract

### POST /api/v1/query

```json
{
  "query": "What is the company's parental leave policy?",
  "filters": {
    "tenant_id": "t-123",
    "doc_types": ["HR", "Benefits"],
    "date_range": {"from": "2024-01-01", "to": null},
    "author": null
  },
  "user_id": "u-456",
  "options": {
    "top_k": 5,
    "include_sources": true,
    "temperature": 0.1,
    "max_tokens": 1024
  }
}

// Response
{
  "query_id": "q-789",
  "answer": "The parental leave policy provides 16 weeks of fully paid leave...",
  "citations": [
    {
      "doc_id": "doc-101",
      "title": "Employee Benefits Handbook 2025",
      "chunk": "Parental Leave: All full-time employees...",
      "relevance_score": 0.94,
      "source_uri": "https://sharepoint.company.com/hr/benefits.pdf",
      "page": 12
    },
    {
      "doc_id": "doc-205",
      "title": "US Benefits Summary",
      "chunk": "Parental leave applies to birth, adoption...",
      "relevance_score": 0.87,
      "source_uri": "https://confluence.company.com/hr/us-benefits"
    }
  ],
  "confidence": 0.92,
  "latency_ms": 680,
  "retrieval_method": "hybrid_dense_sparse"
}
```

### POST /api/v1/documents/ingest

```json
// Request
{
  "tenant_id": "t-123",
  "docs": [
    {
      "source_type": "s3",
      "source_uri": "s3://company-docs/hr/policy.pdf",
      "title": "HR Policy 2025",
      "metadata": {"author": "HR Team", "department": "HR"}
    }
  ],
  "chunk_config": {
    "strategy": "recursive",
    "chunk_size": 512,
    "chunk_overlap": 128
  }
}

// Response
{
  "job_id": "job-456",
  "status": "accepted",
  "docs": [
    {"doc_id": "doc-101", "status": "queued"}
  ],
  "estimated_completion_s": 30
}
```

### GET /api/v1/feedback?query_id=q-789

```json
// Submit feedback
POST /api/v1/feedback
{
  "query_id": "q-789",
  "rating": 4,
  "selected_sources": ["chunk-uuid-1", "chunk-uuid-2"],
  "comment": "Helpful but missed the adoption leave detail"
}
```

## 7. Sequence Diagram

```
Admin    IngestionSvc   Parser    Embedder   VectorDB   QuerySvc   LLM
 │           │            │          │          │          │        │
 ├─Upload────►            │          │          │          │        │
 │           ├─Parse──────►          │          │          │        │
 │           │            ├─Chunks──►          │          │        │
 │           │            │          ├─Embed───►│          │        │
 │           │            │          │          │          │        │
 │           │            │          │          │          │        │
User         │            │          │          │          │        │
 ├─Query─────────────────────────────────────────►          │        │
 │           │            │          │          ├─Search───►│        │
 │           │            │          │          │          ├─Rerank│
 │           │            │          │          │          ├─Prompt┤
 │           │            │          │          │          ├─Gen───►
 │◄─Response─┤            │          │          │          │        │
```

## 8. Scaling Strategy

| Stage | Bottleneck | Solution |
|-------|-----------|----------|
| Ingestion | PDF parsing | Parallel workers (SQS + Lambda), Unstructured.io distributed |
| Embedding | API rate limits | Batch embedding (512 docs/batch), async with retry + backoff |
| Vector Index | ANN build time | Incremental index with HNSW, nightly full rebuild for merge |
| Query peak | LLM generation | Response caching (TTL by doc freshness), semantic cache (threshold 0.95) |
| Tenant isolation | Cross-tenant noise | Per-tenant namespaces in vector DB, dedicated ES index shards |
| Reranker | Cross-encoder cost | 2-stage: first pass (bi-encoder top-50), second pass (cross-encoder top-10) |

## 9. Failure Handling

| Scenario | Mitigation |
|----------|-----------|
| Embedding API fails | Fall back to BM25-only retrieval |
| Vector DB timeout | Degrade to Elasticsearch sparse search |
| LLM returns hallucinated citation | Reject; return "Answer unavailable" with top-3 sources |
| Document parse failure | Dead letter queue + retry with simpler parser |
| Stale index | Change detection webhook; async reindex within 2 min |
| Cross-tenant data leak | Every query includes tenant_id filter; test in CI |
| Token limit exceeded | Truncate chunks to fit context window; mention truncation in response |

## 10. Monitoring & Observability

| Metric | Purpose | Target |
|--------|---------|--------|
| recall@5 | Retrieval quality | > 90% |
| mrr@10 | Ranking quality | > 0.85 |
| answer_acceptance | User satisfaction (thumbs up) | > 80% |
| hallucination_rate | LLM factual accuracy | < 2% |
| index_freshness | Lag from doc change to index | < 2 min |
| p99_query_latency | User experience | < 1 s |
| ingestion_throughput | Documents indexed/minute | > 500/min |
| cost_per_query | Infrastructure efficiency | < $0.002 |

## 11. Security

- **Document-level access control**: Enforced at retrieval — filter chunks by user's RBAC groups
- **Tenant isolation**: Separate vector index namespaces, separate ES indices, separate DB schemas
- **Encryption**: AES-256 at rest (S3 + RDS), TLS 1.3 in transit
- **PII redaction**: Presidio + custom regex before indexing; also redact LLM output
- **Audit trail**: Every query logged with user_id, tenant_id, sources returned
- **Prompt injection**: Input guardrails; output guardrails block prompt leakage
- **Data retention**: Configurable per tenant (default 180 days); auto-delete via lifecycle policy

## 12. Trade-offs

| Decision | Alternative | Rationale |
|----------|-----------|-----------|
| Hybrid search (dense+sparse) | Dense-only | 8-12% recall improvement justifies 2x index storage |
| Per-tenant namespaces | Shared index | Stronger isolation but 15% more storage |
| Async ingestion pipeline | Sync | Sync would block on large PDFs; async adds complexity |
| Cross-encoder reranker | Cohere Rerank | Self-hosted avoids egress costs; 5ms vs 50ms latency |
| Recursive chunking | Semantic chunking | Simpler, proven; semantic better for well-structured docs |
| PostgreSQL + Vector DB | All-in-one (pgvector) | pgvector slower for 500M+ vectors; separate DBs scale independently |

## 13. Interview Tips

- **Clarify scope first**: "Are we building for internal knowledge base or customer-facing FAQ?" Different latency SLAs
- **Chunking is a make-or-break**: Explain chunk size trade-offs — small chunks miss context, large chunks dilute relevance. Mention "small-to-big" retrieval (retrieve small chunks, return parent chunks)
- **Hybrid search**: Be ready to explain why dense alone fails (new terms, rare names) and how RRF fusion works
- **Freshness vs latency**: Indexing pipeline async, but mention "streaming ingestion" for low-latency requirements
- **Cost estimation**: 10M docs = ~3 TB vector DB ≈ $30K/month for Pinecone; embedding API calls ≈ $10K/month
- **Evaluation is hard**: Mention RAGAS framework (faithfulness, answer relevancy, context precision)
- **Common question variants**: "Design a RAG for legal documents", "Design internal search for a 50K employee company", "Design a research assistant"
