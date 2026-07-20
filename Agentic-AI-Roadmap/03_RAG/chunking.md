# Document Chunking Strategies

## 1. What is it?
**ELI5:** Imagine a really long book. You can't fit the whole thing in your brain at once. So you tear out chapters, and sometimes paragraphs, to read one at a time. Document chunking is the same — it splits a long document into smaller, manageable pieces before storing them in the knowledge base. Each piece (chunk) gets its own embedding and is searchable independently.

**Technical Definition:** Document chunking is the process of dividing a document into smaller segments (chunks) before indexing them into a retrieval system. Each chunk must be independently meaningful, self-contained, and optimally sized for the embedding model's context window and the downstream LLM's context budget. Chunking strategy directly determines retrieval quality — poor chunking produces poor RAG results regardless of the sophistication of the rest of the pipeline.

## 2. Why do we need it?
- **Embedding model limits:** Embedding models have a maximum input length (512 tokens for BERT, 8191 for text-embedding-3). Documents longer than this must be split.
- **Semantic granularity:** Retrieval works best when matching query to a focused passage, not a book-length document. A query about "error code 503" should match the paragraph about 503 errors, not the entire 200-page operations manual.
- **Context budget:** The LLM's context window is finite (8K-200K tokens). You can only inject a few retrieved chunks. Each chunk should pack maximum relevant information per token.
- **Precision of citation:** When the answer cites a chunk, that chunk should be small enough that the user can find the exact information. A 50-page chunk is useless as a citation.
- **Cross-document relationships:** Chunking preserves document boundaries so the system can track which document a fact came from.

## 3. Real-world Example
- **LlamaIndex:** Offers 5+ chunking strategies: SentenceSplitter, TokenTextSplitter, SemanticSplitterNodeParser. Most popular is SentenceSplitter with chunk_size=1024, overlap=200.
- **LangChain's Text Splitters:** RecursiveCharacterTextSplitter with chunk_size=1000, chunk_overlap=200 is the default for most RAG tutorials and production deployments.
- **Unstructured.io:** Enterprise document parsing — chunks PDFs, HTML, markdown by document structure (by-title, by-section). Used by Fortune 500 for document ingestion.
- **Cohere's Embed v3:** Recommends chunking at 512 tokens with 10% overlap. Their API includes a built-in chunk endpoint.
- **Airbnb's DocQA:** Chunks knowledge base articles at the paragraph level, with semantic overlap for bridging context.

## 4. Architecture Diagram (ASCII)
```
Document: "Long article about machine learning..."
    |
    v
[Chunking Strategies]
    |
    +----> Fixed-size: split every N characters/tokens
    |     "Long article about machine learning and"
    |     " and deep learning covers neural networks"
    |
    +----> Recursive: split by \n\n, then \n, then .
    |     "Long article about machine learning" (section)
    |     "This section covers neural networks..." (section)
    |
    +----> Semantic: split at topic boundaries
    |     "Introduction to Machine Learning" (topic 1)
    |     "Neural Networks and Deep Learning" (topic 2)
    |
    +----> Document-aware: respect headers, sections
    |     "## 1. Overview  ..." (section boundary)
    |     "## 2. Methods ..."
    |
    v
[Chunks with Metadata: source_doc, section, position, tokens]
```

## 5. Internal Working
1. **Content extraction:** Parse the document format (PDF, HTML, Markdown, plain text) into clean text.
2. **Structural analysis:** Identify natural boundaries: headings, paragraphs, sentences, bullet points, code blocks.
3. **Strategy selection:** Choose strategy based on document type and use case:
   - **Fixed-size token split:** Simple, fast. Split every N tokens with overlap. Risk: cuts sentences in half.
   - **Recursive character split:** Split by `\n\n` first, then `\n`, then `.`. Respects natural boundaries.
   - **Semantic split:** Use embeddings or an LLM to detect topic boundaries. Most accurate, slowest.
   - **Document-aware split:** Use markdown headers, PDF sections, or HTML headings as natural boundaries.
4. **Overlap application:** Add N tokens of overlap between consecutive chunks to preserve context across the boundary.
5. **Metadata attachment:** Each chunk gets: source_document, chunk_index, section_title, token_count, embedding.

