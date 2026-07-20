# Event-Driven Architecture for AI

> Author: **Tamilselvan** · ✉️ tamilselvan.sde@gmail.com · 🔗 [LinkedIn](https://www.linkedin.com/in/tamilselvan-ai/)
>

## 1. What is it?
Event-Driven Architecture (EDA) for AI uses asynchronous event streams to decouple components — ingestion, embedding, indexing, inference — so they communicate through events rather than direct API calls. Each component produces and consumes events independently.

## 2. Why do we need it?
Synchronous AI pipelines create tight coupling: if embedding is slow, ingestion blocks. If the LLM is down, the entire pipeline stalls. EDA decouples components, buffers spikes via message queues, enables event replay for recovery, and allows each component to scale independently.

## 3. Real-world Example
**Uber's ML Pipeline** processes 100M+ events/day through Kafka. When a trip completes, an event triggers feature computation, model prediction, and result storage — all asynchronously. If prediction lags, events buffer. If it crashes, events replay from the last offset. Multiple ML teams consume the same events independently.

## 4. Architecture Diagram (ASCII)
```
+------------------+     +------------------+
|  Document Source  |     |   User Actions   |
+--------+---------+     +--------+---------+
         |                          |
         v                          v
+--------+---------+     +--------+---------+
|    Kafka           |     |    Kafka         |
|   raw-documents    |     |   user-queries   |
+--------+---------+     +--------+---------+
         |                          |
         v                          v
+--------+---------+     +--------+---------+
|  Ingestion Service |     |  Retrieval       |
|  (Chunk + Embed)   |     |  (Vector Search) |
+--------+---------+     +--------+---------+
         |                          |
         v                          v
+--------+---------+     +--------+---------+
|    Kafka           |     |    Kafka         |
|   embeddings       |     |   contexts       |
+--------+---------+     +--------+---------+
         |                          |
         v                          v
+--------+---------+     +--------+---------+
|  Indexing Service  |     |  Generation      |
|  (Vector DB Write) |     |  (LLM Call)     |
+--------------------+     +-----------------+
```

## 5. Internal Working
Events are immutable: DocumentUploaded, ChunkEmbedded, IndexUpdated, QueryReceived, ResponseGenerated. Each service subscribes to topics, processes events, and emits new events. The event log is the source of truth — any service reconstructs state by replaying the log. Kafka stores events with configurable retention.

## 6. Production Flow
```
Document Upload -> raw-documents -> Ingestion chunks + embeds
-> embeddings -> Indexing writes to vector DB -> IndexUpdated

User Query -> user-queries -> Retrieval (vector search) -> contexts
-> Generation (LLM) -> responses -> WebSocket/Webhook to user
```

## 7. HLD
| Topic | Producer | Consumer | Retention | Partitions |
|-------|----------|----------|-----------|------------|
| raw-documents | Upload API | Ingestion | 7 days | 8 |
| embeddings | Embedding | Indexing | 3 days | 16 |
| user-queries | Gateway | Retrieval | 24h | 32 |
| contexts | Retrieval | Generation | 24h | 32 |
| responses | Generation | Notifier | 24h | 32 |
| monitoring | All | Logstash | 30 days | 8 |

## 8. LLD
```python
@dataclass
class Event:
    id: str = field(default_factory=lambda: str(uuid.uuid4()))
    type: str = ""
    source: str = ""
    timestamp: str = field(default_factory=lambda: datetime.utcnow().isoformat())
    data: dict = field(default_factory=dict)
    correlation_id: str = ""

class EventBus:
    async def publish(self, topic: str, event: Event):
        await self.producer.send(topic, value=json.dumps(asdict(event)).encode())

class IngestionService:
    def __init__(self, event_bus: EventBus):
        self.event_bus = event_bus
        self.chunker = DocumentChunker()
        self.embedder = EmbeddingClient()

    async def handle_document_uploaded(self, event: Event):
        doc = event.data
        chunks = self.chunker.chunk(doc["content"])
        embeddings = await asyncio.gather(*[self.embedder.embed(c) for c in chunks])
        for chunk, embedding in zip(chunks, embeddings):
            await self.event_bus.publish("embeddings", Event(
                type="ChunkEmbedded",
                source="ingestion",
                data={"chunk": chunk, "embedding": embedding, "doc_id": doc["id"]},
                correlation_id=event.correlation_id,
            ))
```

## 9. Python Implementation
```python
from aiokafka import AIOKafkaConsumer
import asyncio, json

class AsyncEventProcessor:
    def __init__(self):
        self.handlers = {}

    def register(self, event_type: str, handler):
        self.handlers[event_type] = handler

    async def process_loop(self):
        consumer = AIOKafkaConsumer(
            "embeddings", "chunks", "monitoring",
            bootstrap_servers="localhost:9092",
            group_id="indexing-service",
        )
        await consumer.start()
        try:
            async for msg in consumer:
                event = json.loads(msg.value)
                handler = self.handlers.get(event["type"])
                if handler:
                    asyncio.create_task(handler(Event(**event)))
        finally:
            await consumer.stop()

class IndexingService:
    async def on_chunk_embedded(self, event: Event):
        await self.vector_db.upsert(
            collection="docs",
            point=event.data["embedding"],
            metadata={"doc_id": event.data["doc_id"]},
        )
        await self.event_bus.publish("monitoring", Event(
            type="IndexUpdated",
            source="indexing",
            data={"doc_id": event.data["doc_id"]},
        ))
```

## 10. Folder Structure
```
eda-platform/
  events/                 # Event schemas and types
  producers/              # Event emitters
    upload_api.py, gateway_events.py
  consumers/              # Event processors
    ingestion/, embedding/, indexing/, retrieval/, generation/
  infrastructure/
    kafka_cluster.py, schema_registry.py
  monitoring/
    event_lag.py, dead_letter.py
```

## 11. Configuration
```yaml
kafka:
  bootstrap_servers: kafka-1:9092,kafka-2:9092,kafka-3:9092
  replication_factor: 3

topics:
  raw-documents:
    partitions: 8
    retention: 7d
    cleanup: delete
  embeddings:
    partitions: 16
    retention: 3d
    cleanup: compact

consumers:
  ingestion:
    group_id: ingestion-group
    concurrency: 8
    max_poll_records: 500

dead_letter:
  topic: dead-letter
  max_retries: 3
```

## 12. Flowchart
```
Upload -> raw-documents topic
-> Ingestion consumer (chunk + embed)
-> embeddings topic
-> Indexing consumer (vector DB write)
-> monitoring topic
-> Metrics consumer (dashboards)

Query -> user-queries topic
-> Retrieval consumer (vector search)
-> contexts topic
-> Generation consumer (LLM call)
-> responses topic
-> Notifier -> User
```

## 13. Sequence Diagram
```
Upload   Kafka:raw   Ingestion   Kafka:emb   Indexing   Kafka:mon   Metrics
 |          |            |           |           |          |           |
 |--pub---->|            |           |           |          |           |
 |          |--poll----->|           |           |          |           |
 |          |            |--chunk----|           |          |           |
 |          |            |--embed----|           |          |           |
 |          |            |--pub------>|          |           |           |
 |          |            |           |--poll---->|          |           |
 |          |            |           |           |--upsert  |           |
 |          |            |           |           |--pub----->|           |
 |          |            |           |           |          |--log----->|
```

## 14. Pros
- Loose coupling (change one service independently); Natural load leveling; Event replay for recovery; Independent scaling; Immutable audit trail; Multi-consumer (multiple teams, same events).

## 15. Cons
- Eventual consistency; Debugging complexity; Schema evolution management; Higher infrastructure cost; Ordering challenges.

## 16. Alternatives
- **Sync HTTP/gRPC**: Simpler, lower latency; **RabbitMQ/SQS**: Better for point-to-point; **Redis Streams**: Lower latency, smaller scale; **EventBridge**: Managed.

## 17. Performance Considerations
- Partition count = max parallelism; Consumer lag indicates bottleneck; Batch writes to vector DB (1000 at once); Compaction keeps topics manageable; Schema registry adds ~2ms per message.

## 18. Scaling to Millions
- **10K ev/s**: 8-16 partitions per topic; **100K**: 32-64 partitions, 3-6 brokers; **1M**: 128+ partitions, multi-region; Use compression (snappy/zstd); Tiered storage for old events; Dead letter for failures.

## 19. Failure Scenarios
- **Consumer crash**: Kafka rebalances; **Broker failure**: In-sync replica takes over; **Poison pill**: Dead letter after 3 retries; **Consumer lag > threshold**: Alert + scale.

## 20. Security
- TLS for Kafka client-broker; SASL/SCRAM auth; ACL-based topic authorization; Schema registry auth; Encrypt sensitive fields; Network isolation.

## 21. Monitoring
- Consumer lag (primary health metric); Throughput (msg/s per topic); Error rate; Rebalance frequency; Disk usage; Dead letter queue size.

## 22. Interview Questions
1. *Design event-driven document ingestion.* — Upload -> Kafka -> chunking -> embedding -> indexing, each as separate consumers.
2. *How to handle event ordering?* — Partition by key, same partition = ordered.
3. *At-least-once vs exactly-once?* — At-least-once: may duplicate; exactly-once: idempotent + transactional.

## 23. Cheat Sheet
```
EDA Core: Event = immutable fact + timestamp; Topic = append-only log
Partition = shard (ordered); Consumer Group = parallel processing
Key Pattern: CQRS + Event Sourcing for AI pipelines
```

## 24. Common Mistakes
- Too many topics; No schema evolution plan; Ignoring consumer lag; No dead letter queue; Over-partitioning; Sharing groups across services.

## 25. Production Best Practices
- Design events as past tense facts; Schema registry from day one; Set retention (not infinite); Monitor consumer lag as primary alert; Idempotent consumers; Dead letter with alerting; Test rebalancing; Keep partitions at 2x max consumers.
