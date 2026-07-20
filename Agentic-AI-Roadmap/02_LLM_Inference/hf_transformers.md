# HuggingFace Transformers for Inference

## 1. What is it?

HuggingFace Transformers is the de facto standard Python library for working with transformer models — providing a unified API for loading, running, and serving thousands of pre-trained models from the HuggingFace Hub.

**ELI5:** Think of HuggingFace Transformers as the "pip install" for AI models. Just as you install Python packages with `pip install`, you load any transformer model with `AutoModel.from_pretrained("model-name")`. It handles all the complex wiring of model architecture, tokenization, and configuration behind a single consistent API.

**Technical Definition:** Transformers is an open-source Python library providing AutoModel classes, tokenizers, configurations, and pipelines for transformer architectures (BERT, GPT, Llama, T5, etc.). It supports PyTorch, TensorFlow, and JAX backends with automatic device mapping, mixed precision, and model parallelism.

## 2. Why do we need it?

**Problem:** Before Transformers, using a new model meant reimplementing the architecture from scratch, writing custom tokenizers, and creating ad-hoc training scripts. Every model was a bespoke integration effort taking weeks.

**Pain without it:** To switch from BERT to RoBERTa, you'd need to rewrite the attention implementation, positional encodings, and layer normalization. To use GPT-2, you'd need a completely different codebase. Teams maintained separate codebases for each model family, duplicating effort and introducing bugs.

**Why companies use it:** Transformers abstracts away model architecture. The same `model.generate()` call works for Llama, Mistral, GPT-2, T5, and BLOOM. This allows teams to A/B test models without changing serving code, prototype new architectures instantly, and share infrastructure across model families.

## 3. Real-world Example

- **Google**: TPU research teams use Transformers with JAX backend for scaling experiments.
- **Meta AI**: Llama and Llama-2 were released with Transformers integration as the primary inference method.
- **Microsoft Azure**: ML Studio uses Transformers as the default model runtime for NLP pipelines.
- **Amazon SageMaker**: Built-in HuggingFace containers for one-click model deployment.
- **Adobe**: Content moderation and classification pipelines use Transformers for batch inference across millions of documents.
- **Grammarly**: Text classification and grammar correction models built on finetuned Transformers.
- **HuggingFace itself**: The Inference API and Inference Endpoints are built on Transformers.

## 4. Architecture Diagram (ASCII)

```
┌──────────────────────────────────────────────────────────┐
│                  HuggingFace Transformers                   │
│                                                            │
│  ┌────────────────────────────────────────────────────┐   │
│  │                    Auto Classes                      │   │
│  │  ┌────────────┐  ┌──────────┐  ┌────────────────┐  │   │
│  │  │AutoModel   │  │AutoTokeni│  │AutoConfig      │  │   │
│  │  │ForCausalLM │  │zer       │  │                │  │   │
│  │  └──────┬─────┘  └────┬─────┘  └───────┬────────┘  │   │
│  └─────────┼──────────────┼────────────────┼───────────┘   │
│            │              │                │               │
│            ▼              ▼                ▼               │
│  ┌────────────────────────────────────────────────────┐   │
│  │                   Pipeline API                      │   │
│  │  text-generation  │  text-classification  │  ...    │   │
│  └──────────────────────┬─────────────────────────────┘   │
│                         │                                 │
│            ┌────────────┴────────────┐                    │
│            ▼                         ▼                    │
│  ┌────────────────┐      ┌────────────────────┐          │
│  │    PyTorch     │      │  Backend-specific  │          │
│  │    Module      │      │  Optimizations     │          │
│  │  (Default)     │      │  (Flash Attn,      │          │
│  │                │      │   SDPA, BetterTrans)│          │
│  └────────────────┘      └────────────────────┘          │
│                                                            │
│  ┌────────────────────────────────────────────────────┐   │
│  │              Device & Memory Management             │   │
│  │  device_map="auto"  │  torch_dtype  │  offload      │   │
│  │                     │               │  folder       │   │
│  └────────────────────────────────────────────────────┘   │
└──────────────────────────────────────────────────────────┘
```

## 5. Internal Working

**Model Loading Pipeline:**
1. `AutoModel.from_pretrained("model-id")` triggers:
   - Check local cache (`~/.cache/huggingface/`)
   - If missing, download from HuggingFace Hub (config.json, model.safetensors, tokenizer.json)
   - Parse config.json to determine architecture type (LlamaConfig, GPT2Config, etc.)
   - Instantiate the correct model class dynamically
   - Load state dict into model (using safetensors for safety)
   - Apply device_map and dtype settings

