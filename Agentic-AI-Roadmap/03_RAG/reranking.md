# Reranking

## 1. What is it?
**ELI5:** Imagine you're looking for a specific book in a library. First, a librarian quickly scans the shelves and brings you 50 books that might be relevant (retrieval). Then you sit down and carefully read each title and summary to pick the best 5 (reranking). Reranking is the careful reading step — it uses a more powerful but slower model to re-evaluate the top candidates from a fast initial retrieval.

**Technical Definition:** Reranking is a two-stage retrieval architecture. A fast, efficient first-stage retriever (BM25 or bi-encoder embeddings) retrieves a large set of candidate documents (top-50 to top-1000). A more accurate but computationally expensive second-stage reranker (typically a cross-encoder transformer) scores each candidate by jointly encoding the query and document together, producing a refined ranking. This separates recall (stage 1) from precision (stage 2).

## 2. Why do we need it?
- **Quality gap:** Bi-encoder retrievers compute query and document embeddings independently — they lose the fine-grained interactions between query tokens and document tokens. Cross-encoders capture these interactions, improving NDCG@10 by 10-15% absolute.
- **Efficiency:** Running a cross-encoder on 10M documents is infeasible (hours per query). Running it on top-100 is fast (milliseconds). Two-stage design gives you the best of both.
- **Context-aware scoring:** The reranker sees the full query-document pair, enabling it to detect subtle relevance signals (e.g., "Python" the snake vs "Python" the programming language).
- **Domain adaptation:** Rerankers can be fine-tuned on domain-specific relevance judgments, while the first-stage retriever stays generic.
- **Final quality gate:** In production RAG, the reranker is often the last step before the LLM — it's the final quality filter.

## 3. Real-world Example
- **Google Search:** The initial retrieval (index serving) gets ~1000 candidates. A BERT-based reranker (RankBrain / BERT) re-ranks them for the final top-10 results shown to users.
- **Microsoft Bing:** Uses a multi-stage pipeline: BM25 → dense retrieval → LightGBM reranker → transformer reranker → final presentation.
- **Cohere Rerank:** Cohere's commercial reranking API — send your query + up to 1000 documents, get relevance scores. Used by enterprise RAG systems for final document selection.
- **Elasticsearch Learning to Rank:** Elasticsearch supports pluggable reranking models (XGBoost, neural) that re-score top-N results from BM25.
- **Pinecone + Rerank:** Pinecone's vector search can pass results to a reranker (Cohere or cross-encoder) for improved final ranking.

## 4. Architecture Diagram (ASCII)
```
                    Stage 1 (Retrieval)
                    ===================
                    Fast: Bi-Encoder
                    Query → [Encoder] → q_vec
                    Doc   → [Encoder] → d_vec
                    Score = cos(q_vec, d_vec)
                    Retrieve top-100
                           |
                           v
                    Stage 2 (Reranking)
                    ===================
                    Accurate: Cross-Encoder
                    For each (query, doc) pair:
                    [CLS] query [SEP] doc [SEP]
                            ↓
                       [Transformer]
                            ↓
                       Relevance Score
                    
                    Rank by cross-encoder score → top-5
                           |
                           v
                     To LLM / User
```

## 5. Internal Working
**Cross-encoder architecture:**
The query and document are concatenated with special tokens: `[CLS] query text [SEP] document text [SEP]`. This joint sequence is fed through a full transformer (e.g., BERT). The `[CLS]` token's final hidden state is passed through a classification head that outputs a relevance score (typically 0-1 or -1 to 1).

**Why it's better than bi-encoder:**
- Bi-encoder: Query and doc are encoded independently → interaction happens only at the last dot product.
- Cross-encoder: Every query token can attend to every document token through all transformer layers → deep token-level interaction.

**Latency tradeoff:**
- Bi-encoder: Query embedding ~5ms, doc embedding is pre-computed. Total per query: ~5ms + ANN search.
- Cross-encoder: Must score each (query, doc) pair. For 100 candidates at ~10ms each = ~1000ms total.

## 6. Production Flow
```
Query: "Best practices for microservices monitoring"
    |
    v
[Stage 1: First-stage retrieval (fast)]
  BM25 / Vector DB → top-100 candidates (100-200ms)
    |
    v
[Deduplicate candidates by doc_id]
    |
    v
[Stage 2: Reranking (accurate)]
  For each of 100 candidates:
    → Format: [CLS] Query [SEP] Doc [SEP]
    → Cross-encoder inference (batch size = 8-32)
    → Relevance score per candidate
  (~500-1000ms for 100 candidates on GPU)
    |
    v
[Sort by cross-encoder score descending]
    |
    v
[Return top-5 to RAG pipeline]
```

