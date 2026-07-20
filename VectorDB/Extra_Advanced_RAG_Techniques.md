# Advanced RAG Techniques

> Author: **Tamilselvan** · ✉️ tamilselvan.sde@gmail.com · 🔗 [LinkedIn](https://www.linkedin.com/in/tamilselvan-ai/)
>


### Contextual Retrieval (Anthropic)

**Contextual retrieval** enriches chunks with surrounding document context before embedding.

```python
def contextual_chunk(chunk, document, window=500):
    """Add document-level context to each chunk."""
    return f"""
Document: {document[:window]}
This chunk is from section: {chunk.section_title}
---
{chunk.text}
"""
```

**Why it works:** Chunks alone often lack context. "The model achieves 92% accuracy" — which model? The context prefix answers this.

---

### Step-Back Prompting

```python
def step_back_query(question):
    """Generate a broader 'step-back' question for better retrieval."""
    step_back = llm.invoke(f"""
Original question: {question}
Generate a broader, more conceptual question that would help answer the original.
Example: Original: "What was the GDP growth in Q3 2024?"
         Step-back: "What economic trends emerged in 2024?"
Original question: {question}
Step-back question:""")
    
    # Retrieve using both original and step-back query
    original_results = retrieve(question)
    stepback_results = retrieve(step_back)
    
    # Merge and deduplicate
    return merge_results(original_results, stepback_results)
```

---

### Fusion Retrieval with Query Decomposition

```python
def decompose_and_fuse(complex_question, retriever, k=5):
    """Break complex question into sub-questions, retrieve each, fuse."""
    sub_questions = llm.invoke(f"""
Break this question into simpler sub-questions:
{complex_question}

Output as JSON array of strings.
""")
    sub_questions = json.loads(sub_questions)
    
    all_results = []
    for sq in sub_questions:
        results = retriever.retrieve(sq, top_k=k)
        all_results.extend(results)
    
    # Fuse and deduplicate
    fused = fuse_results(all_results, method="rrf")
    return fused[:k]
```

---

