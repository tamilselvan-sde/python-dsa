# Advanced Index Building: FAISS GPU

> Author: **Tamilselvan** · ✉️ tamilselvan.sde@gmail.com · 🔗 [LinkedIn](https://www.linkedin.com/in/tamilselvan-ai/)
>


```python
import faiss
import numpy as np

def build_gpu_index(vectors, dim=768):
    """Build FAISS index on GPU for maximum performance."""
    # Configure GPU resources
    res = faiss.StandardGpuResources()
    res.setDefaultNullStreamAllDevices()
    
    # Build IVF_PQ index on GPU
    config = faiss.GpuIndexIVFPQConfig()
    config.device = 0
    config.useFloat16CoarseQuantizer = False
    config.usePrecomputed = False
    
    # Create GPU index
    index = faiss.GpuIndexIVFPQ(
        res, dim, 4096, 96, 8,
        faiss.METRIC_L2,
        config
    )
    
    # Train and add
    index.train(vectors)
    index.add(vectors)
    index.nprobe = 20
    
    return index

# Search on GPU
vectors = np.random.random((1000000, 768)).astype('float32')
query = np.random.random((1, 768)).astype('float32')

gpu_index = build_gpu_index(vectors)
D, I = gpu_index.search(query, k=10)

# Multi-GPU
def build_multi_gpu_index(vectors, dim=768):
    """Build index across all available GPUs."""
    ngpus = faiss.get_num_gpus()
    print(f"Using {ngpus} GPUs")
    
    resources = [faiss.StandardGpuResources() for _ in range(ngpus)]
    for res in resources:
        res.setDefaultNullStreamAllDevices()
    
    # Build CPU index
    cpu_index = faiss.IndexIVFPQ(
        faiss.IndexFlatL2(dim), dim, 4096, 96, 8
    )
    cpu_index.train(vectors)
    
    # Distribute across GPUs
    gpu_index = faiss.index_cpu_to_gpu_multiple(
        resources, cpu_index
    )
    
    return gpu_index
```

---

## Milvus Production Configuration

### Milvus Cluster Architecture (Kubernetes)

```yaml
# milvus-values.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: milvus-config
data:
  milvus.yaml: |
    # Log configuration
    log:
      level: info
      file:
        maxSize: 500
        maxAge: 7
    
    # Proxy config
    proxy:
      port: 19530
      http:
        enabled: true
      maxFieldNum: 64
      maxVectorFieldNum: 4
    
    # Root coordinator
    rootCoord:
      port: 53100
    
    # Data coordinator
    dataCoord:
      segment:
        maxSize: 512  # MB
        sealInterval: 3600  # seconds
    
    # Query node
    queryNode:
      cache:
        enabled: true
        maxMemory: 16  # GB
    
    # Index node
    indexNode:
      enableDisk: true
      buildParallel: 2
    
    # Storage
    minio:
      address: minio:9000
      bucketName: milvus-bucket
    
    # Channels
    msgChannel:
      chanNamePrefix:
        cluster: milvus-cluster
```

### Milvus Index Configuration

```python
from pymilvus import Collection, connections

# Production index parameters
index_params = {
    "metric_type": "IP",  # Inner product (for normalized vectors)
    "index_type": "IVF_PQ",
    "params": {
        "nlist": 4096,           # Clusters for training
        "m": 64,                 # Sub-quantizers
        "nbits": 8,              # Bits per code
    }
}

# Search parameters for production
search_params = {
    "metric_type": "IP",
    "params": {
        "nprobe": 20,            # Clusters to search
        "offset": 0,             # Pagination
    }
}

# Create with optimized config
collection = Collection("production_docs")

# Index building
collection.create_index(
    field_name="vector",
    index_params=index_params,
    index_name="ivf_pq_64",
)

# Load with replica for high availability
collection.load(replica_number=2)

# Query with timeout
results = collection.search(
    data=[query_vector],
    anns_field="vector",
    param=search_params,
    limit=100,
    timeout=10,  # seconds
    partition_names=["2024"],  # Only search specific partition
    output_fields=["id", "text", "score"]
)
```

---

## Pinecone Production Patterns

### Namespace Strategy

```python
import pinecone

class PineconeMultiTenant:
    def __init__(self, index_name):
        pinecone.init(api_key=os.environ["PINECONE_API_KEY"])
        self.index = pinecone.Index(index_name)
    
    def upsert_tenant_data(self, tenant_id, vectors, metadata):
        """Store vectors in tenant-specific namespace."""
        self.index.upsert(
            vectors=[
                (v_id, vec, meta)
                for v_id, vec, meta in zip(
                    metadata["ids"], vectors, metadata["payloads"]
                )
            ],
            namespace=f"tenant_{tenant_id}"
        )
    
    def search_tenant(self, tenant_id, query_vec, k=10):
        """Search only within tenant's namespace."""
        return self.index.query(
            vector=query_vec,
            top_k=k,
            namespace=f"tenant_{tenant_id}",
            include_metadata=True
        )
    
    def delete_tenant(self, tenant_id):
        """Completely delete tenant's data."""
        self.index.delete(
            delete_all=True,
            namespace=f"tenant_{tenant_id}"
        )
```

### Pod Configuration

```python
# Create optimized pod
pinecone.create_index(
    name="production-index",
    dimension=768,
    metric="cosine",
    pod_type="p2.x2",
    pods=5,
    replicas=2,
    shards=5,
    metadata_config={
        "indexed": ["tenant_id", "date", "category"]
    }
)
# p2.x2: 2x memory of p2.x1
# pods=5: 5 pods for throughput
# replicas=2: HA across pods
# shards=5: distribute data
```

---

## Weaviate Production Configurations

### Schema with Vectorization Modules

```python
import weaviate

client = weaviate.Client("http://localhost:8080")

# Schema with multi-vector and modules
class_schema = {
    "class": "Document",
    "description": "Production document with multimodal search",
    "vectorizer": "text2vec-transformers",
    "moduleConfig": {
        "text2vec-transformers": {
            "model": "sentence-transformers/all-MiniLM-L6-v2",
            "poolingStrategy": "masked_mean",
            "inferenceUrl": "http://t2v-transformers:8080"
        },
        "generative-openai": {
            "model": "gpt-4"
        }
    },
    "properties": [
        {
            "name": "title",
            "dataType": ["text"],
            "description": "Document title"
        },
        {
            "name": "content",
            "dataType": ["text"],
            "description": "Document content",
            "moduleConfig": {
                "text2vec-transformers": {
                    "skip": False,
                    "vectorizePropertyName": False
                }
            }
        },
        {
            "name": "author",
            "dataType": ["string"],
            "description": "Author name"
        },
        {
            "name": "publishedDate",
            "dataType": ["date"]
        },
        {
            "name": "tags",
            "dataType": ["text[]"]
        },
        {
            "name": "tenantId",
            "dataType": ["string"],
            "indexFilterable": True,
            "indexSearchable": False
        }
    ],
    "vectorIndexConfig": {
        "distance": "cosine",
        "ef": 200,
        "efConstruction": 400,
        "maxConnections": 32,
        "dynamicEfFactor": 10,
        "skip": False,
        "cleanupIntervalSeconds": 300
    },
    "multiTenancyConfig": {"enabled": True}
}

client.schema.create_class(class_schema)

# Insert with multi-tenancy
client.data_object.create(
    data_object={
        "title": "Vector DB Guide",
        "content": "Comprehensive guide...",
        "author": "John Doe",
        "tenantId": "acme_corp"
    },
    class_name="Document",
    tenant="acme_corp"
)
```

---

