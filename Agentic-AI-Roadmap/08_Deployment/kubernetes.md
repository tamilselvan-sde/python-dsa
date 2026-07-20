# Kubernetes for AI Systems

> Author: **Tamilselvan** · ✉️ tamilselvan.sde@gmail.com · 🔗 [LinkedIn](https://www.linkedin.com/in/tamilselvan-ai/)
>

## 1. What is it?
Kubernetes (K8s) for AI is the orchestration platform for deploying, scaling, and managing containerized AI workloads — LLM inference pods, embedding services, vector databases, and data pipelines — across GPU and CPU nodes with automated scheduling, health checking, and rolling updates.

## 2. Why do we need it?
AI workloads have unique infrastructure needs: GPU scheduling, model versioning, auto-scaling based on inference load, and high availability. Kubernetes provides declarative deployment, self-healing, horizontal scaling, and resource isolation — essential for production AI at scale.

## 3. Real-world Example
**OpenAI** runs ChatGPT on Kubernetes across thousands of A100/H100 GPUs. K8s handles: scheduling inference pods to GPU nodes, rolling model updates without downtime, scaling replicas based on queue depth, and automatically replacing failed pods. Each model version is a separate deployment with canary traffic routing.

## 4. Architecture Diagram (ASCII)
```
+------------------------------------------------------+
|                  Kubernetes Cluster                   |
|                                                        |
|  +------------------+  +------------------+           |
|  |   Control Plane   |  |   Control Plane   |         |
|  |   (HA: 3 nodes)   |  |   (HA: 3 nodes)   |         |
|  | API Server  etcd  |  | Scheduler  CM     |         |
|  +------------------+  +------------------+           |
|                                                        |
|  +-------------------------------------------------+  |
|  |              Worker Nodes                        |  |
|  |                                                   | |
|  |  +--------+  +--------+  +--------+  +--------+ | |
|  |  | GPU    |  | GPU    |  | CPU    |  | CPU    | | |
|  |  | Node 1 |  | Node 2 |  | Node 3 |  | Node 4 | | |
|  |  +--------+  +--------+  +--------+  +--------+ | |
|  |                                                   | |
|  |  Pods: LLM Inference, Embedding, Retrieval, etc   | |
|  +-------------------------------------------------+  |
+------------------------------------------------------+
```

## 5. Internal Working
Kubernetes schedules pods to nodes based on resource requests (CPU, memory, GPU). For AI, GPU nodes are tainted to prevent non-GPU pods from landing there. NodePort/Ingress routes traffic to inference pods. HorizontalPodAutoscaler scales pods based on CPU/memory/custom metrics. Cluster Autoscaler adds/removes nodes based on pending pod resource requirements.

## 6. Production Flow
```
Developer pushes code -> CI/CD builds image -> Helm deploy
-> K8s creates/updates pods -> Service routes traffic -> HPA scales
-> Cluster Autoscaler adds GPU nodes -> Steady state serving
```

## 7. HLD
| Component | K8s Resource | Scaling | Notes |
|-----------|-------------|---------|-------|
| LLM Inference | Deployment | HPA (GPU utilization) | NodeSelector for GPU nodes |
| Embedding | Deployment | HPA (CPU/RAM) | CPU-only nodes |
| Qdrant | StatefulSet | Manual sharding | PersistentVolume + local SSD |
| Orchestrator | Deployment | HPA (req/s) | Stateless, CPU-only |
| Redis | StatefulSet | Redis Cluster | PersistentVolume |
| Monitoring | DaemonSet | Node-level | Prometheus + Grafana |

## 8. LLD
```yaml
# llm-inference-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: llm-inference
spec:
  replicas: 3
  selector:
    matchLabels:
      app: llm
  template:
    metadata:
      labels:
        app: llm
    spec:
      nodeSelector:
        cloud.google.com/gke-accelerator: nvidia-tesla-a100
      tolerations:
        - key: nvidia.com/gpu
          operator: Exists
          effect: NoSchedule
      containers:
        - name: vllm
          image: vllm/vllm-openai:latest
          ports:
            - containerPort: 8000
          resources:
            limits:
              nvidia.com/gpu: 1
              memory: "64Gi"
              cpu: "8"
            requests:
              nvidia.com/gpu: 1
              memory: "48Gi"
              cpu: "6"
          env:
            - name: MODEL_NAME
              value: "meta-llama/Llama-3.1-8B-Instruct"
            - name: MAX_MODEL_LEN
              value: "4096"
          readinessProbe:
            httpGet:
              path: /health
              port: 8000
            initialDelaySeconds: 60
```

## 9. Python Implementation
```python
# Kubernetes client for dynamic scaling
from kubernetes import client, config

class K8sManager:
    def __init__(self):
        config.load_incluster_config()
        self.apps_v1 = client.AppsV1Api()
        self.autoscaling_v1 = client.AutoscalingV1Api()

    def scale_deployment(self, name: str, namespace: str, replicas: int):
        body = {"spec": {"replicas": replicas}}
        self.apps_v1.patch_namespaced_deployment_scale(
            name=name, namespace=namespace, body=body
        )

    def get_deployment_status(self, name: str, namespace: str):
        deploy = self.apps_v1.read_namespaced_deployment(name, namespace)
        return {
            "desired": deploy.spec.replicas,
            "ready": deploy.status.ready_replicas or 0,
            "available": deploy.status.available_replicas or 0,
        }

    def create_hpa(self, name: str, namespace: str, min: int, max: int, cpu_target: int):
        hpa = client.AutoscalingV1.HorizontalPodAutoscaler(
            metadata={"name": name},
            spec={
                "scaleTargetRef": {
                    "apiVersion": "apps/v1",
                    "kind": "Deployment",
                    "name": name,
                },
                "minReplicas": min,
                "maxReplicas": max,
                "targetCPUUtilizationPercentage": cpu_target,
            },
        )
        self.autoscaling_v1.create_namespaced_horizontal_pod_autoscaler(
            namespace=namespace, body=hpa
        )
```

## 10. Folder Structure
```
k8s/
  base/                      # Common configurations
    kustomization.yaml
    namespace.yaml
  llm-inference/
    deployment.yaml
    service.yaml
    hpa.yaml
    pdb.yaml                 # PodDisruptionBudget
    service-account.yaml
  embedding/
    deployment.yaml
    service.yaml
    hpa.yaml
  qdrant/
    statefulset.yaml
    service.yaml
    pvc.yaml
    configmap.yaml
  ingress/
    ingress.yaml
    cert-manager.yaml
  monitoring/
    prometheus.yaml
    grafana.yaml
  helm/                      # Helm charts for complex deployments
    Chart.yaml
    values.yaml
    templates/
```

## 11. Configuration
```yaml
# values.yaml (Helm)
inference:
  replicas: 3
  model: "meta-llama/Llama-3.1-8B-Instruct"
  resources:
    gpu: 1
    memory: "64Gi"
  autoscaling:
    enabled: true
    minReplicas: 3
    maxReplicas: 20
    metrics:
      - type: Resource
        resource:
          name: nvidia.com/gpu-memory
          target:
            type: Utilization
            averageUtilization: 80

cluster:
  gpuNodePool:
    machineType: a2-highgpu-1g
    initialNodeCount: 5
    minNodeCount: 3
    maxNodeCount: 50
  cpuNodePool:
    machineType: n2-standard-8
    initialNodeCount: 5
    autoscaling: true
```

## 12. Flowchart
```
kubectl apply -> API Server validates -> Scheduler assigns pods
-> Kubelet pulls image -> Container starts -> Readiness probe passes
-> Service routes traffic -> HPA monitors metrics
-> Pod CPU > 80% -> HPA scales up replicas
-> Node resources insufficient -> Cluster Autoscaler adds node
-> Node resources idle -> Cluster Autoscaler removes node
```

## 13. Sequence Diagram
```
Developer   K8s API     Scheduler   GPU Node   HPA   ClusterAutoscaler
  |           |            |           |        |          |
  |--apply--->|            |           |        |          |
  |           |--validate--|           |        |          |
  |           |--schedule------------->|        |          |
  |           |            |           |--pull  |          |
  |           |            |           |--start |          |
  |           |            |           |--ready |          |
  |           |            |           |        |          |
  |           |            |           |        |--watch---|
  |           |            |           |--scale |          |
  |           |            |           |        |          |
  |           |            |           |        |--add node (if pending pods)
```

## 14. Pros
- Declarative infrastructure as code; Self-healing (restarts failed pods); GPU scheduling with nodeSelector/tolerations; HPA for automatic scaling; Rolling updates with zero downtime; Rich ecosystem (Helm, Prometheus, Istio).

## 15. Cons
- Operational complexity (etcd, networking, upgrades); GPU node provisioning can be slow (5-15 min); Resource overhead (kubelet, dameonsets); Network bottlenecks for model weight downloads; Cost for control plane nodes; Debugging is harder than Docker Compose.

## 16. Alternatives
- **Docker Swarm**: Simpler but no GPU scheduling; **AWS ECS**: Managed, less flexible; **Slurm**: HPC-focused, better for training; **Serverless (Lambda)**: No infra, limited to small models; **Nomad**: Simpler, multi-datacenter.

## 17. Performance Considerations
- GPU pods need exclusive node access (no sharing unless MIG); Model download on startup can take 2-5 mins — pre-pull images; Use nodeSelector for GPU vs CPU workloads; Set resource requests == limits for GPU pods; Use PodDisruptionBudget for inference availability; Enable TopologyManager for NUMA-aware GPU scheduling.

## 18. Scaling to Millions
- **1K req/s**: 5-10 GPU pods, 3 replica services; **10K**: 50-100 GPU pods, multi-region; **100K**: 500+ pods, cluster per region; Cluster Autoscaler with 5-min cooldown to avoid flapping; Preemptible/spot GPU instances for cost savings (with PDB); Pod priority classes for critical inference vs batch jobs.

## 19. Failure Scenarios
- **GPU node failure**: Pod rescheduled to another GPU node (cluster autoscaler adds replacement); **etcd quorum loss**: Cluster becomes read-only — run 5 etcd nodes; **Node pool full**: Cluster Autoscaler can't scale — set max limits wisely; **CNI issues**: Pods stuck in ContainerCreating — check network plugins; **Image pull backoff**: Registry rate limits — use image pull secrets + mirror.

## 20. Security
- RBAC for API access; NetworkPolicies for pod isolation; PodSecurityAdmission (restricted profile); Secrets in etcd with encryption at rest; ServiceAccounts with least privilege; Image scanning in CI/CD; Pod resource limits prevent DoS.

## 21. Monitoring
- **kubectl top nodes/pods** for resource usage; **Prometheus**: node_exporter, kube-state-metrics, GPU metrics (DCGM); **Grafana dashboards**: cluster overview, per-pod resource, HPA status; **Alerting**: Pod crash loop, pending pods, GPU memory pressure, node NotReady; **Metrics**: API server latency, etcd leader changes, scheduler queue depth.

## 22. Interview Questions
1. *How does Kubernetes schedule GPU pods?* — nodeSelector for GPU label + tolerations for nvidia.com/gpu taint + resource limits.
2. *How to handle model updates without downtime?* — RollingUpdate strategy, canary deployment with Istio traffic splitting.
3. *How to scale inference during traffic spikes?* — HPA on GPU utilization/custom metric, Cluster Autoscaler for nodes.
4. *PodDisruptionBudget for inference.* — Ensure minAvailable=2 for 3 replicas during node maintenance.

## 23. Cheat Sheet
```bash
# GPU pod scheduling
kubectl apply -f deployment.yaml
kubectl get pods -l app=llm
kubectl logs -l app=llm --tail=100
kubectl describe pod llm-inference-xxx

# Scale
kubectl scale deployment llm-inference --replicas=5
kubectl get hpa

# Debug
kubectl exec -it pod-name -- nvidia-smi
kubectl top pods
kubectl describe node gpu-node
```

## 24. Common Mistakes
- No resource limits on GPU pods (one pod starves others); Running etcd on small disks (high I/O wears SSDs); Overly large cluster autoscaler cooldown; Not setting PodDisruptionBudget; Mixing GPU and CPU pods on same node; Not using nodeAffinity for GPU scheduling; Pulling model weights on every pod start (slow).

## 25. Production Best Practices
- Taint GPU nodes for exclusive GPU workloads; Pre-pull model images on nodes (DaemonSet init); Use pod priority classes (inference > batch); Set PodDisruptionBudget for inference (minAvailable=2); Enable VerticalPodAutoscaler for CPU workloads; Use Helm for versioned deployments; Run 3+ etcd nodes on dedicated instances; Schedule backups of etcd; Test GPU node autoscaling with load generator; Audit pod resource requests vs actual usage quarterly.
