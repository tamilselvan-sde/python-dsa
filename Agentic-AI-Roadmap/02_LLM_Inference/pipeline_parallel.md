# Pipeline Parallelism

## 1. What is it?

Pipeline parallelism splits a model by layers across multiple GPUs. GPU 0 holds layers 0-9, GPU 1 holds layers 10-19, GPU 2 holds layers 20-29, and so on. During inference, a request passes through GPU 0 → GPU 1 → GPU 2 → GPU 3 sequentially.

**ELI5:** Imagine an assembly line with 4 workers. Worker 1 attaches wheels, Worker 2 paints the car, Worker 3 installs the engine, Worker 4 does final inspection. Each worker specializes in one part of the process, and the car moves down the line. Pipeline parallelism is that assembly line for LLM inference.

**Technical Definition:** Pipeline parallelism (PP) partitions the model into stages (contiguous groups of transformer layers), each assigned to a different GPU. During forward inference, micro-batches are pipelined through stages to overlap computation — while stage 1 processes micro-batch 2, stage 2 processes micro-batch 1. The most common scheduling is 1F1B (one forward, one backward) for training; for inference, simpler schedules suffice since there's no backward pass.

## 2. Why do we need it?

**Problem:** Tensor parallelism is limited by inter-GPU bandwidth. Beyond 8 GPUs in a single node, the communication overhead of all-reduce per layer becomes prohibitive. To scale to 64+ GPUs, you need a parallelism strategy that communicates less frequently.

**Pain without it:** Without pipeline parallelism, you can't serve a 405B model on 32 GPUs efficiently. TP across 32 GPUs would require 32 all-reduces per layer — the communication would dominate (90%+ overhead). PP communicates only between stage boundaries (once per micro-batch), making it feasible across nodes with moderate network bandwidth.

**Why companies use it:** PP enables serving models that don't fit in a single node's aggregate memory. By splitting layer groups across nodes, each node handles fewer layers, reducing per-node memory requirements. Combined with TP (TP within node, PP across nodes), it's the standard strategy for 100B+ parameter models.

## 3. Real-world Example

- **Google PaLM**: 540B model trained with pipeline parallelism (and TP) across 6144 TPUv4 chips.
- **Meta Llama 3 405B**: Uses pipeline parallelism across nodes combined with tensor parallelism within nodes (4D parallelism: TP × PP × DP × FSDP).
- **NVIDIA Megatron-LM**: Pioneered the 1F1B pipeline schedule for efficient training of GPT-3 175B.
- **OpenAI GPT-4**: Believed to use a combination of tensor and pipeline parallelism across multiple nodes.
- **DeepSpeed Inference (Microsoft)**: Implements automatic pipeline partitioning for inference with minimal latency overhead.
- **Anthropic Claude**: Inference infrastructure uses pipeline parallelism for large-scale serving.

## 4. Architecture Diagram (ASCII)

```
Pipeline Parallelism (4 Stages):

Micro-batches flowing through pipeline:

Time →
Stage 0 (GPU 0)  Stage 1 (GPU 1)  Stage 2 (GPU 2)  Stage 3 (GPU 3)
Layers 0-19       Layers 20-39     Layers 40-59      Layers 60-79
┌──────────┐      ┌──────────┐      ┌──────────┐      ┌──────────┐
│ MB1      │──────▶│ MB1      │──────▶│ MB1      │──────▶│ MB1      │
│ prefill  │      │ prefill  │      │ prefill  │      │ prefill  │
├──────────┤      ├──────────┤      ├──────────┤      ├──────────┤
│ MB2      │──────▶│ MB2      │──────▶│ MB2      │──────▶│ MB2 MB1  │
│ prefill  │      │ prefill  │      │ prefill  │      │ pre+d   │
├──────────┤      ├──────────┤      ├──────────┤      ├──────────┤
│ MB3      │──────▶│ MB3 MB2  │──────▶│ MB3 MB2  │──────▶│ MB3 MB2  │
│ prefill  │      │ pre+d    │      │ pre+d    │      │ pre+d    │
├──────────┤      ├──────────┤      ├──────────┤      ├──────────┤
│ MB4 MB1  │──────▶│ MB4 MB3  │──────▶│ MB4 MB3  │──────▶│ MB4 MB3  │
│ pre+d    │      │ pre+d    │      │ pre+d    │      │ pre+d    │
├──────────┤      ├──────────┤      ├──────────┤      ├──────────┤
│ MB5 MB2  │──────▶│ MB5 MB4  │──────▶│ MB5 MB4  │──────▶│ MB5 MB4  │
│ pre+d    │      │ pre+d    │      │ pre+d    │      │ pre+d    │
└──────────┘      └──────────┘      └──────────┘      └──────────┘
    │                  │                  │                  │
    │                  │                  │                  │
    └──────────────────┴──────────────────┴──────────────────┘
                        │
                        ▼
                 Final output
```