**Inference Flow:**
1. Text → Tokenizer (encode to input_ids, attention_mask)
2. input_ids → Model forward pass (hidden states through transformer layers)
3. Logits → Sampler (greedy, beam search, top-k, top-p, temperature)
4. Output token_ids → Tokenizer (decode back to text)

**Pipeline API:**
The Pipeline wraps all steps into a single callable:
```python
pipe = pipeline("text-generation", model="meta-llama/Llama-2-7b")
result = pipe("Hello, I am", max_new_tokens=50)
```

## 6. Production Flow

```
Model Registry (HF Hub)
    │
    ▼
┌─────────────────────────────┐
│ Download weights & config   │
│ → local disk cache          │
│ → verify safetensors        │
└────────────┬────────────────┘
             │
             ▼
┌─────────────────────────────┐
│ Load model                  │
│ → determine device_map      │
│ → set torch_dtype           │
│ → enable flash_attention    │
│ → warm up CUDA kernels      │
└────────────┬────────────────┘
             │
             ▼
┌─────────────────────────────┐
│ Accept request              │
│ → tokenize                  │
│ → generate (loop)           │
│ → detokenize                │
│ → return response           │
└─────────────────────────────┘
```

## 7. HLD (High-Level Design)

```
┌──────────┐   ┌──────────┐   ┌──────────┐
│  Client  │   │  Client  │   │  Client  │
└────┬─────┘   └────┬─────┘   └────┬─────┘
     │              │              │
     └──────────────┼──────────────┘
                    │
            ┌───────▼────────┐
            │   API Gateway   │
            └───────┬────────┘
                    │
            ┌───────▼────────┐
            │  Load Balancer  │
            └───────┬────────┘
                    │
     ┌──────────────┼──────────────┐
     │              │              │
┌────▼────┐   ┌────▼────┐   ┌────▼────┐
│Pipeline │   │Pipeline │   │Pipeline │
│ Server 1│   │ Server 2│   │ Server N│
│(GPU:0)  │   │(GPU:1)  │   │(GPU:N)  │
└────┬────┘   └────┬────┘   └────┬────┘
     │              │              │
     └──────────────┼──────────────┘
                    │
            ┌───────▼────────┐
            │  Model Cache   │
            │  (Shared FS)   │
            └────────────────┘
```

## 8. LLD (Low-Level Design)

**Pipeline internals:**
```python
class TextGenerationPipeline(Pipeline):
    def __init__(self, model, tokenizer, **kwargs):
        self.model = model
        self.tokenizer = tokenizer
        self.generation_config = GenerationConfig.from_pretrained(
            model.config._name_or_path
        )

    def _forward(self, model_inputs, **generate_kwargs):
        input_ids = model_inputs["input_ids"]
        attention_mask = model_inputs.get("attention_mask")
        with torch.inference_mode():
            outputs = self.model.generate(
                input_ids=input_ids,
                attention_mask=attention_mask,
                **{**self.generation_config.to_dict(), **generate_kwargs},
            )
        return {"generated_ids": outputs}

    def preprocess(self, text, **kwargs):
        return self.tokenizer(
            text,
            return_tensors=self.framework,
            truncation=True,
            max_length=kwargs.get("max_length", 2048),
        )

    def postprocess(self, model_outputs, **kwargs):
        generated_ids = model_outputs["generated_ids"]
        return self.tokenizer.decode(
            generated_ids[0],
            skip_special_tokens=True,
        )
```

**Device map strategy (Accelerate):**
```
"auto" strategy:
1. Get available devices (CPU + each GPU)
2. Estimate memory for each module (via dummy forward pass)
3. Assign layers to devices like a bin-packing algorithm
4. Layers within same device are sequential; cross-device via model parallelism

Example for Llama-2-70B on 2x H100 (80GB):
- embedding → GPU 0
- layers 0-39 → GPU 0 (fits in ~75GB)
- layers 40-79 → GPU 1 (fits in ~75GB)
- lm_head → GPU 1
- Remaining RAM for KV cache on each GPU
```

## 9. Python Implementation