## 6. Production Flow
```
Raw Document (PDF, DOCX, HTML, MD)
    |
    v
[Parser: extract text, strip formatting]
    |
    v
[Chunker: apply strategy]
    |
    +---> Sentence splitter → sentences
    |
    +---> Group sentences into chunks (target: 512 tokens)
    |
    +---> Apply overlap (200 tokens between chunks)
    |
    v
[Post-process: filter empty chunks, attach metadata]
    |
    v
[Embed each chunk → Vector DB]
```

## 7. HLD
```
                    +-------------------------+
                    | Document Ingestion API  |
                    +-----------+-------------+
                                |
                    +-----------v-------------+
                    | Document Parser         |
                    | (PDF, HTML, MD, DOCX)    |
                    +-----------+-------------+
                                |
                    +-----------v-------------+
                    | Chunking Service        |
                    | (Strategy selector)     |
                    +-----------+-------------+
                +---------------+---------------+
                |               |               |
    +-----------v------+ +-----v--------+ +----v----------+
    | Fixed-size       | | Recursive    | | Semantic      |
    | Chunker          | | Chunker      | | Chunker       |
    +-----------+------+ +-----+--------+ +----+----------+
                |               |               |
                +-------+-------+------+--------+
                        |              |
                +-------v------+  +----v----------+
                | Metadata     |  | Embedding     |
                | Enricher     |  | Service       |
                +-------+------+  +-------+-------+
                        |                 |
                        +-------+---------+
                                |
                    +-----------v-------------+
                    | Vector DB (chunk store)  |
                    +-------------------------+
```

## 8. LLD
```python
from dataclasses import dataclass, field
from enum import Enum
from typing import List, Optional
import tiktoken

class ChunkingStrategy(str, Enum):
    FIXED = "fixed"
    RECURSIVE = "recursive"
    SEMANTIC = "semantic"
    DOCUMENT_AWARE = "document_aware"

@dataclass
class Chunk:
    id: str
    text: str
    metadata: dict = field(default_factory=dict)
    token_count: int = 0
    embedding: Optional[List[float]] = None

@dataclass
class ChunkingConfig:
    strategy: ChunkingStrategy = ChunkingStrategy.RECURSIVE
    chunk_size: int = 512
    chunk_overlap: int = 200
    min_chunk_size: int = 50

class BaseChunker:
    def __init__(self, config: ChunkingConfig):
        self.config = config
        self.tokenizer = tiktoken.get_encoding("cl100k_base")

    def count_tokens(self, text: str) -> int:
        return len(self.tokenizer.encode(text))

class RecursiveChunker(BaseChunker):
    def __init__(self, config: ChunkingConfig):
        super().__init__(config)
        self.separators = ["\n\n", "\n", ". ", " ", ""]

    def chunk(self, doc_id: str, text: str, metadata: dict) -> List[Chunk]:
        chunks = []
        current = text
        while self.count_tokens(current) > self.config.chunk_size:
            best_split = self._find_split(current, self.config.chunk_size)
            chunk_text = current[:best_split].strip()
            chunks.append(self._make_chunk(doc_id, chunk_text, metadata, len(chunks)))
            # Overlap: go back overlap tokens
            overlap_chars = self._backtrack(current, best_split)
            current = current[best_split - overlap_chars:]
        if current.strip():
            chunks.append(self._make_chunk(doc_id, current.strip(), metadata, len(chunks)))
        return chunks

    def _find_split(self, text: str, target_tokens: int) -> int:
        for sep in self.separators:
            pos = text.rfind(sep, 0, min(len(text), target_tokens * 4))
            if pos != -1:
                return pos + len(sep)
        return int(target_tokens * 3.5)  # fallback: approximate char count

    def _backtrack(self, text: str, split_pos: int) -> int:
        overlap = self.config.chunk_overlap
        return min(overlap * 4, split_pos)  # approximate chars

    def _make_chunk(
        self, doc_id: str, text: str, metadata: dict, idx: int
    ) -> Chunk:
        return Chunk(
            id=f"{doc_id}_chunk_{idx}",
            text=text,
            metadata={**metadata, "chunk_index": idx, "source_doc": doc_id},
            token_count=self.count_tokens(text),
        )
```

