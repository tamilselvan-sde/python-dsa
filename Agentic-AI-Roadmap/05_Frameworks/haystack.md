# Haystack Framework

## 1. What is it?

Haystack is an open-source NLP framework created by deepset (Berlin-based AI company) for building production-ready search systems, question-answering pipelines, and RAG applications. First released in 2020, Haystack predates the LLM boom and has evolved from a search-centric framework to a full-featured LLM orchestration platform. It provides modular "pipeline" components that can be connected into DAGs for document processing, retrieval, and generation.

## 2. Why do we need it?

Building production search and QA systems requires composing many components: document stores, retrievers, readers, rerankers, and generators. Haystack solves:
- **Component modularity:** Pre-built nodes for every stage of the NLP pipeline
- **Document store abstraction:** Unified API over Elasticsearch, Chroma, Pinecone, Qdrant, Weaviate, and in-memory
- **Hybrid search:** Combines dense (vector) and sparse (BM25) retrieval out of the box
- **Pipeline evaluation:** Built-in evaluation metrics for retrieval and generation
- **Production focus:** REST API generation, async support, tracing, and monitoring

## 3. Real-world Example

**Enterprise document search:** A legal firm ingests 100,000 contracts into Haystack. A lawyer asks "Find all contracts with non-compete clauses that expire in 2025." Haystack's pipeline: Preprocess PDFs → Extract text → Split into passages → Embed → Index into Elasticsearch → Hybrid search (BM25 + dense) → Rerank with cross-encoder → Generate answer with citations → Return results.

## 4. Architecture Diagram (ASCII)

```
                +-----------------------------+
                |       Haystack Pipeline     |
                +-----------------------------+
                |                             |
    +-----------v-----+   +-----------------+ |
    |  Document Store  |   |  Pipeline       | |
    |  (Elasticsearch) |   |  DAG of Nodes   | |
    +------------------+   +-------+---------+ |
                                  |           |
    +------------------+   +------v--------+ |
    |  Embedding       |   |  Retriever    | |
    |  Model           |   |  (dense/sparse)| |
    +------------------+   +-------+-------+ |
                                  |           |
    +------------------+   +------v--------+ |
    |  Reader /        |   |  Reranker     | |
    |  Generator (LLM)  |   |  (cross-enc.)| |
    +------------------+   +--------------+ |
                                  |           |
                           +------v--------+ |
                           |  Output (Docs | |
                           |  / Answer)    | |
                           +--------------+ |
                +-----------------------------+
```

## 5. Internal Working

Haystack's core is the **Pipeline** — a directed acyclic graph (DAG) of processing nodes. Each node is a component with typed inputs and outputs. Execution:

1. **Pipeline initialisation:** Nodes are added with connection edges. Haystack validates the DAG (no cycles, types match).
2. **Data flow:** Input data enters the pipeline root. Each node transforms data and passes it to connected nodes.
3. **Parallelism:** Nodes without dependencies run concurrently.
4. **Output:** Final node(s) produce the pipeline result.

Key node types:
- **Document Store:** Where documents are stored and retrieved
- **Retriever:** Fetches relevant documents (BM25, embedding similarity, hybrid)
- **Reader/Generator:** Extracts answer or generates response from retrieved docs
- **Preprocessor:** Cleans, splits, and embeds documents
- **Reranker:** Reorders retrieved documents using a cross-encoder

## 6. Production Flow

```
Documents → PreProcessor → split → embed → write to DocumentStore
                                                      |
User Query → Retriever → fetch top-k → Reranker → re-rank →
Generator (LLM) → produce answer with citations
```

## 7. HLD

