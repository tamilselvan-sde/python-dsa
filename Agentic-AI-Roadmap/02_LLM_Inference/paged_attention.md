# PagedAttention

## 1. What is it?

PagedAttention is a memory management technique for transformer inference that organizes the KV cache into fixed-size blocks and uses a block table to map logical cache indices to physical memory locations — analogous to how operating systems manage virtual memory with pages.

**ELI5:** Imagine you're writing a long letter on a scroll. Normally, you'd need a contiguous section of the scroll long enough for the whole letter. PagedAttention is like using sticky notes instead — you write on individual notes that can be scattered across your desk, and you keep an index of which note comes next. You never waste space, and you can easily add or remove notes.

**Technical Definition:** PagedAttention divides the KV cache into blocks of fixed size (typically 16 tokens). During attention computation, the block table translates logical block IDs (per sequence) to physical block addresses (in GPU memory). This enables non-contiguous KV cache storage, eliminates internal fragmentation, and allows efficient memory sharing for sequences with common prefixes.

## 2. Why do we need it?

**Problem:** Traditional KV cache allocation reserves a contiguous block of GPU memory per sequence based on the maximum possible sequence length. For a 70B model with 8K context, this means reserving ~2GB per sequence. In practice, most sequences are much shorter (100-500 tokens), wasting 60-80% of KV cache memory.

**Pain without it:** Without PagedAttention, production inference systems suffer from:
- **Internal fragmentation**: 60-80% of KV cache memory is allocated but unused
- **External fragmentation**: freed contiguous blocks may not be reusable for different-sized sequences
- **Inability to share prefixes**: system prompts duplicated across every sequence's KV cache

**Why companies use it:** PagedAttention enables 2-4x more sequences per GPU, directly reducing inference costs. Combined with continuous batching, it delivers up to 24x throughput improvement over naive implementations.

## 3. Real-world Example

- **vLLM**: Origin of PagedAttention. All inference in vLLM uses it. Achieves 24x throughput over HuggingFace Transformers.
- **TensorRT-LLM**: NVIDIA's implementation called "inflight batching with Paged KV cache" since TensorRT-LLM 0.5.
- **HuggingFace TGI**: Added PagedAttention support (called "paged attention" internally) in version 1.4.0.
- **OpenAI**: The paper's authors cite inspiration from OS memory management; OpenAI's internal inference likely uses similar techniques.
- **Together AI**: Production deployment uses PagedAttention across thousands of GPUs.

## 4. Architecture Diagram (ASCII)

```
Traditional Contiguous KV Cache:

┌────────────────────────────────────────────────────┐
│ GPU Memory                                          │
│                                                     │
│ KV Cache Seq A: [████████████████░░░░░░░░░░░░░░░░] │
│    Allocated for max_len=1024, used=256 → 75% waste │
│                                                     │
│ KV Cache Seq B: [████████████████████░░░░░░░░░░░░] │
│    Allocated for max_len=1024, used=400 → 60% waste │
│                                                     │
│ KV Cache Seq C: Waiting for contiguous block → 🚫  │
└────────────────────────────────────────────────────┘

PagedAttention Non-Contiguous KV Cache:

┌────────────────────────────────────────────────────┐
│ GPU Memory                                          │
│                                                     │
│ Physical Blocks:                                     │
│ ┌───┐ ┌───┐ ┌───┐ ┌───┐ ┌───┐ ┌───┐ ┌───┐ ┌───┐  │
│ │ 0 │ │ 1 │ │ 2 │ │ 3 │ │ 4 │ │ 5 │ │ 6 │ │ 7 │  │
│ │ A0│ │ A1│ │ C0│ │ B0│ │ A2│ │ B1│ │ C1│ │ A3│  │
│ └───┘ └───┘ └───┘ └───┘ └───┘ └───┘ └───┘ └───┘  │
│                                                     │
│ Block Tables (Logical → Physical):                  │
│ Seq A: [0 → 0, 1 → 1, 2 → 4, 3 → 7]               │
│ Seq B: [0 → 3, 1 → 5]                              │
│ Seq C: [0 → 2, 1 → 6]                              │
│                                                     │
│ Free Blocks: []                                     │
└────────────────────────────────────────────────────┘
```

## 5. Internal Working

