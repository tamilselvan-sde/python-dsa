# NGINX for AI Systems

## 1. What is it?
NGINX in AI systems serves as a high-performance reverse proxy, load balancer, and API gateway — handling TLS termination, request routing, rate limiting, caching, and serving static assets for AI web applications at the edge.

## 2. Why do we need it?
AI inference endpoints need protection from traffic spikes, DDoS, and abuse. NGINX provides production-grade traffic management with minimal overhead (< 1ms added latency), connection pooling to upstream LLM services, response buffering for streaming, and observability through access logs.

## 3. Real-world Example
**Hugging Face Inference API** uses NGINX as the entry point serving 100M+ requests/day. NGINX terminates TLS, rate-limits per API key (token bucket at the proxy level), caches model metadata responses, load-balances across 1000+ inference pods, and provides real-time metrics for the Hugging Face dashboard.

## 4. Architecture Diagram (ASCII)
```
                    +---------------------+
                    |   Internet          |
                    +----------+----------+
                               |
                               v
                    +---------------------+
                    |   NGINX (LB Layer)   |
                    |   TLS Termination   |
                    |   Rate Limiting     |
                    |   Request Routing   |
                    +----------+----------+
                               |
              +----------------+----------------+
              |                |                |
      +-------v----+   +------v------+  +------v------+
      |  AI API     |   |  Embedding  |  |  Monitoring |
      |  (vLLM)     |   |  Service    |  |  (Prometheus)|
      +------+------+   +------+------+  +------+------+
             |                   |                |
      +------v------+    +------v------+         |
      |  OpenAI API  |   |  Embedding  |         |
      |  Compatible  |   |  (GPU Pod)  |         |
      +-------------+    +-------------+         |
```

## 5. Internal Working
NGINX accepts client connections, terminates TLS, parses the request, applies rate limiting using shared memory zones, checks the cache (if enabled), then proxies the request to the upstream AI service. For streaming endpoints (SSE), NGINX uses buffering directives to handle the response without consuming excessive memory. Metrics are exported via the stub_status module.

## 6. Production Flow
```
Client -> NGINX -> TLS handshake -> Rate limit check -> Cache check
-> Proxy to upstream (vLLM/OpenAI) -> Buffer response -> Client
```

## 7. HLD
| Feature | Implementation | Purpose |
|---------|---------------|---------|
| TLS Termination | SSL certificate via cert-manager | Encrypt traffic |
| Rate Limiting | `limit_req_zone` per IP/key | Prevent abuse |
| Load Balancing | `upstream` block, least_conn | Distribute traffic |
| Request Routing | `location` blocks | Route to services |
| Caching | `proxy_cache` for GET | Reduce load |
| Streaming | `proxy_buffering off` for SSE | Real-time |
| Monitoring | `stub_status` + Prometheus | Metrics export |

## 8. LLD
```nginx
# nginx.conf - AI Gateway
upstream llm_backend {
    least_conn;
    server llm-service-0:8000 max_fails=3 fail_timeout=30s;
    server llm-service-1:8000 max_fails=3 fail_timeout=30s;
    keepalive 64;
}

upstream embedding_backend {
    random two least_conn;
    server embedding-service:8001;
}

limit_req_zone $binary_remote_addr zone=ai_api:10m rate=10r/s;
limit_req_zone $http_x_api_key zone=api_key:10m rate=100r/s;

proxy_cache_path /var/cache/nginx levels=1:2 keys_zone=api_cache:10m
                 max_size=10g inactive=60m;

server {
    listen 443 ssl http2;
    server_name ai.company.com;

    # TLS
    ssl_certificate /etc/nginx/certs/tls.crt;
    ssl_certificate_key /etc/nginx/certs/tls.key;
    ssl_protocols TLSv1.2 TLSv1.3;

    # Rate limiting for /v1/chat
    location /v1/chat/completions {
        limit_req zone=ai_api burst=20 nodelay;
        limit_req zone=api_key burst=50;

        proxy_pass http://llm_backend;
        proxy_http_version 1.1;
        proxy_set_header Connection "";

        # Streaming support
        proxy_set_header X-Request-ID $request_id;
        proxy_buffering off;
        proxy_cache off;
        proxy_read_timeout 120s;

        # Access log with timing
        access_log /var/log/nginx/ai_access.log ai_format;
    }

    # Cache for embedding (idempotent)
    location /v1/embeddings {
        limit_req zone=api_key burst=100;

        proxy_pass http://embedding_backend;
        proxy_cache api_cache;
        proxy_cache_key $request_body;
        proxy_cache_valid 200 1h;
        proxy_cache_use_stale error timeout;
        add_header X-Cache-Status $upstream_cache_status;
    }

    # Health
    location /health {
        access_log off;
        return 200 "OK";
    }

    # Metrics
    location /nginx_status {
        stub_status;
        access_log off;
        allow 10.0.0.0/8;
        deny all;
    }
}
```

