# Monitoring for AI Systems

## 1. What is it?
Monitoring for AI systems collects, visualizes, and alerts on metrics from AI infrastructure — model performance (latency, throughput, token usage), system health (GPU utilization, memory), and business outcomes (cost per query, user satisfaction) — providing real-time visibility into production AI operations.

## 2. Why do we need it?
AI systems fail in unique ways: model drift reduces answer quality without crashing, GPU memory leaks build slowly, LLM API latency spikes under load, and token costs can explode silently. Traditional monitoring (CPU/memory pings) misses these. AI-specific monitoring catches quality and cost issues before they impact users.

## 3. Real-world Example
**GitHub Copilot** monitors every inference request across millions of users. Dashboard tracks: p50/p95/p99 autocomplete latency, acceptance rate (model quality proxy), GPU utilization per region, model version performance (canary vs stable), token cost per developer, and hallucination rate via user feedback. Alerts trigger when acceptance rate drops > 5% or latency exceeds 500ms p99.

## 4. Architecture Diagram (ASCII)
```
+--------------------------------------------------+
|                   Monitoring Stack                |
|                                                    |
|  +----------+  +----------+  +----------------+   |
|  | Metrics  |  | Logs     |  | Traces         |   |
|  | (Prom)   |  | (Loki)   |  | (Tempo)        |   |
|  +----+-----+  +-----+----+  +-------+--------+   |
|       |              |               |            |
|  +----v-----+  +-----v----+  +-------v--------+   |
|  | Grafana  |  | LogQL    |  | TraceQL        |   |
|  | Dash     |  | Explore  |  | Explore        |   |
|  +----------+  +----------+  +----------------+   |
|                                                    |
|  +----------+  +----------+  +----------------+   |
|  | Alerts   |  | SLO      |  | Cost           |   |
|  | (Alertmgr)|  | Calculator|  | (tracker)      |   |
|  +----------+  +----------+  +----------------+   |
+--------------------------------------------------+
                     |
    +----------------+----------------+
    |                |                |
+---v----+    +------v------+    +----v---+
| LLM    |    | Vector DB   |    | API    |
| Nodes  |    | (Qdrant)    |    | Gateway|
+--------+    +-------------+    +--------+
```

## 5. Internal Working
Every AI service exposes a `/metrics` endpoint in Prometheus format. Prometheus scrapes these every 15-30s. Metrics include: latency histograms, request counters, active requests, GPU utilization, token counts, and error rates. Loki collects structured logs (JSON) with correlation IDs. Tempo traces requests across services. Grafana visualizes all three with unified dashboards.

## 6. Production Flow
```
Request -> Gateway -> Orchestrator -> LLM -> Response
  |           |            |           |        |
  +--metric-->+--metric-->+--metric-->+->metric->+
  +--log----->+--log----->+--log----->+->log---->
  +--trace--------------------------------------->
```

## 7. HLD
| Storage | Metrics | Logs | Traces |
|---------|---------|------|--------|
| Tech | Prometheus + Thanos | Loki + S3 | Tempo + S3 |
| Retention | 30 days (1s), 1yr (5m) | 15 days | 7 days |
| Volume | 10GB/day | 50GB/day | 100GB/day |
| Query | PromQL | LogQL | TraceQL |
| Cost | $200/mo | $400/mo | $300/mo |

## 8. LLD
```python
# AI-specific metrics instrumentation
from prometheus_client import Histogram, Counter, Gauge
import time

class AIMetrics:
    _instance = None

    def __new__(cls):
        if not cls._instance:
            cls._instance = super().__new__(cls)
            cls._instance._init_metrics()
        return cls._instance

    def _init_metrics(self):
        self.llm_latency = Histogram(
            "llm_request_duration_seconds",
            "LLM request latency",
            buckets=[0.1, 0.5, 1.0, 2.0, 5.0, 10.0, 30.0],
            labelnames=["model", "endpoint"],
        )
        self.llm_tokens = Counter(
            "llm_tokens_total",
            "Total tokens consumed",
            labelnames=["model", "type"],  # type: prompt | completion
        )
        self.llm_errors = Counter(
            "llm_errors_total",
            "LLM API errors",
            labelnames=["model", "error_code"],
        )
        self.active_requests = Gauge(
            "ai_active_requests",
            "Concurrent active AI requests",
            labelnames=["service"],
        )
        self.cache_hits = Counter(
            "ai_cache_hits_total",
            "Semantic cache hits",
            labelnames=["cache_type"],
        )
        self.retrieval_latency = Histogram(
            "retrieval_duration_seconds",
            "Vector DB search latency",
            buckets=[0.01, 0.05, 0.1, 0.5, 1.0],
            labelnames=["collection"],
        )

    def instrument_llm_call(self, model: str):
        def decorator(func):
            async def wrapper(*args, **kwargs):
                start = time.monotonic()
                self.active_requests.labels(service="llm").inc()
                try:
                    result = await func(*args, **kwargs)
                    self.llm_latency.labels(model=model, endpoint="chat").observe(
                        time.monotonic() - start
                    )
                    if hasattr(result, "usage"):
                        self.llm_tokens.labels(model=model, type="prompt").inc(
                            result.usage.prompt_tokens
                        )
                        self.llm_tokens.labels(model=model, type="completion").inc(
                            result.usage.completion_tokens
                        )
                    return result
                except Exception as e:
                    self.llm_errors.labels(model=model, error_code=type(e).__name__).inc()
                    raise
                finally:
                    self.active_requests.labels(service="llm").dec()
            return wrapper
        return decorator

metrics = AIMetrics()
```

