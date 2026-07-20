# Model Registry — Model Registry and Versioning

> Author: **Tamilselvan** · ✉️ tamilselvan.sde@gmail.com · 🔗 [LinkedIn](https://www.linkedin.com/in/tamilselvan-ai/)
>

## 1. What is it?
A Model Registry is a central catalog for managing ML model versions across their lifecycle. It stores model binaries, metadata (metrics, lineage, tags), stage transitions (Staging → Production → Archived), and deployment targets. Think of it as "Docker Hub for ML models" — but with semantic versioning, stage management, and deployment hooks.

## 2. Why do we need it?
Without a registry, ML teams struggle with:
- **Which model is in production?** — no source of truth
- **Can we rollback?** — no version history, no binary traceability
- **What data was this trained on?** — missing lineage
- **Who approved this deployment?** — no audit trail

A Model Registry provides governance, reproducibility, and deployment automation.

## 3. Real-world Example
**Spotify's** Model Registry (built on MLflow + internal tooling) manages 5,000+ registered models. A playlist recommendation model goes through: v1 (dev) → v2 (staging, A/B test) → v3 (production, 100% traffic). Each version logs: training dataset hash, evaluation metrics (precision@10, recall@10), feature schema, and deployment timestamp. When v3 causes a 2% engagement drop, the team auto-rolls back to v2 in 3 minutes.

## 4. Architecture Diagram (ASCII)

```
┌──────────────────────────────────────────────────────────────────────┐
│                         Model Registry                                │
│                                                                       │
│  ┌────────────┐  ┌────────────┐  ┌────────────┐  ┌────────────────┐ │
│  │ API Layer  │  │ Metadata   │  │ Storage    │  │ Integration    │ │
│  │            │  │ Store      │  │ Layer      │  │ Layer          │ │
│  │ REST/gRPC  │  │ Postgres   │  │ S3/GCS     │  │ CI/CD → Kafka  │ │
│  │ Auth       │  │            │  │ MinIO      │  │ Slack → Webhook│ │
│  │ Rate Limit │  │ Lineage    │  │ Glacier    │  │ PagerDuty      │ │
│  └────────────┘  └────────────┘  └────────────┘  └────────────────┘ │
│                        │             │                                │
└────────────────────────┼─────────────┼────────────────────────────────┘
                         │             │
                         ▼             ▼
               ┌──────────────────────────────┐
               │     Registered Model          │
               │  "fraud-detection-xgboost"    │
               │                               │
               │  ┌─ Version 1 ──── Stage: None         │
               │  ├─ Version 2 ──── Stage: Staging      │
               │  ├─ Version 3 ──── Stage: Production   │
               │  └─ Version 4 ──── Stage: Archived     │
               └──────────────────────────────┘
```

## 5. Internal Working
The registry is built on a metadata database + blob storage:

**Core entities:**
- **RegisteredModel**: Logical grouping (e.g., "fraud-detection-xgboost")
- **ModelVersion**: Immutable snapshot — version number, URI to artifact, creation time
- **Stage enumeration**: None → Staging → Production → Archived (ordered transitions)
- **Lineage**: Links to training run (MLflow run ID), dataset hash, source commit
- **Tags**: Key-value metadata (owner="ml-team", framework="xgboost")
- **Aliases**: Named pointers (e.g., "champion", "challenger")

**Transitions:**
```
None ──────▶ Staging ──────▶ Production ──────▶ Archived
  ▲                             │                    │
  │                             ▼                    │
  └────────────────────────── None (reject) ─────────┘
```

Each transition creates an audit event: `{user, timestamp, from_stage, to_stage, comment}`.

## 6. Production Flow
```
Train Pipeline
    │
    ▼
Create Model Version v42 ────▶ Stage: None
    │
    ▼
CI passes? ───YES──▶ Transition to Staging
    │
    ▼
Manual review or auto-gate
    │
    ▼
Transition to Production ────▶ Deploy to Serving
    │                              │
    │                              ▼
    │                         Monitor metrics
    │                              │
    ├── metrics OK? ───────────────┘
    │
    ├── metrics drop? ────▶ Rollback to v41 (Production)
    │                              │
    ▼                              ▼
Version v42 stays Prod        v41 re-deployed
```

## 7. HLD

```
┌────────────────────────────────────────────────────────────────────────┐
│                       Model Registry System                             │
│                                                                         │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │                       API Gateway                                 │   │
│  │  ┌────────────┐  ┌────────────┐  ┌────────────┐  ┌──────────┐  │   │
│  │  │ REST API   │  │ gRPC API   │  │ Auth (OIDC)│  │ Rate     │  │   │
│  │  │            │  │            │  │            │  │ Limiter  │  │   │
│  │  └────────────┘  └────────────┘  └────────────┘  └──────────┘  │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                                                                          │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │                     Core Service                                   │   │
│  │                                                                  │   │
│  │  ┌────────────┐  ┌────────────┐  ┌────────────┐  ┌──────────┐  │   │
│  │  │ Model CRUD │  │ Version    │  │ Stage      │  │ Lineage  │  │   │
│  │  │ Service    │  │ Manager    │  │ Controller │  │ Tracker  │  │   │
│  │  └────────────┘  └────────────┘  └────────────┘  └──────────┘  │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                                                                          │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │                  Data Layer                                       │   │
│  │                                                                  │   │
│  │  ┌──────────────────────────┐  ┌──────────────────────────────┐ │   │
│  │  │ Metadata DB (PostgreSQL) │  │ Artifact Store (S3/MinIO)    │ │   │
│  │  │                          │  │                              │ │   │
│  │  │ registered_models        │  │ models/fraud-detection/      │ │   │
│  │  │ model_versions           │  │   ├── v1/model.pkl           │ │   │
│  │  │ stage_transitions        │  │   ├── v2/model.pkl           │ │   │
│  │  │ aliases                  │  │   ├── v3/                    │ │   │
│  │  │ tags                     │  │   │   ├── model.pkl          │ │   │
│  │  └──────────────────────────┘  │   │   ├── preprocessor.pkl  │ │   │
│  │                               │   │   └── requirements.txt   │ │   │
│  │                               │   └──────────────────────────│ │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                                                                          │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │                  Integration Layer                                │   │
│  │  ┌────────────┐  ┌────────────┐  ┌────────────┐  ┌──────────┐  │   │
│  │  │ CI/CD      │  │Deploy      │  │ Monitoring │  │ Slack /  │  │   │
│  │  │ Webhook    │  │Controller  │  │ Alert      │  │ PagerDuty│  │   │
│  │  └────────────┘  └────────────┘  └────────────┘  └──────────┘  │   │
│  └─────────────────────────────────────────────────────────────────┘   │
└────────────────────────────────────────────────────────────────────────┘
```

## 8. LLD

**Database Schema:**
```sql
CREATE TABLE registered_models (
    name VARCHAR(255) PRIMARY KEY,
    creation_time TIMESTAMP DEFAULT NOW(),
    last_updated_time TIMESTAMP DEFAULT NOW(),
    description TEXT,
    owner VARCHAR(100)
);

CREATE TABLE model_versions (
    id SERIAL PRIMARY KEY,
    model_name VARCHAR(255) REFERENCES registered_models(name),
    version INT NOT NULL,
    creation_time TIMESTAMP DEFAULT NOW(),
    stage VARCHAR(20) DEFAULT 'None'
        CHECK (stage IN ('None','Staging','Production','Archived')),
    artifact_uri VARCHAR(1024),
    run_id VARCHAR(64),
    source_commit VARCHAR(40),
    dataset_hash VARCHAR(64),
    model_format VARCHAR(50),  -- 'mlflow', 'tensorflow', 'onnx', 'pytorch'
    status VARCHAR(20) DEFAULT 'READY'
        CHECK (status IN ('PENDING','READY','FAILED')),
    UNIQUE(model_name, version)
);

CREATE TABLE stage_transitions (
    id SERIAL PRIMARY KEY,
    model_name VARCHAR(255),
    version INT,
    from_stage VARCHAR(20),
    to_stage VARCHAR(20),
    user_id VARCHAR(100),
    comment TEXT,
    transition_time TIMESTAMP DEFAULT NOW(),
    FOREIGN KEY (model_name, version)
        REFERENCES model_versions(model_name, version)
);

CREATE TABLE model_tags (
    model_name VARCHAR(255),
    version INT,
    key VARCHAR(100),
    value TEXT,
    PRIMARY KEY (model_name, version, key)
);

CREATE TABLE aliases (
    model_name VARCHAR(255),
    alias VARCHAR(100),
    version INT,
    PRIMARY KEY (model_name, alias),
    FOREIGN KEY (model_name, version)
        REFERENCES model_versions(model_name, version)
);
```

**API Contract:**
```
# Core APIs
GET    /api/models                                           # list registered models
POST   /api/models/{name}/versions                           # create new version
GET    /api/models/{name}/versions/{version}                 # get version details
PUT    /api/models/{name}/versions/{version}/transition      # change stage
GET    /api/models/{name}/versions/{version}/lineage         # get lineage
DELETE /api/models/{name}/versions/{version}                 # archive a version

# Deployment APIs
POST   /api/models/{name}/versions/{version}/deploy          # trigger deploy
GET    /api/models/{name}/versions/{version}/deployments     # list deployments

# Query APIs
GET    /api/models/{name}/versions?stage=Production          # get prod version
GET    /api/models/{name}/aliases/{alias}                    # resolve alias
```

## 9. Python Implementation

```python
# model_registry_client.py
from dataclasses import dataclass
from datetime import datetime
import requests

@dataclass
class ModelVersion:
    model_name: str
    version: int
    stage: str
    artifact_uri: str
    run_id: str
    source_commit: str
    metrics: dict

class ModelRegistryClient:
    def __init__(self, base_url: str, token: str):
        self.base_url = base_url
        self.session = requests.Session()
        self.session.headers.update({"Authorization": f"Bearer {token}"})

    def register_version(
        self, model_name: str, artifact_uri: str,
        run_id: str, metrics: dict, commit: str
    ) -> ModelVersion:
        resp = self.session.post(
            f"{self.base_url}/api/models/{model_name}/versions",
            json={
                "artifact_uri": artifact_uri,
                "run_id": run_id,
                "source_commit": commit,
                "metrics": metrics,
            },
        )
        resp.raise_for_status()
        data = resp.json()
        return ModelVersion(**data)

    def transition(
        self, model_name: str, version: int,
        to_stage: str, comment: str = ""
    ) -> dict:
        resp = self.session.put(
            f"{self.base_url}/api/models/{model_name}/versions/"
            f"{version}/transition",
            json={"to_stage": to_stage, "comment": comment},
        )
        resp.raise_for_status()
        return resp.json()

    def get_current_prod(self, model_name: str) -> ModelVersion:
        resp = self.session.get(
            f"{self.base_url}/api/models/{model_name}/versions",
            params={"stage": "Production"},
        )
        resp.raise_for_status()
        return ModelVersion(**resp.json())

# Usage in CD pipeline
client = ModelRegistryClient("https://registry.ml.example.com", token)

client.transition(
    "fraud-detection", version=42,
    to_stage="Production",
    comment="Accuracy improved 0.2%, latency unchanged",
)

prod = client.get_current_prod("fraud-detection")
print(f"Current production: v{prod.version} (deployed from run {prod.run_id})")
```

## 10. Folder Structure
```
registry_service/
├── api/
│   ├── main.py                   # FastAPI app
│   ├── routes/
│   │   ├── models.py             # CRUD endpoints
│   │   ├── versions.py           # Version management
│   │   ├── transitions.py        # Stage transitions
│   │   └── deployments.py        # Deployment triggers
│   └── middleware/
│       ├── auth.py               # OIDC authentication
│       └── audit.py             # Audit logging
├── core/
│   ├── models.py                 # SQLAlchemy models
│   ├── registry.py               # Business logic
│   ├── storage.py                # S3 artifact operations
│   └── hooks.py                  # Webhook dispatcher
├── migrations/
│   └── versions/
├── tests/
│   ├── test_registry.py
│   └── test_transitions.py
├── Dockerfile
├── docker-compose.yaml
└── requirements.txt
```

## 11. Configuration
```yaml
# config.yaml
server:
  port: 8080
  workers: 4
  read_timeout: 30s

database:
  url: postgresql://registry:pass@postgres:5432/registry
  pool_size: 20
  max_overflow: 10

storage:
  s3_bucket: ml-models-prod
  s3_prefix: registry/
  region: us-east-1
  encryption: aws:kms
  kms_key_id: arn:aws:kms:...

stages:
  production:
    require_approval: true
    min_versions_to_keep: 5
    auto_rollback_timeout: 30m

notifications:
  on_transition:
    - channel: slack
      webhook: https://hooks.slack.com/...
    - channel: pagerduty
      routing_key: ${PAGERDUTY_KEY}

audit:
  retention_days: 365
  export_to_s3: true
```

## 12. Flowchart

```
    Training complete
         │
         ▼
    Register version
         │
         ▼
    ┌──────────────────┐
    │ Version created   │───Status: READY────▶ Auto-eval
    │ Stage: None       │                         │
    └──────────────────┘                    ┌─────┴─────┐
                                           ▼           ▼
                                     Metrics OK?   Metrics Bad?
                                         │             │
                                         ▼             ▼
                                   Transition     Mark as FAILED
                                   to Staging     (do not promote)
                                         │
                                         ▼
                                   ┌──────────────────┐
                                   │ Staging          │
                                   │ Deploy to shadow │
                                   └────────┬─────────┘
                                            │
                                    ┌───────┴───────┐
                                    ▼               ▼
                              Manual review?    Auto-approve?
                                    │               │
                                    ▼               ▼
                              Human approves    Metrics OK
                                    │               │
                                    └───────┬───────┘
                                            ▼
                                    Transition to Production
                                            │
                                            ▼
                                    Deploy to serving fleet
                                            │
                                            ▼
                                    Monitor 30min
                                     /          \
                                OK               Degraded
                                 │                 │
                                 ▼                 ▼
                            Done              Rollback to v(n-1)
                                              (Transition prev → Prod)
```

## 13. Sequence Diagram
```
Training Pipeline         Registry API            Metadata DB         Deploy Controller
     │                        │                       │                     │
     │──register v42─────────▶│                       │                     │
     │                        │──INSERT model_version─▶│                     │
     │◀──version=42, stage=None                        │                     │
     │                        │                       │                     │
     │──transition(Staging)──▶│                       │                     │
     │                        │──INSERT stage_event──▶│                     │
     │                        │──UPDATE stage────────▶│                     │
     │                        │──webhook: stage_changed───────────────────▶│
     │                        │                       │                     │
     │                        │                       │                     │──deploy(shadow)
     │                        │                       │                     │
CI/CD Gate (metrics check)    │                       │                     │
     │──transition(Prod)─────▶│                       │                     │
     │                        │──UPDATE stage────────▶│                     │
     │                        │──webhook─────────────▶│                     │──deploy(prod, canary 5%)
     │                        │                       │                     │
     │◀──Production: v42      │                       │                     │
     │                        │                       │                     │
Monitor (30min)               │                       │                     │
     │◀──metrics degraded─────│────────────────────────────────────────────│──rollback v41
     │                        │──webhook─────────────▶│                     │──deploy(v41, prod)
```

## 14. Pros
- **Single source of truth**: Everyone knows which model version is in production
- **Rollback in minutes**: One API call → previous version re-deployed
- **Audit trail**: Every stage transition logged with user + timestamp + reason
- **Lineage tracking**: Model → training run → data → code — full provenance
- **CI/CD integration**: Webhooks trigger automatic deployment on promotion
- **Version isolation**: Each version is an immutable artifact — no overwriting
- **Alias-based deploys**: `deploy --alias=champion` always picks the right version

## 15. Cons
- **Storage cost**: Keeping every version's binary (100MB+ each) adds up
- **Transition complexity**: Concurrent promotions can cause race conditions
- **No built-in A/B testing**: Registry manages versions but doesn't split traffic
- **Registry drift**: Production model may differ from registry if deployed outside pipeline
- **Cold start**: First-time deploy to new environment lacks cached model binary
- **Version numbering**: Sequential version numbers vs. semantic versions — both have tradeoffs

## 16. Alternatives
| Tool | Strength | Weakness |
|------|----------|----------|
| **MLflow Registry** | Open-source, integrated with Tracking | No A/B, basic RBAC |
| **SageMaker Model Registry** | Deep AWS integration, lineage | AWS-only |
| **Vertex AI Model Registry** | GCP-native, AutoML integration | GCP-only |
| **DVC** | Git-native, no DB needed | No registry UI |
| **Seldon Core** | K8s-native, canary deployments | Complex setup |
| **BentoML** | ML model serving + registry | Python-only |
| **Hugging Face Hub** | Model hub, community | Not enterprise-grade |

## 17. Performance Considerations
- **Metadata DB**: Index on `(model_name, version)`, `(model_name, stage)`; partition `stage_transitions` by month
- **Artifact downloads**: Cache frequently downloaded versions (latest Production) on CDN; S3 Transfer Acceleration
- **Version listing**: Use pagination (`limit=100, offset=0`); cache listing for popular models
- **Bulk operations**: Batch model registration (100 per request) to reduce DB round-trips
- **Storage**: 1000 versions × 200MB = 200GB; S3 lifecycle policy: archive versions >6 months to Glacier
- **DB cleanup**: Archive transition events >1 year to cold table; use `pg_partman` for table partitioning

## 18. Scaling to Millions
- **Shard by team**: Separate registry instances per ML domain (ads, search, recommendations)
- **Read replicas**: Postgres read replicas for listing queries; primary for writes/stage transitions
- **Artifact CDN**: CloudFront + S3 for global teams; signed URLs for access control
- **Version pruning**: Configurable retention: keep last N versions in production, archive rest
- **Message queue**: Use SQS/Kafka for webhook delivery — fan-out to CI/CD, Slack, monitoring
- **Alias as primary key**: Clients resolve `alias=champion` instead of querying by stage — avoids N+1
- **Caching layer**: Redis cache for `get_current_production()` — invalidated on stage transition

## 19. Failure Scenarios
- **Concurrent promotion race**: Two CI pipelines promote v42 and v43 simultaneously → both see "Production" → lost update. *Fix*: Use `SELECT ... FOR UPDATE` on stage transition or optimistic locking with version number.
- **Artifact corrupted on upload**: Model binary truncated. *Fix*: Compute and store MD5 checksum; verify before staging.
- **Rollback during active deployment**: Rollback request arrives while v42 is deploying. *Fix*: Queue rollback; wait for deploy to finish or abort.
- **DB outage during transition**: Promotion API fails after updating stage but before webhook fires. *Fix*: Two-phase: mark stage as "PENDING" → deploy → confirm + update to actual stage.
- **Accidental promotion to Production**: v13 (bad model) promoted to Prod by mistake. *Fix*: Immediate rollback via `transition(v12, Production)`; optional: soft-delete v13 from Production.

## 20. Security
- **Authentication**: OIDC (Keycloak, Okta) for API access; service account tokens for CI/CD
- **Authorization**: RBAC with three roles:
  - **Viewer**: Read models and versions
  - **Developer**: Register, transition to Staging
  - **Admin**: Transition to Production, delete versions, manage permissions
- **Artifact encryption**: S3 SSE-KMS for model binaries at rest; TLS 1.3 in transit
- **Audit logging**: CloudTrail for all API calls; `stage_transitions` table for promotions
- **Model signing**: Sign model binaries with KMS key; verify signature before deployment
- **Vulnerability scanning**: Scan model pickle for malicious code before promoting to Production
- **Approval gates**: Production promotion requires MFA-enabled approval from second person

## 21. Monitoring
- **Key metrics**: Version registration rate; promotion latency (None → Production); rollback frequency; storage consumed per model; API p99 latency
- **Alerts**: Rollback executed → page #ml-oncall; Production stage transition without CI approval → security alert; Storage >80% → scale S3 lifecycle; API p99 >500ms → auto-scale
- **Health**: Registry /health endpoint; DB connection pool usage; S3 read/write throughput
- **Dashboards**: Grafana panel showing model version timeline, stage transition graph, deployment status per model

## 22. Interview Questions
**Q1:** Design a model registry that handles 10,000 concurrent model promotion requests per minute.  
*A1:* Use optimistic locking on `stage` column. Each model gets a `lock_version` integer. On transition: `UPDATE model_versions SET stage = $1, lock_version = lock_version + 1 WHERE model_name = $2 AND version = $3 AND lock_version = $4`. If 0 rows updated, retry. Queue webhook notifications via SQS for async delivery.

**Q2:** How do you implement "champion/challenger" pattern in a model registry?  
*A2:* Use aliases: `champion` = current Production, `challenger` = current Staging. Deploy via aliases not version numbers. Auto-promote challenger to champion after A/B test passes. Rollback: set `champion` alias to previous version.

**Q3:** A model registry shows Production version is v10, but the serving infrastructure is running v8. How do you detect and fix this drift?  
*A3:* Deploy controller reports back to registry: `POST /deployments { version: 8, status: "running" }`. Registry compares `deployments.version` vs `current_production_stage`. Alert if mismatch > 1 hour. Fix: redeploy the registry-indicated version.

## 23. Cheat Sheet
```bash
# MLflow Registry
mlflow models --help
mlflow register_model -m runs:/run_id/model -n fraud-detection
mlflow models serve -m models:/fraud-detection/Production -p 5001

# Python
client = MlflowClient()
client.transition_model_version_stage(
    "fraud-detection", 42, "Production"
)
client.get_model_version("fraud-detection", 42)

# API
curl -X POST /api/models/fraud-detection/versions \
  -d '{"artifact_uri": "s3://bucket/v42/"}'
curl -X PUT /api/models/fraud-detection/versions/42/transition \
  -d '{"to_stage": "Production"}'
```

## 24. Common Mistakes
1. **Manual stage transitions without CI check**: Someone promotes a model that hasn't passed evaluation
2. **Overwriting versions**: Re-using a version number — always immutable; use new version
3. **No rollback plan**: Promoting without testing rollback path → panic during incident
4. **Ignoring stale deployments**: Registry says Production=v10, infra runs v8 — silent drift
5. **Storing full model in DB**: Model binaries should go to blob storage, not postgres
6. **Not cleaning old versions**: 500 versions of a 500MB model = 250GB wasted storage
7. **Sequential version numbering across models**: Global version counter is an anti-pattern; version per model

## 25. Production Best Practices
- Models are immutable: once registered, never overwrite a version
- Always deploy via registry alias (`champion`, `challenger`) not version numbers
- Require MFA + second approval for Production promotion
- Automate rollback with a 5-minute observation window after deployment
- Store model lineage (run ID, dataset hash, commit SHA) as metadata on every version
- Archive versions >90 days to cold storage (S3 Glacier); retain metadata indefinitely
- Run model vulnerability scan (`model_hunter` / pickle scan) before Staging promotion
- Configure webhook on `stage_transition` to notify CI/CD, monitoring, and Slack
- Use registry health check pings from deployment infrastructure to detect drift
- Cascade rollback: if you rollback from v42 to v41, record the reason and invalidate v42's alias
