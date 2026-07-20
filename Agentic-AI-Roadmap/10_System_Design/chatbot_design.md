# Enterprise Chatbot System Design

> Author: **Tamilselvan** · ✉️ tamilselvan.sde@gmail.com · 🔗 [LinkedIn](https://www.linkedin.com/in/tamilselvan-ai/)
>

## 1. Requirements

### Functional Requirements
- Multi-turn conversational AI with context retention
- Support for text, voice, images, and file attachments
- Slot-filling and intent recognition
- Human handoff escalation
- Multi-language support (50+ languages)
- Session persistence across devices
- Rich response types (cards, carousels, buttons, forms)
- Integration with external APIs (CRM, ticketing, payment)
- Sentiment analysis and tone adaptation
- Personalization based on user history and profile

### Non-Functional Requirements
- **Latency**: P99 < 500ms for text, < 2s for retrieval-augmented responses
- **Availability**: 99.99% uptime
- **Throughput**: 100K QPS peak
- **Consistency**: Eventual consistency for chat logs; strong for user state
- **Durability**: Zero message loss
- **Compliance**: SOC 2, HIPAA, GDPR, CCPA
- **Rate Limiting**: Per-user, per-tenant, per-IP

## 2. Capacity Estimation

### Traffic
- **Peak QPS**: 100,000
- **Avg conversation length**: 12 messages
- **Concurrent sessions**: 8M

### Storage
- **Message**: ~2KB per message (text), ~200KB (media)
- **Daily volume**: 100K QPS × 86400 × 2KB = ~17.3 TB/day (text only)
- **Conversation metadata**: ~500 bytes/conversation × 500M conv/day = ~250 GB/day
- **User profiles**: 1KB/user × 500M users = ~500 GB
- **Model cache**: ~2 TB for response cache

### Bandwidth
- **Ingress**: 17.3 TB/day ≈ 1.6 Gbps (text); + media = ~20 Gbps peak
- **Egress**: Symmetric + response generation tokens

## 3. High-Level Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                          CDN / CloudFront                        │
└─────────────────────────────────────────────────────────────────┘
                                │
                                ▼
┌──────────────────────────────────────────────────────────────────┐
│                       Load Balancer (ALB/GLB)                    │
│                    TLS termination, traffic routing               │
└──────────────────────────────────────────────────────────────────┘
                                │
                                ▼
        ┌────────────────────────────────────────────────────┐
        │                  API Gateway                        │
        │   Auth, Rate Limit, Throttle, Request Validation    │
        └──────────┬──────────────┬─────────────────┬────────┘
                   │              │                   │
        ┌──────────▼──┐  ┌───────▼──────────┐  ┌─────▼────────┐
        │ Chat Service │  │  Voice Service   │  │ Media Service│
        │  (WebSocket) │  │  (WebRTC/ASR)    │  │  (Upload)    │
        └──────┬───────┘  └───────┬──────────┘  └──────┬──────┘
               │                  │                      │
        ┌──────▼──────────────────▼──────────────────────▼──────┐
        │               Message Router / Orchestrator            │
        │        Intent Classifier → Dialogue Manager → NLG      │
        └──────┬──────────────────┬──────────────────────┬───────┘
               │                  │                      │
     ┌─────────▼───┐    ┌────────▼────────┐    ┌────────▼──────┐
     │  NLU Engine  │    │  Knowledge Base │    │  Response Gen │
     │  (BERT/GPT)  │    │  (Vector + SQL) │    │  (LLM + Temp) │
     └──────────────┘    └─────────────────┘    └───────────────┘
               │                  │                      │
               ▼                  ▼                      ▼
     ┌──────────────────────────────────────────────────────────┐
     │                  State & Data Stores                      │
     │  ┌────────┐  ┌──────────┐  ┌─────────┐  ┌─────────────┐  │
     │  │ Redis  │  │  PostgreSQL │  │ S3      │  │ Elasticsearch│
     │  │(Session) │  │(Users,Tenant)│  │(Media) │  │ (Chat Logs) │
     │  └────────┘  └──────────┘  └─────────┘  └─────────────┘  │
     │  ┌────────┐  ┌──────────────────────┐  ┌─────────────┐  │
     │  │ Kafka  │  │ Vector DB (Pinecone)  │  │ Prometheus  │  │
     │  │(Events) │  │ (Embeddings + Index)  │  │ (Metrics)   │  │
     │  └────────┘  └──────────────────────┘  └─────────────┘  │
     └──────────────────────────────────────────────────────────┘
```

## 4. Low-Level Design

### Component Breakdown

| Component | Responsibility | Tech Stack |
|-----------|---------------|------------|
| Chat Service | WebSocket management, session routing | Node.js/Python + FastAPI |
| Intent Classifier | Intent + entity extraction | Fine-tuned BERT, ONNX Runtime |
| Dialogue Manager | State tracking, policy, context window | Rasa/PyTorch + Redis |
| NLG Engine | Response generation | GPT-4o/Mistral, vLLM |
| Knowledge Retriever | Dense + sparse retrieval | LangChain + Pinecone |
| Human Handoff | Agent assignment, queue mgmt | Custom service + Salesforce |
| Sentiment Analyzer | Tone detection per message | DistilBERT, deployed on Triton |

### Key Classes / Interfaces

```python
class Message:
    id: UUID
    session_id: UUID
    user_id: UUID
    content: MessageContent  # text | image | voice | file
    timestamp: datetime
    metadata: dict

class Session:
    id: UUID
    user_id: UUID
    channel: ChannelType  # web | mobile | slack | whatsapp
    context: ConversationContext  # sliding window of last N turns
    state: DialogState  # active | waiting_handoff | closed
    expiry: timedelta = 30min

class IntentResult:
    intent: str
    entities: list[Entity]
    confidence: float
    sentiment: SentimentScore

class DialogPolicy:
    async def next_action(state: DialogState, intent: IntentResult) -> Action
    # Returns: Reply | API_Call | Form_Fill | Handoff | Fallback

class Response:
    text: str
    rich_content: list[Card | Carousel | Button | Form]
    suggested_actions: list[QuickReply]
    source: list[DocumentReference]  # for RAG responses
```

## 5. Database Schema

### PostgreSQL — Users & Tenants

```sql
CREATE TABLE tenants (
    id UUID PRIMARY KEY,
    name TEXT NOT NULL,
    config JSONB,           -- per-tenant NLU models, branding
    rate_limit_qps INT DEFAULT 1000,
    created_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE TABLE users (
    id UUID PRIMARY KEY,
    tenant_id UUID REFERENCES tenants(id),
    external_id TEXT,
    profile JSONB,          -- name, preferences, custom fields
    embedding VECTOR(384),  -- for personalization
    created_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE TABLE conversations (
    id UUID PRIMARY KEY,
    user_id UUID REFERENCES users(id),
    tenant_id UUID REFERENCES tenants(id),
    channel TEXT,
    status TEXT DEFAULT 'active',
    started_at TIMESTAMPTZ DEFAULT NOW(),
    ended_at TIMESTAMPTZ
);

CREATE TABLE messages (
    id UUID PRIMARY KEY,
    conversation_id UUID REFERENCES conversations(id),
    role TEXT CHECK (role IN ('user', 'assistant', 'system', 'agent')),
    content_type TEXT DEFAULT 'text',
    content TEXT,
    metadata JSONB,
    token_count INT,
    created_at TIMESTAMPTZ DEFAULT NOW()
);
```

### Vector DB — Knowledge Index

```python
# Pinecone Index Schema
index_name: "kb-index-v2"
dimension: 1536  # ada-002
metric: cosine
pods: 10 x p1x2

metadata:
  - chunk_id: str
  - doc_id: str
  - tenant_id: str
  - source_url: str
  - chunk_index: int
  - title: str
  - created_at: timestamp
```

### Redis — Session State

```
session:{session_id} → Hash
  context_window: JSON (last 10 turns)
  state: "active"|"escalated"
  current_intent: str
  slot_filling: JSON
  sentiment_trend: float[]
  last_activity_ts: timestamp
  ttl: 1800 (30 min)
```

## 6. API Contract

### POST /api/v1/chat/message

```json
// Request
{
  "session_id": "uuid-optional-for-new",
  "user_id": "uuid",
  "tenant_id": "uuid",
  "message": {
    "type": "text",
    "content": "What is my order status?"
  },
  "metadata": {
    "channel": "web",
    "timezone": "America/New_York"
  }
}

// Response
{
  "session_id": "uuid",
  "reply": {
    "type": "rich",
    "text": "Your order #ORD-1234 is out for delivery.",
    "cards": [
      {
        "title": "Order Details",
        "image_url": "https://cdn.example.com/order.png",
        "buttons": [
          {"label": "Track", "action": "url", "value": "/track/ORD-1234"},
          {"label": "Support", "action": "handoff", "value": "order"}
        ]
      }
    ],
    "suggested_replies": ["Track another order", "Return policy"],
    "source": [{"title": "Order DB", "confidence": 0.97}]
  },
  "intent": "order_status.check",
  "sentiment": "neutral",
  "latency_ms": 145
}
```

### WebSocket — /ws/chat/{session_id}

```
[Client → Server]
{
  "type": "message",
  "payload": { "content": "Hello" }
}
{
  "type": "typing",
  "payload": { "is_typing": true }
}

[Server → Client]
{
  "type": "response",
  "payload": { "text": "Hi! How can I help?" }
}
{
  "type": "suggestions",
  "payload": { "options": ["Order status", "Returns"] }
}
{
  "type": "handoff",
  "payload": { "agent_name": "John", "wait_time_s": 12 }
}
```

## 7. Sequence Diagram

```
User         ChatSvc      NLU        DM       KG        LLM        DB
 │              │          │          │        │         │          │
 ├─Message──────►          │          │        │         │          │
 │              ├─Parse────►          │        │         │          │
 │              │          ├─Intent──►│        │         │          │
 │              │          │          ├─Query──►         │          │
 │              │          │          │        ├─Results─┤          │
 │              │          │          ├─RAG────►         │          │
 │              │          │          │        │         ├─Generate─┤
 │              │          │          │        │         ├─Save────►│
 │◄──Response───┤          │          │        │         │          │
 │              │          │          │        │         │          │
```

## 8. Scaling Strategy

| Component | Strategy |
|-----------|----------|
| Chat Service | Horizontal pod autoscaler (CPU > 70%), WebSocket sticky sessions via Redis pub/sub |
| NLU Engine | Batch inference with ONNX Runtime, GPU-backed pods, model sharding |
| Knowledge Retriever | Read replicas for vector DB, caching for top-100 queries, ANN index rebuild nightly |
| LLM Generator | Tensor Parallelism across 2-4 GPUs, continuous batching (vLLM), speculative decoding |
| Session Store | Redis Cluster with 16 shards, Sentinel for HA |
| Message Queue | Kafka with 32 partitions, acks=all, min.insync.replicas=2 |
| Media | S3 + CloudFront CDN, presigned URLs for upload, async transcoding |

## 9. Failure Handling

| Failure | Mitigation |
|---------|-----------|
| LLM timeout | Fallback to template-based reply; retry with shorter prompt |
| Vector DB unavailable | Sparse retrieval (BM25) on Elasticsearch |
| Redis outage | Session state persisted to PostgreSQL; degraded experience |
| Kafka broker down | Messages buffered in local queue (Kafka client), replay on reconnect |
| Downstream API fails | Circuit breaker with fallback response; queue and retry |
| NLU confidence < 0.6 | Clarification prompt ("Did you mean...?") |
| Full system failure | Static fallback page with known-good responses |

## 10. Monitoring

| Metric | Tool | Alert Threshold |
|--------|------|-----------------|
| P50/P95/P99 latency | Prometheus + Grafana | P99 > 2s → PagerDuty |
| Error rate (5xx) | Datadog | > 1% → Slack alert |
| NLU confidence avg | Custom dashboard | < 0.7 in last 5 min |
| Handoff rate | Elasticsearch + Kibana | > 30% → review NLU |
| Session drop rate | Grafana | > 0.1% → investigate |
| Token throughput | vLLM metrics dashboard | Per-GPU > 90% → scale |
| Knowledge recall@5 | A/B evaluation pipeline | < 80% → reindex |

## 11. Security

- **TLS 1.3** for all in-transit communication
- **mTLS** between microservices
- **JWT** with short expiry (15 min) + refresh tokens
- **API keys** for tenant authentication, rotated every 90 days
- **Input sanitization**: PII redaction, prompt injection detection (Guardrails AI)
- **Rate limiting**: Token bucket per user (100/min), per tenant (10K/min)
- **Audit log**: All messages logged to immutable store (S3 + Glacier)
- **Data retention**: Configurable per tenant (default 90 days)

## 12. Trade-offs

| Decision | Pros | Cons |
|----------|------|------|
| WebSocket vs SSE | Bidirectional, lower latency | Harder to scale, stateful |
| Monolith NLU vs micro-NLU | Simpler to train | Cold-start for new intents |
| Dense + Sparse hybrid retrieval | Higher recall, robust to OOD | 2x infrastructure cost |
| vLLM vs TensorRT-LLM | Easier deployment | 5-10% throughput gap |
| Redis Cluster vs ElastiCache | No vendor lock | Operational overhead |
| Pre-computed embedding vs real-time | Faster inference | Staleness, storage cost |

## 13. Interview Tips

- **Start with scope**: Clarify channel (web vs mobile vs voice), language count, and critical integrations
- **Draw the full path**: Show you understand tokenization → intent → context → retrieval → generation → guardrails → response
- **State management is key**: Explain context window sizing, session expiry, and cross-device sync with Redis
- **Scale numbers**: Be ready with back-of-envelope — 100K QPS × 2KB = 200 MB/s ingress; 8M concurrent sessions = 64 GB RAM for Redis
- **Mention cost optimization**: Response caching for FAQ (80% cache hit), speculative decoding for 2x throughput, model quantization (FP16→INT8)
- **Human handoff**: This is a common gotcha — design the escalation protocol, agent queue, and context transfer
- **Compliance**: HIPAA requires audit trails + encryption at rest; GDPR requires data deletion APIs
- **Common system design question variants**: "Design WhatsApp chatbot", "Design customer support bot", "Design Slackbot for enterprise"
