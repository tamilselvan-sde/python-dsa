# GPU Scheduling for AI

## 1. What is it?
GPU scheduling is the allocation and management of GPU resources (cores, memory, bandwidth) across AI workloads — ensuring that training jobs and inference services get the right GPU type (A100, H100, T4) with appropriate isolation, and that GPU memory is not wasted.

## 2. Why do we need it?
GPUs are the most expensive resource in AI infrastructure ($10-40/hr per A100). Without proper scheduling: GPU memory fragmentation wastes capacity, jobs pile up waiting for GPUs, one workload can starve another, and training jobs are interrupted by inference spikes. Effective GPU scheduling maximizes utilization and reduces cost.

## 3. Real-world Example
**Google DeepMind** uses GPU scheduling across 10,000+ TPU/GPU accelerators. Training jobs get priority on reserved A100s; inference gets on-demand H100s with elastic scaling. GPU memory is oversubscribed 1.5x (not all models use full memory). Preemptible jobs fill unused GPU slots during batch processing windows. Utilization consistently above 85%.

## 4. Architecture Diagram (ASCII)
```
+--------------------------------------------------+
|              GPU Scheduler                         |
|  (Kubernetes + Volcano/Kueue/VPA)                 |
+--------------------------------------------------+
          |             |              |
   +------v------+ +---v--------+ +---v--------+
   | Training    | | Inference  | | Batch      |
   | (Priority)  | | (Elastic)  | | (Best-eff) |
   +------+------+ +---+--------+ +---+--------+
          |             |              |
   +------v------+ +---v--------+ +---v--------+
   | A100x8      | | H100x1     | | T4x4       |
   | Reserved    | | On-demand  | | Spot       |
   +------+------+ +---+--------+ +---+--------+
          |             |              |
          +------+------+--------------+
                 |
          +------v------+
          |  GPU Nodes   |
          |  (NVIDIA)    |
          |  +--------+ |
          |  | GPU 0  | |
          |  | GPU 1  | |
          |  | GPU 2  | |
          |  | GPU 3  | |
          |  +--------+ |
          +-------------+
```

## 5. Internal Working
GPU schedulers manage a queue of workloads requesting GPUs. Each workload specifies: GPU type, memory (GB), compute mode (MPS/MIG exclusive), and priority. The scheduler matches workloads to available GPUs, considering: node affinities, GPU memory availability, NUMA topology, fragmentation, and preemption policies. MIG (Multi-Instance GPU) partitions A100/H100 into up to 7 isolated instances.

## 6. Production Flow
```
Job submitted -> Scheduler queues -> Priority sorting ->
GPU availability check -> Node selection -> GPU isolation setup
-> Container starts with GPU -> Monitoring tracks utilization
-> Job completes -> GPU freed -> Next job scheduled
```

## 7. HLD
| Component | Role | Tech |
|-----------|------|------|
| Queue Manager | Prioritize and order jobs | Volcano/Kueue |
| Node Selector | Match jobs to GPU nodes | K8s scheduler + extended resources |
| GPU Isolator | MIG partition, memory limits | NVIDIA MIG Manager |
| Preemptor | Interrupt low-priority for high | Kueue preemption |
| Metrics | GPU utilization tracking | DCGM + Prometheus |
| Oversubscription | Allow >100% requested allocation | Custom quota system |

## 8. LLD
```python
from dataclasses import dataclass
from enum import Enum

class GPUType(Enum):
    T4 = "nvidia-tesla-t4"
    A100 = "nvidia-tesla-a100"
    H100 = "nvidia-h100-80gb"

class Priority(Enum):
    CRITICAL = 0   # Production inference
    HIGH = 1       # Training
    MEDIUM = 2     # Evaluation
    LOW = 3        # Batch

@dataclass
class GPURequest:
    job_id: str
    gpu_type: GPUType
    gpu_count: int
    memory_per_gpu: int  # in GB
    priority: Priority
    max_duration_minutes: int | None = None
    preemptible: bool = False

class GPUScheduler:
    def __init__(self):
        self.queue: list[GPURequest] = []
        self.available_gpus = self._discover_gpus()

    def submit(self, request: GPURequest):
        self.queue.append(request)
        self.queue.sort(key=lambda r: r.priority.value)

    def schedule(self) -> list[GPURequest]:
        allocated = []
        for req in self.queue[:]:
            if self._can_allocate(req):
                self._allocate(req)
                allocated.append(req)
                self.queue.remove(req)
        return allocated

    def _can_allocate(self, req: GPURequest) -> bool:
        return self.available_gpus.get(req.gpu_type, 0) >= req.gpu_count

    def _allocate(self, req: GPURequest):
        self.available_gpus[req.gpu_type] -= req.gpu_count

    def preempt(self, priority_threshold: Priority) -> list[str]:
        # Preempt lower-priority jobs running on GPUs needed by higher-priority queued jobs
        preempted = []
        for job in self.running_jobs:
            if job.priority.value > priority_threshold.value:
                self._stop_job(job.id)
                self.available_gpus[job.gpu_type] += job.gpu_count
                preempted.append(job.id)
        return preempted
```

