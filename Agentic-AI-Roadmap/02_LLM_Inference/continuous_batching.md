# Continuous Batching

## 1. What is it?

Continuous batching (also called dynamic batching or iteration-level batching) is a technique that allows an inference server to add or remove sequences from the running batch at every decoder iteration, rather than waiting for a fixed batch to complete before starting a new one.

**ELI5:** Imagine a bus that picks up passengers. Traditional batching is like saying "the bus only leaves when it's full" — everyone waits. Continuous batching is more like a city bus that stops at every station: passengers get on and off at every stop, and the bus is always moving with whoever is currently aboard.

**Technical Definition:** In continuous batching, the scheduler selects a new set of sequences to process at each forward pass iteration. Sequences that finish (reach EOS or max_tokens) are immediately removed and their KV cache blocks are freed. New sequences that arrive can be inserted on the next iteration. This maximizes GPU utilization by ensuring every iteration processes the maximum possible batch size.

## 2. Why do we need it?

**Problem:** LLMs generate tokens one at a time, and sequences have varying lengths. In static batching, all sequences in a batch must generate all their tokens before any result is returned. If one sequence needs 50 tokens and another needs 500, the first sequence waits idle for 90% of the time, wasting GPU capacity.

**Pain without it:** Without continuous batching, serving 100 concurrent users with static batch size 8 means 92 users are queued. Each batch processes at the speed of the longest sequence. GPU utilization drops to 30-40% because shorter sequences finish early and leave KV cache unused. Users experience high tail latency.

**Why companies use it:** Continuous batching increases inference throughput by 2-5x vs static batching with the same hardware. It reduces p99 latency by eliminating wait time from finished sequences. It enables efficient handling of diverse sequence lengths — the typical pattern in production workloads.

## 3. Real-world Example