## 5. Internal Working

**Pipeline scheduling for inference (no backward pass):**

**Simple FCFS (First-Come-First-Served):**
1. Stage 0 processes micro-batch M1's prefill → sends activations to Stage 1
2. Stage 1 processes M1 prefill → sends to Stage 2
3. Continue until last stage produces output
4. Last stage sends token back to Stage 0 → Stage 0 processes decode

**Interleaved 1F1B (more efficient):**
1. Stage 0 processes M1 prefill, sends to Stage 1
2. Stage 0 starts M2 prefill (while Stage 1 works on M1)
3. Continue pipelining — each stage works on a different micro-batch
4. Result: GPUs spend less time idle (higher utilization)

**Communication pattern:**
```
Each stage sends to next stage:
  - Forward (prefill): activations [batch_size × seq_len × d_model]
  - Decode: single token embedding [batch_size × 1 × d_model]
  - Communication is point-to-point (P2P), not collective (all-reduce)
```

## 6. Production Flow

```
Input tokens → Stage 0 (GPU 0, layers 0-19)
    │
    │  (forward: send activations via P2P)
    ▼
Stage 1 (GPU 1, layers 20-39)
    │
    ▼
Stage 2 (GPU 2, layers 40-59)
    │
    ▼
Stage 3 (GPU 3, layers 60-79 + lm_head)
    │
    ▼
Output logits → Sample → Decode
    │
    ▼
Token sent back to Stage 0 (for next decode step)
    │
    ▼
[Repeat decode loop for max_tokens steps]
```

## 7. HLD (High-Level Design)

```
┌─────────────────────────────────────────────────────────────┐
│                   Pipeline (4 Stages, 4 GPUs)                │
│                                                               │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  ┌──┐ │
│  │  Compute     │  │  Compute     │  │  Compute     │  │...│ │
│  │  Node 1      │──│  Node 2      │──│  Node 3      │──│  │ │
│  │  GPU 0       │  │  GPU 1       │  │  GPU 2       │  │ 3│ │
│  │  (layers 0-9)│  │  (layers 10-19)│  (layers 20-29)│  │  │ │
│  └──────────────┘  └──────────────┘  └──────────────┘  └──┘ │
│         │               │               │               │    │
│         └───────────────┴───────────────┴───────────────┘    │
│                         │                                    │
│                P2P Communication                             │
│        (via RDMA / InfiniBand / TCP)                        │
└─────────────────────────────────────────────────────────────┘

Memory per stage = (total_layers / pp_size) * layer_memory + KV_cache / pp_size
```

## 8. LLD (Low-Level Design)

**Pipeline stage implementation:**
```python
class PipelineStage(nn.Module):
    """A contiguous group of transformer layers assigned to one GPU."""

    def __init__(
        self,
        stage_id: int,
        layers: list[nn.Module],
        prev_stage: Optional[int],
        next_stage: Optional[int],
    ):
        super().__init__()
        self.stage_id = stage_id
        self.layers = nn.ModuleList(layers)
        self.prev_stage = prev_stage
        self.next_stage = next_stage

    def forward(
        self,
        x: torch.Tensor,
        kv_cache: Optional[list] = None,
    ) -> torch.Tensor:
        for i, layer in enumerate(self.layers):
            cache = kv_cache[i] if kv_cache else None
            x = layer(x, use_cache=cache is not None)
        return x
```

