# Docker for AI Systems

## 1. What is it?
Docker containers for AI package applications with all dependencies — Python version, CUDA libraries, model weights, system packages — into portable images that run identically across dev, staging, and production environments.

## 2. Why do we need it?
AI applications have complex dependency chains (PyTorch, CUDA, cuDNN, Triton, transformers) that conflict across projects. "Works on my machine" is the #1 AI deployment failure. Docker guarantees reproducible environments and enables GPU passthrough for model inference.

## 3. Real-world Example
**Netflix's ML Platform** runs 1000+ models in Docker containers on Kubernetes. Each model image bundles its exact PyTorch version, CUDA runtime, preprocessing code, and model weights. A new model just needs a Dockerfile — the deployment pipeline is identical for every model.

## 4. Architecture Diagram (ASCII)
```
+--------------------------------------------------+
|              Docker Container                      |
|  +----------------------------------------------+ |
|  |  Application (FastAPI / vLLM)                 | |
|  +----------------------------------------------+ |
|  |  Python 3.12 + Dependencies (pip packages)    | |
|  +----------------------------------------------+ |
|  |  CUDA 12.1 + cuDNN 8.9 + TensorRT            | |
|  +----------------------------------------------+ |
|  |  NVIDIA Container Toolkit (nvidia-docker)     | |
|  +----------------------------------------------+ |
|  |  Ubuntu 22.04 Base Image                      | |
|  +----------------------------------------------+ |
+--------------------------------------------------+
           |                          |
    +------v------+          +-------v-------+
    |  GPU (NVIDIA)|          |  CPU (Infer)  |
    |  (Training)  |          |  (Small models)|
    +-------------+          +---------------+
```

## 5. Internal Working
Docker uses layered images: base OS layer, CUDA layer (from nvidia/cuda image), Python layer, dependencies layer, application layer. Each layer is cached, so rebuilding after code changes is fast. NVIDIA Container Toolkit mounts GPU devices and CUDA libraries into containers.

## 6. Production Flow
```
Developer code -> Dockerfile -> Build -> Container Registry (ECR)
-> Kubernetes pulls image -> Container runs on GPU node -> Serving
```

## 7. HLD
| Component | Image | Base | Size |
|-----------|-------|------|------|
| LLM Inference | llm-inference:1.2 | nvidia/cuda:12.1-runtime | 4.2GB |
| Embedding | embedding:1.0 | python:3.12-slim | 350MB |
| Retrieval | retrieval:2.1 | python:3.12-slim | 280MB |
| Orchestrator | orchestrator:1.5 | python:3.12-slim | 250MB |
| Data Pipeline | pipeline:0.9 | python:3.12-slim | 400MB |

## 8. LLD

```dockerfile
# Multi-stage build for production LLM serving
FROM python:3.12-slim AS builder
WORKDIR /app
COPY requirements.txt .
RUN pip install --user --no-cache-dir -r requirements.txt

FROM nvidia/cuda:12.1.0-runtime-ubuntu22.04 AS runtime
WORKDIR /app
RUN apt-get update && apt-get install -y --no-install-recommends \
    libgomp1 && rm -rf /var/lib/apt/lists/*
COPY --from=builder /root/.local /root/.local
COPY src/ ./src/
COPY models/ ./models/
ENV PATH=/root/.local/bin:$PATH
ENV CUDA_VISIBLE_DEVICES=0
EXPOSE 8000
CMD ["uvicorn", "src.main:app", "--host", "0.0.0.0", "--port", "8000"]
```

## 9. Python Implementation
```python
# docker-compose.yml for local AI stack
version: "3.9"
services:
  qdrant:
    image: qdrant/qdrant:v1.10
    volumes:
      - ./qdrant_storage:/qdrant/storage
    ports:
      - "6333:6333"
    environment:
      - QDRANT__SERVICE__GRPC_PORT=6334

  embedding:
    build:
      context: ./embedding
      dockerfile: Dockerfile
    deploy:
      resources:
        reservations:
          devices:
            - driver: nvidia
              count: 1
              capabilities: [gpu]

  orchestrator:
    build: ./orchestrator
    depends_on:
      - qdrant
      - embedding
    environment:
      - OPENAI_API_KEY=${OPENAI_API_KEY}
    ports:
      - "8000:8000"
```

## 10. Folder Structure
```
ai-project/
  Dockerfile                  # Multi-stage build
  docker-compose.yml          # Local development
  .dockerignore
  requirements.txt
  src/
  tests/
  deploy/
    docker-stack.yml          # Docker Swarm
    Dockerfile.gpu            # GPU-specific variant
    entrypoint.sh
```