## 9. Python Implementation
```python
# NGINX config generator for dynamic upstream management
import jinja2

NGINX_TEMPLATE = """
upstream {{ name }} {
    least_conn;
    {% for host in hosts %}
    server {{ host }} max_fails=3 fail_timeout=30s;
    {% endfor %}
}

location {{ path }} {
    proxy_pass http://{{ name }};
    proxy_http_version 1.1;
    proxy_set_header Connection "";
    proxy_buffering off;
}
"""

def generate_nginx_config(service_name: str, hosts: list[str], path: str) -> str:
    template = jinja2.Template(NGINX_TEMPLATE)
    return template.render(name=service_name, hosts=hosts, path=path)

# Example: Generate config for 3 LLM pods
config = generate_nginx_config(
    "llm-v2", [
        "10.0.1.5:8000", "10.0.1.6:8000", "10.0.1.7:8000"
    ],
    "/v2/chat/completions"
)
```

## 10. Folder Structure
```
nginx/
  conf.d/
    ai-gateway.conf         # Main AI routing config
    rate-limiting.conf      # Rate limit zones
    upstreams.conf          # Upstream definitions
    cache.conf              # Cache configuration
    logging.conf            # Custom log format
  ssl/
    certs/                  # TLS certificates
  cache/                    # Proxy cache directory
  html/                     # Static error pages
  docker/
    Dockerfile
    nginx.conf              # Main config
  k8s/
    configmap.yaml
    deployment.yaml
    service.yaml
```

## 11. Configuration
```yaml
# nginx-configmap.yaml for K8s
apiVersion: v1
kind: ConfigMap
metadata:
  name: nginx-ai-config
data:
  ai-gateway.conf: |
    limit_req_zone $binary_remote_addr zone=ai_api:10m rate=10r/s;
    limit_req_zone $http_x_api_key zone=api_key:10m rate=100r/s;

    server {
      listen 443 ssl;
      location /v1/chat/ {
        limit_req zone=ai_api burst=20;
        proxy_pass http://llm-service:8000;
        proxy_buffering off;
        proxy_read_timeout 120s;
      }
      location /v1/embeddings/ {
        proxy_pass http://embedding-service:8001;
        proxy_cache api_cache;
      }
    }
```

## 12. Flowchart
```
Client Request
  |
  v
NGINX: Accept connection
  |
  v
TLS Handshake (cert validation)
  |
  v
Rate Limit Check
  |        \
  |       (blocked -> 429)
  v
Cache Check (GET only)
  |        \
  |       (hit -> return cached)
  v
Select Upstream (least_conn / random)
  |
  v
Proxy Request
  |
  v
Wait for Upstream Response
  |
  +-> Streaming: buffer off, chunked transfer
  +-> Blocking: buffer on, aggregate response
  |
  v
Log Request (access log)
  |
  v
Response to Client
```

## 13. Sequence Diagram
```
Client     NGINX      Upstream_1   Upstream_2
  |         |             |            |
  |--TLS--->|             |            |
  |<--OK----|             |            |
  |         |             |            |
  |--POST--------------->|            |
  |         |--rate check|            |
  |         |--upstream sel           |
  |         |             |            |
  |         |--proxy----------------->|
  |         |             |--process   |
  |         |<--response--------------|
  |         |             |            |
  |<--resp--|             |            |
  |         |--log req    |            |
```

