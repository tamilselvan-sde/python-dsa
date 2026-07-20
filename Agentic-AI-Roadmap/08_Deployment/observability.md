# Observability for AI Systems

## 1. What is it?
Observability for AI systems is the practice of understanding AI system internals by analyzing telemetry data — metrics (quantitative measurements), logs (structured events), and traces (request journeys across services) — enabling teams to debug issues, optimize performance, and understand behavior without deploying new code.

## 2. Why do we need it?
AI systems are non-deterministic: the same input can produce different outputs. Traditional debugging (breakpoints, logging on error) fails when errors are semantic (wrong answer, hallucinations) rather than syntactic (500 error). Observability answers "why did the model respond that way?" by correlating prompt, context, retrieved documents, and model state.

## 3. Real-world Example
**Notion AI** observes every AI query across the three pillars: **Metrics** track latency p99 (target < 2s), token consumption per workspace, and cache hit ratio. **Logs** capture every prompt, retrieved context, and generated response (with PII redaction). **Traces** follow the request through query rewriting -> embedding -> vector search -> reranking -> LLM generation. When a user reports a bad answer, the trace shows exactly which documents were retrieved and how the prompt was constructed.

## 4. Architecture Diagram (ASCII)
```
+--------------------------------------------------+
|              Observability Stack                   |
|                                                    |
|  +--------+   +--------+   +--------+             |
|  |Metrics |   | Logs   |   | Traces |             |
|  |(Prom)  |   |(Loki)  |   |(Tempo) |             |
|  +----+---+   +---+----+   +----+---+             |
|       |           |            |                  |
|  +----v-----------v------------v---+              |
|  |         Grafana                 |              |
|  |  Dashboards | Explore | Alerts |              |
|  +----------------+----------------+              |
|                                                    |
|  +----------------+----------------+               |
|  | SLO Dashboard   | Service Graph  |              |
|  | Availability     | Dependencies   |              |
|  | Latency budget   | Latency heatmap|              |
|  | Error budget     | Error cascade  |              |
|  +-----------------+----------------+              |
+--------------------------------------------------+
           |                |               |
    +------v------+  +-----v------+  +-----v------+
    | AI Service  |  | AI Service |  | AI Service |
    | (LLM)       |  | (Retrieval)|  | (Embedding)|
    +------+------+  +-----+------+  +-----+------+
           |                |               |
      Instrumented with: OpenTelemetry SDK
      Exports: Metrics (/metrics), Logs (stdout JSON), Traces (OTLP)
```

## 5. Internal Working
Every service runs the OpenTelemetry SDK. It creates spans for each operation (embedding, retrieval, generation), attaches attributes (model name, token count, document IDs), and exports via OTLP to the collector. The collector batches, samples, and routes data: metrics to Prometheus, logs to Loki, traces to Tempo. Grafana correlates all three using trace IDs in logs and exemplars in metrics.

## 6. Production Flow
```
Request -> [Span: rag_query] -> Embedding [Span: embed] -> Retrieval [Span: search]
-> Rerank [Span: rerank] -> LLM [Span: generate] -> Response
  Each span: start_time, end_time, attributes (tokens, model, docs)
  Logs: structured JSON with trace_id, span_id
  Metrics: latency histograms, counters
```

## 7. HLD
| Pillar | Tool | Data | Query | Retention | Cost |
|--------|------|------|-------|-----------|------|
| Metrics | Prometheus + Thanos | Time-series (latency, tokens, errors) | PromQL | 30d (high-res), 1y (aggregated) | $200/mo |
| Logs | Loki + S3 | Structured JSON (prompts, responses) | LogQL | 15d hot, 1y cold | $400/mo |
| Traces | Tempo + S3 | Span trees (req journey) | TraceQL | 7d hot, 30d cold | $300/mo |
| Profiles | Pyroscope | CPU/memory profiles | Flame graphs | 7d | $100/mo |
| RUM | Grafana Faro | Frontend metrics | Explore | 30d | $50/mo |