**Block table structure:**
```
For sequence "The cat sat on the mat":
Logical blocks:  [Block 0] [Block 1] [Block 2]
Tokens:          "The cat  "  "sat on "  "the mat"
                 
Block table:     [0→12, 1→45, 2→7] where 12,45,7 are physical block IDs
```

**Attention with PagedAttention:**
```
Standard attention:
    Q · K^T → S  for contiguous K,V

PagedAttention:
    For each block b in block_table:
        load physical block table[b] → K_block, V_block
        Q · K_block^T → partial_S
    S = concatenate(partial_S)
    P = softmax(S)
    output = Σ p_i · V_i  (gathered from blocks)
```

**Block management operations:**
```
ALLOCATE(seq_id, num_blocks):
    blocks = free_list[:num_blocks]
    free_list = free_list[num_blocks:]
    block_table[seq_id].extend(blocks)
    for b in blocks: ref_count[b] += 1

FREE(seq_id):
    for b in block_table[seq_id]:
        ref_count[b] -= 1
        if ref_count[b] == 0:
            free_list.append(b)
    delete block_table[seq_id]

COPY_ON_WRITE(seq_id, block_idx):
    # Used when two sequences share a prefix but one needs to diverge
    old_block = block_table[seq_id][block_idx]
    if ref_count[old_block] > 1:
        new_block = free_list.pop()
        copy(old_block → new_block)  # GPU memcpy
        ref_count[old_block] -= 1
        block_table[seq_id][block_idx] = new_block
        ref_count[new_block] = 1
```

## 6. Production Flow

```
New sequence comes in
    │
    ▼
Compute needed blocks: ceil(prompt_len / block_size)
    │
    ▼
Query block manager for free blocks
    │
    ▼
If blocks available:
    ├── Allocate blocks, populate block table
    ├── Compute prefill on allocated blocks
    └── Continue to decode
    │
If no blocks:
    ├── Preempt lowest-priority sequence
    │   └── Either swap its blocks to CPU or mark for recompute
    └── Reclaim freed blocks
    │
    ▼
Decode step: append new token to last block
    │
    ▼
If last block full → allocate new block from free list
    │
    ▼
If sequence finished → free all blocks
```

## 7. HLD (High-Level Design)

```
┌─────────────────────────────────────────────────────────────────────┐
│                         PagedAttention Manager                        │
│                                                                       │
│  ┌─────────────────────┐  ┌──────────────────┐  ┌────────────────┐  │
│  │ Block Allocator      │  │ Block Table      │  │ Ref Counter    │  │
│  │ • Free block list    │  │ • Logical→Physical│  │ • Shared block │  │
│  │ • Allocation policy  │  │ • Per-sequence    │  │ • Copy-on-write│  │
│  │ • Defragmentation    │  │ • GPU-resident    │  │   detection    │  │
│  └──────────┬──────────┘  └────────┬─────────┘  └───────┬────────┘  │
│             │                      │                      │          │
│             └──────────────────────┼──────────────────────┘          │
│                                    │                                 │
│  ┌─────────────────────────────────┴─────────────────────────────┐  │
│  │                     GPU Memory Pool                            │  │
│  │  ┌──────┐ ┌──────┐ ┌──────┐ ┌──────┐ ┌──────┐ ┌──────┐       │  │
│  │  │Block0│ │Block1│ │Block2│ │Block3│ │Block4│ │Block5│  ...   │  │
│  │  │16K/V │ │16K/V │ │16K/V │ │16K/V │ │16K/V │ │16K/V │       │  │
│  │  └──────┘ └──────┘ └──────┘ └──────┘ └──────┘ └──────┘       │  │
│  └────────────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────────┘
```

## 8. LLD (Low-Level Design)