- **vLLM**: Pioneered continuous batching in open source with PagedAttention, achieving 24x throughput over naive implementations.
- **HuggingFace TGI**: Implements continuous batching with dynamic batch sizing for optimal GPU utilization.
- **NVIDIA TensorRT-LLM**: Continuous batching in the inflight_batcher component of the Triton Inference Server backend.
- **OpenAI**: Uses continuous batching across their API infrastructure (inferred from GPT-4's low latency under high load).
- **Anyscale**: vLLM's continuous batching handles thousands of concurrent requests across their inference platform.
- **Together AI**: Leverages continuous batching with PagedAttention to serve 100+ models simultaneously.

## 4. Architecture Diagram (ASCII)

```
Static Batching (without continuous batching):

Time → ──────────────────────────────────────────────►
┌──────────────────────────────────────────────────────────┐
│ Batch 1:  │ Seq A │ Seq B │ Seq C │ Seq D │              │
│           │ █████████████████████████████████░░░░░░░ │
│           │ ████████████████████░░░░░░░░░░░░░░░░░░ │
│           │ ██████████████████████████████████████ │
│           │ ████████░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░ │
└──────────────────────────────────────────────────────────┘
          ↑ Seq C holds everyone up            │
          │ A, B, D finish early, idle        │
                                               │
            Batch 2 waits for Batch 1          │
                                               ▼

Continuous Batching:

Time → ──────────────────────────────────────────────►
GPU: ██████████████████████████████████████████████████████
      A│B│C│D│  A│B│C│D│  A│B│C│D│  A│B│C│D│  A│B│C│D│
      │  Decode │  Decode │  Decode │                   │
      │  step 1 │  step 2 │  step 3 │                   │
      │         │         │         │                    │
      │      D done → remove D        │                    │
      │      E arrives → add E        │                    │
      │         │         │         │                    │
      │      A│B│C│E│  Decode│  A│B│C│E│  Decode        │
```

## 5. Internal Working

**Scheduler operations per iteration:**

```
1. Receive new requests → add to waiting queue
2. Check for completed sequences → remove from running batch
   → Free KV cache blocks for completed sequences
3. Calculate available GPU budget (memory slots, compute capacity)
4. From waiting queue, pop sequences up to budget limit
   → Allocate KV cache blocks for new sequences
   → Run prefill (compute first token's logits)
5. Run decode step for all sequences in running batch
6. Sample next token for each sequence
7. Check EOS/max_tokens for each → mark completed
8. Append new tokens to sequence states
9. Return completed sequences' outputs (streaming or final)
10. Go to step 1
```

**Key data structures:**
```python
class SequenceState:
    tokens: list[int]           # Generated tokens so far
    kv_cache_blocks: list[int]  # Physical block indices
    finished: bool              # EOS or max_tokens reached
    prefix_len: int             # Length of shared prefix (if prefix caching)

class Scheduler:
    waiting_queue: deque[Sequence]     # Not yet started
    running_batch: list[SequenceState] # Currently computing
    completed: list[Sequence]          # Ready to return
```

## 6. Production Flow

```
Request  ──▶  Tokenizer  ──▶  Scheduler
                                  │
                                  ├─── waiting_queue (new requests)
                                  │
                                  ▼
                          ┌────────────────┐
                          │  Can we fit?    │
                          │  Budget check   │
                          └────┬───────────┘
                               │
                     ┌─────────┴─────────┐
                     │                   │
                ┌────▼────┐        ┌─────▼─────┐
                │ Yes:    │        │ No:       │
                │ Prefill │        │ Wait      │
                └────┬────┘        │ next iter │
                     │             └───────────┘
                     │
                     ▼
              ┌──────────────┐
              │ Decode Step   │
              │ (all running) │
              └──────┬───────┘
                     │
                     ▼
              ┌──────────────┐
              │ Sample Token │
              └──────┬───────┘
                     │
           ┌─────────┴─────────┐
           │                   │
      ┌────▼────┐        ┌────▼────┐
      │ EOS?    │        │ Continue │
      │ Return  │        │ Decode   │
      └─────────┘        └─────────┘
```

## 7. HLD (High-Level Design)

```
┌─────────────────────────────────────────────────────────────┐
│                     Continuous Batching Server                │
│                                                               │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────────┐   │
│  │ Request      │  │ Scheduler    │  │ KV Cache         │   │
│  │ Queue        │──│ (per iter)   │──│ Manager           │   │
│  │ (Redis/InMem)│  │              │  │ (Paged/Block)    │   │
│  └──────────────┘  └──────┬───────┘  └──────────────────┘   │
│                            │                                  │
│                            ▼                                  │
│                    ┌────────────────┐                        │
│                    │ GPU Executor   │                        │
│                    │ (CUDA kernel   │                        │
│                    │  launcher)     │                        │
│                    └────────────────┘                        │
└─────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────┐
│                     Per-Iteration Flow                       │
│                                                               │
│  1. Pop from queue                   ┌────────────────┐      │
│  2. Compute budget (free_blocks)     │ Scheduler Clock │      │
│  3. Add sequences to batch           │   (30ms iter)  │      │
│  4. Prepare input tensors             └────────────────┘      │
│     (padding, attention mask)                               │
│  5. Launch fused kernel                                      │
│  6. Sample tokens                                            │
│  7. Update KV block table                                    │
│  8. Return completed outputs                                 │
└─────────────────────────────────────────────────────────────┘
```

## 8. LLD (Low-Level Design)

```python
import torch
from dataclasses import dataclass, field
from typing import List, Optional
from collections import deque
import time

@dataclass
class Sequence:
    id: int
    input_ids: List[int]
    generated_ids: List[int] = field(default_factory=list)
    max_tokens: int = 512
    finished: bool = False
    arrival_time: float = 0.0

class ContinuousBatchScheduler:
    def __init__(
        self,
        max_batch_size: int = 64,
        max_num_tokens: int = 8192,
        block_size: int = 16,
        num_gpu_blocks: int = 10000,
    ):
        self.max_batch_size = max_batch_size
        self.max_num_tokens = max_num_tokens
        self.block_size = block_size
        self.free_blocks = set(range(num_gpu_blocks))
        self.waiting_queue: deque[Sequence] = deque()
        self.running: List[Sequence] = []
        self.completed: List[Sequence] = []
        self.seq_block_tables: dict[int, List[int]] = {}
        self.next_seq_id = 0
        self.total_blocks = num_gpu_blocks

    def add_request(self, input_ids: List[int], max_tokens: int = 512):
        seq = Sequence(
            id=self.next_seq_id,
            input_ids=input_ids,
            max_tokens=max_tokens,
            arrival_time=time.time(),
        )
        self.next_seq_id += 1
        self.waiting_queue.append(seq)
        return seq.id

    def get_available_budget(self) -> int:
        """Calculate how many new sequences we can add this iteration."""
        current_batch_size = len(self.running)
        can_add = self.max_batch_size - current_batch_size

        # Also check token budget
        total_tokens = sum(
            len(s.input_ids) + len(s.generated_ids)
            for s in self.running
        )
        remaining_tokens = self.max_num_tokens - total_tokens

        # Block budget
        free_block_count = len(self.free_blocks)
        seqs_can_fit_blocks = free_block_count // (
            self.max_num_tokens // self.block_size
        )

        return min(can_add, remaining_tokens // 256, seqs_can_fit_blocks)

    def schedule_iteration(self):
        """Called once per decoder iteration."""

        # 1. Remove finished sequences
        newly_finished = [s for s in self.running if s.finished]
        for seq in newly_finished:
            self._free_blocks(seq)
            self.completed.append(seq)
        self.running = [s for s in self.running if not s.finished]

        # 2. Add new sequences
        budget = self.get_available_budget()
        for _ in range(budget):
            if not self.waiting_queue:
                break
            seq = self.waiting_queue.popleft()
            self._allocate_blocks(seq, len(seq.input_ids) // self.block_size + 1)
            self.running.append(seq)

        # 3. If nothing to run, skip
        if not self.running:
            return None

        # 4. Prepare batched input
        #    (Concatenate all sequences' next tokens, build attention mask)
        input_tokens = torch.tensor([
            s.input_ids + s.generated_ids[-1:] if s.generated_ids
            else s.input_ids[-1:]
            for s in self.running
        ])

        return input_tokens  # → fed to model forward pass

    def _allocate_blocks(self, seq: Sequence, num_blocks: int):
        blocks = sorted(list(self.free_blocks))[:num_blocks]
        self.free_blocks -= set(blocks)
        self.seq_block_tables[seq.id] = blocks

    def _free_blocks(self, seq: Sequence):
        if seq.id in self.seq_block_tables:
            self.free_blocks.update(self.seq_block_tables[seq.id])
            del self.seq_block_tables[seq.id]

    def on_token_generated(self, seq_id: int, token: int):
        """Called after model forward pass with the sampled token."""
        for seq in self.running:
            if seq.id == seq_id:
                seq.generated_ids.append(token)
                if token == 0 or len(seq.generated_ids) >= seq.max_tokens:
                    seq.finished = True
                # Allocate KV block if needed
                total_tokens = len(seq.input_ids) + len(seq.generated_ids)
                needed = total_tokens // self.block_size + 1
                current = len(self.seq_block_tables[seq_id])
                if needed > current:
                    self._allocate_blocks(seq, needed - current)
                break
```

## 9. Python Implementation

```python
import asyncio
import torch
from fastapi import FastAPI, HTTPException
from pydantic import BaseModel, Field
from typing import List, Optional, Dict, Any
from dataclasses import dataclass
from collections import deque
import time
import logging

logging.basicConfig(level=logging.INFO)
logger = logging.getLogger("continuous-batching")

app = FastAPI(title="Continuous Batching Server")

# Simplified model interface
class MockModel:
    def __init__(self, vocab_size=32000, max_batch=64):
        self.vocab_size = vocab_size
        self.max_batch = max_batch

    @torch.inference_mode()
    def forward(self, input_ids: torch.Tensor) -> torch.Tensor:
        batch_size = input_ids.shape[0]
        return torch.randn(batch_size, 1, self.vocab_size)

model = MockModel()
response_cache: dict[int, str] = {}

@dataclass
class Sequence:
    id: int
    prompt: str
    input_ids: List[int]
    generated_tokens: List[int] = None
    max_tokens: int = 512
    finished: bool = False
    event: asyncio.Event = None

    def __post_init__(self):
        self.generated_tokens = []
        self.event = asyncio.Event()

class Scheduler:
    def __init__(self, max_batch: int = 8):
        self.max_batch = max_batch
        self.waiting: deque = deque()
        self.running: list = []
        self.completed: dict = {}

    def submit(self, seq: Sequence):
        self.waiting.append(seq)

    def step(self):
        # Remove finished
        done = [s for s in self.running if s.finished]
        for s in done:
            s.event.set()
        self.running = [s for s in self.running if not s.finished]

        # Add new
        budget = self.max_batch - len(self.running)
        for _ in range(budget):
            if not self.waiting:
                break
            self.running.append(self.waiting.popleft())

        if not self.running:
            return None

        # Process batch
        tokens = torch.tensor([
            s.input_ids[-1:] if s.generated_tokens
            else s.input_ids[-1:]
            for s in self.running
        ])
        logits = model.forward(tokens)

        for i, seq in enumerate(self.running):
            next_token = torch.argmax(logits[i, -1, :]).item()
            seq.generated_tokens.append(next_token)
            if len(seq.generated_tokens) >= seq.max_tokens:
                seq.finished = True

        return logits

scheduler = Scheduler(max_batch=8)

@app.on_event("startup")
async def start_scheduler():
    async def loop():
        while True:
            scheduler.step()
            await asyncio.sleep(0.01)  # ~10ms per iteration
    asyncio.create_task(loop())

class ChatRequest(BaseModel):
    prompt: str
    max_tokens: int = Field(default=128, ge=1, le=2048)

@app.post("/generate")
async def generate(request: ChatRequest):
    seq_id = int(time.time() * 1_000_000)
    seq = Sequence(
        id=seq_id,
        prompt=request.prompt,
        input_ids=list(range(10, 20)),  # Mock tokenization
        max_tokens=request.max_tokens,
    )
    scheduler.submit(seq)
    await seq.event.wait()

    text = " ".join(str(t) for t in seq.generated_tokens)
    return {
        "id": seq_id,
        "text": text,
        "tokens": len(seq.generated_tokens),
    }

@app.get("/queue-depth")
async def queue_depth():
    return {
        "waiting": len(scheduler.waiting),
        "running": len(scheduler.running),
        "max_batch": scheduler.max_batch,
    }
```

## 10. Folder Structure

```
continuous_batching/
├── Dockerfile
├── config.yaml
├── serve.py
├── cb_inference/
│   ├── __init__.py
│   ├── server.py             # FastAPI server
│   ├── scheduler.py          # Continuous batching scheduler
│   ├── sequence.py           # Sequence state management
│   ├── block_manager.py      # KV cache block allocation
│   ├── batch.py              # Batch preparation (padding, masking)
│   ├── metrics.py            # Performance monitoring
│   └── config.py             # Pydantic configuration
├── tests/
│   ├── test_scheduler.py
│   ├── test_batch.py
│   └── test_integration.py
└── k8s/
    ├── deployment.yaml
    └── hpa.yaml
```

## 11. Configuration

```yaml
scheduler:
  max_batch_size: 64
  max_num_tokens: 8192
  max_waiting_time_ms: 5        # Max queue wait before forcing iteration

kv_cache:
  block_size: 16                # Tokens per block
  num_blocks: 5000              # Total GPU memory budget in blocks
  preemption_mode: recompute    # recompute or swap

prefill:
  budget_ratio: 0.3             # Portion of batch budget for new sequences
  max_prefill_tokens: 4096      # Max tokens in prefill for one sequence

decode:
  max_decode_tokens: 2048       # Max output tokens per sequence
  stop_on_eos: true

monitoring:
  log_every_n_iters: 100
  metrics_port: 8001
```

## 12. Flowchart

```
                   ┌──────────────┐
                   │ New Request  │
                   └──────┬───────┘
                          │
                          ▼
                  ┌───────────────┐
                  │ Tokenize &    │
                  │ Enqueue       │
                  └──────┬───────┘
                         │
                         ▼
            ┌────────────────────────┐
            │ Scheduler Iteration    │
            │ (every ~10-50ms)       │
            └──────┬─────────────────┘
                   │
          ┌────────┴────────┐
          │                 │
    ┌─────▼─────┐    ┌──────▼──────┐
    │ Collect    │    │ Free blocks │
    │ completed  │    │ of finished │
    │ sequences  │    │ sequences   │
    └─────┬─────┘    └──────┬──────┘
          │                 │
          └────────┬────────┘
                   │
          ┌────────▼────────┐
          │ Budget Check    │
          │ free_gpu_mem    │
          │ free_blocks     │
          │ batch_slots     │
          └────────┬────────┘
                   │
          ┌────────▼────────┐
          │ Pop from queue  │
          │ up to budget    │
          └────────┬────────┘
                   │
          ┌────────▼────────┐
          │ Prepare batch:  │
          │ - pad tokens    │
          │ - build attn_msk│
          │ - set blocks    │
          └────────┬────────┘
                   │
          ┌────────▼────────┐
          │ GPU Forward     │
          │ (prefill new +  │
          │ decode running) │
          └────────┬────────┘
                   │
          ┌────────▼────────┐
          │ Sample + Append │
          │ tokens per seq  │
          └────────┬────────┘
                   │
          ┌────────▼────────┐
          │ Check EOS/max   │
          │ Mark completed  │
          └──────┬──────────┘
                 │
          ┌──────▼──────────┐
          │ Return outputs  │
          │ (stream or batch)│
          └─────────────────┘
```

## 13. Sequence Diagram

```
Request A    Request B    Scheduler      GPU          Response
   │            │            │            │              │
   │───req A───▶│            │            │              │
   │            │───req B───▶│            │              │
   │            │            │            │              │
   │            │      ┌─────┴─────┐     │              │
   │            │      │ Iter 1    │     │              │
   │            │      │ Prefill A │────▶│ prefill A    │
   │            │      │ Prefill B │────▶│ prefill B    │
   │            │      └─────┬─────┘     │              │
   │            │            │            │              │
   │            │      ┌─────┴─────┐     │              │
   │            │      │ Iter 2    │     │              │
   │            │      │ Decode A+B│────▶│ decode       │
   │            │      │ token 1   │────▶│              │
   │            │      └─────┬─────┘     │              │
   │            │            │            │              │
   │            │        req C arrives   │              │
   │            │            │            │              │
   │            │      ┌─────┴─────┐     │              │
   │            │      │ Iter 3    │     │              │
   │            │      │ Prefill C │────▶│ prefill C    │
   │            │      │ Decode A+B│────▶│ decode A+B   │
   │            │      │ token 2   │     │              │
   │            │      └─────┬─────┘     │              │
   │            │            │            │              │
   │◀───resp A──┼────────────│────────────│──────────────│
   │            │◀───resp B──│────────────│──────────────│
```

## 14. Pros

- **High throughput** — 2-5x improvement over static batching
- **Low tail latency** — no "wait for longest sequence" problem
- **Efficient memory** — KV cache freed immediately on sequence completion
- **Flexible** — handles diverse sequence lengths naturally
- **Good GPU utilization** — GPU always busy with full batch
- **Smooth performance** — throughput doesn't drop when one sequence is very long
- **Lower cost** — same throughput with fewer GPUs

## 15. Cons

- **Higher complexity** — scheduler must manage per-sequence state
- **Larger TTFT** — new sequences may wait for the next iteration boundary
- **Memory fragmentation** — if block management isn't careful (PagedAttention fixes this)
- **Padding overhead** — variable-length sequences require padding or special kernels
- **Scheduling latency** — scheduler overhead becomes significant at very high batch sizes
- **Debugging difficulty** — reproducing issues is harder with dynamic scheduling

## 16. Alternatives

| Alternative | Description | When to Use |
|------------|-------------|-------------|
| **Static batching** | Fixed batch, all sequences start and end together | Simple workloads, throughput not critical |
| **Dynamic batching** | Batch formed from queue, but runs to completion per sequence | Very short requests, homogeneous lengths |
| **In-flight batching** | NVIDIA Triton's name for continuous batching | NVIDIA GPU stacks |
| **Spatial batching** | Batch across model dimensions (expert parallelism) | MoE models (Mixtral) |
| **Speculative batching** | Batch with draft model verification | Very high throughput needs |

## 17. Performance Considerations

**Optimal batch size:**
```
Rule: max_batch_size = GPU_memory / (KV_cache_per_token * max_seq_len)

For H100 80GB with Llama-3-8B (bfloat16):
- Weights: 16GB
- KV cache per token: ~0.5MB (32 layers × 2 × 4096 × 2 bytes)
- Max context 4096 → 2GB per sequence
- 80 - 16 = 64GB available
- 64GB / 2GB = 32 sequences max at 4096 context
- With short context (512): 64GB / 256MB = 256 sequences
```

**Iteration time vs batch size:**
```
Batch 4:    ~20ms per iteration → 50 tok/s per seq, 200 tok/s total
Batch 16:   ~25ms per iteration → 40 tok/s per seq, 640 tok/s total
Batch 64:   ~40ms per iteration → 25 tok/s per seq, 1600 tok/s total

The tradeoff: larger batch = higher total throughput, but lower per-sequence speed
```

**Scheduling overhead:**
```
For N sequences per iteration:
- Queue operations: O(1) with deque
- Block allocation: O(num_blocks_to_allocate)
- Batch preparation: O(N * max_seq_len) — can be expensive
- Mask computation: O(N²) for attention mask — use sparse kernels
```

## 18. Scaling to Millions

**Horizontal scaling:**
1. **Shard by model** — separate continuous batching servers for different model sizes
2. **Least-connections routing** — load balancer sends to server with smallest waiting queue
3. **Request classification** — separate batch pools for short and long sequences
4. **Auto-scaling** — scale replicas based on `queue_depth` metric

**For 5M requests/day:**
```
Requests/sec:      ~60
Avg tokens/req:    512 input + 256 output = 768
Tokens/day:        ~3.8B
GPU requirements:  ~4 H100-80GB (with continuous batching)
vs static:         ~12 H100-80GB (without continuous batching)
Savings:           3x GPU reduction
```

**Bottleneck:**
The scheduler becomes CPU-bound at very high batch sizes (>256). Mitigate by:
- Batching scheduling operations (process 8 sequences at a time)
- Using C++/CUDA for batch preparation (block table manipulation)
- Pre-allocating tensor buffers for batch input

## 19. Failure Scenarios

| Failure | Impact | Mitigation |
|---------|--------|------------|
| **Scheduler livelock** | No progress, CPU 100% | Max iterations per sequence guard |
| **Memory leak** | Gradual OOM over hours | Periodic KV block table defragmentation |
| **Oversized prefill** | Single sequence uses all GPU memory | Cap `max_prefill_tokens`, preempt long prefills |
| **Batch corruption** | Wrong token mapped to wrong sequence | Sequence ID validation on each iteration |
| **Scheduler crash** | All in-flight requests lost | Persistent queue (Redis), checkpoint running states |

## 20. Security

| Concern | Mitigation |
|---------|------------|
| **DoS via long sequences** | Enforce max_tokens per request, max queue depth per user |
| **Scheduler starvation** | Fair scheduling across users/API keys (weighted round-robin) |
| **Information leakage** | Don't leak other sequences' tokens through timing side channels |
| **Priority inversion** | High-priority requests waiting behind low-priority ones |
| **Resource exhaustion** | Enforce per-user queue depth and max concurrency limits |

## 21. Monitoring

**Key metrics:**
```
cb:running_sequences        — Currently in batch
cb:waiting_sequences        — Queue depth
cb:iteration_time_ms        — Time per scheduler iteration
cb:batch_utilization_%      — Current batch size / max batch size
cb:prefill_per_iteration    — Number of prefills in current iteration
cb:decode_per_iteration     — Number of decodes in current iteration
cb:throughput_tokens_per_s  — Aggregate token generation rate
cb:avg_seq_latency_ms       — Average request end-to-end latency
cb:preemptions_total        — Number of preempted sequences
```

**Grafana panels:**
1. Running vs waiting sequences over time
2. Batch utilization heatmap
3. Iteration time breakdown (scheduling vs forward vs sampling)
4. TTFT and TPOT histograms
5. Queue depth by priority/user

**Alerts:**
- Running sequences = 0 with waiting > 0 → scheduler stalled
- Iteration time > 100ms → kernel or batch size issue
- Preemption rate > 5% → reduce load or increase GPU memory
- Batch utilization < 30% for > 1 minute → too many GPUs or reduce replicas

## 22. Interview Questions

**Q1: How does continuous batching improve throughput compared to static batching?**
*A: Static batching waits for all sequences in a batch to complete before returning results. If sequences have varying lengths (e.g., 50 vs 500 tokens), short sequences sit idle while longer ones finish — wasting GPU memory and compute. Continuous batching removes completed sequences immediately and adds new ones at every iteration, keeping the GPU fully utilized and increasing throughput 2-5x.*

**Q2: What's the tradeoff between batch size and per-sequence latency?**
*A: Larger batches process more total tokens per second (higher throughput) but each individual sequence waits longer — both for scheduling and because the forward pass is slower with more sequences. The optimal batch size depends on your latency SLO. For real-time chat (TTFT < 500ms), use smaller batches (8-32). For batch processing, use larger batches (64-256).*

**Q3: How does continuous batching interact with PagedAttention?**
*A: PagedAttention's block-based KV cache is essential for continuous batching. When a sequence finishes, its blocks are freed and can be immediately allocated to a new sequence. Without PagedAttention (contiguous KV cache), freeing memory is slower and fragmentation prevents efficient reuse. vLLM's combination of continuous batching + PagedAttention is why it achieves 24x throughput improvement.*

**Q4: Describe how you would implement priority-aware continuous batching.**
*A: Two approaches: (1) Multiple priority queues — each priority level has its own queue, scheduler drains higher-priority queues first up to a max batch fraction (e.g., 70% high, 30% low). (2) Weighted scheduling — each sequence has a weight, scheduler maximizes weighted throughput subject to fairness constraints. The challenge is preventing starvation while maintaining GPU efficiency.*

## 23. Cheat Sheet

```python
# Key parameters
max_batch_size = 64            # Maximum sequences per iteration
max_num_tokens = 8192          # Maximum tokens in a single batch
block_size = 16                # KV cache block size
max_prefill_tokens = 4096      # Max prefill length for a new sequence

# Scheduler pseudo-code
def schedule_iteration():
    free_completed()
    add_new_sequences(up_to_budget())
    if no_running: return
    prepare_batch()
    gpu_forward()
    sample_tokens()
    check_eos()

# Budget calculation
budget = min(
    max_batch - current_batch,
    free_gpu_memory / kv_cache_per_token,
    remaining_tokens_budget,
)

# Metrics to watch
# - iteration_time: target < 50ms
# - batch_utilization: target > 70%
# - preemption_rate: target < 1%
```

## 24. Common Mistakes

1. **Too large batch sizes for latency-sensitive apps** — batch size 64 gives great throughput but each sequence waits for all 63 others. For chatbots, batch size 8-16 is better.

2. **Not limiting prefill length** — a single sequence with 32K prompt will monopolize GPU. Always cap `max_prefill_tokens`.

3. **Preemption starvation** — if preempted sequences are always deprioritized, they never complete. Use aging (increase priority over time).

4. **Neglecting scheduler overhead** — at 200+ sequences per iteration, Python scheduler overhead dominates. Use C++ extensions or slice scheduling.

5. **Uniform batch preparation** — padding all sequences to max length wastes memory. Use ragged tensor operations or varlen attention.

6. **Ignoring NCCL in multi-GPU** — for tensor-parallel batches, all-reduce must happen per iteration. Ensure NCCL communication doesn't become the bottleneck.

## 25. Production Best Practices

1. **Set `max_batch_size` empirically** — benchmark latency vs throughput for batch sizes 4, 8, 16, 32, 64, 128. Pick the point before diminishing returns.

2. **Use separate prefill and decode budgets** — allocate 30% of batch capacity to new prefills, 70% to ongoing decodes. Prevents prefill-heavy batches from starving decodes.

3. **Implement queue aging** — add a timestamp to each waiting sequence; if waiting > 5 seconds, promote to front of queue to prevent starvation.

4. **Monitor block fragmentation** — over time, freed blocks become fragmented. Track `free_blocks / total_blocks` ratio. If >20% fragmentation, trigger defragmentation.

5. **Use priority queues for multi-tenant** — separate free-tier (batch queue) from paid-tier (priority queue). Dedicate a minimum batch fraction to each tier.

6. **Pre-allocate tensor buffers** — don't allocate new tensors per iteration. Pre-allocate max-sized buffers and slice into them.

7. **Use `torch.compile` for batch preparation** — padding and masking operations can be compiled with `torch.compile` for 2-3x speedup.

8. **Set `max_waiting_time_ms`** — don't wait for a full batch if some sequences have been waiting too long. Force an iteration after 5ms.

9. **Warm up CUDA graphs** — after determining the most common batch sizes during startup, pre-compile CUDA graphs for those sizes.

10. **Implement graceful degradation** — if queue depth exceeds threshold, switch to a smaller model or enable additional quantization to maintain throughput.