```python
import torch
import asyncio
from fastapi import FastAPI, HTTPException
from pydantic import BaseModel, Field
from typing import List, Optional, Dict, Any
from transformers import (
    AutoModelForCausalLM,
    AutoTokenizer,
    pipeline,
    BitsAndBytesConfig,
)
import logging

logging.basicConfig(level=logging.INFO)
logger = logging.getLogger("hf-inference")

app = FastAPI(title="HuggingFace Transformers Server")

class ChatMessage(BaseModel):
    role: str
    content: str

class ChatRequest(BaseModel):
    model: str = "meta-llama/Llama-3.2-3B-Instruct"
    messages: List[ChatMessage]
    temperature: float = Field(default=0.7, ge=0.0, le=2.0)
    max_tokens: int = 512
    top_p: float = 0.95
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

pipe = None
tokenizer = None
model = None

@app.on_event("startup")
async def startup():
    global pipe, tokenizer, model
    model_name = "meta-llama/Llama-3.2-3B-Instruct"

    quant_config = BitsAndBytesConfig(
        load_in_4bit=True,
        bnb_4bit_use_double_quant=True,
        bnb_4bit_quant_type="nf4",
        bnb_4bit_compute_dtype=torch.bfloat16,
    )

    tokenizer = AutoTokenizer.from_pretrained(model_name)
    tokenizer.pad_token = tokenizer.eos_token

    model = AutoModelForCausalLM.from_pretrained(
        model_name,
        device_map="auto",
        torch_dtype=torch.bfloat16,
        quantization_config=quant_config,
        attn_implementation="flash_attention_2",
        use_cache=True,
    )
    model.eval()

    pipe = pipeline(
        "text-generation",
        model=model,
        tokenizer=tokenizer,
        device_map="auto",
    )
    logger.info(f"Model loaded: {model_name}")

@app.post("/v1/chat/completions", response_model=ChatResponse)
async def chat_completions(request: ChatRequest):
    try:
        prompt = tokenizer.apply_chat_template(
            [m.model_dump() for m in request.messages],
            tokenize=False,
            add_generation_prompt=True,
        )

        outputs = pipe(
            prompt,
            max_new_tokens=request.max_tokens,
            temperature=request.temperature,
            top_p=request.top_p,
            do_sample=request.temperature > 0,
            return_full_text=False,
            pad_token_id=tokenizer.eos_token_id,
        )

        generated_text = outputs[0]["generated_text"]

        # Count tokens (approximate)
        input_tokens = len(tokenizer.encode(prompt))
        output_tokens = len(tokenizer.encode(generated_text))

        return ChatResponse(
            id=f"chatcmpl-{int(asyncio.get_event_loop().time())}",
            created=int(asyncio.get_event_loop().time()),
            model=request.model,
            choices=[{
                "index": 0,
                "message": {"role": "assistant", "content": generated_text},
                "finish_reason": "stop",
            }],
            usage=Usage(
                prompt_tokens=input_tokens,
                completion_tokens=output_tokens,
                total_tokens=input_tokens + output_tokens,
            ),
        )
    except torch.cuda.OutOfMemoryError:
        logger.error("GPU OOM — restarting model")
        raise HTTPException(status_code=507, detail="GPU out of memory")
    except Exception as e:
        logger.error(f"Inference failed: {e}", exc_info=True)
        raise HTTPException(status_code=500, detail=str(e))

@app.get("/health")
async def health():
    return {
        "status": "healthy",
        "model": model.config._name_or_path,
        "device": str(model.device),
    }

@app.get("/v1/models")
async def list_models():
    return {
        "data": [
            {
                "id": model.config._name_or_path,
                "object": "model",
                "created": int(asyncio.get_event_loop().time()),
                "owned_by": "huggingface",
            }
        ]
    }
```

## 10. Folder Structure

```
hf_inference_project/
├── Dockerfile
├── requirements.txt
├── config.yaml
├── serve.py                  # Entry point
├── hf_app/
│   ├── __init__.py
│   ├── server.py             # FastAPI server
│   ├── config.py             # Model config
│   ├── model_loader.py       # Optimized model loading
│   ├── quantization.py       # BitsAndBytes config
│   ├── tokenizer_utils.py    # Chat templates
│   └── metrics.py
├── models/
│   └── cache_config.py       # Cache directory management
├── k8s/
│   ├── deployment.yaml
│   ├── service.yaml
│   ├── pvc.yaml              # Persistent volume for model cache
│   └── hpa.yaml
└── tests/
    ├── test_server.py
    └── test_pipeline.py
```

## 11. Configuration

