# Tensor Parallelism

## 1. What is it?

Tensor parallelism splits individual model layers (specifically the weight matrices) across multiple GPUs, so each GPU holds a shard of every layer. During inference, GPUs compute their shards in parallel and synchronize via all-reduce operations.

**ELI5:** Imagine a giant spreadsheet calculation. Instead of one person doing the whole thing, you split the spreadsheet into vertical slices. Each person works on their slice simultaneously. At the end, everyone shares their results to reconstruct the full answer. Tensor parallelism is that "vertical slicing" for neural network weights.

**Technical Definition:** Tensor parallelism (also called intra-layer model parallelism) partitions the weight matrices of each transformer layer along either the row or column dimension across GPUs. Forward and backward passes use collective communication (all-reduce, reduce-scatter, all-gather) to synchronize. The most common implementations are from Megatron-LM (NVIDIA) and the TPU-specific GSPMD (Google).

## 2. Why do we need it?

**Problem:** Modern LLMs are too large for a single GPU. Llama-3-70B in FP16 requires 140GB of memory — no single GPU has that much (H100 peaks at 80GB, A100 at 80GB). Even if it fit, a single GPU would be compute-bound for inference.

**Pain without it:** Without tensor parallelism, you cannot run models larger than your largest GPU's memory. You'd be limited to 30B-40B models on an 80GB H100 (with quantization). For 70B+ models, you simply couldn't run inference without either tensor parallelism or pipeline parallelism.

**Why companies use it:** Tensor parallelism minimizes latency by distributing computation across GPUs that are tightly coupled (NVLink/NVSwitch). For latency-sensitive applications like chat, it's the preferred parallelism strategy because all GPUs compute simultaneously with minimal idle time.

## 3. Real-world Example

- **NVIDIA Megatron-LM**: Originated tensor parallelism for transformer models. Used to train Megatron-Turing NLG 530B and Nemotron-4.
- **OpenAI GPT-4**: Uses tensor parallelism across multiple GPU nodes. The scale is believed to be massive model parallelism across thousands of GPUs.
- **Meta Llama 3**: 405B model trained with tensor parallelism on 16,384 H100 GPUs using 4D parallelism (TP + PP + DP + FSDP).
- **Google PaLM/Gemini**: Uses GSPMD (a tensor parallelism compiler) that automatically shards computations across TPU pods.
- **Microsoft DeepSpeed**: Implements tensor parallelism via DeepSpeed-Inference for serving.
- **Anyscale/vLLM**: Built-in tensor parallelism support via `--tensor-parallel-size`.

## 4. Architecture Diagram (ASCII)

```
Without Tensor Parallelism (Single GPU):
┌───────────────────────────────┐
│          Entire Model         │
│ ┌───┐ ┌───┐ ┌───┐           │
│ │W1 │→│W2 │→│W3 │→ ... → out│
│ └───┘ └───┘ └───┘           │
│  GPU 0 (OOM for large models) │
└───────────────────────────────┘

With Tensor Parallelism (2 GPUs, Column-Sharded):

GPU 0                               GPU 1
┌────────────────────┐              ┌────────────────────┐
│ W1_shard0           │              │ W1_shard1           │
│ ┌──┐ ┌──┐ ┌──┐    │              │ ┌──┐ ┌──┐ ┌──┐    │
│ │W1a│→│W2a│→│W3a│→ Z0            │ │W1b│→│W2b│→│W3b│→ Z1
│ └──┘ └──┘ └──┘    │              │ └──┘ └──┘ └──┘    │
└────────┬───────────┘              └────────┬───────────┘
         │                                   │
         └──────────────┬────────────────────┘
                        │ all-reduce
                        ▼
                 ┌────────────┐
                 │  Z = Z0+Z1 │  (full output)
                 └────────────┘
```

## 5. Internal Working

**Megatron-LM style tensor parallelism for transformer layers:**

**1. Self-Attention (column-parallel):**
- QKV weight matrix `W_qkv` (shape: `[d_model, 3*d_head*n_heads]`) is split column-wise
- Each GPU computes its shard of Q, K, V
- After attention, output projection matrix `W_o` (shape: `[n_heads*d_head, d_model]`) is split row-wise
- All-reduce sums contributions from all GPUs