**Pipeline scheduler:**
```python
class PipelineScheduler:
    """Schedules micro-batches through pipeline stages."""

    def __init__(self, num_stages: int, num_microbatches: int):
        self.num_stages = num_stages
        self.num_microbatches = num_microbatches

    def schedule(self) -> list[PipelineStep]:
        """Generate schedule steps for one pipeline cycle (inference)."""
        steps = []

        # Prefill: warm-up phase
        for mb in range(self.num_microbatches):
            for stage in range(min(mb + 1, self.num_stages)):
                steps.append(PipelineStep(
                    stage=stage,
                    microbatch=mb,
                    phase="prefill",
                ))

        # Steady state: all stages busy
        total_decodes = self.num_microbatches + self.num_stages - 1
        for step_id in range(total_decodes):
            for stage in range(self.num_stages):
                mb = step_id - stage
                if 0 <= mb < self.num_microbatches:
                    steps.append(PipelineStep(
                        stage=stage,
                        microbatch=mb,
                        phase="decode",
                    ))

        return steps

    @property
    def num_latency_units(self) -> int:
        """Number of time units for one full pipeline cycle."""
        return self.num_microbatches + self.num_stages - 1

    @property
    def bubble_ratio(self) -> float:
        """Fraction of idle time due to pipeline bubbles."""
        return (self.num_stages - 1) / self.num_latency_units
```

## 9. Python Implementation

```python
import torch
import torch.distributed as dist
from torch import nn
from typing import Optional, List
from transformers import AutoModelForCausalLM
import logging

logging.basicConfig(level=logging.INFO)
logger = logging.getLogger("pipeline-parallel")

class PipelineInference:
    """Run inference across pipeline-parallel GPUs."""

    def __init__(
        self,
        model_name: str,
        pp_size: int,
        pp_rank: int,
        device: torch.device,
    ):
        self.pp_size = pp_size
        self.pp_rank = pp_rank
        self.device = device

        # Load model and partition
        full_model = AutoModelForCausalLM.from_pretrained(
            model_name, torch_dtype=torch.bfloat16
        )
        self.stage_model = self._partition_model(full_model)

        # Determine next/prev ranks
        self.prev_rank = pp_rank - 1 if pp_rank > 0 else None
        self.next_rank = pp_rank + 1 if pp_rank < pp_size - 1 else None
        self.is_last = pp_rank == pp_size - 1
        self.is_first = pp_rank == 0

        logger.info(
            f"PP rank {pp_rank}/{pp_size} — "
            f"{'first' if self.is_first else 'middle' if not self.is_last else 'last'} stage, "
            f"{self._count_params()/1e6:.1f}M params"
        )

    def _partition_model(self, model) -> nn.Module:
        """Split model layers across pipeline stages."""
        layers = model.model.layers
        total_layers = len(layers)
        layers_per_stage = total_layers // self.pp_size

        start_idx = self.pp_rank * layers_per_stage
        end_idx = (
            start_idx + layers_per_stage
            if self.pp_rank < self.pp_size - 1
            else total_layers
        )

        # Keep only our layers; remove others to save memory
        for i in range(total_layers):
            if i < start_idx or i >= end_idx:
                model.model.layers[i] = nn.Identity()

        # Last stage keeps lm_head
        if not self.is_last:
            model.lm_head = nn.Identity()

        return model.to(self.device)

    @torch.inference_mode()
    def forward(self, input_ids: torch.Tensor) -> torch.Tensor:
        """Run forward pass through the pipeline."""

        # Receive from previous stage (if not first)
        if not self.is_first:
            # Receive hidden states from previous stage
            shape_tensor = torch.zeros(4, dtype=torch.long, device=self.device)
            dist.recv(shape_tensor, src=self.prev_rank)
            h_shape = tuple(shape_tensor.tolist())
            x = torch.zeros(*h_shape, dtype=torch.bfloat16, device=self.device)
            dist.recv(x, src=self.prev_rank)
        else:
            # First stage: embed tokens
            x = self.stage_model.model.embed_tokens(input_ids)

        # Run our layers
        for layer in self.stage_model.model.layers:
            if isinstance(layer, nn.Identity):
                continue
            x = layer(x)[0]  # HuggingFace returns tuple

        # Final norm (only last stage has real lm_head)
        if self.is_last:
            x = self.stage_model.model.norm(x)
            logits = self.stage_model.lm_head(x)
            return logits

        # Send to next stage
        shape_tensor = torch.tensor(list(x.shape), dtype=torch.long, device=self.device)
        dist.send(shape_tensor, dst=self.next_rank)
        dist.send(x, dst=self.next_rank)
        return None

    def _count_params(self) -> int:
        return sum(
            p.numel() for p in self.stage_model.parameters()
            if not isinstance(p, nn.Identity)
        )


def pp_generate(
    pipe: PipelineInference,
    input_ids: torch.Tensor,
    max_new_tokens: int = 100,
) -> Optional[torch.Tensor]:
    """Generate tokens using pipeline parallelism."""
    generated = input_ids

    for step in range(max_new_tokens):
        if pipe.is_first:
            logits = pipe.forward(generated)
        else:
            logits = pipe.forward(None)

        if pipe.is_last and logits is not None:
            next_token = torch.argmax(logits[:, -1, :], dim=-1, keepdim=True)
            # Send token back to first stage
            dist.send(next_token, dst=0)

        if pipe.is_first:
            # Receive token from last stage
            token = torch.zeros(1, 1, dtype=torch.long, device=pipe.device)
            dist.recv(token, src=pipe.pp_size - 1)
            generated = torch.cat([generated, token], dim=-1)

    return generated if pipe.is_first else None
```