```
                   +--------------------------+
                   |   API Gateway            |
                   +------------+-------------+
                                |
                   +------------+-------------+
                   |   Haystack Query Service |
                   |   (REST / gRPC)         |
                   +------------+-------------+
                                |
              +-----------------+------------------+
              |                 |                  |
    +---------v----+   +-------v------+   +-------v------+
    | Retriever    |   |  Reader /   |   |  Preprocessor|
    | (embedding   |   |  Generator  |   |  (ingestion  |
    |  + keyword)  |   |  (LLM)      |   |   pipeline)  |
    +------+-------+   +------+------+   +------+-------+
           |                  |                  |
    +------v-------+   +------v------+          |
    | Vector Store |   | Embedding   |          |
    | (Pinecone)   |   | Service     |          |
    +--------------+   +-------------+          |
                                        +-------v-------+
                                        | Document Store |
                                        | (Elasticsearch)|
                                        +---------------+
```

## 8. LLD

```
Pipeline
  ├── add_component(name, instance)
  ├── connect(from, to)
  ├── run(data)
  └── draw(path)  # visualise pipeline

DocumentStore
  ├── write_documents(docs)
  ├── delete_documents(filters)
  └── count_documents()

Retriever
  ├── EmbeddingRetriever
  ├── BM25Retriever
  └── MultiFieldRetriever

Reader
  ├── ExtractiveReader
  └── PromptTemplate

Generator
  ├── OpenAIGenerator
  ├── HuggingFaceGenerator
  └── AzureGenerator

PreProcessor
  ├── clean_empty_lines, clean_whitespace
  ├── split_by_sentence, split_by_word
  └── add_embeddings

Reranker
  ├── SentenceTransformersReranker
  └── CohereReranker

Evaluation
  ├── eval_retrieval(recall, mrr)
  └── eval_generation(semantic_answer_similarity)
```

## 9. Python Implementation

```python
from haystack import Pipeline, Document
from haystack.components.retrievers import InMemoryBM25Retriever
from haystack.components.embedding_retrieval import EmbeddingRetriever
from haystack.components.generators import OpenAIGenerator
from haystack.components.builders import PromptBuilder
from haystack.document_stores.in_memory import InMemoryDocumentStore
from haystack.components.preprocessors import DocumentSplitter
from haystack.components.writers import DocumentWriter
from haystack_integrations.components.embedders.openai import (
    OpenAIDocumentEmbedder, OpenAITextEmbedder
)

# Document store
document_store = InMemoryDocumentStore()

# Ingestion pipeline
ingestion = Pipeline()
ingestion.add_component("splitter", DocumentSplitter(split_by="sentence", split_length=10))
ingestion.add_component("embedder", OpenAIDocumentEmbedder())
ingestion.add_component("writer", DocumentWriter(document_store))
ingestion.connect("splitter", "embedder")
ingestion.connect("embedder", "writer")

docs = [Document(content="Haystack is an NLP framework...")]
ingestion.run({"splitter": {"documents": docs}})

# Query pipeline
query_pipeline = Pipeline()
query_pipeline.add_component("text_embedder", OpenAITextEmbedder())
query_pipeline.add_component("retriever", EmbeddingRetriever(document_store=document_store))
query_pipeline.add_component("prompt_builder", PromptBuilder(
    template="Answer based on context:\nContext: {{documents}}\nQuestion: {{question}}"
))
query_pipeline.add_component("llm", OpenAIGenerator(model="gpt-4o-mini"))
query_pipeline.connect("text_embedder.embedding", "retriever.query_embedding")
query_pipeline.connect("retriever", "prompt_builder.documents")
query_pipeline.connect("prompt_builder", "llm")

result = query_pipeline.run({
    "text_embedder": {"text": "What is Haystack?"},
    "prompt_builder": {"question": "What is Haystack?"}
})
print(result["llm"]["replies"][0])

# Export pipeline visualization
query_pipeline.draw("haystack_pipeline.png")
```

## 10. Folder Structure

```
haystack-app/
├── pipelines/
│   ├── __init__.py
│   ├── ingestion_pipeline.py
│   ├── query_pipeline.py
│   └── hybrid_pipeline.py
├── components/
│   ├── __init__.py
│   ├── custom_retriever.py
│   └── custom_reranker.py
├── stores/
│   └── document_store.py
├── api/
│   └── main.py              # FastAPI with Haystack
├── config.yaml
├── docker-compose.yml
├── tests/
└── pyproject.toml
```