## 11. Configuration
```yaml
# docker-compose.prod.yml
version: "3.9"
services:
  llm:
    image: ${ECR_REPO}/llm:${TAG}
    runtime: nvidia
    environment:
      - CUDA_VISIBLE_DEVICES=0,1
      - MODEL_NAME=${MODEL}
      - MAX_MODEL_LEN=4096
    volumes:
      - model-cache:/root/.cache/huggingface
    deploy:
      replicas: 3
      resources:
        reservations:
          devices:
            - capabilities: [gpu]
              count: 2

volumes:
  model-cache:
```

## 12. Flowchart
```
Write Dockerfile -> docker build -> Image in local registry
-> docker push -> Container Registry (ECR/Docker Hub)
-> Production pulls image -> Container starts -> Health check
-> Traffic routed -> Serving inference
```

## 13. Sequence Diagram
```
Developer    Docker     Registry    Production Node
  |            |           |              |
  |--build---->|           |              |
  |            |--layer cache check       |
  |            |--build image             |
  |<--image----|           |              |
  |            |           |              |
  |--push----------------->|              |
  |            |           |              |
  |            |           |--pull-------->|
  |            |           |              |--run container
  |            |           |              |--health check
  |            |           |              |--serve traffic
```

## 14. Pros
- Reproducible environments; GPU passthrough; Layer caching for fast builds; Immutable deployments (no config drift); Consistent dev/prod parity; Rich ecosystem of base images.

## 15. Cons
- Large image sizes (CUDA images are 3-6GB); Build time for AI images can be 10+ mins; CUDA version conflicts with host driver; Secrets management in images; Storage costs for many model images.

## 16. Alternatives
- **Podman**: Daemonless, rootless containers; **Singularity**: HPC-focused, better for scientific computing; **Bare metal**: No container overhead, but no isolation; **AWS Lambda**: Serverless, limited to 10GB/15min.

## 17. Performance Considerations
- Multi-stage builds reduce image size 50-70%; CUDA images are 3-6GB — use slim variants; Model weights should be mounted, not baked in; Layer caching — put stable layers (CUDA, pip) first; `.dockerignore` prevents sending 10GB model cache to build context; Use `--gpus all` sparingly — specify device count.

## 18. Scaling to Millions
- **1-10 containers**: Docker Compose; **10-100**: Docker Swarm or single-host; **100+**: Kubernetes; Container registry with geo-replication for global pull speed; Image pull is the bottleneck in auto-scaling — pre-pull on nodes.

## 19. Failure Scenarios
- **CUDA version mismatch**: Host driver must support container CUDA; **Container OOM**: Set `--memory` limits; **Image pull failure**: Always use image digest for production; **Disk full**: Cleanup unused images/images; **Container crash**: Docker restart policy + health checks.

## 20. Security
- Never bake secrets in images (use env vars or secrets mount); Scan images with Trivy/Snyk; Use staging-specific base images (pin digests); Run containers as non-root user; Read-only root filesystem; Drop all capabilities.

## 21. Monitoring
- `docker stats` for CPU/memory/GPU; Container restart count; Image pull latency; Disk usage for images and volumes; Docker daemon health; Container startup time.

## 22. Interview Questions
1. *Multi-stage builds for AI applications.* — Builder stage installs pip packages, runtime stage copies only artifacts, reducing size.
2. *How to pass GPU to Docker containers?* — NVIDIA Container Toolkit, `--gpus` flag, `nvidia/cuda` base image.
3. *Why is Docker important for AI deployments?* — Reproducible environments with exact CUDA/Python/model versions.
4. *How to handle large model files in Docker?* — Mount as volumes, don't bake into image; pre-download to shared cache.

## 23. Cheat Sheet
```dockerfile
# Multi-stage GPU inference image
FROM python:3.12-slim AS builder
COPY requirements.txt .
RUN pip install --user -r requirements.txt

FROM nvidia/cuda:12.1-runtime
COPY --from=builder /root/.local /root/.local
COPY src/ src/
CMD ["uvicorn", "src.app:app", "--host", "0.0.0.0"]
```

## 24. Common Mistakes
- Baking model weights into image (use volumes); Not using `.dockerignore` (sends GB to build context); Running as root in container; No health check; Single-stage images (1GB+ overhead); Pinning pip packages without hashes; Not setting memory limits.

## 25. Production Best Practices
- Use multi-stage builds; Pin base image digests; Mount model weights from shared volume; Set memory and GPU limits; Implement health checks; Use non-root user; Scan images for vulnerabilities; Tag images with git SHA; Pre-pull images on nodes; Clean up unused images.