## 7. HLD
```
                       +---------------------+
                       | First-Stage         |
                       | Retriever           |
                       | (BM25 / Vector DB)  |
                       +---------+-----------+
                                 |
                                 | top-100
                                 v
                       +---------------------+
                       | Candidate            |
                       | Deduplicator         |
                       +---------+-----------+
                                 |
                                 v
                       +---------------------+
                       | Reranker Service    |
                       | (Cross-Encoder)     |
                       | GPU cluster / ONNX  |
                       +---------+-----------+
                                 |
                                 | top-5
                                 v
                       +---------------------+
                       | RAG Orchestrator    |
                       +---------------------+
```

## 8. LLD
```python
import asyncio
from dataclasses import dataclass
from typing import List, Optional
import numpy as np

@dataclass
class RerankerConfig:
    model_name: str = "cross-encoder/ms-marco-MiniLM-L-6-v2"
    batch_size: int = 32
    max_length: int = 512
    device: str = "cuda"

class CrossEncoderReranker:
    def __init__(self, config: RerankerConfig):
        self.config = config
        self.model = None  # Load from HuggingFace in production
        self.tokenizer = None

    async def rerank(
        self, query: str, candidates: List[dict], top_k: int = 5
    ) -> List[dict]:
        if len(candidates) == 0:
            return []

        # Prepare pairs
        pairs = [
            (query, doc["text"][:self.config.max_length])
            for doc in candidates
        ]

        # Score in batches
        all_scores = []
        for i in range(0, len(pairs), self.config.batch_size):
            batch = pairs[i : i + self.config.batch_size]
            scores = await self._score_batch(batch)
            all_scores.extend(scores)

        # Attach scores and rerank
        for doc, score in zip(candidates, all_scores):
            doc["rerank_score"] = score

        candidates.sort(key=lambda x: x["rerank_score"], reverse=True)
        return candidates[:top_k]

    async def _score_batch(self, pairs: List[tuple]) -> List[float]:
        # In production: model inference
        # Returns random scores as placeholder
        return [float(np.random.random()) for _ in pairs]
```

## 9. Python Implementation
```python
from typing import List, Optional

import torch
from fastapi import FastAPI
from pydantic import BaseModel, Field
from transformers import AutoModelForSequenceClassification, AutoTokenizer

app = FastAPI(title="Reranking Service")

class RerankQuery(BaseModel):
    query: str = Field(..., min_length=1)
    documents: List[str] = Field(..., min_length=1, max_length=100)
    top_k: int = Field(default=5, ge=1, le=50)

class RerankItem(BaseModel):
    index: int
    score: float
    text: str

class RerankResponse(BaseModel):
    results: List[RerankItem]
    model: str
    latency_ms: float

class CrossEncoder:
    def __init__(self, model_name: str = "cross-encoder/ms-marco-MiniLM-L-6-v2"):
        self.device = torch.device("cuda" if torch.cuda.is_available() else "cpu")
        self.tokenizer = AutoTokenizer.from_pretrained(model_name)
        self.model = AutoModelForSequenceClassification.from_pretrained(model_name)
        self.model.to(self.device)
        self.model.eval()

    @torch.no_grad()
    def score(self, query: str, documents: List[str], batch_size: int = 32) -> List[float]:
        scores = []
        for i in range(0, len(documents), batch_size):
            batch = documents[i : i + batch_size]
            pairs = [(query, doc) for doc in batch]
            inputs = self.tokenizer(
                pairs,
                padding=True,
                truncation=True,
                max_length=512,
                return_tensors="pt",
            ).to(self.device)
            outputs = self.model(**inputs)
            batch_scores = outputs.logits.squeeze(-1).cpu().tolist()
            if isinstance(batch_scores, float):
                batch_scores = [batch_scores]
            scores.extend(batch_scores)
        return scores

reranker = CrossEncoder()

@app.post("/rerank", response_model=RerankResponse)
async def rerank(request: RerankQuery):
    import time
    start = time.time()

    scores = reranker.score(request.query, request.documents)
    indexed = list(enumerate(scores))
    indexed.sort(key=lambda x: x[1], reverse=True)
    top = indexed[: request.top_k]

    return RerankResponse(
        results=[
            RerankItem(
                index=idx,
                score=score,
                text=request.documents[idx],
            )
            for idx, score in top
        ],
        model=reranker.model.config._name_or_path,
        latency_ms=(time.time() - start) * 1000,
    )

@app.get("/health")
async def health():
    return {"status": "healthy", "device": str(reranker.device)}
```

