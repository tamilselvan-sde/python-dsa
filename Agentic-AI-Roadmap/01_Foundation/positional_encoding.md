# Positional Encoding

## 1. What is it?

**ELI5:** If you scramble the words of a sentence, you get nonsense. But self-attention treats "I love you" the same as "you love I" because it sees all words at once without order. Positional encoding is how we tell the model "these words were in this order" — we add a unique pattern to each word's representation that encodes where it sits in the sequence.

**Simple Explanation:** Positional encoding adds information about the position of each token in a sequence to its embedding vector. Since self-attention is permutation-invariant (it doesn't know order), positional encoding is how the Transformer understands sequence order. The original Transformer used sinusoidal functions at different frequencies to create unique position signatures.

**Technical Definition:** Positional encoding (PE) is a deterministic or learned function that maps position indices to d-dimensional vectors, which are added to token embeddings to provide positional information. The sinusoidal variant from Vaswani et al. (2017) uses:
```
PE(pos, 2i)   = sin(pos / 10000^(2i/d_model))
PE(pos, 2i+1) = cos(pos / 10000^(2i/d_model))
```
where `pos` is the token's position and `i` is the dimension index. These create a unique encoding for each position with properties that allow the model to easily learn relative positions.

## 2. Why do we need it?

**Problem It Solves:**
Self-attention computes weighted sums of token values based on content similarity, not position. Without positional information:
- "The dog bit the man" and "The man bit the dog" produce identical representations
- The model has no concept of word order, grammar, or sentence structure
- Language understanding requires knowing that "dog" comes before "bit" and "man" comes after

**Pain Without It:**
- **Bag-of-words behavior:** The model treats all permutations of a sentence identically
- **No grammatical understanding:** Subject-verb-object relationships require positional information
- **Impossible tasks:** "Find the third word" has no meaning without position
- **No sequence boundaries:** Model can't distinguish "The cat sat." from ".sat cat The"

**Why Companies Invest:**
- Positional encoding enables Transformers to match/beat RNNs on sequence tasks
- Sinusoidal PE allows extrapolation to longer sequences than trained on (tested up to 2x training length)
- No additional learned parameters → no extra training cost
- Deterministic encoding → consistent behavior across model initializations

## 3. Real-world Example

| Company | Model | PE Method | Details |
|---------|-------|-----------|---------|
| **Google** | Transformer (original) | Sinusoidal | Base for all modern NLP |
| **OpenAI** | GPT-1/2 | Learned absolute | Embedding lookup for each position |
| **Google** | BERT | Learned absolute | 512 max position embedding |
| **OpenAI** | GPT-3/4 | Learned absolute + RoPE | 2048 → 128K context |
| **Meta** | LLaMA | RoPE (Rotary) | Extends to 32K+ context |
| **Google** | ALiBi (Press et al.) | Linear bias | Test-time extrapolation to 2x+ |
| **Mistral** | Mistral 7B | RoPE with NTK scaling | 32K effective context |
| **Anthropic** | Claude | RoPE | 200K context window |

## 4. Architecture Diagram (ASCII)

```
                POSITIONAL ENCODING VISUALIZED

                Position 0    Position 1    Position 2    Position 3
                     │             │             │             │
                     ▼             ▼             ▼             ▼
Dimension 0:    sin(0)         sin(1)         sin(2)         sin(3)
                1.00           0.84           0.91          -0.14

Dimension 1:    cos(0)         cos(1)         cos(2)         cos(3)
                1.00          -0.54          -0.42          -0.99

Dimension 2:  sin(0/10000)  sin(1/10000)   sin(2/10000)   sin(3/10000)
                0.00          0.0001         0.0002         0.0003
     ...

    Each position gets a UNIQUE vector of length d_model
    ┌─────────────────────────────────────────────────────────┐
    │ Position 0: [1.00, 1.00, 0.00, 1.00, 0.00, 1.00, ...] │
    │ Position 1: [0.84, -0.54, 0.0001, 1.00, 0.00, 1.00]  │
    │ Position 2: [0.91, -0.42, 0.0002, 1.00, 0.00, 1.00]  │
    │ Position 3: [-0.14, -0.99, 0.0003, 1.00, 0.00, 1.00] │
    └─────────────────────────────────────────────────────────┘

    PE WAVES (different frequencies):
    dim=0: ▔▔▔▔╲╲╲╲╲▁▁▁▁▁▁╱╱╱╱╱▔▔▔▔  (fast oscillation)
    dim=1: ▔▔▔▔▔▔▔╲╲╲╲╲╲▁▁▁▁▁▁▁╱╱╱  (medium oscillation)  
    dim=2: ▔▔▔▔▔▔▔▔▔▔▔╲╲╲╲╲╲╲╲╲▁▁▁  (slow oscillation)
    dim=N: ╱╱╱╱╱╱╱╱╱╱╱╱╱╱╱╱╱╱╱╱╱╱  (very slow / constant)
```

## 5. Internal Working

**Step-by-step Sinusoidal Positional Encoding:**

**Step 1 — Define Position and Dimension Indices:**
- `pos` = integer position (0, 1, 2, ..., n-1)
- `i` = dimension index (0, 1, 2, ..., d_model-1)

**Step 2 — Compute Frequency for Each Dimension:**
```
frequency[i] = 1 / 10000^(2i/d_model)
```
- Dimension 0: frequency = 1/1 = 1.0 (fastest oscillation)
- Dimension d/2: frequency = 1/10000 ≈ 0.0001 (slowest oscillation)
- Geometric progression: wavelengths from 2π to 10000·2π

**Step 3 — Compute Sin and Cos Values:**
```
For each pos (0 to n-1):
  For each even dimension (2i):
    PE(pos, 2i) = sin(pos · frequency[i])
  For each odd dimension (2i+1):
    PE(pos, 2i+1) = cos(pos · frequency[i])
```

**Step 4 — Add to Token Embedding:**
```
final_embedding[pos] = token_embedding[pos] + PE[pos]
```
The positional signal is additive — the model must learn to use it.

**Properties of Sinusoidal PE:**
1. **Unique for each position:** No two positions have the same encoding
2. **Bounded values:** Each value ∈ [-1, 1], doesn't grow with sequence length
3. **Relative position learnable:** For any fixed offset k, PE(pos+k) can be represented as a linear function of PE(pos)
4. **Extrapolatable:** Can compute for positions beyond training maximum
5. **Deterministic:** No learned parameters, same across all models

## 6. Production Flow

```
┌──────────────┐
│  Token IDs   │
│  [464, 5432] │
└──────┬───────┘
       │
┌──────▼───────┐
│  Token       │
│  Embedding   │
│  Lookup      │
│  [2 × 512]   │
└──────┬───────┘
       │
┌──────▼───────┐
│  Position    │
│  Encoding    │
│  Compute     │
│  [2 × 512]   │
└──────┬───────┘
       │
┌──────▼───────┐
│  Add:        │
│  x = E + PE  │
│  [2 × 512]   │
└──────┬───────┘
       │
┌──────▼───────┐
│  Dropout     │
└──────┬───────┘
       │
       ▼
  Transformer Layers

Production with KV cache:
- PE is pre-computed for all positions up to max_seq_len
- Cached in GPU memory: shape [max_seq_len, d_model]
- For generation: PE for new positions is read from cache
- No recomputation needed per forward pass
```

## 7. HLD (High-Level Design)

```
┌─────────────────────────────────────────────────────────────────────┐
│                POSITIONAL ENCODING SYSTEM                           │
│                                                                     │
│  ┌────────────┐  ┌────────────┐  ┌────────────┐                    │
│  │ Tokenizer  │→ │ Embedding  │→ │ Positional │→ To Transformer   │
│  │            │  │ Table      │  │ Encoder    │                    │
│  └────────────┘  └────────────┘  └────────────┘                    │
│                                         │                           │
│                                    ┌────▼────┐                     │
│                                    │ PE Cache │                     │
│                                    │ [max_len │                     │
│                                    │ × d_model]                     │
│                                    └─────────┘                     │
│                                                                     │
│  Components:                                                        │
│  - PECache: Pre-computed sinusoidal values for all positions       │
│  - ExtrapolationModule: Handles positions beyond max trained        │
│  - ScalingModule: Adjusts PE magnitude for stable training          │
│  - RoPEAlternate: Switchable between sinusoidal and rotary          │
└─────────────────────────────────────────────────────────────────────┘
```

## 8. LLD (Low-Level Design)

```python
# positional_encoding.py — Production implementation
import torch
import torch.nn as nn
import math

class SinusoidalPositionalEncoding(nn.Module):
    """Sinusoidal positional encoding as in 'Attention Is All You Need'."""

    def __init__(self, d_model: int, max_len: int = 8192, dropout: float = 0.1):
        super().__init__()
        self.dropout = nn.Dropout(dropout)

        # Pre-compute PE matrix
        pe = torch.zeros(max_len, d_model)
        position = torch.arange(0, max_len, dtype=torch.float).unsqueeze(1)

        # Compute the frequency for each dimension pair
        div_term = torch.exp(
            torch.arange(0, d_model, 2).float()
            * (-math.log(10000.0) / d_model)
        )

        # Apply sin to even dimensions, cos to odd dimensions
        pe[:, 0::2] = torch.sin(position * div_term)  # even indices
        pe[:, 1::2] = torch.cos(position * div_term)  # odd indices

        pe = pe.unsqueeze(0)  # shape: (1, max_len, d_model)
        self.register_buffer("pe", pe, persistent=False)

    def forward(self, x: torch.Tensor) -> torch.Tensor:
        # x shape: (batch, seq_len, d_model)
        seq_len = x.size(1)
        # Add positional encoding to embedding
        x = x + self.pe[:, :seq_len, :].to(x.device)
        return self.dropout(x)


class LearnedPositionalEncoding(nn.Module):
    """Learned absolute positional encoding (used in BERT, GPT)."""

    def __init__(self, d_model: int, max_len: int = 512, dropout: float = 0.1):
        super().__init__()
        self.dropout = nn.Dropout(dropout)
        self.position_embeddings = nn.Embedding(max_len, d_model)

    def forward(self, x: torch.Tensor) -> torch.Tensor:
        seq_len = x.size(1)
        positions = torch.arange(seq_len, device=x.device).unsqueeze(0)
        pos_embeds = self.position_embeddings(positions)
        x = x + pos_embeds
        return self.dropout(x)


class RelativePositionalEncoding(nn.Module):
    """Relative position encoding (Transformer-XL style).

    Encodes relative distances between tokens rather than absolute positions.
    """

    def __init__(self, d_model: int, max_relative: int = 512):
        super().__init__()
        self.max_relative = max_relative
        # Embedding for relative position bias
        self.embedding = nn.Embedding(2 * max_relative + 1, d_model)

    def forward(self, seq_len: int, device: torch.device) -> torch.Tensor:
        # Create relative position matrix
        range_vec = torch.arange(seq_len, device=device)
        relative_positions = range_vec.unsqueeze(1) - range_vec.unsqueeze(0)
        # Clamp to [-max_relative, max_relative]
        relative_positions = torch.clamp(relative_positions, -self.max_relative, self.max_relative)
        # Shift to non-negative indices [0, 2*max_relative]
        relative_positions += self.max_relative
        # Lookup embeddings: (seq_len, seq_len, d_model)
        return self.embedding(relative_positions)


class PositionalEncodingManager:
    """Manager for different PE strategies, switchable at runtime."""

    def __init__(self, d_model: int, max_len: int = 8192, strategy: str = "sinusoidal"):
        self.d_model = d_model
        self.max_len = max_len
        self.strategy = strategy
        self._cache = {}

    def get_encoding(self, positions: torch.Tensor) -> torch.Tensor:
        """Get positional encoding for given positions."""
        key = f"{self.strategy}_{positions.shape}"
        if key not in self._cache:
            if self.strategy == "sinusoidal":
                encoding = self._sinusoidal(positions)
            elif self.strategy == "learned":
                encoding = self._learned(positions)
            else:
                raise ValueError(f"Unknown strategy: {self.strategy}")
            self._cache[key] = encoding
        return self._cache[key]

    def _sinusoidal(self, positions: torch.Tensor) -> torch.Tensor:
        pe = torch.zeros(*positions.shape, self.d_model)
        div_term = torch.exp(
            torch.arange(0, self.d_model, 2).float()
            * (-math.log(10000.0) / self.d_model)
        )
        pe[..., 0::2] = torch.sin(positions.float().unsqueeze(-1) * div_term)
        pe[..., 1::2] = torch.cos(positions.float().unsqueeze(-1) * div_term)
        return pe

    @torch.no_grad()
    def extend_to(self, new_max_len: int):
        """Extrapolate PE to longer sequences (sinusoidal only)."""
        if new_max_len <= self.max_len:
            return
        self._cache.clear()
        self.max_len = new_max_len
```

## 9. Python Implementation

```python
# pe_server.py — FastAPI service for positional encoding
import math
import torch
from fastapi import FastAPI, HTTPException
from pydantic import BaseModel, Field
from prometheus_client import Histogram
import time
import uuid

PE_LATENCY = Histogram("pe_compute_ms", "PE computation latency")

app = FastAPI(title="Positional Encoding Service", version="1.0.0")

class PERequest(BaseModel):
    positions: list[int]
    d_model: int = Field(default=512, le=8192)
    encoding_type: str = Field(default="sinusoidal", pattern="^(sinusoidal|learned)$")

class PEResponse(BaseModel):
    encoding: list[list[float]]
    latency_ms: float
    request_id: str

_PE_CACHE = {}

@app.post("/positional-encoding", response_model=PEResponse)
async def compute_pe(request: PERequest):
    start = time.perf_counter()
    request_id = str(uuid.uuid4())

    try:
        positions = torch.tensor(request.positions)
        d_model = request.d_model

        pe = torch.zeros(len(request.positions), d_model)
        div_term = torch.exp(
            torch.arange(0, d_model, 2).float()
            * (-math.log(10000.0) / d_model)
        )
        pe[:, 0::2] = torch.sin(positions.float().unsqueeze(-1) * div_term)
        pe[:, 1::2] = torch.cos(positions.float().unsqueeze(-1) * div_term)

        latency = (time.perf_counter() - start) * 1000
        PE_LATENCY.observe(latency)

        return PEResponse(
            encoding=pe.tolist(),
            latency_ms=round(latency, 2),
            request_id=request_id,
        )
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))
```

## 10. Folder Structure

```
positional-encoding/
├── api/
│   └── server.py
├── encoding/
│   ├── __init__.py
│   ├── sinusoidal.py      # Sinusoidal PE
│   ├── learned.py         # Learned absolute PE
│   ├── relative.py        # Relative position encoding
│   ├── rotary.py          # RoPE (rotary position embedding)
│   └── alibi.py           # ALiBi linear bias
├── cache/
│   └── pe_cache.py        # PE caching layer
├── tests/
│   ├── test_sinusoidal.py
│   └── test_extrapolation.py
└── config.yaml
```

## 11. Configuration

```yaml
positional_encoding:
  type: "sinusoidal"  # sinusoidal, learned, relative, rope, alibi
  d_model: 4096
  max_len: 8192
  dropout: 0.0  # Usually 0 for PE
  scale: 1.0  # Multiplier for PE
  sinusoidal:
    base: 10000.0  # Frequency base
  learned:
    share_across_layers: false
  extrapolation:
    method: "none"  # none, linear_interpolation, ntk
    ntk_alpha: 1.0  # NTK-aware scaling factor
```

## 12. Flowchart

```
                    ┌──────────────┐
                    │ pos: [0..n-1]│
                    └──────┬───────┘
                           │
                    ┌──────▼───────┐
                    │  i = 0..d/2  │
                    │  dim pairs   │
                    └──────┬───────┘
                           │
              ┌────────────┴────────────┐
              │                         │
         ┌────▼────┐              ┌─────▼─────┐
         │ Sin     │              │ Cos       │
         │(pos ×   │              │(pos ×     │
         │ freq[i])│              │ freq[i])  │
         └────┬────┘              └─────┬─────┘
              │                         │
         ┌────▼────┐              ┌─────▼─────┐
         │ dim=2i  │              │ dim=2i+1  │
         └─────────┘              └───────────┘
              │                         │
              └────────────┬────────────┘
                           │
                    ┌──────▼───────┐
                    │ PE Vector    │
                    │ [1 × d_model]│
                    └──────┬───────┘
                           │
                    ┌──────▼───────┐
                    │ Add to Token │
                    │ Embedding    │
                    │ x = E + PE   │
                    └──────────────┘
```

## 13. Sequence Diagram

```
Token Embedding          PE Generator              Transformer Layer
     │                       │                          │
     │── Input IDs ─────────►│                          │
     │                       │                          │
     │                       │── Compute PE for pos 0 ──►│
     │                       │── pos 0: [1.0, ...]      │
     │                       │                          │
     │                       │── Compute PE for pos 1 ──►│
     │                       │── pos 1: [0.84, ...]     │
     │                       │                          │
     │◄── E + PE ────────────│                          │
     │                       │                          │
     │                       │── Cache PE ─────────────►│
     │                       │                          │
     │── Next generation ───►│                          │
     │                       │── Read pos 2 from cache ─►│
     │◄── pos 2 PE ──────────│                          │
```

## 14. Pros

1. **Deterministic (sinusoidal):** No learned parameters, consistent behavior, no training cost.

2. **Extrapolation (sinusoidal):** Can compute PE for positions beyond training maximum. Tested up to 2x training length.

3. **Relative positioning:** The sinusoidal encoding allows the model to easily learn relative positions because PE(pos+k) is a linear function of PE(pos).

4. **Flexible:** Multiple variants (sinusoidal, learned, relative, rotary, ALiBi) for different use cases.

5. **Simple addition:** Just add to token embeddings. No architectural changes needed.

6. **Bounded values:** [-1, 1] range prevents embedding explosion at long sequences.

7. **No O(n²) cost:** O(n) computation and memory.

## 15. Cons

1. **Fixed resolution:** Same encoding for all models/datasets. Learned embeddings can adapt to data.

2. **Extrapolation limits:** Sinusoidal PE doesn't extrapolate well beyond ~2x training length. Quality degrades.

3. **Absolute vs relative disconnect:** Sinusoidal PE encodes absolute positions. For tasks where only relative order matters, this is wasteful.

4. **Additive interference:** Adding PE to embeddings can "smear" the embedding signal. Scale factor must be tuned.

5. **No length generalization:** Learned PE is fixed at training max_len. Cannot handle longer sequences.

6. **Interaction with subword tokens:** BPE/SentencePiece tokens have variable "length" in characters, but PE treats each token identically.

7. **Rotationally sensitive:** The sin/cos pairing assumes certain alignment that may not be optimal for all tasks.

## 16. Alternatives

| Method | Description | Pros | Cons | Used By |
|--------|-------------|------|------|---------|
| **Sinusoidal** | Fixed sin/cos frequencies | No params, extrapolatable | Fixed resolution | Original Transformer |
| **Learned absolute** | Embedding lookup per position | Adaptable to data | Can't extrapolate | BERT, GPT-2 |
| **Relative (T5)** | Learned relative position bias | Better for length generalization | More params | T5 |
| **RoPE** | Rotary position embedding | Relates to complex rotation | Slight compute overhead | LLaMA, GPT-4 |
| **ALiBi** | Linear bias on attention scores | Excellent extrapolation | Fixed slope | BLOOM, MPT |
| **NoPE (NoPE)** | No explicit position encoding | Model must infer from data | Limited to short sequences | Some small models |
| **xPos** | Extended RoPE | Better long-range decay | Complex | Some research models |

## 17. Performance Considerations

**Compute Cost:**
- Sinusoidal PE: O(n × d) → negligible (<1% of total compute)
- Learned PE: embedding lookup → near zero cost
- RoPE: requires complex multiplication → ~2% overhead per layer
- ALiBi: just adds bias to scores → <0.5% overhead

**Memory:**
- Pre-computed PE cache: max_len × d_model × 4 bytes (FP32)
- 8192 × 4096 × 4 = 128MB — tiny compared to model weights

**Optimization:**
- Pre-compute all PE values up to max_len at startup
- Cache on GPU memory — index directly during training/inference
- For very long contexts (>100K), compute on-the-fly or use ALiBi (no storage needed)

## 18. Scaling to Millions

**PE at Scale:**
- Pre-compute PE for maximum possible sequence length
- For extrapolation: compute PE on-the-fly for positions beyond cache
- In distributed training: compute PE once, broadcast to all GPUs

**Challenges at Extreme Scale:**
- 1M token context: full sinusoidal PE requires computing 1M × 4K values
- ALiBi is more practical: just a bias matrix, no per-position computation
- RoPE requires rotary computation per token — still O(n × d)

## 19. Failure Scenarios

| Failure | Symptom | Cause | Fix |
|---------|---------|-------|-----|
| **Position overflow** | Quality drops at >max_len | PE not computed for new positions | Use extrapolation/ALiBi |
| **PE too large** | Training unstable | PE magnitude >> embedding magnitude | Scale PE by 0.1-1.0 |
| **Extrapolation collapse** | Model outputs garbage | Sinusoidal PE beyond 2x training length | NTK-aware scaling |
| **Learned PE OOM** | Can't increase max_len beyond limit | Embedding matrix too large | Switch to sinusoidal/ALiBi |
| **Mixed precision error** | NaN in PE for large pos | FP16 underflow for very small div_term | Use BF16 or FP32 for PE |

## 20. Security

| Threat | Impact | Mitigation |
|--------|--------|------------|
| **Position injection** | Adversarial positions manipulated | Validate position lengths, no user-controlled pos |
| **Cache poisoning** | Corrupted PE cache | Immutable cache, hash validation |
| **Side-channel via position** | Extract sequence length from PE compute | Fixed compute time |

## 21. Monitoring

```yaml
metrics:
  - name: pe_max_value
    type: gauge
    help: "Maximum PE value (should be ≈1.0)"
  - name: position_overflow_count
    type: counter
    help: "Requests exceeding max trained sequence length"
  - name: pe_compute_latency_us
    type: histogram
    buckets: [1, 5, 10, 50, 100]

alerts:
  - condition: position_overflow_count > 0
    severity: info
    description: "Requests exceeding max trained length — consider increasing max_len"
```

## 22. Interview Questions

**Beginner:**
- Q: Why do Transformers need positional encoding?
  A: Self-attention is permutation-invariant (doesn't know order). PE adds sequence position information so the model can use word order.

- Q: What's the formula for sinusoidal PE?
  A: `PE(pos, 2i) = sin(pos/10000^(2i/d_model))`, `PE(pos, 2i+1) = cos(pos/10000^(2i/d_model))`

**Intermediate:**
- Q: Why use both sin and cos in alternating dimensions?
  A: Allows the model to learn relative positions because PE(pos+k) is a linear function of PE(pos) due to trig identities.

- Q: How does learned positional encoding differ from sinusoidal?
  A: Learned PE uses an embedding table (trainable) instead of a fixed formula. Adaptable to data but can't extrapolate beyond training length.

**Senior:**
- Q: Your model performs well at training length but degrades at longer sequences. What strategies can you use?
  A: (1) Switch from learned to sinusoidal PE. (2) Use ALiBi or RoPE which extrapolate better. (3) NTK-aware RoPE scaling. (4) Position interpolation (compress positions to fit within trained range). (5) Block-wise positional encoding.

- Q: Compare RoPE vs ALiBi for production LLMs.
  A: RoPE (Meta LLaMA) encodes position through rotation of Q/K vectors — integrates naturally with attention and provides good long-range decay. ALiBi (MosaicML MPT) adds direct bias to attention scores — simpler, excellent extrapolation (up to 32x training length), but doesn't work as well with learned absolute positions. Industry has converged on RoPE.

**Staff Engineer:**
- Q: Design a positional encoding scheme for multi-modal Transformers processing 100K frames of video.
  A: (1) Hierarchical: encode temporal (frame-level + micro-level timestamps). (2) RoPE with very high base frequency. (3) Relative position bias within chunks. (4) Global position encoding across chunks. (5) 3D position (x, y, t) for spatiotemporal data.

## 23. Cheat Sheet

```
┌─────────────────────────────────────────────────────────────────────┐
│                POSITIONAL ENCODING CHEAT SHEET                      │
├─────────────────────────────────────────────────────────────────────┤
│                                                                    │
│  SINUSOIDAL:                                                        │
│  PE(pos, 2i)   = sin(pos / 10000^(2i/d_model))                    │
│  PE(pos, 2i+1) = cos(pos / 10000^(2i/d_model))                    │
│                                                                    │
│  LEARNED: Embedding table with max_len rows                        │
│  ROPE: Rotate (q,k) by angle proportional to position             │
│  ALIBI: Linear bias added to attention scores                     │
│                                                                    │
│  QUICK REFERENCE:                                                   │
│  ┌─────────────────────────────────────────────────────┐           │
│  │ Short context (<512):   Learned is fine             │           │
│  │ General (512-8K):       Sinusoidal or RoPE          │           │
│  │ Long (>8K):             RoPE + NTK or ALiBi        │           │
│  │ Extreme (>128K):        ALiBi or position interp   │           │
│  └─────────────────────────────────────────────────────┘           │
│                                                                    │
│  COMPUTATION: O(n × d_model), negligible vs attention O(n² · d)    │
│  MEMORY: max_len × d_model × 4 bytes (FP32 cache)                 │
│                                                                    │
│  BEST PRACTICE: RoPE with NTK-aware scaling for modern LLMs        │
└─────────────────────────────────────────────────────────────────────┘
```

## 24. Common Mistakes

1. **Using learned PE when context varies widely:** Learned PE can't handle sequences longer than training length. Always use sinusoidal or RoPE if length varies.

2. **Not pre-computing PE:** Computing sin/cos on-the-fly in the training loop adds unnecessary GPU kernel launches.

3. **Too large PE magnitude:** If PE has magnitude ~100 and embeddings ~1, the positional signal dominates. Scale appropriately.

4. **Expecting perfect extrapolation:** Sinusoidal PE extrapolates ~2x at most. Beyond that, use RoPE, ALiBi, or position interpolation.

5. **Applying dropout after PE addition:** Dropout after PE+embedding can corrupt positional information. Use low dropout (0.1) or no dropout.

6. **Ignoring position in cross-attention:** Cross-attention also needs positional information. Either encoder positions propagate, or decoder positions used.

7. **Same PE across layers:** Each layer gets the same PE added to its input. This is intentional (reinforces position at every layer).

## 25. Production Best Practices

1. **Pre-compute PE at init:** Compute up to max_seq_len once and cache on GPU. O(1) during training/inference.

2. **Use RoPE for production LLMs:** It's the industry standard (LLaMA, Mistral, GPT-4) with good length generalization and minimal overhead.

3. **NTK-aware scaling for long context:** When deploying with extended context, use NTK-aware RoPE scaling (α > 1.0) instead of position interpolation — better quality.

4. **Scale PE appropriately:** Multiply sinusoidal PE by 1.0 for d_model ≤ 512, consider 0.1-0.5 for d_model ≥ 4096 to prevent embedding signal drowning.

5. **Test extrapolation before production:** Benchmark model quality at 1x, 2x, 4x training length. Know where quality drops.

6. **For encoder-only (BERT-style):** Learned PE is fine because sequence length is fixed. For decoder-only (GPT-style): RoPE or ALiBi is strongly preferred.

7. **Log position overflow warnings:** Track how many requests exceed training max_len. Plan model upgrades accordingly.

8. **Don't mix PE strategies:** Using RoPE in some layers and sinusoidal in others creates inconsistent positional signals.

9. **Consider rotary for multi-modal:** RoPE can encode 2D/3D positions by extending to multiple axes (x, y, time).

10. **ALiBi for extreme long context:** If deploying with 1M+ token context, ALiBi's O(n) computation and no position storage make it the practical choice.
