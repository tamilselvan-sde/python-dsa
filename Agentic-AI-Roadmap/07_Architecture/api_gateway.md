# API Gateway for AI

## 1. What is it?
An API Gateway for AI systems is a reverse proxy between clients and AI backends, handling authentication, rate limiting, request routing, caching, prompt inspection, content filtering, and cost tracking — all AI-specific concerns a standard gateway does not address.

## 2. Why do we need it?
AI APIs face unique challenges: LLM costs per token require granular rate limiting, prompt injection must be detected before reaching the model, streaming requires special proxy support, and multi-model routing demands a unified endpoint. The gateway centralizes these cross-cutting concerns.

## 3. Real-world Example
**Anthropic's API Gateway** handles millions of requests/day: authenticates via API keys, applies per-tier rate limits (free 10 RPM, pro 100 RPM), inspects prompts for PII, routes to correct model version, streams responses, and logs every request for billing and abuse detection.

## 4. Architecture Diagram (ASCII)
```
                    +---------------------+
                    |   Clients           |
                    +----------+----------+
                               |
                               v
                    +---------------------+
                    |   CDN (CloudFront)   |
                    +----------+----------+
                               |
                               v
                    +---------------------+
                    |   API Gateway        |
                    |   (Kong/Envoy)       |
                    +----------+----------+
                               |
              +----------------+----------------+
              |                |                |
      +-------v----+   +------v------+  +------v------+
      |  Auth      |   |  Rate       |  |  Prompt     |
      |  Service   |   |  Limiter    |  |  Inspector  |
      +-------+----+   +------+------+  +------+------+
              |                |                |
              +----------------+----------------+
                               |
                    +----------v----------+
                    |   Model Router      |
                    +----------+----------+
                               |
              +----------------+----------------+
              |                |                |
      +-------v----+   +------v------+  +------v------+
      | OpenAI     |   |  Anthropic  |  |  Self-host  |
      +------------+   +-------------+  +-------------+
```

## 5. Internal Working
The gateway intercepts every request at layer 7. It authenticates the API key, checks rate limits (token-bucket per user/model), passes through prompt inspection (PII redaction, injection detection), selects the target model based on routing rules, proxies the request with streaming support, collects token usage, and caches responses.

## 6. Production Flow
```
Client -> CDN -> Gateway -> Auth -> Rate Limiter -> Prompt Inspector
-> Router -> Backend -> Streaming -> Usage Logging -> Cache -> Client
```

## 7. HLD
| Layer | Component | Tech |
|-------|-----------|------|
| Edge | CDN, DDoS protection | CloudFront/Cloudflare |
| Auth | API key validation, JWT | OAuth2 Proxy |
| Rate Limiting | Token bucket per user/model | Redis |
| Security | PII redaction, injection detection | Guardrails / custom |
| Routing | Model selection | Lua plugin / custom |
| Cache | Response caching | Redis |
| Observability | Usage logging, tracing | ELK / Loki |

## 8. LLD
```python
class APIGatewayHandler:
    def __init__(self):
        self.auth = AuthService()
        self.rate_limiter = RateLimiter()
        self.inspector = PromptInspector()
        self.router = ModelRouter()
        self.cache = ResponseCache()

    async def handle_request(self, request: Request) -> Response:
        api_key = request.headers.get("Authorization")
        user = await self.auth.validate(api_key)
        if not user:
            return Response(status_code=401)

        allowed = await self.rate_limiter.check(user.id, request.model)
        if not allowed:
            return Response(status_code=429)

        cleaned_prompt = await self.inspector.inspect(request.body)
        cache_key = self._cache_key(cleaned_prompt, request.model)
        cached = await self.cache.get(cache_key)
        if cached:
            return cached

        model = self.router.select(cleaned_prompt, user.plan)
        response = await self._proxy_to_model(model, cleaned_prompt)
        await self.cache.set(cache_key, response)
        await self.logger.log(user, model, response.usage)
        return response
```

## 9. Python Implementation
```python
from fastapi import FastAPI, Request
import time, re, hashlib

app = FastAPI()

class TokenBucketRateLimiter:
    def __init__(self, redis_url: str):
        self.redis = Redis.from_url(redis_url)
        self.limits = {"free": 10, "pro": 100, "enterprise": 10000}

    async def check(self, user_id: str, tier: str) -> bool:
        key = f"rate_limit:{user_id}"
        current = await self.redis.incr(key)
        if current == 1:
            await self.redis.expire(key, 60)
        return current <= self.limits.get(tier, 10)

class PromptInspector:
    def __init__(self):
        self.pii_patterns = [r"\b\d{16}\b", r"[^@]+@[^.]+\..+"]
        self.injection_patterns = ["ignore previous instructions"]

    async def inspect(self, prompt: str) -> str:
        for pattern in self.injection_patterns:
            if pattern.lower() in prompt.lower():
                raise HTTPException(400, "Prompt rejected")
        for pattern in self.pii_patterns:
            prompt = re.sub(pattern, "[REDACTED]", prompt)
        return prompt

class ModelRouter:
    def __init__(self):
        self.models = {
            "gpt-4o": "https://api.openai.com/v1/chat/completions",
            "gpt-4o-mini": "https://api.openai.com/v1/chat/completions",
        }
        self.costs = {"gpt-4o": 0.01, "gpt-4o-mini": 0.001}

    def select(self, prompt: str, plan: str) -> str:
        if plan == "free":
            return "gpt-4o-mini"
        return "gpt-4o"

@app.post("/v1/chat/completions")
async def chat_completion(request: Request):
    body = await request.json()
    user = request.state.user
    rate_limiter = TokenBucketRateLimiter("redis://redis:6379")
    inspector = PromptInspector()
    router = ModelRouter()

    allowed = await rate_limiter.check(user["id"], user["plan"])
    if not allowed:
        return JSONResponse(status_code=429, content={"error": "rate_limit"})

    cleaned = await inspector.inspect(str(body.get("messages", "")))
    model = router.select(cleaned, user["plan"])

    async with httpx.AsyncClient() as client:
        resp = await client.post(router.models[model], json=body, timeout=30)
    return resp.json()
```

