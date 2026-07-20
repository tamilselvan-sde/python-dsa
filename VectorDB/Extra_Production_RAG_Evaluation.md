# Detailed Production RAG Evaluation

> Author: **Tamilselvan** · ✉️ tamilselvan.sde@gmail.com · 🔗 [LinkedIn](https://www.linkedin.com/in/tamilselvan-ai/)
>


### Creating a RAG Test Set

```python
def create_rag_test_set(
    documents: List[str],
    num_questions: int = 200,
    llm=None
) -> List[Dict]:
    """Generate question-answer pairs from documents."""
    
    test_set = []
    docs_per_question = max(1, num_questions // len(documents))
    
    for doc in documents[:num_questions // docs_per_question]:
        prompt = f"""
Given this document, generate {docs_per_question} question-answer pairs.
Questions should be answerable ONLY from the document content.
Include a mix of:
- Factual questions (what, who, when, where)
- Explanatory questions (how, why)
- Comparative questions

Document:
{doc}

Output as JSON array:
[
    {{"question": "...", "answer": "...", "difficulty": "easy|medium|hard"}}
]
"""
        response = llm.invoke(prompt)
        pairs = json.loads(response)
        
        for pair in pairs:
            test_set.append({
                "question": pair["question"],
                "expected_answer": pair["answer"],
                "source_doc": doc,
                "difficulty": pair.get("difficulty", "medium"),
            })
    
    return test_set
```

### Automated RAG Pipeline Testing

```python
class RAGEvaluator:
    def __init__(self, rag_pipeline, test_set, llm_judge):
        self.pipeline = rag_pipeline
        self.test_set = test_set
        self.judge = llm_judge
    
    def run_evaluation(self) -> dict:
        results = []
        
        for i, test_case in enumerate(self.test_set):
            # Run pipeline
            response = self.pipeline.answer(test_case["question"])
            
            # Evaluate
            eval_result = self.judge.evaluate(
                question=test_case["question"],
                expected_answer=test_case["expected_answer"],
                actual_answer=response["answer"],
                retrieved_context=response.get("context", ""),
            )
            
            results.append({
                **test_case,
                **response,
                **eval_result,
            })
        
        return self._aggregate(results)
    
    def _aggregate(self, results: List[Dict]) -> dict:
        easy = [r for r in results if r["difficulty"] == "easy"]
        medium = [r for r in results if r["difficulty"] == "medium"]
        hard = [r for r in results if r["difficulty"] == "hard"]
        
        def avg_score(items):
            return np.mean([
                r.get("faithfulness_score", 0) 
                for r in items
            ]) if items else 0
        
        return {
            "overall": avg_score(results),
            "by_difficulty": {
                "easy": avg_score(easy),
                "medium": avg_score(medium),
                "hard": avg_score(hard),
            },
            "retrieval_recall": np.mean([
                r.get("retrieval_recall", 0) for r in results
            ]),
            "latency_p50": np.percentile(
                [r.get("latency", 0) for r in results], 50
            ),
            "latency_p95": np.percentile(
                [r.get("latency", 0) for r in results], 95
            ),
            "total_tests": len(results),
            "failures": len([
                r for r in results 
                if r.get("faithfulness_score", 0) < 0.7
            ]),
        }
```

---