## 10. Folder Structure
```
reranking-service/
├── src/
│   ├── __init__.py
│   ├── main.py
│   ├── cross_encoder.py
│   ├── config.py
│   ├── models/
│   │   └── schemas.py
│   └── utils.py
├── tests/
├── deploy/
│   ├── Dockerfile
│   └── k8s/
├── pyproject.toml
└── README.md
```

## 11. Configuration
```yaml
reranker:
  model:
    name: cross-encoder/ms-marco-MiniLM-L-6-v2
    revision: main
    max_length: 512

  inference:
    batch_size: 32
    device: cuda  # cpu | cuda
    fp16: true    # half precision for speed

  api:
    max_input_docs: 100
    top_k_default: 5
    top_k_max: 50
```

## 12. Flowchart
```
Start
  |
  v
Receive (query, candidate_docs[100])
  |
  v
[Deduplicate by doc_id]
  |
  v
[For each (query, doc) pair]
  |
  v
[Tokenize: [CLS] query [SEP] doc [SEP]]
  |
  v
[Cross-encoder forward pass]
  |
  v
[Extract [CLS] → linear → sigmoid → score]
  |
  v
[Accumulate scores]
  |
  v
[Sort by score descending]
  |
  v
[Return top-k (e.g., 5)]
  |
  End
```

## 13. Sequence Diagram
```
Retriever     Reranker Svc     Cross-Encoder     RAG
    |              |                |              |
    |--100 docs---->                |              |
    |              |                |              |
    |              |--score pair 1->|              |
    |              |<--0.92---------|              |
    |              |--score pair 2->|              |
    |              |<--0.85---------|              |
    |              |--...----------->              |
    |              |<--...----------               |
    |              |                |              |
    |              |--sort + select->|              |
    |              |                |              |
    |<--top-5------|                |              |
    |              |                |              |
    |              |---top-5 docs---------------->|
```

## 14. Pros
- ~10-15% absolute NDCG improvement over bi-encoder only
- Captures deep query-document token interactions
- Can handle nuanced relevance signals (negation, specificity, intent)
- Trainable on domain-specific data with standard cross-entropy loss
- Significantly improves RAG final answer quality

## 15. Cons
- Expensive: O(top_k) transformer inferences per query
- Adds 200-1000ms latency to the pipeline
- Requires GPU for acceptable latency (CPU is 5-10x slower)
- Context window limits — document must be truncated to 512 tokens
- Cannot scale to full corpus — must always be paired with a first-stage retriever

## 16. Alternatives
| Method | Description | When to use |
|--------|-------------|-------------|
| Bi-encoder + scoring | Same as retriever, no reranker | Latency-critical, lower quality needed |
| Late interaction (ColBERT) | Token-level scoring without full cross-attention | Balance of quality and speed |
| ListNet / LambdaRank | Learns to rank across lists | Full training data available |
| LLM-as-judge | Use GPT-4 to rerank by prompting | Small candidate sets, want interpretability |
| SetBERT | Cross-encoder across all docs jointly | Max accuracy, highest compute cost |

## 17. Performance Considerations
- **Batch inference:** Never score documents one at a time — batch at least 32 for 10x throughput.
- **FP16 inference:** Uses half the memory and runs 2x faster with negligible quality loss.
- **Model size:** MiniLM-L-6-v2 (22M params) is 10x faster than DeBERTa-large (300M params) with only 2-3% quality loss.
- **Max length:** 512 tokens. Longer docs must be truncated or segmented.
- **ONNX / TensorRT:** Export to ONNX for 2-3x speedup on CPU and GPU.
- **Cache scores for duplicate docs:** If the same doc appears in multiple candidate sets, cache its score.

