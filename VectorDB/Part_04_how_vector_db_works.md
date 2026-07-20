# Part 4: How Vector DB Works

> Author: **Tamilselvan** · ✉️ tamilselvan.sde@gmail.com · 🔗 [LinkedIn](https://www.linkedin.com/in/tamilselvan-ai/)
>

## End-to-End Flow

```mermaid
graph TB
    subgraph "INGESTION PIPELINE"
        A[Raw Data] --> B[Document]
        B --> C[Chunking]
        C --> D[Chunks]
        D --> E[Embedding Model]
        E --> F[Dense Vectors]
        F --> G[Vector Index]
        G --> H[(Vector Database)]
    end
    
    subgraph "QUERY PIPELINE"
        I[User Query] --> J[Embedding Model]
        J --> K[Query Vector]
        K --> L[ANN Search]
        L --> M[Top-K Results]
        M --> N[Reranker]
        N --> O[Final Results]
        O --> P[LLM / Application]
    end
    
    H -.->|Index files| G
    H -.->|Query| L
```

### Step-by-Step: Ingestion

```mermaid
sequenceDiagram
    participant User
    participant App as Application
    participant Chunker
    participant Embedder as Embedding Model
    participant DB as Vector Database
    
    User->>App: Upload PDF Document
    App->>Chunker: Split into chunks
    Chunker->>Chunker: Chunk 1: "Introduction to..."
    Chunker->>Chunker: Chunk 2: "The methodology..."
    Chunker->>Embedder: Send chunks for embedding
    Embedder->>Embedder: Generate vectors
    Embedder->>DB: Insert vectors + metadata
    DB->>DB: Build/update index
    DB-->>App: Confirm stored
    App-->>User: Document processed
```

### Step-by-Step: Query

```mermaid
sequenceDiagram
    participant User
    participant App as Application
    participant Embedder as Embedding Model
    participant DB as Vector Database
    participant Reranker
    participant LLM
    
    User->>App: "What is vector search?"
    App->>Embedder: Embed query
    Embedder-->>App: Query vector [0.23, -0.45, ...]
    App->>DB: ANN search (query_vector, k=10)
    DB->>DB: Traverse HNSW graph / IVF clusters
    DB-->>App: Top-10 chunks with scores
    App->>Reranker: Rerank top-10
    Reranker-->>App: Reranked top-3
    App->>LLM: Context + query
    LLM-->>App: Generated answer
    App-->>User: "Vector search finds similar..."
```

---

## Inside a Vector DB: Architecture

```mermaid
architecture-beta
    group api[API Layer]
    group index[Index Layer]
    group storage[Storage Layer]
    
    service client(Users/client)[Client] in api
    service embedder(Users/robot)[Embedding Service] in api
    service filter(Users/gear)[Filter Engine] in api
    
    service hnsw(Users/database)[HNSW Index] in index
    service ivf(Users/database)[IVF Index] in index
    service pq(Users/compress)[Product Quantization] in index
    
    service ram(Users/device)[RAM] in storage
    service ssd(Users/device)[SSD/Disk] in storage
    service cache(Users/database)[Cache] in storage
    
    client:R <--> L:filter
    filter:R --> L:hnsw
    filter:R --> L:ivf
    hnsw:T <--> B:pq
    ivf:T <--> B:pq
    pq:T <--> B:ram
    ram:T <--> B:ssd
    ram:T <--> B:cache
```

---

## Core Components

```mermaid
graph TD
    A[Vector Database] --> B[Storage Layer]
    A --> C[Index Layer]
    A --> D[Query Layer]
    A --> E[API Layer]
    
    B --> B1[Vector Storage]
    B --> B2[Metadata Storage]
    B --> B3[WAL / Write-Ahead Log]
    
    C --> C1[Index Builder]
    C --> C2[Index Optimizer]
    C --> C3[Quantization Engine]
    
    D --> D1[ANN Search]
    D --> D2[Filter Executor]
    D --> D3[Reranking]
    D --> D4[Hybrid Search]
    
    E --> E1[REST API]
    E --> E2[gRPC API]
    E --> E3[Client SDKs]
```

---

## Storage Format: How Vectors Are Stored

```mermaid
graph LR
    subgraph "Vector File (on disk)"
        A["Header<br/>dim=768, count=1M, dtype=float32"]
        B["Vector 0<br/>[0.12, -0.34, ..., 0.89]"]
        C["Vector 1<br/>[0.45, 0.67, ..., -0.12]"]
        D["..."]
        E["Vector 999999<br/>[-0.78, 0.01, ..., 0.34]"]
    end
    
    subgraph "Metadata File"
        F["ID→Payload Map"]
        G["Filter Indexes"]
        H["Tenant Mapping"]
    end
```

**Storage Calculation:**
```
1 million vectors × 768 dimensions × 4 bytes (float32) = 3.07 GB
+ metadata overhead ~ 20-30%
Total ~ 3.7-4 GB per million vectors
```

---

## Index Types Supported

```mermaid
graph TD
    A{Vector Database} --> B{Index Type}
    B --> C[Flat / Brute Force]
    B --> D[ANN Index]
    
    D --> E[HNSW]
    D --> F[IVF]
    D --> G[IVF_PQ]
    D --> H[HNSW_PQ]
    D --> I[SCANN]
    D --> J[DiskANN]
    
    C --> K[Exact<br/>100% recall<br/>Slow on large data]
    E --> L[Fast search<br/>High memory<br/>Best for <10M]
    F --> M[Low memory<br/>Moderate speed<br/>Best for 10M+]
    G --> N[Very low memory<br/>Good speed<br/>Best for 100M+]
    J --> O[SSD-based<br/>Billions scale]
```

---

### ELI5: How Vector DB Works

> Imagine a giant library where every book has been given a secret code — a list of 1000 numbers. Books about cooking have similar codes. Books about space have different codes. When you ask "find me books like this recipe book," the librarian:
>
> 1. Converts your request into a code (embedding)
> 2. Doesn't check every single book (too slow!)
> 3. Uses a special map (index) to jump directly to the neighborhood of similar codes
> 4. Checks only the nearby books (ANN search)
> 5. Returns the closest matches
>
> This is 1000x faster than checking every book!

---

### Production Tip
> **Read vs Write optimization:** Vector DBs are designed for read-heavy workloads. Ingestion is typically batch-based (offline), while queries are real-time (online). Design your pipeline accordingly — use message queues (Kafka, RabbitMQ) between ingestion and indexing.

---

### Common Mistake
> **❌ Not separating embedding generation from vector storage.** Running an embedding model inline during query time adds 100-500ms latency. Pre-compute and cache embeddings whenever possible.

---