## 10. Folder Structure

```
pipeline_parallel_project/
├── launch.sh               # mpirun / torchrun launcher
├── config.yaml
├── pp_inference/
│   ├── __init__.py
│   ├── main.py             # Entry point per rank
│   ├── pipeline_stage.py   # PipelineStage module
│   ├── scheduler.py        # Pipeline schedule generator
│   ├── communication.py    # P2P send/recv helpers
│   ├── partitioner.py      # Model partition logic
│   └── server.py           # FastAPI (rank 0 only)
├── k8s/
│   ├── statefulset.yaml    # StatefulSet per stage
│   └── service.yaml
└── tests/
    └── test_pipeline.py
```

## 11. Configuration

```yaml
pipeline_parallel:
  size: 4                      # Number of pipeline stages
  schedule: 1f1b               # 1F1B, simple, interleaved
  microbatch_size: 4           # Micro-batches for pipelining
  communication_backend: nccl  # nccl, gloo, mpi

model:
  name: meta-llama/Llama-3.2-3B-Instruct
  dtype: bfloat16
  layers_per_stage: auto       # auto = total_layers // pp_size

inference:
  max_seq_len: 4096
  max_batch_size: 8
  warmup_steps: 3              # Warmup steps before serving

network:
  p2p_backend: nccl            # P2P communication backend
  use_rdma: true               # RDMA for inter-node communication
  bandwidth_gbps: 200          # InfiniBand bandwidth
```

## 12. Flowchart

```
First Stage (Rank 0):            Middle Stage (Rank 1):         Last Stage (Rank 2):
┌──────────────────┐             ┌──────────────────┐          ┌──────────────────┐
│  Receive Input   │             │  Wait for P2P    │          │  Wait for P2P    │
└────────┬─────────┘             └────────┬─────────┘          └────────┬─────────┘
         │                               │                            │
         ▼                               ▼                            ▼
┌──────────────────┐             ┌──────────────────┐          ┌──────────────────┐
│  Embedding       │             │  Recv hidden     │          │  Recv hidden     │
└────────┬─────────┘             │  from Stage 0    │          │  from Stage 1    │
         │                       └────────┬─────────┘          └────────┬─────────┘
         ▼                               │                            │
┌──────────────────┐                      ▼                            ▼
│  Layers 0-9      │             ┌──────────────────┐          ┌──────────────────┐
│  (10 layers)     │             │  Layers 10-19    │          │  Layers 20-29    │
└────────┬─────────┘             └────────┬─────────┘          └────────┬─────────┘
         │                               │                            │
         ▼                               ▼                            ▼
┌──────────────────┐             ┌──────────────────┐          ┌──────────────────┐
│  Send hidden     │             │  Send hidden     │          │  Final Norm      │
│  to Stage 1      │             │  to Stage 2      │          │  + LM Head       │
└──────────────────┘             └──────────────────┘          └────────┬─────────┘
                                                                        │
                                                                        ▼
                                                                 ┌──────────────────┐
                                                                 │  Sample Token    │
                                                                 │  → First Stage   │
                                                                 └──────────────────┘
```