## 11. Configuration

```yaml
# config.yaml
document_store:
  type: elasticsearch
  hosts: ["http://localhost:9200"]
  index: haystack_docs

retrieval:
  retriever: embedding
  embedding_model: text-embedding-3-small
  top_k: 10

generator:
  provider: openai
  model: gpt-4o-mini
  temperature: 0
  max_tokens: 512

preprocessing:
  split_by: sentence
  split_length: 10
  split_overlap: 2
```

## 12. Flowchart

```
                     Ingestion Pipeline
    ┌─────────┐   ┌─────────┐   ┌─────────┐   ┌─────────┐
    │  File   │──▶│  Split  │──▶│  Embed  │──▶│  Write  │
    │  Load   │   │         │   │         │   │         │
    └─────────┘   └─────────┘   └─────────┘   └────┬────┘
                                                    │
                                                    v
                                              ┌──────┴──────┐
                                              │  Document   │
                                              │  Store      │
                                              └──────┬──────┘
                                                     │
                     Query Pipeline                   │
    ┌─────────┐   ┌─────────┐   ┌─────────┐          │
    │  Query  │──▶│  Embed  │──▶│Retrieve │──────────┘
    └─────────┘   └─────────┘   └────┬────┘
                                      │
                               ┌──────v──────┐
                               │  Prompt     │
                               │  Builder    │
                               └──────┬──────┘
                                      │
                               ┌──────v──────┐
                               │  Generator  │
                               │  (LLM)      │
                               └──────┬──────┘
                                      │
                               ┌──────v──────┐
                               │   Answer    │
                               └─────────────┘
```

## 13. Sequence Diagram

```
User       Query Pipeline    Retriever      Generator    DocStore
 |               |               |              |           |
 |--query()----->|               |              |           |
 |               |--retrieve---->|              |           |
 |               |               |--search----->|           |
 |               |               |<--docs-------|           |
 |               |<--documents---|              |           |
 |               |--generate------------------>|           |
 |               |<--------answer---------------|           |
 |<--response----|               |              |           |
```

## 14. Pros

- **Production-proven** — 5+ years of development, used in enterprise deployments
- **Clean pipeline abstraction** — DAG pipelines are intuitive and debuggable
- **Multiple retrieval strategies:** BM25, dense, hybrid, multi-field
- **Excellent evaluation tools** — built-in metrics for retrieval and generation
- **REST API generation** — `haystack-api` CLI auto-generates API from pipeline
- **Enterprise focus:** deepset Cloud for managed hosting

## 15. Cons

- **LLM era adaptation** — originally a search framework, LLM integration feels bolted on
- **Smaller community** — less adoption than LangChain or LlamaIndex
- **Limited agent support** — no native agent loop or tool-calling (focused on search pipelines)
- **Documentation gaps** — some components lack worked examples
- **Version migration** — Haystack 2.0 (2024) was a complete rewrite, breaking all 1.x pipelines

## 16. Alternatives

| Framework | When to Choose |
|-----------|---------------|
| LangChain | General LLM orchestration, larger ecosystem |
| LlamaIndex | Data-centric RAG, better for document-heavy workloads |
| Elasticsearch alone | If you only need keyword search, no LLM |
| Weaviate | Vector store with built-in RAG |
| Cohere | Managed RAG platform |

## 17. Performance Considerations

- **Pipeline parallelism:** Haystack runs independent nodes concurrently. Use this for parallel retrieval + generation.
- **Embedding throughput:** Batch embeddings (haystack supports batching natively). Use GPU-accelerated embedding servers for scale.
- **Document store selection:** InMemory is fast (<1ms) but not persistent. Elasticsearch adds 5-20ms but scales to billions.
- **Reranker latency:** Cross-encoders are 10-100x slower than bi-encoders. Rerank only top-20, not top-100.
- **Preprocessing bottleneck:** Heavy preprocessing (OCR, complex splitting) should be a separate async job.
- **Streaming:** Use `streaming_callback` on generators for token-by-token responses.

