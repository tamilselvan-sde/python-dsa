# Query Rewriting / Expansion

## 1. What is it?
**ELI5:** Imagine you ask a friend "Where's the thing?" and they have no idea what you mean. But if you first clarify your own question — "Where did I put my car keys?" — they can help. Query rewriting is like that: it takes a vague, short, or ambiguous user query and turns it into a clearer, more search-friendly version before looking up documents.

**Technical Definition:** Query rewriting is a preprocessing step in RAG that transforms the raw user query into one or more optimized queries before retrieval. This can involve expansion (adding synonyms or related terms), decomposition (splitting complex queries), disambiguation (resolving pronouns), or reformulation (rephrasing for better retrieval match). It's typically done by a lightweight LLM call or a dedicated model like T5 or GPT-3.5.

## 2. Why do we need it?
- **Short queries lose information:** Users type "reset password" — too short for dense retrieval to find the right document. Rewritten: "How do I reset my account password on the web platform?"
- **Vocabulary mismatch:** Users say "fix car" but documents say "automotive repair." Query expansion bridges the gap.
- **Ambiguous pronouns:** "How does it work with the API?" — what is "it"? A rewritten query resolves the referent.
- **Complex multi-part questions:** "What are the pricing plans and do they include support?" → split into two focused queries.
- **Conversational context:** In multi-turn chat, each turn references previous context. Query rewriting incorporates that context.

## 3. Real-world Example
- **Google Search:** Google's Query Understanding system automatically expands "apple" to "apple fruit nutrition" or "apple inc stock" based on context, user history, and trending data.
- **OpenAI GPTs + RAG:** When querying a GPT with knowledge base retrieval, OpenAI sends the rewritten query (incorporating conversation history) rather than the raw user message.
- **LlamaIndex's QueryTransform:** Built-in query rewriting pipeline: HyDE (Hypothetical Document Embedding), query expansion, and rephrasing for better retrieval.
- **LangChain's MultiQueryRetriever:** Takes a user query, uses an LLM to generate 5 variations, retrieves for each, and combines results. Handles edge cases where one phrasing fails.
- **Elasticsearch's query expansion:** Uses synonym graphs to expand "car" → "car OR automobile OR vehicle" during query time.

## 4. Architecture Diagram (ASCII)
```
+----------+
| Raw      |
| Query    |
+----+-----+
     |
     v
+------------------+
| Query Rewriter   |
| (LLM / T5 model) |
+------------------+
     |
     +---> Expansion: "reset password" → "reset password account login forgot credentials"
     |
     +---> Decomposition: "pricing and support" → ["What are the pricing plans?", "What support options exist?"]
     |
     +---> Rephrasing: "how do i do the thing" → "How do I reset my password using the web interface?"
     |
     +---> Contextualization: "and its API?" → "How does the Stripe payment API work?"
     |
     v
+------------------+
| Enhanced Query   |
| → Retriever     |
+------------------+
```

## 5. Internal Working
1. **Query classification** (optional but recommended): Determine query type (factoid, procedural, comparison, navigational) to select the right rewriting strategy.
2. **Context assembly** (conversational): Extract relevant conversation history — last 2-3 turns — to resolve pronouns and references.
3. **Rewriting generation:** An LLM (or T5 fine-tuned on query-doc pairs) generates the rewritten query. Prompt template:
   ```
   Given a user query and conversation history, generate a self-contained search query that would retrieve the most relevant documents. Query: {user_query}. History: {history}. Rewritten query:
   ```
4. **Expansion (alternative approach):** Use a thesaurus or WordNet to add synonyms. Or use an LLM to generate 3-5 paraphrases.
5. **Validation:** Check that rewritten query is non-empty, not too long, and doesn't introduce hallucinated terms.

## 6. Production Flow
```
User: "How do I reset it?"
    |
    v
[Get conversation history: previous turn was "I forgot my admin password"]
    |
    v
[Query Rewriter LLM]
    Prompt: "Rewrite for search: Given history 'I forgot my admin password', rewrite 'How do I reset it?' as a self-contained search query"
    Output: "How do I reset my admin password on the platform?"
    |
    v
[Rewrite Validation: length check, token count]
    |
    v
[Enhanced query goes to retriever]
```

