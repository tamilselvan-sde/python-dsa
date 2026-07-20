# CI/CD for ML/AI

## 1. What is it?
CI/CD for ML (MLOps CI/CD) extends software CI/CD with ML-specific stages: data validation, feature pipeline testing, model training verification, evaluation gate checks, model registry promotion, and deployment monitoring. The pipeline ensures that code + data + model changes are automatically tested, validated, and deployed to production.

## 2. Why do we need it?
ML systems fail silently. A data schema change, a feature drift, or a degraded model can ship to production without detection. ML CI/CD provides:
- **Data quality gates**: Fail pipeline if nulls spike or distributions shift
- **Reproducible training**: Every commit produces the same model (given same data + seed)
- **Model evaluation gates**: Regression tests on accuracy, latency, and fairness
- **Safe deployment**: Canary → staging → production with automatic rollback

## 3. Real-world Example
**Google's** CI/CD pipeline for Search ranking models runs on every PR. When an ML engineer changes feature code: (1) GitHub Actions runs unit tests on feature engineering, (2) a small training run on 1% data validates the model trains end-to-end, (3) evaluation compares new model vs. baseline on 100+ metrics, (4) a human reviews the diff dashboard, (5) on merge, a full training run produces the production model, (6) canary deployment with automatic rollback if core metrics drop >0.1%.

## 4. Architecture Diagram (ASCII)

```
┌─────────────────────────────────────────────────────────────────────┐
│                        CI/CD Pipeline                                │
│                                                                       │
│  ┌────────┐  ┌────────┐  ┌────────┐  ┌────────┐  ┌────────┐  ┌───┐ │
│  │ Code   │  │ Data   │  │ Unit   │  │ Train  │  │ Model  │  │De-│ │
│  │ Lint   │  │ Validation│ │ Tests  │  │ (mini) │  │ Eval   │  │ploy│
│  └────────┘  └────────┘  └────────┘  └────────┘  └────────┘  └───┘ │
│       │           │            │           │           │         │   │
│       ▼           ▼            ▼           ▼           ▼         ▼   │
│  ┌─────────────────────────────────────────────────────────────┐    │
│  │                    Gate Checks                                │    │
│  │  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌────────────────┐ │    │
│  │  │ Coding   │ │ Data     │ │ Model    │ │ Fairness       │ │    │
│  │  │ Standards│ │ Quality  │ │ Accuracy │ │ Bias           │ │    │
│  │  │ Pass/Fail│ │ Score>0.8│ │ >0.95    │ │ <0.05 diff     │ │    │
│  │  └──────────┘ └──────────┘ └──────────┘ └────────────────┘ │    │
│  └─────────────────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────────────┘
                               │
                               ▼
┌─────────────────────────────────────────────────────────────────────┐
│                      Deployment Stages                               │
│                                                                       │
│  ┌────────────┐   ┌────────────┐   ┌────────────┐   ┌────────────┐ │
│  │ Staging    │──▶│ Canary 5%  │──▶│ Canary 50% │──▶│ Production │ │
│  │ (shadow)   │   │ (validate) │   │ (observe)  │   │ (100%)     │ │
│  └────────────┘   └────────────┘   └────────────┘   └────────────┘ │
│       │                │                │                │          │
│       ▼                ▼                ▼                ▼          │
│  ┌──────────────────────────────────────────────────────────────┐   │
│  │                   Observability Backplane                     │   │
│  │  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌────────────────┐  │   │
│  │  │ Accuracy │ │ Latency  │ │ Drift    │ │ Rollback       │  │   │
│  │  │ Monitor  │ │ Monitor  │ │ Monitor  │ │ Trigger        │  │   │
│  │  └──────────┘ └──────────┘ └──────────┘ └────────────────┘  │   │
│  └──────────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────────┘
```

## 5. Internal Working
A standard ML CI/CD pipeline has these phases:

**CI (every PR):**
1. **Code check**: Lint, type check (mypy), security scan (bandit)
2. **Data validation**: Run Great Expectations tests on sample data; check schema, distributions, null rates
3. **Unit tests**: Test feature engineering, model prediction shape, data transforms
4. **Mini-training**: Train on 1-5% of data; verify model converges (not full accuracy)
5. **Evaluation**: Compare new model vs baseline on metrics — compute diff with confidence intervals
6. **Gate check**: All checks must pass; human approval for model changes; auto-fail for data issues