## 18. Scaling to Millions
- **Shard reranking across GPUs:** stateless — 100 queries × 100 docs can be distributed across 10 GPUs at 10 queries each.
- **Pre-filter candidates aggressively:** If you retrieve 1000 candidates, use a lightweight model first to cull to 100, then the heavy cross-encoder.
- **Cascade reranking:** Stage 1 (fast bi-encoder) → Stage 2 (small cross-encoder) → Stage 3 (large cross-encoder).
- **ASIC inference:** Deploy on AWS Inferentia or Google TPU for cost-effective batch inference.

## 19. Failure Scenarios
| Failure | Impact | Mitigation |
|---------|--------|------------|
| GPU OOM | Batch fails | Reduce batch size, enable model sharding |
| Model drift | Scores diverge from human judgments | Monitor score distribution, re-fine-tune |
| Token overflow | Documents truncated | Chunk + score each chunk, take max |
| Cold start | No scores for new documents | Use retriever score as fallback |

## 20. Security
- Validate all input text lengths to prevent adversarial long inputs (OOM attacks).
- Rate-limit API per user (reranking is expensive compute).
- Monitor for data poisoning in training data.
- Use signed model artifacts to prevent model substitution.

## 21. Monitoring
- **Score distribution:** Mean, std, min/max of reranker scores — drift signals model degradation.
- **Latency:** p50, p95, p99 per request and per candidate.
- **Throughput:** Candidates reranked per second per GPU.
- **Fallback rate:** How often does the reranker fail → fall back to retriever score?
- **Quality metrics:** NDCG@10, MRR compared to first-stage ranking.

## 22. Interview Questions
| Level | Question | Key Points |
|-------|----------|------------|
| L4 | Why use a cross-encoder instead of a bi-encoder for reranking? | Cross-encoder captures token-level interactions, bi-encoder only sees dot product |
| L4 | How would you speed up cross-encoder reranking? | Batching, FP16, smaller model, ONNX, cascade |
| L5 | Design a reranking system for 1000 candidates | Cascade: light model to 200, heavy model to top-10 |
| L5 | How do you train a cross-encoder reranker? | Positive/negative pairs from click logs or human judgments, binary cross-entropy |
| L5 | How would you evaluate a reranker's impact on RAG quality? | A/B test: answer faithfulness, user satisfaction, retrieval recall@k |

## 23. Cheat Sheet
```python
# Reranking with cross-encoder
from transformers import AutoModelForSequenceClassification, AutoTokenizer

model = AutoModelForSequenceClassification.from_pretrained("cross-encoder/ms-marco-MiniLM-L-6-v2")
tokenizer = AutoTokenizer.from_pretrained("cross-encoder/ms-marco-MiniLM-L-6-v2")
pairs = [(query, doc) for doc in candidates]
inputs = tokenizer(pairs, padding=True, truncation=True, return_tensors="pt")
scores = model(**inputs).logits.squeeze(-1)
top_k = sorted(zip(candidates, scores), key=lambda x: x[1], reverse=True)[:5]
```

## 24. Common Mistakes
- **Reranking too many candidates** — cross-encoder on 1000 docs takes ~10 seconds. Limit to 50-100.
- **Single-document scoring** — not batching. Batch size 32-64 for optimal GPU utilization.
- **Ignoring score calibration** — cross-encoder scores are not probabilities. Use softmax or Platt scaling.
- **Reranking without first-stage filtering** — a cross-encoder cannot replace a retriever; it's too slow.
- **Using different truncation than the model was trained on** — some models expect 512, others 128.
- **Not deduplicating before reranking** — identical docs get identical scores, wasting compute.

## 25. Production Best Practices
1. **Always use a reranker between retriever and LLM** — it's the single highest-leverage quality improvement for RAG.
2. **Batch candidates** during reranker inference — aim for batch_size ≥ 32 for good GPU utilization.
3. **Use FP16 inference** — 2x speedup, 2x memory reduction, negligible quality loss.
4. **Limit first-stage candidates to 50-100** — beyond 100, the reranker's marginal quality gain isn't worth the latency cost.
5. **Truncate documents to the model's max length** — don't feed 4000-token documents into a 512-token model.
6. **Monitor score distribution** — a drift toward 0.5 for all documents means the model is confused.
7. **Cache reranking scores** for identical query-doc pairs within a short time window.
8. **Deploy on GPU** — CPU cross-encoder inference is 10-50x slower and unacceptable for production RAG.
9. **Fine-tune on domain data** — a generic MS MARCO reranker improves 5-10% when fine-tuned on your domain.
10. **Fall back to retriever score** if the reranker crashes — one-stage response is better than no response.