```python
import torch
from dataclasses import dataclass, field
from typing import Dict, List, Optional
from collections import defaultdict

@dataclass
class BlockManager:
    """Manages KV cache blocks with PagedAttention."""

    num_blocks: int
    block_size: int = 16
    num_layers: int = 32
    num_heads: int = 32
    head_dim: int = 128
    dtype: torch.dtype = torch.bfloat16

    def __post_init__(self):
        # Physical blocks stored as pre-allocated GPU memory
        # Shape: [num_blocks, 2, num_layers, block_size, num_heads, head_dim]
        self.kv_cache = torch.zeros(
            self.num_blocks,
            2,  # 0=key, 1=value
            self.num_layers,
            self.block_size,
            self.num_heads,
            self.head_dim,
            dtype=self.dtype,
            device="cuda",
        )

        # Free blocks tracking
        self.free_blocks: List[int] = list(range(self.num_blocks))
        self.ref_counts: Dict[int, int] = defaultdict(int)

        # Block tables: seq_id → list of physical block IDs
        self.block_tables: Dict[int, List[int]] = {}

    def allocate(self, seq_id: int, num_blocks: int) -> List[int]:
        """Allocate blocks for a sequence from the free list."""
        if num_blocks > len(self.free_blocks):
            return []  # Will trigger preemption

        blocks = self.free_blocks[:num_blocks]
        self.free_blocks = self.free_blocks[num_blocks:]
        self.block_tables[seq_id] = blocks
        for b in blocks:
            self.ref_counts[b] += 1
        return blocks

    def free(self, seq_id: int):
        """Free all blocks belonging to a completed sequence."""
        if seq_id not in self.block_tables:
            return
        for b in self.block_tables[seq_id]:
            self.ref_counts[b] -= 1
            if self.ref_counts[b] == 0:
                self.free_blocks.append(b)
        del self.block_tables[seq_id]

    def append_block(self, seq_id: int) -> Optional[int]:
        """Allocate one more block for a growing sequence."""
        if not self.free_blocks:
            return None
        block = self.free_blocks.pop(0)
        self.block_tables[seq_id].append(block)
        self.ref_counts[block] += 1
        return block

    def copy_on_write(self, seq_id: int, block_idx: int) -> bool:
        """Copy a shared block before modifying it (diverging sequences)."""
        block = self.block_tables[seq_id][block_idx]
        if self.ref_counts[block] == 1:
            return False  # No sharing; safe to modify
        if not self.free_blocks:
            return False  # No memory for copy
        new_block = self.free_blocks.pop(0)

        # GPU memcpy shared → private
        self.kv_cache[new_block].copy_(self.kv_cache[block])

        self.ref_counts[block] -= 1
        self.ref_counts[new_block] = 1
        self.block_tables[seq_id][block_idx] = new_block
        return True

    def get_physical_block(self, seq_id: int, logical_idx: int) -> int:
        """Translate logical block index to physical block ID."""
        return self.block_tables[seq_id][logical_idx]

    @property
    def usage(self) -> float:
        return 1.0 - len(self.free_blocks) / self.num_blocks

    @property
    def fragmentation(self) -> float:
        """Estimate of unusable free blocks due to fragmentation."""
        if not self.free_blocks:
            return 0.0
        # Simple metric: gaps in free block sequence
        free_sorted = sorted(self.free_blocks)
        gaps = sum(
            1 for i in range(len(free_sorted) - 1)
            if free_sorted[i + 1] - free_sorted[i] > 1
        )
        return gaps / len(self.free_blocks)
```

## 9. Python Implementation