**CD (on merge to main):**
1. **Full training**: Train on 100% data with production parameters
2. **Model registry**: Register model version; transition to Staging
3. **Staging deploy**: Deploy to isolated environment; run integration tests
4. **Canary deploy**: Route 5% traffic — monitor latency, accuracy, error rate
5. **Gradual rollout**: 25% → 50% → 100% with monitoring at each step
6. **Auto-rollback**: If any metric degrades > threshold, revert automatically

## 6. Production Flow
```
 PR Opened
    │
    ▼
┌─────────────────────┐
│ CI Pipeline          │
│                      │
│  ┌──────────────┐   │
│  │ Lint + Tests  │───▶ Fail → Comment on PR
│  └──────────────┘   │
│  ┌──────────────┐   │
│  │ Data Quality  │───▶ Fail → Block PR
│  └──────────────┘   │
│  ┌──────────────┐   │
│  │ Mini Train    │───▶ Fail → Block PR
│  └──────────────┘   │
│  ┌──────────────┐   │
│  │ Model Eval    │───▶ Produce diff report
│  └──────────────┘   │
│  ┌──────────────┐   │
│  │ Profiling     │───▶ Memory, latency report
│  └──────────────┘   │
└─────────────────────┘
         │ Pass
         ▼
  Human Review (opt)
         │ Approve
         ▼
 PR Merged to Main
         │
         ▼
┌─────────────────────┐
│ CD Pipeline          │
│                      │
│  ┌──────────────┐   │
│  │ Full Train    │───▶ Model + artifacts
│  └──────────────┘   │
│  ┌──────────────┐   │
│  │ Model Registry│───▶ Version 42 → Staging
│  └──────────────┘   │
│  ┌──────────────┐   │
│  │ Staging Deploy│───▶ Shadow tests pass
│  └──────────────┘   │
│  ┌──────────────┐   │
│  │ Canary 5%     │───▶ Observe 1h
│  └──────────────┘   │
│  ┌──────────────┐   │
│  │ Gradual 100%  │───▶ Done
│  └──────────────┘   │
└─────────────────────┘
```

## 7. HLD

```
┌────────────────────────────────────────────────────────────────────────┐
│                          CI/CD Architecture                             │
│                                                                          │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │                     Version Control (GitHub)                      │   │
│  │  ┌────────────┐  ┌────────────┐  ┌────────────┐  ┌───────────┐ │   │
│  │  │ Feature    │  │ PR         │  │ Main       │  │ Release   │ │   │
│  │  │ Branch     │  │ (CI runs)  │  │ (CD runs)  │  │ Tags      │ │   │
│  │  └────────────┘  └────────────┘  └────────────┘  └───────────┘ │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                                  │                                     │
│  ┌───────────────────────────────▼─────────────────────────────────┐   │
│  │                     GitHub Actions                               │   │
│  │                                                                  │   │
│  │  ┌────────────────┐ ┌────────────────┐ ┌────────────────────┐  │   │
│  │  │ ci.yml         │ │ cd.yml         │ │ nightly.yml        │  │   │
│  │  │ (on PR)        │ │ (on push main) │ │ (cron: nightly)    │  │   │
│  │  └────────────────┘ └────────────────┘ └────────────────────┘  │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                                  │                                     │
│  ┌───────────────────────────────▼─────────────────────────────────┐   │
│  │                       Compute Layer                               │   │
│  │                                                                  │   │
│  │  ┌──────────────────┐  ┌──────────────────┐  ┌────────────────┐ │   │
│  │  │ CPU Runner       │  │ GPU Runner        │  │ Large Mem (64GB)│ │   │
│  │  │ (lint, tests,    │  │ (mini training)   │  │ (full training)  │ │   │
│  │  │ data validation) │  │                   │  │                  │ │   │
│  │  └──────────────────┘  └──────────────────┘  └────────────────┘ │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                                                                          │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │                     Storage Layer                                 │   │
│  │  ┌────────────┐  ┌────────────┐  ┌────────────┐  ┌───────────┐ │   │
│  │  │ S3: Data   │  │ S3: Models │  │ MLflow     │  │ Artifactory│ │   │
│  │  │            │  │            │  │ Registry   │  │ (Docker)   │ │   │
│  │  └────────────┘  └────────────┘  └────────────┘  └───────────┘ │   │
│  └─────────────────────────────────────────────────────────────────┘   │
└────────────────────────────────────────────────────────────────────────┘
```

