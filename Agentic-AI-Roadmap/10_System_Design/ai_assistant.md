# AI Assistant (Copilot-like) System Design

## 1. Requirements

### Functional Requirements
- Conversational AI with multi-turn context
- Code generation, explanation, review, and refactoring
- Context-aware — read open files, project structure, git history
- Execute commands and return results (sandboxed)
- File creation, editing, and diff preview
- Web search and API integration
- Image and document upload for context
- Multi-language support for code (Python, JS, TS, Go, Rust, Java, etc.)
- Context management — project-level memory, user preferences
- Session persistence and sync across devices
- Plugin/tool extensibility

### Non-Functional Requirements
- **Latency**: First token < 500ms, streaming must feel real-time
- **Availability**: 99.95%
- **Throughput**: 50K concurrent users
- **Code accuracy**: Generated code compiles > 80% of the time (internal benchmark)
- **Security**: Sandboxed execution — no host access
- **Context window**: Support up to 200K tokens
- **Compliance**: Code not stored for training without consent; SOC 2

## 2. Capacity Estimation

| Metric | Value |
|--------|-------|
| Concurrent users | 50,000 |
| Avg session length | 15 min |
| Avg messages per session | 30 |
| Avg tokens per message | 2,000 (in) + 800 (out) |
| Daily LLM tokens | 50K × 30 × 2,800 = ~4.2B tokens/day |
| Code execution requests | 100K/day |
| File operations | 500K/day |
| Context storage | 500 KB/user × 10M users = 5 TB |
| Session state (Redis) | 50 KB/session × 50K concurrent = 2.5 GB |
| Search index | 1 TB (code corpus, docs) |

## 3. High-Level Architecture

```
User Device (IDE / CLI / Web)
        │
        ▼
┌──────────────────┐
│   API Gateway     │  Auth, Rate Limit, WSS Upgrade
└────────┬─────────┘
         │
         ▼
┌──────────────────────────────────────────────────────────────┐
│                    Session Manager                            │
│  ┌────────────────┐  ┌────────────────┐  ┌──────────────┐   │
│  │ Connection Pool│  │ Context Manager│  │ History Store │   │
│  │ (WebSocket)    │  │ (Project+File) │  │ (Conversation)│   │
│  └────────────────┘  └────────────────┘  └──────────────┘   │
└──────────────────────────────────────────────────────────────┘
         │
         ▼
┌──────────────────────────────────────────────────────────────┐
│                    Orchestrator / Router                       │
│                                                                  │
│  Intent Classifier → Task Router → Response Generator          │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────────┐ │
│  │ Intent   │  │ Code     │  │ Explain  │  │ General      │ │
│  │ Router   │─►│ Handler  │─►│ Handler  │─►│ Chat Handler │ │
│  └──────────┘  └──────────┘  └──────────┘  └──────────────┘ │
└───────────────────────┬───────────────────────────────────────┘
                        │
                        ▼
┌───────────────────────────────────────────────────────────────┐
│                   LLM Inference Layer                          │
│  ┌──────────────────┐  ┌──────────────────┐  ┌─────────────┐ │
│  │ GPT-4o / Claude  │  │ Fine-tuned       │  │ Code LLM    │ │
│  │ (General)        │  │ (Code-optimized) │  │ (DeepSeek-  │ │
│  │                  │  │                  │  │  Coder)     │ │
│  └──────────────────┘  └──────────────────┘  └─────────────┘ │
│  ┌──────────────────┐  ┌──────────────────┐                   │
│  │ Embedding Model  │  │ Reranker         │                   │
│  │ (Code + Text)    │  │ (Cross-encoder)  │                   │
│  └──────────────────┘  └──────────────────┘                   │
└───────────────────────────────────────────────────────────────┘
         │
         ▼
┌───────────────────────────────────────────────────────────────┐
│                   Execution & Tool Layer                        │
│  ┌─────────────────────────────────────────────────────────┐  │
│  │  Code Sandbox (gVisor / Firecracker)                     │  │
│  │  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌────────────┐ │  │
│  │  │ Python   │ │ Node.js  │ │ Shell    │ │ File       │ │  │
│  │  │ Runner   │ │ Runner   │ │ Exec     │ │ System     │ │  │
│  │  └──────────┘ └──────────┘ └──────────┘ └────────────┘ │  │
│  └─────────────────────────────────────────────────────────┘  │
│  ┌────────────┐ ┌────────────┐ ┌────────────┐ ┌────────────┐ │
│  │ Web Search │ │ Git Reader │ │ Package DB │ │ Knowledge  │ │
│  │ (Bing/Serp)│ │ (Repo ctx) │ │ (pip/npm)  │ │ Base (RAG) │ │
│  └────────────┘ └────────────┘ └────────────┘ └────────────┘ │
└───────────────────────────────────────────────────────────────┘
         │
         ▼
┌───────────────────────────────────────────────────────────────┐
│                     Data Stores                                │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌───────────────┐ │
│  │ Redis    │  │ Postgres │  │ S3       │  │ Vector DB     │ │
│  │(Session) │  │(Users,   │  │(File     │  │ (Code Search) │ │
│  │(Cache)   │  │ History) │  │ Cache)   │  │               │ │
│  └──────────┘  └──────────┘  └──────────┘  └───────────────┘ │
│  ┌──────────┐  ┌──────────────────────┐                      │
│  │ Kafka    │  │ Prometheus + Grafana  │                      │
│  │(Events)  │  │ (Monitoring+Alerting) │                      │
│  └──────────┘  └──────────────────────┘                      │
└───────────────────────────────────────────────────────────────┘
```