## 9. Python Implementation
```python
from enum import Enum
from typing import List, Optional

from fastapi import FastAPI
from pydantic import BaseModel, Field
import tiktoken

app = FastAPI(title="Document Chunking Service")

class ChunkStrategy(str, Enum):
    fixed = "fixed"
    recursive = "recursive"
    semantic = "semantic"

class ChunkRequest(BaseModel):
    doc_id: str = Field(..., min_length=1)
    text: str = Field(..., min_length=1, max_length=100000)
    strategy: ChunkStrategy = ChunkStrategy.recursive
    chunk_size: int = Field(default=512, ge=100, le=8192)
    chunk_overlap: int = Field(default=200, ge=0, le=1000)
    metadata: dict = {}

class ChunkResponse(BaseModel):
    doc_id: str
    chunks: List[dict]
    strategy: str
    total_chunks: int
    total_tokens: int

class Chunker:
    def __init__(self):
        self.enc = tiktoken.get_encoding("cl100k_base")

    def count_tokens(self, text: str) -> int:
        return len(self.enc.encode(text))

    def chunk_fixed(self, text: str, size: int, overlap: int) -> List[str]:
        tokens = self.enc.encode(text)
        chunks = []
        start = 0
        while start < len(tokens):
            end = min(start + size, len(tokens))
            chunk_tokens = tokens[start:end]
            chunks.append(self.enc.decode(chunk_tokens))
            start += size - overlap
        return chunks

    def chunk_recursive(self, text: str, size: int, overlap: int) -> List[str]:
        separators = ["\n\n", "\n", ". ", " "]
        chunks = []
        current = text

        while self.count_tokens(current) > size:
            best_pos = len(current)
            for sep in separators:
                pos = current.rfind(sep, 0, size * 4)
                if pos != -1 and pos < best_pos:
                    best_pos = pos + len(sep)
            if best_pos >= len(current):
                best_pos = min(size * 4, len(current))

            chunk_text = current[:best_pos].strip()
            if chunk_text and self.count_tokens(chunk_text) > 0:
                chunks.append(chunk_text)

            overlap_chars = min(overlap * 4, best_pos)
            current = current[best_pos - overlap_chars:]

        if current.strip():
            chunks.append(current.strip())

        return chunks

    def chunk_semantic(self, text: str, size: int, _overlap: int) -> List[str]:
        paragraphs = text.split("\n\n")
        chunks = []
        current_chunk = []
        current_size = 0

        for para in paragraphs:
            para_tokens = self.count_tokens(para)
            if current_size + para_tokens > size and current_chunk:
                chunks.append("\n\n".join(current_chunk))
                current_chunk = []
                current_size = 0
            current_chunk.append(para)
            current_size += para_tokens

        if current_chunk:
            chunks.append("\n\n".join(current_chunk))
        return chunks

chunker = Chunker()

@app.post("/chunk", response_model=ChunkResponse)
async def chunk_document(req: ChunkRequest):
    if req.strategy == ChunkStrategy.fixed:
        raw_chunks = chunker.chunk_fixed(req.text, req.chunk_size, req.chunk_overlap)
    elif req.strategy == ChunkStrategy.semantic:
        raw_chunks = chunker.chunk_semantic(req.text, req.chunk_size, req.chunk_overlap)
    else:
        raw_chunks = chunker.chunk_recursive(req.text, req.chunk_size, req.chunk_overlap)

    chunks = [
        {
            "chunk_id": f"{req.doc_id}_chunk_{i:04d}",
            "text": chunk,
            "token_count": chunker.count_tokens(chunk),
            "metadata": req.metadata,
        }
        for i, chunk in enumerate(raw_chunks)
    ]

    return ChunkResponse(
        doc_id=req.doc_id,
        chunks=chunks,
        strategy=req.strategy,
        total_chunks=len(chunks),
        total_tokens=sum(c["token_count"] for c in chunks),
    )
```