```python
import torch
import torch.nn.functional as F
from typing import Tuple, Optional

class PagedAttention(torch.autograd.Function):
    """Custom attention with Paged KV cache."""

    @staticmethod
    def forward(
        ctx,
        query: torch.Tensor,          # [batch, num_heads, 1, head_dim]
        key_cache: torch.Tensor,      # [num_blocks, block_size, num_heads, head_dim]
        value_cache: torch.Tensor,    # [num_blocks, block_size, num_heads, head_dim]
        block_tables: torch.Tensor,   # [batch, max_blocks_per_seq]
        seq_lens: torch.Tensor,       # [batch] — current length of each sequence
        block_size: int = 16,
    ) -> torch.Tensor:
        """
        PagedAttention forward pass.

        For each sequence, gathers KV from non-contiguous physical blocks
        using the block table, then computes standard attention.
        """
        batch_size = query.shape[0]
        num_heads = query.shape[1]
        head_dim = query.shape[3]

        output = torch.zeros_like(query)

        for i in range(batch_size):
            seq_len = seq_lens[i].item()
            num_blocks = (seq_len + block_size - 1) // block_size

            # Gather K and V from physical blocks
            # block_tables[i, :num_blocks] → physical block indices
            k_blocks = key_cache[block_tables[i, :num_blocks]]  # [num_blocks, block_size, num_heads, head_dim]
            v_blocks = value_cache[block_tables[i, :num_blocks]]

            # Flatten to full KV
            k_full = k_blocks.reshape(-1, num_heads, head_dim)[:seq_len]  # [seq_len, num_heads, head_dim]
            v_full = v_blocks.reshape(-1, num_heads, head_dim)[:seq_len]

            # Standard attention
            attn_weights = torch.matmul(
                query[i], k_full.transpose(-2, -1)
            ) / (head_dim ** 0.5)
            attn_weights = F.softmax(attn_weights, dim=-1)
            output[i] = torch.matmul(attn_weights, v_full)

        return output


def paged_attention_optimized(
    query: torch.Tensor,
    key_cache: torch.Tensor,
    value_cache: torch.Tensor,
    block_tables: torch.Tensor,
    seq_lens: torch.Tensor,
    block_size: int = 16,
) -> torch.Tensor:
    """
    Batched PagedAttention using block-wise attention.

    Instead of gathering all KV first, compute attention per block
    and accumulate results. More memory-efficient.
    """
    batch_size = query.shape[0]
    num_heads = query.shape[1]
    head_dim = query.shape[3]
    max_blocks = block_tables.shape[1]

    output = torch.zeros_like(query)
    scale = head_dim ** -0.5

    for i in range(batch_size):
        seq_len = seq_lens[i].item()
        out_i = torch.zeros_like(query[i])

        for block_idx in range(max_blocks):
            phys_block = block_tables[i, block_idx]
            if phys_block < 0:  # sentinel for unused
                break

            k_block = key_cache[phys_block]  # [block_size, num_heads, head_dim]
            v_block = value_cache[phys_block]

            # How many tokens in this block are valid?
            tokens_in_block = min(block_size, seq_len - block_idx * block_size)

            # Block-wise attention score
            scores = torch.matmul(
                query[i].transpose(0, 1),  # [num_heads, 1, head_dim] wait no
                k_block[:tokens_in_block].transpose(-2, -1)
            ) * scale
            # scores shape: [num_heads, 1, tokens_in_block]

            # Simplified: just do full attention for correctness
            # (In production, block-sparse attention kernels handle this)
            k_full = k_block[:tokens_in_block].unsqueeze(0)
            v_full = v_block[:tokens_in_block].unsqueeze(0)

            attn = torch.matmul(query[i].unsqueeze(1), k_full.transpose(-2, -1)) * scale
            attn = F.softmax(attn, dim=-1)
            out_i = out_i + torch.matmul(attn, v_full).squeeze(1)

        output[i] = out_i

    return output
```

## 10. Folder Structure

```
paged_attention/
├── README.md
├── paged_attention/
│   ├── __init__.py
│   ├── block_manager.py     # Block allocation, free, CoW
│   ├── attention.py         # PagedAttention CUDA kernel wrapper
│   ├── kv_cache.py          # KV cache tensor management
│   ├── scheduler.py         # Continuous batching + PagedAttention
│   └── csrc/
│       ├── paged_attention_kernel.cu   # CUDA kernel
│       └── setup.py                    # Build script
├── tests/
│   ├── test_block_manager.py
│   └── test_attention.py
├── benchmark.py
└── requirements.txt
```

## 11. Configuration

```yaml
paged_attention:
  block_size: 16                  # Tokens per KV cache block
  num_blocks_ratio: 0.9           # Portion of GPU memory for KV cache blocks
  max_num_seqs: 256               # Maximum concurrent sequences
  max_model_len: 8192             # Maximum sequence length

  # Block allocation
  preemption_mode: recompute      # recompute or swap
  swap_space_gb: 16               # CPU swap space for evicted blocks

  # Sharing
  enable_prefix_caching: true     # Share blocks with common prefixes
  max_prefix_sharing: 64          # Max sequences sharing one prefix block

  # Fragmentation
  defrag_threshold_ratio: 0.15    # Trigger defrag when >15% fragmentation
  defrag_interval_seconds: 3600   # Run defrag every hour
```

## 12. Flowchart