## 4. Low-Level Design

### Core Components

```python
class Session:
    id: UUID
    user_id: UUID
    device_id: str
    project_context: ProjectContext
    conversation: ConversationWindow
    preferences: UserPreferences
    tokens_used: int
    created_at: datetime

class ProjectContext:
    root_dir: str
    language: str  # auto-detected
    files: list[OpenFile]  # currently open files
    git_branch: str
    git_diff: str | None
    dependencies: dict  # package.json, requirements.txt etc.
    workspace: FileTree  # abbreviated tree of project

class ConversationWindow:
    messages: deque[Message]  # max 200K tokens
    tokens: int
    def add(self, message: Message) -> None:
        while self.tokens + message.tokens > MAX_TOKENS:
            oldest = self.messages.popleft()
            self.tokens -= oldest.tokens
        self.messages.append(message)
        self.tokens += message.tokens

class CodeHandler:
    async def generate(context: ProjectContext, prompt: str) -> CodeResult:
        # 1. Extract relevant context (open files, selection, imports)
        # 2. Construct prompt with context + instructions
        # 3. LLM call with streaming
        # 4. Validate syntax (AST parse)
        # 5. Return diff + explanation

    async def explain(context, prompt):
        # 1. Parse selected code into AST
        # 2. Generate explanation with line-level annotations
        # 3. Return markdown explanation

    async def review(context, prompt):
        # 1. Lint code (ruff, eslint)
        # 2. Static analysis (type checking)
        # 3. LLM-based code review
        # 4. Aggregate results into review report
```

### Intent Classification & Routing

```python
class IntentRouter:
    # Lightweight classifier (fine-tuned DistilBERT or few-shot LLM)
    INTENTS = {
        "code_generate": 0.6,  # threshold
        "code_explain": 0.7,
        "code_review": 0.75,
        "code_refactor": 0.65,
        "debug_help": 0.7,
        "command_execute": 0.8,
        "web_search": 0.75,
        "general_chat": 0.5,
        "file_operation": 0.7,
        "project_question": 0.65,
    }

    async def route(self, message: str, context: ProjectContext) -> Intent:
        has_code_block = "```" in message
        selected_code = len(context.selection) > 0
        error_message = "error" in message.lower() or "exception" in message.lower()

        if error_message and selected_code:
            return "debug_help"
        if selected_code and not has_code_block:
            return "code_explain"
        if has_code_block and selected_code:
            return "code_review"
        # ... fall through to LLM intent classification