## 14. Pros
- Extremely high performance (100K+ concurrent connections); Low overhead (< 1ms per request); Mature and battle-tested; Rich module ecosystem (Lua scripting); Native streaming support; Small footprint (5MB per instance).

## 15. Cons
- Configuration reload requires signal (no hot-reload); Lua scripting for complex logic; No native service discovery (DNS resolution); Static upstream configuration (requires reload for changes); Limited observability (OpenTelemetry via third-party module).

## 16. Alternatives
- **Envoy**: Advanced L7, service mesh native; **Kong**: API Gateway with plugin ecosystem; **Traefik**: Dynamic config, auto-SSL; **HAProxy**: TCP optimization, simpler config; **Caddy**: Auto-HTTPS, simpler config; **AWS ALB**: Managed, auto-scaling.

## 17. Performance Considerations
- `worker_connections` = max concurrent connections per worker; `keepalive` connections reduce overhead (64-128); `proxy_buffering` for non-streaming endpoints; SSL session cache for faster handshakes; `sendfile` + `tcp_nopush` for static files; Rate limit shared memory zone size (`10m` = ~160K entries).

## 18. Scaling to Millions
- **10K req/s**: 2 NGINX workers, standard config; **100K**: 4-8 workers, keepalive 128; **1M**: 16+ workers, multiple NGINX instances behind cloud LB; Use multiple NGINX replicas in K8s with Service load balancing; Offload TLS to dedicated LB; Use HTTP/2 for multiplexing.

## 19. Failure Scenarios
- **Upstream failure**: `max_fails=3` marks server down, `proxy_next_upstream` retries; **Cache full**: LRU eviction via `proxy_cache_path` size; **Rate limit zone full**: Old entries evicted; **Worker crash**: OS restart policy, multiple workers for HA; **SSL cert expiry**: Auto-renewal with cert-manager.

## 20. Security
- TLS 1.2/1.3 only (disable old protocols); HSTS headers; Rate limiting per IP and API key; Buffer overflow protection (`client_body_buffer_size`); Hide NGINX version; Restricted access to /nginx_status; Request size limits; CORS headers for web apps.

## 21. Monitoring
- `stub_status`: Active connections, accepts, handled, requests; Custom log format with upstream timing; Prometheus NGINX Exporter for metrics; Grafana dashboard: requests/s, latency, upstream health, cache hit ratio; Alert on `5xx > 1%`, upstream failures, connection limit.

## 22. Interview Questions
1. *How does NGINX handle streaming responses?* — `proxy_buffering off;` disables buffering, sends chunks as received.
2. *How would you rate limit LLM API calls per user?* — `limit_req_zone $http_x_api_key` with per-key rate defined.
3. *How to handle TLS termination in NGINX?* — SSL certificate, `ssl_certificate` + `ssl_certificate_key`, protocols TLSv1.2+.
4. *How to cache LLM responses in NGINX?* — `proxy_cache` with request body as cache key (must be idempotent).

## 23. Cheat Sheet
```
Key Directives:
  worker_processes: CPU cores
  worker_connections: 4096 (per worker)
  keepalive: 64 (upstream connections)
  limit_req_zone: rate limiting
  proxy_cache: response caching
  proxy_buffering: off for streaming

Common Use:
  /v1/chat/* -> streaming, no cache, rate limited
  /v1/embeddings/* -> buffered, cache, rate limited
  /health -> no limit, no log
```

## 24. Common Mistakes
- Buffering streaming responses (memory blowup); No rate limiting on LLM endpoints (cost explosion); Small keepalive values (connection overhead); No health checks (routing to dead upstreams); Large proxy_buffers for unbounded responses; Mixed SSL and non-SSL on same server block; No access log format customization.

## 25. Production Best Practices
- Use multiple worker processes equal to CPU cores; Enable keepalive connections (64-128); Implement rate limiting at both IP and API key level; Disable proxy_buffering for streaming endpoints; Set proxy_read_timeout appropriately (60-120s for LLM); Use access log buffer and custom format; Monitor upstream health with passive checks; Rotate SSL certificates automatically; Run NGINX as non-root; Use ConfigMap for K8s deployments.