## 7. HLD
```
                    +------------------+
                    | Chat History     |
                    | Store (Redis)    |
                    +--------+---------+
                             |
+----------+      +---------v---------+
| User     |------> Query Rewriter    |
| Query    |      | Service           |
+----------+      +---------+---------+
                             |
              +--------------+--------------+
              |              |              |
    +---------v--------+ +---v------------+ +---v------------+
    | Expansion        | | Decomposition  | | Rephrasing     |
    | (synonym graph,  | | (multi-query   | | (LLM-based)    |
    |  LLM expansion)  | |  splitter)     | |                |
    +------------------+ +----------------+ +----------------+
              |              |              |
              +------+-------+------+------+
                     |              |
              +------v------+  +---v----------+
              | Query 1    |  | Query 2...N  |
              +------+------+  +----+---------+
                     |              |
                     v              v
              +----------------------------------+
              | Multi-Query Retriever (fuse results)|
              +----------------------------------+
```

## 8. LLD
```python
from dataclasses import dataclass
from typing import List, Optional
import asyncio

@dataclass
class ConversationTurn:
    user_query: str
    assistant_response: str

@dataclass
class RewrittenQuery:
    original: str
    rewritten: str
    strategy: str  # expansion | decomposition | rephrasing | contextualization

class QueryRewriter:
    def __init__(self, llm_client, max_context_turns: int = 3):
        self.llm = llm_client
        self.max_context_turns = max_context_turns

    async def rewrite(
        self, query: str, history: Optional[List[ConversationTurn]] = None
    ) -> RewrittenQuery:
        if history and len(history) > 0:
            return await self._contextualize(query, history)
        if len(query.split()) < 4:
            return await self._expand(query)
        return RewrittenQuery(original=query, rewritten=query, strategy="none")

    async def _expand(self, query: str) -> RewrittenQuery:
        prompt = (
            f"Expand this short search query with related terms for better "
            f"document retrieval. Query: '{query}'. Output only the expanded query."
        )
        expanded = await self.llm.generate(prompt)
        return RewrittenQuery(original=query, rewritten=expanded, strategy="expansion")

    async def _contextualize(
        self, query: str, history: List[ConversationTurn]
    ) -> RewrittenQuery:
        context = "\n".join(
            f"User: {t.user_query}\nAssistant: {t.assistant_response}"
            for t in history[-self.max_context_turns:]
        )
        prompt = (
            f"Conversation history:\n{context}\n\n"
            f"Rewrite this follow-up query as a self-contained search query "
            f"that doesn't need context. Query: '{query}'\nRewritten:"
        )
        rewritten = await self.llm.generate(prompt)
        return RewrittenQuery(
            original=query, rewritten=rewritten, strategy="contextualization"
        )

    async def decompose(self, query: str) -> List[RewrittenQuery]:
        prompt = (
            f"Split this complex query into separate focused search queries. "
            f"Query: '{query}'. Output one query per line."
        )
        result = await self.llm.generate(prompt)
        sub_queries = [q.strip() for q in result.split("\n") if q.strip()]
        return [
            RewrittenQuery(original=query, rewritten=sq, strategy="decomposition")
            for sq in sub_queries
        ]
```

## 9. Python Implementation
```python
from typing import List, Optional

from fastapi import FastAPI
from openai import AsyncOpenAI
from pydantic import BaseModel, Field

app = FastAPI(title="Query Rewriting Service")

class RewriteRequest(BaseModel):
    query: str = Field(..., min_length=1)
    history: Optional[List[dict]] = Field(default=None, max_length=10)
    strategy: str = Field(default="auto", pattern=r"^(auto|expand|decompose|rephrase)$")

class RewriteResponse(BaseModel):
    original: str
    rewritten: List[str]  # one or more rewritten queries
    strategy: str
    latency_ms: float

class QueryRewriterService:
    def __init__(self, model: str = "gpt-4o-mini"):
        self.client = AsyncOpenAI()
        self.model = model

    async def rewrite(self, query: str, history: Optional[List[dict]] = None) -> List[str]:
        messages = [{"role": "system", "content": (
            "You are a query rewriting assistant. Rewrite the user's query "
            "to be a self-contained, detailed search query optimized for "
            "retrieval. If history is provided, use it to resolve pronouns "
            "and references. Output ONLY the rewritten query, nothing else."
        )}]

        if history:
            for turn in history[-3:]:
                messages.append({"role": "user", "content": turn.get("user", "")})
                messages.append({"role": "assistant", "content": turn.get("assistant", "")})

        messages.append({"role": "user", "content": f"Rewrite for search: {query}"})

        resp = await self.client.chat.completions.create(
            model=self.model,
            messages=messages,
            temperature=0.1,
            max_tokens=256,
        )

        rewritten = resp.choices[0].message.content.strip()
        return [rewritten] if rewritten else [query]

    async def decompose(self, query: str) -> List[str]:
        resp = await self.client.chat.completions.create(
            model=self.model,
            messages=[{"role": "user", "content": (
                f"Split this query into separate focused search queries, "
                f"one per line. Query: {query}"
            )}],
            temperature=0.1,
        )
        content = resp.choices[0].message.content.strip()
        return [q.strip() for q in content.split("\n") if q.strip()]

    async def expand(self, query: str) -> List[str]:
        resp = await self.client.chat.completions.create(
            model=self.model,
            messages=[{"role": "user", "content": (
                f"Expand this query with synonyms and related terms. "
                f"Return ONLY the expanded query. Query: {query}"
            )}],
            temperature=0.1,
        )
        return [resp.choices[0].message.content.strip()]

rewriter = QueryRewriterService()

@app.post("/rewrite", response_model=RewriteResponse)
async def rewrite(request: RewriteRequest):
    import time
    start = time.time()

    if request.strategy == "expand":
        rewritten = await rewriter.expand(request.query)
    elif request.strategy == "decompose":
        rewritten = await rewriter.decompose(request.query)
    else:
        rewritten = await rewriter.rewrite(request.query, request.history)

    return RewriteResponse(
        original=request.query,
        rewritten=rewritten,
        strategy=request.strategy,
        latency_ms=(time.time() - start) * 1000,
    )
```