## 8. LLD

**CI Pipeline (`.github/workflows/ci.yml`):**
```yaml
name: ML CI Pipeline
on: pull_request

jobs:
  lint:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: pip install ruff && ruff check src/
      - run: pip install mypy && mypy src/

  data_validation:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: pip install great_expectations
      - run: great_expectations checkpoint run data_quality

  unit_tests:
    needs: [lint]
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: pip install -r requirements-dev.txt
      - run: pytest tests/ -x --cov=src/ --cov-fail-under=80

  mini_train:
    needs: [data_validation, unit_tests]
    runs-on: [self-hosted, gpu]
    steps:
      - uses: actions/checkout@v4
      - name: Mini training
        run: |
          python src/train.py --sample-ratio 0.01 \
            --max-epochs 3 \
            --output-dir /tmp/model/
      - name: Upload mini model
        uses: actions/upload-artifact@v4
        with:
          name: mini-model
          path: /tmp/model/

  model_eval:
    needs: [mini_train]
    runs-on: ubuntu-latest
    steps:
      - uses: actions/download-artifact@v4
      - name: Compare with baseline
        run: |
          python scripts/eval_compare.py \
            --new-model ./mini-model/ \
            --baseline models/baseline.pkl \
            --metrics accuracy,f1,latency
      - name: Post metrics to PR
        uses: actions/github-script@v7
        with:
          script: |
            github.rest.issues.createComment({
              owner: context.repo.owner,
              repo: context.repo.repo,
              issue_number: context.issue.number,
              body: `## Model Evaluation\naccuracy: 0.972\nf1: 0.943\nlatency: 4.2ms\n`
            })
```

**CD Pipeline (`.github/workflows/cd.yml`):**
```yaml
name: ML CD Pipeline
on:
  push:
    branches: [main]

jobs:
  full_train:
    runs-on: [self-hosted, gpu]
    steps:
      - uses: actions/checkout@v4
      - run: |
          dvc pull
          python src/train.py --sample-ratio 1.0 \
            --output-dir models/prod/
      - uses: actions/upload-artifact@v4
        with:
          name: prod-model
          path: models/prod/

  register_model:
    needs: [full_train]
    runs-on: ubuntu-latest
    steps:
      - run: |
          mlflow.register_model \
            runs:/$RUN_ID/model \
            "recommendation-model" \
            --version $VERSION

  deploy:
    needs: [register_model]
    runs-on: ubuntu-latest
    steps:
      - run: python scripts/deploy.py --stage canary --percent 5
      - run: python scripts/wait_for_observation.py --duration 30m
      - run: python scripts/deploy.py --stage production --percent 100
```

## 9. Python Implementation

```python
# scripts/eval_compare.py — CI gate check
import sys
import json
import joblib
import numpy as np
from sklearn.metrics import accuracy_score, f1_score

def evaluate_model(model_path, data_path):
    model = joblib.load(model_path)
    X_test = np.load(f"{data_path}/X_test.npy")
    y_test = np.load(f"{data_path}/y_test.npy")
    preds = model.predict(X_test)
    return {
        "accuracy": accuracy_score(y_test, preds),
        "f1": f1_score(y_test, preds, average="weighted"),
    }

if __name__ == "__main__":
    new = evaluate_model(sys.argv[2], "data/test")
    baseline = evaluate_model(sys.argv[4], "data/test")

    thresholds = {"accuracy": 0.005, "f1": 0.01}  # 0.5% / 1% tolerance

    for metric, threshold in thresholds.items():
        diff = new[metric] - baseline[metric]
        if diff < -threshold:
            print(f"❌ {metric}: {new[metric]:.4f} vs {baseline[metric]:.4f} (Δ={diff:.4f})")
            sys.exit(1)
        print(f"✅ {metric}: {new[metric]:.4f} vs {baseline[metric]:.4f} (Δ={diff:.4f})")