## 13. Sequence Diagram

```
Stage 0 (Rank 0)       Stage 1 (Rank 1)       Stage 2 (Rank 2)       Stage 3 (Rank 3)
     │                       │                       │                       │
     │───input──────────────▶│                       │                       │
     │                       │───hidden─────────────▶│                       │
     │                       │                       │───hidden─────────────▶│
     │                       │                       │                       │
     │                       │                       │◀──────logits──────────│
     │◀─────token────────────│─token (via P2P back)─│                       │
     │                       │                       │                       │
     │───next input─────────▶│                       │                       │
     │                       │───hidden─────────────▶│                       │
     │                       │                       │───hidden─────────────▶│
     │                       │                       │◀──────logits──────────│
     │◀─────token────────────│──token───────────────│                       │
     │                       │                       │                       │
```

## 14. Pros

- **Memory proportional** — each GPU holds only `1/pp_size` of model layers
- **Lower communication** — P2P between stage boundaries is cheaper than all-reduce in TP
- **Cross-node capable** — works over moderate network bandwidth (50-200 GB/s)
- **Good for training** — 1F1B schedule achieves near-linear scaling
- **Flexible sizing** — adapt to any number of GPUs, not constrained by layer count
- **Composability** — combine with TP (TP within node, PP across nodes)

## 15. Cons

- **Pipeline bubbles** — some GPUs idle during warm-up and cool-down phases
- **Higher latency** — sequential execution adds `pp_size` network hops
- **Load imbalance** — uneven layer compute leads to straggler stages
- **Complex scheduling** — requires careful micro-batch management for efficiency
- **KV cache imbalance** — last stage has lm_head overhead; first has embedding
- **Fault sensitivity** — one slow stage slows the entire pipeline

## 16. Alternatives

| Method | Best For | Tradeoffs |
|--------|----------|-----------|
| **Tensor Parallelism** | Latency-sensitive inference | High bandwidth requirement |
| **DeepSpeed ZeRO** | Training large models | Not optimized for inference |
| **Expert Parallelism** | MoE models (Mixtral) | Load imbalance across experts |
| **Sequence Parallelism** | Long-context training | More complex implementation |
| **FSDP** | Distributed training | Higher communication than PP |

## 17. Performance Considerations

**Pipeline bubble ratio:**
```
Formula: bubble_ratio = (pp_size - 1) / (pp_size * microbatch_count)
Example: pp_size=4, microbatches=8 → (4-1)/(32) = 9.4% bubbles
          pp_size=4, microbatches=2 → (4-1)/(8) = 37.5% bubbles

For inference: higher micro-batch count reduces bubble ratio
For latency: fewer micro-batches reduces per-request latency
Tradeoff: throughput vs latency
```

**Stage balancing:**
```
Goal: Each stage has equal compute time
- Llama-3-70B (80 layers): 20 layers/stage at PP=4 ✓
- GPT-3 175B (96 layers): 24 layers/stage at PP=4 ✓
- Problem: embedding and lm_head are on first/last stages
  Solution: add a dummy layer or redistribute
```

**Optimal PP size:**
```
PP=2: Minimal benefit, 2x memory reduction, ~25% bubble with 4 micro-batches
PP=4: Good balance, 4x memory reduction, ~9% bubble
PP=8: Significant bubbles, only for massive models (405B+)
```

## 18. Scaling to Millions

**Production scaling with PP + TP (3D parallelism):**