**2. MLP layers (row-parallel + column-parallel):**
- First MLP weight `W1` (shape: `[d_model, 4*d_model]`) — column split
- Activation (GeLU/SiLU) applied independently per GPU
- Second MLP weight `W2` (shape: `[4*d_model, d_model]`) — row split
- All-reduce after W2 to get full output

**Communication pattern:**
```
For each transformer layer:
  1. Input X is identical on all GPUs (no communication needed initially)
  2. Column-parallel: each GPU has full X, partial W → partial output → no communication
  3. Row-parallel: partial input * partial W → need all-reduce to sum partial outputs
  4. After row-parallel: all-reduce(X_i) → full output on all GPUs → next layer input ready
```

## 6. Production Flow

```
Input tokens (broadcast to all TP ranks)
    │
    ▼
┌─────────────────────────────────────────────┐
│ Embedding (lookup identical, replicated)     │
└──────────────────┬──────────────────────────┘
                   │
     ┌─────────────┴─────────────┐
     │                           │
┌────▼────┐               ┌─────▼─────┐
│ GPU 0   │               │ GPU 1     │
│ Layer 0 │               │ Layer 0   │
│ shard   │               │ shard     │
│         │               │           │
│ ┌─────┐ │               │ ┌─────┐   │
│ │Attn │ │               │ │Attn │   │
│ └──┬──┘ │               │ └──┬──┘   │
│    │    │               │    │      │
│ ┌──▼──┐ │←─────all-reduce─────→│  │
│ │MLP  │ │               │ ┌──▼──┐   │
│ └──┬──┘ │               │ │MLP  │   │
│    │    │               │ └──┬──┘   │
│ ┌──▼──┐ │←─────all-reduce─────→│  │
│ │Norm │ │               │ ┌──▼──┐   │
│ └─────┘ │               │ │Norm │   │
└────┬────┘               │ └─────┘   │
     │                    └─────┬─────┘
     │                          │
     └──────────┬───────────────┘
                │
         ┌──────▼──────┐
         │ Lm_head +   │
         │ gather      │
         └──────┬──────┘
                │
         ┌──────▼──────┐
         │ All-gather  │
         │ logits      │
         └──────┬──────┘
                │
          Output token (full on rank 0)
```

## 7. HLD (High-Level Design)

```
┌────────────────────────────────────────────┐
│              TP Group (4 GPUs)              │
│                                              │
│  ┌─────────┐  ┌─────────┐  ┌─────────┐  ┌─┐│
│  │  GPU 0  │  │  GPU 1  │  │  GPU 2  │  │3││
│  │ rank 0  │  │ rank 1  │  │ rank 2  │  │ ││
│  └────┬────┘  └────┬────┘  └────┬────┘  └─┘│
│       │            │            │           │
│       └────────────┼────────────┘           │
│                    │                        │
│           ┌────────┴────────┐               │
│           │  NVSwitch/NVLink │               │
│           │  (900 GB/s)     │               │
│           └─────────────────┘               │
└────────────────────────────────────────────┘
        │
┌───────┴───────────────────────────────────────────┐
│ Each GPU holds:                                    │
│ • 1/4 of every weight matrix                       │
│ • 1/4 of KV cache blocks                          │
│ • Full activation for 1/4 of hidden dimension      │
└───────────────────────────────────────────────────┘
```

## 8. LLD (Low-Level Design)

**Column-parallel linear layer:**
```python
class ColumnParallelLinear(nn.Module):
    """Split weight along dimension 1 (columns)."""
    def __init__(self, in_features, out_features, tp_size, rank):
        super().__init__()
        self.tp_size = tp_size
        self.rank = rank
        shard_size = out_features // tp_size

        # Full weight (only stored for reference; each GPU stores its shard)
        full_weight = torch.randn(in_features, out_features)

        # Each GPU stores only its column shard
        self.weight = nn.Parameter(
            full_weight[:, rank * shard_size : (rank + 1) * shard_size]
        )

    def forward(self, x):
        # x has full in_features, output has shard_size features
        return F.linear(x, self.weight)
```

