# Prefix Caching

> Author: **Tamilselvan** · ✉️ tamilselvan.sde@gmail.com · 🔗 [LinkedIn](https://www.linkedin.com/in/tamilselvan-ai/)
>

## 1. What is it?

Prefix caching is an optimization that reuses the KV cache computation for common token prefixes across multiple inference requests. When two requests share the same starting tokens (e.g., system prompt), the KV cache for those tokens is computed once and shared.

**ELI5:** Imagine you're a chef making sandwiches for a catering order of 100 sandwiches. 90 are ham and cheese with the same bread, cheese, and ham. Instead of assembling each sandwich from scratch, you pre-assemble a pile of "base sandwiches" (bread + cheese + ham) and only customize the toppings for each order. Prefix caching does the same for LLM prompts — pre-compute the KV cache for the shared prefix once.

**Technical Definition:** Prefix caching stores the KV cache entries for initial tokens in a data structure (hash table or radix tree) keyed by the token sequence. When a new request arrives, its prefix is matched against cached entries. Matching KV blocks are loaded directly instead of recomputed. The system must handle cache eviction (LRU), copy-on-write for diverging sequences, and block table updates.

## 2. Why do we need it?

**Problem:** In production LLM deployments, every request includes system prompts (500-2000 tokens), few-shot examples (200-1000 tokens), chat history (variable), and tool schemas (100-500 tokens). These prefixes are often identical across many requests. Without caching, every request recomputes the KV cache for these identical prefixes — wasting 40-80% of prefill computation.

**Pain without it:** Consider a chatbot with a 1500-token system prompt serving 100K requests/day. Without prefix caching, 150M tokens of system prompt KV cache are recomputed daily. At $0.01 per 1K tokens (H100), that's $1,500/day wasted on redundant computation. In latency terms, each request's TTFT includes 1500 tokens of prefill they didn't need.

**Why companies use it:** Prefix caching reduces TTFT by 30-70% for requests with shared prefixes, increases throughput by 2-5x for agentic workloads (where tool schemas are shared), and reduces GPU costs by eliminating redundant prefill computation.

## 3. Real-world Example

- **vLLM**: Built-in prefix caching via `--enable-prefix-caching`. Uses hash-based block matching.
- **SGLang**: RadixAttention — a radix tree (prefix tree) structure for KV cache that makes prefix caching the central design principle.
- **Together AI**: Reports 3-5x latency reduction from prefix caching for their agentic workloads.
- **OpenAI**: ChatGPT's system prompt is cached server-side, so user messages start generation almost instantly.
- **Anthropic Claude**: Long system prompts and few-shot examples benefit from prefix caching across concurrent conversations.
- **Google Gemini**: System instructions are cached; only user messages trigger new prefill computation.

## 4. Architecture Diagram (ASCII)

```
Without Prefix Caching:
┌─────────┐     ┌─────────┐     ┌─────────┐
│Request A│────▶│System   │────▶│User msg │────▶ Generate
│         │     │Prompt   │     │         │
└─────────┘     │(compute │     └─────────┘
                │ KV)     │
┌─────────┐     └─────────┘     ┌─────────┐
│Request B│────▶│System   │────▶│User msg │────▶ Generate (REDUNDANT KV)
│         │     │Prompt   │     │         │
└─────────┘     │(compute │     └─────────┘
                │ KV AGAIN)│
                └─────────┘

With Prefix Caching:
┌─────────┐     ┌─────────────────────────────────────────┐
│Request A│────▶│ Cache ── System Prompt KV Blocks         │
│         │     │ ┌───┐ ┌───┐ ┌───┐ ┌───┐ ┌───┐         │
├─────────┤     │ │ S0│ │ S1│ │ S2│ │ S3│ │ S4│         │
│Request B│────▶│ └───┘ └───┘ └───┘ └───┘ └───┘         │
│         │     │                          ↑              │
├─────────┤     │   New: User msg KV        │             │
│Request C│────▶│   ┌───┐ ┌───┐ ┌───┐      │             │
│         │     │   │ U0│ │ U1│ │ U2│      │             │
└─────────┘     │   └───┘ └───┘ └───┘      │             │
                │                          │             │
                │ Everyone shares S0-S4!  │             │
                └──────────────────────────┘             │
```

## 5. Internal Working

**Hash-based prefix caching (vLLM approach):**

1. **Tokenization**: Input tokens are divided into blocks of `block_size` (default 16).
2. **Block hash**: Each block is hashed (using the token IDs and the block's position) to produce a unique content hash.
3. **Cache lookup**: When a sequence needs KV computation, each block's hash is checked against the global cache.
4. **Cache hit**: If found, the physical block address is retrieved from the hash table.
5. **Cache miss**: KV is computed and the block is added to the cache with its hash.
6. **Reference counting**: Shared blocks have ref_count > 1. When a sequence diverges (different next token), copy-on-write creates a private copy.

**Radix-tree prefix caching (SGLang approach):**

1. **Trie insertion**: Token sequences are inserted into a radix tree (compressed trie).
2. **Longest prefix match**: New request finds the longest matching prefix in the tree.
3. **Node sharing**: Common prefixes are shared as tree nodes with KV blocks attached.
4. **Fork/join**: When sequences diverge, tree nodes split. When they converge (e.g., tool call responses), nodes merge.
5. **Eviction**: LRU on leaf nodes. Evicted nodes' KV blocks are freed.

## 6. Production Flow

```
Request arrives → Tokenize
    │
    ▼
Split into blocks of block_size tokens
    │
    ▼
For each block:
    Compute content hash (token_ids + position)
    │
    ▼
    Lookup in hash table
    │
    ├── Cache HIT → Use physical block from cache (skip computation)
    │                 Increment ref_count
    │
    └── Cache MISS → Compute KV on GPU
                      Allocate physical block
                      Store in hash table with ref_count=1
    │
    ▼
For the last block (where new generation starts):
    If user_msg continues beyond prefix:
        Compute remaining KV
    │
    ▼
Start generation from cached + computed KV
```

## 7. HLD (High-Level Design)

```
┌─────────────────────────────────────────────────────────────────┐
│                      Prefix Cache System                          │
│                                                                   │
│  ┌──────────────────────┐  ┌──────────────────────────────┐     │
│  │ Hash Table / Radix   │  │ Block Table                  │     │
│  │ Tree                  │  │ Logical → Physical           │     │
│  │ (Index by content)   │  │ (Per-sequence mapping)       │     │
│  └──────────┬───────────┘  └──────────────┬───────────────┘     │
│             │                             │                      │
│             ▼                             ▼                      │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │                  KV Cache Blocks (GPU)                    │   │
│  │  ┌──────┐ ┌──────┐ ┌──────┐ ┌──────┐ ┌──────┐ ┌──────┐  │   │
│  │  │Hash_A│ │Hash_A│ │Hash_B│ │Hash_B│ │Hash_C│ │      │  │   │
│  │  │ref=2 │ │ref=2 │ │ref=1 │ │ref=1 │ │ref=3 │ │free  │  │   │
│  │  └──────┘ └──────┘ └──────┘ └──────┘ └──────┘ └──────┘  │   │
│  └──────────────────────────────────────────────────────────┘   │
│                                                                   │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │              Eviction Policy (LRU/LFU)                    │   │
│  │  Least recently used cache entries → free blocks          │   │
│  └──────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────┘
```

## 8. LLD (Low-Level Design)

```python
import hashlib
from dataclasses import dataclass, field
from typing import Dict, List, Optional, Tuple
from collections import OrderedDict
import time

@dataclass
class PrefixCache:
    """Hash-based prefix cache for KV blocks."""

    block_size: int = 16
    max_capacity: int = 5000  # Number of blocks to cache

    def __post_init__(self):
        # Hash → physical block mapping
        self.cache: OrderedDict[str, int] = OrderedDict()
        # Physical block → ref_count
        self.ref_counts: Dict[int, int] = {}
        # Physical block → last access time
        self.last_access: Dict[int, float] = {}

    def compute_hash(self, token_ids: List[int], block_index: int) -> str:
        """Compute content hash for a block of tokens."""
        content = str(block_index) + ":" + ",".join(str(t) for t in token_ids)
        return hashlib.sha256(content.encode()).hexdigest()[:16]

    def lookup(self, token_ids: List[int], block_index: int) -> Optional[int]:
        """Look up a block in the cache. Returns physical block ID or None."""
        hash_key = self.compute_hash(token_ids, block_index)
        if hash_key in self.cache:
            phys_block = self.cache[hash_key]
            self.last_access[phys_block] = time.time()
            # Move to end (most recently used)
            self.cache.move_to_end(hash_key)
            return phys_block
        return None

    def insert(
        self, token_ids: List[int], block_index: int, phys_block: int
    ):
        """Insert a newly computed block into the cache."""
        hash_key = self.compute_hash(token_ids, block_index)

        if len(self.cache) >= self.max_capacity:
            # LRU eviction
            evict_key, evict_block = self.cache.popitem(last=False)
            if self.ref_counts[evict_block] == 0:
                # Free physical block
                del self.ref_counts[evict_block]
                del self.last_access[evict_block]
                self.cache[hash_key] = phys_block
            else:
                # Block still in use; cannot evict
                # Try next eviction candidate
                return

        self.cache[hash_key] = phys_block
        self.ref_counts[phys_block] = self.ref_counts.get(phys_block, 0) + 1
        self.last_access[phys_block] = time.time()

    def release(self, phys_block: int):
        """Decrement reference count when a sequence no longer uses a block."""
        if phys_block in self.ref_counts:
            self.ref_counts[phys_block] -= 1
            if self.ref_counts[phys_block] < 0:
                self.ref_counts[phys_block] = 0

    def find_longest_prefix(
        self, token_ids: List[int]
    ) -> Tuple[int, List[int]]:
        """Find the longest cached prefix for a token sequence.
        Returns (matching_block_count, list_of_physical_block_ids)."""
        matched_blocks = 0
        phys_blocks = []

        for block_idx in range(len(token_ids) // self.block_size + 1):
            start = block_idx * self.block_size
            end = min(start + self.block_size, len(token_ids))
            block_tokens = token_ids[start:end]
            if len(block_tokens) == 0:
                break

            phys = self.lookup(block_tokens, block_idx)
            if phys is not None:
                matched_blocks += 1
                phys_blocks.append(phys)
            else:
                break

        return matched_blocks, phys_blocks

    @property
    def hit_rate(self) -> float:
        """Fraction of lookups that were hits (approximate)."""
        if not hasattr(self, '_total_lookups'):
            return 0.0
        return self._hits / self._total_lookups if self._total_lookups > 0 else 0.0
```

## 9. Python Implementation

```python
import asyncio
import torch
from fastapi import FastAPI, HTTPException
from pydantic import BaseModel
from typing import List, Optional
from collections import OrderedDict
import hashlib
import time
import logging

logging.basicConfig(level=logging.INFO)
logger = logging.getLogger("prefix-cache")

app = FastAPI(title="Prefix-Cached Inference Server")

# Simplified KV cache representation
class KVCacheSimulator:
    def __init__(self):
        self.blocks: dict[int, torch.Tensor] = {}
        self.next_id = 0

    def compute(self, tokens: List[int]) -> int:
        """Simulate KV cache computation for a block of tokens."""
        block_id = self.next_id
        self.next_id += 1
        # In real implementation: model.forward on token block
        self.blocks[block_id] = torch.randn(1024)  # Dummy KV
        return block_id

    def release(self, block_id: int):
        self.blocks.pop(block_id, None)

class PrefixCache:
    def __init__(self, block_size: int = 16, max_blocks: int = 1000):
        self.block_size = block_size
        self.max_blocks = max_blocks
        self.hash_to_block: OrderedDict[str, int] = OrderedDict()
        self.block_refs: dict[int, int] = {}
        self.kv_cache = KVCacheSimulator()
        self.hits = 0
        self.misses = 0

    def _hash_block(self, tokens: List[int], pos: int) -> str:
        return hashlib.md5(
            str(pos).encode() + b"|" + str(tokens).encode()
        ).hexdigest()

    def process_tokens(self, tokens: List[int]) -> List[int]:
        """Process tokens, using cache where possible.
        Returns list of KV block IDs (logical)."""
        block_ids = []
        num_blocks = (len(tokens) + self.block_size - 1) // self.block_size

        for i in range(num_blocks):
            start = i * self.block_size
            end = min(start + self.block_size, len(tokens))
            block_tokens = tokens[start:end]
            hash_key = self._hash_block(block_tokens, i)

            if hash_key in self.hash_to_block:
                phys_block = self.hash_to_block[hash_key]
                self.hash_to_block.move_to_end(hash_key)
                self.block_refs[phys_block] += 1
                self.hits += 1
                logger.debug(f"Cache HIT for block {i}")
            else:
                phys_block = self.kv_cache.compute(block_tokens)
                self._evict_if_needed()
                self.hash_to_block[hash_key] = phys_block
                self.block_refs[phys_block] = 1
                self.misses += 1
                logger.debug(f"Cache MISS for block {i}")

            block_ids.append(phys_block)

        return block_ids

    def _evict_if_needed(self):
        while len(self.hash_to_block) >= self.max_blocks:
            hash_key, phys_block = self.hash_to_block.popitem(last=False)
            if self.block_refs.get(phys_block, 0) <= 0:
                self.kv_cache.release(phys_block)
                self.block_refs.pop(phys_block, None)
            else:
                # Block in use, re-add and try next
                self.hash_to_block[hash_key] = phys_block
                self.hash_to_block.move_to_end(hash_key)
                break

    @property
    def hit_rate(self) -> float:
        total = self.hits + self.misses
        return self.hits / total if total > 0 else 0.0

cache = PrefixCache(block_size=16, max_blocks=5000)

class GenerateRequest(BaseModel):
    system_prompt: str = "You are a helpful assistant."
    user_message: str
    max_tokens: int = 128

class GenerateResponse(BaseModel):
    text: str
    cache_hit_rate: float
    prefix_blocks_used: int

@app.post("/generate", response_model=GenerateResponse)
async def generate(request: GenerateRequest):
    try:
        # Simulate tokenization (in production, use real tokenizer)
        system_tokens = list(range(100, 100 + len(request.system_prompt)))
        user_tokens = list(range(200, 200 + len(request.user_message)))
        full_tokens = system_tokens + user_tokens

        # Process with prefix caching
        kv_blocks = cache.process_tokens(full_tokens)

        # Simulate generation (in production, run decoder loop)
        generated = request.user_message[::-1]  # Dummy output

        return GenerateResponse(
            text=generated,
            cache_hit_rate=cache.hit_rate,
            prefix_blocks_used=len(kv_blocks),
        )
    except Exception as e:
        logger.error(f"Generation failed: {e}")
        raise HTTPException(status_code=500, detail=str(e))

@app.get("/cache-stats")
async def cache_stats():
    return {
        "hits": cache.hits,
        "misses": cache.misses,
        "hit_rate": cache.hit_rate,
        "cached_blocks": len(cache.hash_to_block),
        "max_blocks": cache.max_blocks,
    }
```

## 10. Folder Structure

```
prefix_cache/
├── Dockerfile
├── config.yaml
├── serve.py
├── prefix_cache/
│   ├── __init__.py
│   ├── cache.py              # PrefixCache (hash-based)
│   ├── radix_cache.py        # PrefixCache (radix tree)
│   ├── block_manager.py      # Physical block management
│   ├── server.py             # FastAPI server
│   └── config.py
├── k8s/
│   ├── deployment.yaml
│   └── configmap.yaml
├── tests/
│   ├── test_cache.py
│   └── test_prefix_matching.py
└── benchmark.py
```

## 11. Configuration

```yaml
prefix_cache:
  enabled: true
  block_size: 16
  max_cached_blocks: 5000       # Max blocks in cache (evict when exceeded)
  eviction_policy: lru           # lru, lfu, fifo

  # Hash-based (vLLM style)
  hash_algorithm: sha256         # sha256, md5, xxhash
  hash_prefix_len: 16            # Truncate hash to this many chars

  # Radix tree (SGLang style)
  use_radix_tree: false          # Enable radix tree instead of hash table
  max_tree_depth: 512            # Max prefix tree depth

  # Sharing
  max_sharing_per_block: 64      # Max sequences sharing one block
  copy_on_write: true            # Enable copy-on-write for divergent sequences

  # Monitoring
  log_hit_rate_interval: 60      # Seconds between hit rate logs
```

## 12. Flowchart

```
                   ┌──────────────┐
                   │ New Request  │
                   └──────┬───────┘
                          │
                          ▼
                  ┌───────────────┐
                  │ Tokenize:     │
                  │ [sys][few][usr]│
                  └──────┬───────┘
                         │
                         ▼
                  ┌───────────────┐
                  │ Split into    │
                  │ blocks of 16  │
                  └──────┬───────┘
                         │
                         ▼
                  ┌───────────────┐
                  │ For each block│
                  │ Compute hash  │
                  └──────┬───────┘
                         │
              ┌──────────┴──────────┐
              │                     │
         ┌────▼────┐          ┌─────▼─────┐
         │ Hash in │          │ Hash NOT  │
         │ cache?  │          │ in cache  │
         └────┬────┘          └─────┬─────┘
              │                     │
         ┌────▼────┐          ┌─────▼─────┐
         │ Use     │          │ Compute   │
         │ cached  │          │ KV for    │
         │ block   │          │ block     │
         └────┬────┘          └─────┬─────┘
              │                     │
              └──────┬──────────────┘
                     │
                     ▼
              ┌───────────────┐
              │ All blocks    │
              │ processed?    │
              └──────┬───────┘
                     │
              ┌──────┴──────┐
              │             │
         ┌────▼────┐  ┌────▼────┐
         │ No:     │  │ Yes:    │
         │ Continue│  │ Start   │
         │ loop    │  │ decode  │
         └─────────┘  └─────────┘
```

## 13. Sequence Diagram

```
Request A        Prefix Cache       GPU               Request B
    │                │                │                    │
    │───[sys][msg]──▶│                │                    │
    │                │──hash block 0─▶│                    │
    │                │◀──cache miss──│                    │
    │                │──compute KV──▶│                    │
    │                │◀──store──────│                    │
    │                │──hash block 1─▶│                    │
    │                │◀──cache miss──│                    │
    │                │──compute KV──▶│                    │
    │                │◀──store──────│                    │
    │◀──result──────│                │                    │
    │                │                │                    │
    │                │                │    Request B       │
    │                │                │◀───[sys][msg2]────│
    │                │                │                    │
    │                │◀──hash block 0─│                    │
    │                │──cache HIT───▶│ (reuse block 0)    │
    │                │◀──hash block 1─│                    │
    │                │──cache HIT───▶│ (reuse block 1)    │
    │                │◀──hash block 2─│  (msg2, new)      │
    │                │──cache miss──▶│                    │
    │                │──compute KV──▶│                    │
    │                │◀──store──────│                    │
    │                │                │───result─────────▶│
```

## 14. Pros

- **Reduces TTFT by 30-70%** — skip prefill for shared prefixes
- **Increases throughput** — free GPU cycles for more requests
- **Reduces cost** — fewer FLOPs per request = more requests per GPU
- **Transparent** — works without client-side changes
- **Better for agentic workloads** — tool schemas, formatting instructions shared across calls
- **Composable** — works with PagedAttention, continuous batching, tensor parallelism

## 15. Cons

- **Cache management overhead** — hash computation, LRU tracking, copy-on-write
- **Memory overhead** — storing cached blocks uses GPU memory that could hold new sequences
- **Cold start** — cache is empty after deployment; hit rate grows over minutes
- **Stale cache** — if system prompt changes, old blocks become invalid
- **Hash collisions** — though extremely unlikely with SHA-256
- **Uneven benefit** — only helps workloads with shared prefixes; random prompts see no gain

## 16. Alternatives

| Alternative | Approach | When to Use |
|------------|----------|-------------|
| **System prompt injection** | Prepend system prompt at application level | Simple prefix reuse |
| **No caching** | Full prefill every time | Random, diverse prompts |
| **Prompt compression** | Compress (e.g., LLMLingua) before inference | Very long prompts |
| **Speculative prefill** | Predict next tokens during prefill | When compute is cheap |

## 17. Performance Considerations

**Cache hit rate factors:**
```
Shared system prompt length: longer → higher savings
Number of distinct system prompts: fewer → higher hit rate
Variable user prompt length: longer → lower relative hit rate
Cache size: larger → retains more blocks, higher hit rate
Block size: smaller → more granular sharing, but more metadata

Typical hit rates:
- Chatbot (shared system prompt):    60-80%
- Agent (shared tools/schemas):      70-90%
- Batch extraction (same schema):    80-95%
- Random QA (no shared prefix):      0-10%
```

**Memory tradeoff:**
```
Block cache size = hit_rate * avg_daily_prefix_tokens / block_size * block_memory

Example: 100K requests/day, 1500 token prefix, block_size=16, hit_rate=70%
- Blocks saved: 100K * 1500/16 * 0.7 = 6.5M blocks/day
- GPU memory needed at any time: depends on concurrency, not daily volume
- For 100 concurrent requests with 1500 prefix: 100 * 1500/16 = 9375 blocks
- At 8.4MB/block (Llama-3-8B): 9375 * 8.4MB = 78GB — won't fit entirely
- LRU eviction keeps only the most common prefixes
- Rule: allocate 20-30% of total KV cache memory for prefix cache
```

## 18. Scaling to Millions

**For 10M requests/day with prefix caching:**
```
Without prefix caching:
- Prefill tokens/day: 10M × 1500 = 15B tokens
- GPU hours: 15B / 500K tok/s = 30,000 GPU-hours/month
- Cost: ~$90,000/month (H100 at $3/hr)

With 70% prefix caching:
- Recomputed prefill tokens: 10M × 1500 × 0.3 = 4.5B tokens
- GPU hours: 4.5B / 500K = 9,000 GPU-hours/month
- Cost: ~$27,000/month
Savings: ~$63,000/month
```

**Cache warmup strategy:**
1. On deploy, seed cache with common prefixes (synthetic requests)
2. Cache reaches steady state within minutes under load
3. For canary deployments, route traffic gradually to build cache
4. Consider persisting cache across restarts (checkpoint to disk)

## 19. Failure Scenarios

| Failure | Impact | Mitigation |
|---------|--------|------------|
| **Cache stampede** | All requests miss simultaneously after cache flush | Gradual cache re-population, graceful degradation |
| **Hash collision** | Wrong KV blocks reused | 128-bit hash sufficient; verify with token comparison |
| **Stale prefix** | Old system prompt cached → wrong behavior | Cache invalidation on system prompt change (version key) |
| **Memory leak** | Blocks not evicted → OOM over hours | Max block age, periodic full sweep |
| **Block corruption** | Shared block corrupted → all sharing sequences affected | Read-after-write verification for critical blocks |

## 20. Security

| Concern | Mitigation |
|---------|------------|
| **Cross-user data in cache** | Users with different system prompts don't share cache keys. Same key = same content |
| **Cache side channel** | Timing of cache hit reveals prefix length. Pad to fixed prefix length if sensitive |
| **Cache poisoning** | Malicious input producing same hash as legitimate prefix | Content-aware collision detection (verify full token match) |
| **Data leakage via eviction** | Evicted block data remains in GPU memory | Zero blocks on eviction (memzero) |
| **Tenant isolation** | Multi-tenant environments sharing cache | Per-tenant cache namespaces |

## 21. Monitoring

**Key metrics:**
```
pc:hit_rate                    — Fraction of block lookups that hit
pc:miss_rate                   — Fraction of block lookups that miss
pc:blocks_cached               — Number of blocks in cache
pc:cache_memory_bytes          — GPU memory used by cached blocks
pc:blocks_shared               — Blocks with ref_count > 1
pc:avg_sharing_degree          — Average sequences per shared block
pc:evictions_per_second        — Block eviction rate
pc:copy_on_write_count         — CoW operations per second
pc:cache_warmup_seconds        — Time to reach steady-state hit rate
pc:prefix_savings_tokens       — Tokens saved by caching (cumulative)
```

**Grafana panels:**
1. Cache hit rate over time (should stabilize within minutes)
2. Cached blocks vs evictions (lines cross = cache capacity reached)
3. Sharing degree histogram (how many sequences per cached block)
4. Memory savings (tokens not computed × time)
5. Hit rate by prefix length

**Alerts:**
- Hit rate < 20% after 10 minutes → check for prefix diversity
- Eviction rate > 100/s → cache too small; increase capacity
- CoW operations > 50% of cache hits → high prefix divergence
- Cache memory > 80% of KV cache budget → reduce cache allocation

## 22. Interview Questions

**Q1: How does prefix caching differ from traditional KV cache reuse?**
*A: Traditional KV cache is per-sequence: each sequence's KV is stored contiguously and freed when the sequence ends. Prefix caching stores KV blocks by content, making them shareable across sequences. Two sequences with the same 1000-token system prompt share those 1000 KV cache entries instead of each storing duplicate copies.*

**Q2: What happens when two requests share a prefix but then diverge?**
*A: Copy-on-write (CoW). Both requests initially share the same physical KV blocks for the common prefix. When they diverge (different user messages), the block where divergence starts is copied to a new physical block owned exclusively by the diverging sequence. The remaining shared prefix blocks stay shared until the sequences' lifetimes end or they diverge further.*

**Q3: How would you implement cache invalidation when the system prompt changes?**
*A: Include a version identifier in the cache key. E.g., hash = sha256(version + token_ids). When the system prompt changes, increment the version. Old cached blocks with the old version are naturally cache-missed. The old blocks are eventually evicted by LRU. This also enables A/B testing of different system prompts without interference.*

**Q4: What's the memory tradeoff of prefix caching — doesn't storing cached blocks reduce available memory for new sequences?**
*A: Yes, there's a tradeoff. Cached blocks use memory that could otherwise serve new sequences. The benefit depends on hit rate. If 30% of KV cache memory is used for prefix caching but achieves 70% hit rate, the net effect is positive: 0.3 * cache_size saved in compute per request. The optimal allocation is typically 20-30% of total KV cache budget for prefix storage, tuned via A/B testing.*

## 23. Cheat Sheet

```bash
# Enable prefix caching in vLLM
python -m vllm.entrypoints.openai.api_server \
    --model meta-llama/Llama-3.2-3B-Instruct \
    --enable-prefix-caching \
    --max-num-seqs 256

# Verify caching is working
# Monitor: /metrics endpoint
curl http://localhost:8000/metrics | grep vllm:prefix_cache

# Expected metrics:
# vllm:prefix_cache_hit_rate
# vllm:prefix_cache_blocks_cached

# SGLang uses RadixAttention (built-in prefix caching)
python -m sglang.srt.server \
    --model meta-llama/Llama-3.2-3B-Instruct \
    --radix-cache-size 4000000000  # 4GB for prefix cache

# Key concept: maximize prefix sharing
# - Use consistent system prompts
# - Canonicalize inputs (trim whitespace)
# - Sort tool schemas alphabetically
# - Use deterministic tokenization
```

## 24. Common Mistakes

1. **Not canonicalizing inputs** — trailing whitespace, different newline characters, or unordered JSON keys create different token sequences, reducing prefix sharing by 50-80%. Always normalize inputs.

2. **Using variable few-shot examples** — randomly selected few-shot examples create many unique prefixes. Fix the example selection or use deterministic sampling.

3. **Over-allocating cache** — dedicating 50%+ of KV cache to prefix storage leaves too little for active sequences. Start with 20-25% and monitor.

4. **Ignoring cold start** — right after deployment, the cache is empty. This causes a latency spike for the first few minutes. Pre-warm with synthetic requests.

5. **Not invalidating stale prefixes** — when you change the system prompt, old cached blocks remain (they just won't be hit due to different hashes). This wastes memory until eviction — use versioned keys.

6. **Using block_size that's too large** — with block_size=64, a 100-token unique suffix after a shared prefix wastes 28 tokens of recomputation. Smaller blocks (16) give finer-grained sharing.

## 25. Production Best Practices

1. **Canonicalize all system prompts** — strip whitespace, normalize line endings, sort JSON keys, use deterministic tokenization. This maximizes prefix matching.

2. **Version your system prompts** — include a version string in the cache key so that prompt changes don't cause correctness issues. Old prompts' blocks naturally expire via LRU.

3. **Pre-warm the cache** — on deployment, send a batch of synthetic requests with your most common system prompts and tool schemas. This ensures the cache has hit rate > 50% from the first real request.

4. **Monitor hit rate by prefix type** — instrument separate hit rate metrics for system prompt, few-shot examples, and user messages. This helps identify which prefix layers need optimization.

5. **Use block_capacity alerts** — set up an alert when cache block usage exceeds 80% of budget. This signals it's time to either increase capacity or reduce prefix diversity.

6. **Implement graceful degradation** — if cache hit rate drops below 10% (e.g., after a system prompt change), consider disabling prefix caching entirely until the new prompt accumulates sufficient entries.

7. **A/B test cache allocation** — run parallel deployments with 20%, 30%, and 40% of KV cache budget for prefix storage. Measure overall throughput (not just hit rate) to find the optimum.

8. **Set version in application middleware** — have the application layer include a cache version header. The inference server uses this to key the cache, enabling per-tenant isolation and prompt-version testing.

9. **Combine with prompt compression** — for extremely long prefixes (>4000 tokens), apply LLMLingua or similar compression before caching. The compressed prefix still benefits from caching but uses fewer blocks.

10. **Defragment periodically** — over time, shared blocks become fragmented as sequences with different lengths share prefixes. Periodically compact the cache to maximize contiguous free space.