## 10. Folder Structure
```
query-rewriting/
├── src/
│   ├── __init__.py
│   ├── main.py
│   ├── rewriter.py
│   ├── strategies/
│   │   ├── __init__.py
│   │   ├── expansion.py
│   │   ├── decomposition.py
│   │   ├── rephrasing.py
│   │   └── contextualization.py
│   ├── classification/
│   │   ├── __init__.py
│   │   └── intent.py
│   └── models/
│       └── schemas.py
├── tests/
├── deploy/
└── pyproject.toml
```

## 11. Configuration
```yaml
query_rewriting:
  enabled: true
  strategy: auto  # auto | expand | decompose | rephrase
  llm:
    model: gpt-4o-mini
    temperature: 0.1
    max_tokens: 256
  expansion:
    max_synonyms: 3
    use_wordnet: false
  decomposition:
    max_sub_queries: 5
  contextualization:
    max_history_turns: 3
  caching:
    enabled: true
    ttl_seconds: 3600
```

## 12. Flowchart
```
Start
  |
  v
[Receive Raw Query + History]
  |
  v
[Classify Query Type]
  |--------|-----------|-----------|
  v        v           v           v
[Short]  [Ambiguous] [Complex]   [Well-formed]
  |        |           |           |
  v        v           v           v
[Expand] [Contextualize] [Decompose] [Pass through]
  |        |           |           |
  +--------+-----------+-----+-----+
                              |
                              v
                    [Validate Rewritten Query]
                              |
                     +--------+--------+
                     |                 |
                     v                 v
                [Single Query]   [Multiple Queries]
                                     |
                                     v
                              [Search + Fuse Results]
                              |
                              v
                             End
```

## 13. Sequence Diagram
```
User       ChatSvc     Rewriter      LLM       Retriever
  |          |            |           |           |
  |--Query-->|            |           |           |
  |          |--history-->|           |           |
  |          |            |           |           |
  |          |--rewrite------------->|           |
  |          |            |           |           |
  |          |<--rewritten------------|           |
  |          |            |           |           |
  |          |            |--search-------------->|
  |          |            |           |           |
  |          |<--results--------------|           |
  |<--Response             |           |           |
```

## 14. Pros
- Significantly improves retrieval recall (10-30% improvement in recall@k)
- Makes the RAG system robust to poorly phrased queries
- Enables conversational RAG with context resolution
- Decomposition handles complex multi-faceted questions elegantly
- Simple to implement — one LLM call with a good prompt

## 15. Cons
- Adds an LLM call = extra latency (200-500ms) and cost
- Can introduce hallucinated terms that don't match any documents
- Over-expansion dilutes precision — too many terms = noisy search
- Decomposition multiplies retrieval cost (N queries instead of 1)
- Strategy selection is itself a classification problem

## 16. Alternatives
| Method | Description |
|--------|-------------|
| Raw query (no rewriting) | Simplest, no extra latency or cost |
| HyDE (Hypothetical Document Embedding) | Generate a hypothetical ideal document, embed that instead |
| Query2Doc | Expand query into a full passage, then search |
| Step-back prompting | Abstract the query to a more general form before searching |
| Pseudo-relevance feedback | Take top-3 retrieved results, extract terms, re-query |