**Row-parallel linear layer:**
```python
class RowParallelLinear(nn.Module):
    """Split weight along dimension 0 (rows), all-reduce after computation."""
    def __init__(self, in_features, out_features, tp_size, rank):
        super().__init__()
        self.tp_size = tp_size
        self.rank = rank
        shard_size = in_features // tp_size

        full_weight = torch.randn(in_features, out_features)
        self.weight = nn.Parameter(
            full_weight[rank * shard_size : (rank + 1) * shard_size, :]
        )

    def forward(self, x):
        # x has shard_size features (partial), weight has shard_size×out_features
        partial_out = F.linear(x, self.weight)
        # All-reduce to sum all partial outputs
        full_out = partial_out
        dist.all_reduce(full_out, op=dist.ReduceOp.SUM)
        return full_out
```

**All-reduce overhead:**
```
Communication volume per all-reduce: 2 * shard_size * dtype_bytes
For Llama-3-8B with d_model=4096, tp_size=4:
  Volume per all-reduce: 2 * (4096/4) * 2 = 4096 bytes
  Total per layer: 2 all-reduces (attn_out + mlp_out) = 8192 bytes
  With NVLink 900 GB/s: ~9ns communication time
```

## 9. Python Implementation

```python
import torch
import torch.distributed as dist
import torch.nn as nn
from torch.distributed._tensor import DeviceMesh, Shard
from torch.distributed.tensor.parallel import (
    PairwiseParallel,
    parallelize_module,
    ColwiseParallel,
    RowwiseParallel,
)
from transformers import AutoModelForCausalLM, AutoTokenizer
import logging

logging.basicConfig(level=logging.INFO)
logger = logging.getLogger("tensor-parallel")

def setup_tp_group(tp_size: int = 2):
    """Initialize the tensor parallel process group."""
    dist.init_process_group(backend="nccl")
    rank = dist.get_rank()
    torch.cuda.set_device(rank)

    # Create a 1D device mesh for tensor parallelism
    tp_mesh = DeviceMesh("cuda", list(range(tp_size)))
    logger.info(f"TP group initialized. Rank {rank}/{tp_size}")
    return rank, tp_mesh

def shard_model(model_name: str, tp_mesh: DeviceMesh) -> nn.Module:
    """Load and shard a model across tensor parallel ranks."""
    rank = dist.get_rank()

    # Load model on meta device first (no memory allocated yet)
    with torch.device("meta"):
        model = AutoModelForCausalLM.from_pretrained(model_name)

    # Define parallelism plan
    # For GPT/Llama-style decoder layers:
    # - QKV projection: Column-wise parallel (split output dim)
    # - Attention output: Row-wise parallel (split input dim, all-reduce)
    # - MLP gate/up: Column-wise parallel
    # - MLP down: Row-wise parallel
    plan = {
        "model.layers.0.self_attn.q_proj": ColwiseParallel(),
        "model.layers.0.self_attn.k_proj": ColwiseParallel(),
        "model.layers.0.self_attn.v_proj": ColwiseParallel(),
        "model.layers.0.self_attn.o_proj": RowwiseParallel(),
        "model.layers.0.mlp.gate_proj": ColwiseParallel(),
        "model.layers.0.mlp.up_proj": ColwiseParallel(),
        "model.layers.0.mlp.down_proj": RowwiseParallel(),
    }

    # Apply parallelism to first layer; repeat for all layers
    parallelized_model = parallelize_module(model, tp_mesh, plan)
    logger.info(f"Model sharded across {tp_mesh.size()} GPUs")
    return parallelized_model

def generate_tp(
    model: nn.Module,
    input_ids: torch.Tensor,
    max_new_tokens: int = 100,
) -> torch.Tensor:
    """Generate with tensor parallelism. Only rank 0 returns full output."""
    rank = dist.get_rank()

    for _ in range(max_new_tokens):
        with torch.inference_mode():
            outputs = model(input_ids)
            logits = outputs.logits[:, -1, :]

        # Sample next token (all ranks sample independently — same input, same output)
        next_token = torch.argmax(logits, dim=-1, keepdim=True)

        # Broadcast token from rank 0 if needed (safety — should be identical)
        if dist.get_world_size() > 1:
            dist.broadcast(next_token, src=0)

        input_ids = torch.cat([input_ids, next_token], dim=-1)

    return input_ids if rank == 0 else None

def main():
    rank, tp_mesh = setup_tp_group(tp_size=2)
    model = shard_model("meta-llama/Llama-3.2-3B-Instruct", tp_mesh)
    tokenizer = AutoTokenizer.from_pretrained("meta-llama/Llama-3.2-3B-Instruct")

    if rank == 0:
        inputs = tokenizer("Hello, how are you?", return_tensors="pt")
    else:
        inputs = None

    # Broadcast inputs from rank 0 to all ranks
    if dist.get_world_size() > 1:
        inputs["input_ids"] = inputs["input_ids"].cuda() if rank == 0 else None
        inputs["input_ids"] = dist.broadcast(inputs["input_ids"], src=0)

    outputs = generate_tp(model, inputs["input_ids"].cuda())
    if rank == 0:
        print(tokenizer.decode(outputs[0]))

if __name__ == "__main__":
    main()
```