## 8. LLD
```python
from opentelemetry import trace
from opentelemetry.sdk.trace import TracerProvider
from opentelemetry.sdk.trace.export import BatchSpanProcessor
from opentelemetry.exporter.otlp.proto.grpc.trace_exporter import OTLPSpanExporter
from opentelemetry.instrumentation.httpx import HTTPXClientInstrumentor

import structlog
import logging

# Configure structured logging
structlog.configure(
    processors=[
        structlog.stdlib.add_log_level,
        structlog.processors.TimeStamper(fmt="iso"),
        structlog.dev.ConsoleRenderer(),
    ],
    wrapper_class=structlog.stdlib.BoundLogger,
    context_class=dict,
    cache_logger_on_first_use=True,
)
logger = structlog.get_logger()

# Configure OpenTelemetry
provider = TracerProvider()
processor = BatchSpanProcessor(
    OTLPSpanExporter(endpoint="http://otel-collector:4317")
)
provider.add_span_processor(processor)
trace.set_tracer_provider(provider)

HTTPXClientInstrumentor().instrument()

tracer = trace.get_tracer(__name__)

class RAGWorkflow:
    async def query(self, user_query: str) -> dict:
        with tracer.start_as_current_span("rag_workflow") as root_span:
            root_span.set_attribute("query_length", len(user_query))
            root_span.set_attribute("user_id", user_id)

            logger.info("rag_query_started", query=user_query)

            # Embedding
            with tracer.start_as_current_span("embed") as embed_span:
                vector = await self.embed(user_query)
                embed_span.set_attribute("embedding_dim", len(vector))
                embed_time = time.monotonic()

            # Retrieval
            with tracer.start_as_current_span("retrieve") as retrieve_span:
                docs = await self.retrieve(vector, top_k=5)
                retrieve_span.set_attribute("docs_count", len(docs))
                retrieve_span.set_attribute("doc_ids", [d.id for d in docs])
                logger.info("docs_retrieved", count=len(docs), ids=[d.id for d in docs])

            # Generation
            with tracer.start_as_current_span("generate") as gen_span:
                response = await self.generate(docs, user_query)
                gen_span.set_attribute("response_length", len(response))
                gen_span.set_attribute("model", "gpt-4o")
                gen_span.set_attribute("tokens", response.usage.total_tokens)
                logger.info("llm_response",
                    model="gpt-4o",
                    tokens=response.usage.total_tokens,
                    response_length=len(response.text),
                )

            root_span.set_attribute("total_latency_ms",
                (time.monotonic() - embed_time) * 1000)

            return response
```

## 9. Python Implementation
```python
# OpenTelemetry collector configuration as Python
OTEL_COLLECTOR_CONFIG = """
receivers:
  otlp:
    protocols:
      grpc:
        endpoint: 0.0.0.0:4317
      http:
        endpoint: 0.0.0.0:4318

processors:
  batch:
    timeout: 1s
    send_batch_size: 1024
  attributes:
    actions:
      - key: environment
        value: production
        action: upsert
  filter:
    error_mode: ignore
    metrics:
      include:
        match_type: regexp
        metric_names:
          - ^llm_.*
          - ^retrieval_.*
          - ^rag_.*

exporters:
  prometheus:
    endpoint: 0.0.0.0:8889
    namespace: ai
  otlp:
    endpoint: tempo:4317
    tls:
      insecure: true
  loki:
    endpoint: http://loki:3100/loki/api/v1/push

service:
  pipelines:
    metrics:
      receivers: [otlp]
      processors: [batch, attributes, filter]
      exporters: [prometheus]
    traces:
      receivers: [otlp]
      processors: [batch, attributes]
      exporters: [otlp]
    logs:
      receivers: [otlp]
      processors: [batch, attributes]
      exporters: [loki]
"""
```

## 10. Folder Structure
```
observability/
  otel/
    collector-config.yaml
    instrumentation.py
  dashboards/
    rag-overview.json
    llm-performance.json
    cost-tracking.json
    service-graph.json
    slo-burn-rate.json
  alerts/
    latency-slo.yaml
    error-budget.yaml
    cost-anomaly.yaml
  exporters/
    custom-metrics.py
    log-enricher.py
  runbooks/
    high-latency.md
    error-spike.md
    cost-spike.md
```

## 11. Configuration
```yaml
# OpenTelemetry SDK configuration per service
instrumentation:
  service_name: rag-orchestrator
  service_version: 1.2.3
  exporter_otlp_endpoint: http://otel-collector:4317
  sampling:
    strategy: parent_based
    root_sampler:
      type: trace_id_ratio
      ratio: 0.1  # 10% of all root traces sampled
  resource_attributes:
    environment: production
    team: ai-platform
    region: us-east-1

log_level:
  structured: true
  format: json
  includes:
    - correlation_id
    - trace_id
    - span_id
```

## 12. Flowchart
```
Request
  |
  v
OpenTelemetry SDK creates root span
  |
  v
For each operation:
  Create child span
  Add attributes (model, tokens, docs)
  Log structured event with trace_id
  Export span to collector
  |
  v
Collector batches and routes:
  Metrics -> Prometheus -> Grafana
  Logs -> Loki -> Grafana
  Traces -> Tempo -> Grafana
  |
  v
Grafana correlates via:
  trace_id in logs to open Tempo trace
  exemplars in metrics to open Tempo trace
  |
  v
Engineers explore using:
  Grafana Explore (ad-hoc)
  Dashboards (pre-built)
  Alerts (automated)
```