```
┌─────────────────────────────────────────────────────────────┐
│  Node 1 (DGX H100)          Node 2 (DGX H100)                │
│  ┌─────────────────────┐    ┌─────────────────────┐          │
│  │ PP Stage 0           │    │ PP Stage 1           │          │
│  │ ┌───┐ ┌───┐ ┌───┐  │    │ ┌───┐ ┌───┐ ┌───┐  │          │
│  │ │G0 │ │G1 │ │G2 │  │    │ │G0 │ │G1 │ │G2 │  │          │
│  │ └───┘ └───┘ └───┘  │    │ └───┘ └───┘ └───┘  │          │
│  │   TP group of 4     │    │   TP group of 4     │          │
│  └─────────┬───────────┘    └─────────┬───────────┘          │
│            │  P2P (IB NDR200)         │                      │
│            └──────────────────────────┘                      │
│                                                               │
│  Node 3 (DGX H100)          Node 4 (DGX H100)                │
│  ┌─────────────────────┐    ┌─────────────────────┐          │
│  │ PP Stage 2           │    │ PP Stage 3           │          │
│  │ TP group of 4       │    │ TP group of 4       │          │
│  └─────────┬───────────┘    └─────────┬───────────┘          │
│            │                          │                       │
│            └──────────────────────────┘                       │
└─────────────────────────────────────────────────────────────┘
```

**For 1M tokens/day (Llama 405B):**
```
Model size:      405B parameters → 810GB (FP16)
Weights memory:  810GB → ~10 H100 80GB for weights alone
KV cache:        ~200GB (8K context, batch 256)
Total:           1010GB → ~13 H100-80GB
With PP=4 + TP=4: 16 H100-80GB (with headroom)
Throughput:      ~500K tokens/hour
```

## 19. Failure Scenarios

| Failure | Impact | Mitigation |
|---------|--------|------------|
| **Stage failure** | Entire pipeline stops | Replicate pipelines; load balance |
| **Slow stage** | All stages slowed down | Monitor per-stage latency; balance layers |
| **P2P timeout** | Pipeline stalls | P2P timeout, retry, fail to secondary pipeline |
| **Memory imbalance** | One stage OOM | Adjust layer distribution per stage |
| **Network partition** | Inter-node P2P fails | Fall back to single-node PP (reduce PP size) |

## 20. Security

| Concern | Mitigation |
|---------|------------|
| **Inter-node P2P** | Data in transit between stages. Use encrypted InfiniBand or tunnel |
| **Side channels** | Stage timing reveals sequence length. Pad to fixed length |
| **Stage isolation** | Each stage on separate node limits blast radius |
| **Cross-node auth** | Authenticate pipeline membership, prevent unauthorized stage injection |

## 21. Monitoring

**Key metrics:**
```
pp:stage_compute_time_ms        — Time per stage for one micro-batch
pp:p2p_send_time_ms             — Time for P2P send
pp:p2p_recv_time_ms             — Time for P2P receive (includes wait)
pp:pipeline_bubble_ratio        — Idle / total time per stage
pp:token_throughput              — Tokens per second (whole pipeline)
pp:stage_memory_used_gb         — Per-stage memory (weights + cache)
pp:kv_cache_size_per_stage_mb   — KV cache per stage
```

**Alerts:**
- Stage compute time imbalance > 20% → rebalance layers
- P2P recv wait time > 100ms → network bottleneck or straggler
- Pipeline throughput < 50% of theoretical → check for hardware issues
- Any stage OOM → reduce batch size or adjust partitioning

## 22. Interview Questions

**Q1: Explain the 1F1B pipeline schedule.**
*A: 1F1B (One Forward, One Backward) is the standard pipeline schedule for training. A stage processes one micro-batch forward, then immediately switches to processing another micro-batch's backward pass, overlapping computation. For inference, the simpler "FCFS" schedule works — each micro-batch flows forward through all stages sequentially, with pipelining across micro-batches.*

**Q2: What are pipeline bubbles and how do you minimize them?**
*A: Pipeline bubbles are idle time when a stage has no work because earlier stages haven't finished processing. Bubble ratio = (P-1) / (P * M) where P=stages, M=micro-batches. Minimize by: (1) increasing micro-batches, (2) reducing stage count, (3) using interleaved 1F1B schedule, (4) overlapping communication with computation.*

**Q3: How does pipeline parallelism differ for training vs inference?**
*A: Training requires both forward and backward passes, needing 1F1B or similar schedules. Inference only requires forward pass — simpler scheduling, higher efficiency, lower memory (no gradients). Inference also benefits from KV caching, which adds per-stage memory proportional to the number of layers in that stage.*