```

## 10. Folder Structure
```
.github/
├── workflows/
│   ├── ci.yml                 # PR checks (lint, test, mini train, eval)
│   ├── cd.yml                 # Merge to main (full train, deploy)
│   ├── nightly.yml            # Data drift checks, model retraining
│   └── release.yml            # Tagged release → prod deploy
├── actions/
│   ├── setup-ml-env/          # Composite action: pip install + dvc pull
│   │   ├── action.yaml
│   │   └── Dockerfile
│   └── model-eval/            # Composite: run eval + comment on PR
│       └── action.yaml
scripts/
├── deploy.py                  # Deployment logic (staging, canary, prod)
├── wait_for_observation.py    # Monitors canary metrics
├── rollback.py                # Emergency rollback
├── eval_compare.py            # CI gate: compare models
└── data_quality_check.py      # Great Expectations checkpoint runner
tests/
├── test_features.py
├── test_model.py
└── conftest.py
config/
├── deployment.yaml            # Deploy config: stages, thresholds
└── eval_thresholds.yaml       # Metric thresholds per model
```

## 11. Configuration
```yaml
# config/deployment.yaml
stages:
  staging:
    traffic: 0%
    duration: 10m
    validation: integration_tests
  canary_5:
    traffic: 5%
    duration: 30m
    validation:
      accuracy_drop: 0.01
      p99_latency_ms: 200
      error_rate: 0.001
  canary_50:
    traffic: 50%
    duration: 1h
    validation:
      accuracy_drop: 0.005
      p99_latency_ms: 150
      error_rate: 0.0005
  production:
    traffic: 100%
    validation:
      accuracy_drop: 0.005
      p99_latency_ms: 100
      error_rate: 0.0001

rollback:
  auto_rollback: true
  cooldown_minutes: 5
  notify_slack: true
```

## 12. Flowchart

```
      Developer pushes code
             │
             ▼
    ┌────────────────┐
    │  CI Pipeline   │
    │                │
    │  ┌──────┐      │
    │  │ Lint │──Bad─▶ Fail → Fix
    │  └──┬───┘      │
    │  ┌──▼───┐      │
    │  │ Unit │──Bad─▶ Fail → Fix
    │  │ Test │      │
    │  └──┬───┘      │
    │  ┌──▼──────┐   │
    │  │ Data    │──Bad─▶ Fail → Fix
    │  │ Quality │   │
    │  └──┬──────┘   │
    │  ┌──▼──────┐   │
    │  │ Mini    │──Bad─▶ Fail → Fix
    │  │ Train   │   │
    │  └──┬──────┘   │
    │  ┌──▼───┐      │
    │  │ Eval  │──Bad─▶ Comment → Discuss
    │  └──┬───┘      │
    └─────┼──────────┘
          │ Pass
          ▼
    Human Review?
     /        \
   YES         NO (auto-approve)
    │           │
    ├───────────┘
    │
    ▼
  Merge to main
    │
    ▼
    ┌────────────────┐
    │  CD Pipeline   │
    │                │
    │  ┌──────────┐  │
    │  │Full Train │  │
    │  └────┬─────┘  │
    │  ┌────▼──────┐ │
    │  │Model      │ │
    │  │Registry   │ │
    │  └────┬──────┘ │
    │  ┌────▼──────┐ │
    │  │Staging    │ │
    │  └────┬──────┘ │
    │  ┌────▼──────┐ │
    │  │Canary 5%  ├──▶ Metrics OK?
    │  └────┬──────┘ │   /          \
    │  ┌────▼──────┐ │ YES           NO
    │  │Canary 50% ├──▶│              │
    │  └────┬──────┘ │ │              ▼
    │  ┌────▼──────┐ │ │         Rollback
    │  │Prod 100%  │ │ │
    │  └───────────┘ │ │
    └────────────────┘ │
                       │
                       ▼
                 Slack: "Deployed ✓"