```yaml
model:
  name: meta-llama/Llama-3.2-3B-Instruct
  cache_dir: /models/cache  # Shared PVC for model weights
  torch_dtype: bfloat16
  device_map: auto
  trust_remote_code: false
  use_safetensors: true

quantization:
  enabled: true
  method: bitsandbytes  # bitsandbytes, gptq, awq
  load_in_4bit: true
  bnb_4bit_use_double_quant: true
  bnb_4bit_quant_type: nf4
  bnb_4bit_compute_dtype: bfloat16

attention:
  implementation: flash_attention_2  # eager, sdpa, flash_attention_2

generation:
  max_new_tokens: 2048
  temperature: 0.7
  top_p: 0.95
  repetition_penalty: 1.1
  do_sample: true
  num_beams: 1  # 1 = greedy, >1 for beam search

server:
  host: 0.0.0.0
  port: 8000
  workers: 1  # Always 1 for GPU inference
  max_queue_size: 100
```

## 12. Flowchart

```
Request → Tokenize → Prefill → Decode Loop → Detokenize → Response
                          │
                          ▼
               ┌─────────────────────┐
               │  Flash Attention?   │
               │  ┌──┐  ┌───┐       │
               │  │No│  │Yes│       │
               │  └──┘  └───┘       │
               │  eager   flash_attn2│
               └─────────────────────┘
                          │
                          ▼
               ┌─────────────────────┐
               │  Sampling Strategy  │
               │  temperature=0?     │
               │  ├── Yes: greedy   │
               │  └── No: sample    │
               │     top-k / top-p   │
               └─────────────────────┘
                          │
                          ▼
               ┌─────────────────────┐
               │  KV Cache update    │
               │  (contiguous memory)│
               └─────────────────────┘
                          │
                          ▼
                   ┌───────────┐
                   │ max_tokens│
                   │ reached?  │
                   │ EOS?      │
                   └───────────┘
                        │
                        ▼
                    Return output
```

## 13. Sequence Diagram

```
Client        FastAPI          Pipeline          Model            Tokenizer
  │              │                │                │                 │
  │───Request──▶│                │                │                 │
  │              │───pipe(text)─▶│                │                 │
  │              │                │───encode─────▶│                 │
  │              │                │◀──input_ids───│                 │
  │              │                │                │                 │
  │              │                │───generate───▶│                 │
  │              │                │                │───forward─────▶│
  │              │                │                │   (prefill)    │
  │              │                │                │◀──logits───────│
  │              │                │                │───sample──────▶│
  │              │                │                │◀──token────────│
  │              │                │                │───forward─────▶│
  │              │                │                │   (decode)     │
  │              │                │                │◀──logits───────│
  │              │                │                │───sample──────▶│
  │              │                │                │   (repeat)     │
  │              │                │◀──tokens──────│                 │
  │              │                │───decode─────▶│                 │
  │              │                │◀──text────────│                 │
  │              │◀──result──────│                │                 │
  │◀──Response──│                │                │                 │
```

## 14. Pros

- **Largest model ecosystem** — 500K+ models on HuggingFace Hub
- **Unified API** — same code works for BERT, GPT, T5, Llama, etc.
- **Multi-framework** — PyTorch, TensorFlow, JAX support in one library
- **Quantization ready** — built-in BitsAndBytes, GPTQ, AWQ support
- **Active development** — 250K+ GitHub stars, weekly releases
- **Pipeline abstraction** — one-line inference for text, image, audio, video
- **Comprehensive docs** — extensive tutorials, course, and API reference
- **Community models** — access to latest research within days of release

## 15. Cons

- **Slower than optimized engines** — 3-10x slower than vLLM/TGI for production
- **Contiguous KV cache** — no PagedAttention; memory fragmentation issues
- **No continuous batching** — need custom batching logic for concurrency
- **Python dependency** — heavy deployment footprint (PyTorch alone is ~2GB)
- **Model loading time** — downloading and loading large models takes minutes
- **CUDA graph overhead** — no CUDA graph caching by default; recompiles on each call
- **Memory fragmentation** — repeated generation without restart increases memory usage over time

## 16. Alternatives

| Alternative | Strengths | Weaknesses |
|------------|-----------|------------|
| **vLLM** | PagedAttention, 24x faster | Limited model support |
| **TGI** | HF-native, optimized for HF models | Requires separate server |
| **TensorRT-LLM** | NVIDIA-optimized, best latency | Complex, vendor lock-in |
| **Llama.cpp** | CPU inference, GGUF format | No Pythonic API |
| **ONNX Runtime** | Cross-platform optimization | Limited model coverage |

## 17. Performance Considerations