```
                    ┌──────────────┐
                    │ New Sequence │
                    └──────┬───────┘
                           │
                           ▼
              ┌──────────────────────┐
              │  prompt_len = N      │
              │  need = ceil(N/16)   │
              │  blocks              │
              └──────┬───────────────┘
                     │
                     ▼
              ┌──────────────────────┐
              │  Free blocks >=      │──── No ──→ Preempt + reclaim
              │  needed?             │
              └──────┬───────────────┘
                     │ Yes
                     ▼
              ┌──────────────────────┐
              │  Allocate blocks     │
              │  from free list      │
              │  Update block table  │
              └──────┬───────────────┘
                     │
                     ▼
              ┌──────────────────────┐
              │  Prefill: compute    │
              │  KV for prompt tokens│
              │  Store in blocks     │
              └──────┬───────────────┘
                     │
                     ▼
              ┌──────────────────────┐
              │  Decode loop per     │
              │  token:              │
              │  1. PagedAttention   │
              │  2. Sample           │
              │  3. Append token     │
              │  4. If block full →  │
              │     allocate next    │
              └──────┬───────────────┘
                     │
              ┌──────┴──────┐
              │              │
         ┌────▼────┐   ┌────▼────┐
         │ EOS?    │   │ Max?   │
         │ Free    │   │ Free   │
         │ blocks  │   │ blocks │
         └─────────┘   └─────────┘
```

## 13. Sequence Diagram

```
Scheduler      BlockManager       GPU           Sequence
    │               │               │              │
    │───allocate──▶│               │              │
    │               │───pop free──▶│              │
    │               │◀──blocks────│              │
    │               │               │              │
    │───prefill───▶│               │              │
    │               │───store KV─▶│              │
    │               │               │              │
    │───decode────▶│               │              │
    │               │───gather────▶│              │
    │               │   via block  │              │
    │               │   table      │              │
    │               │◀──output────│              │
    │◀──token──────│               │              │
    │               │               │              │
    │───append────▶│               │              │
    │               │───(optional)─▶│              │
    │               │   new block  │              │
    │               │               │              │
    │───free───────▶│               │              │
    │               │───push free─▶│              │
    │               │   ref_count--│              │
```

## 14. Pros

- **Near-zero memory waste** — eliminates internal fragmentation in KV cache
- **Prefix sharing** — sequences with common prefixes share KV cache blocks (copy-on-write)
- **Efficient continuous batching** — freed blocks immediately reusable by new sequences
- **Demand-based paging** — allocate blocks only as the sequence grows (not upfront)
- **CoW for divergence** — shared prefixes efficiently fork when sequences diverge
- **Enables larger batches** — 2-4x more sequences per GPU compared to contiguous KV cache
- **Composability** — works with tensor parallelism, quantization, Flash Attention

## 15. Cons

- **Implementation complexity** — custom CUDA kernels needed for efficient block-gather attention
- **Block table overhead** — each sequence's block table uses ~1KB per 1K tokens
- **Small blocks = more overhead** — block_size=1 wastes pointer storage; block_size=128 wastes memory. 16 is optimal for most workloads
- **No cross-block attention optimization** — Flash Attention's tiling is harder with non-contiguous blocks
- **Defragmentation needed** — over time, blocks become fragmented, reducing effective cache size
- **Copy-on-write cost** — forking shared prefixes requires GPU memcpy

## 16. Alternatives

| Alternative | Description | When to Use |
|------------|-------------|-------------|
| **Contiguous KV cache** | Traditional static allocation | Simple implementations, small models |
| **vLLM's PagedAttention** | Open-source reference | Production LLM serving |
| **TensorRT-LLM paged KV** | NVIDIA's implementation | NVIDIA-optimized inference |
| **RadixAttention** | SGLang's prefix-tree KV | Structured generation workloads |
| **HuggingFace sliding window** | Mistral's windowed attention | Long-context with limited budget |
| **StreamingLLM** | Rolling KV cache for infinite length | Unlimited streaming, lower quality |

## 17. Performance Considerations

**Optimal block size:**
```
block_size=8:    Lower memory waste (~8%), higher overhead (12.5% for metadata)
block_size=16:   Good balance (default in vLLM, ~10% waste)
block_size=32:   Lower overhead, higher waste (~20%)
block_size=64:   Lower overhead, higher waste (~30%)

Rule: block_size × num_layers × 2 × num_heads × head_dim × dtype_bytes should fit in L2 cache
For H100: L2=50MB, block_size=16 → 16×32×2×4096×2=8.4MB ✓
```

**Memory overhead comparison:**
```
Model           Contiguous (max=4096)    PagedAttention (block=16)
Llama-3-8B      2.1 GB/seq               ~0.5 GB/seq (avg 512 tokens)
Llama-3-70B     16 GB/seq                ~4 GB/seq (avg 512 tokens)
Savings:        4x reduction in KV cache memory
```

