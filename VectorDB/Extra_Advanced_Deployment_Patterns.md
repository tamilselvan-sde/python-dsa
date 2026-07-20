# Advanced Deployment Patterns

> Author: **Tamilselvan** · ✉️ tamilselvan.sde@gmail.com · 🔗 [LinkedIn](https://www.linkedin.com/in/tamilselvan-ai/)
>


### Kubernetes Helm Chart for Qdrant

```yaml
# values.yaml
replicaCount: 3

image:
  repository: qdrant/qdrant
  tag: v1.8.0

config:
  # Storage config
  storage:
    optimizers:
      default_segment_number: 8
      memmap_threshold_kb: 50000
    wal:
      wal_capacity_mb: 1024
    
  # Performance tuning
  performance:
    max_workers: 8
    max_search_threads: 4
    max_optimization_threads: 2
    
  # gRPC
  service:
    grpc_port: 6334
    http_port: 6333
    max_grpc_message_size: 104857600  # 100MB

resources:
  requests:
    memory: "16Gi"
    cpu: "4"
  limits:
    memory: "32Gi"
    cpu: "8"

persistence:
  size: 500Gi
  storageClass: fast-ssd

service:
  type: ClusterIP
  ports:
    - name: http
      port: 6333
    - name: grpc
      port: 6334

ingress:
  enabled: true
  annotations:
    nginx.ingress.kubernetes.io/backend-protocol: "GRPC"
  hosts:
    - qdrant.example.com
```

### Terraform for Vector DB Infrastructure

```hcl
# main.tf - AWS Infrastructure for Milvus

provider "aws" {
  region = "us-west-2"
}

# VPC
resource "aws_vpc" "vector_db" {
  cidr_block = "10.0.0.0/16"
  enable_dns_hostnames = true
}

# EFS for shared storage
resource "aws_efs_file_system" "milvus_storage" {
  creation_token   = "milvus-storage"
  performance_mode = "maxIO"
  throughput_mode  = "elastic"
  
  tags = {
    Name = "Milvus Storage"
  }
}

# EKS Cluster
resource "aws_eks_cluster" "vector_db" {
  name     = "vector-db-cluster"
  role_arn = aws_iam_role.eks.arn
  version  = "1.28"
  
  vpc_config {
    subnet_ids = aws_subnet.private[*].id
  }
}

# Node group with high-memory instances
resource "aws_eks_node_group" "vector_nodes" {
  cluster_name    = aws_eks_cluster.vector_db.name
  node_group_name = "vector-db-nodes"
  node_role_arn   = aws_iam_role.node.arn
  
  instance_types = ["r6i.8xlarge"]  # 256GB RAM
  disk_size      = 500
  
  scaling_config {
    desired_size = 3
    min_size     = 3
    max_size     = 10
  }
  
  labels = {
    "node-type" = "vector-db"
  }
  
  taint {
    key    = "vector-db"
    value  = "true"
    effect = "NO_SCHEDULE"
  }
}

# ElastiCache for Redis caching
resource "aws_elasticache_cluster" "vector_cache" {
  cluster_id           = "vector-cache"
  engine               = "redis"
  node_type            = "cache.r6g.large"
  num_cache_nodes      = 3
  parameter_group_name = "default.redis7"
  engine_version       = "7.0"
  
  subnet_group_name = aws_elasticache_subnet_group.cache.name
}
```

---

## Detailed Similarity Search Mathematics

### Inner Product vs Cosine: The Full Picture

```
Given vector a and b:

Dot Product:  a · b = Σ(a_i × b_i) = |a||b|cos(θ)

Cosine:       cos(θ) = (a · b) / (|a||b|)

When vectors are L2-normalized (|a| = |b| = 1):
  dot(a, b) = cos(θ)  (identical!)

When vectors are NOT normalized:
  dot(a, b) = |a||b|cos(θ)
  - Same direction: both are positive
  - Different magnitudes: dot product is affected by length
  - cos(θ) ignores magnitude
```

### Distance vs Similarity Conversion

```python
import numpy as np

# Cosine similarity to cosine distance
cosine_distance = 1 - cosine_similarity  # Range: [0, 2]

# Euclidean to similarity
euclidean_similarity = 1 / (1 + euclidean_distance)  # Range: (0, 1]

# Dot product to cosine (for normalized vectors)
cosine_sim = dot_product  # When ||a|| = ||b|| = 1

# Angular distance
angular_distance = np.arccos(cosine_sim) / np.pi  # Range: [0, 1]
```

### Mathematical Properties of Distance Metrics

```python
def is_metric(distance_func, points_triple):
    """Check if distance function satisfies metric properties."""
    a, b, c = points_triple
    d_ab = distance_func(a, b)
    d_bc = distance_func(b, c)
    d_ac = distance_func(a, c)
    
    properties = {
        "non_negative": d_ab >= 0,
        "identity": distance_func(a, a) == 0,
        "symmetric": d_ab == distance_func(b, a),
        "triangle_inequality": d_ac <= d_ab + d_bc,
    }
    return properties

# Euclidean is a true metric (all properties satisfied)
# Cosine similarity is NOT a metric (fails triangle inequality)
# Angular distance IS a metric (fixed cosine)
```

---

## Data Quality Checks for Vector Databases

```python
class VectorDataQualityChecker:
    """Validate vector data quality before indexing."""
    
    def __init__(self, vectors: np.ndarray, metadata: List[Dict]):
        self.vectors = vectors
        self.metadata = metadata
        self.issues = []
    
    def run_all_checks(self) -> Dict:
        self.check_vector_dimensions()
        self.check_for_nan_inf()
        self.check_normalization()
        self.check_zero_vectors()
        self.check_duplicate_vectors()
        self.check_outlier_vectors()
        self.check_metadata_completeness()
        self.check_metadata_types()
        
        return {
            "total_vectors": len(self.vectors),
            "dimension": self.vectors.shape[1] if len(self.vectors) > 0 else 0,
            "issues_found": len(self.issues),
            "issues": self.issues,
            "pass": len(self.issues) == 0,
        }
    
    def check_vector_dimensions(self):
        if len(self.vectors) == 0:
            self.issues.append("CRITICAL: Empty vectors array")
        elif len(self.vectors.shape) != 2:
            self.issues.append(f"CRITICAL: Expected 2D array, got {len(self.vectors.shape)}D")
    
    def check_for_nan_inf(self):
        nan_count = np.isnan(self.vectors).sum()
        inf_count = np.isinf(self.vectors).sum()
        if nan_count > 0:
            self.issues.append(f"CRITICAL: {nan_count} NaN values in vectors")
        if inf_count > 0:
            self.issues.append(f"CRITICAL: {inf_count} Infinity values in vectors")
    
    def check_normalization(self, tolerance=1e-5):
        norms = np.linalg.norm(self.vectors, axis=1)
        unnormalized = np.sum(np.abs(norms - 1.0) > tolerance)
        if unnormalized > 0:
            pct = unnormalized / len(self.vectors) * 100
            self.issues.append(
                f"WARNING: {unnormalized} vectors ({pct:.1f}%) not L2-normalized"
                f" (for cosine search, all vectors should be normalized)"
            )
    
    def check_zero_vectors(self):
        norms = np.linalg.norm(self.vectors, axis=1)
        zero_count = np.sum(norms < 1e-10)
        if zero_count > 0:
            self.issues.append(f"CRITICAL: {zero_count} zero/near-zero vectors")
    
    def check_duplicate_vectors(self, threshold=0.999):
        if len(self.vectors) > 10000:
            # Sample-based check for large datasets
            sample = self.vectors[:1000]
            sim_matrix = sample @ sample.T
            np.fill_diagonal(sim_matrix, 0)
            duplicates = np.sum(sim_matrix > threshold)
        else:
            sim_matrix = self.vectors @ self.vectors.T
            np.fill_diagonal(sim_matrix, 0)
            duplicates = np.sum(sim_matrix > threshold) // 2
        
        if duplicates > 0:
            self.issues.append(
                f"WARNING: ~{duplicates} near-duplicate vector pairs found"
            )
    
    def check_outlier_vectors(self, zscore_threshold=5):
        norms = np.linalg.norm(self.vectors, axis=1)
        z_scores = np.abs((norms - np.mean(norms)) / np.std(norms))
        outliers = np.sum(z_scores > zscore_threshold)
        if outliers > 0:
            pct = outliers / len(self.vectors) * 100
            self.issues.append(
                f"INFO: {outliers} vector outliers ({pct:.2f}%) detected"
            )
    
    def check_metadata_completeness(self):
        if not self.metadata:
            self.issues.append("WARNING: No metadata provided")
            return
        
        required_fields = ["id", "text"]
        for field in required_fields:
            missing = sum(
                1 for m in self.metadata if field not in m
            )
            if missing > 0:
                self.issues.append(
                    f"WARNING: {missing} vectors missing '{field}' in metadata"
                )
    
    def check_metadata_types(self):
        for i, meta in enumerate(self.metadata[:1000]):
            for key, value in meta.items():
                if isinstance(value, np.integer):
                    self.issues.append(
                        f"INFO: Metadata '{key}' uses numpy type, convert to native"
                    )
                    return
```

---

## Common Error Messages & Solutions

| Error | Likely Cause | Solution |
|-------|-------------|----------|
| `Dimension mismatch: expected 768, got 512` | Wrong embedding model | Use same model for all vectors |
| `Out of memory: cannot allocate 3.2 GB` | Dataset too large for RAM | Enable PQ, reduce dimensions, add nodes |
| `Timeout: search exceeded 5000ms` | Index too large or no index | Create HNSW/IVF index, reduce efSearch |
| `ValueError: vector contains NaN` | Bad embedding generation | Check model, validate input text |
| `IndexError: index 0 is out of bounds` | Empty collection | Check collection exists, verify data loaded |
| `413 Request Entity Too Large` | Batch too large | Reduce batch size (try 1000) |
| `429 Too Many Requests` | Rate limit exceeded | Implement backoff, increase capacity |
| `Connection refused` | DB not running | Check container/process status |
| `No connection available` | Connection pool exhausted | Increase pool size, add replicas |
| `WAL exceeds max size` | Checkpoint too slow | Increase checkpoint interval, reduce write rate |
| `Segment too large` | Merge policy issue | Adjust segment configuration |
| `Leader not available` | Cluster split | Check network, etcd/Raft consensus |
| `Index build failed: data contains infinity` | Bad input | Validate vectors before indexing |
| `query_vector argument must be...` | Wrong type | Convert to list/numpy array correctly |

---

