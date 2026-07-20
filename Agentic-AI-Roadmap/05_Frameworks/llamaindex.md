# LlamaIndex Framework

> Author: **Tamilselvan** · ✉️ tamilselvan.sde@gmail.com · 🔗 [LinkedIn](https://www.linkedin.com/in/tamilselvan-ai/)
>

## 1. What is it?

LlamaIndex (formerly GPT Index) is an open-source data framework for LLM applications, released in November 2022 by Jerry Liu. It specialises in connecting LLMs to external data through a comprehensive ingestion, indexing, and retrieval pipeline. LlamaIndex provides data connectors for 150+ sources (PDFs, databases, APIs, Slack, Confluence), advanced chunking strategies, hybrid search, and query engines that synthesise answers from retrieved context. It is the leading framework for production Retrieval-Augmented Generation (RAG) systems.

## 2. Why do we need it?

LLMs have knowledge cutoffs and no access to private data. Naively stuffing documents into prompts hits token limits and degrades quality. LlamaIndex solves:
- **Data ingestion:** Loaders for PDF, HTML, Notion, Confluence, Google Docs, SQL, and 150+ other formats
- **Intelligent chunking:** Sentence-aware splitting with overlap, preserving metadata lineage
- **Structured indexing:** Multiple index types (vector, keyword, tree, knowledge graph, document summary)
- **Advanced retrieval:** Hybrid (vector + keyword), auto-merged, recursive, and agentic retrieval
- **Query synthesis:** Multi-hop queries, sub-question decomposition, structured analysis

## 3. Real-world Example

**Enterprise knowledge base assistant:** A Fortune 500 company ingests 50,000 documents from Confluence, SharePoint, and PDFs into a LlamaIndex pipeline. When a new hire asks, "What is our expense reimbursement policy for international travel?", the system retrieves the most relevant chunks across multiple sources, reranks them, and synthesises a concise answer with inline citations to the original documents.

## 4. Architecture Diagram (ASCII)

```
                +---------------------+
                |   Data Connectors   |
                | (Slack, Notion, PDF)|
                +---------+-----------+
                          |
                +---------v-----------+
                |  Document Loaders   |
                | (pypdf, Unstructured)|
                +---------+-----------+
                          |
                +---------v-----------+
                |  Node Parsers       |
                | (SentenceSplitter,  |
                |  TokenTextSplitter) |
                +---------+-----------+
                          |
                +---------v-----------+
                |  Embedding Model    |
                | (OpenAI, HuggingFace)|
                +---------+-----------+
                          |
                +---------v-----------+
                |  Vector/Keyword     |
                |  Index              |
                +---------+-----------+
                          |
                +---------v-----------+
                |  Retriever +        |
                |  Reranker           |
                +---------+-----------+
                          |
                +---------v-----------+
                |  Query Engine       |
                |  (Synthesizer)      |
                +---------+-----------+
                          |
                +---------v-----------+
                |   Final Answer      |
                +---------------------+
```

## 5. Internal Working

LlamaIndex centres on three stages:

1. **Ingestion:** Raw documents are loaded via `BaseReader` subclasses. Each document is split into `Node` objects (chunks with metadata) via `NodeParser`. Nodes pass through optional transformations (e.g., embedding, keyword extraction).
2. **Indexing:** Nodes are indexed into an `BaseIndex`. `VectorStoreIndex` embeds nodes into a vector DB. `SummaryIndex` stores nodes sequentially. `KeywordTableIndex` extracts keywords. `KnowledgeGraphIndex` extracts entities and relationships.
3. **Querying:** A `QueryEngine` receives a user query. The engine retrieves relevant nodes via the index's `Retriever`, optionally reranks them via `NodePostprocessor`, and passes them to a `Synthesizer` that calls an LLM to produce the final answer.

## 6. Production Flow

```
Documents > Load > Parse > Chunk > Embed > Index > Store
                                            |
User Query > Embed > Vector Search > Retrieve Top-K > Rerank
> Compress Context > LLM Synthesis > Answer with Citations
```

## 7. HLD

```
                   +---------------------+
                   |   API Gateway       |
                   +---------+-----------+
                             |
            +----------------+----------------+
            |                |                 |
    +-------v----+   +------v------+  +-------v----+
    | Query API  |   | Ingestion   |  | Management |
    | Service    |   | Pipeline    |  |  Console   |
    +------------+   +------+------+  +------------+
                            |
            +---------------+---------------+
            |               |               |
    +-------v----+   +------v------+  +-------v----+
    | Embedding   |   |   Vector   |  |   Metadata |
    | Service     |   |   Store    |  |   Store    |
    +------------+   +-------------+  +------------+
```

## 8. LLD

```
BaseReader
  ├── SimpleDirectoryReader
  ├── PDFReader
  ├── NotionPageReader
  └── SlackReader

NodeParser
  ├── SentenceSplitter
  ├── TokenTextSplitter
  ├── SemanticSplitterNodeParser
  └── HierarchicalNodeParser

BaseIndex
  ├── VectorStoreIndex
  ├── SummaryIndex
  ├── KeywordTableIndex
  └── KnowledgeGraphIndex

BaseRetriever
  ├── VectorIndexRetriever
  ├── KeywordTableRetriever
  ├── BM25Retriever
  └── AutoMergingRetriever

BaseNodePostprocessor
  ├── SentenceTransformerRerank
  ├── CohereRerank
  ├── SimilarityPostprocessor
  └── KeywordNodePostprocessor

QueryEngine
  ├── RetrieverQueryEngine
  ├── SubQuestionQueryEngine
  └── RouterQueryEngine
```

## 9. Python Implementation

```python
from llama_index.core import (
    VectorStoreIndex, SimpleDirectoryReader, StorageContext,
    Settings, load_index_from_storage
)
from llama_index.core.node_parser import SentenceSplitter
from llama_index.core.postprocessor import SentenceTransformerRerank
from llama_index.vector_stores.chroma import ChromaVectorStore
from llama_index.embeddings.openai import OpenAIEmbedding
from llama_index.llms.openai import OpenAI
import chromadb

# Settings
Settings.embed_model = OpenAIEmbedding(model="text-embedding-3-small")
Settings.llm = OpenAI(model="gpt-4o-mini", temperature=0)
Settings.chunk_size = 512
Settings.chunk_overlap = 128

# Ingestion
documents = SimpleDirectoryReader("./data").load_data()
parser = SentenceSplitter(chunk_size=512, chunk_overlap=128)
nodes = parser.get_nodes_from_documents(documents)

# Indexing
chroma_client = chromadb.PersistentClient(path="./chroma_db")
chroma_collection = chroma_client.get_or_create_collection("company_docs")
vector_store = ChromaVectorStore(chroma_collection=chroma_collection)
storage_context = StorageContext.from_defaults(vector_store=vector_store)
index = VectorStoreIndex(nodes, storage_context=storage_context)

# Query
reranker = SentenceTransformerRerank(
    model="cross-encoder/ms-marco-MiniLM-L-2-v2",
    top_k=3
)
query_engine = index.as_query_engine(
    similarity_top_k=10,
    node_postprocessors=[reranker],
    response_mode="tree_summarize"
)
response = query_engine.query("What is the expense reimbursement policy?")
print(response)

# Source tracking
for node in response.source_nodes:
    print(f"Source: {node.node.metadata['file_path']}, Score: {node.score}")
```

## 10. Folder Structure

```
llamaindex-app/
├── ingestion/
│   ├── __init__.py
│   ├── pipeline.py
│   └── readers.py
├── index/
│   ├── __init__.py
│   └── build_index.py
├── query/
│   ├── __init__.py
│   ├── engine.py
│   ├── router.py
│   └── postprocessors.py
├── api/
│   └── main.py
├── tests/
├── data/              # Source documents
├── storage/           # Persisted index
├── config.yaml
├── docker-compose.yml
└── pyproject.toml
```

## 11. Configuration

```yaml
# config.yaml
embedding:
  model: text-embedding-3-small
  dimensions: 1536
  batch_size: 100

chunking:
  size: 512
  overlap: 128
  parser: sentence

vector_store:
  type: chroma
  path: ./chroma_db
  collection_name: company_docs

retrieval:
  similarity_top_k: 10
  rerank_top_k: 3
  rerank_model: cross-encoder/ms-marco-MiniLM-L-2-v2

llm:
  model: gpt-4o-mini
  temperature: 0
  max_tokens: 1024
```

## 12. Flowchart

```
           Documents
              |
              v
     SimpleDirectoryReader
              |
              v
     SentenceSplitter
     (chunk_size=512)
              |
        +-----+-----+
        |           |
        v           v
    Embedding   Store Metadata
        |           |
        +-----+-----+
              |
              v
    ┌─────────────────┐
    |  VectorStoreIndex|
    └────────┬────────┘
             |
    User Query
             |
             v
    Embed Query
             |
             v
    Similarity Search (Top-10)
             |
             v
    Reranker (Top-3)
             |
             v
    LLM Synthesis
             |
             v
    Answer + Citations
```

## 13. Sequence Diagram

```
 User       QueryEngine    Retriever     Reranker        LLM
  |             |             |             |             |
  |--query------>|             |             |             |
  |             |--embed------>|             |             |
  |             |              |--search---->|             |
  |             |<--top-10-----|             |             |
  |             |--rerank------|------------>|             |
  |             |<--reranked---|-------------|             |
  |             |--synthesize-|-------------------------->|
  |             |<-----------response---------------------|
  |<--answer----|             |             |             |
```

## 14. Pros

- **Best-in-class RAG** — purpose-built for retrieval, not an afterthought
- **150+ data connectors** via LlamaHub, the largest open-source connector registry
- **Advanced retrieval strategies:** auto-merging, recursive, sub-question, agentic
- **LlamaCloud:** managed ingestion, indexing, and query hosting
- **Modular design:** swap retrievers, postprocessors, and synthesizers independently
- **Pydantic integration:** structured output via `PydanticOutputParser`

## 15. Cons

- **Overkill for simple use cases** — basic Q&A does not need the full ingestion pipeline
- **Learning curve** — many index types and retrieval strategies can overwhelm beginners
- **Index rebuild time** — re-indexing large document sets can take hours without incremental indexing
- **Documentation gaps** — some advanced features lack worked examples
- **Memory overhead** — loading many documents into memory for chunking can exhaust resources

## 16. Alternatives

| Framework | When to Choose |
|-----------|---------------|
| LangChain | General LLM orchestration with optional RAG |
| Haystack | Production NLP pipelines (PDF, audio, images) |
| Unstructured | Document parsing only (no RAG orchestration) |
| Chroma | Vector store only (pair with other frameworks) |
| Weaviate | Vector store with built-in RAG module |

## 17. Performance Considerations

- **Chunk size trade-off:** 256 tokens = higher precision, lower recall; 1024 tokens = more context, higher token cost. Start with 512, tune on your data.
- **Embedding throughput:** Use batch embedding (`batch_size=100`). `text-embedding-3-small` is 8x faster than `-large` with negligible quality loss.
- **Reranking impact:** Reranking adds ~50ms latency but improves precision by 10-20%. Cross-encoder models (ms-marco) outperform bi-encoders.
- **Vector store selection:**
  - Chroma: Good for dev, up to 10M vectors
  - Pinecone: Best for >10M vectors, managed
  - Qdrant: Self-hosted, high performance
- **Metadata filtering:** Apply filters before similarity search to reduce candidate set size.

## 18. Scaling to Millions

1. **Incremental indexing:** Use `DocStoreStrategy.DUPLICATES_ONLY` to skip unchanged documents. Track document hashes for change detection.
2. **Distributed ingestion:** Split documents across workers (Kubernetes Jobs). Each worker indexes its batch and merges into the same vector store.
3. **Vector store sharding:** Pinecone pods or Qdrant shards partition data across nodes. Query fans out to all shards and merges results.
4. **Query routing:** Use `RouterQueryEngine` to dispatch queries to specialised indices (e.g., finance vs. engineering docs) based on intent classification.
5. **Caching:** Cache query results with TTL (30s for dynamic data, permanent for static). Use Redis with `CacheBackedEmbeddings`.
6. **Index compression:** For knowledge graph indices, prune low-weight edges. For vector indices, use product quantisation (PQ) to reduce memory.

## 19. Failure Scenarios

1. **No relevant documents:** Fallback to keyword-only search or LLM knowledge. Monitor `zero_retrieval_rate` — if >5%, adjust chunking or embedding model.
2. **Stale index:** Implement webhook-based re-indexing when source documents change. Use document versioning to detect staleness.
3. **Token overflow in synthesis:** Truncate retrieved context to fit LLM's context window. LlamaIndex's `response_mode="compact"` squeezes more context.
4. **Embedding model API outage:** Fallback to `BM25Retriever` (keyword search) until embedding service recovers.
5. **Chunk boundary breaks:** When an answer spans chunks, use `AutoMergingRetriever` to merge hierarchical chunks into coherent passages.

## 20. Security

- **Document-level access control:** Store document metadata (owner, group, ACL) in the vector store. Filter by permissions at retrieval time.
- **PII redaction:** Scan documents during ingestion with Presidio. Redact PII before embedding.
- **Query sanitisation:** Strip SQL injection, prompt injection from user queries before routing to synthesis LLM.
- **Encryption at rest:** Encrypt vector store data at rest (AES-256). Encrypt document cache in Redis.
- **Audit logging:** Log every query, retrieved document IDs, and the synthesized answer for compliance.

## 21. Monitoring

- **Key metrics:**
  - Query latency (p50/p95/p99)
  - Retrieval precision@k (measured against golden dataset)
  - Zero retrieval rate (% of queries with no relevant chunks)
  - Ingestion throughput (docs/minute)
  - Token usage per query (retrieval + synthesis)
- **LlamaCloud dashboard:** Built-in monitoring for hosted customers.
- **Custom logging:** Integrate with OpenTelemetry. Log structured events: `{event: "retrieval", query_id, chunks_retrieved, latency_ms}`.

## 22. Interview Questions

1. How do LlamaIndex's index types differ (VectorStoreIndex vs SummaryIndex vs KeywordTableIndex)?
2. Explain the auto-merging retrieval strategy and when you would use it.
3. How does SentenceSplitter handle chunk boundary awareness differently from TokenTextSplitter?
4. Implement a custom node postprocessor that filters based on a metadata field.
5. Compare RouterQueryEngine vs SubQuestionQueryEngine.
6. How would you implement a hybrid search (vector + BM25) in LlamaIndex?
7. Explain the role of `StorageContext` in managing multiple stores.
8. How does LlamaIndex handle query decomposition for multi-hop questions?

## 23. Cheat Sheet

| Task | Code |
|------|------|
| Quick RAG | `index = VectorStoreIndex.from_documents(docs)` |
| Custom chunking | `parser = SentenceSplitter(chunk_size=512)` |
| With reranker | `index.as_query_engine(node_postprocessors=[reranker])` |
| Streaming | `query_engine.stream(query)` yields chunks |
| Persist | `index.storage_context.persist(persist_dir="./")` |
| Load persisted | `load_index_from_storage(StorageContext.from_defaults(persist_dir="./"))` |
| Add documents | `index.insert(document)` |
| Query with filters | `query_engine.query(query, filters=MetadataFilters(...))` |

## History

LlamaIndex began as GPT Index in November 2022, created by Jerry Liu (a former Uber engineer) after being frustrated with the boilerplate involved in building RAG applications. The repository grew rapidly, reaching 10k GitHub stars within months. In January 2023, it was renamed to LlamaIndex to reflect broader ambitions beyond GPT. Version 0.6 introduced the modular router/retriever/synthesizer architecture. LlamaHub, the community connector registry, launched in April 2023. Version 1.0 (January 2024) stabilised the API. LlamaCloud, the managed SaaS platform, launched in mid-2024. As of 2025, LlamaIndex has over 150 connectors, 500k monthly downloads, and is the leading data framework for LLM applications.

## Core Components Explained

- **Readers** (`BaseReader`): Load documents from various sources. `SimpleDirectoryReader` reads a local directory. Integration readers: `NotionPageReader`, `SlackReader`, `GoogleDocsReader`.
- **Nodes:** The fundamental unit of data — a chunk of text with `metadata` (source, page number, timestamp) and `relationships` (next/prev node, parent node for hierarchy).
- **Indices** (`BaseIndex`): Data structures for efficient retrieval. `VectorStoreIndex` is the most common — embeds nodes and stores in a vector DB. `SummaryIndex` stores nodes as a flat list. `KeywordTableIndex` maps keywords to nodes.
- **Retrievers:** Extract relevant nodes from an index. `VectorIndexRetriever` does embedding similarity. `BM25Retriever` does keyword search. `AutoMergingRetriever` recursively retrieves granular chunks and merges into parent chunks.
- **Postprocessors:** Filter/rerank retrieved nodes. `SimilarityPostprocessor` filters by score. `SentenceTransformerRerank` reranks with a cross-encoder.
- **Query Engines:** The user-facing interface. `RetrieverQueryEngine` is the standard. `SubQuestionQueryEngine` breaks complex questions into sub-questions. `RouterQueryEngine` routes to the best-suited index.
- **Synthesizers:** The LLM call that generates the final answer from retrieved context. Modes: `compact` (concatenate context), `tree_summarize` (recursive summary), `refine` (iterative refinement).

## Installation

```bash
pip install llama-index llama-index-embeddings-openai llama-index-llms-openai
# Vector stores
pip install llama-index-vector-stores-chroma llama-index-vector-stores-pinecone
# Document readers
pip install llama-index-readers-file pypdf unstructured
# LlamaHub CLI
pip install llama-hub
```

## Advanced Features

- **Structured data extraction:** Extract entities, relationships, and summaries from documents during ingestion via `MetadataExtractor`.
- **Agent + RAG combination:** LlamaIndex `AgentRunner` wraps tools (including query engines) in a LangChain-compatible agent loop.
- **Multi-modal RAG:** Index images alongside text. Retrieve images via CLIP embeddings and text via standard embeddings.
- **Graph RAG:** `KnowledgeGraphIndex` extracts entities and relationships, enabling multi-hop relational queries.
- **Structured output:** `PydanticOutputParser` extracts structured data from LLM responses (JSON, Python objects).
- **Async ingestion:** `aiohttp`-based async document loading. `run_jobs` for parallel transformations.

## Performance Tuning

1. **Batch embedding:** Set `embed_batch_size=100` to maximise throughput. Monitor GPU memory if using local models.
2. **Index only what's needed:** Not all documents need embedding. Use `MetadataIndex` for low-importance documents and `VectorIndex` for high-importance.
3. **Chunk overlap:** Set `chunk_overlap = chunk_size * 0.2`. Overlap prevents context being cut mid-sentence but increases storage.
4. **Reranker batch:** Cross-encoder rerankers are slower on many candidates. Limit to top-20 for reranking.
5. **Caching:** `CacheBackedEmbeddings` caches embeddings on disk. First invocation is slow, subsequent invocations are instant.

## Debugging Tips

1. `print(response.get_formatted_sources())` shows which documents contributed to the answer.
2. `query_engine.as_query_engine(verbose=True)` prints retrieved chunks and scores.
3. Inspect node content: `for n in nodes: print(n.get_content(metadata_mode="all"))`.
4. Check embedding dimension mismatch errors (common when switching models).
5. Use `IngestionPipeline(..., disable_cache=True)` during development to avoid stale cache issues.

## Production Deployment

1. **Separate ingestion from query:** Ingestion pipelines run as batch jobs (Kubernetes CronJob). Query services are stateless HTTP servers.
2. **Vector store backup:** Regular snapshots of the vector store. Use Pinecone's built-in backup or Qdrant's snapshot API.
3. **Index versioning:** Tag each index with a version ID. Route queries to the correct version during A/B testing.
4. **Graceful degradation:** On vector store failure, fall back to BM25 keyword search. On embedding failure, fall back to raw LLM.
5. **Monitoring dashboard:** Grafana dashboard with query latency, retrieval metrics, and token usage. PagerDuty alerts on anomaly.