**Attention kernel overhead:**
```
Scatter-gather from non-contiguous blocks adds ~5-10% overhead vs contiguous attention
Mitigated by: caching block table in shared memory, prefetching blocks
```

## 18. Scaling to Millions

**For 10M requests/day with PagedAttention:**
```
Avg tokens/request: 1024
KV blocks per request: ~64 (at block_size=16)
Total blocks needed per second: 64 * 116 = ~7,400 blocks/s
Block reuse with prefix caching: ~60% hit rate → ~3,000 new blocks/s
GPU memory for blocks (H100, 80GB, 90% utilization): ~45,000 blocks
Maximum concurrent sequences: ~700 (at 512 tokens avg)
Throughput: ~35K tokens/s

Without PagedAttention (contiguous at max=4096):
Maximum concurrent sequences: ~175 (4x fewer)
Throughput: ~9K tokens/s (4x lower)
```

**Clustered deployment:**
```
With PagedAttention + prefix caching:
- 16 × H100-80GB serves 10M+ requests/day
- 4x fewer GPUs vs non-paged KV cache
- Cost savings: ~$200K/year in GPU rental
```

## 19. Failure Scenarios

| Failure | Impact | Mitigation |
|---------|--------|------------|
| **Block table OOM** | Cannot allocate more logical blocks | Increase max_num_seqs, or reduce max_model_len |
| **GPU block corruption** | Garbled output from wrong KV | Checksums per block, CRC-32 validation |
| **Copy-on-write deadlock** | Two sequences both waiting for CoW | Lock-free block table with atomic operations |
| **Free list corruption** | Double-free or lost blocks | Reference counting validation |
| **Block leak** | Gradual memory loss (blocks not freed) | Periodic block sweep with GC |

## 20. Security

| Concern | Mitigation |
|---------|------------|
| **Cross-sequence data in blocks** | Block reuse could leak data — zero blocks on free |
| **Prefix cache poisoning** | Crafted prompt shares prefix with sensitive data | Hash prefix before caching |
| **Block table access** | Malformed block table reads arbitrary GPU memory | Bounds checking on every block access |
| **CoW race condition** | Concurrent fork of shared prefix | Atomic compare-and-swap for block references |

## 21. Monitoring

**Key metrics:**
```
paged_attn:num_blocks_total        — Total physical blocks allocated
paged_attn:num_blocks_free         — Currently free blocks
paged_attn:num_blocks_used         — Allocated by sequences
paged_attn:block_utilization       — Used / Total
paged_attn:num_sequences           — Active sequences
paged_attn:preemption_rate         — Preemptions per second
paged_attn:block_fragmentation     — Fragmentation ratio
paged_attn:copy_on_write_count     — CoW operations per second
paged_attn:avg_blocks_per_seq      — Average blocks per sequence
paged_attn:prefix_cache_hit_rate   — Shared prefix block reuse rate
```

**Grafana panels:**
1. Block utilization over time (should be >70% under load)
2. Preemption rate (should be <1%)
3. Fragmentation heatmap by GPU
4. Prefix cache hit rate (should be >40% with shared system prompts)
5. Block allocation throughput per second

**Alerts:**
- Block utilization > 95% → scale up or increase preemption
- Preemption rate > 5% → insufficient KV cache
- Fragmentation > 20% → trigger defragmentation
- Prefix cache hit rate < 10% → not sharing prefixes effectively

## 22. Interview Questions

**Q1: How does PagedAttention reduce memory fragmentation compared to static KV cache?**
*A: Static KV cache allocates contiguous memory equal to max_sequence_length per sequence. If a sequence only uses 10% of its allocation, 90% is wasted (internal fragmentation). PagedAttention allocates in fixed-size blocks on demand, so a sequence using 100 tokens with block_size=16 gets 7 blocks (112 token capacity) — only 9% waste. Blocks are freed immediately when a sequence completes.*

**Q2: Explain the copy-on-write mechanism in PagedAttention and why it matters for prefix caching.**
*A: When two sequences share a prefix (e.g., same system prompt), their block tables point to the same physical blocks. If one sequence needs to modify a shared block (because it has generated a different next token), CoW allocates a new block, copies the data, and updates only that sequence's block table. This prevents the modification from affecting the other sequence while preserving memory sharing for the still-identical prefix.*