## 9. Python Implementation
```python
# Minimal GPU scheduler using Kubernetes APIs
from kubernetes import client, config, watch

class GPUNodePoller:
    def __init__(self):
        config.load_incluster_config()
        self.core_v1 = client.CoreV1Api()

    async def get_gpu_availability(self) -> dict:
        nodes = self.core_v1.list_node()
        gpu_available = {"t4": 0, "a100": 0, "h100": 0}
        for node in nodes.items:
            gpu_type = node.metadata.labels.get(
                "nvidia.com/gpu.product", ""
            )
            capacity = int(node.status.capacity.get(
                "nvidia.com/gpu", 0
            ))
            allocatable = int(node.status.allocatable.get(
                "nvidia.com/gpu", 0
            ))
            if "a100" in gpu_type.lower():
                gpu_available["a100"] += allocatable
            elif "h100" in gpu_type.lower():
                gpu_available["h100"] += allocatable
            else:
                gpu_available["t4"] += allocatable
        return gpu_available

    def get_gpu_pod_spec(self, gpu_type: str, count: int = 1):
        return {
            "containers": [{
                "name": "gpu-worker",
                "image": "nvidia/cuda:12.1-runtime",
                "resources": {
                    "limits": {"nvidia.com/gpu": count},
                    "requests": {"nvidia.com/gpu": count},
                },
            }],
            "nodeSelector": {
                "nvidia.com/gpu.product": gpu_type,
            },
            "tolerations": [{
                "key": "nvidia.com/gpu",
                "operator": "Exists",
                "effect": "NoSchedule",
            }],
        }
```

## 10. Folder Structure
```
gpu-scheduler/
  scheduler/
    core.py          # Queue management and allocation
    priority.py      # Priority sorting and preemption
    isolation.py     # MIG partitioning logic
  k8s-integration/
    gpu-node-poller.py
    extended-resource-registrar.py
  monitoring/
    dcgm-exporter.py
    gpu-utilization-dashboard.py
  policies/
    priority-classes.yaml
    preemption-policy.yaml
  config/
    gpu-types.yaml
    quotas.yaml
```

## 11. Configuration
```yaml
gpu_scheduling:
  classes:
    t4:
      type: nvidia-tesla-t4
      memory: 16GB
      compute_mode: MIG 1g.5gb
      cost_per_hour: 0.35
    a100:
      type: nvidia-tesla-a100
      memory: 80GB
      compute_mode: MIG 2g.20gb
      cost_per_hour: 3.50

  queues:
    inference: { priority: 0, gpu_types: [a100], min_available: 5 }
    training: { priority: 1, gpu_types: [a100, h100] }
    batch: { priority: 2, gpu_types: [t4], preemptible: true }

  oversubscription:
    enabled: true
    ratio: 1.5  # Allow scheduling 150% of physical GPUs
    preemption_timeout: 30  # seconds to free GPU
```

## 12. Flowchart
```
Job Submitted
  |
  v
Priority Classification (Inference > Training > Batch)
  |
  v
Enqueue (priority-sorted)
  |
  v
Schedule Loop (every 5s):
  |
  v
Check GPU Availability
  |
  +-- GPU available -> Allocate -> Add tolerations + nodeSelector
  |
  +-- No GPU available:
       +-- Preemptible job? -> Check preempt lower-priority -> Run
       +-- Not preemptible? -> Wait in queue
  |
  v
Job Runs (GPU metrics tracked via DCGM)
  |
  v
Job Completes -> Free GPU -> Schedule next
```

## 13. Sequence Diagram
```
Client    Scheduler    K8s API     GPU Node    DCGM
  |          |           |            |          |
  |--submit->|           |            |          |
  |          |--queue----|            |          |
  |          |           |            |          |
  |          |--schedule |            |          |
  |          |--check GPU availability |          |
  |          |           |            |          |
  |          |--create pod----------->|          |
  |          |           |            |--start   |
  |          |           |            |--nvidia-smi
  |          |           |            |          |
  |          |           |            |--metrics-|
  |          |           |            |          |
  |<--status-|           |            |          |
```