**Memory overhead:**
```
Model           dtype    Weights   KV Cache (4K)  Total
Llama-3-8B      bf16     16 GB       2 GB         18 GB
Llama-3-8B      4-bit     5 GB       2 GB          7 GB
Llama-3-70B     bf16    140 GB      16 GB        156 GB
Llama-3-70B     4-bit    35 GB      16 GB         51 GB
```

**Optimizations:**
- `attn_implementation="flash_attention_2"` — 2-4x faster attention, less memory
- `torch.compile(model)` — 10-30% speedup via CUDA graph fusion (set `mode="reduce-overhead"`)
- `device_map="auto"` with multiple GPUs — automatic model parallelism
- `use_cache=True` — enables KV cache (must be True for generation)
- `torch.inference_mode()` context — disables autograd for speed

**Generation parameters:**
- `num_beams=1` (greedy) is fastest. Beam search doubles compute per beam.
- `do_sample=False` is slightly faster than sampling (no random number generation).
- Larger `max_new_tokens` increases GPU memory — set only what you need.

## 18. Scaling to Millions

**Strategy for scale with Transformers:**

1. **Pre-compile with `torch.compile`** — reduces first-iteration overhead by JIT-fusing CUDA kernels.

2. **Batch inference** — pass multiple prompts in a single batch via `pipe(texts, batch_size=8)`. Improves throughput but increases TTFT.

3. **Model sharding across GPUs** — use `device_map="auto"` with `accelerate` to split models across multiple GPUs.

4. **Quantization** — 4-bit quantization reduces memory by 4x, enabling larger models on fewer GPUs.

5. **Inference endpoint pattern** — one model replica per GPU behind a load balancer, with auto-scaling based on queue depth.

**Maximum realistic throughput (Llama-3-8B, 1x H100):**
- Naive pipeline: 20-30 req/min (1 request at a time)
- With batch_size=8: 80-120 req/min
- With vLLM: 200-400 req/min

## 19. Failure Scenarios

| Failure | Impact | Mitigation |
|---------|--------|------------|
| **CUDA OOM** | Request fails, model state corrupted | Lower `max_new_tokens`, enable quantization, use `clear_memory()` |
| **Tokenizer mismatch** | Incorrect encoding → garbled output | Verify tokenizer matches model_id exactly |
| **Model ID not found** | 404 on startup | Fallback to local path, verify HF_TOKEN |
| **Safetensors missing** | Loading fails | Set `use_safetensors=False`, log warning |
| **GPU lost** | CUDA error, model unusable | Restart pod, reinitialize model |
| **Cache corruption** | Modified weights → random output | Clear cache, verify checksums |

## 20. Security

| Concern | Mitigation |
|---------|------------|
| **Model supply chain** | Use `use_safetensors=True` (no pickle), verify model authors |
| **Malicious config** | Set `trust_remote_code=False` to prevent arbitrary code execution |
| **Prompt injection** | Implement output filtering, content safety check after generation |
| **Data leakage** | Don't log prompt/response content in production |
| **Model theft** | Use private repos with HF_TOKEN environment variable |
| **Dependency vulns** | Pin versions, use `pip audit` / `safety check` |

## 21. Monitoring

**Prometheus metrics to export:**
```
transformers:request_duration_ms    — End-to-end latency
transformers:token_generation_speed — Tokens per second (throughput)
transformers:gpu_memory_used_gb     — GPU memory consumption
transformers:model_load_time_ms     — Model loading duration
transformers:queue_depth            — Requests waiting
transformers:batch_size             — Current batch size (if batching)
transformers:kv_cache_size_mb       — KV cache memory usage
```

**Alerts:**
- GPU memory > 90% of VRAM → consider quantization or reduce context
- Generation speed < 10 tok/s for 7B → check for CPU fallback
- OOM rate > 1% → reduce max_tokens or batch size
- Health check failure > 3 consecutive restarts → roll back model version

## 22. Interview Questions

**Q1: How does `device_map="auto"` work?**
*A: Accelerate's `infer_auto_device_map` function estimates the memory usage of each model module by running a dummy forward pass, then uses a bin-packing algorithm to assign modules to devices (CPU + GPUs). It respects the memory budget per device and minimizes cross-device communication. The model is split across devices using hooks that transfer hidden states between GPUs.*

**Q2: What's the difference between `from_pretrained` and loading from local files?**
*A: `from_pretrained` checks the HuggingFace Hub cache first (`~/.cache/huggingface/`). If not cached, it downloads the full model (config.json, model.safetensors, tokenizer.json, etc.). To force local loading, set `local_files_only=True`. The Hub version is preferred for first use; subsequent loads use cached weights.*

