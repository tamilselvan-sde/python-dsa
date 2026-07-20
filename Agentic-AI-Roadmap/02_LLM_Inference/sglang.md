# SGLang Inference Framework

> Author: **Tamilselvan** · ✉️ tamilselvan.sde@gmail.com · 🔗 [LinkedIn](https://www.linkedin.com/in/tamilselvan-ai/)
>

## 1. What is it?

SGLang is an inference framework designed for structured generation with large language models. It introduces **RadixAttention** — a prefix-sharing mechanism that reuses KV cache across generations with common prefixes, and a domain-specific language (DSL) for expressing complex generation workflows.

**ELI5:** Imagine you're asking a chef to make 100 custom pizzas. Instead of making each from scratch, SGLang notices that 80 pizzas share the same base sauce and cheese, so it preps all the bases together, then customizes each one. This saves enormous effort.

**Technical Definition:** SGLang (Structured Generation Language) is a serving system that introduces RadixAttention for automatic prefix sharing in the KV cache and a Pythonic DSL for programming LLM interactions — enabling structured outputs, constrained decoding, multi-turn branching, and parallel generation with significantly reduced redundancy.

## 2. Why do we need it?

**Problem:** In agentic AI systems, LLM calls follow predictable patterns: system prompts, tool schemas, few-shot examples, and conversation histories are shared across requests. Traditional inference engines recompute the KV cache for these repetitive prefixes every time, wasting 40-80% of computation.

**Pain without it:** Consider an agent that calls 5 tools per user request. Each call includes the full system prompt and conversation history. Without prefix sharing, the system prompt is recomputed 5 times per user — each time costing as much as the actual tool call. For 10K users, this means 200K KV cache prefill operations instead of 10K.

**Why companies use it:** SGLang's DSL lets developers express structured generation (JSON mode, constrained decoding, parallel tool calls) in native Python, then automatically optimizes the execution plan. It reduces latency by 2-5x for agentic workloads without any developer effort.

## 3. Real-world Example

- **Anthropic** (indirectly): Claude's tool use patterns inspired SGLang's constrained generation for function calling.
- **Databricks** (via MosaicML): Uses SGLang for structured data extraction from enterprise documents at scale.
- **LMSYS Org**: The framework originates from the LMSYS research group at UC Berkeley, used for Chatbot Arena evaluation infrastructure.
- **Together AI**: Offers SGLang as a deployment option alongside vLLM for structured generation workloads.
- **Agentic workflow platforms**: LangChain, AutoGen, and CrewAI integrations benefit from SGLang's parallel tool-calling optimizations.

## 4. Architecture Diagram (ASCII)

```
┌──────────────────────────────────────────────────────────────────┐
│                       SGLang Runtime                               │
│                                                                    │
│  ┌─────────────────────┐  ┌────────────────────────────────────┐  │
│  │    SGLang DSL        │  │      Request Router                │  │
│  │  (Pythonic API)      │  │   /generate, /classify, /extract   │  │
│  └─────────┬───────────┘  └────────────┬───────────────────────┘  │
│            │                           │                          │
│            ▼                           ▼                          │
│  ┌────────────────────────────────────────────────────────────┐   │
│  │                      Scheduler                              │   │
│  │  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐   │   │
│  │  │Program   │  │Program   │  │Program   │  │Program   │   │   │
│  │  │Queue     │  │Optimizer │  │Executor  │  │Compiler  │   │   │
│  │  └────┬─────┘  └────┬─────┘  └────┬─────┘  └────┬─────┘   │   │
│  └───────┼─────────────┼─────────────┼──────────────┼─────────┘   │
│          │             │             │              │             │
│          ▼             ▼             ▼              ▼             │
│  ┌────────────────────────────────────────────────────────────┐   │
│  │                RadixAttention Manager                       │   │
│  │                                                              │   │
│  │          System Prompt  (shared prefix tree)                 │   │
│  │               │              │                               │   │
│  │          ┌────┴────┐   ┌────┴────┐                          │   │
│  │     Tool Schema  "User A"  Tool Schema  "User B"            │   │
│  │          │              │              │                    │   │
│  │    ┌─────┴─────┐  ┌────┴────┐     ┌────┴────┐              │   │
│  │  tool/1  tool/2   user/msg   user/tools  user/msg           │   │
│  │                                                              │   │
│  └──────────────────────────┬───────────────────────────────────┘   │
│                             │                                      │
│                             ▼                                      │
│  ┌────────────────────────────────────────────────────────────┐   │
│  │                   GPU Workers (CUDA)                        │   │
│  └────────────────────────────────────────────────────────────┘   │
└──────────────────────────────────────────────────────────────────┘
```

## 5. Internal Working

SGLang's core innovation is **RadixAttention**, a trie-based KV cache management system.

**How RadixAttention works:**
1. Every sequence of tokens is stored as a path in a **radix tree** (prefix tree)
2. When a new request arrives, SGLang finds the longest matching prefix in the tree
3. The KV cache for the prefix is reused — only the new suffix is computed
4. The tree supports **copy-on-write** — shared nodes are not duplicated until a fork diverges
5. When memory is full, **LRU eviction** removes leaf nodes (least recently used branches)

**The DSL (Domain-Specific Language):**
SGLang lets you write generation logic as a Python program:
```python
@function
def tool_call(s, question):
    s += "You are a helpful assistant."
    s += "Question: " + question
    s += "Answer (as JSON):"
    s += gen("answer", max_tokens=256, regex=json_schema)
```

This is **compiled into an execution plan** — the framework knows the exact structure before execution, enabling:
- **Program parallelism** — independent generation calls run concurrently
- **Constrained decoding** — output is forced to match regex/grammar during generation
- **Automatic batching** — multiple program instances are fused into optimal GPU kernels

## 6. Production Flow

```
Request (DSL program) → Parser → AST → Optimizer → Executor → Scheduler
                                                                  │
     ┌────────────────────────────────────────────────────────────┘
     │
     ▼
RadixAttention Lookup → Match prefix → KV Cache Hit? 
     │                                      │
  (miss)                              (hit)
     │                                      │
     ▼                                      ▼
Compute new KV blocks              Reuse cached blocks
     │                                      │
     └──────────────┬───────────────────────┘
                    ▼
           GPU Decode Loop
                    │
                    ▼
          Stream output tokens
```

## 7. HLD (High-Level Design)

```
┌──────────┐
│  Client  │──────────┐
└──────────┘          │
                      ▼
               ┌────────────────┐
               │  API Gateway   │
               │  (TLS + Auth)  │
               └────────┬───────┘
                        │
               ┌────────▼───────┐
               │  SGLang Router │
               └────────┬───────┘
                        │
        ┌───────────────┼───────────────┐
        │               │               │
┌───────▼───────┐ ┌─────▼───────┐ ┌─────▼───────┐
│ SGLang Node 1 │ │ SGLang Node 2│ │ SGLang Node N│
│  + RadixTree  │ │  + RadixTree│ │  + RadixTree│
│  + GPU (H100) │ │  + GPU (H100)│ │  + GPU (H100)│
└───────┬───────┘ └─────┬───────┘ └─────┬───────┘
        │               │               │
        └───────┬───────┴───────┬───────┘
                │               │
        ┌───────▼───────┐ ┌─────▼───────┐
        │  Prometheus   │ │  Redis Cache│
        │  + Grafana    │ │  (sharded)  │
        └───────────────┘ └─────────────┘
```

## 8. LLD (Low-Level Design)

**Radix Tree Node Structure:**
```python
class RadixNode:
    def __init__(self, prefix_ids: list[int]):
        self.prefix_ids = prefix_ids
        self.children: dict[int, 'RadixNode'] = {}
        self.kv_cache_blocks: list[int] = []
        self.last_access_time: float = time.time()
        self.lock_count: int = 0  # copy-on-write tracking

    def match_prefix(self, tokens: list[int]) -> int:
        """Return length of matching prefix."""
        match_len = 0
        for i, tid in enumerate(self.prefix_ids):
            if i >= len(tokens) or tokens[i] != tid:
                break
            match_len += 1
        return match_len
```

**Execution Plan Compilation:**
```python
class SGLangProgram:
    def __init__(self, ast: list[ASTNode]):
        self.ast = ast
        self.execution_plan = None

    def optimize(self):
        """Analyze data dependencies to find parallelizable operations."""
        dag = DependencyGraph(self.ast)
        self.execution_plan = dag.topological_sort()

    def execute(self, engine, prefix_tree):
        for step in self.execution_plan:
            if step.is_parallel():
                # Fork multiple generation streams
                futures = [engine.generate(node, prefix_tree) for node in step.children]
                results = await asyncio.gather(*futures)
                # Merge back into single stream
                prefix_tree.merge(results)
            else:
                result = engine.generate(step, prefix_tree)
```

## 9. Python Implementation

```python
import asyncio
from fastapi import FastAPI
from pydantic import BaseModel, Field
from typing import Optional, List, Dict, Any
import sglang as sgl
from sglang.srt.server import Runtime
from sglang.srt.managers.router import Router
import logging

logging.basicConfig(level=logging.INFO)
logger = logging.getLogger("sglang-server")

app = FastAPI(title="SGLang Inference Server")

# --- SGLang DSL Functions ---

@sgl.function
def extract_json(s, text, schema):
    s += "Extract information from the following text:\n"
    s += text + "\n"
    s += "Output as JSON following this schema:\n"
    s += schema + "\n"
    s += sgl.gen("output", max_tokens=512, regex=schema)

@sgl.function
def parallel_tool_calls(s, query, tools):
    s += "You are an AI assistant with access to tools.\n"
    s += "Query: " + query + "\n"
    s += "Available tools:\n"
    for tool in tools:
        s += f"- {tool['name']}: {tool['description']}\n"
    s += "You MUST call ALL relevant tools.\n"

    # Parallel generation for each tool
    forks = []
    for tool in tools:
        forks.append(
            sgl.gen(
                f"call_{tool['name']}",
                max_tokens=128,
                regex=tool.get("schema", ".*"),
            )
        )

    s += sgl.gen("tool_calls", max_tokens=2048, fork=forks)

# --- API Layer ---

class ToolDef(BaseModel):
    name: str
    description: str
    schema: Optional[str] = None

class ExtractRequest(BaseModel):
    text: str
    schema: str

class ToolRequest(BaseModel):
    query: str
    tools: List[ToolDef]
    stream: bool = False

class FunctionCall(BaseModel):
    name: str
    arguments: str

class ToolResponse(BaseModel):
    tool_calls: List[FunctionCall]
    usage: Dict[str, int]

runtime: Runtime = None

@app.on_event("startup")
async def startup():
    global runtime
    runtime = Runtime(
        model_path="meta-llama/Meta-Llama-3-8B-Instruct",
        tokenizer_path="meta-llama/Meta-Llama-3-8B-Instruct",
        port=30000,
        tp_size=1,
        radix_cache_size=2_000_000_000,  # ~2GB for radix cache
    )
    await runtime.start()
    logger.info("SGLang runtime initialized")

@app.post("/v1/extract", response_model=Dict[str, Any])
async def extract(request: ExtractRequest):
    try:
        state = extract_json.run(
            text=request.text,
            schema=request.schema,
            temperature=0.1,
        )
        return {"output": state["output"], "usage": state.get_usage()}
    except Exception as e:
        logger.error(f"Extraction failed: {e}")
        raise HTTPException(status_code=500, detail=str(e))

@app.post("/v1/tool-call", response_model=ToolResponse)
async def tool_call(request: ToolRequest):
    try:
        state = parallel_tool_calls.run(
            query=request.query,
            tools=[t.model_dump() for t in request.tools],
            temperature=0.0,
        )
        calls = []
        for t in request.tools:
            key = f"call_{t.name}"
            if key in state:
                calls.append(FunctionCall(name=t.name, arguments=state[key]))
        return ToolResponse(tool_calls=calls, usage=state.get_usage())
    except Exception as e:
        logger.error(f"Tool call failed: {e}")
        raise HTTPException(status_code=500, detail=str(e))

@app.get("/health")
async def health():
    return {"status": "healthy", "cache_size": runtime.radix_cache_size}
```

## 10. Folder Structure

```
sglang_project/
├── Dockerfile
├── requirements.txt
├── config.yaml
├── serve.py                    # Entry point
├── sglang_app/
│   ├── __init__.py
│   ├── server.py               # FastAPI server
│   ├── config.py               # Configuration models
│   ├── functions/
│   │   ├── __init__.py
│   │   ├── extract.py          # JSON extraction DSL
│   │   ├── tool_calls.py       # Parallel tool calling
│   │   ├── classification.py   # Constrained classification
│   │   └── multi_turn.py       # Multi-turn conversations
│   ├── middleware/
│   │   ├── auth.py
│   │   ├── rate_limit.py
│   │   └── audit.py
│   └── metrics.py
├── k8s/
│   ├── deployment.yaml
│   ├── service.yaml
│   ├── radix-cache-config.yaml
│   └── hpa.yaml
└── tests/
    ├── test_functions.py
    └── test_server.py
```

## 11. Configuration

```yaml
model:
  path: meta-llama/Meta-Llama-3-70B-Instruct
  dtype: bfloat16
  tp_size: 4

runtime:
  radix_cache_size: 4000000000  # 4GB for radix tree (KV prefix cache)
  max_total_tokens: 8192
  max_running_requests: 256
  mem_fraction_static: 0.88

scheduler:
  policy: lru  # Least Recently Used for radix eviction
  max_waiting_time_ms: 100
  batch_size_cap: 128

serving:
  host: 0.0.0.0
  port: 8000
  enable_metrics: true
  log_level: info

tools:
  max_parallel_tools: 10
  default_temperature: 0.0
```

## 12. Flowchart

```
                    ┌───────────────┐
                    │ DSL Program   │
                    └───────┬───────┘
                            │
                            ▼
                    ┌───────────────┐
                    │ Parse to AST  │
                    └───────┬───────┘
                            │
                            ▼
                    ┌───────────────┐
                    │ Optimize DAG  │
                    │ (parallel?)   │
                    └───────┬───────┘
                            │
                            ▼
                    ┌───────────────┐
                    │ Tokenize      │
                    └───────┬───────┘
                            │
                            ▼
                    ┌───────────────┐
                    │ Radix Lookup  │
                    └───────┬───────┘
                            │
              ┌─────────────┴─────────────┐
              │                           │
         ┌────▼────┐               ┌──────▼──────┐
         │ Cache   │               │ Cache Miss  │
         │ Hit     │               │ Compute     │
         └────┬────┘               └──────┬──────┘
              │                           │
              └──────┬────────────────────┘
                     │
                     ▼
          ┌─────────────────────┐
          │ Constrained Decode  │
          │ (regex/grammar)     │
          └──────────┬──────────┘
                     │
                     ▼
          ┌─────────────────────┐
          │  Backtrack if      │
          │  needed (fork)     │
          └──────────┬──────────┘
                     │
                     ▼
          ┌─────────────────────┐
          │  Return Output      │
          └─────────────────────┘
```

## 13. Sequence Diagram

```
Client        FastAPI       SGLang Runtime    RadixTree      GPU
  │              │                │               │            │
  │───DSL──────▶│                │               │            │
  │              │───parse─────▶│               │            │
  │              │                │───find_pref─▶│            │
  │              │                │◀───match────│            │
  │              │                │               │            │
  │              │           ┌────┴──────┐                   │
  │              │           │fork paths?│                   │
  │              │           └────┬──────┘                   │
  │              │                │                          │
  │              │     ┌──────────┴──────────┐               │
  │              │     │                     │               │
  │              │  ┌──▼──┐             ┌────▼───┐          │
  │              │  │A│B│C│             │D│E│F  │          │
  │              │  └──┬──┘             └────┬───┘          │
  │              │     │                     │               │
  │              │     └──────────┬──────────┘               │
  │              │                │                          │
  │              │       ┌────────▼────────┐                 │
  │              │       │  GPU Execute    │                 │
  │              │       │  (prefill+)     │────────────────▶│
  │              │       │  decode loop    │◀────────────────│
  │              │       └────────┬────────┘                 │
  │              │                │                          │
  │              │        ┌───────▼────────┐                 │
  │              │        │  Insert into   │                 │
  │              │        │  Radix Tree    │                 │
  │              │        └───────┬────────┘                 │
  │              │◀────result─────│                          │
  │◀───response──│                │                          │
```

## 14. Pros

- **RadixAttention** — automatic prefix sharing reduces prefill compute by 2-10x
- **Structured generation DSL** — native Python syntax for complex generation workflows
- **Constrained decoding** — regex/grammar-guided output (JSON, SQL, code)
- **Parallel execution** — automatic fork-join for independent generation branches
- **Higher throughput** than vLLM for agentic/structured workloads (2-5x)
- **Copy-on-write** — safe prefix sharing without data races
- **OpenAI-compatible API** available
- **Tool calling support** — first-class citizen in the DSL

## 15. Cons

- **Newer project** — smaller ecosystem and community than vLLM (4K vs 30K GitHub stars)
- **Less battle-tested** — fewer production deployments at hyperscale
- **DSL learning curve** — developers must learn SGLang syntax vs standard inference APIs
- **Radix tree memory overhead** — the tree itself uses memory for pointers and metadata
- **No built-in multi-modal** — text-only inference
- **Limited hardware support** — CUDA only, no AMD ROCm or Apple Silicon
- **Fewer quantization options** — mainly AWQ/FP8; no GPTQ dynamic quantization

## 16. Alternatives

| Alternative | When to Use | Limitations |
|------------|-------------|-------------|
| **vLLM** | General-purpose LLM serving | No structured generation DSL |
| **Outlines** | Constrained generation library | No serving, Python library only |
| **Guidance** (Microsoft) | Structured generation with grammars | No prefix sharing, slower |
| **LMQL** | Declarative LLM programming | Smaller ecosystem |
| **JSON-mode in TGI** | Simple JSON output | No parallelism, no sharing |
| **Ollama** | Local development | No production features |

## 17. Performance Considerations

**Radix tree efficiency:**
- Cache hit rate depends on prefix diversity. For shared system prompts with N users, the first request pays full prefill cost; subsequent requests reuse N-1 copies of the prefix.
- Tree depth impacts lookup speed. Keep radix tree shallow by limiting prefix length to 4096 tokens in config.
- Eviction policy matters: LRU works well for chat workloads; LFU better for repeated extraction patterns.

**Constrained decoding overhead:**
- Regex-guided decoding adds 10-30% per token overhead vs free generation.
- For JSON mode, use SGLang's built-in JSON schema compiler (faster than regex).
- For complex grammars, precompile the grammar into a DFA for O(1) per-token checking.

**Batching strategy:**
- SGLang's scheduler groups programs by structure — programs with the same DSL structure are batched together for optimal GPU utilization.
- For maximum throughput: submit programs with identical prefixes in short succession.
- For latency-sensitive workloads: limit batch size to 32 sequences.

## 18. Scaling to Millions

**Horizontal scaling:**
1. **Shard by model** — different SGLang deployments for different model sizes
2. **Distributed radix tree** — use Redis-backed radix tree for cross-node prefix sharing
3. **Program caching** — cache compiled execution plans (AST → optimized DAG) in Redis
4. **Auto-scaling** — scale based on `sglang:queue_depth` metric

**For 1M structured generation requests/day:**
```
Requests/sec:    ~12 structured calls
Avg program:    3 parallel branches + 2 sequential steps
KV cache reuse: 70% hit rate (shared system prompt + schemas)
GPUs needed:    ~8 H100-80GB (Llama 70B)
Throughput:     ~500 tokens/sec per node
```

**Bottleneck:** Radix tree mutation locking. At high concurrency, the radix tree becomes a contention point. Mitigate by using per-bucket locks (hash-based sharding within the tree).

## 19. Failure Scenarios

| Failure | Impact | Mitigation |
|---------|--------|------------|
| **Radix tree corruption** | Incorrect KV cache reuse, garbled output | Checksums per node, periodic tree validation |
| **Constrained decode deadlock** | No valid token matches grammar | Relax regex to include fallback token, backtrack |
| **Program compilation timeout** | DSL with infinite loop | Max compilation time (5s), validate AST cycles |
| **Cache stampede** | All requests miss cache simultaneously | Pre-warm radix tree with common prefixes |
| **Memory leak in radix tree** | Gradual memory growth over days | Periodic tree compaction, max node age limit |

## 20. Security

| Concern | Mitigation |
|---------|------------|
| **Regex injection** | Malicious regex causing catastrophic backtracking | Limit regex complexity, timeout per regex attempt |
| **DSL injection** | Unsanitized user input in DSL programs | Sandbox DSL compilation, no dynamic code execution |
| **Cache poisoning** | Crafted input causing incorrect KV reuse | Input canonicalization, cache key includes hash of full prefix |
| **Schema DoS** | Deeply nested JSON schema causing OOM | Max schema depth (16), max JSON size (8K tokens) |
| **Grammar backtracking** | Exponential time in constrained decoding | Use DFA-based regex compilation (linear time) |

## 21. Monitoring

**Key metrics:**
```
sglang:radix_cache_hit_rate       — Hit rate for KV prefix cache
sglang:radix_cache_size_bytes     — Current radix tree memory usage
sglang:program_queue_depth        — DSL programs waiting for execution
sglang:constrained_decode_ratio   — Fraction of tokens decoded with constraints
sglang:avg_program_complexity     — Avg parallel branches per program
sglang:ttft_ms                    — Time to first token
sglang:tpot_ms                    — Time per output token
sglang:cache_eviction_rate        — Nodes evicted per minute from radix tree
```

**Grafana panels:**
1. Cache hit rate over time (prefix reuse effectiveness)
2. Program complexity distribution (branches, steps)
3. Constrained vs free decoding throughput
4. Radix tree memory breakdown
5. TTFT by program type (extraction, tool call, chat)

**Alerts:**
- Cache hit rate < 20% → no shared prefixes; check system prompt design
- Radix tree memory > 90% of budget → increase cache size or reduce diversity
- Constrained decode failures > 1% → check regex/schema validity
- Program compilation time > 3s → check for complex/cyclic DSL programs

## 22. Interview Questions

**Q1: Explain how RadixAttention differs from PagedAttention in vLLM.**
*A: PagedAttention focuses on memory efficiency via virtual-memory-style block tables for the KV cache. RadixAttention additionally organizes these blocks into a prefix tree (trie) so that sequences sharing prefixes automatically share KV cache blocks. While PagedAttention can be combined with prefix caching, RadixAttention makes prefix sharing a first-class architectural feature with copy-on-write and LRU eviction at the prefix level.*

**Q2: How does SGLang's DSL enable better performance than standard API calls?**
*A: The DSL lets SGLang analyze the entire generation workflow before execution. It can identify parallel branches (independent gen calls), optimize scheduling, and batch operations that would otherwise require multiple sequential API calls. This program-awareness enables execution plan optimization that's impossible with stateless API calls.*

**Q3: What types of workloads benefit most from SGLang vs vLLM?**
*A: SGLang excels at structured generation workloads: tool calling (many parallel function calls), JSON extraction, classification, and multi-step agentic workflows. These benefit from the DSL's parallelism and constrained decoding. vLLM is better for standard chat/completion workloads where the DSL overhead isn't justified.*

**Q4: How would you handle a 10x increase in prefix diversity — reducing cache hit rate from 80% to 20%?**
*A: First, analyze the root cause: are system prompts truly diverse or is it tokenization causing artificial diversity? If genuine, implement prefix canonicalization (normalize whitespace, sort tool schemas). If that's insufficient, increase radix cache size and switch to LFU eviction (frequent prefixes stay cached). For extreme diversity, disable prefix sharing and fall back to standard PagedAttention.*

## 23. Cheat Sheet

```python
# Basic SGLang function
@sgl.function
def hello(s, name):
    s += "Hello, my name is "
    s += sgl.gen("name", max_tokens=10)

# Run it
state = hello.run(name="World")
print(state["name"])

# JSON extraction
@sgl.function
def extract_person(s, text):
    s += f"Extract from: {text}\n"
    s += "Output JSON: {"
    s += sgl.gen("json", max_tokens=200, regex=r'"name":\s*"[\w\s]+",\s*"age":\s*\d+')

# Parallel tool calls
@sgl.function
def multi_tool(s, query):
    s += "Query: " + query + "\n"
    s += "Calling tools in parallel:\n"
    gen1 = sgl.gen("tool1", max_tokens=100)
    gen2 = sgl.gen("tool2", max_tokens=100)
    s += sgl.gen("combined", fork=[gen1, gen2])

# Start server
python -m sglang.srt.server \
    --model meta-llama/Meta-Llama-3-8B-Instruct \
    --tp-size 1 \
    --radix-cache-size 2000000000

# Query
curl http://localhost:8000/generate \
  -d '{"text": "Hello", "sampling_params": {"max_new_tokens": 128}}'
```

## 24. Common Mistakes

1. **Not using `fork()` for parallel generations** — sequential generation negates SGLang's main advantage. Always use `fork` for independent generation branches.

2. **Overly complex regex schemas** — regex with nested groups and backtracking slows constrained decoding exponentially. Prefer JSON schema compilation over raw regex.

3. **Ignoring radix cache size** — default cache is small. For production, set `--radix-cache-size` to at least 2GB per GPU.

4. **Not canonicalizing inputs** — trailing whitespace or different newline characters create different prefixes. Always strip/normalize inputs before feeding to SGLang.

5. **Mixing DSL and non-DSL requests in same server** — dedicated servers for structured generation perform better than mixed workloads due to different scheduling needs.

6. **Forgetting to set temperature=0 for extraction** — extraction tasks need deterministic output. Non-zero temperature causes schema violations.

## 25. Production Best Practices

1. **Pre-warm the radix tree** — before accepting traffic, send a batch of synthetic requests that represent your most common system prompts, tool schemas, and few-shot examples. This ensures high cache hit rate from the start.

2. **Use `temperature=0` for all structured outputs** — extraction, classification, and JSON generation should be deterministic. Reserve non-zero temperature for creative generation.

3. **Normalize all inputs** — strip whitespace, canonicalize newlines, sort JSON keys in tool schemas to maximize prefix sharing.

4. **Monitor cache hit rate** — if it drops below 30%, investigate prefix diversity. Common causes: randomized few-shot examples, user-specific formatting, variable-length conversation histories.

5. **Set `mem_fraction_static` conservatively** — leave 12-15% headroom for radix tree metadata. The tree nodes themselves use memory beyond the KV blocks.

6. **Use structured logging** — SGLang's internal events (cache hits, evictions, compilation) are valuable for debugging. Log them to stdout and collect with Fluentd.

7. **Batch similar programs together** — if your application sends many similar extraction requests, batch them and submit as a single program with multiple parallel branches.

8. **Set request-level timeouts** — constrained decoding can get stuck if grammar is too restrictive. Set `max_new_tokens` with a safety margin.

9. **Pin radix cache in memory** — configure Kubernetes memory limits to account for radix cache size + model weights + activations. OOM kills cost cache warmup.

10. **A/B test cache hit rate impact** — compare latency between cold start and steady state to quantify the benefits of RadixAttention for your specific workload.