## 9. Python Implementation
```python
import structlog
from opentelemetry import trace
from opentelemetry.exporter.otlp.proto.grpc.trace_exporter import OTLPSpanExporter
from opentelemetry.sdk.trace import TracerProvider

# Structured logging
logger = structlog.get_logger()
logger.info("llm_request", model="gpt-4o", prompt_tokens=150, latency_ms=1200)

# OpenTelemetry tracing
provider = TracerProvider()
processor = BatchSpanProcessor(OTLPSpanExporter(endpoint="http://tempo:4317"))
provider.add_span_processor(processor)
trace.set_tracer_provider(provider)

tracer = trace.get_tracer(__name__)

async def process_query(query: str):
    with tracer.start_as_current_span("rag_query") as span:
        span.set_attribute("query_length", len(query))
        async with tracer.start_as_current_span("retrieve") as retrieve_span:
            docs = await retrieve(query)
            retrieve_span.set_attribute("docs_count", len(docs))
        async with tracer.start_as_current_span("generate") as gen_span:
            response = await generate(docs, query)
            gen_span.set_attribute("response_length", len(response))
        span.set_attribute("total_latency_ms", ...)
        return response
```

## 10. Folder Structure
```
monitoring/
  prometheus/
    prometheus.yml
    rules/
      ai_alerts.yml
      llm_recording.yml
    scrape_configs/
      ai_services.yml
  loki/
    loki-config.yaml
  tempo/
    tempo-config.yaml
  grafana/
    dashboards/
      llm_performance.json
      cost_tracking.json
      system_health.json
    datasources/
      prometheus.yaml
      loki.yaml
      tempo.yaml
  exporters/
    ai_metrics_exporter.py
    gpu_exporter.py
    custom_metrics.py
  alerts/
    alertmanager.yaml
    pagerduty_config.yaml
```

## 11. Configuration
```yaml
# prometheus/ai_alerts.yml
groups:
  - name: ai_alerts
    rules:
      - alert: HighLLMLatency
        expr: histogram_quantile(0.99, llm_request_duration_seconds_bucket) > 5
        for: 5m
        labels: { severity: critical }
        annotations:
          summary: "p99 LLM latency > 5s for 5 minutes"

      - alert: HighErrorRate
        expr: rate(llm_errors_total[5m]) / rate(llm_requests_total[5m]) > 0.05
        for: 2m
        labels: { severity: critical }
        annotations:
          summary: "LLM error rate > 5%"

      - alert: CostAnomaly
        expr: rate(llm_tokens_total[1h]) > 2 * avg(rate(llm_tokens_total[7d]))
        for: 30m
        labels: { severity: warning }
        annotations:
          summary: "Token usage 2x weekly average"

      - alert: HighGPUUtilization
        expr: avg(DCGM_FI_DEV_GPU_UTIL) > 90
        for: 10m
        labels: { severity: warning }
```

## 12. Flowchart
```
Service starts -> Exposes /metrics endpoint -> Prometheus scrapes (15s)
-> Stores in TSDB -> Grafana queries -> Dashboards updated
-> Alert rules evaluated -> If triggered -> Alertmanager -> PagerDuty/Slack

Log flow: Service stdout -> JSON lines -> Promtail -> Loki -> Grafana

Trace flow: Service -> OTLP exporter -> Tempo -> Grafana Explore
```

