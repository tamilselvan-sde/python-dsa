# Advanced RAG Evaluation

> Author: **Tamilselvan** · ✉️ tamilselvan.sde@gmail.com · 🔗 [LinkedIn](https://www.linkedin.com/in/tamilselvan-ai/)
>


### RAG Evaluation Metrics

```mermaid
graph TB
    subgraph "RAG Quality Metrics"
        A[Retrieval Quality] --> A1[Recall@k]
        A --> A2[Precision@k]
        A --> A3[MRR]
        A --> A4[NDCG]
        
        B[Generation Quality] --> B1[Faithfulness]
        B --> B2[Answer Relevance]
        B --> B3[Context Precision]
        B --> B4[Hallucination Rate]
        
        C[End-to-End] --> C1[Answer Correctness]
        C --> C2[Toxicity]
        C --> C3[Bias]
        C --> C4[User Satisfaction]
    end
```

### Retrieval Evaluation Implementation

```python
import numpy as np
from typing import List, Dict, Tuple
from dataclasses import dataclass

@dataclass
class RetrievalMetrics:
    recall_at_k: float
    precision_at_k: float
    mrr: float  # Mean Reciprocal Rank
    ndcg: float  # Normalized Discounted Cumulative Gain

def evaluate_retrieval(
    queries: List[str],
    relevant_docs: List[List[str]],  # Ground truth doc IDs per query
    retrieved_docs: List[List[str]],  # Retrieved doc IDs per query
    k_values: List[int] = [1, 3, 5, 10, 20]
) -> Dict[int, RetrievalMetrics]:
    
    results = {}
    
    for k in k_values:
        recalls = []
        precisions = []
        mrrs = []
        ndcgs = []
        
        for q_rel, q_ret in zip(relevant_docs, retrieved_docs):
            retrieved_k = q_ret[:k]
            relevant_set = set(q_rel)
            
            # Recall@k
            hits = len(set(retrieved_k) & relevant_set)
            recall = hits / len(relevant_set) if relevant_set else 0
            recalls.append(recall)
            
            # Precision@k
            precision = hits / k
            precisions.append(precision)
            
            # MRR
            mrr = 0
            for rank, doc_id in enumerate(retrieved_k, 1):
                if doc_id in relevant_set:
                    mrr = 1.0 / rank
                    break
            mrrs.append(mrr)
            
            # NDCG
            dcg = 0
            idcg = 0
            for i, doc_id in enumerate(retrieved_k, 1):
                if doc_id in relevant_set:
                    dcg += 1 / np.log2(i + 1)
            for i in range(min(k, len(q_rel))):
                idcg += 1 / np.log2(i + 2)
            ndcg = dcg / idcg if idcg > 0 else 0
            ndcgs.append(ndcg)
        
        results[k] = RetrievalMetrics(
            recall_at_k=float(np.mean(recalls)),
            precision_at_k=float(np.mean(precisions)),
            mrr=float(np.mean(mrrs)),
            ndcg=float(np.mean(ndcgs)),
        )
    
    return results

# Usage
metrics = evaluate_retrieval(queries, ground_truth, retrieved)
for k, m in metrics.items():
    print(f"@{k}: Recall={m.recall_at_k:.3f}, "
          f"Precision={m.precision_at_k:.3f}, "
          f"MRR={m.mrr:.3f}, NDCG={m.ndcg:.3f}")
```

### Faithfulness Evaluation

```python
class FaithfulnessEvaluator:
    """Check if LLM answer is supported by retrieved context."""
    
    def __init__(self, llm):
        self.llm = llm
    
    def evaluate(self, question: str, answer: str, context: str) -> dict:
        prompt = f"""
Context: {context}

Question: {question}

Answer: {answer}

Evaluate the answer for faithfulness to the context.
Consider:
1. Does every claim in the answer appear in the context?
2. Are there any unsupported statements?
3. Are there any contradictions with the context?

Respond in JSON format:
{{
    "faithfulness_score": 0.0-1.0,
    "supported_claims": ["claim 1", "claim 2"],
    "unsupported_claims": ["claim 3"] or [],
    "contradictions": ["contradiction"] or [],
    "explanation": "brief explanation"
}}
"""
        response = self.llm.invoke(prompt)
        return json.loads(response)
    
    def batch_evaluate(self, samples: List[Dict]) -> dict:
        scores = []
        all_unsupported = []
        
        for sample in samples:
            result = self.evaluate(
                sample["question"],
                sample["answer"],
                sample["context"]
            )
            scores.append(result["faithfulness_score"])
            if result["unsupported_claims"]:
                all_unsupported.append({
                    "question": sample["question"],
                    "unsupported": result["unsupported_claims"]
                })
        
        return {
            "mean_faithfulness": np.mean(scores),
            "std_faithfulness": np.std(scores),
            "min_faithfulness": min(scores),
            "unsupported_examples": all_unsupported[:5]
        }
```

---