**Q4: When would you use pipeline parallelism over tensor parallelism?**
*A: Use PP when: (1) model spans multiple nodes (TP is limited to node boundary), (2) inter-GPU bandwidth is limited (P2P is cheaper than all-reduce), (3) you need fine-grained memory control per stage. Use TP when: (1) latency is critical, (2) model fits within single NVLink domain, (3) you want simpler scheduling. In practice, use both: TP within node, PP across nodes.*

## 23. Cheat Sheet

```bash
# Launch 4-stage pipeline across 4 nodes
# Each node has 4 GPUs (TP=4 within node, PP=4 across nodes)
# Node 0:
torchrun --nproc_per_node=4 --nnodes=4 --node_rank=0 \
    pp_inference/main.py --pp-size 4 --tp-size 4

# Node 1:
torchrun --nproc_per_node=4 --nnodes=4 --node_rank=1 \
    pp_inference/main.py --pp-size 4 --tp-size 4

# Node 2:
torchrun --nproc_per_node=4 --nnodes=4 --node_rank=2 \
    pp_inference/main.py --pp-size 4 --tp-size 4

# Node 3:
torchrun --nproc_per_node=4 --nnodes=4 --node_rank=3 \
    pp_inference/main.py --pp-size 4 --tp-size 4

# Environment variables
export MASTER_ADDR=10.0.0.1    # Node 0's IP
export MASTER_PORT=29500
export NCCL_NET=IB             # Use InfiniBand for P2P
export NCCL_IB_HCA=mlx5_0:1    # IB interface
```

## 24. Common Mistakes

1. **Uneven layer distribution** — if layers have different compute time (e.g., first layer has embedding), stages are imbalanced. Measure per-layer latency and distribute accordingly.

2. **Using PP when model fits on one node** — PP adds unnecessary latency for models that fit on a single DGX node. Use TP within node, PP only for cross-node.

3. **Not overlapping P2P with compute** — P2P send/recv blocks by default. Use `torch.distributed.batch_isend_irecv()` to overlap communication with computation.

4. **Too few micro-batches** — micro-batch count = 1 means no pipelining (serial execution). Need at least `pp_size` micro-batches for full pipeline efficiency.

5. **Ignoring network bandwidth** — P2P across nodes at 200 GB/s is 3-4x slower than NVLink (900 GB/s). Factor this into stage placement and batch size decisions.

6. **Not pinning stages to NUMA nodes** — cross-NUMA P2P adds latency. Pin each stage's CPU threads to the NUMA node closest to its GPU.

## 25. Production Best Practices

1. **Combine TP + PP optimally** — use TP within a node (NVLink bandwidth), PP across nodes (InfiniBand bandwidth). Typical ratio: TP=4, PP=4 for 16 GPUs.

2. **Benchmark per-stage latency** — before production, run a profiling pass that measures compute time per layer. Distribute layers so each stage has equal total time.

3. **Minimize P2P transfers** — for decode steps, only send the last token's hidden state (not the full sequence). This reduces P2P volume from `seq_len × d_model` to `1 × d_model`.

4. **Pre-allocate P2P buffers** — create send/recv buffers during initialization to avoid allocation overhead during inference.

5. **Use `batch_isend_irecv`** — non-blocking P2P allows overlapping computation with communication, hiding network latency.

6. **Monitor per-stage GPU utilization** — if one stage is at 100% while others are at 50%, rebalance layers. Target: within 10% utilization across all stages.

7. **Implement per-stage health checks** — each stage should report its health to a central monitor. If a stage fails, the entire pipeline must be restarted.

8. **Warm up all stages** — send a dummy request through the full pipeline before serving real traffic. This loads CUDA kernels and establishes P2P connections.

9. **Set appropriate NCCL P2P settings** — `NCCL_P2P_DISABLE=0`, `NCCL_P2P_LEVEL=NVL` for intra-node, `NCCL_SOCKET_IFNAME=ib0` for inter-node.

10. **Use gradient checkpointing during warmup** — for initial testing, gradient checkpointing reduces memory but adds compute. Remove in production for best performance.