## 10. Folder Structure

```
tensor_parallel_project/
├── launch.sh               # torchrun / mpirun launcher
├── config.yaml
├── tp_inference/
│   ├── __init__.py
│   ├── main.py             # Entry point with torchrun
│   ├── model_sharding.py   # TP model loading
│   ├── parallel_layers.py  # Custom col/row parallel layers
│   ├── communication.py    # All-reduce, all-gather helpers
│   ├── server.py           # FastAPI server (rank 0 only)
│   └── scheduler.py        # Token scheduling across TP
├── k8s/
│   ├── statefulset.yaml    # StatefulSet with rank ordering
│   ├── service.yaml
│   └── tp-group-config.yaml
└── tests/
    └── test_parallel.py
```

## 11. Configuration

```yaml
tensor_parallel:
  size: 4                     # Number of GPUs in TP group
  backend: nccl               # Communication backend
  master_addr: localhost
  master_port: 29500

model:
  name: meta-llama/Llama-3.2-3B-Instruct
  dtype: bfloat16
  use_flash_attention: true

communication:
  all_reduce_algo: ring        # ring, tree, nvlink
  gradient_compression: false  # Not useful for inference
  overlap_communication: true  # Overlap all-reduce with compute

hardware:
  interconnect: nvlink         # nvlink, pcie, infiniband
  nvlink_bandwidth_gbps: 900   # H100: 900 GB/s, A100: 600 GB/s
  pcie_bandwidth_gbps: 64      # PCIe Gen5 x16
```

## 12. Flowchart

```
Input Token IDs (full, all ranks)
        │
        ▼
 ┌──────────────────┐
 │ Embedding Layer  │  (replicated, all ranks have full)
 └──────┬───────────┘
        │
        ▼
 ┌──────────────────┐
 │ TP Decoder Block  │
 │                   │
 │   ┌───────────┐   │
 │   │ RMS Norm   │   │  (all ranks have full — no communication)
 │   └─────┬─────┘   │
 │         │         │
 │   ┌─────▼─────┐   │
 │   │ QKV Proj   │   │  (column parallel — each rank has partial)
 │   └─────┬─────┘   │
 │         │         │
 │   ┌─────▼─────┐   │
 │   │ Self-Attn  │   │  (local — partial heads on each rank)
 │   └─────┬─────┘   │
 │         │         │
 │   ┌─────▼─────┐   │
 │   │ Out Proj    │   │  (row parallel — need all-reduce)
 │   └─────┬─────┘   │
 │         │         │
 │   ┌─────▼─────┐   │
 │   │   ALL-    │   │  ←──── all-reduce across TP ranks
 │   │   REDUCE  │   │
 │   └─────┬─────┘   │
 │         │         │
 │   ┌─────▼─────┐   │
 │   │ MLP Gate   │   │  (column parallel)
 │   │ + Up Proj  │   │
 │   └─────┬─────┘   │
 │         │         │
 │   ┌─────▼─────┐   │
 │   │ Activation │   │  (SiLU/GELU — element-wise, local)
 │   └─────┬─────┘   │
 │         │         │
 │   ┌─────▼─────┐   │
 │   │ Down Proj  │   │  (row parallel — need all-reduce)
 │   └─────┬─────┘   │
 │         │         │
 │   ┌─────▼─────┐   │
 │   │   ALL-    │   │  ←──── all-reduce across TP ranks
 │   │   REDUCE  │   │
 │   └───────────┘   │
 └──────────────────┘
        │  (repeat for N layers)
        ▼
 ┌──────────────────┐
 │ Final RMS Norm   │
 └──────┬───────────┘
        │
        ▼
 ┌──────────────────┐
 │ LM Head (column  │
 │ parallel)        │
 └──────┬───────────┘
        │
        ▼
 ┌──────────────────┐
 │ All-Gather       │  ←──── gather full logits on rank 0
 │ Logits           │
 └──────┬───────────┘
        │
        ▼
 Output Token (rank 0)
```

