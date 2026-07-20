# Autoscaling for AI Systems

> Author: **Tamilselvan** · ✉️ tamilselvan.sde@gmail.com · 🔗 [LinkedIn](https://www.linkedin.com/in/tamilselvan-ai/)
>

## 1. What is it?
Autoscaling for AI dynamically adjusts compute resources (pods, GPU nodes, storage) based on real-time demand — scaling up during traffic spikes and down during lulls. Key metrics include request rate, GPU utilization, queue depth, and inference latency.

## 2. Why do we need it?
AI inference traffic is highly variable: 10x spikes during promotions, quiet over weekends, zero-load at 3am. Fixed infrastructure either wastes money (idle GPUs) or fails under load. Autoscaling ensures cost-efficient, reliable serving without manual capacity planning.

## 3. Real-world Example
**Grammarly** serves 30M+ daily users with AI writing suggestions. Their autoscaler scales inference pods based on request queue depth with a 2-minute cooldown. GPU nodes are pre-provisioned with model weights. During weekday peak (9am-11am), 200+ pods serve 50K req/s; overnight, it drops to 20 pods — saving 80% GPU cost.

## 4. Architecture Diagram (ASCII)
```
+-----------------------+     +------------------------+
|   Metrics Sources     |     |   Scaling Policies       |
|                       |     |                          |
| Request Rate          |     | HPA: CPU > 80% -> +1    |
| GPU Utilization       |     | HPA: GPU > 75% -> +2    |
| Queue Depth           |     | KPA: RPS > 1000 -> +1   |
| Inference Latency     |     | Cron: 9am -> min=20     |
| Memory/CPU            |     | Cron: 11pm -> min=3     |
+----------+------------+     +----------+--------------+
           |                            |
           v                            v
+----------+----------------------------+--------------+
|                   Autoscaler                          |
|   (K8s HPA + KEDA + Cluster Autoscaler)              |
+----------+----------------------------+--------------+
           |                            |
           v                            v
+----------+-----------+   +------------+--------------+
|   Pod Autoscaling     |   |   Node Autoscaling        |
|   (Deployment HPA)    |   |   (Cluster Autoscaler)    |
|   Scale pods          |   |   Add/remove GPU nodes    |
|   per deployment      |   |   Per node pool           |
+-----------------------+   +---------------------------+
```

## 5. Internal Working
Three levels: **Horizontal Pod Autoscaler (HPA)** scales pod replicas based on CPU/memory/custom metrics. **KEDA** extends HPA with event-driven scalers (Kafka consumer lag, queue depth). **Cluster Autoscaler** adds/removes nodes when pods are pending or nodes are underutilized. Scaling has cooldown periods to avoid thrashing.

## 6. Production Flow
```
Traffic spike -> Request rate increases -> Queue depth grows
-> KEDA detects lag -> HPA scales pods -> Pods pending for GPUs
-> Cluster Autoscaler adds GPU node -> Pods scheduled -> Serving
... traffic drops -> HPA scales pods down -> Node underutilized
-> Cluster Autoscaler removes node (after cooldown)
```

## 7. HLD
| Layer | Scaler | Metric | Behavior |
|-------|--------|--------|----------|
| Pod | HPA | GPU utilization > 75% | Scale up 2 pods, scale down after 5 min |
| Pod | KEDA | Kafka consumer lag > 100 | Scale up, 1 per 50 lag |
| Pod | CPA | Inference latency p99 > 1s | Scale up, aggressive |
| Node | Cluster Autoscaler | Pending pods > 0 | Add GPU node (5-15 min) |
| Node | Cluster Autoscaler | Node utilization < 30% for 10min | Remove node |
| Region | Global LB | Regional capacity < 80% | Route traffic to other region |

## 8. LLD
```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: llm-inference-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: llm-inference
  minReplicas: 3
  maxReplicas: 50
  behavior:
    scaleUp:
      stabilizationWindowSeconds: 60
      policies:
        - type: Pods
          value: 4
          periodSeconds: 15
        - type: Percent
          value: 100
          periodSeconds: 15
      selectPolicy: Max
    scaleDown:
      stabilizationWindowSeconds: 300
      policies:
        - type: Pods
          value: 1
          periodSeconds: 60
  metrics:
    - type: Resource
      resource:
        name: nvidia.com/gpu-memory
        target:
          type: Utilization
          averageUtilization: 75
    - type: Pods
      pods:
        metric:
          name: inference_queue_depth
        target:
          type: AverageValue
          averageValue: 10
```

## 9. Python Implementation
```python
from dataclasses import dataclass
import time, statistics

@dataclass
class ScalingDecision:
    action: str  # "scale_up", "scale_down", "none"
    replicas: int
    reason: str

class CustomAutoscaler:
    def __init__(self, min_replicas=3, max_replicas=50):
        self.min = min_replicas
        self.max = max_replicas
        self.current = min_replicas
        self.metrics_history = []
        self.cooldown_until = 0

    def collect_metrics(self) -> dict:
        return {
            "request_rate": self._get_request_rate(),
            "gpu_util": self._get_avg_gpu_util(),
            "queue_depth": self._get_queue_depth(),
            "p99_latency": self._get_p99_latency(),
        }

    def decide(self) -> ScalingDecision:
        metrics = self.collect_metrics()
        desired = self.current

        # GPU utilization scaling
        if metrics["gpu_util"] > 80:
            desired = int(self.current * 1.5)
        elif metrics["gpu_util"] > 70:
            desired = int(self.current * 1.2)

        # Queue depth scaling (more aggressive)
        if metrics["queue_depth"] > 100:
            desired = max(desired, int(metrics["queue_depth"] / 10))

        # Latency scaling
        if metrics["p99_latency"] > 2000:  # > 2s
            desired = max(desired, self.current * 2)

        # Scale down conservatively
        if metrics["gpu_util"] < 30 and metrics["queue_depth"] < 10:
            desired = max(self.min, self.current - 1)

        desired = max(self.min, min(self.max, desired))

        if desired > self.current:
            return ScalingDecision("scale_up", desired, f"High load")
        elif desired < self.current:
            return ScalingDecision("scale_down", desired, f"Low load")
        return ScalingDecision("none", self.current, "Stable")

    def _get_request_rate(self) -> float: ...
    def _get_avg_gpu_util(self) -> float: ...
    def _get_queue_depth(self) -> int: ...
    def _get_p99_latency(self) -> float: ...
```

## 10. Folder Structure
```
autoscaling/
  policies/
    hpa.yaml
    keda-scaled-object.yaml
    cluster-autoscaler-config.yaml
    vpa.yaml
  metrics/
    prometheus-rules.yaml
    custom-metrics-server.yaml
  scaling-logic/
    predictive_scaler.py
    cron_scaler.py
    custom_metrics.py
  tests/
    test_scaling.py
```

## 11. Configuration
```yaml
autoscaling:
  inference:
    min: 3
    max: 50
    metrics:
      - gpu_utilization: { target: 75, scale_up_delta: 2 }
      - queue_depth: { target: 10, scale_up_delta: 1 }
    cooldown:
      scale_up: 60s
      scale_down: 300s
    cron:
      - schedule: "0 8 * * 1-5"  # 8am weekdays
        min_replicas: 20
      - schedule: "0 22 * * *"    # 10pm daily
        min_replicas: 3

  embedding:
    min: 2
    max: 20
    metrics:
      - cpu: { target: 80 }
      - request_rate: { target: 100 }
```

## 12. Flowchart
```
Start
  |
  v
Collect Metrics (GPU util, queue depth, latency)
  |
  v
Evaluate Scale-Up Rules:
  GPU > 80% OR Queue > 100 OR Latency > 2s
  |                        |
  Yes                      No
  |                        |
  v                        v
Check Cooldown (60s)     Evaluate Scale-Down:
  |                        GPU < 30% AND Queue < 10
  |                        AND Cooldown (300s)
  |                        |
  v                        v
Scale Up                Scale Down
  |                        |
  v                        v
Update HPA              Update HPA
  |                        |
  v                        v
Cluster Autoscaler      Cluster Autoscaler
  (add nodes if needed)   (remove nodes if idle)
  |
  v
Back to Start (poll every 15s)
```

## 13. Sequence Diagram
```
LoadGen     HPA       Pods      ClusterAutoscaler    NodePool
  |         |          |               |                |
  |--spike->|          |               |                |
  |         |--collect metrics         |                |
  |         |--GPU > 80%              |                |
  |         |--scale to 6            |                |
  |         |--create pods---------->|                |
  |         |          |--pending GPU resource         |
  |         |          |               |                |
  |         |          |--pending------|                |
  |         |          |               |--add node----->|
  |         |          |               |                |--provision GPU
  |         |          |               |<--ready--------|
  |         |          |--schedule----|                |
  |         |<--ready--|               |                |
  |         |          |               |                |
  |--drain->|          |               |                |
  |         |--GPU < 30%              |                |
  |         |--scale to 4           |                |
  |         |--delete pods--------->|                |
  |         |          |               |--node idle     |
  |         |          |               |--remove node-->|
```

## 14. Pros
- Cost savings (30-80% GPU cost reduction); Automatic capacity management; No manual intervention for load spikes; Supports diverse metrics (GPU, queue, custom); Predictive and cron-based scaling for known patterns.

## 15. Cons
- Complex configuration (multiple scalers interact); Cooldown tuning is fragile; Aggressive scale-down causes thrashing; GPU node provisioning is slow (5-15min); HPA doesn't handle GPU memory natively; Cost if misconfigured (scale up and never down).

## 16. Alternatives
- **Manual scaling**: Simple but wasteful; **Cron-based only**: Good for predictable patterns; **Predictive scaling**: ML models forecast demand; **Serverless (Lambda)**: Auto-scale inherently; **Spot/Preemptible instances**: Scale with cost awareness.

## 17. Performance Considerations
- HPA reaction time: 15-30s (metrics collection + decision); Cluster Autoscaler: 5-15min (node provisioning); Pre-pull model weights to avoid startup delay; Scale up aggressively, scale down slowly; Use predictive scaling for known traffic patterns; GPU allocation is slower than CPU — plan buffer.

## 18. Scaling to Millions
- **10K req/min**: 5 pods -> auto-scale to 20, 3-5 GPU nodes; **100K**: 50 pods -> auto-scale to 200, 20-30 GPU nodes; **1M**: 500+ pods, multi-region; Use RPS-based scaling for large clusters; Implement max surge limit to protect downstream; Pre-create GPU nodes in warm pool; Different scaling policies per region.

## 19. Failure Scenarios
- **HPA flapping**: Update cooldown windows; **Cluster Autoscaler stuck**: Check pending pods for scheduling constraints; **GPU node provisioning fails**: Switch node pool or region; **Metrics server down**: HPA uses last known values; **Scale-down too aggressive**: Set minReplicas higher; **Node pool exhausted**: Set max node limits.

## 20. Security
- HPA respects resource quotas; Limit max replica count to prevent DoS; Monitor autoscaler decisions for anomalies; RBAC restricts who can modify HPA policies; Node scaling actions logged; Alert on unexpected scaling events; Rate limit scaling API calls.

## 21. Monitoring
- **HPA metrics**: Current/target replicas; **Scaling events**: Frequency and direction; **Node pool**: current/desired/max nodes; **GPU idle cost**: Cost of underutilized resources; **Scale-up latency**: Time from metric breach to pod ready; **Autoscaler errors**: Failed decisions; **Dashboard**: Grafana with HPA + CA panels.

## 22. Interview Questions
1. *Design autoscaling for an LLM inference service.* — HPA on GPU utilization + queue depth + latency, Cluster Autoscaler for nodes, cron for known patterns.
2. *How do you prevent autoscaler thrashing?* — Cooldown windows, stabilization period, scale-up fast / scale-down slow.
3. *What metrics matter for AI inference autoscaling?* — GPU utilization, request queue depth, p99 latency, memory.
4. *How to handle cold starts in autoscaling?* — Pre-pull images, pre-download models, warm node pool, predictive scaling.

## 23. Cheat Sheet
```
Autoscaling Levels:
  Pod: HPA/KEDA (seconds)
  Node: Cluster Autoscaler (minutes)
  Region: Global Load Balancer (minutes)

Scale Up: GPU > 75% or Queue > 100 or Latency > 2s
Scale Down: GPU < 30% for 5min steady period
Cooldown: Up 60s, Down 300s
```

## 24. Common Mistakes
- Aggressive scale-down leading to thrashing; No GPU-specific HPA metrics; Same scaling policy for inference and batch; Cluster Autoscaler not configured with GPU pools; No cron scaling for known patterns; Metrics server not collecting GPU metrics; Too aggressive maxReplicas (cost shock).

## 25. Production Best Practices
- Start conservative (high minReplicas) for critical services; Implement cron-based scaling for predictable traffic; Use different HPA behaviors for scale-up (fast) and down (slow); Pre-warm GPU nodes with model weights; Monitor autoscaler decisions (Grafana dashboard); Set maxReplicas with cost budget; Test autoscaling with load generators; Combine HPA + KEDA for event-driven + resource metrics; Audit scaling events weekly.
