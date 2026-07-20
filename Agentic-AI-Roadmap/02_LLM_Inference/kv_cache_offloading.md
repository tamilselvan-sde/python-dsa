# KV Cache Offloading

> Author: **Tamilselvan** · ✉️ tamilselvan.sde@gmail.com · 🔗 [LinkedIn](https://www.linkedin.com/in/tamilselvan-ai/)
>

## 1. What is it?

KV cache offloading is a memory management technique that moves less-frequently accessed KV cache entries from GPU memory to CPU memory (or NVMe storage) during LLM inference, freeing GPU memory for more important computations.

**ELI5:** Imagine your desk (GPU) is small, but you have a filing cabinet (CPU RAM) next to it. You keep the papers you're currently reading on your desk. When your desk gets full, you file away older papers you won't need soon. If you need an old paper again, you fetch it from the filing cabinet. KV cache offloading is that filing system for the model's "working memory."

**Technical Definition:** KV cache offloading swaps KV cache blocks between GPU and CPU memory based on access patterns. During the decode loop, the most recent N tokens' KV stays on GPU (fast access); older tokens' KV is evicted to CPU (slow access). On cache miss, the block is loaded back to GPU via PCIe DMA. The GPU computation is overlapped with the PCIe transfer to hide latency.

## 2. Why do we need it?

**Problem:** The KV cache grows linearly with sequence length and batch size. For Llama-3-70B with 32K context and batch size 64, the KV cache requires ~500GB of GPU memory — far beyond any single GPU's capacity (80GB H100, 141GB H200, 192GB GH200).

**Pain without it:** Without offloading, you're limited to either very short contexts (2-4K tokens), very small batches (1-4 sequences), or extremely expensive hardware (multiple H100s just for KV cache). Long-context applications (RAG with 32K+ contexts, code analysis, document processing) are impractical or prohibitively expensive.

**Why companies use it:** KV cache offloading enables long-context inference on limited hardware. A 70B model with 128K context that would require 8+ H100s for KV cache alone can run on a single H100 with offloading (at reduced throughput). For batch inference with varying-length sequences, offloading allows oversubscribing GPU memory during peak loads.

## 3. Real-world Example

- **FlexGen (Stanford)**: Pioneered KV cache offloading with a flexible offloading strategy that considers GPU, CPU, and NVMe memory tiers.
- **DeepSpeed Inference (Microsoft)**: Implements automatic KV cache offloading in ZeRO-Inference, moving cache between GPU and CPU.
- **vLLM**: Supports swap-to-CPU mode for KV cache blocks when GPU memory is exhausted (`--swap-space`).
- **HuggingFace TGI**: Offloads KV cache to CPU when `max_input_tokens` exceeds GPU capacity.
- **Apple MLX**: Uses unified memory architecture where KV cache automatically pages between GPU and CPU (Metal unified memory).
- **NVIDIA TensorRT-LLM**: Supports KV cache offloading via the "inflight batching" scheduler, which can swap to CPU.

## 4. Architecture Diagram (ASCII)

```
Memory Hierarchy for KV Cache:

Speed (fastest → slowest)        Capacity (smallest → largest)
┌──────────────────────┐
│ L1/L2 Cache (on-chip) │   ~50MB    — KV block currently being used
└──────────┬───────────┘
           │
┌──────────▼───────────┐
│ GPU HBM (on-board)   │  80-192GB  — "Hot" KV blocks
│ ┌──────────────────┐ │            (recent tokens + frequently accessed)
│ │ Active KV Blocks │ │
│ └──────────────────┘ │
└──────────┬───────────┘
           │  PCIe / NVLink-C2C (64-900 GB/s)
           │
┌──────────▼───────────┐
│ CPU DRAM (system)    │  256GB-2TB — "Warm" KV blocks
│ ┌──────────────────┐ │            (older tokens, swapped out)
│ │ Swapped KV Blks  │ │
│ └──────────────────┘ │
└──────────┬───────────┘
           │  SSD / NVMe (3-14 GB/s)
           │
┌──────────▼───────────┐
│ NVMe Storage         │  1-30TB    — "Cold" KV blocks
│ ┌──────────────────┐ │            (infrequently accessed)
│ │ Cold KV Blks     │ │
│ └──────────────────┘ │
└──────────────────────┘
```

## 5. Internal Working

**Offloading strategies:**

**1. Full offloading (all KV on CPU):**
- KV cache is entirely in CPU memory
- Each attention step transfers one block to GPU, computes, then discards
- Simple but very slow — GPU bandwidth is bottleneck

**2. Selective offloading (FlexGen approach):**
- Keep recent N blocks on GPU (sliding window of "hot" tokens)
- Older blocks on CPU (lookup requires PCIe transfer)
- Based on attention pattern: recent tokens are accessed every step, early tokens rarely accessed

**3. Predictive offloading:**
- Learn attention patterns to predict which blocks will be needed
- Prefetch predicted blocks before they're needed (overlap with compute)
- Uses the observation that attention layers access certain positions more frequently

**Operation flow:**
```
Decode step t:
  1. Need KV for positions [0, 1, ..., t]
  2. Check which positions are in GPU memory
  3. For GPU-resident blocks: use directly
  4. For CPU-resident blocks: 
     a. Issue PCIe DMA read (async)
     b. Meanwhile, compute attention for GPU-resident blocks
     c. When DMA completes, compute attention for offloaded blocks
     d. After computation, optionally keep or evict back to CPU
  5. After step: append new token's KV to GPU resident set
  6. If GPU memory exceeds threshold: evict least-recently-used block to CPU
```

## 6. Production Flow

```
Request arrives → Tokenize → Prefill (KV computed)
    │
    ▼
Prefilled KV initially in GPU memory
    │
    ▼
Start decode loop
    │
    ▼
For each decode step:
    1. Check GPU KV memory budget
       │
    2. If budget exceeded:
       │
       └──→ Select victim blocks (LRU)
            │
            ├──→ Copy KV blocks to CPU (async memcpy)
            └──→ Update block table (GPU → CPU location)
       │
    3. For needed attention positions:
       ├──→ If on GPU: use directly
       └──→ If on CPU: issue DMa read, wait, then compute
       │
    4. Compute attention (overlapped with PCIe)
    5. Generate next token
    6. Append new KV to GPU resident set
    │
    ▼
Continue until EOS or max_tokens
```

## 7. HLD (High-Level Design)

```
┌──────────────────────────────────────────────────────────────────┐
│                     KV Cache Offloading Manager                    │
│                                                                    │
│  ┌─────────────────────────┐  ┌────────────────────────────┐     │
│  │ GPU Block Allocator      │  │ CPU Block Allocator        │     │
│  │ • Fast allocation        │  │ • Large capacity           │     │
│  │ • PagedAttention blocks  │  │ • Pinned memory for DMA    │     │
│  │ • Limited capacity       │  │ • Pre-allocated pool       │     │
│  └───────────┬─────────────┘  └───────────┬────────────────┘     │
│              │                            │                       │
│              ▼                            ▼                       │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │              Eviction / Admission Policy                  │   │
│  │                                                           │   │
│  │  Policy: LRU (Least Recently Used)                       │   │
│  │  Decision per step:                                      │   │
│  │    - Which blocks to evict (GPU → CPU)                   │   │
│  │    - Which blocks to prefetch (CPU → GPU)                │   │
│  │    - When to promote to GPU                              │   │
│  └──────────────────────┬───────────────────────────────────┘   │
│                         │                                        │
│                         ▼                                        │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │              DMA Engine (Async PCIe Transfers)            │   │
│  │  • Uses CUDA streams for overlap with compute             │   │
│  │  • Pinned memory for fast CPU↔GPU transfer                │   │
│  │  • Fuse small transfers into larger batches               │   │
│  └──────────────────────────────────────────────────────────┘   │
└──────────────────────────────────────────────────────────────────┘
```

## 8. LLD (Low-Level Design)

```python
import torch
from dataclasses import dataclass, field
from typing import Dict, List, Optional, Tuple
from enum import Enum
import time

class MemoryTier(Enum):
    GPU = "gpu"
    CPU = "cpu"
    NVME = "nvme"

@dataclass
class KVBlockInfo:
    """Metadata for a single KV cache block."""
    physical_id: int
    tier: MemoryTier
    seq_id: int
    logical_block_idx: int
    last_access_time: float
    ref_count: int = 1
    gpu_location: Optional[int] = None  # GPU buffer index if on GPU
    cpu_location: Optional[int] = None  # CPU buffer index if on CPU

class OffloadingBlockManager:
    """Manages KV blocks across GPU and CPU memory."""

    def __init__(
        self,
        gpu_capacity_blocks: int = 5000,
        cpu_capacity_blocks: int = 20000,
        block_size: int = 16,
        num_layers: int = 32,
        num_heads: int = 32,
        head_dim: int = 128,
        dtype: torch.dtype = torch.bfloat16,
    ):
        self.gpu_capacity = gpu_capacity_blocks
        self.cpu_capacity = cpu_capacity_blocks
        self.block_size = block_size

        # Block metadata per block
        self.block_info: Dict[int, KVBlockInfo] = {}

        # GPU memory pool (pre-allocated)
        self.gpu_pool = self._create_pool(gpu_capacity_blocks, "cuda")
        self.gpu_free: List[int] = list(range(gpu_capacity_blocks))
        self.gpu_used: Dict[int, int] = {}  # physical_id → gpu_index

        # CPU memory pool (pinned for fast DMA)
        self.cpu_pool = self._create_pool(cpu_capacity_blocks, "cpu", pinned=True)
        self.cpu_free: List[int] = list(range(cpu_capacity_blocks))
        self.cpu_used: Dict[int, int] = {}

        # Stream for overlapping transfers
        self.transfer_stream = torch.cuda.Stream()

    def _create_pool(
        self, num_blocks: int, device: str, pinned: bool = False
    ) -> torch.Tensor:
        """Pre-allocate block memory pool."""
        shape = (num_blocks, self.block_size, 2, 32, 128)  # Simplified
        if device == "cuda":
            return torch.zeros(shape, dtype=torch.bfloat16, device="cuda")
        else:
            if pinned:
                return torch.zeros(shape, dtype=torch.bfloat16, device="cpu").pin_memory()
            return torch.zeros(shape, dtype=torch.bfloat16, device="cpu")

    def allocate_on_gpu(self, seq_id: int, logical_idx: int) -> int:
        """Allocate a block on GPU."""
        if not self.gpu_free:
            return self._evict_to_cpu()  # Free up space first
        gpu_idx = self.gpu_free.pop(0)
        phys_id = f"seq_{seq_id}_blk_{logical_idx}"
        self.block_info[phys_id] = KVBlockInfo(
            physical_id=phys_id,
            tier=MemoryTier.GPU,
            seq_id=seq_id,
            logical_block_idx=logical_idx,
            last_access_time=time.time(),
            gpu_location=gpu_idx,
        )
        self.gpu_used[phys_id] = gpu_idx
        return phys_id

    def _evict_to_cpu(self) -> int:
        """Evict the least recently used block from GPU to CPU."""
        # Find LRU block on GPU
        gpu_blocks = [
            info for info in self.block_info.values()
            if info.tier == MemoryTier.GPU
        ]
        if not gpu_blocks:
            raise RuntimeError("No GPU blocks to evict")
        victim = min(gpu_blocks, key=lambda x: x.last_access_time)

        # Allocate CPU slot
        if not self.cpu_free:
            raise RuntimeError("CPU memory exhausted")
        cpu_idx = self.cpu_free.pop(0)

        # Async transfer GPU → CPU
        with torch.cuda.stream(self.transfer_stream):
            self.cpu_pool[cpu_idx].copy_(self.gpu_pool[self.gpu_used[victim.physical_id]])

        # Update metadata
        victim.tier = MemoryTier.CPU
        victim.cpu_location = cpu_idx
        self.cpu_used[victim.physical_id] = cpu_idx
        self.gpu_free.append(self.gpu_used.pop(victim.physical_id))

        return self.gpu_free[-1]  # Return freed GPU slot

    def promote_to_gpu(self, phys_id: str) -> bool:
        """Move a CPU-resident block back to GPU (cache hit)."""
        info = self.block_info.get(phys_id)
        if not info or info.tier != MemoryTier.CPU:
            return False

        # Ensure GPU space
        if not self.gpu_free:
            self._evict_to_cpu()

        gpu_idx = self.gpu_free.pop(0)
        cpu_idx = info.cpu_location

        # Async transfer CPU → GPU
        with torch.cuda.stream(self.transfer_stream):
            self.gpu_pool[gpu_idx].copy_(self.cpu_pool[cpu_idx])

        # Update metadata
        info.tier = MemoryTier.GPU
        info.gpu_location = gpu_idx
        info.last_access_time = time.time()
        self.gpu_used[phys_id] = gpu_idx
        self.cpu_free.append(self.cpu_used.pop(phys_id))

        return True

    def access_block(self, phys_id: str) -> Optional[torch.Tensor]:
        """Get the data for a block, promoting if needed."""
        info = self.block_info.get(phys_id)
        if not info:
            return None

        info.last_access_time = time.time()

        if info.tier == MemoryTier.GPU:
            return self.gpu_pool[info.gpu_location]
        elif info.tier == MemoryTier.CPU:
            self.promote_to_gpu(phys_id)
            return self.gpu_pool[info.gpu_location]
        return None
```

## 9. Python Implementation

```python
import asyncio
import torch
from fastapi import FastAPI, HTTPException
from pydantic import BaseModel, Field
from typing import Optional
import time
import logging

logging.basicConfig(level=logging.INFO)
logger = logging.getLogger("kv-offloading")

app = FastAPI(title="KV Cache Offloading Server")

class OffloadingConfig(BaseModel):
    gpu_cache_blocks: int = Field(default=3000, description="GPU KV cache capacity in blocks")
    cpu_cache_blocks: int = Field(default=12000, description="CPU KV cache capacity in blocks")
    block_size: int = Field(default=16, description="Tokens per block")
    eviction_policy: str = Field(default="lru", description="lru, lfu, or fifo")
    prefetch_ahead: int = Field(
        default=5,
        description="Number of blocks ahead to prefetch from CPU",
    )
    async_transfers: bool = Field(default=True, description="Use async DMA transfers")
    pinned_memory: bool = Field(default=True, description="Use pinned CPU memory")

class GenerateRequest(BaseModel):
    prompt: str
    max_tokens: int = Field(default=256, ge=1, le=131072)

class OffloadingStats(BaseModel):
    gpu_blocks_used: int
    cpu_blocks_used: int
    evictions: int
    promotions: int
    avg_promotion_latency_ms: float
    gpu_cache_hit_rate: float

class SimulatedModel:
    """Simulates a model with offloaded KV cache."""

    def __init__(self, config: OffloadingConfig, model):
        self.config = config
        self.model = model
        self.gpu_blocks = set()
        self.cpu_blocks = set()
        self.block_timestamps = {}
        self.block_data = {}
        self.evictions = 0
        self.promotions = 0
        self.gpu_hits = 0
        self.gpu_misses = 0
        self.transfer_times = []

    def access_kv(self, block_id: int) -> bool:
        """Access a KV block. Returns True if on GPU (hit), False if CPU (miss)."""
        self.block_timestamps[block_id] = time.time()

        if block_id in self.gpu_blocks:
            self.gpu_hits += 1
            return True

        # Must be on CPU
        self.gpu_misses += 1
        start = time.time()

        # Simulate PCIe transfer (5-50μs real, compressed to 1ms for demo)
        if block_id in self.cpu_blocks:
            self._promote(block_id)
            self.transfer_times.append((time.time() - start) * 1000)
            return False

        # Block not found (shouldn't happen in normal operation)
        return False

    def _promote(self, block_id: int):
        """Move block from CPU to GPU."""
        self.promotions += 1
        self.cpu_blocks.discard(block_id)

        # Ensure GPU space
        while len(self.gpu_blocks) >= self.config.gpu_cache_blocks:
            self._evict()

        self.gpu_blocks.add(block_id)

    def _evict(self):
        """Evict LRU block from GPU to CPU."""
        self.evictions += 1

        # Find LRU
        gpu_blocks_list = [
            b for b in self.gpu_blocks
        ]
        if not gpu_blocks_list:
            return

        victim = min(gpu_blocks_list, key=lambda b: self.block_timestamps.get(b, 0))

        self.gpu_blocks.discard(victim)
        while len(self.cpu_blocks) >= self.config.cpu_cache_blocks:
            # Evict from CPU too (LRU)
            cpu_victim = min(
                self.cpu_blocks,
                key=lambda b: self.block_timestamps.get(b, 0),
            )
            self.cpu_blocks.discard(cpu_victim)
            self.block_data.pop(cpu_victim, None)

        self.cpu_blocks.add(victim)

    def add_block(self, block_id: int, data: torch.Tensor):
        """Add a newly computed KV block (starts on GPU)."""
        self.block_data[block_id] = data
        self.block_timestamps[block_id] = time.time()

        if len(self.gpu_blocks) >= self.config.gpu_cache_blocks:
            self._evict()
        self.gpu_blocks.add(block_id)

    @property
    def gpu_hit_rate(self) -> float:
        total = self.gpu_hits + self.gpu_misses
        return self.gpu_hits / total if total > 0 else 0.0

    @property
    def avg_promotion_latency(self) -> float:
        if not self.transfer_times:
            return 0.0
        return sum(self.transfer_times) / len(self.transfer_times)

class OffloadingServer:
    def __init__(self, config: OffloadingConfig):
        self.config = config
        # Placeholder model
        self.model = None
        self.kv_manager = SimulatedModel(config, self.model)

    async def generate(self, prompt: str, max_tokens: int) -> str:
        """Generate text with KV cache offloading."""
        num_blocks = (len(prompt) // self.config.block_size) + max_tokens // self.config.block_size + 1

        # Prefill: compute all blocks
        for i in range((len(prompt) + self.config.block_size - 1) // self.config.block_size):
            block_data = torch.randn(1024)  # Simulated KV
            self.kv_manager.add_block(i, block_data)

        # Decode loop with offloading
        generated = []
        for step in range(max_tokens):
            # Access all previous blocks (simulating causal attention)
            total_blocks = (len(prompt) + step) // self.config.block_size + 1
            for bid in range(max(0, total_blocks - 10), total_blocks):
                self.kv_manager.access_kv(bid)

            # Simulate token generation
            generated.append(chr(ord('a') + (step % 26)))
            new_block_id = total_blocks
            self.kv_manager.add_block(new_block_id, torch.randn(1024))

            # Brief sleep to simulate compute
            await asyncio.sleep(0.001)

        return "".join(generated)

config = OffloadingConfig()
server = OffloadingServer(config)

@app.post("/generate")
async def generate(request: GenerateRequest):
    try:
        start = time.time()
        text = await server.generate(request.prompt, request.max_tokens)
        latency = time.time() - start

        return {
            "text": text,
            "latency_seconds": round(latency, 2),
            "offloading_stats": OffloadingStats(
                gpu_blocks_used=len(server.kv_manager.gpu_blocks),
                cpu_blocks_used=len(server.kv_manager.cpu_blocks),
                evictions=server.kv_manager.evictions,
                promotions=server.kv_manager.promotions,
                avg_promotion_latency_ms=round(
                    server.kv_manager.avg_promotion_latency, 2
                ),
                gpu_cache_hit_rate=round(server.kv_manager.gpu_hit_rate, 3),
            ),
        }
    except Exception as e:
        logger.error(f"Generation failed: {e}")
        raise HTTPException(status_code=500, detail=str(e))

@app.get("/config")
async def get_config():
    return config.model_dump()
```

## 10. Folder Structure

```
kv_cache_offloading/
├── Dockerfile
├── config.yaml
├── serve.py
├── offloading/
│   ├── __init__.py
│   ├── block_manager.py      # Cross-tier block management
│   ├── eviction_policy.py     # LRU, LFU, custom policies
│   ├── transfer_engine.py     # Async PCIe DMA operations
│   ├── pinned_memory.py       # CPU pinned memory pool
│   ├── prefetcher.py          # Predictive prefetching
│   ├── metrics.py             # Offloading performance metrics
│   └── config.py
├── k8s/
│   ├── deployment.yaml
│   └── configmap.yaml
├── tests/
│   ├── test_eviction.py
│   ├── test_transfer.py
│   └── test_offloading.py
└── benchmark.py
```

## 11. Configuration

```yaml
offloading:
  enabled: true
  gpu_cache_blocks: 5000        # GPU-resident block capacity
  cpu_cache_blocks: 30000       # CPU-resident block capacity
  nvme_cache_blocks: 0          # NVMe tier (0 = disabled)

  eviction:
    policy: lru                  # lru, lfu, fifo, custom
    warmup_steps: 10             # No evictions until after this many steps
    max_evictions_per_step: 8    # Limit evictions to avoid latency spikes

  transfer:
    async_mode: true             # Async DMA for overlap
    pin_memory: true             # Pin CPU memory for fast transfer
    transfer_chunk_size: 1       # Blocks per transfer (fuse small transfers)
    default_stream_priority: 0   # CUDA stream priority

  prefetch:
    enabled: true
    prefetch_ahead: 5            # Prefetch blocks N steps ahead
    prefetch_window: 50          # Max blocks to consider for prefetch
    min_hit_rate_for_prefetch: 0.3  # Disable prefetch if hit rate too low

  thresholds:
    gpu_eviction_watermark: 0.85    # Evict when GPU >85% full
    gpu_admission_watermark: 0.70   # Admit until 70% full
    hard_limit_gpu: 0.95            # Force eviction at 95%
```

## 12. Flowchart

```
                    ┌──────────────┐
                    │ Decode Step  │
                    └──────┬───────┘
                           │
                           ▼
              ┌───────────────────────┐
              │ GPU memory usage >    │
              │ eviction watermark?   │
              └──────┬───────────────┘
                     │
              ┌──────┴──────┐
              │             │
         ┌────▼────┐  ┌────▼────┐
         │ No      │  │ Yes     │
         └─────────┘  │ Evict   │
                      │ LRU     │
                      │ block   │
                      │ GPU→CPU │
                      └────┬────┘
                           │
                           ▼
              ┌───────────────────────┐
              │ For each attention    │
              │ position:             │
              └──────┬───────────────┘
                     │
              ┌──────┴──────┐
              │             │
         ┌────▼────┐  ┌────▼────┐
         │ On GPU  │  │ On CPU  │
         └────┬────┘  │ Issue   │
              │       │ DMA     │
              │       │ read    │
              │       └────┬────┘
              │            │
              └──────┬─────┘
                     │
                     ▼
              ┌───────────────────────┐
              │ Compute Attention     │
              │ (GPU: use directly)   │
              │ (CPU: wait for DMA)   │
              └──────┬───────────────┘
                     │
                     ▼
              ┌───────────────────────┐
              │ Generate token +      │
              │ append new KV block   │
              └──────┬───────────────┘
                     │
                     ▼
              ┌───────────────────────┐
              │ New block pushes      │
              │ out LRU block         │
              │ (if GPU full)         │
              └──────┬───────────────┘
                     │
                     ▼
              ┌───────────────────────┐
              │ Next step             │
              └───────────────────────┘
```

## 13. Sequence Diagram

```
GPU Decode Loop          Transfer Engine          CPU Memory
    │                         │                      │
    │───Step N────────────────▶│                     │
    │                         │                      │
    │───Check GPU budget─────▶│                     │
    │                         │                      │
    │   (GPU 90% full)        │                      │
    │───Evict block X────────▶│───memcpy async──────▶│
    │                         │◀──stored─────────────│
    │   (Continue compute)    │                      │
    │                         │                      │
    │───Need old block Y─────▶│                      │
    │   (Check: on CPU)       │───memcpy async──────▶│
    │                         │◀──loaded─────────────│
    │   (DMA completes)       │                      │
    │   (Compute attention)   │                      │
    │                         │                      │
    │───Promote block Y──────▶│                      │
    │   (Make room)           │                      │
    │                         │                      │
    │───Step N+1─────────────▶│                      │
    │                         │                      │
    │───Prefetch block Z─────▶│───memcpy async──────▶│
    │   (Next 5 steps)        │◀──loaded─────────────│
    │                         │                      │
```

## 14. Pros

- **Extends effective context** — run 128K context on a single GPU that has only 80GB HBM
- **Cost reduction** — use cheaper CPU memory instead of buying more GPUs
- **Flexible memory hierarchy** — trade latency for capacity across GPU/CPU/NVMe
- **Graceful degradation** — instead of OOM, performance gradually decreases
- **Enables larger batches** — oversubscribe GPU memory during peak loads
- **Better throughput per dollar** — CPU memory is 10-20x cheaper per GB than HBM

## 15. Cons

- **Latency penalty** — PCIe transfers add 10-100μs per offloaded block access
- **PCIe bandwidth bottleneck** — CPU↔GPU bandwidth (64 GB/s PCIe Gen5) is 10-15x slower than HBM bandwidth (2000+ GB/s)
- **Implementation complexity** — async transfers, prefetching, eviction policies
- **Pinned memory overhead** — CPU pinned memory can't be swapped by OS
- **Diminishing returns** — at 90% offload rate, 90% of attention time is spent on PCIe transfers
- **Not beneficial for short prompts** — overhead is not worth it for <4K contexts

## 16. Alternatives

| Alternative | Description | When to Use |
|------------|-------------|-------------|
| **No offloading** | Keep all KV on GPU | Short contexts, ample GPU memory |
| **Context window reduction** | Limit max tokens | When offloading overhead > benefit |
| **Sliding window attention** | Mistral's approach, keep recent N tokens | Locality-dominated attention |
| **KV cache quantization** | FP8 or 4-bit KV cache | Memory reduction without transfers |
| **Model parallelism** | Spread KV across GPUs | Multi-GPU setups |
| **Unified memory** | Apple MLX style, OS-managed paging | Apple Silicon hardware |

## 17. Performance Considerations

**Transfer overhead analysis:**
```
PCIe Gen5 x16:      64 GB/s theoretical, ~55 GB/s achievable
NVLink (4x H100):   900 GB/s theoretical, ~800 GB/s achievable

KV block size (Llama-3-70B): 16 tokens × 80 layers × 2 × 256 × 128 × 2 bytes = 

Let me recalculate:
For Llama-3-70B:
- num_layers = 80
- num_heads = 64 (KV heads)
- head_dim = 128
- dtype = bfloat16 (2 bytes)
- block_size = 16 tokens

KV per block = block_size * num_layers * 2 * num_kv_heads * head_dim * dtype_bytes
= 16 * 80 * 2 * 8 * 128 * 2  (GQA: 8 KV heads)
= 16 * 80 * 2 * 8 * 128 * 2
= 5,242,880 bytes ≈ 5.2 MB

Transfer time (PCIe Gen5): 5.2 MB / 55 GB/s ≈ 95 μs
NVLink: 5.2 MB / 800 GB/s ≈ 6.5 μs

For 100 block transfers per step: 9.5ms on PCIe, 0.65ms on NVLink
```

**Impact on generation speed:**
```
No offloading:         ~50 tok/s (all KV on GPU)
10% offloaded:         ~45 tok/s (5% penalty)
50% offloaded:         ~25 tok/s (50% penalty)
90% offloaded:         ~8 tok/s (84% penalty)

Rule: keep at least the last 10-20% of KV tokens on GPU for acceptable performance
```

## 18. Scaling to Millions

**For long-context RAG at scale (128K context, 10M requests/day):**

Without offloading:
- 128K KV cache per request: ~6.5 GB/seq (Llama-3-70B)
- Max concurrent: 80 / 6.5 ≈ 12 sequences per H100
- GPUs needed: 10M / (12 × 3600 × 24) × avg_seconds_per_req ≈ 160 H100s

With 80% offloading (20% KV on GPU):
- GPU KV memory per seq: 1.3 GB
- Max concurrent: 80 / 1.3 ≈ 61 sequences per H100
- GPUs needed: ~60 H100s — 2.7x reduction
- Throughput penalty: ~40% slower
- Net effect: 1.6x more throughput per dollar

**Optimal strategy:** Keep 20-30% of most recent tokens on GPU. Attention to recent tokens dominates (locality of reference), so 70% of attention compute stays GPU-resident.

## 19. Failure Scenarios

| Failure | Impact | Mitigation |
|---------|--------|------------|
| **DMA timeout** | Step hangs, no progress | Timeout watchdog, retry transfer |
| **Pinned memory exhaustion** | Cannot allocate CPU buffers | Pre-allocate at startup, fall back to pageable memory |
| **PCIe bandwidth saturation** | All transfers slow, GPU idle | Limit concurrent transfers, use transfer budget |
| **CPU OOM** | Server crash | Limit cpu_cache_blocks, swap cold blocks to NVMe |
| **Prefetch misprediction** | Wrong blocks brought to GPU, wasted bandwidth | Disable prefetching if accuracy < 40% |

## 20. Security

| Concern | Mitigation |
|---------|------------|
| **CPU memory data exposure** | KV cache on CPU may be accessible to other processes | Use encrypted memory or process-isolated memory pools |
| **Transfer timing side channel** | Transfer latency reveals cache hit/miss → sequence length | Pad transfers to uniform timing |
| **Pinned memory vulnerability** | DMA-capable memory accessible by devices | Restrict DMA access to authorized devices only |

## 21. Monitoring

**Key metrics:**
```
offload:gpu_cache_utilization    — GPU KV cache fill ratio
offload:cpu_cache_utilization    — CPU KV cache fill ratio
offload:evictions_per_step       — Blocks moved GPU→CPU per step
offload:promotions_per_step      — Blocks moved CPU→GPU per step
offload:gpu_hit_rate             — Fraction of KV accesses on GPU
offload:avg_transfer_latency_us  — Average PCIe transfer time
offload:transfer_bandwidth_util  — PCIe bandwidth usage ratio
offload:prefetch_accuracy        — Fraction of prefetches that were used
offload:attention_on_gpu_ratio   — Fraction of attention compute on GPU
offload:step_time_breakdown      — Compute vs transfer vs wait
```

**Grafana panels:**
1. GPU vs CPU KV cache utilization over time
2. Eviction/promotion rate (should be low in steady state)
3. GPU hit rate (target > 80% for acceptable latency)
4. Transfer bandwidth utilization
5. Prefetch accuracy and benefit
6. Step time breakdown (compute/transfer/wait)

**Alerts:**
- GPU hit rate < 50% → too aggressive offloading; reduce eviction
- Transfer bandwidth > 80% → PCIe bottleneck; reduce offload rate
- Prefetch accuracy < 20% → disable prefetching; pattern unpredictable
- Eviction rate > 50/step → cache thrashing; increase GPU budget
- Avg step time > 2x baseline → offloading overhead too high

## 22. Interview Questions

**Q1: When does KV cache offloading make sense vs not making sense?**
*A: Offloading makes sense when: (1) context is very long (32K+) and GPU memory is insufficient, (2) batch size is small and you want to increase it, (3) PCIe bandwidth is high (Gen5, NVLink-C2C). It doesn't make sense when: (1) context fits entirely on GPU (<8K), (2) latency is the primary concern, (3) PCIe bandwidth is limited (Gen3).*

**Q2: Explain the tradeoff between offloading rate and generation speed.**
*A: Each offloaded block adds ~95μs of PCIe transfer time (Gen5). For a 128K context with 8000 blocks (block_size=16), 80% offloaded means 6400 blocks on CPU. In each decode step, ~100 blocks are accessed for attention (locality means mostly recent blocks). If 80% of those accessed blocks are on CPU, that's 80 × 95μs = 7.6ms added per step vs ~10ms compute for the forward pass — a 76% slowdown. The tradeoff is 5x context capacity for ~2x latency.*

**Q3: How would you implement prefetching for KV cache offloading?**
*A: Use attention pattern prediction. In typical causal attention, the query attends to all previous positions equally in the first few layers, but later layers show positional locality (more weight on recent tokens). A simple prefetcher loads the next N blocks ahead of the current position (spatial locality). More advanced: use a small MLP to predict which blocks will have highest attention scores based on query position and block position features.*

**Q4: What's the impact of different memory tiers (GPU vs CPU vs NVMe) on performance?**
*A: GPU HBM: ~2 TB/s bandwidth, 80-192 GB capacity, ~10μs latency. CPU DRAM: ~50 GB/s via PCIe, 256GB-2TB capacity, ~100μs transfer latency. NVMe SSD: ~7 GB/s, 1-30TB capacity, ~1ms transfer latency. The ratio: GPU:CPU:NVMe ≈ 40:1:0.005 in bandwidth, 1:10:100 in capacity per dollar. The optimal strategy: keep 20-30% on GPU, 60-70% on CPU, and only spill to NVMe for extreme cases.*

## 23. Cheat Sheet

```bash
# Enable KV offloading in vLLM
python -m vllm.entrypoints.openai.api_server \
    --model meta-llama/Llama-3.2-3B-Instruct \
    --swap-space 16 \          # GB of CPU memory for swapped blocks
    --max-num-seqs 256 \
    --gpu-memory-utilization 0.85

# The swap-space configures CPU memory for offloaded KV blocks
# Recommended: 4-8 GB per 10B parameters

# Monitor offloading:
# vLLM metrics
vllm:num_requests_swapped   # Sequences currently swapped to CPU
vllm:gpu_cache_usage        # GPU KV cache usage (0-1)

# Check swap activity
curl http://localhost:8000/metrics | grep vllm:num_requests_swapped

# Estimate memory needed:
# Llama-3-70B, 32K context, block_size=16
# KV per block = 16 * 80 * 2 * 8 * 128 * 2 = 5.2 MB
# Total blocks for 32K = 2000
# Total KV = 2000 * 5.2 MB = 10.4 GB per sequence
# With 16GB swap per sequence: 10.4 GB on CPU, rest on GPU
# 16GB swap / (10.4 * 0.8) = ~2 sequences per 16GB swap
```

## 24. Common Mistakes

1. **Not using pinned memory** — pageable CPU memory causes double-copy (CPU→pinned→GPU), doubling transfer time. Always use `pin_memory=True`.

2. **Evicting recent tokens** — the most recent tokens are accessed every step. Never evict the last 10% of blocks. Use a "recently-used" protection set.

3. **No transfer overlap** — synchronous PCIe transfers block the GPU. Always use async transfers with CUDA streams to overlap with computation.

4. **Too many small transfers** — 100 individual 5MB transfers are slower than one 500MB transfer. Fuse contiguous blocks into larger transfer batches.

5. **Ignoring PCIe bandwidth sharing** — all GPUs on a node share the same PCIe root complex. If 8 GPUs offload simultaneously, each gets 1/8th bandwidth.

6. **No prefetching** — waiting for transfer on-demand adds latency. Prefetch predicted blocks before they're needed.

7. **Aggressive eviction under load** — when GPU memory is full, eviction creates more CPU transfers → more PCIe contention → slower generation for all sequences. Use admission control instead.

## 25. Production Best Practices

1. **Profile your attention patterns first** — measure which token positions receive the most attention weight in each layer. This informs the eviction policy. Typically, 80% of attention weight goes to the last 20% of tokens.

2. **Use the "young generation" protection** — keep the most recent N blocks (e.g., 20% of context) permanently in GPU memory. Never evict them. This covers the locality-dominated attention pattern.

3. **Benchmark PCIe bandwidth early** — use `nvidia-smi topo -m` to understand PCIe topology. If GPUs are behind different root complexes, offload bandwidth is better.

4. **Tune eviction watermark empirically** — run a sweep of eviction thresholds (70%, 80%, 85%, 90%). The optimal point is where eviction rate × transfer time = attention compute savings.

5. **Use host memory as a cache, not storage** — think of CPU as a victim cache for GPU, not the primary KV storage. The majority of KV should still be on GPU if performance matters.

6. **Implement backpressure** — if average PCIe transfer latency exceeds 500μs (indicating saturation), throttle new sequence admissions until transfers drain.

7. **Fuse small transfers** — don't transfer individual KV blocks. Accumulate and batch-transfer blocks every 4-8 steps. The latency improvement from batching is significant.

8. **Set `cpu_cache_blocks` generously** — the cost of a CPU block is ~1/1000th of GPU. Oversubscribe CPU memory 3-5x relative to expected needs. The CPU eviction path (CPU→nothing) is free.

9. **Disable offloading for latency-sensitive workloads** — if your application has a p99 latency SLO < 2s, keep all KV on GPU and reduce context length or batch size instead.

10. **Monitor for thrashing** — if eviction and promotion rates are both high (each block ping-pongs between GPU and CPU), the working set doesn't fit in GPU. Increase GPU budget or reduce concurrency.