## 14. Pros
- Maximizes GPU utilization (85%+ target); Priority-aware scheduling (inference always runs); Preemption for urgent workloads; MIG partitioning for isolation; Cost savings via spot GPU usage; Multi-tenant GPU sharing.

## 15. Cons
- Scheduler complexity (queue management, preemption); MIG has memory fragmentation; Oversubscription risks OOM; Preemption wastes GPU time for checkpointing; GPU memory is not migratable (no live migration); Requires DCGM/nvidia-smi monitoring.

## 16. Alternatives
- **Kubernetes default scheduler**: Limited GPU awareness; **Volcano**: Batch-aware, gang scheduling; **Kueue**: Queue-based, integrates with K8s; **Slurm**: HPC-focused; **AWS Batch**: Managed, less flexible; **Google CAIP**: Fully managed GPU scheduling.

## 17. Performance Considerations
- GPU memory is the most constrained resource; MIG adds 5-10% overhead vs full GPU; Preemption checkpoint saves 30-60s of recompute; NUMA affinity matters for multi-GPU (NVLink vs PCIe); Warm GPU nodes avoid 5-15 min cold start; Model loading from shared filesystem vs local NVMe (5x difference).

## 18. Scaling to Millions
- **10 GPUs**: Single queue, manual; **100**: Scheduler with priority queues; **1000**: Multi-cluster, federated scheduling; **10000**: Hierarchical (cluster-level + node-level); Use spot/preemptible for batch; MIG for inference with predictable load.

## 19. Failure Scenarios
- **GPU memory oversubscription**: OOM kills GPU processes — set hard limits; **GPU node failure**: Reschedule pods — need standby GPU capacity; **Driver mismatch**: CUDA version incompatible — use consistent base images; **MIG partition failure**: Reset GPU via nvidia-smi; **DCGM exporter down**: Blind scheduling — use node status fallback.

## 20. Security
- GPU isolation via MIG (hardware-level); Prevent GPU timing side-channel attacks (MIG vs Time-Slicing); Container-level GPU resource limits; Monitor GPU utilization for crypto-mining; Restrict nvidia-smi via PodSecurityPolicy; Isolate multi-tenant GPU workloads.

## 21. Monitoring
- **GPU utilization**: DCGM metrics (util %, memory used, temperature); **Queue depth**: Pending GPU requests; **Preemption rate**: How often lower-priority jobs interrupted; **Fragmentation**: GPU memory wasted due to MIG; **Cost**: Per-job GPU cost tracking; **Scheduling latency**: Time from submit to running; **Dashboard**: Grafana with DCGM Exporter.

## 22. Interview Questions
1. *How would you schedule inference and training on the same GPU cluster?* — Priority queues, inference > training, preemption for inference.
2. *What is MIG and when would you use it?* — Multi-Instance GPU partitions A100 into isolated GPUs; use for multi-tenant inference.
3. *How to handle GPU memory fragmentation?* — MIG profiles (1g.10gb, 2g.20gb), GPU memory defragmentation, job packing algorithms.
4. *How to maximize GPU utilization without impacting latency?* — Batch inference jobs in idle time, oversubscription with preemption, MIG for small models.

## 23. Cheat Sheet
```
GPU Scheduling Concepts:
- MIG: Hardware partition (A100/H100) — max 7 instances
- Time-slicing: Software sharing — no isolation
- Oversubscription: Over-allocate GPU memory, preempt when needed
- Priority Queues: Inference > Training > Batch

Key Tools: Volcano, Kueue, NVIDIA MIG Manager, DCGM
```

## 24. Common Mistakes
- No priority-based scheduling (inference starves); Oversubscription without preemption (OOM); MIG on wrong GPU (only A100/H100/H200); Not setting GPU memory limits; Ignoring NUMA topology (cross-socket GPU slows); No GPU monitoring (blind scheduling); Cold-start every pod (model download delays).

## 25. Production Best Practices
- Use priority classes for all GPU workloads; Monitor GPU memory fragmentation with DCGM; Implement preemption with graceful shutdown (30s grace); Reserve GPU capacity for critical inference; Use MIG for multi-tenant inference on A100s; Pre-pull model weights on GPU nodes; Implement GPU quota per team/user; Schedule GPU capacity planning quarterly; Test GPU failover with chaos engineering.