## 18. Scaling to Millions

1. **Document store sharding:** Elasticsearch shards across nodes. Each shard holds a subset of documents. Search fans out to all shards.
2. **Embedding service:** Dedicated embedding service (e.g., `text-embedding-inference` server) with GPU and horizontal scaling.
3. **Async ingestion:** Separate ingestion pipeline runs as a batch job (Spark or Kubernetes Job). Writes to the same document store.
4. **Query side:** Stateless query pipeline pods behind a load balancer. Caching layer (Redis) for frequently asked queries.
5. **Index rollover:** Time-based index rollover for logs/documents. Hot (SSD), warm (HDD), cold (S3) tiered storage.
6. **deepset Cloud:** Managed hosting with autoscaling, monitoring, and zero-config deployment.

## 19. Failure Scenarios

1. **Document store down:** Haystack allows fallback to a secondary store. Implement health checks before each query.
2. **Embedding model failure:** Degraded to BM25-only retrieval (keyword search). Monitor embedding service uptime.
3. **LLM API error:** Retry with exponential backoff. If all retries fail, return raw retrieved documents instead of generated answer.
4. **Pipeline type mismatch:** Haystack validates types at connection time. For runtime errors, check input/output types match.
5. **Index corruption:** Regular Elasticsearch snapshots. Point-in-time recovery to last good snapshot.

## 20. Security

- **Document-level access control:** Apply filters based on user role/permissions at query time. Haystack's filters support metadata-based access control.
- **PII redaction:** Add a custom component in the preprocessing pipeline to redact PII (regex-based or using Presidio).
- **API security:** Haystack REST API supports JWT authentication. Rate limiting per API key.
- **LLM prompt injection:** Use Haystack's `PromptBuilder` with strict templates. Never interpolate raw user input into system prompts.
- **Audit logging:** Log every query, retrieved documents, and generated response. Haystack's `tracing` integration captures pipeline runs.

## 21. Monitoring

- **OpenTelemetry tracing:** Haystack 2.0 has native OpenTelemetry support. Each pipeline run is traced with spans for each component.
- **deepset Cloud dashboard:** Built-in monitoring for managed deployments.
- **S3 + Grafana:** Store pipeline run logs in S3. Use Grafana to visualise latency, throughput, and error rates.
- **Metrics:** Track `pipeline_run_duration_ms`, `retrieval_recall@k`, `generator_latency`, `document_store_query_time`.

## 22. Interview Questions

1. How does Haystack's pipeline differ from LangChain's chain? What are the trade-offs?
2. Explain the difference between `EmbeddingRetriever` and `BM25Retriever`. When would you use each?
3. How does Haystack implement hybrid search (dense + sparse)?
4. Design a pipeline that extracts answers from PDFs and generates summaries.
5. How would you evaluate a Haystack pipeline? What metrics matter?
6. Explain how Haystack 2.0's component system differs from Haystack 1.x's nodes.
7. How would you implement a custom component in Haystack?
8. Compare Haystack's approach to document preprocessing with LlamaIndex.

## 23. Cheat Sheet

| Task | Code |
|------|------|
| Create pipeline | `Pipeline()` |
| Add component | `pipeline.add_component("name", ComponentClass())` |
| Connect | `pipeline.connect("comp1.output", "comp2.input")` |
| Run | `pipeline.run({"comp1": {"input": data}})` |
| Visualise | `pipeline.draw("pipeline.png")` |
| Create document store | `InMemoryDocumentStore()` |
| Write documents | `document_store.write_documents([Document(...)])` |
| Custom component | `@component` decorator on class with `@component.output_types` |

## History

