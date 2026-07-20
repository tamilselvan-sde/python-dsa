# MLflow — ML Platform

> Author: **Tamilselvan** · ✉️ tamilselvan.sde@gmail.com · 🔗 [LinkedIn](https://www.linkedin.com/in/tamilselvan-ai/)
>

## 1. What is it?
MLflow is an open-source platform for managing the ML lifecycle: experimentation, reproducibility, deployment, and a central model registry. It provides four components: Tracking (log params/metrics/artifacts), Projects (packaging), Models (format + serving), and Registry (versioned model store).

## 2. Why do we need it?
ML experiments produce thousands of runs. Without a tracking system, teams lose reproducibility, cannot compare experiments systematically, and deploy from ad-hoc artifacts. MLflow provides a single source of truth for every experiment, enabling audit trails, rollback, and collaboration.

## 3. Real-world Example
**Netflix** uses MLflow internally (via their Metaflow-inspired stack) to track 10,000+ daily experiments. A recommendation engineer trains a candidate-generation model, logs 50 hyperparameter combinations, compares them in the UI, registers the best run, and promotes it to staging — all without DevOps help.

## 4. Architecture Diagram (ASCII)

```
┌─────────────────────────────────────────────────────────────┐
│                      MLflow Client                           │
│  (Python SDK / CLI / REST API / UI)                          │
└──────────┬────────────────────────────────┬──────────────────┘
           │                                │
           ▼                                ▼
┌─────────────────────┐       ┌─────────────────────────────┐
│   Tracking Server    │       │     Model Registry           │
│  (REST API + UI)     │       │  (versioned model store)     │
│                      │       │                              │
│  ┌────────────────┐  │       │  ┌────────────────────────┐  │
│  │   Backend Store │  │       │  │  Registered Models     │  │
│  │  (SQLite/MySQL  │  │       │  │  Model Versions (v1,  │  │
│  │   /Postgres)    │  │       │  │   v2, v3)             │  │
│  └────────────────┘  │       │  │  Stages (Staging,      │  │
│  ┌────────────────┐  │       │  │   Production, Archived)│  │
│  │  Artifact Store  │  │       │  └────────────────────────┘  │
│  │ (S3/GCS/FS/NFS)  │  │       └─────────────────────────────┘
│  └────────────────┘  │
└─────────────────────┘
```

## 5. Internal Working
MLflow Tracking records **runs** inside **experiments**. Each run captures:
- **Parameters** (immutable: `lr=0.01`)
- **Metrics** (mutable: `accuracy=0.94`)
- **Tags** (metadata: `env=prod`)
- **Artifacts** (files: model.pkl, plots, confusion matrices)
- **Source** (git commit, entry point)
- **Start/End time**, **Status**

The Tracking Server writes to a backend store (relational DB for runs/params/metrics) and an artifact store (blob storage for files). The Model Registry uses the same backend store with additional tables for registered models, versions, and stage transitions.

## 6. Production Flow
```
Dev box ──mlflow run──▶ Tracking Server ──store──▶ Postgres + S3
                              │
                    ┌─────────┴─────────┐
                    ▼                   ▼
              MLflow UI            CI/CD Trigger
              (compare runs)       (best metric → register)
                                         │
                    ┌────────────────────┘
                    ▼
              Model Registry (v2 → Staging)
                    │
              ┌─────┴─────┐
              ▼           ▼
         Canary Eval   Batch Jobs
              │
         Promotion (Staging → Production)
              │
              ▼
         Production Serving
```

## 7. HLD

```
┌──────────────────────────────────────────────────────────────┐
│                        Clients                                │
│  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────────────┐ │
│  │ Python    │ │  CLI     │ │  REST    │ │  MLflow UI       │ │
│  │ Tracking  │ │  mlflow  │ │  API     │ │  (React)         │ │
│  └──────────┘ └──────────┘ └──────────┘ └──────────────────┘ │
└──────────────────────────┬───────────────────────────────────┘
                           │
┌──────────────────────────▼───────────────────────────────────┐
│                    API Gateway / LB                           │
│                 (Gunicorn + Nginx)                            │
└──────────────────────────┬───────────────────────────────────┘
                           │
┌──────────────────────────▼───────────────────────────────────┐
│                   Tracking Server                             │
│                                                               │
│  ┌────────────────────────┐  ┌───────────────────────────┐   │
│  │  Metadata Store        │  │  Artifact Store           │   │
│  │  (PostgreSQL RDS)      │  │  (S3 / GCS / MinIO)      │   │
│  │                        │  │                           │   │
│  │  experiments           │  │  └── runs/{exp}/{run}/    │   │
│  │  runs                  │  │      ├── model.pkl        │   │
│  │  params                │  │      ├── conda.yaml       │   │
│  │  metrics               │  │      └── plots/           │   │
│  │  tags                  │  │                           │   │
│  │  registered_models     │  └───────────────────────────┘   │
│  │  model_versions        │                                    │
│  └────────────────────────┘                                    │
└──────────────────────────────────────────────────────────────┘
```

## 8. LLD

**Database schema (simplified):**
```
experiments(experiment_id, name, artifact_location, lifecycle_stage)
runs(run_uuid, experiment_id, name, source_name, source_type,
     start_time, end_time, status, artifact_uri)
params(run_uuid, key, value)  — PK(run_uuid, key)
metrics(run_uuid, key, value, timestamp, step)  — PK(run_uuid, key, step)
tags(run_uuid, key, value)
registered_models(name, creation_time, last_updated_time)
model_versions(name, version, creation_time, stage, description)
```

**API endpoints:**
```
POST   /api/2.0/mlflow/runs/create
POST   /api/2.0/mlflow/runs/log-parameter
POST   /api/2.0/mlflow/runs/log-metric
POST   /api/2.0/mlflow/runs/log-artifact
GET    /api/2.0/mlflow/runs/get
POST   /api/2.0/mlflow/registered-models/create
POST   /api/2.0/mlflow/model-versions/create
POST   /api/2.0/mlflow/model-versions/transition-stage
```

## 9. Python Implementation

```python
import mlflow
from sklearn.ensemble import RandomForestClassifier
from sklearn.metrics import accuracy_score

mlflow.set_tracking_uri("http://mlflow.example.com:5000")
mlflow.set_experiment("fraud-detection-v2")

with mlflow.start_run() as run:
    mlflow.log_param("n_estimators", 200)
    mlflow.log_param("max_depth", 15)

    model = RandomForestClassifier(n_estimators=200, max_depth=15)
    model.fit(X_train, y_train)

    preds = model.predict(X_test)
    acc = accuracy_score(y_test, preds)
    mlflow.log_metric("accuracy", acc)

    mlflow.sklearn.log_model(model, "model")

    run_id = run.info.run_id
    mlflow.register_model(
        f"runs:/{run_id}/model",
        "fraud-detection-rf"
    )
    # Transitions to Staging automatically
    client = mlflow.tracking.MlflowClient()
    client.transition_model_version_stage(
        "fraud-detection-rf", version=1, stage="Staging"
    )
```

## 10. Folder Structure
```
mlruns/
├── 0/                          # experiment_id
│   ├── a1b2c3d4/               # run_id
│   │   ├── params/
│   │   │   └── lr
│   │   ├── metrics/
│   │   │   └── accuracy
│   │   ├── tags/
│   │   │   └── mlflow.user
│   │   └── artifacts/
│   │       ├── model/
│   │       │   ├── model.pkl
│   │       │   └── conda.yaml
│   │       └── confusion_matrix.png
│   └── b2e3f4a5/
└── 1/
mlflow_project/
├── MLproject
├── conda.yaml
├── train.py
└── src/
    └── features.py
```

## 11. Configuration
```ini
# tracking server config
MLFLOW_TRACKING_URI=http://mlflow.internal:5000
MLFLOW_S3_ENDPOINT_URL=https://s3.us-east-1.amazonaws.com
MLFLOW_TRACKING_USERNAME=admin
MLFLOW_TRACKING_PASSWORD=${SECRET}

# artifact store
MLFLOW_ARTIFACT_LOCATION=s3://mlflow-artifacts-${ACCOUNT_ID}
MLFLOW_DEFAULT_ARTIFACT_ROOT=s3://mlflow-artifacts/
```

## 12. Flowchart

```
    Start
      │
      ▼
  Set tracking URI & experiment
      │
      ▼
  Start run ───────────────────┐
      │                        │
      ▼                        ▼
  Log params (immutable)    Log metrics (mutable)
      │                        │
      └────────┬───────────────┘
               ▼
         Train model
               │
               ▼
         Log model artifact
               │
               ▼
         Register model
               │
               ▼
         [accuracy > 0.95?]
          /              \
        YES              NO
         │                │
         ▼                ▼
   Transition to      End (discard)
   Staging
         │
         ▼
   Eval on canary
         │
         ▼
   Transition to Prod
         │
         ▼
        End
```

## 13. Sequence Diagram
```
Data Scientist           MLflow Tracking          Model Registry        Production
     │                        │                       │                    │
     │──mlflow.start_run()───▶│                       │                    │
     │──log_param("lr")──────▶│                       │                    │
     │──log_metric("acc")────▶│                       │                    │
     │──log_model()──────────▶│                       │                    │
     │                        │──store to S3─────────▶│                    │
     │──register_model()─────▶│──────────────────────▶│                    │
     │                        │                       │──version=1────────▶│
     │──transition(Staging)──▶│──────────────────────▶│                    │
     │                        │                       │                    │
  CI/CD Trigger               │                       │                    │
     │──deploy(staging)───────│───────────────────────│───────────────────▶│
     │                        │                       │                    │
Canary Eval: accuracy 0.97   │                       │                    │
     │──transition(Prod)─────▶│──────────────────────▶│                    │
     │                        │                       │──serve v1─────────▶│
```

## 14. Pros
- **Language-agnostic**: Python, R, Java, REST API
- **Framework-agnostic**: sklearn, PyTorch, TF, XGBoost, Keras
- **Pluggable stores**: MySQL → Postgres, S3 → GCS → MinIO
- **Batteries-included UI**: compare runs, visualize metrics, search
- **Model Registry**: stage transitions, versioning, audit trail
- **Lightweight**: single `pip install`; no cluster needed
- **Full MLflow Project**: reproducible runs via `MLproject` + `conda.yaml`

## 15. Cons
- **No authentication built-in** (requires proxy sidecar)
- **Backend store becomes bottleneck** at scale (10M+ runs)
- **No native scheduling** — needs Airflow/Prefect integration
- **Model serving is basic** — no A/B testing, traffic splitting
- **Large artifact uploads block** the API (async gap)
- **Registry RBAC is coarse** — lacks per-team permissions

## 16. Alternatives
| Tool | Strength | Weakness |
|------|----------|----------|
| **Weights & Biases** | Rich UI, collaboration | Hosted-only, expensive |
| **Neptune** | Metadata store, teams | No model registry |
| **Kubeflow Pipelines** | Full MLOps on K8s | Ops-heavy |
| **Metaflow** | Cloud-native, step functions | Tight to AWS |
| **DVC** | Git-native, no DB needed | No UI, no registry |
| **Azure ML** | Deep Azure integration | Vendor lock-in |

## 17. Performance Considerations
- **Backend store**: Add read replicas for Postgres; partition `metrics` by `run_uuid`; index `(key, value)` composite
- **Artifact store**: Use S3 with multipart upload for files >100MB; enable S3 Transfer Acceleration
- **Tracking server**: Horizontally scale Gunicorn workers (`--workers=$((2*CPU))`); add Redis cache for read-heavy workloads
- **Batch logging**: Use `mlflow.log_batch()` (logs params + metrics + tags in one API call)
- **Avoid N+1**: Use `mlflow.search_runs()` with DataFrame API instead of per-run fetches
- **Retention**: Archive runs >90 days to cold storage; SQL `DELETE` on old experiments

## 18. Scaling to Millions
- **Shard by experiment**: Pre-create experiments per team/project; separate DB schemas
- **gRPC gateway**: Replace REST with gRPC for 10x throughput at >10K runs/min
- **Async artifact upload**: Client uploads to S3 directly (presigned URL); server verifies async
- **Read-through cache**: Redis for recent runs, metrics aggregations, dashboard queries
- **Metric rollups**: Pre-aggregate metrics per run (min/max/avg/last) into summary tables
- **Partition pruning**: List runs by date range; use time-based partition keys in Postgres (`runs_202601`, `runs_202602`)

## 19. Failure Scenarios
- **Backend store down**: Runs cannot be logged; retry with exponential backoff; buffer locally via `MLFLOW_ASYNC_LOGGING=true`
- **Artifact store inaccessible**: Model logging fails; use S3 lifecycle rules to auto-retry
- **Concurrent stage transitions**: Last-writer-wins; implement optimistic locking via DB version number
- **Corrupt artifact on upload**: Compute MD5 checksum; verify on download; retry on mismatch
- **OOM on large runs**: Stream artifacts in chunks; log metrics in batches of 1000

## 20. Security
- **Authentication**: Nginx sidecar with OAuth2/OIDC (Keycloak, Auth0); restrict to `mlflow` user group
- **Authorization**: Custom middleware for per-experiment RBAC; model registry read/write ACLs
- **Encryption**: TLS 1.3 for all API traffic; S3 SSE-S3 or KMS for artifacts at rest; DB encryption at rest
- **Audit logging**: Log all API calls to CloudTrail; track stage transitions with user identity
- **Network isolation**: Deploy in VPC; use VPC endpoints for S3; no public IPs
- **Secrets**: Never log secrets as params; use environment variables or Vault

## 21. Monitoring
- **Track these metrics** from MLflow:
  - Run count per experiment (daily)
  - Model registration rate
  - Stage transition frequency
  - API latency (p50, p99)
  - Artifact store throughput
- **Alerts on**: Backend store connection errors > 5/min; artifact uploads failing; registry write conflicts
- **Dashboards**: Grafana backed by MLflow metrics exported to CloudWatch; per-experiment success rates

## 22. Interview Questions
**Q1:** How would you design a system to compare 10,000 ML experiments with sub-second latency?  
*A1:* Pre-aggregate metric rollups into summary tables; partition by experiment; use Redis for top-K experiment comparison queries; materialize `search_runs()` results.

**Q2:** How do you handle concurrent model version promotions?  
*A2:* Implement optimistic locking using a `version_number` column. On transition, `UPDATE ... WHERE version_number = :old_version`. If row count is 0, retry with fresh read.

**Q3:** Walk me through deploying an MLflow model to production with zero downtime.  
*A3:* Register model → transition to Staging → deploy behind canary → run shadow evaluation → transition to Production → deploy to main serving fleet → Archived previous version.

## 23. Cheat Sheet
```bash
# Tracking
mlflow experiments create --name "fraud-v2"
mlflow run . -P lr=0.01 --experiment-id 0
mlflow ui -p 5000

# Registry
mlflow models --help
mlflow models serve -m models:/fraud-detection/Production -p 5001

# Python
import mlflow
mlflow.set_experiment("exp")
with mlflow.start_run() as run:
    mlflow.log_param("lr", 0.01)
    mlflow.sklearn.log_model(model, "model")
```

## 24. Common Mistakes
1. **Logging mutable params**: Params are immutable; change metrics for mutable values
2. **Not setting tracking URI**: Defaults to local `mlruns/` — lost on container restart
3. **Artifact path collisions**: Two runs logging `model/` to same experiment — use run UUID paths
4. **Skipping model registration**: Relying on run URIs instead of registry — breaks promotion workflows
5. **Logging everything in one run**: Separate preprocessing, training, eval into distinct runs
6. **No garbage collection**: Old runs accumulate — set TTL or retention policies
7. **Hard-coding experiment names**: Creates duplicates; use `get_or_create_experiment()`

## 25. Production Best Practices
- Deploy Tracking Server behind a load balancer (Nginx + Gunicorn) with SSL termination
- Use PostgreSQL (not SQLite) for backend store in multi-user deployments
- Set `MLFLOW_TRACKING_URI` via environment variable — never hard-code
- Enable async logging (`MLFLOW_ASYNC_LOGGING=true`) for non-blocking training
- Configure S3 bucket lifecycle to transition artifacts >30 days to Glacier
- Run `mlflow gc` periodically to purge deleted experiments and orphaned runs
- Pin MLflow version in conda.yaml for reproducibility
- Integrate registry promotions with CI/CD — no manual stage transitions in production
- Monitor tracking server health via `/health` endpoint behind ALB
- Use `mlflow.pyfunc` custom models for non-Python or complex inference logic