## 10. Folder Structure
```
api-gateway/
  plugins/
    auth.py, rate_limiter.py, prompt_inspector.py, model_router.py, cache.py
  config/
    routes.yaml, models.yaml, limits.yaml
  middleware/
    logging.py, tracing.py
  storage/
    redis_client.py, usage_logger.py
  tests/
    test_gateway.py
  deploy/
    kong.yaml
```

## 11. Configuration
```yaml
routes:
  - path: /v1/chat/completions
    methods: [POST]
    plugins: [auth, rate_limiter, prompt_inspector, model_router]

rate_limits:
  free: { requests: 10/min, tokens: 10000/min }
  pro: { requests: 100/min, tokens: 100000/min }
  enterprise: { requests: 10000/min, tokens: 10000000/min }

models:
  gpt-4o: { provider: openai, cost: 0.01, max_tokens: 4096 }
  gpt-4o-mini: { provider: openai, cost: 0.001, max_tokens: 16384 }
```

## 12. Flowchart
```
Request -> CDN -> Gateway
  -> Auth: valid key? No -> 401
  -> Rate Limit: under quota? No -> 429
  -> Prompt Inspector: safe? No -> 400
  -> Cache Hit? Yes -> Return cached
  -> Model Router -> Proxy to Backend
  -> Stream Response -> Log Usage -> Cache Response
```

## 13. Sequence Diagram
```
Client    Gateway    Auth    RateLimiter  Inspector   Router   Backend   Cache
 |          |         |          |           |          |        |        |
 |--req---->|         |          |           |          |        |        |
 |          |--auth-->|          |           |          |        |        |
 |          |<--ok----|          |           |          |        |        |
 |          |--check----------->|           |          |        |        |
 |          |<--allowed----------|           |          |        |        |
 |          |--inspect--------------------->|          |        |        |
 |          |<--cleaned----------------------|          |        |        |
 |          |--cache check--------------------------->|        |        |
 |          |<--miss-----------------------------------|        |        |
 |          |--route-------------------------------------------->|        |
 |          |--proxy------------------------------------------------------>|
 |          |<--stream------------------------------------------------------|
 |          |--log----------------------->|          |        |        |
 |<--resp---|         |          |           |          |        |        |
```

## 14. Pros
- Single entry point for all AI APIs; Centralized auth, rate limiting, cost tracking; Protocol translation (REST to gRPC); Canary deployments via traffic routing; Caching reduces LLM costs.

## 15. Cons
- Single point of failure (mitigate with HA); Adds 5-15ms latency; Configuration complexity; Plugin ordering matters; Harder to debug through proxy.

## 16. Alternatives
- **Service mesh (Istio)**: Sidecar-based; **Client SDK**: No gateway latency; **K8s Ingress**: Simpler, limited AI features; **Custom NGINX**: Flexible.

## 17. Performance Considerations
- Stateless gateway for horizontal scaling; Redis rate limiter adds 1-2ms; Prompt inspection is CPU-bound; Cache hit rate 5-30% for identical prompts; Streaming needs tuned buffer sizes; Connection pooling to backends.

## 18. Scaling to Millions
- **1M req/day**: 2 pods, single Redis; **10M**: 10 pods, Redis Cluster; **100M**: 50 pods, multi-region GLB; Each request passes in <20ms (excluding backend).

## 19. Failure Scenarios
- **Redis down**: Allow without rate limiting; **Backend timeout**: Return 504, open circuit breaker; **Auth down**: Fallback to local JWT; **Cache storm**: Stampede protection with locking.

## 20. Security
- TLS 1.3 everywhere; API key rotation via Vault; Prompt injection detection; PII redaction at proxy; DDoS via CDN + rate limiting; IP allowlisting; Audit log all requests.

## 21. Monitoring
- Request rate per user/model/endpoint; P50/P99 latency; Error rate; Rate limit hit count; Cache hit ratio; Cost per user; Active API keys.

## 22. Interview Questions
1. *Design an API Gateway for AI.* — Auth, rate limiting, prompt inspection, model routing, caching, observability.
2. *How to rate limit by token count?* — Estimate from prompt length, track rolling window in Redis.
3. *How to support multiple LLM providers?* — Unified API format, router selects by cost/quality/availability.

## 23. Cheat Sheet
```
Gateway Pipeline: Auth -> Rate Limit -> Inspect -> Route -> Cache -> Proxy -> Log
Tools: Kong (Lua/Python), Envoy (WASM), Custom (FastAPI)
```

## 24. Common Mistakes
- Blocking on Redis call; Poor streaming handling; Hardcoded costs; No downstream retry logic; No plugin isolation.

## 25. Production Best Practices
- Run 2+ gateway nodes for HA; Connection pooling to backends; Pre-compute rate limit keys; Alert on 5xx > 1%; Canary routing for model deploys; Budget alerts per user.