Haystack was created by deepset in 2020, originally as a framework for search and open-domain QA. The first version focused on extractive QA using BERT-based models. In 2021, Haystack added generative QA support. 2022 brought dense retrieval (DPR, EmbeddingRetriever), hybrid search, and REST API generation. 2023 saw the addition of LLM generators (OpenAI, Cohere) and custom components. Haystack 2.0, released in February 2024, was a ground-up rewrite with a new component system, typed pipeline connections, OpenTelemetry tracing, and a cleaner API. deepset Cloud launched as the managed platform. As of 2025, Haystack is a mature, enterprise-grade framework focused on production search and RAG.

## Core Components Explained

- **Pipeline:** The core orchestration primitive — a DAG of components. Data flows through `connect()` edges. Supports branching, merging, and parallelism.
- **Component:** A base class or `@component` decorated class with `run()` method. Components have typed inputs and outputs defined with `@component.output_types`.
- **Document:** The standard data unit — contains `content` (text), `meta` (metadata dict), `embedding` (vector), and `id`.
- **Retriever:** Fetches documents from a store. `EmbeddingRetriever` (dense), `BM25Retriever` (keyword), `MultiFieldRetriever` (combine).
- **Generator:** Produces text from an LLM. `OpenAIGenerator`, `HuggingFaceLocalGenerator`, `AzureGenerator`.
- **Reader:** Extracts answers from documents. `ExtractiveReader` (span extraction), `PromptTemplate` (generation).
- **PreProcessor:** Cleans and prepares documents. Splitting, cleaning, embedding addition.
- **Reranker:** Reorders retrieved documents using a cross-encoder for better relevance.

## Installation

```bash
pip install haystack-ai
# Integrations
pip install haystack-elasticsearch
pip install haystack-pinecone
pip install haystack-cohere
# Optional
pip install haystack-core-integrations
```

## Advanced Features

- **Async pipelines:** All pipeline components support `async run()` for non-blocking execution.
- **Tracing:** `haystack.tracing` with OpenTelemetry — spans for every component invocation.
- **Ranking:** `HuggingFaceRanker`, `CohereRanker` for reranking.
- **Evaluation:** `eval_retrieval()` (recall, MRR, MAP) and `eval_generation()` (SAS, BLEU, ROUGE).
- **Custom components:** Decorate any Python class with `@component`. Define inputs/outputs with `@component.output_types`.
- **Agent support:** Haystack 2.2+ includes `Tool` and basic agent capabilities (experimental).

## Performance Tuning

1. **Batching:** Set `batch_size` on `OpenAIDocumentEmbedder` (default 32). Higher = faster but consumes more memory.
2. **Parallel writes:** Use `Pool`-based parallelism in `DocumentWriter` for high-throughput ingestion.
3. **Indexing every N docs:** Flush the document store writer every N documents to balance memory and I/O.
4. **Caching:** Haystack provides `cache` parameter on generators. Use Redis-based caching for identical queries.
5. **Pipeline warm-up:** Run a dummy query to warm model caches during deployment startup.

## Debugging Tips

1. `pipeline.run(data, include_outputs_from=["component_name"])` returns intermediate outputs.
2. `haystack.set_pipeline_debug(True)` enables verbose logging of every component.
3. Use `haystack.utils.printable_memory` to monitor memory usage during ingestion.
4. Common errors: `PipelineConnectError` (type mismatch), `ComponentError` (component raised exception).

## Production Deployment

1. **Containerise:** Docker image with Haystack application. Use `gunicorn` with `fastapi-slim` for REST API.
2. **deepset Cloud:** Managed hosting with autoscaling. Integrates with Haystack's native REST API.
3. **Self-hosted:** Deploy behind Nginx reverse proxy. Elasticsearch cluster for document store. GPU node for embeddings.
4. **Scaling:** Separate ingestion (batch jobs) from query (real-time API). Scale query pods based on request queue depth.
5. **Backup:** Regular Elasticsearch snapshots to S3. Haystack index settings with replica shards for high availability.