```

## 5. Database Schema

### PostgreSQL — User Data & History

```sql
CREATE TABLE users (
    id UUID PRIMARY KEY,
    email TEXT UNIQUE,
    api_key_hash TEXT,
    plan TEXT DEFAULT 'free',
    tokens_used_monthly INT DEFAULT 0,
    token_budget INT DEFAULT 1000000,
    preferences JSONB DEFAULT '{"theme": "light", "auto_suggest": true}',
    created_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE TABLE conversations (
    id UUID PRIMARY KEY,
    user_id UUID REFERENCES users(id),
    project_hash TEXT,  -- anonymized project identifier
    title TEXT,
    message_count INT DEFAULT 0,
    tokens_used INT DEFAULT 0,
    model TEXT DEFAULT 'gpt-4o',
    created_at TIMESTAMPTZ DEFAULT NOW(),
    updated_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE TABLE messages (
    id UUID PRIMARY KEY,
    conversation_id UUID REFERENCES conversations(id),
    role TEXT CHECK (role IN ('user', 'assistant', 'system', 'tool')),
    content TEXT NOT NULL,
    content_type TEXT DEFAULT 'text',  -- text | code_diff | error | image
    tokens INT DEFAULT 0,
    tool_calls JSONB,
    tool_results JSONB,
    latency_ms INT,
    model TEXT,
    created_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE TABLE code_executions (
    id UUID PRIMARY KEY,
    session_id UUID,
    user_id UUID,
    language TEXT,
    code_hash TEXT,  -- SHA-256 of code
    code TEXT,
    stdout TEXT,
    stderr TEXT,
    exit_code INT,
    execution_time_ms INT,
    sandbox_id TEXT,
    created_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE TABLE user_feedback (
    id UUID PRIMARY KEY,
    message_id UUID REFERENCES messages(id),
    user_id UUID,
    rating INT CHECK (rating BETWEEN 1 AND 5),
    comment TEXT,
    category TEXT,  -- accurate | wrong | helpful | hallucinated
    created_at TIMESTAMPTZ DEFAULT NOW()
);
```

### Redis — Session & Cache

```
session:{session_id} → Hash
  user_id
  device_id
  project_root_hash
  open_files: JSON (path, language, selection)
  context_window_tokens: 12450
  preferences: JSON
  ttl: 1800 (30 min idle)

context:{project_hash}:{file_path} → String (file content)
  ttl: 300 (5 min — invalidate on save)
```

### Vector DB — Code Search Index

```python
# Pinecone index for semantic code search
index:
  name: "copilot-code-index"
  dimension: 768  # code-embedding model
  metric: cosine

metadata:
  - file_path: str
  - repo_hash: str
  - language: str
  - function_name: str
  - line_start: int
  - line_end: int
```

## 6. API Contract

### WebSocket — /ws/assist

```json
// Client → Server
{
  "type": "message",
  "payload": {
    "content": "Write a function to merge two sorted lists",
    "context": {
      "language": "python",
      "open_file": "src/sorting.py",
      "selection": {"start": {"line": 10, "char": 0}, "end": {"line": 12, "char": 0}},
      "git_diff": null
    }
  }
}

{
  "type": "file_change",
  "payload": {
    "path": "src/sorting.py",
    "content": "def merge(l1, l2):\n    ..."
  }
}

{
  "type": "execute",
  "payload": {
    "language": "python",
    "code": "print(merge([1,3], [2,4]))",
    "timeout_ms": 5000
  }
}

// Server → Client (streaming)
event: token
data: {"text": "Here's", "done": false}

event: token
data: {"text": " a merge function:", "done": false}

event: code_diff
data: {
  "path": "src/sorting.py",
  "diff": "@@ -10,3 +10,15 @@\n+def merge(l1: list, l2: list) -> list:\n+    i = j = 0\n+    result = []\n+    while i < len(l1) and j < len(l2):\n...",
  "explanation": "This function uses two-pointer technique..."
}

event: execution_result
data: {
  "stdout": "[1, 2, 3, 4]\n",
  "stderr": "",
  "exit_code": 0,
  "duration_ms": 45
}

event: done
data: {
  "message_id": "m-789",
  "tokens_used": 350,
  "latency_ms": 2400
}
```

### REST — Context Management

```json
// POST /api/v1/context/set-project
{
  "project_hash": "abc123",
  "language": "python",
  "dependencies": {"flask": "3.0", "pydantic": "2.0"},
  "file_tree": [
    {"path": "src/app.py", "language": "python", "size": 1024},
    {"path": "src/models.py", "language": "python", "size": 2048}
  ]
}

// GET /api/v1/conversations?limit=10
// Return list of recent conversations with summary
```

## 7. Sequence Diagram (Code Generation)

```
User        SessionMgr    IntentRouter   CodeHandler    LLM       Sandbox    Git
 │              │              │              │          │          │        │
 ├─msg─────────►              │              │          │          │        │
 │              ├─getContext──►              │          │          │        │
 │              │              ├─route──────►│          │          │        │
 │              │              │              │          │          │        │
 │              │              │              ├─readFile─►          │        │
 │              │              │              │          │          │        │
 │              │              │              ├─prompt───►          │        │
 │◄─stream───┬──┤              │              │◄─stream──┤          │        │
 │           │  │              │              │          │          │        │
 │           │  │              │              ├─validate─►          │        │
 │           │  │              │              │          │          │        │
 │           │  │              │              ├─execute──►          │        │
 │◄─result───┴──┤              │              │◄─result──┤          │        │
 │              │              │              │          │          │        │
 │              ├─saveHistory──►              │          │          │        │
 │◄─done────────┤              │              │          │          │        │
```

## 8. Scaling Strategy

| Component | Strategy |
|-----------|----------|
| Session Manager | Sharded by user_id hash (64 shards); sticky routing via API Gateway |
| Context Manager | LRU cache with TTL; project context loaded lazily |
| Intent Router | DistilBERT ONNX model — CPU inference, 10ms routing |
| LLM Inference | vLLM with tensor parallelism; prompt caching for system prompts |
| Code Sandbox | Firecracker micro-VM pool; pre-warmed, recycled every 100 executions |
| File Cache | Redis with project_hash:file_path key; invalidate on editor save event |
| History Search | Elasticsearch on messages.content with ngram analyzer for code |
| Token Budget | Per-user allowing; soft limit (warning) at 80%, hard stop at 100% |

## 9. Failure Handling

| Failure | Mitigation |
|---------|-----------|
| LLM unavailable | Failover to secondary model (e.g., GPT-4o → Claude 3.5); degraded caching for FAQ patterns |
| Sandbox execution timeout | Kill after timeout_ms + 2s grace; return partial output |
| Context overflow | Smart truncation — keep system prompt, last N turns, prune middle |
| Network partition on WebSocket | Reconnect with session_id; server replays last 2 messages |
| File read permission error | Transparent to LLM; return "Cannot access file" to user |
| Rate limit exceeded | Queue non-urgent requests (code review) while serving urgent (code generation) |
| Injection attack (prompt) | Guardrails pre-filter; reject and log |
| Code execution OOM | Sandbox memory limit 512 MB; kill and return "Code used too much memory" |

## 10. Monitoring

| Metric | Target | Alert |
|--------|--------|-------|
| First token latency | < 500ms P95 | > 1s → Investigate model serving |
| End-to-end streaming | < 5s for 500 tokens | > 10s → Scale LLM pods |
| Code execution success rate | > 95% | < 90% → Check sandbox health |
| Intent classification accuracy | > 90% | < 85% → Retrain classifier |
| Session reconnect rate | < 1% | > 5% → Check WebSocket infra |
| Feedback positive rate | > 75% | < 60% → Review model quality |
| Token usage per session | < 10K avg | > 50K → Investigate loops |
| Sandbox cold start | < 200ms | > 500ms → Increase pool size |

## 11. Security

- **Code sandbox**: Firecracker micro-VM with no network (unless explicitly allowed), no host FS access, 2 GB RAM limit, 5s CPU timeout. All packages pre-approved.
- **File system**: Scoped to project directory only. Path traversal detection (`../../etc/passwd`).
- **PII detection**: Presidio-based on both input and output. Mask API keys, tokens, passwords.
- **Network isolation**: Sandboxed code has no egress by default. Opt-in with user consent.
- **API key management**: Keys hashed with bcrypt; never logged or sent to LLM.
- **Secret scanning**: Tokenized content scanned before LLM send; reject if secrets detected.
- **Data retention**: Conversation logs retained 90 days (configurable). User can delete all data.

## 12. Trade-offs

| Decision | Alternative | Rationale |
|----------|-----------|-----------|
| vLLM (self-hosted) vs API | API = simpler ops | Self-hosted reduces latency 2x and cost 3x at scale |
| Firecracker vs Docker sandbox | Docker = faster startup | Firecracker = stronger isolation (VM boundary) |
| Streaming vs batch | Streaming = better UX | More WebSocket connection overhead |
| Fine-tuned intent router vs LLM | LLM = more accurate | Fine-tuned = 10ms vs 200ms, cheaper |
| Persistent context vs ephemeral | Ephemeral = simpler | Persistent = continuity across sessions |
| Client-side context vs server-side | Server-side = consistent | More memory on server; potential latency for large projects |

## 13. Interview Tips

- **Context management is the hardest problem**: Explain the sliding window, token budgeting, and smart truncation strategy. Mention that losing system prompt context is worse than losing conversation history.
- **Streaming architecture**: Show you understand SSE vs WebSocket, token-by-token generation, and why streaming is critical for UX in coding assistants.
- **Sandboxing is mandatory**: Never skip this. Explain the threat model — users can ask the assistant to run `rm -rf /`. The sandbox is not optional.
- **Multi-model routing**: Not every query needs GPT-4. Cheap models for simple questions, expensive models for complex code generation. Show cost-conscious design.
- **Code-specific optimizations**: Mention AST-level diff generation (not string diff), syntax validation before showing code, and function-level code search.
- **This is a hot interview topic**: "Design GitHub Copilot", "Design Cursor", "Design an AI pair programmer". Be ready for questions about context window optimization, latency, and sandboxing.
- **Evaluation**: How do you measure quality? Compilation rate, test pass rate, user acceptance rate. A/B testing with canary model deployments.