## 13. Sequence Diagram

```
Rank 0               Rank 1               Rank 2               Rank 3
  │                    │                    │                    │
  │───input broadcast────────────────────────────────────────────▶│
  │◀─────────────────────────────────────────────────────────────│
  │                    │                    │                    │
  │───Layer i────────────────────────────────────────────────────▶│
  │ QKV:col-parallel   │ QKV:col-parallel   │ QKV:col-parallel   │
  │ Attn:local         │ Attn:local         │ Attn:local         │
  │                    │                    │                    │
  │───all-reduce─────────────────────────────────────────────────▶│
  │◀─────────────────────────────────────────────────────────────│
  │                    │                    │                    │
  │───MLP:col-parallel───────────────────────────────────────────▶│
  │───all-reduce─────────────────────────────────────────────────▶│
  │◀─────────────────────────────────────────────────────────────│
  │                    │                    │                    │
  │───next layer─────────────────────────────────────────────────▶│
  │                    │                    │                    │
  │───LM Head─────────▶│all-gather◀────────│◀────────────────────│
  │◀─────────────────────────────────────────────────────────────│
  │                    │                    │                    │
  │───token───────────▶│(only rank 0 outputs)                    │
```

## 14. Pros

- **Low latency** — all GPUs compute simultaneously; no idle pipeline bubbles
- **Memory proportional** — each GPU holds `1/tp_size` of weights and KV cache
- **Scale within node** — NVLink provides 600-900 GB/s inter-GPU bandwidth
- **Minimal code changes** — framework handles sharding with decorators
- **Good for inference** — no pipeline bubbles; consistent per-token latency
- **Compatible with other parallelism** — works alongside PP, DP, FSDP

## 15. Cons

- **Communication overhead** — all-reduce per layer adds latency proportional to hidden dimension
- **Requires fast interconnect** — NVLink or InfiniBand required; PCIe is too slow
- **Scale limited to node** — typical max TP size is 8 (one DGX node)
- **All ranks must stay synchronized** — straggler GPU holds up the entire group
- **Memory overhead for replicated activations** — some activations are replicated across ranks
- **Higher complexity** — debugging distributed communication issues is hard

## 16. Alternatives

| Method | Best For | Tradeoffs |
|--------|----------|-----------|
| **Pipeline Parallelism** | Training, batch inference | Pipeline bubbles, higher latency |
| **DeepSpeed ZeRO-3** | Training large models | Not optimized for inference |
| **FSDP** | Training, any scale | Higher communication than TP |
| **Sequence Parallelism** | Long-context inference | More complex, specific to attention |
| **Expert Parallelism** | MoE models (Mixtral) | Load balancing challenges |

## 17. Performance Considerations

**Communication-to-computation ratio:**
```
Metric: communication_time / computation_time
- TP size 2:  ~5% overhead (communication is small relative to compute)
- TP size 4:  ~10% overhead
- TP size 8:  ~20% overhead
- TP size 16: ~40% overhead (only viable with NVSwitch across nodes)

Rule of thumb: TP size should not exceed hidden_dim // 1024
Example: d_model=4096 → max TP=4
```

**NVLink vs PCIe comparison:**
```
NVLink (4x H100):   900 GB/s bidirectional — all-reduce of 4KB in ~9ns
PCIe Gen5 x16:      64 GB/s bidirectional — all-reduce of 4KB in ~125ns
Network (IB NDR):   50 GB/s — all-reduce of 4KB in ~160ns
```

**Optimal TP sizes for common models:**
```
Llama-3-8B  (d_model=4096):  TP=1 (best on single GPU; TP adds overhead)
Llama-3-70B (d_model=8192):  TP=4 (optimal, 4x H100 80GB)
Llama-3-405B:                TP=8 (requires DGX H100 node with NVSwitch)
GPT-4 (estimated ~1.8T):     TP=64 (across 8 nodes with NVSwitch)
```

## 18. Scaling to Millions

**Production TP deployment:**

