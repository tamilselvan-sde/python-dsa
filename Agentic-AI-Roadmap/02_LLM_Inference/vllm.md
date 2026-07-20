# vLLM Inference Engine

> Author: **Tamilselvan** · ✉️ tamilselvan.sde@gmail.com · 🔗 [LinkedIn](https://www.linkedin.com/in/tamilselvan-ai/)
>

## 1. What is it?

vLLM is an open-source high-performance inference engine for large language models, developed at UC Berkeley. It is purpose-built for serving LLMs with minimal latency and maximum throughput.

**ELI5:** Imagine a library where every time someone returns a book, you remember exactly where it was on the shelf so the next person can grab it instantly. vLLM does the same for the "memory" (KV cache) an LLM uses when generating text, so it doesn't waste time recalculating things.

**Technical Definition:** vLLM is a Python-based inference engine that uses PagedAttention to manage KV cache memory efficiently, supports continuous batching, tensor parallelism, and prefix caching to deliver up to 24x higher throughput than naive HuggingFace Transformers implementations.

## 2. Why do we need it?

**Problem:** Serving LLMs is memory-bound. The KV cache for a single 13B parameter model with 2048 tokens requires ~1.6GB of GPU memory. With naive implementations, 60–80% of this memory is wasted due to fragmentation and static allocation. This means fewer concurrent requests, higher cost per token, and unacceptable latency.

**Pain without it:** Without vLLM, production teams resort to hand-rolled optimizations, suffer from OOM errors during traffic spikes, and waste GPU dollars on underutilized hardware. Companies running naive inference might serve 10 concurrent requests when vLLM could handle 60+ on the same hardware.

**Why companies use it:** It drops in as a FastAPI server, supports OpenAI-compatible API, handles model warmup, and eliminates memory fragmentation — reducing GPU count by 3-5x for the same traffic load.

## 3. Real-world Example

- **OpenAI** (early inspiration): The PagedAttention technique was inspired by how operating systems handle virtual memory paging, which is what OS kernels do — the same concept OpenAI uses internally.
- **Together AI**: Uses vLLM as their primary inference engine across thousands of GPUs serving Mixtral, Llama, and other open models.
- **Anyscale**: The company behind vLLM, uses it to power production LLM serving for enterprise customers.
- **Perplexity AI**: Runs vLLM behind their search-based LLM service to serve millions of queries.
- **Microsoft Azure**: Integrates vLLM into Azure ML model serving for efficient Llama-2/3 deployments.
- **Amazon SageMaker**: Provides a vLLM container as a first-party inference option.

## 4. Architecture Diagram (ASCII)

```
┌─────────────────────────────────────────────────────┐
│                    Client Requests                    │
└────────────────────┬────────────────────────────────┘
                     │
                     ▼
┌─────────────────────────────────────────────────────┐
│                 API Server (FastAPI)                  │
│           OpenAI-compatible /v1/chat/completions      │
└────────────────────┬────────────────────────────────┘
                     │
                     ▼
┌─────────────────────────────────────────────────────┐
│                  LLM Engine (AsyncLLMEngine)          │
│                                                       │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐ │
│  │  Scheduler   │  │  Block Mgr   │  │ Worker Pool  │ │
│  └──────┬───────┘  └──────┬───────┘  └──────┬───────┘ │
│         │                 │                  │         │
│         ▼                 ▼                  ▼         │
│  ┌─────────────────────────────────────────────────┐  │
│  │           PagedAttention Manager                 │  │
│  │  ┌─────┐ ┌─────┐ ┌─────┐ ┌─────┐ ┌─────┐       │  │
│  │  │K/V  │ │K/V  │ │K/V  │ │K/V  │ │K/V  │       │  │
│  │  │Block│ │Block│ │Block│ │Block│ │Block│       │  │
│  │  └─────┘ └─────┘ └─────┘ └─────┘ └─────┘       │  │
│  └─────────────────────────────────────────────────┘  │
└────────────────────┬────────────────────────────────┘
                     │
                     ▼
┌─────────────────────────────────────────────────────┐
│                 GPU Workers (ray/tp)                  │
│  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐ │
│  │  GPU 0   │ │  GPU 1   │ │  GPU 2   │ │  GPU 3   │ │
│  │ (Shard 0)│ │ (Shard 1)│ │ (Shard 2)│ │ (Shard 3)│ │
│  └──────────┘ └──────────┘ └──────────┘ └──────────┘ │
└─────────────────────────────────────────────────────┘
```

## 5. Internal Working

vLLM's core innovation is **PagedAttention**, which divides the KV cache into fixed-size blocks (typically 16 tokens). Each attention computation references a logical block table, similar to how an OS manages virtual memory pages.

**Key components:**
1. **Block Manager:** Allocates and manages physical GPU memory blocks. Tracks free blocks, reference counts, and copy-on-write for shared prefixes.
2. **Scheduler:** Decides which sequences to run in each iteration based on priority, preemption mode (swap or recompute), and available blocks.
3. **Worker Pool:** Distributes attention computations across GPUs when using tensor parallelism.
4. **Cache Engine:** Manages the GPU KV cache blocks, handles allocation/deallocation.

**Decoding flow:**
1. Request arrives → Scheduler adds it to waiting queue
2. Scheduler selects next batch of sequences to run
3. For each sequence, Block Manager ensures physical blocks exist for new token
4. GPU Workers compute attention using PagedAttention (scattered reads from non-contiguous blocks)
5. Output logits are sampled, new token appended
6. If sequence completes → blocks are freed; if blocks needed → preempt lowest-priority sequence

## 6. Production Flow

```
Request → Auth → Rate Limit → Queue → Scheduler → Block Alloc → GPU Infer → Tokenize → Response
                                         │                              │
                                         └── Preemption (swap/recomp) ─┘
```

In production:
- Requests hit an API gateway (Kong/Envoy) before reaching vLLM
- Gateway handles auth, rate limiting, and canary routing
- vLLM exposes Prometheus metrics for queue depth, throughput, TTFT, TPOT
- A load balancer distributes across multiple vLLM replicas
- Each replica is pinned to a specific GPU or GPU group (via CUDA_VISIBLE_DEVICES)

## 7. HLD (High-Level Design)

```
┌─────────────┐  ┌─────────────┐  ┌─────────────┐
│   Client    │  │   Client    │  │   Client    │
└──────┬──────┘  └──────┬──────┘  └──────┬──────┘
       │                │                │
       └────────────────┼────────────────┘
                        │
               ┌────────▼────────┐
               │   API Gateway   │
               │ (Kong/Envoy/Traefik) │
               └────────┬────────┘
                        │
               ┌────────▼────────┐
               │  Load Balancer  │
               └────────┬────────┘
                        │
        ┌───────────────┼───────────────┐
        │               │               │
┌───────▼───────┐ ┌─────▼───────┐ ┌─────▼───────┐
│  vLLM Node 1  │ │ vLLM Node 2 │ │ vLLM Node N │
│  (H100:80GB)  │ │ (H100:80GB) │ │ (H100:80GB) │
└───────┬───────┘ └─────┬───────┘ └─────┬───────┘
        │               │               │
        └───────────────┼───────────────┘
                        │
               ┌────────▼────────┐
               │   Prometheus    │
               │  + Grafana      │
               └─────────────────┘
```

## 8. LLD (Low-Level Design)

**Core classes:**

```python
class BlockManager:
    """Manages physical GPU memory blocks."""
    def __init__(self, block_size: int = 16, num_gpu_blocks: int):
        self.free_blocks = list(range(num_gpu_blocks))
        self.block_tables = {}  # seq_id → list of block_ids
        self.ref_counts = defaultdict(int)

    def allocate(self, seq_id: int, num_blocks: int) -> list[int]:
        blocks = self.free_blocks[:num_blocks]
        self.free_blocks = self.free_blocks[num_blocks:]
        self.block_tables[seq_id] = blocks
        for b in blocks:
            self.ref_counts[b] += 1
        return blocks

    def free(self, seq_id: int):
        for b in self.block_tables.get(seq_id, []):
            self.ref_counts[b] -= 1
            if self.ref_counts[b] == 0:
                self.free_blocks.append(b)
        del self.block_tables[seq_id]
```

**Scheduling policy:** vLLM uses a "first-come-first-served with starvation prevention" policy. New requests get a maximum of 2 iterations before preempting longer-running ones to ensure no request waits indefinitely.

**Memory budget calculation:**
```
num_blocks = free_gpu_memory / (block_size * num_layers * 2 * dtype_bytes)
```

## 9. Python Implementation

```python
import asyncio
from fastapi import FastAPI, HTTPException
from pydantic import BaseModel, Field
from typing import List, Optional, Dict, Any
from vllm import AsyncLLMEngine, SamplingParams
from vllm.engine.arg_utils import AsyncEngineArgs
import time
import logging

logging.basicConfig(level=logging.INFO)
logger = logging.getLogger("vllm-server")

app = FastAPI(title="vLLM Inference Server")

class ChatMessage(BaseModel):
    role: str
    content: str

class ChatRequest(BaseModel):
    model: str = "default"
    messages: List[ChatMessage]
    temperature: float = Field(default=0.7, ge=0.0, le=2.0)
    max_tokens: int = Field(default=512, ge=1, le=8192)
    top_p: float = Field(default=0.95, ge=0.0, le=1.0)
    stream: bool = False

class Usage(BaseModel):
    prompt_tokens: int
    completion_tokens: int
    total_tokens: int

class ChatResponse(BaseModel):
    id: str
    object: str = "chat.completion"
    created: int
    model: str
    choices: List[Dict[str, Any]]
    usage: Usage

engine = None

@app.on_event("startup")
async def startup():
    global engine
    args = AsyncEngineArgs(
        model="meta-llama/Meta-Llama-3-8B-Instruct",
        tensor_parallel_size=1,
        max_num_seqs=256,
        max_num_batched_tokens=8192,
        gpu_memory_utilization=0.90,
        dtype="bfloat16",
        trust_remote_code=True,
    )
    engine = AsyncLLMEngine.from_engine_args(args)
    logger.info("vLLM engine initialized")

@app.post("/v1/chat/completions", response_model=ChatResponse)
async def chat_completions(request: ChatRequest):
    try:
        prompt = format_chat_template(request.messages)
        sampling_params = SamplingParams(
            temperature=request.temperature,
            max_tokens=request.max_tokens,
            top_p=request.top_p,
        )
        start = time.time()
        result = await engine.generate(prompt, sampling_params)
        latency = time.time() - start

        output = result.outputs[0]
        return ChatResponse(
            id=f"chatcmpl-{int(time.time())}",
            created=int(time.time()),
            model=request.model,
            choices=[{
                "index": 0,
                "message": {"role": "assistant", "content": output.text},
                "finish_reason": output.finish_reason,
            }],
            usage=Usage(
                prompt_tokens=len(result.prompt_token_ids),
                completion_tokens=len(output.token_ids),
                total_tokens=len(result.prompt_token_ids) + len(output.token_ids),
            ),
        )
    except Exception as e:
        logger.error(f"Inference failed: {e}", exc_info=True)
        raise HTTPException(status_code=500, detail=str(e))

def format_chat_template(messages: List[ChatMessage]) -> str:
    return engine.tokenizer.apply_chat_template(
        [m.model_dump() for m in messages],
        tokenize=False,
        add_generation_prompt=True,
    )
```

## 10. Folder Structure

```
vllm_project/
├── Dockerfile
├── requirements.txt
├── config.yaml
├── serve.py                  # Entry point
├── vllm_engine/
│   ├── __init__.py
│   ├── server.py             # FastAPI server
│   ├── config.py             # Pydantic config models
│   ├── middleware.py          # Auth, rate limiting
│   └── metrics.py            # Prometheus metrics
├── models/
│   ├── __init__.py
│   ├── model_registry.py     # Model versioning & loading
│   └── quantization.py       # AWQ/GPTQ loaders
├── utils/
│   ├── __init__.py
│   ├── tokenizer.py          # Chat template utils
│   └── logging.py            # Structured logging
├── k8s/
│   ├── deployment.yaml
│   ├── service.yaml
│   ├── hpa.yaml
│   └── ingress.yaml
└── tests/
    ├── test_server.py
    └── test_engine.py
```

## 11. Configuration

```yaml
model: meta-llama/Meta-Llama-3-70B-Instruct
tensor_parallel_size: 4
dtype: bfloat16
max_model_len: 8192
gpu_memory_utilization: 0.92
max_num_seqs: 256
max_num_batched_tokens: 8192

# Scheduling
scheduler_policy: fcfs  # first-come-first-served
preemption_mode: recompute  # or "swap"
swap_space: 4  # GB

# Quantization
quantization: awq  # or gptq, fp8
kv_cache_dtype: fp8_e5m2

# Serving
host: 0.0.0.0
port: 8000
max_logprobs: 5
enforce_eager: false
disable_log_stats: false
```

## 12. Flowchart

```
                   ┌──────────────┐
                   │  New Request │
                   └──────┬───────┘
                          │
                          ▼
              ┌──────────────────────┐
              │    Tokenize Prompt   │
              └──────┬───────────────┘
                     │
                     ▼
              ┌──────────────────────┐
              │   Schedule Sequence  │──── Preempt if blocks needed
              └──────┬───────────────┘
                     │
                     ▼
              ┌──────────────────────┐
              │  Allocate KV Blocks  │
              └──────┬───────────────┘
                     │
                     ▼
              ┌──────────────────────┐
              │   Run Prefill Pass   │────── Compute logits for prompt
              └──────┬───────────────┘
                     │
                     ▼
              ┌──────────────────────┐
              │   Sample Next Token  │
              └──────┬───────────────┘
                     │
                     ▼
         ┌───────────┴───────────┐
         │                       │
    ┌────▼────┐           ┌──────▼──────┐
    │  EOS?   │           │ Out of KV?  │
    └────┬────┘           └──────┬──────┘
         │                       │
    ┌────┴────┐                  │
    │ Done    │                  │
    └─────────┘     ┌────────────▼───────────┐
                    │  Preempt next sequence │
                    │  or swap to CPU        │
                    └────────────┬───────────┘
                                 │
                    ┌────────────▼───────────┐
                    │ Continue Decode Loop   │
                    └────────────────────────┘
```

## 13. Sequence Diagram

```
Client          FastAPI         AsyncLLME      Scheduler      BlockMgr       GPU Worker
  │                │                │              │              │              │
  │───Request────▶│                │              │              │              │
  │                │───generate───▶│              │              │              │
  │                │                │───add_seq──▶│              │              │
  │                │                │              │───alloc────▶│              │
  │                │                │              │◀───blocks───│              │
  │                │                │───step──────▶│              │              │
  │                │                │              │───────execute───▶ pref───▶│
  │                │                │              │◀────────tokens────────────│
  │                │                │───step──────▶│              │              │
  │                │                │              │───────execute───▶ decode─▶│
  │                │                │              │◀────────token────────────│
  │                │                │   (repeat for max_tokens or until EOS)  │
  │                │◀──result─────│              │              │              │
  │◀──Response────│                │              │              │              │
```

## 14. Pros

- **Up to 24x throughput** improvement over naive HF Transformers
- **Near-zero memory waste** via PagedAttention block management
- **OpenAI-compatible API** — drop-in replacement for existing clients
- **Supports continuous batching** — dynamically add/remove sequences per iteration
- **Built-in tensor/pipeline parallelism** via Ray
- **Prefix caching** — share KV blocks across requests with common prefixes
- **Quantization support** — AWQ, GPTQ, FP8, SqueezeLLM
- **Active development** — 30K+ GitHub stars, constant performance improvements

## 15. Cons

- **CUDA-only** — requires NVIDIA GPUs; no AMD/Apple Silicon support
- **Memory overhead** for block tables (tens of MB per sequence)
- **Cold start latency** — model loading takes 30-120 seconds for large models
- **No built-in multi-modal** — images/audio require separate service
- **Complex debugging** — distributed tracing across GPU workers is challenging
- **Limited scheduling policies** — only FCFS; no priority queues or QoS

## 16. Alternatives

| Engine | Strengths | Weaknesses |
|--------|-----------|------------|
| **SGLang** | Structured generation, RadixAttention | Less mature, smaller community |
| **TGI** (HuggingFace) | Tight HF integration, simpler setup | Slower, less memory efficient |
| **TensorRT-LLM** | NVIDIA-optimized, best latency | Complex build, vendor lock-in |
| **MLC-LLM** | Multi-platform (CUDA/Metal/WebGPU) | Smaller ecosystem, fewer models |
| **llama.cpp** | CPU-friendly, no GPU needed | Limited to small models on CPU |
| **OpenAI API** | Zero ops, best quality | Costly, vendor lock-in |

## 17. Performance Considerations

**Throughput vs Latency tradeoff:**
- Larger batch sizes → higher throughput, higher TTFT (time to first token)
- vLLM defaults to 256 max sequences — tune based on your latency SLO
- For real-time chat: lower batch size (64-128), lower max_num_batched_tokens

**Memory tuning:**
- `gpu_memory_utilization=0.90` leaves 10% for activations and weights
- For 70B models with 8K context: requires ~140GB for weights, ~40GB for KV cache
- Use `--kv-cache-dtype fp8` to halve KV cache memory

**Optimal tensor parallel size:**
- 7B: 1 GPU (no TP needed)
- 13B: 1-2 GPUs
- 70B: 4-8 GPUs (H100 80GB)
- Use TP only when model doesn't fit on single GPU (communication overhead is 10-20%)

## 18. Scaling to Millions

**Horizontal scaling strategy:**
1. **Model sharding:** Use tensor parallelism for model fit, pipeline parallelism for larger deployments
2. **Replica pool:** Deploy N vLLM replicas behind a load balancer
3. **Auto-scaling:** HPA based on `vllm:num_requests_running` metric
4. **Request queue:** Redis-based queue for peak smoothing
5. **Model isolation:** Dedicated replicas for different model sizes

**For 1M+ daily requests:**
```
Requests/sec:  ~12
Tokens/day:    ~1B
GPUs needed:   ~32 H100-80GB (Llama 70B, 8K context)
vLLM replicas: 8 (4 GPUs each)
```

**Key bottleneck:** KV cache memory. At 1B tokens/day with 8K context, you need ~40TB of aggregate KV cache. With FP8, ~20TB. With prefix caching, potentially 5-10TB.

## 19. Failure Scenarios

| Failure | Impact | Mitigation |
|---------|--------|------------|
| **OOM** | Server crashes, all requests fail | Reserve headroom (gpu_memory_utilization < 0.95), enable swap space |
| **GPU hang** | Deadlocked worker, no progress | Watchdog timer, auto-restart worker, drain node |
| **Model load failure** | Server down at startup | Health check, canary deploy, rollback |
| **KV cache corruption** | Garbled output | Block checksums, CRC per block, retry on mismatch |
| **Scheduler livelock** | No progress, CPU spinning | Timeout per sequence, max iterations guard |
| **Network partition** | TP workers disconnected | NCCL timeout, NCCL_RETRY_CNT, ring reinit |

**Graceful degradation plan:**
1. When memory pressure reaches 90%, reject new requests with 507 status
2. When GPU errors > 5% in 1 minute, mark node unhealthy
3. When KV cache corruption detected, soft-restart the worker
4. Fallback to CPU offload if swap space available and GPU memory exhausted

## 20. Security

| Concern | Mitigation |
|---------|------------|
| **Prompt injection** | Input sanitization, content filters (NeMo Guardrails) |
| **Model stealing** | Don't expose raw logits/logprobs in production |
| **DoS via long context** | Enforce max_model_len, rate limit by token count |
| **Model poisoning** | Verify model hash (SHA-256) before loading |
| **Network MITM** | TLS between client and server, mTLS between workers |
| **Sensitive data in logs** | PII redaction middleware, audit log access |
| **Container escape** | Run as non-root user, seccomp profile, read-only root FS |

**Production security checklist:**
- [ ] Disable `/v1/completions` (use chat endpoint only)
- [ ] Set `max_logprobs: 0` in production
- [ ] Enforce auth via API keys (JWT validation)
- [ ] Rate limit by user, IP, and API key
- [ ] Enable audit logging for all requests
- [ ] Run health checks that don't reveal model architecture
- [ ] Use Egress network policies to block outbound from inference pods

## 21. Monitoring

**Key metrics (Prometheus):**

```
vllm:num_requests_running    — Current in-flight requests
vllm:num_requests_waiting    — Queue depth
vllm:num_requests_swapped    — Swapped out sequences
vllm:gpu_cache_usage         — GPU KV cache utilization (0-1)
vllm:gpu_cache_usage_perc    — Percentage used
vllm:time_to_first_token_ms  — TTFT histogram
vllm:time_per_output_token_ms— TPOT histogram
vllm:throughput_tokens_per_s — Overall tokens/sec
vllm:e2e_request_latency_ms  — End-to-end latency
vllm:preemption_rate         — Fraction of requests preempted
```

**Grafana dashboard panels:**
1. Request rate + queue depth (time series)
2. TTFT / TPOT histograms (p50, p95, p99)
3. GPU memory breakdown (weights, KV cache, activations)
4. Cache hit ratio (prefix caching)
5. Error rate by type (OOM, timeout, invalid request)
6. Cost per 1M tokens (GPU-hours / tokens served)

**Alerts:**
- P95 TTFT > 2s → investigate scheduling or memory pressure
- GPU cache usage > 95% → scale up or reduce max_num_seqs
- Preemption rate > 5% → reduce load or increase batch size
- Error rate > 1% → investigate model or hardware

## 22. Interview Questions

**Q1: How does PagedAttention improve memory utilization compared to traditional attention?**
*A: Traditional attention allocates contiguous KV cache per sequence based on max_length, wasting 60-80% of memory. PagedAttention divides the KV cache into fixed-size blocks and uses a block table (like virtual memory pages), allowing non-contiguous physical storage and eliminating internal fragmentation.*

**Q2: What happens when GPU memory is exhausted during inference?**
*A: vLLM's scheduler preempts lowest-priority sequences. With recompute mode, the preempted sequence's KV blocks are freed and will be recomputed from the beginning when rescheduled. With swap mode, blocks are copied to CPU and loaded back. Recompute is generally preferred (no CPU-GPU bandwidth bottleneck).*

**Q3: How would you tune vLLM for a real-time chatbot vs a batch summarization workload?**
*A: For chatbots, minimize TTFT by using smaller batches (max_num_seqs=64), enable preemption mode=recompute, and keep max_num_batched_tokens low. For batch summarization, maximize throughput by increasing max_num_seqs (256+), enabling larger continuous batches, and using preemption mode=swap for prefill-heavy workloads.*

**Q4: Explain the tradeoffs between tensor parallelism and pipeline parallelism in vLLM.**
*A: Tensor parallelism splits each layer across GPUs (all-to-all communication per layer), which is latency-optimal but requires high-bandwidth interconnects (NVLink >200GB/s). Pipeline parallelism splits layers across GPUs, reducing communication but introducing idle bubbles. vLLM primarily uses tensor parallelism; pipeline parallelism is available but less common.*

## 23. Cheat Sheet

```bash
# Start server
python -m vllm.entrypoints.openai.api_server \
    --model meta-llama/Meta-Llama-3-8B-Instruct \
    --tensor-parallel-size 1 \
    --max-num-seqs 256 \
    --gpu-memory-utilization 0.9 \
    --dtype bfloat16

# Query with curl
curl http://localhost:8000/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{
    "model": "default",
    "messages": [{"role": "user", "content": "Hello!"}],
    "max_tokens": 100
  }'

# Check health
curl http://localhost:8000/health

# Metrics
curl http://localhost:8000/metrics

# Benchmark
python -m vllm.benchmarks.benchmark_throughput \
    --model meta-llama/Meta-Llama-3-8B-Instruct \
    --num-prompts 1000 \
    --input-len 512 \
    --output-len 256
```

## 24. Common Mistakes

1. **Setting `gpu_memory_utilization=0.99`** — leaving no headroom for activations causes OOM during heavy load. Keep at 0.85-0.92.

2. **Using default dtype (float16) with HuggingFace models** — many models expect bfloat16; float16 causes NaN loss. Always specify `--dtype bfloat16`.

3. **Not pre-compiling CUDA kernels** — set `VLLM_USE_PRECOMPILED=1` or run once to warm up. First request takes 10-30s otherwise.

4. **Oversizing tensor_parallel_size** — TP adds communication overhead. For Llama-3-8B on H100, TP=1 is optimal. Only use TP > 1 for models that don't fit on one GPU.

5. **Ignoring `max_model_len`** — default is 4096; setting to 8192 doubles KV cache memory. Only set what you need.

6. **Not using prefix caching for system prompts** — shared system prompts waste KV compute. Enable `--enable-prefix-caching` for 2-5x speedup on system-prompt-heavy workloads.

7. **Running without swap space** — swap prevents OOM during spikes but costs throughput. Set `--swap-space 16`.

## 25. Production Best Practices

1. **Pre-warm the model** — send a health check that includes a warmup request to trigger kernel compilation and KV cache allocation before serving traffic.

2. **Use `max_num_batched_tokens`** — controls the maximum tokens processed per iteration. Higher values improve throughput but increase TTFT. For chatbots, 4096; for batch, 8192+.

3. **Monitor block utilization** — if `gpu_cache_usage_perc < 50%` at steady state, reduce GPU count or increase load.

4. **Enable `--enforce-eager` for debugging** — turns off CUDA graph optimization (which caches operations). Disable in production for 20-40% speedup.

5. **Use Ray for multi-node** — `--tensor-parallel-size 8 --pipeline-parallel-size 2` across 2 nodes. Ensure NCCL environment variables are set.

6. **Pin GPUs with CUDA_VISIBLE_DEVICES** — in Kubernetes, set `spec.containers[].env[].name: CUDA_VISIBLE_DEVICES` to the GPU allocation.

7. **Set `--disable-log-stats` in high-throughput scenarios** — logging stats adds overhead. Use structured metrics instead.

8. **Implement circuit breaker** — if error rate exceeds 10% in 30s, stop sending new requests to that replica and restart it.

9. **Use host memory swap as safety net** — allocate 32-64GB swap space per GPU. vLLM swaps less-used sequences to CPU during memory pressure.

10. **Pin critical Kubernetes resources** — set resource requests = limits for GPU, memory, and CPU to avoid noisy neighbor issues.