## 17. Performance Considerations
- **LLM call latency:** 200-500ms for GPT-4o-mini. Use a faster model or cache frequent rewrites.
- **Cache rewritten queries:** Key = (original_query, last_history_turn). TTL = 1 hour.
- **Batch decomposition:** Run sub-queries in parallel with asyncio.gather.
- **Streaming:** If rewriting is the bottleneck, stream the rewritten query to retrieval while generating more.
- **Fallback:** If the LLM rewrite fails or times out, use the original query.

## 18. Scaling to Millions
- **Dedicated rewrite model:** Fine-tune a small T5 model (220M params) on (original query, gold rewritten query) pairs. Runs in 20ms instead of 200ms.
- **Cache at the rewrite level:** 80% of queries are repeated. Cache rewritten versions.
- **Pre-compute expansions:** For known entities and terms, pre-compute synonym sets and expand at query time with a lookup, not an LLM call.
- **Batch rewrite:** Process queries in batches of 8-16 for better throughput.

## 19. Failure Scenarios
| Failure | Impact | Mitigation |
|---------|--------|------------|
| LLM rewrites to nonsense | Retrieval fails, bad answer | Validate: length, token overlap with original |
| Decomposition explodes | 50 sub-queries | Hard limit max_sub_queries = 5 |
| Contextualization invents facts | Hallucinated answers | Strip entities not in original query or history |
| Timeout | Query delay | Fallback to original query |

## 20. Security
- Sanitize query input before sending to LLM (prompt injection defense).
- Don't rewrite queries that contain PII or secrets — pass them through unchanged.
- Validate rewritten output: check for jailbreak attempts, prompt leaks.
- Log original + rewritten pairs for audit trail.

## 21. Monitoring
- **Rewrite latency:** p50, p95, p99
- **Rewrite strategies used:** Distribution of expand vs decompose vs rephrase vs none
- **Term overlap:** % of terms shared between original and rewritten query (should be >50%)
- **Retrieval improvement:** recall@k with vs without rewriting on golden set
- **Cache hit rate:** Query rewrite cache
- **Failure rate:** % of rewrites that fall back to original

## 22. Interview Questions
| Level | Question | Key Points |
|-------|----------|------------|
| L3 | Why do we need query rewriting in RAG? | Users write short/vague queries; retrieval works better with detailed queries |
| L4 | Explain HyDE and how it differs from query expansion | HyDE generates a hypothetical document, not an expanded query |
| L4 | How would you handle multi-turn conversation queries? | Contextualization: inject history into the rewrite prompt |
| L5 | Design a query rewriting system for millions of queries per day | Cache + fine-tuned small model + fallback to original |
| L5 | How do you evaluate whether query rewriting is helping? | A/B test: recall@k with vs without, final answer quality |

## 23. Cheat Sheet
```python
# Query rewriting with LLM in 5 lines
prompt = f"Rewrite for search: '{query}'"
rewritten = await llm.chat(messages=[{"role": "user", "content": prompt}])
# Or multi-query:
prompt = f"Generate 5 search queries for: '{query}'"
queries = (await llm.chat(...)).split("\n")
```

## 24. Common Mistakes
- **Rewriting well-formed queries** — not every query needs rewriting. Classify first.
- **Over-expansion** — "car repair" becomes "car automobile vehicle motor transport fix" — too noisy.
- **Losing intent** — "I hate the new UI" should not be rewritten to "How to use the new UI features."
- **Not validating rewritten output** — LLMs can hallucinate entities that don't exist in the knowledge base.
- **Ignoring history staleness** — using 10-turn-old context to resolve a pronoun is worse than no context.
- **No caching** — same query rewritten identically every time. Cache aggressively.

## 25. Production Best Practices
1. **Classify before rewriting** — well-formed queries pass through; only rewrite short, ambiguous, or complex ones.
2. **Cache rewritten queries** — keyed by (original_query, last_history_turn) with 1-hour TTL.
3. **Validate rewritten output** — check it's non-empty, shares >50% tokens with original, and doesn't introduce off-topic entities.
4. **Use a fast, fine-tuned model for rewriting** — small T5 or GPT-4o-mini, not the full RAG LLM.
5. **Set max decomposition to 3-5 sub-queries** — more than that and retrieval cost explodes.
6. **Fall back to the original query** if the rewriter fails, times out, or produces nonsense.
7. **Log original + rewritten + retrieval results** for debugging and quality analysis.
8. **Monitor recall improvement** — regularly measure recall@k with rewriting vs without to ensure it's still helping.
9. **Don't rewrite PII queries** — queries containing emails, SSNs, or API keys should pass through raw.
10. **Include the rewritten query in the LLM context** — the generation LLM should know it's answering a reformulated query.