1. **Choose optimal TP size** — benchmark latency vs batch size for 2, 4, 8 GPUs. Often TP=4 is the sweet spot.

2. **Combine with data parallelism for throughput** — replicate TP groups, each group handles a separate request. N TP groups = Nx throughput.

3. **NCCL tuning**:
   ```bash
   export NCCL_ALGO=Ring       # Ring all-reduce for large messages
   export NCCL_PROTO=Simple    # Simple protocol for latency
   export NCCL_MIN_NCHANNELS=4 # More channels = better bandwidth utilization
   ```

4. **Topology-aware mapping** — map TP ranks to GPUs within same NVSwitch domain to minimize latency:
   ```bash
   # 8 GPUs in DGX H100: ranks 0-7 are in same domain
   CUDA_VISIBLE_DEVICES=0,1,2,3,4,5,6,7 torchrun ...
   ```

5. **Model warmup** — run a few synthetic tokens through all TP ranks to trigger kernel compilation and NCCL connection setup before serving traffic.

## 19. Failure Scenarios

| Failure | Impact | Mitigation |
|---------|--------|------------|
| **GPU hang** | All ranks hang waiting for all-reduce | Timeout on collective ops (NCCL_TIMEOUT), watchdog |
| **NCCL timeout** | Exception, all ranks crash | Increase NCCL_TIMEOUT, check PCIe/NVLink health |
| **Rank failure** | One GPU fails → full TP group down | Replicate TP groups; route around failed group |
| **Link degradation** | Slow communication, increased latency | Monitor NVLink errors; replace hardware |
| **Memory imbalance** | One rank OOM, others fine | Verify sharding is balanced; check for uneven KV cache |
| **CUDA graph mismatch** | TP incompatible with CUDA graphs | Disable CUDA graphs for TP; use eager mode |

## 20. Security

| Concern | Mitigation |
|---------|------------|
| **Cross-rank data exposure** | All TP ranks see the same data (inputs/outputs shared via all-reduce). Encrypt at-rest; trust intra-node network |
| **NCCL network sniffing** | All-reduce traffic is in plaintext. Use IB with encryption or trusted networks |
| **Model theft** | Each rank has only a weight shard. Collecting all shards reconstructs full model |
| **Node isolation** | Run TP groups within trusted nodes; network isolation for NCCL |

## 21. Monitoring

**Key metrics:**
```
tp:all_reduce_latency_us     — Time for all-reduce per layer
tp:computation_time_ms       — Time for forward pass (excluding communication)
tp:communication_overhead_%  — comm_time / (comm_time + compute_time)
tp:throughput_tokens_per_s   — Tokens generated per second (entire TP group)
tp:gpu_memory_per_rank_gb    — Memory used per GPU
tp:nvlink_bandwidth_util_%   — NVLink utilization percentage
tp:nccl_errors_count         — NCCL communication errors
```

**Grafana panels:**
1. Communication vs computation time per TP rank
2. All-reduce latency histogram
3. Memory balance across ranks
4. Token throughput per TP group
5. NVLink bandwidth utilization

**Alerts:**
- Communication overhead > 25% → reduce TP size or check NVLink health
- Memory imbalance > 10% between ranks → fix sharding
- NCCL errors > 0 → check GPU and NVLink health
- Any TP rank not participating → failover TP group

## 22. Interview Questions

**Q1: Explain the difference between column-wise and row-wise tensor parallelism.**
*A: Column-wise splits the weight matrix along the output dimension (columns). Each GPU gets a vertical slice. Input must be full on all GPUs. Output is partial. Row-wise splits along the input dimension (rows). Each GPU gets a horizontal slice. Input must be partial. Output requires all-reduce to combine. In Megatron-LM, the first MLP layer is column-parallel (no communication), the second is row-parallel (all-reduce).*

**Q2: Why can't we use tensor parallelism across nodes with slow interconnects?**
*A: Tensor parallelism requires all-reduce after every transformer layer (typically 2 all-reduces per layer). With ~80 layers in Llama-3-70B, that's 160 all-reduces per token. Each all-reduce of the hidden dimension (8192 * 2 bytes = 16KB) takes ~160ns on NVLink but ~25μs on network (100x slower). At 160 all-reduces per token, that's 25μs × 160 = 4ms extra per token on network vs 1.6μs on NVLink — a 2500x difference that kills latency.*