**Q3: How does PagedAttention interact with Flash Attention?**
*A: Flash Attention tiles the attention computation to fit in SRAM, processing blocks of queries and keys. PagedAttention's non-contiguous blocks require gathering KV from scattered physical addresses before Flash Attention can run. Recent work integrates PagedAttention's block table lookup into Flash Attention's tiling loop, fetching KV blocks on-demand rather than pre-gathering. The vLLM team has implemented this in their latest kernels.*

**Q4: What block size would you choose and why?**
*A: 16 tokens is the default in vLLM and generally optimal. Smaller blocks (8) reduce waste but increase metadata overhead and block table size. Larger blocks (32-64) reduce metadata but increase internal fragmentation. On H100 with 50MB L2 cache, block_size=16 keeps one block's KV data (8.4MB for 70B) fitting in L2 for fast access. For very long sequences (32K+), block_size=32 might be better to reduce block table size.*

## 23. Cheat Sheet

```bash
# vLLM uses PagedAttention by default
# Key parameters:
python -m vllm.entrypoints.openai.api_server \
    --model meta-llama/Llama-3.2-3B-Instruct \
    --max-model-len 4096 \
    --max-num-seqs 256 \
    --gpu-memory-utilization 0.9 \
    --block-size 16  # default

# Memory calculation:
# num_blocks = free_gpu_memory / (block_size * num_layers * 2 * hidden_dim * dtype_bytes)
# For Llama-3-8B on H100 80GB:
# Weights: 16GB
# Free for KV: 80 * 0.9 - 16 = 56GB
# Block size: 16 * 32 * 2 * 4096 * 2 = 8.4MB
# num_blocks: 56GB / 8.4MB ≈ 6800 blocks

# Monitor block usage:
# vLLM metrics: /metrics endpoint
vllm:gpu_cache_usage  # 0.0-1.0
vllm:num_blocks_free  # Physical blocks remaining
```

## 24. Common Mistakes

1. **Choosing wrong block size** — small blocks (1-4) create massive block tables. Large blocks (128+) waste memory. Benchmark 8, 16, 32 for your workload.

2. **Not using prefix caching** — without prefix caching enabled, every sequence with the same system prompt duplicates KV blocks. Enable `--enable-prefix-caching` for 30-60% block savings.

3. **Oversubscribing block count** — setting `gpu_memory_utilization=0.99` leaves no room for block metadata, causing OOM. Keep at 0.85-0.92.

4. **Ignoring block fragmentation** — over hours, blocks become fragmented and utilization appears lower than actual. Monitor fragmentation ratio and trigger compaction.

5. **Not profiling block allocation latency** — block allocation from Python lists becomes slow at 10K+ blocks. Use a slab allocator or pre-allocated arrays.

6. **Using swap mode when recompute is better** — swapping blocks to CPU is only beneficial when CPU-GPU bandwidth is very high. For most setups, recompute mode is faster and simpler.

## 25. Production Best Practices

1. **Benchmark block size for your model** — run a sweep of block_size = [8, 16, 32, 64] at your typical request size distribution. Measure throughput and memory utilization.

2. **Enable prefix caching** — `--enable-prefix-caching` in vLLM. For maximum benefit, canonicalize system prompts (trim whitespace, sort tool schemas) to maximize prefix sharing.

3. **Monitor `block_utilization`** — if it's consistently below 60%, reduce `gpu_memory_utilization` or increase `max_num_seqs`. If above 90%, consider more GPUs.

4. **Set `max_model_len` conservatively** — the default is often larger than needed. If your workload never exceeds 4096 tokens, don't reserve blocks for 8192.

5. **Implement block defragmentation** — periodically compact blocks when fragmentation exceeds 15%. Run during low-traffic periods.

6. **Use FP8 KV cache** — `--kv-cache-dtype fp8` halves the memory per block, allowing 2x more blocks in the same GPU memory.

7. **Pre-allocate block manager structures** — allocate the block table and free list arrays during initialization rather than using Python lists that grow dynamically.

8. **Optimize for zero-block allocation** — for decode-only steps (no new prefill), block allocation rate is zero. The scheduler should skip block manager calls entirely in this case.

9. **Align block table to warp size** — the block table lookup kernel benefits from memory alignment. Pad block tables to multiples of 32 entries.

10. **Test preemption paths** — verify that when block pressure is high, preemption (swap or recompute) works correctly. This is the most common source of production bugs.
