# Production Deployment Checklist

> Author: **Tamilselvan** · ✉️ tamilselvan.sde@gmail.com · 🔗 [LinkedIn](https://www.linkedin.com/in/tamilselvan-ai/)
>


```markdown
## Pre-Deployment Checklist

### Infrastructure
- [ ] Sufficient RAM allocated (vectors × dim × 4 × 1.5)
- [ ] SSD storage with enough IOPS
- [ ] Network bandwidth ≥ 10 Gbps
- [ ] Multi-AZ deployment for HA
- [ ] Load balancer configured
- [ ] Monitoring stack installed (Prometheus + Grafana)

### Database Configuration
- [ ] Index type selected and benchmarked
- [ ] efSearch / nprobe tuned for latency target
- [ ] PQ compression enabled if memory constrained
- [ ] Replication factor configured
- [ ] WAL enabled for durability
- [ ] Backup strategy in place
- [ ] Vector dimension matches embedding model

### Embedding Pipeline
- [ ] Consistent model for index and query
- [ ] Embedding service deployed (warm)
- [ ] Batch size optimized
- [ ] Retry logic with exponential backoff
- [ ] Monitoring for embedding latency

### Query Pipeline
- [ ] Caching layer configured (Redis)
- [ ] Rate limiting in place
- [ ] Connection pool configured
- [ ] Timeout set per operation
- [ ] Fallback strategy if DB is down

### Testing
- [ ] Ground truth computed for recall validation
- [ ] Latency benchmarks meeting SLAs
- [ ] Load testing at 2x expected QPS
- [ ] A/B test infrastructure ready
- [ ] Rollback plan documented

### Monitoring
- [ ] p50/p95/p99 latency dashboard
- [ ] Recall monitoring dashboard
- [ ] Memory usage alert
- [ ] Error rate (>1% critical alert)
- [ ] Index build status monitoring
- [ ] Query logging with anonymization

### Security
- [ ] Tenant isolation tested
- [ ] API authentication configured
- [ ] TLS enabled for all connections
- [ ] Network policies restrict access
- [ ] Audit logging enabled
```

---