**Q3: How does tensor parallelism interact with Flash Attention?**
*A: Tensor parallelism splits attention heads across GPUs; each GPU computes Flash Attention on its shard of heads. This works naturally because Flash Attention processes each head independently. The output projection after attention (which combines heads) uses row parallelism with all-reduce. No Flash Attention modification is needed for TP.*

**Q4: How would you decide between tensor parallelism and pipeline parallelism?**
*A: TP is best for latency-sensitive inference (each token is equally fast). PP is better for throughput-bound training or batch inference where pipeline bubbles are amortized. For chat applications: prefer TP. For batch document processing: consider PP. For serving large models (70B+): use both (TP within node, PP across nodes).*

## 23. Cheat Sheet

```bash
# 2-GPU tensor parallel inference with vLLM
python -m vllm.entrypoints.openai.api_server \
    --model meta-llama/Llama-3.2-3B-Instruct \
    --tensor-parallel-size 2

# 4-GPU with torchrun
torchrun --nproc_per_node=4 \
    tp_inference/main.py \
    --model meta-llama/Llama-3.2-3B-Instruct

# Environment variables for NCCL
export NCCL_DEBUG=INFO                  # Debug logging
export NCCL_ALGO=Ring                   # Ring allreduce
export NCCL_PROTO=Simple                # Simple protocol
export NCCL_NET=IB                      # Use InfiniBand
export NCCL_IB_HCA=mlx5_0:1,mlx5_1:1   # IB devices

# Check NVLink status
nvidia-smi nvlink --capabilities
nvidia-smi nvlink --status

# Check NCCL topology
python -c "import torch; print(torch.cuda.nccl.nccl_version())"
```

## 24. Common Mistakes

1. **Using TP when model fits on one GPU** — TP adds communication overhead with no memory benefit. A 7B model fits on a single H100 — don't use TP.

2. **Not setting `CUDA_VISIBLE_DEVICES`** — without explicit GPU mapping, NCCL may use sub-optimal topology, increasing latency 5x.

3. **PCIe-based TP** — using TP across GPUs connected via PCIe (instead of NVLink) increases all-reduce time by 10-50x.

4. **Oversized TP group** — using 8 GPUs for a model that fits on 4 GPUs. Communication overhead exceeds any benefit beyond TP=hidden_dim/1024.

5. **Not using `torchrun` or `mpirun`** — manually spawning processes without proper NCCL init causes hangs.

6. **Forgetting to set `torch.inference_mode()`** — autograd tracks all-reduce operations, consuming memory that grows unbounded during generation.

7. **Mixing data types across ranks** — if one rank has float16 and another has bfloat16, all-reduce produces incorrect results.

## 25. Production Best Practices

1. **Use `torchrun` or `mpirun`** — never launch TP processes manually. `torchrun --nproc_per_node=$TP_SIZE` handles NCCL initialization and fault recovery.

2. **Benchmark TP latency vs throughput** — create a heatmap of TP size × batch size. For many workloads, TP=4 with medium batch is the Pareto-optimal point.

3. **Pin GPUs with `CUDA_VISIBLE_DEVICES`** — ensure TP ranks map to GPUs within the same NVLink domain (all 8 GPUs on a DGX node, or 4+4 on dual-socket systems).

4. **Enable NCCL `IB` and `NVLink`** — set `NCCL_NET=IB` if InfiniBand is available; set `NCCL_ALGO=Ring` for large message sizes.

5. **Pre-allocate NCCL communicators** — send a dummy all-reduce during model loading to establish connections before serving requests.

6. **Monitor per-rank memory** — if one rank uses significantly more memory, the sharding is unbalanced. Adjust TP plan or use `max_memory` constraints.

7. **Use `torch.cuda.set_device(rank)`** — ensures each process uses the correct GPU. Without this, all processes default to GPU 0.

8. **Set `OMP_NUM_THREADS=1` for TP workers** — prevents PyTorch from oversubscribing CPU threads (each process gets 1 CPU thread for best GPU utilization).

9. **Implement NCCL timeout** — in Kubernetes, set `NCCL_TIMEOUT=180` to prevent indefinite hangs if a GPU fails during collective communication.

10. **Use checkpoint-based recovery** — for long-running deployments, periodically verify TP group health. On failure, restart all ranks (partial recovery not possible in TP).
