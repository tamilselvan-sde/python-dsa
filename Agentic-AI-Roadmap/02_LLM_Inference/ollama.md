# Ollama Local LLM Runner

> Author: **Tamilselvan** · ✉️ tamilselvan.sde@gmail.com · 🔗 [LinkedIn](https://www.linkedin.com/in/tamilselvan-ai/)
>

## 1. What is it?

Ollama is a user-friendly, open-source tool for running large language models locally on consumer hardware. It wraps model weights, configuration, and dependencies into single portable packages called Modelfiles.

**ELI5:** Ollama is like Docker for LLMs. Just as Docker lets you run containerized applications with `docker run`, Ollama lets you run language models with `ollama run llama3`. It handles model downloading, GPU acceleration, and API serving with a single command.

**Technical Definition:** Ollama is a Go-based local inference server that manages model distribution (via OCI-compatible registries), quantization (via llama.cpp backend), GPU offloading (CUDA/Metal/Vulkan), and exposes an OpenAI-compatible REST API on localhost:11434.

## 2. Why do we need it?

**Problem:** Running LLMs locally has been a complex chore — installing CUDA drivers, managing Python dependencies, downloading multi-GB weights, configuring quantization, and setting up API servers. This friction prevents developers from experimenting with local AI.

**Pain without it:** Without Ollama, a developer wanting to test Llama 3 locally needs to: install Python 3.10+, set up a venv, install PyTorch with CUDA, clone transformers, download weights from HuggingFace, write a tokenizer wrapper, and build a FastAPI server. This takes 30-60 minutes and often breaks.

**Why companies use it:** While not typically used for production serving at scale, teams use Ollama for: local development and testing of prompts before deployment, offline/air-gapped environments, prototyping agentic workflows, and CI/CD pipelines that need model inference without cloud costs.

## 3. Real-world Example

- **JetBrains AI Assistant**: Uses Ollama as an optional local backend for code completion in IDEs.
- **GitHub** (Copilot alternatives): Ollama recommended for local code model testing before using Copilot API.
- **Apple**: Metal GPU acceleration for Ollama is showcased as a macOS ML development workflow.
- **LangChain/LlamaIndex**: Ollama is the default local provider for development and testing.
- **VSCode extensions**: Continue.dev, Cody, and other AI coding assistants support Ollama as a local backend.
- **Enterprise air-gapped deployments**: Banks and defense contractors use Ollama for fully offline LLM inference on classified data.

## 4. Architecture Diagram (ASCII)

```
┌───────────────────────────────────────────────────────┐
│                      Ollama CLI                         │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌────────┐ │
│  │  ollama  │  │  ollama  │  │  ollama  │  │  ollama│ │
│  │  run     │  │  pull    │  │  create  │  │  ps    │ │
│  └────┬─────┘  └────┬─────┘  └────┬─────┘  └───┬────┘ │
└───────┼──────────────┼─────────────┼────────────┼──────┘
        │              │             │            │
        ▼              ▼             ▼            ▼
┌───────────────────────────────────────────────────────┐
│                   Ollama Server (Go)                   │
│         Port 11434 — REST API + gRPC internal          │
│                                                         │
│  ┌────────────┐  ┌────────────┐  ┌───────────────┐    │
│  │ Model      │  │ Request    │  │ Scheduler     │    │
│  │ Registry   │  │ Router     │  │ (Single Queue)│    │
│  └─────┬──────┘  └─────┬──────┘  └──────┬────────┘    │
│        │               │                │              │
└────────┼───────────────┼────────────────┼──────────────┘
         │               │                │
         ▼               ▼                ▼
┌───────────────────────────────────────────────────────┐
│                    llama.cpp Backend                    │
│                                                         │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐ │
│  │ GGUF Model   │  │ GPU Offload  │  │ KV Cache     │ │
│  │ Loader       │  │ (CUDA/Metal) │  │ Manager      │ │
│  └──────────────┘  └──────────────┘  └──────────────┘ │
│                                                         │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐ │
│  │ Tokenizer    │  │ Sampler      │  │ Grammar      │ │
│  │ (SentencePiece) │ (temp, top_p) │  │ (GBNF)       │ │
│  └──────────────┘  └──────────────┘  └──────────────┘ │
└───────────────────────────────────────────────────────┘
                        │
                        ▼
┌───────────────────────────────────────────────────────┐
│                    Hardware Layer                       │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌────────┐ │
│  │  CPU     │  │  GPU     │  │  RAM     │  │ Disk   │ │
│  │ (AVX2)  │  │ (CUDA)   │  │          │  │ (SSD)  │ │
│  └──────────┘  └──────────┘  └──────────┘  └────────┘ │
└───────────────────────────────────────────────────────┘
```

## 5. Internal Working

Ollama has three major subsystems:

**1. Model Registry & Distribution:**
- Models are stored as OCI-compatible layers (like Docker images) in a registry
- Each model is a manifest + layers (GGUF weights, tokenizer config, Modelfile)
- `ollama pull` downloads layers, verifies SHA256 checksums, and extracts to `~/.ollama/models/`
- Versioned tags (`llama3:70b`, `llama3:8b-instruct-fp16`)

**2. llama.cpp Backend Integration:**
- Ollama embeds llama.cpp via CGo bindings
- The GGUF file format contains: model weights (quantized), hyperparameters, tokenizer, and metadata
- GPU offloading: layers are progressively offloaded to GPU. `num_gpu` parameter controls how many layers run on GPU
- Fallback to CPU when GPU memory is insufficient

**3. Inference Server:**
- Go HTTP/gRPC server on port 11434
- Manages concurrent requests via a single queue (ollama doesn't support continuous batching in the traditional sense — processes one request at a time)
- Supports streaming responses via Server-Sent Events (SSE)
- OpenAI-compatible `/v1/chat/completions` and `/v1/embeddings` endpoints

## 6. Production Flow

```
ollama pull llama3.2:3b
     │
     ▼
┌──────────────┐
│ Download     │
│ GGUF layers  │
│ from registry│
└──────┬───────┘
       │
       ▼
┌──────────────┐
│ Verify SHA256│
│ Extract to   │
│ ~/.ollama    │
└──────┬───────┘
       │
       ▼
┌──────────────┐
│ ollama run   │
│ llama3.2:3b  │
│ (or API call)│
└──────┬───────┘
       │
       ▼
┌──────────────┐
│ Load GGUF    │
│ → CPU memory │
│ ↓ offload    │
│ layers→GPU   │
└──────┬───────┘
       │
       ▼
┌──────────────┐
│ Tokenize →   │
│ Prefill →    │
│ Decode →     │
│ Detokenize   │
│ → Stream out │
└──────────────┘
```

## 7. HLD (High-Level Design)

```
┌────────────────────┐
│    User/Client     │
│  (CLI, curl, app)  │
└────────┬───────────┘
         │
         ▼
┌────────────────────┐
│   Ollama CLI       │
│  (Go binary)       │
└────────┬───────────┘
         │
         ▼
┌────────────────────┐
│   Ollama Server    │
│  localhost:11434   │
│  REST + gRPC       │
└────────┬───────────┘
         │
         ▼
┌────────────────────┐
│  llama.cpp Runner  │
│  (CGo process)     │
└────────┬───────────┘
         │
    ┌────┴────┐
    │  CPU    │
    │  + GPU  │
    └─────────┘
```

## 8. LLD (Low-Level Design)

**Modelfile format and parsing:**
```python
# A Modelfile is analogous to Dockerfile for LLMs
# Built-in: ollama create mymodel -f ./Modelfile

FROM llama3.2:3b
PARAMETER temperature 0.7
PARAMETER top_p 0.95
PARAMETER num_ctx 4096
SYSTEM You are a math tutor. Be concise and show steps.
TEMPLATE """
{{- if .System }}<|start_header_id|>system<|end_header_id|>
{{ .System }}<|eot_id|>{{ end }}
<|start_header_id|>user<|end_header_id|>
{{ .Prompt }}<|eot_id|>
<|start_header_id|>assistant<|end_header_id|>
"""
```

**Ollama Go server structure (simplified):**
```go
type Server struct {
    modelRegistry map[string]*Model
    runner       *llama.Runner
    mu           sync.Mutex
}

type Model struct {
    Name       string
    Path       string
    Digest     string
    Modelfile  *Modelfile
    Template   string
    Parameters map[string]interface{}
}

type GenerateRequest struct {
    Model    string  `json:"model"`
    Prompt   string  `json:"prompt"`
    Stream   bool    `json:"stream"`
    Options  Options `json:"options"`
}

type Options struct {
    Temperature float32 `json:"temperature"`
    TopP        float32 `json:"top_p"`
    NumCtx      int     `json:"num_ctx"`
    NumGPU      int     `json:"num_gpu"`
    NumThread   int     `json:"num_thread"`
}
```

## 9. Python Implementation

```python
import httpx
import asyncio
from fastapi import FastAPI, HTTPException
from pydantic import BaseModel, Field
from typing import List, Optional, Dict, Any, AsyncGenerator
import json
import logging

logging.basicConfig(level=logging.INFO)
logger = logging.getLogger("ollama-proxy")

app = FastAPI(title="Ollama Proxy Service")

OLLAMA_HOST = "http://localhost:11434"

class ChatMessage(BaseModel):
    role: str
    content: str

class ChatRequest(BaseModel):
    model: str = Field(default="llama3.2:3b")
    messages: List[ChatMessage]
    temperature: Optional[float] = 0.7
    max_tokens: Optional[int] = 512
    stream: Optional[bool] = False

class OllamaClient:
    def __init__(self, base_url: str = OLLAMA_HOST):
        self.base_url = base_url
        self.client = httpx.AsyncClient(timeout=120.0)

    async def chat(self, request: ChatRequest) -> Dict[str, Any]:
        """Convert OpenAI-style request to Ollama format."""
        # Build prompt from messages
        prompt = self._build_prompt(request.messages)

        payload = {
            "model": request.model,
            "prompt": prompt,
            "stream": False,
            "options": {
                "temperature": request.temperature,
                "num_predict": request.max_tokens,
            },
        }

        response = await self.client.post(
            f"{self.base_url}/api/generate",
            json=payload,
        )
        response.raise_for_status()
        result = response.json()

        return {
            "model": request.model,
            "choices": [{
                "index": 0,
                "message": {
                    "role": "assistant",
                    "content": result.get("response", ""),
                },
                "finish_reason": "stop",
            }],
            "usage": {
                "prompt_tokens": result.get("prompt_eval_count", 0),
                "completion_tokens": result.get("eval_count", 0),
                "total_tokens": (
                    result.get("prompt_eval_count", 0)
                    + result.get("eval_count", 0)
                ),
            },
        }

    async def chat_stream(
        self, request: ChatRequest
    ) -> AsyncGenerator[Dict[str, Any], None]:
        prompt = self._build_prompt(request.messages)
        payload = {
            "model": request.model,
            "prompt": prompt,
            "stream": True,
            "options": {
                "temperature": request.temperature,
                "num_predict": request.max_tokens,
            },
        }

        async with self.client.stream(
            "POST", f"{self.base_url}/api/generate", json=payload
        ) as response:
            async for line in response.aiter_lines():
                if line.strip():
                    chunk = json.loads(line)
                    yield {
                        "choices": [{
                            "index": 0,
                            "delta": {
                                "content": chunk.get("response", ""),
                            },
                            "finish_reason": (
                                "stop" if chunk.get("done") else None
                            ),
                        }]
                    }

    def _build_prompt(self, messages: List[ChatMessage]) -> str:
        prompt = ""
        for msg in messages:
            if msg.role == "system":
                prompt += f"<|system|>\n{msg.content}\n"
            elif msg.role == "user":
                prompt += f"<|user|>\n{msg.content}\n"
            elif msg.role == "assistant":
                prompt += f"<|assistant|>\n{msg.content}\n"
        prompt += "<|assistant|>\n"
        return prompt

client = OllamaClient()

@app.post("/v1/chat/completions")
async def chat_completions(request: ChatRequest):
    try:
        if request.stream:
            return StreamingResponse(
                client.chat_stream(request),
                media_type="text/event-stream",
            )
        result = await client.chat(request)
        return result
    except httpx.RequestError as e:
        logger.error(f"Ollama request failed: {e}")
        raise HTTPException(
            status_code=503,
            detail="Ollama server unreachable. Run 'ollama serve' first.",
        )

@app.get("/v1/models")
async def list_models():
    try:
        response = await client.client.get(f"{OLLAMA_HOST}/api/tags")
        models = response.json().get("models", [])
        return {
            "data": [
                {
                    "id": m["name"],
                    "object": "model",
                    "created": m.get("modified_at", ""),
                }
                for m in models
            ]
        }
    except Exception as e:
        raise HTTPException(status_code=503, detail=str(e))

@app.get("/health")
async def health():
    try:
        await client.client.get(f"{OLLAMA_HOST}/api/tags")
        return {"status": "healthy", "backend": "ollama"}
    except Exception:
        return {"status": "unhealthy"}, 503
```

## 10. Folder Structure

```
~/.ollama/
├── models/
│   ├── blobs/              # GGUF weight files (content-addressed)
│   │   ├── sha256-abc123...
│   │   └── sha256-def456...
│   └── manifests/
│       └── registry.ollama.ai/
│           └── library/
│               ├── llama3.2/
│               │   └── 3b-instruct/
│               │       └── manifest
│               └── mistral/
│                   └── latest/
│                       └── manifest

ollama_project/
├── Dockerfile
├── Modelfile               # Custom model definition
├── serve.py                # Proxy server (above)
├── ollama_app/
│   ├── init.py
│   ├── client.py           # Ollama HTTP client
│   ├── models.py           # Pydantic models
│   ├── modelfile.py        # Modelfile parser
│   └── utils.py
├── docker-compose.yaml     # Ollama + proxy
└── tests/
    ├── test_client.py
    └── test_integration.py
```

## 11. Configuration

**Ollama environment variables:**
```bash
# Server config
OLLAMA_HOST=0.0.0.0        # Bind address
OLLAMA_PORT=11434           # Port
OLLAMA_ORIGINS=*            # CORS origins
OLLAMA_NUM_PARALLEL=1       # Parallel requests (1 = no continuous batching)

# GPU config
OLLAMA_NUM_GPU=999          # Offload all possible layers to GPU
OLLAMA_GPU_OVERHEAD=512     # MB reserved for GPU operations
OLLAMA_LOAD_IN_4BIT=true    # Force 4-bit quantization

# Model config
OLLAMA_KEEP_ALIVE=5m        # How long to keep model in memory after last use
OLLAMA_MAX_LOADED_MODELS=3  # Max models kept in memory simultaneously

# Debug
OLLAMA_DEBUG=0              # Debug logging
OLLAMA_CUDA=0               # Disable CUDA (force CPU)
```

**Docker Compose:**
```yaml
version: '3.8'
services:
  ollama:
    image: ollama/ollama:latest
    ports:
      - "11434:11434"
    volumes:
      - ollama_data:/root/.ollama
    deploy:
      resources:
        reservations:
          devices:
            - driver: nvidia
              count: all
              capabilities: [gpu]
    environment:
      - OLLAMA_NUM_PARALLEL=2
      - OLLAMA_KEEP_ALIVE=10m

  proxy:
    build: .
    ports:
      - "8000:8000"
    environment:
      - OLLAMA_HOST=http://ollama:11434
    depends_on:
      - ollama

volumes:
  ollama_data:
```

## 12. Flowchart

```
                ┌─────────────────┐
                │ ollama run      │
                │ llama3.2:3b     │
                └────────┬────────┘
                         │
                         ▼
                ┌─────────────────┐
                │ Check local     │
                │ ~/.ollama/      │
                └────────┬────────┘
                         │
              ┌──────────┴──────────┐
              │                     │
         ┌────▼────┐          ┌────▼────┐
         │ Found   │          │ Missing │
         └────┬────┘          └────┬────┘
              │                     │
              │              ┌──────▼──────┐
              │              │ ollama pull │
              │              │ from registry│
              │              └──────┬──────┘
              │                     │
              └──────────┬──────────┘
                         │
                         ▼
                ┌─────────────────┐
                │ Load GGUF       │
                │ Map to memory   │
                │ Offload→GPU     │
                └────────┬────────┘
                         │
                         ▼
                ┌─────────────────┐
                │ Process Prompt  │
                │ Prefill →      │
                │ Decode loop    │
                │ (stream tokens)│
                └────────┬────────┘
                         │
                         ▼
                ┌─────────────────┐
                │ Output          │
                │ (print or SSE)  │
                └─────────────────┘
```

## 13. Sequence Diagram

```
User         Ollama CLI         Ollama Server       llama.cpp          GPU
  │              │                    │                  │               │
  │─ollama run──▶│                    │                  │               │
  │              │───POST /api/gen──▶│                  │               │
  │              │                    │───load model───▶│               │
  │              │                    │                  │──allocate──▶│
  │              │                    │                  │◀─────ok─────│
  │              │                    │◀────ready────────│               │
  │              │                    │                  │               │
  │              │                    │───tokenize──────▶│               │
  │              │                    │◀───tokens────────│               │
  │              │                    │                  │               │
  │              │                    │───prefill───────▶│───────eval──▶│
  │              │                    │◀───logits────────│◀─────output─│
  │              │                    │                  │               │
  │              │◀───token──────────│                  │               │
  │◀─stream─────│                    │                  │               │
  │              │                    │───decode────────▶│───────eval──▶│
  │              │                    │◀───token────────│◀─────output─│
  │              │◀───token──────────│                  │               │
  │◀─stream─────│                    │                  │               │
  │    (repeat until done)           │                  │               │
  │              │                    │                  │               │
  │              │◀───done───────────│                  │               │
  │◀─complete───│                    │                  │               │
```

## 14. Pros

- **Zero configuration** — `ollama run llama3.2` works immediately
- **Cross-platform** — macOS (Metal), Windows (CUDA), Linux (CUDA/Vulkan)
- **Model packaging** — Modelfiles make model distribution reproducible
- **OpenAI-compatible API** — works with existing tools and libraries
- **Quantization** — supports Q4_0, Q4_K_M, Q5_K_M, Q8_0 GGUF formats
- **Offline capable** — fully air-gapped operation after model download
- **Active community** — 100K+ GitHub stars, massive model library
- **Lightweight** — Go binary with no Python dependency

## 15. Cons

- **No continuous batching** — processes one request at a time (OLLAMA_NUM_PARALLEL is experimental)
- **Single-node only** — no distributed inference across machines
- **Limited to llama.cpp models** — only GGUF format; no direct HuggingFace model support
- **CPU inference is slow** — 7B model at Q4: 15-25 tok/s on CPU, 50-80 tok/s on M1 Max, 200+ on H100
- **No production serving features** — no load balancing, auto-scaling, or metrics by default
- **No PagedAttention** — KV cache is contiguous and can fragment
- **Limited scheduling** — FIFO queue, no priority or preemption

## 16. Alternatives

| Alternative | When to Use | Limitations |
|------------|-------------|-------------|
| **llama.cpp** | Direct C++ integration, scripting | No server/API included |
| **LM Studio** | GUI-based model exploration | GUI-bound, harder to script |
| **GPT4All** | CPU-optimized local models | Smaller model ecosystem |
| **LocalAI** | Drop-in OpenAI API replacement | Less optimized backend |
| **vLLM** | Production serving | Heavy, requires large GPU |
| **Oogabooga** | Web UI for multiple backends | Complex setup |

## 17. Performance Considerations

**Hardware requirements per model size (Q4_K_M quantization):**
```
Model           RAM Required  Tokens/sec (CPU)  Tokens/sec (GPU)
Mistral 7B      6-8 GB        15-25              80-120 (RTX 4090)
Llama 3 8B      6-8 GB        12-20              70-100 (RTX 4090)
Llama 3 70B     40-50 GB      2-4               200-300 (2x H100)
CodeLlama 34B   20-25 GB      4-8               60-90 (RTX 4090)
```

**Optimization tips:**
- **Layer offloading**: Use `num_gpu` to control how many layers go to GPU. With limited VRAM, offload only the first N layers (attention layers benefit most from GPU).
- **Context length**: `num_ctx=2048` uses less memory than `num_ctx=8192`. Reduce context to minimum needed.
- **Batch size**: `num_batch=512` is Ollama's default. Smaller (128) reduces latency; larger (1024) increases throughput for batch prefill.
- **Thread count**: `num_thread` should match physical CPU cores (not hyperthreads). Too many threads causes cache thrashing.

## 18. Scaling to Millions

Ollama is intentionally not designed for massive production scale. Strategies for higher throughput:

1. **Multiple Ollama instances**: Run one Ollama per GPU, use an nginx load balancer to distribute requests
2. **Model parallelism**: Not supported natively. Use vLLM for multi-GPU setups.
3. **Caching**: Implement a response cache for common queries (Redis with TTL)
4. **Queue worker pattern**: Use Celery/RQ to queue requests and process one at a time
5. **Hybrid approach**: Ollama for dev/QA, vLLM for production

**Maximum realistic throughput:**
- Single Ollama instance (1 GPU): ~50-100 concurrent requests/hour (serialized)
- 10x Ollama instances (10 GPUs): ~500-1000 requests/hour
- Compare with vLLM: 10,000+ requests/hour on single GPU

## 19. Failure Scenarios

| Failure | Impact | Mitigation |
|---------|--------|------------|
| **GPU OOM** | Server crash, model unloaded | Reduce num_gpu, use smaller quantization |
| **Model download failure** | Model not found locally | Retry with backoff, validate checksums |
| **Memory leak** | Gradual slowdown over hours | The `keep_alive` timeout unloads idle models |
| **Corrupted GGUF** | Garbled output, crashes | `ollama rm` and re-pull, verify SHA256 |
| **Port conflict** | Server won't start | Change OLLAMA_PORT, kill conflicting processes |
| **CUDA driver mismatch** | GPU offloading fails | Fall back to CPU, log warning |

## 20. Security

| Concern | Mitigation |
|---------|------------|
| **Model provenance** | SHA256-verified downloads from registry |
| **Local API exposure** | Binds to localhost:11434 by default |
| **No auth** | None built-in. Use proxy with API key validation |
| **Prompt injection** | Same as any LLM — content filtering per application |
| **Data exfiltration** | Fully offline capable; no telemetry for inference data |
| **Model tampering** | Verify checksums; use trusted registries |

**Production hardening:**
```python
# Add API key authentication to Ollama proxy
from fastapi import Header, HTTPException

API_KEY = os.environ.get("OLLAMA_PROXY_KEY", "")

async def verify_auth(authorization: str = Header(None)):
    if API_KEY:
        if not authorization or authorization != f"Bearer {API_KEY}":
            raise HTTPException(status_code=401, detail="Invalid API key")
```

## 21. Monitoring

**Ollama exposes metrics at `localhost:11434/api/ps`:**
```json
{
  "models": [
    {
      "name": "llama3.2:3b",
      "model": "...",
      "size": "2.0 GB",
      "digest": "sha256:...",
      "details": {
        "format": "gguf",
        "family": "llama",
        "parameter_size": "3.2B",
        "quantization": "Q4_K_M"
      },
      "expires_at": "2024-01-01T00:00:00Z",
      "size_vram": "2.0 GB"
    }
  ]
}
```

**Custom monitoring via proxy:**
```
# Log metrics to Prometheus
ollama:request_duration_ms      — Request latency histogram
ollama:prompt_tokens            — Input token count per request
ollama:completion_tokens        — Output token count per request
ollama:tokens_per_second        — Generation speed
ollama:model_load_time_ms       — Time to load model into memory
ollama:memory_usage_bytes       — RAM/VRAM consumption
```

**Alerts:**
- Model load time > 30s → disk I/O issues or corrupted weights
- Tokens per second drop > 50% → CPU throttling or thermal limit
- Out of memory errors → reduce context length or quantization level

## 22. Interview Questions

**Q1: How does Ollama differ from directly using llama.cpp?**
*A: Ollama is a full distribution system built on top of llama.cpp. It adds: model registry with OCI-compatible storage, REST API with OpenAI compatibility, Go-based server for concurrent request handling, and Modelfiles for reproducible model packaging. llama.cpp is just the C++ inference library.*

**Q2: Why doesn't Ollama support continuous batching?**
*A: Continuous batching requires the inference engine to dynamically add/remove sequences between decoder iterations, which demands PagedAttention-style KV cache management. Ollama uses llama.cpp, which allocates contiguous KV cache per sequence. True continuous batching would require a fundamental rewrite of the memory management. Ollama's single-request-at-a-time design prioritizes simplicity over throughput.*

**Q3: When would you choose Ollama over vLLM for a project?**
*A: Choose Ollama for: local development and testing, air-gapped environments, low-concurrency workloads (<10 simultaneous users), rapid prototyping, and when you want a single binary with no Python dependency. Choose vLLM for: production serving with 100+ concurrent users, multi-GPU inference, PagedAttention benefits, and OpenAI API compatibility at scale.*

**Q4: Explain GGUF quantization levels and how to choose between them.**
*A: GGUF levels: Q2 (2-bit, 2.5GB for 7B), Q3 (3-bit, 3.5GB), Q4_0 (4-bit, 4GB), Q4_K_M (4-bit with key-value quantization, balanced), Q5_K_M (5-bit, 5GB), Q8_0 (8-bit, 7.5GB), F16 (16-bit, 14GB). Choose Q4_K_M as default — best quality-size tradeoff. Use Q8_0 for maximum quality. Use Q3 if VRAM constrained. The "K_M" variants use importance-based quantization that preserves more precision for important weights.*

## 23. Cheat Sheet

```bash
# Installation
curl -fsSL https://ollama.com/install.sh | sh

# Basic commands
ollama run llama3.2         # Run interactively
ollama run llama3.2 "Hello" # Single prompt
ollama pull llama3.2:70b    # Download specific version
ollama list                 # List downloaded models
ollama ps                   # Show loaded models
ollama rm llama3.2          # Remove model
ollama stop llama3.2        # Unload from memory

# Custom model
ollama create mymodel -f ./Modelfile
ollama run mymodel

# API
curl http://localhost:11434/api/generate \
  -d '{"model": "llama3.2", "prompt": "Hello"}'

curl http://localhost:11434/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{
    "model": "llama3.2",
    "messages": [{"role": "user", "content": "Hello"}]
  }'

# Run with GPU control
OLLAMA_NUM_GPU=30 ollama run llama3.2:70b

# Server mode
ollama serve
```

## 24. Common Mistakes

1. **Running with default context (2048)** — for chat or code generation, increase `--num-ctx 8192` in Modelfile or `OLLAMA_NUM_CTX=8192`.

2. **Not using `keep_alive`** — default behavior keeps model in memory for 5 minutes after last request. For batch processing, set to `0` to unload immediately. For servers, set to `-1` to keep permanently.

3. **Assuming GPU is being used** — check `ollama ps` to verify `Size VRAM` column. If it shows 0, GPU offloading isn't working.

4. **Running in Docker without GPU passthrough** — need `--gpus all` flag. Without it, inference runs on CPU only.

5. **Downloading every version** — models are 4-40GB each. Use `ollama list` to audit and `ollama rm` unused models.

6. **Not reading model cards** — different models expect different prompt formats. Use the right Modelfile template.

7. **Expecting production throughput** — Ollama is optimized for developer experience, not throughput. Use vLLM/TGI for production.

## 25. Production Best Practices

1. **Use specific tags, not `latest`** — `ollama run llama3.2:3b-instruct-q4_K_M` ensures reproducibility. The `latest` tag changes with releases.

2. **Pre-pull models in Dockerfile** — include `RUN ollama pull llama3.2:3b` in your Docker image to avoid download delays at startup.

3. **Set `keep_alive` for your workload** — interactive: `-1` (keep loaded). Batch: `0` (unload after each). API server: `10m` (balance).

4. **Use a reverse proxy** — nginx in front of Ollama adds TLS termination, rate limiting, and request logging.

5. **Monitor VRAM usage** — use `ollama ps` or `nvidia-smi` to track. If VRAM is full, reduce `OLLAMA_NUM_GPU` or switch to a smaller quantization.

6. **Enable swap space** — 16GB+ swap prevents OOM crashes when running large models with limited RAM.

7. **Isolate models per GPU** — for multiple models, run separate Ollama instances on different ports, each pinned to a specific GPU with `CUDA_VISIBLE_DEVICES=0 ollama serve`.

8. **Use SSD storage** — model loading time is disk-bound. NVMe SSDs reduce load time from minutes to seconds for large models.

9. **Implement client-side retry** — Ollama returns 503 during model loading. Clients should retry with exponential backoff.

10. **Don't expose directly to internet** — Ollama has no built-in auth. Always put an authenticated proxy in front for remote access.