## 13. Sequence Diagram
```
Service     OTEL SDK    Collector   Prometheus   Loki    Tempo   Grafana
  |            |            |            |         |       |        |
  |--create--->|            |            |         |       |        |
  |--span------|            |            |         |       |        |
  |            |            |            |         |       |        |
  |--export--->|            |            |         |       |        |
  |            |--batch---->|            |         |       |        |
  |            |            |--metrics---------------->|        |
  |            |            |--logs---------------------------->|        |
  |            |            |--traces--------------------------->|        |
  |            |            |            |         |       |        |
  |            |            |            |         |       |--query|
  |            |            |            |         |       |       |
  |            |            |            |<--data--|<--data|       |
  |            |            |            |         |       |       |
  |            |            |            |         |       |--show->
```

## 14. Pros
- Full request context across all services (correlation IDs); Debug non-deterministic behavior (see exactly what happened); Proactive detection of model degradation; Cost attribution per query/user; Reduced MTTR (minutes vs hours); SLO-based alerting prevents over-alerting.

## 15. Cons
- High data volume (traces are large); Storage cost scales with request volume; Instrumentation adds 5-10% latency overhead; Complex stack (5+ components); Requires dedicated observability infrastructure; Sampling can miss rare failures.

## 16. Alternatives
- **Datadog APM**: All-in-one, higher cost; **New Relic AI Monitoring**: AI-specific insights; **Grafana Cloud**: Managed OSS; **SigNoz**: Open-source APM; **Honeycomb**: High-cardinality observability; **Arize AI**: AI-specific observability.

## 17. Performance Considerations
- Trace sampling: 10-100% for low volume, 1-10% for high; Exemplars link metrics to traces without full tracing; Batch span export (1024/1s) reduces overhead; Async instrumentation (non-blocking); Profile sampling (1% of CPU time); Log volume growth with prompt size; Compress log payloads.

## 18. Scaling to Millions
- **10K req/day**: 10% trace sampling, single Grafana/Prometheus/Loki; **1M**: 1% sampling, Thanos for long-term metrics, Loki with S3; **10M**: Adaptive sampling (keep all errors, sample successes), multi-tenant OTel collector, dedicated trace storage; Use exemplars instead of full traces for high-volume services.

## 19. Failure Scenarios
- **OTel collector OOM**: Reduce batch size, increase sampling rate; **Trace storage full**: Shorten retention, increase sampling ratio; **Loki write path fail**: Transient write failures are ok (eventual consistency); **High cardinality metric explosion**: Add label allowlist; **Grafana crash**: Multiple replicas with shared PostgreSQL.

## 20. Security
- PII redaction before log/trace export; RBAC for Grafana dashboards and datasources; Encrypt all telemetry in transit; Audit access to trace/log data; Sensitive attributes filtered by OTel collector; Isolate observability stack in separate VPC; Retention limits for compliance.

## 21. Monitoring
- Monitor observability infrastructure health; OTel collector CPU/memory usage; Prometheus disk space and scrape failures; Loki ingestion rate and retention; Tempo span storage rate and query latency; Grafana active users and dashboard load time; Alert on observability pipeline degradation.

## 22. Interview Questions
1. *What is observability for AI systems?* — Three pillars: metrics, logs, traces — correlating them to understand AI system behavior.
2. *How do you trace a RAG query across services?* — OpenTelemetry spans with context propagation, trace ID in logs, exemplars in metrics.
3. *How to debug a hallucination in production?* — Trace shows retrieved documents, log shows prompt, compare to expected context.
4. *How to sample traces without losing error context?* — Head-based (random %) + tail-based (keep all errors).

## 23. Cheat Sheet
```
Three Pillars:
  Metrics: Rate, Errors, Duration (RED) + Cost, Tokens, GPU
  Logs: Structured JSON with trace_id, span_id
  Traces: Span tree with attributes and timing

Key Tools: OpenTelemetry (instrumentation), Grafana (visualization)
LGTM Stack: Loki (logs), Grafana, Tempo (traces), Mimir (metrics)
```

## 24. Common Mistakes
- Logging without structure (no JSON); No correlation IDs across services; 100% trace sampling at high volume; PII in logs and traces; Missing exemplars (metrics and traces not linked); Alert fatigue (too many, too sensitive); Not instrumenting critical paths; No SLO measurement; Ignoring observability costs.

## 25. Production Best Practices
- Start with structured logging (easiest, highest value); Add distributed tracing for critical paths; Use exemplars to link metrics + traces; Sample adaptively (keep all errors); Redact PII at the source; Set up SLO monitoring with burn-rate alerts; Correlate traces, logs, and metrics in dashboards; Document runbooks for common alert patterns; Budget for observability cost (5-10% of infra cost); Review sampling rates quarterly.
