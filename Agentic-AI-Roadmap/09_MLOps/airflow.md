# Airflow for ML Pipelines

> Author: **Tamilselvan** · ✉️ tamilselvan.sde@gmail.com · 🔗 [LinkedIn](https://www.linkedin.com/in/tamilselvan-ai/)
>

## 1. What is it?
Apache Airflow is a DAG-based workflow orchestrator where pipelines are defined as Python code. Each DAG (Directed Acyclic Graph) consists of Tasks connected by dependencies. Airflow handles scheduling, retries, failure alerts, and execution on distributed workers — making it the industry standard for batch ML pipeline orchestration.

## 2. Why do we need it?
ML pipelines involve many steps: data extraction → validation → preprocessing → training → evaluation → deployment. Running these manually is error-prone, non-reproducible, and doesn't scale. Airflow provides:
- **Scheduling** (daily model retraining)
- **Dependency resolution** (preprocessing must finish before training)
- **Retries and alerts** (transient failures auto-retry)
- **Observability** (every task's status, logs, duration)

## 3. Real-world Example
**Uber's** ML Platform runs 50,000+ Airflow DAGs daily. A trip forecasting pipeline: ingest 24h of GPS data → validate schema → feature engineering (50K features) → train XGBoost → evaluate → publish model to registry. If data quality check fails, the DAG alerts the on-call data engineer via PagerDuty and pauses downstream tasks, preventing a bad model from reaching production.

## 4. Architecture Diagram (ASCII)

```
┌───────────────────────────────────────────────────────────────┐
│                         Airflow Scheduler                      │
│   (Reads DAG files → creates DAG Runs → queues Task Instances) │
└──────────────────────┬────────────────────────────────────────┘
                       │
                       ▼
┌───────────────────────────────────────────────────────────────┐
│                     Metadata Database (Postgres)               │
│  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────────────┐ │
│  │ dag      │ │ task     │ │ dag_run  │ │ task_instance    │ │
│  │          │ │          │ │          │ │                  │ │
│  │ dag_id   │ │ task_id  │ │ run_id   │ │ state            │ │
│  │ schedule │ │ pool     │ │ conf     │ │ start_date       │ │
│  │ is_paused│ │ retries  │ │ exec_date│ │ duration         │ │
│  └──────────┘ └──────────┘ └──────────┘ └──────────────────┘ │
└──────────────┬───────────────────────────────────────────────┘
               │
               ▼
┌───────────────────────────────────────────────────────────────┐
│              Executor (Celery / Kubernetes / Local)            │
│                                                               │
│  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────────────┐ │
│  │ Worker 1 │ │ Worker 2 │ │ Worker 3 │ │     ...          │ │
│  │ CPU: 4   │ │ CPU: 8   │ │ GPU: 1   │ │                  │ │
│  │ Mem:16GB │ │ Mem:32GB │ │ Mem:64GB │ │                  │ │
│  └──────────┘ └──────────┘ └──────────┘ └──────────────────┘ │
└──────────────┬───────────────────────────────────────────────┘
               │
               ▼
┌───────────────────────────────────────────────────────────────┐
│              External Components (S3, MLflow, K8s)            │
│  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────────────┐ │
│  │ S3/Data  │ │ MLflow   │ │ Kubeflow │ │  Slack / PagerDuty│ │
│  └──────────┘ └──────────┘ └──────────┘ └──────────────────┘ │
└───────────────────────────────────────────────────────────────┘
```

## 5. Internal Working
Airflow's **Scheduler** polls the DAG folder every `dag_dir_list_interval` seconds. For each DAG, it:
1. Parses the DAG Python file into a DAG object
2. Computes the next execution date based on `schedule_interval`
3. Creates a `DagRun` in the metadata DB
4. Identifies tasks whose dependencies are met (upstream success + no unmet dependencies)
5. Queues `TaskInstance` objects in the executor's queue
6. Workers pick up tasks, execute them, and report status back to the metadata DB

The **Webserver** reads the DB to serve the UI. **Workers** can be Celery (separate machines), Kubernetes (per-task pods), or Local (single machine).

## 6. Production Flow
```
┌──────────┐   ┌──────────┐   ┌──────────┐   ┌──────────┐   ┌──────────┐
│ Schedule │──▶│ Extract  │──▶│ Validate │──▶│ Feature  │──▶│ Train    │
│ Daily    │   │ Data     │   │ Quality  │   │ Engineer │   │ Model    │
└──────────┘   └──────────┘   └──────────┘   └──────────┘   └─────┬────┘
                                                                  │
                                                                  ▼
┌──────────┐   ┌──────────┐   ┌──────────┐   ┌──────────┐   ┌──────────┐
│ Deploy   │◀──│ Promote  │◀──│ Register │◀──│ Evaluate │◀──│          │
│ to Prod  │   │ Staging  │   │ MLflow   │   │ Model    │   │          │
└──────────┘   └──────────┘   └──────────┘   └──────────┘   └──────────┘
```

## 7. HLD

```
┌──────────────────────────────────────────────────────────────────────┐
│                          Airflow Cluster                              │
│                                                                       │
│  ┌────────────────────────────────────────────────────────────────┐  │
│  │                 Web Tier (Webserver x2)                          │  │
│  │  Flask App  │  RBAC Auth  │  Static Assets │  REST API         │  │
│  └────────────────────────────────────────────────────────────────┘  │
│                                                                       │
│  ┌────────────────────────────────────────────────────────────────┐  │
│  │               Scheduler (HA — active/passive)                    │  │
│  │  DAG Processor │  DagFileProcessorManager │  TaskSupervisor    │  │
│  └────────────────────────────────────────────────────────────────┘  │
│                                                                       │
│  ┌────────────────────────────────────────────────────────────────┐  │
│  │             Celery Cluster (workers autoscale 2-50)             │  │
│  │  ┌────────┐ ┌────────┐ ┌────────┐ ┌────────┐ ┌──────────────┐ │  │
│  │  │ CPU    │ │ GPU    │ │ BigMem │ │ Spot   │ │ Preemptible  │ │  │
│  │  │ 4 vCPU │ │ 1 GPU  │ │ 64GB   │ │ 8 vCPU │ │ (cost 70%)   │ │  │
│  │  └────────┘ └────────┘ └────────┘ └────────┘ └──────────────┘ │  │
│  └────────────────────────────────────────────────────────────────┘  │
│                                                                       │
│  ┌────────────────────────────────────────────────────────────────┐  │
│  │          Message Broker (Redis / RabbitMQ)                      │  │
│  │  ┌──────────────────┐ ┌──────────────────┐ ┌────────────────┐ │  │
│  │  │ Queue: default   │ │ Queue: gpu       │ │ Queue: big_mem│ │  │
│  │  └──────────────────┘ └──────────────────┘ └────────────────┘ │  │
│  └────────────────────────────────────────────────────────────────┘  │
│                                                                       │
│  ┌────────────────────────────────────────────────────────────────┐  │
│  │  Metadata DB (PostgreSQL RDS Multi-AZ)                          │  │
│  │  ┌────────────┐ ┌────────────┐ ┌────────────┐ ┌─────────────┐ │  │
│  │  │ dag        │ │ task       │ │ serialized │ │ log         │ │  │
│  │  │            │ │ instance   │ │ dag        │ │             │ │  │
│  │  └────────────┘ └────────────┘ └────────────┘ └─────────────┘ │  │
│  └────────────────────────────────────────────────────────────────┘  │
└──────────────────────────────────────────────────────────────────────┘
```

## 8. LLD

**Key database tables and relationships:**
```
dag(dag_id, is_paused, is_active, next_dagrun, schedule_interval)
task_instance(task_id, dag_id, run_id, start_date, end_date, state,
              duration, try_number, max_tries, queue, pool)
dag_run(dag_id, run_id, execution_date, start_date, state,
        external_trigger, conf)
xcom(dag_run_id, task_id, key, value) — cross-task communication
```

**Task lifecycle states:**
```
none → scheduled → queued → running → success
                                  → failed → up_for_retry → scheduled
                                  → failed → failed (no retries)
```

## 9. Python Implementation

```python
# dags/ml_training_pipeline.py
from airflow import DAG
from airflow.operators.python import PythonOperator
from airflow.providers.amazon.aws.hooks.s3 import S3Hook
from airflow.utils.dates import days_ago
import pandas as pd
import mlflow

default_args = {
    "owner": "ml-team",
    "retries": 2,
    "retry_delay": 5,
    "execution_timeout": 3600,
}

with DAG(
    "ml_training_pipeline",
    default_args=default_args,
    schedule_interval="0 6 * * *",  # daily at 6am
    start_date=days_ago(1),
    catchup=False,
    tags=["ml", "fraud-detection"],
) as dag:

    def extract(**context):
        hook = S3Hook("s3_conn")
        data = hook.read_key("raw/transactions.csv", "ml-data-bucket")
        context["ti"].xcom_push(key="data_path", value="/tmp/transactions.csv")
        return data.shape

    def validate(**context):
        df = pd.read_csv("/tmp/transactions.csv")
        assert df["amount"].isna().sum() == 0, "Null amounts"
        assert df["timestamp"].dtype == "datetime64[ns]"
        assert len(df) > 100_000, "Not enough records"

    def train(**context):
        df = pd.read_csv("/tmp/transactions.csv")
        X, y = df.drop("fraud", axis=1), df["fraud"]

        from xgboost import XGBClassifier
        model = XGBClassifier(n_estimators=200)
        model.fit(X, y)

        import joblib
        joblib.dump(model, "/tmp/model.pkl")
        context["ti"].xcom_push(key="model_path", value="/tmp/model.pkl")

    def register(**context):
        mlflow.set_tracking_uri("http://mlflow:5000")
        with mlflow.start_run():
            mlflow.log_metric("accuracy", 0.97)
            mlflow.sklearn.log_model(
                "/tmp/model.pkl", artifact_path="model"
            )

    extract_task = PythonOperator(task_id="extract", python_callable=extract)
    validate_task = PythonOperator(task_id="validate", python_callable=validate)
    train_task = PythonOperator(task_id="train", python_callable=train)
    register_task = PythonOperator(task_id="register", python_callable=register)

    extract_task >> validate_task >> train_task >> register_task
```

## 10. Folder Structure
```
airflow/
├── dags/
│   ├── ml_training_pipeline.py
│   ├── ml_evaluation_dag.py
│   └── data_quality_monitor.py
├── plugins/
│   ├── operators/
│   │   ├── mlflow_operator.py
│   │   └── model_deploy_operator.py
│   ├── sensors/
│   │   └── model_registry_sensor.py
│   └── hooks/
│       └── mlflow_hook.py
├── scripts/
│   ├── train.py
│   └── evaluate.py
├── include/
│   └── sql/
│       └── feature_queries.sql
├── config/
│   ├── airflow.cfg
│   └── webserver_config.py
├── Dockerfile
├── docker-compose.yaml
└── requirements.txt
```

## 11. Configuration
```ini
# airflow.cfg
[core]
executor = CeleryExecutor
parallelism = 32
dag_dir_list_interval = 30
max_active_runs_per_dag = 3
load_examples = False

[webserver]
rbac = True
authenticate = True
auth_backend = airflow.contrib.auth.backends.password_auth

[celery]
broker_url = redis://redis:6379/0
result_backend = db+postgresql://user:pass@postgres/airflow
worker_concurrency = 8
worker_prefetch_multiplier = 1

[scheduler]
scheduler_heartbeat_sec = 5
task_queues = default,gpu,big_mem
max_tis_per_query = 512
```

## 12. Flowchart

```
    DAG Trigger (Time/Event)
           │
           ▼
    extract_task
    ┌──────────────┐
    │ Pull from S3 │───Fail?──▶ Retry (x2)
    │ Save to /tmp │           │
    └──────┬───────┘           ▼
           │ success         Fail → Alert
           ▼
    validate_task
    ┌──────────────┐
    │ Null check   │───Fail?──▶Alert → Stop
    │ Type check   │
    │ Size check   │
    └──────┬───────┘
           │ success
           ▼
    train_task
    ┌──────────────┐
    │ XGBoost fit  │───Fail?──▶ Retry (x2)
    │ Save model   │           │
    └──────┬───────┘           ▼
           │ success         Fail → Alert
           ▼
    register_task
    ┌──────────────┐
    │ MLflow log   │───Fail?──▶ Alert
    │ Register v2  │
    └──────┬───────┘
           │ success
           ▼
    Slack Notify: "Model v2 deployed ✓"
```

## 13. Sequence Diagram
```
Scheduler            DB                 Celery Queue         Worker           MLflow
    │                 │                     │                  │                │
    │─parse dag──────▶│                     │                  │                │
    │─create run─────▶│                     │                  │                │
    │─enqueue task───▶│────extract_task────▶│                  │                │
    │                 │                     │──pick up────────▶│                │
    │                 │                     │                  │──read S3──────▶│
    │                 │                     │                  │──store /tmp    │
    │                 │                     │◀─success─────────│                │
    │─update state───▶│                     │                  │                │
    │─enqueue───────▶│────valid_task──────▶│──pick up────────▶│                │
    │                 │                     │                  │──validate df   │
    │                 │                     │◀─success─────────│                │
    │─update state───▶│                     │                  │                │
    │─enqueue───────▶│────train_task──────▶│──pick up────────▶│                │
    │                 │                     │                  │──fit model     │
    │                 │                     │◀─success─────────│                │
    │─update state───▶│                     │                  │                │
    │─enqueue───────▶│──register_task─────▶│──pick up────────▶│                │
    │                 │                     │                  │──mlflow.log───▶│
    │                 │                     │◀─success─────────│                │
    │─update state───▶│                     │                  │                │
```

## 14. Pros
- **Python-native DAGs**: Full programming power — no YAML DSL
- **Rich ecosystem**: 900+ providers (AWS, GCP, Azure, MLflow, Snowflake)
- **Proven at scale**: Uber, Twitter, Airbnb, Etsy
- **Granular retries**: Per-task retry policies, exponential backoff
- **SLA monitoring**: Task duration with alerts on miss
- **Dynamic DAGs**: Generate DAGs at runtime based on config
- **Pool management**: Limit concurrent GPU/CPU tasks

## 15. Cons
- **No real-time**: Minimum schedule is ~1 minute; designed for batch
- **DAG parsing bottleneck**: ~10K DAGs → scheduler CPU spikes
- **No native containerization**: KubernetesPodOperator is a bolt-on
- **XCom size limit**: Cross-task communication limited to 48KB (DB)
- **Catchup confusion**: Backfills by default — surprises new users
- **Monitoring is basic**: Default statsd metrics lack ML-specific dimensions
- **Versioning pain**: DAG code and tasks cannot be easily versioned side-by-side

## 16. Alternatives
| Tool | Strength | Weakness |
|------|----------|----------|
| **Prefect** | Pythonic, cloud-native, retries built-in | Newer ecosystem |
| **Dagster** | Asset-focused, data lineage | Steeper learning curve |
| **Kubeflow Pipelines** | K8s-native, GPU support | Kubernetes dependency |
| **Flyte** | ML-focused, typed tasks | Complex setup |
| **Temporal** | Long-running workflows | Not batch-scheduling focused |
| **AWS Step Functions** | Serverless, state machine | Vendor lock-in |

## 17. Performance Considerations
- **Scheduler**: Set `parsing_processes` to CPU core count; use `dag_file_processor_timeout` to kill stuck parsers
- **Database**: Set `max_tis_per_query` to 512-2048; use `min_file_process_interval` to reduce DB write frequency
- **Executor**: Kubernetes Executor scales per-task pods but has launch latency; Celery Executor is faster for sub-5min tasks
- **Queues**: Route GPU-training tasks to `queue=gpu` with 1 task at a time; route I/O tasks to `queue=default` with high concurrency
- **Pools**: Create `pool=ml_training` with 3 slots to prevent concurrent model retrains from consuming all resources
- **Task duration**: Break long tasks (<30min) into smaller ones for better parallelism and failure granularity

## 18. Scaling to Millions
- **Shard by domain**: Separate Airflow deployments per ML domain (ads, recommendations, fraud) — each with its own DB and scheduler
- **DAG autogeneration**: Template-based DAG factory: one Python file produces 500 DAGs with different `schedule_interval` and `params`
- **Read replica for DB**: Offload webserver reads to Postgres read replica; scheduler writes to primary
- **Redis Sentinel**: HA Redis for Celery broker; RabbitMQ with mirrored queues as alternative
- **Task distribution**: Dynamic task queuing based on GPU/CPU availability; `KubernetesPodOperator` for GPU tasks
- **SubDAGs and TaskGroups**: Modular DAGs with `TaskGroup` for logical grouping without SubDAG overhead

## 19. Failure Scenarios
- **DB connection exhausted**: Set `sql_alchemy_pool_size` properly; monitor with `sql_alchemy_pool_overflow`
- **Zombie tasks**: Scheduler marks tasks as failed if heartbeat missed for `scheduler_zombie_task_threshold`
- **Deadlocked DAGs**: All tasks waiting on upstream that failed → set `depends_on_past=False` or add timeout
- **Executor queue buildup**: Workers fall behind → add autoscaling (Celery with `celery.worker_autoscale`)
- **Corrupted serialized DAG**: Scheduler stores DAGs serialized in DB; force re-serialize via delete from `serialized_dag` table

## 20. Security
- **RBAC**: Built-in roles (Admin, User, Viewer, Op); map to LDAP/SSO groups
- **Kerberos**: Integration for secured Hadoop clusters
- **Vault**: HashiCorp Vault integration for secrets (DB passwords, API keys)
- **Network**: DAG files stored in private S3 bucket; workers in private subnets
- **Audit logging**: All UI actions logged; export to CloudWatch
- **Fernet key**: Encrypt connection passwords in DB (`airflow.cfg` → `fernet_key`)
- **CORS**: Restrict API access to known origins; disable `expose_config`

## 21. Monitoring
- **Key metrics** (via statsd + Datadog/Grafana):
  - `dag_processing.last_duration.${dag}` — parse time
  - `scheduler.critical_section_duration` — scheduler loop time
  - `executor.open_slots` — available worker capacity
  - `ti_failures` — failed tasks per DAG
  - `pool_starving` — tasks waiting on pool
- **Alerts**: Any task retried >3 times; scheduler heartbeat missing; queue depth >1000
- **ML-specific alerts**: Model metric drops >0.05; training data age >48h; evaluation failure

## 22. Interview Questions
**Q1:** Design an Airflow DAG for daily model retraining that handles data delays gracefully.  
*A1:* Use `wait_for_data` sensor that checks S3 for `_SUCCESS` file every 30min with timeout of 4h. If data arrives late, use `depends_on_past=False` so today's run isn't blocked by yesterday's failure. Use `max_active_runs=1` to avoid concurrent retrains.

**Q2:** How would you implement GPU-aware task scheduling in Airflow?  
*A2:* Create a `queue=gpu` in Celery. Start GPU workers with `airflow celery worker -q gpu -c 1` (1 task at a time per GPU). Define tasks with `queue="gpu"`. Use pool `pool="gpu_pool"` with `slots=num_gpus`. Monitor GPU memory via custom statsd metric.

**Q3:** How does Airflow handle backfilling 3 years of daily runs without overloading the system?  
*A3:* DAG with `catchup=False` by default. For backfills: `airflow backfill ml_dag -s 2023-01-01 -e 2023-03-01`. The scheduler creates consecutive dag_runs but respects `max_active_runs_per_dag` to prevent overload. Use `backfill --reset-dagruns` to re-run failed partitions.

## 23. Cheat Sheet
```bash
# CLI
airflow dags list
airflow dags unpause ml_pipeline
airflow dags trigger ml_pipeline
airflow tasks list ml_pipeline
airflow tasks clear ml_pipeline --downstream -t train
airflow backfill ml_pipeline -s 2024-01-01 -e 2024-01-10

# UI
localhost:8080/home
localhost:8080/dags/ml_pipeline/grid
localhost:8080/dags/ml_pipeline/duration

# Python DAG
from airflow import DAG
from airflow.operators.python import PythonOperator
dag = DAG("my_dag", schedule_interval="@daily", catchup=False)
task = PythonOperator(task_id="train", python_callable=train, dag=dag)
```

## 24. Common Mistakes
1. **`depends_on_past=True` everywhere**: One failure cascades to all future runs — use sparingly
2. **`start_date` in the past without `catchup=False`**: Triggers hundreds of backfill runs
3. **Heavy imports in DAG file**: Every `parsing_process` imports TensorFlow → OOM; defer imports inside `python_callable`
4. **XCom for large data**: 1GB trained model via XCom → DB crash; use S3 paths instead
5. **No task timeout**: Stuck training task runs forever → set `execution_timeout`
6. **Ubiquitous `default_args`**: Different tasks need different retry/timeout — override per task
7. **No tagging**: 500 DAGs without `tags=["ml"]` → impossible to filter in UI

## 25. Production Best Practices
- Deploy Scheduler in HA mode (active/passive with DB-level leader election)
- Use `KubernetesPodOperator` for GPU training tasks with ephemeral storage
- Set `max_active_runs=1` for model training DAGs to prevent concurrent retrains with stale data
- Implement `ShortCircuitOperator` for data quality gates — stops pipeline on bad data
- Store DAGs in S3/GCS and use `git-sync` sidecar for auto-deployment
- Pin Airflow and provider versions in `requirements.txt` — major version upgrades are breaking
- Enable `scheduler.catchup_by_default = False` in `airflow.cfg`
- Use `TaskFlow API` (`@task` decorator) for simpler, type-hinted DAGs
- Schedule resource-qualified task queues: `KubernetesPodOperator(queue="gpu", namespace="ml")`
- Alert on `task_failures > 0` in production via PagerDuty; retry 3x with 5min delay