```

## 13. Sequence Diagram
```
Developer         GitHub Actions          GPU Runner         MLflow           Production
    │                  │                     │                 │                 │
    │─push branch─────▶│                     │                 │                 │
    │                  │─trigger ci.yml─────▶│                 │                 │
    │                  │                     │─lint + test     │                 │
    │                  │                     │─data validate   │                 │
    │                  │                     │─mini train      │                 │
    │                  │                     │─eval compare    │                 │
    │◀─PR comment──────│                     │                 │                 │
    │                  │                     │                 │                 │
    │─merge to main───▶│                     │                 │                 │
    │                  │─trigger cd.yml─────▶│                 │                 │
    │                  │                     │─full train      │                 │
    │                  │                     │─register model──▶                 │
    │                  │                     │                 │──v42 registered │
    │                  │                     │                 │                 │
    │                  │─staging deploy─────▶│                 │──shadow mode───▶│
    │                  │─canary 5% deploy───▶│                 │──route 5%──────▶│
    │                  │                     │                 │                 │
    │                  │─monitor canary ────▶│                 │                 │
    │                  │  (30min, metrics OK)│                 │                 │
    │                  │                     │                 │                 │
    │                  │─canary 50% ────────▶│                 │──route 50%─────▶│
    │                  │─monitor (1h, OK)   ▶│                 │                 │
    │                  │                     │                 │                 │
    │                  │─prod 100% ─────────▶│                 │──route 100%────▶│
    │                  │                     │                 │                 │
    │◀─Slack "✅"──────│                     │                 │                 │
```

## 14. Pros
- **Automated validation**: Catches data issues, model regressions, and code bugs before deployment
- **Reproducibility**: Every model links to a specific commit + data snapshot
- **Confidence**: Canary + gradual rollout with auto-rollback reduces blast radius
- **Audit trail**: Every change is logged in GitHub + MLflow + deployment history
- **Human-in-the-loop**: Model changes require evaluation review; infra changes auto-deploy
- **Observability dashboards**: Every deploy generates a comparison report

## 15. Cons
- **Slow feedback**: Full pipeline can take hours (data validation → mini train → eval)
- **GPU cost**: Running CI on GPU runners is expensive ($1-5/hour)
- **Training non-determinism**: GPU nondeterminism makes exact model comparison impossible
- **Data dependency**: CI needs representative data snapshot — stale data causes false failures
- **Setup complexity**: Self-hosted runners, GPU machines, artifact storage
- **Flaky tests**: Data-dependent tests fail due to distribution shifts, not code bugs

## 16. Alternatives
| Tool | Strength | Weakness |
|------|----------|----------|
| **Jenkins X** | Java/K8s ecosystem | Heavy, XML config |
| **GitLab CI/CD** | Built-in ML capabilities | GitLab-only |
| **CircleCI** | Fast, caching | No GPU natively |
| **Argo Workflows** | K8s-native, DAGs | Complex setup |
| **TFX** | Full ML pipeline framework | TensorFlow-specific |
| **Kubeflow** | K8s MLOps | Overkill for small teams |

## 17. Performance Considerations
- **Mini-training**: 1-5% sample, 1-3 epochs — should finish in <10min on 1 GPU
- **Caching**: `actions/cache@v4` for `pip` deps, DVC cache; `dvc pull --run-cache` to skip cached stages
- **Parallelism**: Run lint + unit tests + data validation in parallel (3 jobs); mini_train waits for all
- **GPU contention**: Self-hosted GPU runner queue → 1 job at a time; `concurrency: gpu-group` in Actions
- **Artifact size**: model.pkl = 200MB; use `actions/upload-artifact@v4` with compression; set retention to 7 days
- **DVC run cache**: `dvc repro --run-cache` skips unchanged pipeline stages — saves 60%+ on CI time

## 18. Scaling to Millions
- **Matrix builds**: Train on multiple data subsets in parallel — `strategy.matrix: { region: [us, eu, ap] }`
- **Pipeline orchestration**: Use Airflow + CI trigger pattern — CI starts Airflow DAG, Airflow manages training
- **Preemptible instances**: Use spot/preemptible GPU instances for mini training (cost 70% less)
- **Model caching**: Hash-based model cache — skip training if identical code + data + params already produced a model
- **Build artifacts**: Store all intermediate artifacts in S3 with TTL; CI runners are ephemeral
- **Distributed CI**: Split across 3 GPU runners in different regions for data locality

## 19. Failure Scenarios
- **CI timeout on mini training**: Set `timeout-minutes: 20`; if exceeded, fail fast — don't waste GPU
- **Data staleness**: CI data snapshot is 2 weeks old → false positive on feature validation → trigger nightly data refresh
- **GPU not available**: Queue for 10min then fail with "No GPU available — rerun when capacity frees"
- **Model registry outage**: CD continues but registers model later; alert on-call
- **Canary metric degradation**: Auto-rollback within 5min; notify #ml-oncall
- **DVC pull failure**: Network blip → retry 3x; if still failing, fail CI (data integrity issue)

## 20. Security
- **Secrets in CI**: Use GitHub Actions secrets for AWS keys, MLflow passwords, deployment tokens
- **OIDC for AWS**: `aws-actions/configure-aws-credentials` with OIDC — no long-lived keys
- **PR security**: No PR from fork gets access to secrets; use `pull_request_target` carefully
- **Artifact scanning**: Run `bandit` for Python security; Docker image scan with Trivy
- **Dependency scanning**: Dependabot for `requirements.txt`; `pip-audit` in CI pipeline
- **Model scanning**: Check for embedded pickle exploits; use `mlflow.pyfunc` with sandboxed serve
- **Least privilege**: CI runner IAM role only has access to CI-specific S3 paths and MLflow read/write

## 21. Monitoring
- **CI metrics**: Pipeline duration, pass/fail rate per job, GPU utilization, data validation failure rate
- **CD metrics**: Deployment duration, canary success rate, rollback frequency, time-to-prod
- **Drift monitoring**: Compare training data distribution vs. production — trigger CI if drift detected
- **Cost tracking**: Track CI/CD GPU cost per day; alert if > $500/day
- **Bottleneck alerts**: If lint takes >2min or mini_train >15min, flag for optimization

## 22. Interview Questions
**Q1:** Design a CI/CD pipeline for an ML system that retrains weekly. How do you detect model degradation before deployment?  
*A1:* Run evaluation on a held-out golden dataset. Compare new model metrics to production model with statistical significance tests (paired bootstrap). Gate: if accuracy drops >1% or any fairness metric degrades, block deployment and notify ML team.

**Q2:** How do you handle CI/CD for models with 100GB+ training data?  
*A2:* Use DVC for data versioning. CI trains on a 1GB sample (1%). Only CD runs full training on the full dataset using a self-hosted GPU runner with 64GB RAM. DVC run cache prevents re-processing unchanged data.

**Q3:** Your canary deployment shows 0.1% accuracy drop. Do you rollback?  
*A3:* Depends on statistical significance. If p-value <0.05 and effect size > 0.001 (pre-defined minimal detectable effect), yes — rollback automatically. If effect is within noise, let canary continue and alert for manual review.

## 23. Cheat Sheet
```yaml
# ci.yml essentials
on: pull_request
jobs:
  lint: { runs-on: ubuntu, steps: [ruff, mypy] }
  data-quality: { steps: [great_expectations] }
  unit-test: { steps: [pytest] }
  mini-train: { runs-on: [gpu], steps: [train --sample-ratio 0.01] }
  eval: { steps: [eval_compare, comment-on-pr] }