**Q3: How would you reduce GPU memory usage for inference?**
*A: Options ranked by impact: (1) Quantization — BitsAndBytes 4-bit reduces memory by 4x; (2) Reduce `max_new_tokens` and `max_length` — each token of KV cache costs O(layers * d_model); (3) Use `attn_implementation="flash_attention_2"` — 2x less memory for attention; (4) Offload to CPU — `device_map="auto"` puts unused layers on CPU; (5) Use `torch.inference_mode()` to disable autograd overhead.*

**Q4: Explain the pipeline pattern and when you'd use it vs raw model calls.**
*A: Pipeline wraps tokenizer, model, and postprocessing into a single callable. It handles device placement, dtype conversion, batch dimension, and truncation automatically. Use pipelines for rapid prototyping and simple inference (text generation, classification). Use raw model calls when you need fine-grained control: custom generation parameters, manual KV cache management, streaming token-by-token, or interleaving multiple generation calls.*

## 23. Cheat Sheet

```python
# Basic inference
from transformers import pipeline
pipe = pipeline("text-generation", model="gpt2")
print(pipe("Hello, I'm")[0]["generated_text"])

# Load specific model with optimizations
from transformers import AutoModelForCausalLM, AutoTokenizer
import torch

model = AutoModelForCausalLM.from_pretrained(
    "meta-llama/Llama-3.2-3B-Instruct",
    device_map="auto",
    torch_dtype=torch.bfloat16,
    attn_implementation="flash_attention_2",
)
tokenizer = AutoTokenizer.from_pretrained("meta-llama/Llama-3.2-3B-Instruct")

# Generate
inputs = tokenizer("Hello!", return_tensors="pt").to("cuda")
outputs = model.generate(**inputs, max_new_tokens=100)
print(tokenizer.decode(outputs[0]))

# Model info
print(model.config)
print(model.device)
print(model.dtype)

# List available models on HF Hub
from huggingface_hub import list_models
models = list_models(task="text-generation", sort="downloads")

# Clear GPU cache
import torch
torch.cuda.empty_cache()
```

## 24. Common Mistakes

1. **Forgetting `use_cache=True`** — disables KV cache, causing O(N²) runtime instead of O(N) per token.

2. **Not setting `pad_token_id`** — causes warning and potential generation errors. Always set `tokenizer.pad_token = tokenizer.eos_token`.

3. **Using `do_sample=True` with `temperature=0`** — temperature 0 should use greedy decoding (do_sample=False). Combining them wastes compute.

4. **Loading full precision (float32) unnecessarily** — default is float32 without explicit `torch_dtype`. Always set `torch_dtype=torch.bfloat16` or `torch.float16` for inference.

5. **Not using `attention_mask` in padded batches** — model attends to padding tokens, producing garbage. Always pass `attention_mask` in batched input.

6. **Loading models from untrusted sources with `trust_remote_code=True`** — this executes arbitrary code from the Hub. Never use without code review.

7. **Calling `model.train()` instead of `model.eval()`** — enables dropout and batch norm training behavior, reducing output quality.

## 25. Production Best Practices

1. **Always use `model.eval()` and `torch.inference_mode()`** — disables autograd, reduces memory by ~40%, speeds up inference.

2. **Enable Flash Attention 2** — requires `pip install flash-attn --no-build-isolation` and `attn_implementation="flash_attention_2"`. 2-4x faster generation.

3. **Use safetensors** — `use_safetensors=True` (default since Transformers 4.34). Safer and faster than pickle-based loading.

4. **Pre-compile with `torch.compile`** — first call will be slow (30-60s for compilation), but subsequent calls are 10-30% faster.

5. **Share model cache across replicas** — mount a PVC at `~/.cache/huggingface/` so all replicas share downloaded weights.

6. **Pin model version** — use full commit hash in model ID: `meta-llama/Llama-3.2-3B-Instruct@sha256:abc123...` for reproducible loading.

7. **Warm up before serving traffic** — send a synthetic request during startup to compile CUDA graphs and warm the cache.

8. **Set explicit `max_memory`** — `from_pretrained(..., max_memory={0: "70GiB", "cpu": "100GiB"})` prevents device_map from over-allocating.

9. **Monitor with `nvidia-smi`** — watch for GPU memory growth over time. If it grows without bound, you have a memory leak.

10. **Use `generate` kwargs, not config changes** — pass `pad_token_id`, `eos_token_id`, etc. as kwargs to `generate()` rather than modifying model config globally.
