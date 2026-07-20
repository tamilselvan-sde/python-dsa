# Sparse Retrieval

> Author: **Tamilselvan** · ✉️ tamilselvan.sde@gmail.com · 🔗 [LinkedIn](https://www.linkedin.com/in/tamilselvan-ai/)
>

## 1. What is it?
**ELI5:** Sparse retrieval is like searching for a book by looking for specific words in the index at the back. If you want "quantum computing," you flip to the Q section and find every page that mentions "quantum" and "computing." It's a literal word match — no interpretation, no synonyms, just pure word counting.

**Technical Definition:** Sparse retrieval represents documents and queries as sparse, high-dimensional vectors where each dimension corresponds to a unique term in the vocabulary. Non-zero entries represent term presence/frequency in the document. Retrieval scores are computed by matching query terms against the document's term vector using functions like TF-IDF or BM25. The "sparse" name comes from the fact that most entries in these vectors are zero (a document contains only a tiny fraction of the total vocabulary).

## 2. Why do we need it?
- **Exact match reliability:** When you need a specific term, code, or serial number, sparse retrieval finds it — dense vector search dilutes exact tokens into averaged embeddings.
- **Interpretability:** You can explain exactly why a document matched: "this document matched because it contains the words 'GDPR' and 'compliance'."
- **Domain specificity:** In legal, medical, or technical domains, precise terminology matters. "ICD-10-CM R56.9" is an exact code that BM25 matches perfectly.
- **Low computational cost:** No GPU needed. Sparse indexes fit in RAM and searches complete in milliseconds on a laptop.
- **No training required:** BM25 and TF-IDF work out of the box with zero data or tuning.
- **Foundation for hybrid search:** Every hybrid system needs a sparse component to catch what dense retrieval misses.

## 3. Real-world Example
- **Elasticsearch / OpenSearch:** The world's most deployed search engine uses BM25 as its default scoring algorithm. Powers Wikipedia search, GitHub code search, and Stack Overflow.
- **Lucene / Solr:** Apache Lucene's inverted index (tokens → documents) is the gold standard for sparse retrieval. Used by Netflix's search, LinkedIn, and Twitter's early search.
- **Splunk:** Sparse retrieval for log data — search for specific error codes, IP addresses, or timestamps across petabytes of machine data.
- **PubMed:** NLM's biomedical search engine uses TF-IDF weighted term matching to search 35M+ medical citations.
- **Elastic APM:** Error log search relies on sparse retrieval for exact stack trace matching.

## 4. Architecture Diagram (ASCII)
```
+-----------+
| Document  |
+-----+-----+
      |
      v
[Tokenizer + Stemmer]
      |
      v
[Term Frequency Table]
  | term        | freq |
  |-------------|------|
  | "quantum"   | 3    |
  | "computing" | 2    |
  | "physics"   | 1    |
      |
      v
[Inverted Index]
  term        | doc_ids
  ------------|----------------
  "quantum"   | doc1, doc3, doc7
  "computing" | doc1, doc2, doc4
  "physics"   | doc3, doc5
      |
      v
[Query: "quantum physics"]
  --> intersect postings lists
  --> score: doc3 (high), doc1 (medium)
```

## 5. Internal Working
1. **Tokenization:** Document text is split into tokens (words, n-grams, or subwords). Tokens are lowercased, stopwords may be removed, and stemming/lemmatization may be applied.
2. **Inverted index construction:** For each unique term, a posting list is built containing all document IDs where that term appears, along with term frequency in each document.
3. **Query processing:** The input query is tokenized using the same pipeline. Each query term is looked up in the inverted index.
4. **Scoring:** Each matching document receives a score based on:
   - **TF (Term Frequency):** How often the term appears in the document (more = more relevant).
   - **IDF (Inverse Document Frequency):** How rare the term is across all documents (rarer terms are more discriminative).
   - **Document length normalization:** Longer documents have higher TF naturally — normalization compensates.
5. **Ranking:** Documents are sorted by descending score, and the top-K are returned.

## 6. Production Flow
```
Indexing Flow:
  Document → Tokenization → Stemming → Inverted Index → Persist (SSD/RAM)

Query Flow:
  User Query → Tokenization → Stemming → Lookup Postings → Score → Rank → Top-K

Scoring (BM25):
  score(D,Q) = Σ [IDF(qi) * tf(qi,D) * (k1+1) / (tf(qi,D) + k1 * (1-b + b*|D|/avgdl))]
```

## 7. HLD
```
                 +------------------+
                 |   Document       |
                 |   Ingestion      |
                 |   Pipeline       |
                 +--------+---------+
                          |
                 +--------v---------+
                 |   Indexer         |
                 | (Tokenizer,       |
                 |  Inverted Index)  |
                 +--------+---------+
                          |
              +-----------+-----------+
              |                       |
    +---------v--------+    +---------v--------+
    | Shard 1          |    | Shard N          |
    | (term range A-M) |    | (term range N-Z) |
    +---------+--------+    +---------+--------+
              |                       |
              +-----------+-----------+
                          |
                 +--------v---------+
                 |   Query Router   |
                 | (distributed     |
                 |  search scatter- |
                 |  gather)         |
                 +--------+---------+
                          |
                 +--------v---------+
                 |   Results Merger |
                 | (score normalize,|
                 |  sort, top-K)    |
                 +------------------+
```

## 8. LLD
```python
import math
from collections import Counter, defaultdict
from typing import Dict, List, Tuple

class SparseRetrievalIndex:
    def __init__(self):
        self.inverted_index: Dict[str, List[Tuple[str, int]]] = defaultdict(list)
        self.doc_lengths: Dict[str, int] = {}
        self.num_docs: int = 0
        self.avg_doc_length: float = 0.0

    def add_document(self, doc_id: str, tokens: List[str]):
        tf = Counter(tokens)
        self.doc_lengths[doc_id] = len(tokens)
        self.num_docs += 1
        self.avg_doc_length = (
            (self.avg_doc_length * (self.num_docs - 1) + len(tokens))
            / self.num_docs
        )
        for term, freq in tf.items():
            self.inverted_index[term].append((doc_id, freq))

    def search(
        self, query_tokens: List[str], top_k: int = 10
    ) -> List[Tuple[str, float]]:
        scores = defaultdict(float)
        query_tf = Counter(query_tokens)

        for term in query_tokens:
            if term not in self.inverted_index:
                continue

            # IDF
            df = len(self.inverted_index[term])
            idf = math.log(
                (self.num_docs - df + 0.5) / (df + 0.5) + 1
            )

            for doc_id, tf in self.inverted_index[term]:
                doc_len = self.doc_lengths[doc_id]
                # BM25 component for this term
                bm25_score = idf * (tf * 1.5) / (
                    tf + 1.5 * (
                        1 - 0.75 + 0.75 * doc_len / self.avg_doc_length
                    )
                )
                scores[doc_id] += bm25_score

        ranked = sorted(scores.items(), key=lambda x: x[1], reverse=True)
        return ranked[:top_k]
```

## 9. Python Implementation
```python
import math
import re
from collections import Counter, defaultdict
from typing import List, Optional, Tuple

from fastapi import FastAPI
from pydantic import BaseModel, Field

app = FastAPI(title="Sparse Retrieval Service")

class SparseQuery(BaseModel):
    query: str = Field(..., min_length=1)
    top_k: int = Field(default=10, ge=1, le=100)

class SparseResult(BaseModel):
    doc_id: str
    score: float
    text: str

class SparseResponse(BaseModel):
    results: List[SparseResult]
    latency_ms: float

def tokenize(text: str) -> List[str]:
    text = text.lower()
    tokens = re.findall(r"\w+", text)
    return tokens

class BM25Index:
    def __init__(self, k1: float = 1.5, b: float = 0.75):
        self.k1 = k1
        self.b = b
        self.index: Dict[str, List[Tuple[str, int]]] = defaultdict(list)
        self.doc_store: Dict[str, str] = {}
        self.doc_lengths: Dict[str, int] = {}
        self.num_docs = 0
        self.avgdl = 0.0

    def add_document(self, doc_id: str, text: str):
        tokens = tokenize(text)
        self.doc_store[doc_id] = text
        self.doc_lengths[doc_id] = len(tokens)
        self.num_docs += 1
        self.avgdl = (
            (self.avgdl * (self.num_docs - 1) + len(tokens))
            / self.num_docs
        )
        for term, freq in Counter(tokens).items():
            self.index[term].append((doc_id, freq))

    def search(self, query: str, top_k: int = 10) -> List[dict]:
        query_tokens = tokenize(query)
        query_tf = Counter(query_tokens)
        scores = defaultdict(float)

        for term in query_tokens:
            if term not in self.index:
                continue
            df = len(self.index[term])
            idf = math.log((self.num_docs - df + 0.5) / (df + 0.5) + 1)

            for doc_id, tf in self.index[term]:
                doc_len = self.doc_lengths[doc_id]
                numerator = tf * (self.k1 + 1)
                denominator = tf + self.k1 * (
                    1 - self.b + self.b * doc_len / self.avgdl
                )
                scores[doc_id] += idf * numerator / denominator

        ranked = sorted(scores.items(), key=lambda x: x[1], reverse=True)
        return [
            {
                "doc_id": doc_id,
                "score": score,
                "text": self.doc_store.get(doc_id, ""),
            }
            for doc_id, score in ranked[:top_k]
        ]

index = BM25Index()

for i in range(500):
    index.add_document(
        f"doc_{i}",
        f"This is document {i} about machine learning and artificial intelligence."
    )

@app.post("/sparse/search", response_model=SparseResponse)
async def sparse_search(query: SparseQuery):
    import time
    start = time.time()
    results = index.search(query.query, query.top_k)
    return SparseResponse(
        results=[SparseResult(**r) for r in results],
        latency_ms=(time.time() - start) * 1000,
    )

@app.post("/sparse/add")
async def add_document(doc_id: str, text: str):
    index.add_document(doc_id, text)
    return {"status": "ok", "doc_id": doc_id}
```

## 10. Folder Structure
```
sparse-retrieval/
├── src/
│   ├── __init__.py
│   ├── main.py
│   ├── indexer.py
│   ├── tokenizer.py
│   ├── scorer.py
│   ├── models/
│   │   └── schemas.py
│   └── utils.py
├── tests/
├── deploy/
└── pyproject.toml
```

## 11. Configuration
```yaml
sparse_retrieval:
  tokenizer:
    type: whitespace  # whitespace | regex | ngram | custom
    lowercase: true
    stopwords: true
    stemmer: porter   # none | porter | snowball
    min_token_length: 2
    max_token_length: 50

  bm25:
    k1: 1.5
    b: 0.75
    top_k_default: 10

  index:
    storage: memory   # memory | disk | mmap
    update_mode: incremental  # full | incremental
```

## 12. Flowchart
```
Start
  |
  v
[Input Query]
  |
  v
[Tokenize + Stem]
  |
  v
[For Each Query Term]
  |
  v
[Lookup Inverted Index]
  |
  v
[Get Postings List: (doc_id, tf)]
  |
  v
[Compute BM25 Score]
  |
  v
[Accumulate Scores per Doc]
  |
  v
[Sort by Score (desc)]
  |
  v
[Return Top-K Results]
  |
  End
```

## 13. Sequence Diagram
```
Client      API         Indexer       Inverted_Index
  |          |             |               |
  |--Query-->|             |               |
  |          |--tokenize-->|               |
  |          |<--tokens----|               |
  |          |             |               |
  |          |--lookup------------------->|
  |          |             |               |
  |          |<--postings-----------------|
  |          |             |               |
  |          |--score-------------------->|
  |          |             |               |
  |<--Results|             |               |
```

## 14. Pros
- Lightning fast — millisecond search on millions of documents
- Zero training required — works immediately on any text corpus
- Full recall — exact match guaranteed (no ANN approximation)
- Fully interpretable — every match is explained by specific terms
- Low operational cost — runs on CPU, no GPU needed
- Mature tooling — Elasticsearch, Lucene, Solr are battle-tested at Google/Netflix scale

## 15. Cons
- Vocabulary mismatch — "car" doesn't match "automobile" without synonym expansion
- No semantic understanding — "bank" (river) and "bank" (financial) treated identically
- Bag of words — loses word order, phrase structure, and negation
- High-dimensional sparse vectors — memory scales with vocabulary size
- Poor on short documents — little term frequency signal
- Patent/legal text is hard — dense legalese overlaps heavily

## 16. Alternatives
| Method | Description |
|--------|-------------|
| Dense retrieval | Neural embeddings, semantic matching |
| SPLADE | Learned sparse retrieval — neural term weighting |
| ColBERT | Late interaction model for token-level matching |
| Elastic Learned Sparse Encoder | Elasticsearch's trained sparse encoder |
| N-gram based retrieval | Subword matching for morphologically rich languages |

## 17. Performance Considerations
- **Index size:** Inverted index is typically 50-100% of the original corpus size.
- **Search time:** O(number of query terms × average posting list length). Typically <10ms for millions of docs.
- **Memory:** Store index in mmap for large corpora — OS handles page cache.
- **Stemming reduces index size** by ~30% (merges "compute", "computing", "computation").
- **Stopword removal** reduces posting list size for high-frequency terms by ~40%.

## 18. Scaling to Millions
- **Shard by term range** (A-F, G-M, N-Z) across multiple nodes.
- **Replicate shards** for read throughput.
- **Distributed search** — scatter queries to all shards, merge results (Elasticsearch model).
- **Use mmap** for index storage — avoids loading entire index into RAM.
- **Posting list compression** — use variable-byte encoding or Simple9 compression for disk efficiency.

## 19. Failure Scenarios
| Failure | Impact | Mitigation |
|---------|--------|------------|
| No query terms in index | Empty results | Query expansion / fuzzy matching fallback |
| Stopword-only query | All documents match | Reject or expand query |
| Index corruption | Wrong/missing results | Daily index snapshots |
| Memory pressure | OOM | Use mmap-based index storage |

## 20. Security
- Sanitize query input to prevent regex/pattern injection attacks.
- Implement field-level access control (document-level security filters).
- Use query allow-lists for high-risk search interfaces.
- Encrypt the inverted index at rest.
- Rate-limit search endpoint to prevent abuse.

## 21. Monitoring
- **Search latency:** p50, p95, p99
- **Index size:** Disk usage, posting list count
- **Query term count distribution:** Average query length
- **Zero-result rate:** % of queries returning no results
- **Index freshness:** Time since last document indexed
- **Cache hit ratio:** Query result cache

## 22. Interview Questions
| Level | Question | Key Points |
|-------|----------|------------|
| L3 | How does TF-IDF work? | TF = term frequency in doc, IDF = log(N/df), product |
| L3 | What is an inverted index? | Mapping from terms to document IDs + positions |
| L4 | How is BM25 different from TF-IDF? | BM25 has k1 (term saturation) and b (length normalization) |
| L4 | What are the tradeoffs of stemming? | Better recall, worse precision. "University" stems to "univers" |
| L5 | Design a search engine for 100M documents | Elasticsearch-style: sharded inverted index, distributed search, merge-sort |

## 23. Cheat Sheet
```python
# Sparse retrieval in 5 lines
from sklearn.feature_extraction.text import TfidfVectorizer

corpus = ["doc1 text", "doc2 text", "doc3 text"]
vectorizer = TfidfVectorizer()
tfidf_matrix = vectorizer.fit_transform(corpus)
query_vec = vectorizer.transform([query])
scores = (tfidf_matrix @ query_vec.T).toarray().flatten()
top_k = np.argsort(scores)[-10:][::-1]
```

## 24. Common Mistakes
- **Skipping stemming** — "running" and "ran" don't match; stemming fixes this.
- **Including stopwords in the index** — "the", "and", "of" bloat the index and add noise.
- **Not normalizing for document length** — longer docs naturally score higher; BM25's b parameter compensates.
- **Using TF-IDF when BM25 is better** — BM25 is strictly better in every benchmark.
- **Ignoring phrase queries** — "new york" vs "new" + "york" separately should be handled with n-grams or position indexing.

## 25. Production Best Practices
1. **Use BM25, not TF-IDF** — BM25 is strictly superior with no real downsides.
2. **Always apply stemming** — it's a free recall improvement of 10-20%.
3. **Remove stopwords from the index** — reduces index size by ~40% and improves precision.
4. **Use Elasticsearch or Tantivy** — don't build your own sparse index in production; they've solved distributed search, replication, and recovery.
5. **Monitor zero-result rate** — if >5% of queries return nothing, your tokenizer may be too aggressive.
6. **Store positions for phrase queries** — if users search in quotes, you need positional information.
7. **Implement fuzzy matching** — handle typos with Levenshtein automata or n-gram overlap.
8. **Always combine with dense retrieval in a hybrid setup** — sparse alone misses semantic matches that embeddings capture.