# cd.yml essentials
on: push: { branches: [main] }
jobs:
  full-train: { runs-on: [gpu], steps: [dvc pull, train] }
  register: { steps: [mlflow register_model] }
  deploy: { steps: [deploy --canary 5, wait, deploy --prod] }
```

## 24. Common Mistakes
1. **Training on full data in CI**: 1h training on every PR → wasted GPU cost; always use mini training
2. **No data versioning in CI**: CI uses latest data → non-reproducible; always `dvc pull` a specific commit
3. **Ignoring nondeterminism**: GPU training produces slightly different models each run; set tolerances, not exact match
4. **Manual approval gate for every deploy**: Blocks velocity; use automatic gates with human override
5. **No canary rollback**: Deploying directly to 100% → user-facing regression; always do staged rollout
6. **Reusing the same CI runner cache**: Old data artifacts cause stale results; invalidate cache weekly
7. **Not testing model serving**: CI tests training but not serving → model loads but inference crashes in prod

## 25. Production Best Practices
- Use OIDC for cloud provider auth in CI — no long-lived secrets
- Cache pip/DVC/large files intelligently; invalidate weekly
- Mini training must converge in <10min on a representative data slice
- Deploy with blue/green strategy — never in-place update
- Automate rollback with a 5-min cooldown before re-deploying
- Add `model_quality_drop` alert threshold to canary observation window
- Pin Python version + ML library versions in CI Docker image
- Run data quality checks on every PR — not just nightly
- Publish evaluation comparison as an artifact (HTML dashboard)
- Document rollback runbook in the repo's `CONTRIBUTING.md`