## 10. Folder Structure
```
chunking-service/
├── src/
│   ├── __init__.py
│   ├── main.py
│   ├── chunkers/
│   │   ├── __init__.py
│   │   ├── base.py
│   │   ├── fixed.py
│   │   ├── recursive.py
│   │   ├── semantic.py
│   │   └── document_aware.py
│   ├── parser/
│   │   ├── __init__.py
│   │   ├── pdf.py
│   │   ├── html.py
│   │   └── markdown.py
│   └── models/
│       └── schemas.py
├── tests/
├── deploy/
└── pyproject.toml
```

## 11. Configuration
```yaml
chunking:
  default_strategy: recursive
  recursive:
    chunk_size: 512
    chunk_overlap: 200
    separators: ["\n\n", "\n", ". ", " "]
  fixed:
    chunk_size: 512
    chunk_overlap: 200
  semantic:
    chunk_size: 512
    model: text-embedding-3-small
    similarity_threshold: 0.7
  document_aware:
    max_section_depth: 3
    min_section_tokens: 50
  min_chunk_size: 50
  max_chunk_size: 2000
```

## 12. Flowchart
```
Start
  |
  v
[Parse Document]
  |
  v
[Select Strategy]
  |
  +---> Fixed: tokenize → split at N → overlap → decode
  |
  +---> Recursive: try sep1 → sep2 → sep3 → ... → fallback
  |
  +---> Semantic: group paragraphs by embedding similarity
  |
  +---> Doc-aware: detect headers → structure tree → each leaf is a chunk
  |
  v
[Apply Overlap]
  |
  v
[Filter: remove chunks below min_size]
  |
  v
[Attach Metadata]
  |
  v
[Return Chunks for Embedding]
```

## 13. Sequence Diagram
```
Ingestion      Chunker      Tokenizer     Metadata
  |              |              |             |
  |--document--->|              |             |
  |              |--tokenize--->|             |
  |              |<--tokens-----|             |
  |              |              |             |
  |              |--split------>|             |
  |              |<--chunks-----|             |
  |              |              |             |
  |              |--enrich------------------>|
  |              |<--metadata-----------------|
  |              |              |             |
  |<--chunks-----|              |             |
```

## 14. Pros
- Right chunk size maximizes retrieval relevance
- Overlap preserves context across chunk boundaries
- Strategy flexibility for different document types
- Embedding models handle short chunks better than long ones
- Citations point to specific, actionable passages

## 15. Cons
- Too small → context fragmentation (lost connections between ideas)
- Too large → diluted embedding, less precise retrieval
- Overlap wastes tokens (10-30% of storage)
- Semantic chunking is expensive (additional embedding or LLM call)
- Hard boundary at chunk edges — key information can be split across two chunks

## 16. Alternatives
| Strategy | Description | Best for |
|----------|-------------|----------|
| Sentence-level chunks | Each sentence is a chunk | Factoid QA (trivia, definitions) |
| Paragraph-level chunks | Natural paragraph boundaries | News articles, blog posts |
| Section-level chunks | By markdown headers, PDF sections | Technical documentation, manuals |
| Sliding window | Every window of N tokens | Maximum recall, high redundancy |
| LLM-based chunking | LLM identifies topic boundaries | Highest quality, highest cost |
| Agentic chunking | Agent extracts semantically coherent units | Complex documents with mixed content |

## 17. Performance Considerations
- **Chunk size sweet spot:** 256-512 tokens. Below 128: too many chunks, high overhead. Above 1024: embedding dilution.
- **Overlap:** 10-20% of chunk size (50-100 tokens for 512-chunk). Less overlap = fewer total chunks but possible context loss.
- **Embedding cost:** 512-token chunks cost 2x more to embed than 256-token chunks but produce half the chunks.
- **Storage:** More chunks = more vectors = higher vector DB cost. 10K docs × 10 chunks = 100K vectors.
- **Tokenizer choice:** cl100k_base (GPT-4) vs p50k_base (GPT-3) vs BERT tokenizer. Must match downstream model.

## 18. Scaling to Millions
- **Parallelize chunking:** Process documents in batches across worker nodes (K8s job).
- **Streaming ingestion:** Chunk documents as they arrive — don't wait for the full batch.
- **Two-level chunking:** Store large chunks (1K tokens) for retrieval, then extract smaller sub-chunks for LLM context at query time.
- **Deduplicate chunks at scale:** Use MinHashLSH to identify near-identical chunks across documents.
- **Compress chunk metadata:** Use Parquet for chunk storage in data lakes.