## 13. Sequence Diagram
```
Service     Prometheus   Grafana   Alertmanager   PagerDuty   Engineer
  |            |           |            |             |          |
  |--/metrics->|           |            |             |          |
  |            |--store----|            |             |          |
  |            |           |--query---->|             |          |
  |            |           |            |--eval rule  |          |
  |            |           |            |             |          |
  |            |           |            |--alert------>|          |
  |            |           |            |             |--notify->|
  |            |           |            |             |          |--ack
  |            |           |            |             |<--ack----|
```

## 14. Pros
- Real-time visibility into AI system health; Cost tracking per model per user; Quality monitoring via proxy metrics (acceptance rate, relevance); Alert on anomalies before users notice; SLO tracking for AI-specific metrics; Unified observability with traces.

## 15. Cons
- High cardinality (model, user, prompt) can overwhelm TSDB; Tracing overhead (5-10% latency); Storage cost for high-volume logs; Alert fatigue if thresholds not tuned; Custom metrics require code instrumentation; GPU metrics need DCGM exporter setup.

## 16. Alternatives
- **Datadog**: Managed, AI-specific APM; **New Relic**: AI monitoring built-in; **Grafana Cloud**: Managed OSS stack; **Lighstep**: High-cardinality tracing; **MLflow**: Model-centric monitoring; **WhyLabs**: AI observability focus.

## 17. Performance Considerations
- Keep metric cardinality under 100K series per Prometheus instance; Use recording rules for expensive queries; Sample traces (1% head-based sampling for high-traffic); Adjust scrape interval based on metric volatility; GPU metrics every 10s, business metrics every 30s; Use exemplars to link metrics and traces.

## 18. Scaling to Millions
- **10K req/min**: Single Prometheus + Loki, 30-day retention; **100K**: Prometheus HA + Thanos, Loki with S3 backend; **1M**: Multi-tenant Prometheus, cortex/mimir for long-term; Use aggregation before storage (1m → 5m rollups); Trace sampling (1-10% at high volume); Separate metrics by service domain.

## 19. Failure Scenarios
- **Prometheus OOM**: Reduce retention, increase scrape interval; **Loki write path fails**: Buffered writes to S3, retry; **High cardinality explosion**: Drop high-cardinality labels; **Grafana down**: Check PostgreSQL/alert data source; **Alertmanager silence missed**: Regex audit; **Tempo backpressure**: Reduce sampling rate.

## 20. Security
- Authenticate Grafana with OAuth/OIDC; RBAC for dashboards and datasources; Encrypt metrics in transit (TLS); Audit log of dashboard changes; Restrict /metrics endpoint to internal network; PII never in metric labels; Alert notifications via encrypted channels.

## 21. Monitoring
- Monitor the monitor: Prometheus disk usage, scrape failures; Loki disk usage, ingestion rate; Grafana user activity, error rate; Alertmanager silences, notification failures; Cost of monitoring infrastructure; Dashboard: Monitoring Health (green/yellow/red).

## 22. Interview Questions
1. *What metrics matter for LLM inference monitoring?* — Latency p50/p99, throughput, token usage, error rate, GPU utilization, cache hit ratio.
2. *How to detect model drift in production?* — Track output distribution (response length, sentiment), user feedback (thumbs up/down), acceptance rate.
3. *How to monitor cost per user in an AI system?* — Label metrics with user_id (low cardinality), use recording rules for daily cost aggregation, set budget alerts.
4. *How to set up distributed tracing for RAG?* — OpenTelemetry instrumentation at each step (embed, retrieve, generate), trace context propagation via headers.

## 23. Cheat Sheet
```
Core Metrics:
  LLM: latency, tokens, errors, active_requests
  Retrieval: latency, docs_retrieved, filter_selectivity
  Cost: tokens_per_hour, cost_per_model, cost_per_user
  System: GPU_util, GPU_memory, queue_depth, cache_hit_ratio

Tools: Prometheus (metrics), Loki (logs), Tempo (traces), Grafana (dashboards)
```

## 24. Common Mistakes
- Not instrumenting LLM calls (blind to cost); High-cardinality labels (user_id, prompt_hash) exploding TSDB; No GPU memory alerts (OOM kills silently); Not tracking cost anomalies; Alert threshold same for peak and off-peak; No trace sampling at high volume; Dashboard overload (too many panels).

## 25. Production Best Practices
- Start with RED metrics (Rate, Errors, Duration); Add cardinality limits on metric labels; Use recording rules for expensive queries; Set up SLOs for latency and accuracy; Implement structured logging with correlation IDs; Use trace sampling (head-based for low volume, tail-based for high); Create cost dashboard early; Alert on cost anomalies; Review metrics quarterly; Document runbooks for every alert.
