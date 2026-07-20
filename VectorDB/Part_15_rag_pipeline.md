# Part 15: RAG Pipeline

> Author: **Tamilselvan** · ✉️ tamilselvan.sde@gmail.com · 🔗 [LinkedIn](https://www.linkedin.com/in/tamilselvan-ai/)
>

## What is RAG?

**Retrieval-Augmented Generation (RAG)** is a technique that enhances LLM outputs by first retrieving relevant information from a knowledge base, then feeding it to the LLM as context.

```mermaid
graph TD
    subgraph "Without RAG"
        A[User Question] --> B[LLM]
        B --> C[Answer based on<br/>training data only]
    end
    subgraph "With RAG"
        D[User Question] --> E[Retrieve from<br/>Knowledge Base]
        E --> F[Relevant Documents]
        F --> G[LLM + Context]
        D --> G
        G --> H[Answer based on<br/>retrieved data]
    end
```

---

## Complete RAG Flow

```mermaid
sequenceDiagram
    participant User
    participant App as Application
    participant Chunker
    participant Embedder
    participant VDB as Vector DB
    participant Reranker
    participant LLM
    
    Note over User,LLM: INGESTION PHASE
    User->>App: Upload PDF
    App->>Chunker: Split into chunks
    Chunker->>Embedder: Generate embeddings
    Embedder->>VDB: Store vectors + metadata
    VDB-->>App: Ingestion complete
    
    Note over User,LLM: QUERY PHASE
    User->>App: Ask question
    App->>Embedder: Embed query
    Embedder-->>App: Query vector
    App->>VDB: ANN search (k=20)
    VDB-->>App: Top-20 chunks
    App->>Reranker: Rerank with cross-encoder
    Reranker-->>App: Top-5 chunks
    App->>LLM: Build prompt with context + question
    LLM-->>App: Generated answer
    App-->>User: Answer with citations
```

---

## Enhanced RAG with Metadata Filtering

```mermaid
graph TB
    A[User Query] --> B[Query Analysis]
    B --> C[Generate Embedding]
    B --> D[Extract Filters]
    
    C --> E[Vector Search with Filters]
    D --> E
    
    E --> F[Retrieved Chunks]
    F --> G[Reranker]
    G --> H[LLM Prompt Construction]
    H --> I[LLM]
    I --> J[Answer + Citations]
```

### Python RAG Implementation

```python
import openai
from sentence_transformers import SentenceTransformer
from qdrant_client import QdrantClient

class RAGPipeline:
    def __init__(self):
        self.embedder = SentenceTransformer('all-MiniLM-L6-v2')
        self.db = QdrantClient(":memory:")
        self.llm = openai.OpenAI()
        
    def ingest(self, chunks, metadata_list):
        """Store chunks in vector DB."""
        vectors = self.embedder.encode(chunks)
        self.db.upload_collection(
            collection_name="docs",
            vectors=vectors.tolist(),
            payload=[{"text": c, **m} 
                     for c, m in zip(chunks, metadata_list)]
        )
    
    def query(self, question, k=5):
        """Retrieve and generate answer."""
        # 1. Embed query
        query_vec = self.embedder.encode(question)
        
        # 2. Retrieve from vector DB
        results = self.db.search(
            collection_name="docs",
            query_vector=query_vec,
            limit=k
        )
        
        # 3. Build context
        context = "\n\n".join([
            r.payload["text"] for r in results
        ])
        
        # 4. Generate with LLM
        prompt = f"""Answer the question based on the context below.
        
Context:
{context}

Question: {question}

Answer:"""
        
        response = self.llm.chat.completions.create(
            model="gpt-4",
            messages=[{"role": "user", "content": prompt}]
        )
        
        return response.choices[0].message.content
```

---

## Production RAG Architecture

```mermaid
graph TB
    subgraph "Data Pipeline"
        A[Documents] --> B[Document Processor]
        B --> C[Chunker]
        C --> D[Embedding Service]
        D --> E[(Vector DB)]
    end
    
    subgraph "Query Pipeline"
        F[User] --> G[API Gateway]
        G --> H[Query Analyzer]
        H --> I[Embedding Service]
        I --> J[Vector Search]
        J --> K[Reranker]
        K --> L[Prompt Builder]
        L --> M[LLM]
        M --> N[Response]
    end
    
    subgraph "Infrastructure"
        O[Redis Cache] --> J
        P[Monitoring] --- K
        Q[Rate Limiter] --- G
    end
    
    subgraph "Feedback Loop"
        N --> R[User Feedback]
        R --> S[Fine-tune Reranker]
        S --> K
    end
```

### Advanced RAG Techniques

```mermaid
mindmap
  root((RAG Variants))
    Naive RAG
      Ingest → Retrieve → Generate
    Advanced RAG
      Query Rewriting
      Query Decomposition
      HyDE
      Multi-Query
    Modular RAG
      Routing
      Fusion
      Self-RAG
      Corrective RAG
    Agentic RAG
      Tool Use
      Multi-step Retrieval
      Iterative Refinement
```

### Techniques Summary

| Technique | Description | Improvement |
|-----------|-------------|-------------|
| **HyDE** | Generate hypothetical doc from query, embed that | Better query representation |
| **Multi-Query** | Generate multiple query variants, combine results | Better recall |
| **Query Rewriting** | LLM rewrites ambiguous queries | Better precision |
| **Context Compression** | Condense retrieved chunks | Fit more in context |
| **Self-RAG** | LLM decides when to retrieve | Less hallucination |
| **Corrective RAG** | Verify retrieved docs, retry if needed | Higher quality |
| **Hybrid Search** | Dense + sparse fusion | Better coverage |
| **Agentic RAG** | Multi-tool, multi-step retrieval | Complex reasoning |

---

### ELI5: RAG

> Imagine taking an open-book test:
>
> - **Without RAG (Closed book):** You answer from memory. Might get things wrong or make stuff up.
> - **RAG (Open book):** First you look up the answer in your notes. Then you explain it in your own words.
> - **Good RAG:** You highlight the most relevant parts. You check that you have the right page. Then you explain clearly.
> - **Great RAG:** You skim multiple chapters, cross-reference, check which info is most reliable, then synthesize a comprehensive answer.

---

### Production Tip

> **RAG pipeline latency breakdown (typical):**
> 1. Embedding: 50-200ms (API) or 10-50ms (local)
> 2. Vector search: 5-50ms
> 3. Reranking: 50-500ms (per chunk analyzed)
> 4. LLM generation: 500ms-5s
> **Total: ~1-6 seconds**
>
> Optimize by: caching frequent queries, batching embeddings, using smaller rerankers for first pass.

---

### Common Mistake

> **❌ Not including citations in RAG output.** Users (and evaluators) need to know which source documents support the answer. Always return source references with every RAG response.

---