## 19. Failure Scenarios
| Failure | Impact | Mitigation |
|---------|--------|------------|
| Chunk too small | Retrieves irrelevant text | Enforce min_chunk_size with neighbor merging |
| Chunk cuts mid-sentence | Hard to read in context | Use recursive splitter with sentence-aware boundaries |
| Empty chunks after filtering | Document not indexed | Fallback: include min_chunk_size chunks anyway |
| Tokenizer mismatch | Wrong chunk sizes | Validate token counts against the embedding model |
| Encoding errors | Garbled text | Detect and handle non-UTF-8 encodings (latin-1, shift-jis) |

## 20. Security
- Strip sensitive metadata (author, edit history) before chunking.
- Apply PII detection on chunks before embedding.
- Don't chunk binary or encrypted content without decryption.
- Validate chunk output size bounds to prevent DoS via tiny/large chunks.

## 21. Monitoring
- **Chunk size distribution:** Mean, median, min, max tokens per chunk
- **Chunk count per document:** Detect outliers (documents producing 1000+ chunks)
- **Overlap ratio:** % of total tokens that are overlap
- **Chunking latency:** Per document, per chunk
- **Skipped documents:** % of documents that fail chunking
- **Storage efficiency:** Tokens stored vs original tokens (overhead ratio)

## 22. Interview Questions
| Level | Question | Key Points |
|-------|----------|------------|
| L3 | Why can't we just put the whole document in one chunk? | Embedding model limits, context window, granularity |
| L3 | What's the difference between fixed-size and recursive splitting? | Fixed: pure token count. Recursive: respects natural boundaries |
| L4 | How do you choose chunk size for a RAG system? | Balance: small for precision, large for context. 256-512 tokens is standard |
| L4 | What problems does chunk overlap solve? | Prevents context fragmentation at chunk boundaries |
| L5 | Design a chunking strategy for a code repository | Hybrid: top-level by file, then by function/class boundaries |
| L5 | How would you evaluate chunking quality? | Retrieval recall with chunk as unit, human rating of chunk coherence |

## 23. Cheat Sheet
```python
# Recursive chunking in 5 lines
from langchain.text_splitter import RecursiveCharacterTextSplitter

splitter = RecursiveCharacterTextSplitter(
    chunk_size=512, chunk_overlap=200,
    separators=["\n\n", "\n", ". ", " "]
)
chunks = splitter.split_text(document_text)
```

## 24. Common Mistakes
- **Fixed-size across all document types** — a legal contract needs different chunking than a README.
- **Zero overlap** — key sentences at chunk boundaries are lost from both sides.
- **Chunking without considering embedding model limits** — text-embedding-3 supports 8191 tokens, but BERT models only 512. Match chunk size to model.
- **Chunking before cleaning text** — HTML tags, markdown artifacts, and extra whitespace waste token budget.
- **Ignoring metadata propagation** — every chunk must know its source document and position.
- **Tiny chunks (<50 tokens)** — too short for meaningful semantic representation.

## 25. Production Best Practices
1. **Use recursive chunking with sentence-aware boundaries** — it's the best balance of quality and speed.
2. **Set chunk_size to 256-512 tokens** — this is the sweet spot for most embedding models and use cases.
3. **Apply 10-20% overlap** (50-200 tokens) to prevent context fragmentation at boundaries.
4. **Match chunk size to the embedding model's optimal input length** — don't exceed the model's max nor waste capacity.
5. **Propagate metadata to every chunk** — source document, position, section title, URL.
6. **Filter chunks below min_chunk_size (50 tokens)** — they're too short for meaningful retrieval.
7. **Test chunking on a sample first** — visualize chunk boundaries before processing the full corpus.
8. **Consider two-level chunking** — store large chunks (1K) for retrieval, extract smaller sub-chunks (256) at query time for LLM context.
9. **Monitor chunk size distribution** — drift may indicate a change in document characteristics.
10. **Handle different document types with different strategies** — code files, PDFs, markdown, and plain text each need tailored chunking.
