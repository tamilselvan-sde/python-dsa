# DVC — Data Version Control

## 1. What is it?
DVC (Data Version Control) is an open-source tool that brings Git-like versioning to datasets, ML models, and pipelines. It stores metadata in Git (small `.dvc` files pointing to content-addressed storage) while the actual data lives in remote storage (S3, GCS, HDFS, local). DVC also defines reproducible ML pipelines as directed acyclic graphs (DAGs).

## 2. Why do we need it?
ML projects have two versioning problems Git cannot solve:
- **Large files**: Datasets >100MB break Git's design
- **Reproducibility**: A model depends on specific data + params + code + environment — Git tracks code only

DVC solves both: lightweight `.dvc` files in Git, actual data in object storage, and pipelines that track full dependency graphs.

## 3. Real-world Example
**Apple** uses DVC-like patterns in their internal ML platform for Siri's ASR models. A speech team works with 50TB of audio data. Data scientists `dvc pull` only the datasets they need (e.g., English vs. multilingual). The CI pipeline `dvc repro` ensures every model is trained from a locked data snapshot, preventing the "works on my machine" problem across 200+ engineers.

## 4. Architecture Diagram (ASCII)

```
┌────────────────────────────────────────────────────────┐
│                    Git Repository                        │
│                                                         │
│  src/train.py   dvc.yaml   .dvc/config   data.dvc       │
│  params.yaml    metrics.json  .gitignore                │
│                                                         │
│  (SMALL text files)     ┌──────────────────┐           │
│                         │ data.dvc         │           │
│                         │ md5: abc123...   │           │
│                         │ size: 2147483648 │           │
│                         │ # points to S3   │           │
│                         └──────────────────┘           │
└──────────────────────┬─────────────────────────────────┘
                       │ dvc push / dvc pull
                       ▼
┌────────────────────────────────────────────────────────┐
│               Remote Storage (S3/GCS/MinIO)              │
│                                                         │
│  .dvc/cache/                                            │
│  └── ab/                                                │
│      └── c123def456...   ← content-addressed blobs      │
│                                                         │
│  s3://dvc-remote/                                       │
│  └── 2f/                                                │
│      └── 9a1b2c3d...                                    │
└────────────────────────────────────────────────────────┘
```

## 5. Internal Working
DVC computes a **MD5 hash** of every file and directory in the dataset. This hash becomes the cache key. The `.dvc` file stores the hash, file path, and size. When you `dvc push`, DVC copies the file from the local cache to the remote using the hash as the path (`cache/ab/cdef...`). When you `dvc pull`, it computes the expected hash, checks local cache, and downloads missing blobs from remote.

For pipelines, `dvc.yaml` defines stages. Each stage specifies:
- `cmd`: the command to run
- `deps`: input dependencies (data files, source code, params)
- `outs`: output files (models, processed data, metrics)
- `params`: YAML parameter keys

DVC computes a dependency hash graph. Only re-runs stages whose dependencies changed.

## 6. Production Flow
```
Data Engineer          git commit code + data.dvc
     │
     ▼
CI Pipeline: dvc pull → dvc repro → dvc push
     │
     ▼
Results: metrics.json updated in Git
     │
     ▼
Reviewer approves PR
     │
     ▼
Deploy: dvc pull (latest data) → serve model
```

## 7. HLD

```
┌───────────────────────────────────────────────────────────────┐
│                          Data Scientists                       │
│  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────────────┐ │
│  │ dvc pull │ │ dvc repro│ │ dvc diff │ │ dvc experiments  │ │
│  └──────────┘ └──────────┘ └──────────┘ └──────────────────┘ │
└──────────────────────────┬────────────────────────────────────┘
                           │
┌──────────────────────────▼────────────────────────────────────┐
│                     DVC + Git (metadata layer)                  │
│                                                                 │
│  ┌──────────────────────────┐ ┌──────────────────────────────┐ │
│  │  Git (code + .dvc files) │ │ Cache (local .dvc/cache/)   │ │
│  │  GitHub/GitLab/CodeCommit │ │ SSD/NVMe for fast access    │ │
│  └──────────────────────────┘ └──────────────────────────────┘ │
└──────────────────────────┬────────────────────────────────────┘
                           │
┌──────────────────────────▼────────────────────────────────────┐
│                    Remote Storage                               │
│                                                                 │
│  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────────────┐ │
│  │ S3 / GCS │ │  MinIO   │ │  HDFS    │ │  NFS / SFTP      │ │
│  │ (primary) │ │ (on-prem) │ │ (legacy) │ │  (small teams)    │ │
│  └──────────┘ └──────────┘ └──────────┘ └──────────────────┘ │
└───────────────────────────────────────────────────────────────┘
```

## 8. LLD

**`.dvc` file structure:**
```yaml
outs:
- md5: 9a1b2c3d4e5f67890abcdef1234567890
  size: 2147483648
  path: datasets/train.parquet
  cloud: s3://ml-data-bucket/datasets/train.parquet
```

**Cache directory layout:**
```
.dvc/cache/
├── 9a/
│   └── 1b2c3d4e5f...     ← first 2 chars = directory
├── ab/
│   └── cdef123456...
└── 2f/
    └── 9a1b2c3d4e...
```

**Pipeline (`dvc.yaml`):**
```yaml
stages:
  preprocess:
    cmd: python src/preprocess.py
    deps:
      - data/raw/
      - src/preprocess.py
      - params.yaml:
          - preprocess.min_count
    outs:
      - data/processed/
    metrics:
      - metrics/preprocess_stats.json:
          cache: false
  train:
    cmd: python src/train.py
    deps:
      - data/processed/
      - src/train.py
      - params.yaml:
          - train.lr
          - train.epochs
    outs:
      - models/model.pkl
    metrics:
      - metrics/accuracy.json:
          cache: false
```

## 9. Python Implementation

```python
# src/train.py — DVC pipeline stage
import yaml
import pandas as pd
from sklearn.linear_model import LogisticRegression
import json

with open("params.yaml") as f:
    params = yaml.safe_load(f)

lr = params["train"]["lr"]
epochs = params["train"]["epochs"]

data = pd.read_parquet("data/processed/train.parquet")
X, y = data.drop("label", axis=1), data["label"]

model = LogisticRegression(C=1/lr, max_iter=epochs)
model.fit(X, y)

# save model — DVC will track it as output
import joblib
joblib.dump(model, "models/model.pkl")

# save metric
acc = model.score(X, y)
with open("metrics/accuracy.json", "w") as f:
    json.dump({"accuracy": round(acc, 4)}, f)
```

## 10. Folder Structure
```
project/
├── .dvc/
│   ├── config
│   ├── cache/
│   └── tmp/
├── data/
│   ├── raw/
│   │   ├── train.parquet
│   │   └── test.parquet
│   ├── processed/
│   └── raw.dvc              ← tracks data/raw/
├── models/
│   └── model.pkl
├── metrics/
│   ├── accuracy.json
│   └── preprocess_stats.json
├── src/
│   ├── preprocess.py
│   ├── train.py
│   └── evaluate.py
├── dvc.yaml                 ← pipeline definition
├── dvc.lock                 ← locked dependency hashes
├── params.yaml              ← hyperparameters
├── .gitignore               ← /data, /models, .dvc/cache
└── README.md
```

## 11. Configuration
```ini
# .dvc/config
[core]
    remote = myremote
    remote_context = default
['remote "myremote"']
    url = s3://dvc-ml-bucket
    credentialpath = ~/.aws/credentials
    endpointurl = https://s3.us-east-1.amazonaws.com
[ cache ]
    type = symlink
    dir = .dvc/cache
```

## 12. Flowchart

```
     git clone repo
          │
          ▼
    dvc pull (get data)
          │
          ▼
    Edit params.yaml (lr=0.01)
          │
          ▼
    dvc repro (run pipeline)
          │
          ▼
    ┌──────────────┐
    │ Checksum     │
    │ preprocess   ├── hash unchanged? ──▶ skip
    │ dependencies │
    └──────┬───────┘
           ▼ hash changed?
          YES
           │
           ▼
    Run preprocess.py
           │
           ▼
    ┌──────────────┐
    │ Checksum     │
    │ train        ├── hash unchanged? ──▶ skip
    │ dependencies │
    └──────┬───────┘
           ▼ hash changed?
          YES
           │
           ▼
    Run train.py → models/model.pkl
           │
           ▼
    dvc commit (update .dvc files)
           │
           ▼
    git add . && git commit -m "new model"
           │
           ▼
    dvc push (upload data to remote)
```

## 13. Sequence Diagram
```
Data Scientist               Git                  DVC Cache           S3 Remote
     │                        │                     │                   │
     │──dvc pull─────────────▶│────────────────────▶│                   │
     │                        │                     │──miss?───────────▶│
     │                        │                     │◀──data blobs──────│
     │◀───────data ready──────│─────────────────────│                   │
     │                        │                     │                   │
     │──edit params.yaml─────▶│                     │                   │
     │──dvc repro────────────▶│────────────────────▶│                   │
     │                        │  check deps hashes  │                   │
     │                        │  changed → run cmd  │                   │
     │                        │  store outputs      │                   │
     │◀───────model.pkl───────│─────────────────────│                   │
     │                        │                     │                   │
     │──git add / commit─────▶│ .dvc files updated  │                   │
     │──git push─────────────▶│                     │                   │
     │──dvc push─────────────▶│────────────────────▶│──upload blobs────▶│
```

## 14. Pros
- **Git-native UX**: Familiar `add/commit/push/pull` workflow
- **Content-addressable storage**: Deduplication by hash — same file stored once
- **Pipeline DAG caching**: Only re-runs changed stages
- **Cloud-agnostic**: S3, GCS, Azure Blob, MinIO, HDFS, NFS, SSH
- **No database needed**: Everything is files + Git
- **DVC experiments**: Git branch-like experiment tracking without data duplication
- **Lightweight**: Python pip install, no server daemon
- **Metrics diff**: `dvc diff` shows accuracy changes between commits

## 15. Cons
- **Not real-time**: Designed for batch training workflows
- **No UI**: Pure CLI — no experiment comparison dashboard
- **Large file downloads**: Pulling 50TB datasets is slow without partial transfers
- **Cache eviction**: No built-in LRU — must manually prune `dvc gc`
- **Pipeline debugging**: Error messages can be opaque; hard to visualize DAG
- **No model registry**: DVC tracks models as artifacts but lacks stage/promotion semantics
- **Symlink issues**: Cross-filesystem symlinks fail; type=copy is slow

## 16. Alternatives
| Tool | Strength | Weakness |
|------|----------|----------|
| **Git LFS** | Simple, GitHub native | No pipeline, no hash dedup |
| **Pachyderm** | Data lineage, auto-versioning | Heavy (K8s only) |
| **Delta Lake** | ACID transactions, time travel | Spark dependency |
| **Quilt** | Data package management | No pipelines |
| **Dolt** | SQL-native data versioning | Relational only |
| **LakeFS** | Git-like data lake branching | Infrastructure-heavy |

## 17. Performance Considerations
- **Cache type**: Use `type=hardlink` or `type=symlink` on same filesystem for zero-copy; use `type=copy` across filesystem boundaries
- **Remote storage**: S3 with multipart upload for files >5GB; enable S3 Transfer Acceleration
- **DVC pipeline parallelism**: `dvc repro --parallel` runs independent stages concurrently
- **Large datasets (>100GB)**: Split into sharded `.dvc` files for parallel pull
- **`dvc diff` optimization**: Only checks hashes of `.dvc` files changed in Git diff
- **Garbage collection**: `dvc gc -w` removes unused cache entries; schedule weekly
- **Cache on SSD**: Store `.dvc/cache` on NVMe for fast hash lookups

## 18. Scaling to Millions
- **Shard data by project**: Separate DVC remotes per team/project
- **Versioned datasets**: Use dataset manifests (single `.dvc` file listing all dataset files) instead of per-file tracking
- **Partial checkout**: `dvc pull data/current_subset/` instead of full dataset
- **CDN for remote**: CloudFront CDN in front of S3 for global teams
- **Parallel push/pull**: `dvc push -j 8` uses 8 upload threads; tune `-j` to network bandwidth
- **External cache**: Mount EFS or NFS as shared cache across team machines
- **Tiered storage**: Hot data on SSD, warm on HDD, cold on S3 Glacier

## 19. Failure Scenarios
- **Corrupt cache**: `dvc cache --md5check` verifies all cache entries; `dvc checkout` to restore from remote
- **Remote unavailable**: `dvc fetch --all-branches` to preload cache before remote failure
- **Partial pull failure**: `dvc pull` is idempotent — retries after fixing connectivity
- **Merge conflicts in `.dvc` files**: Resolve like Git conflicts — accept correct hash
- **Disk full during repro**: DVC fails gracefully with "No space left"; `dvc gc` to free cache

## 20. Security
- **Remote authentication**: IAM roles for S3; service account keys for GCS; SSH keys for SFTP
- **Encryption in transit**: S3 over HTTPS; SSE-S3/KMS at rest
- **Secrets management**: Never store credentials in `.dvc/config` — use `dvc remote default --local` to keep secrets out of Git
- **`.dvcignore`**: Add sensitive files to `.dvcignore` to prevent accidental tracking
- **Access control**: Git repository permissions gate access to .dvc metadata; S3 bucket policies gate data
- **Audit trail**: Git history is the audit log; `git blame data.dvc` shows who changed what data

## 21. Monitoring
- **Track**: Cache hit ratio; remote upload/download throughput; pipeline stage runtimes; stages skipped vs re-run
- **Alerts**: Remote storage unavailable; cache corruption detected; `dvc gc` space reclaimed >100GB
- **Dashboard**: Collect `dvc metrics diff` output in CI and publish to Slack/Teams; Grafana for pipeline runtimes
- **DVC metrics diff in PRs**: GitHub Actions that run `dvc metrics diff --show-json` and comment on PRs

## 22. Interview Questions
**Q1:** How does DVC handle versioning a 500GB dataset without storing 500GB in Git?  
*A1:* DVC stores the dataset's MD5 hash in a small `.dvc` file committed to Git. The actual data lives in content-addressed object storage (S3). When checking out a Git commit, `dvc checkout` uses the hash to find blobs in the local cache or pulls from remote.

**Q2:** Design a system where DVC pipelines auto-trigger on data changes.  
*A2:* Use a webhook on S3 (S3 Event Notification → SQS → Lambda) that calls `dvc commit` on the pipeline runner. Or use CI triggers: `dvc diff` in GitHub Actions checks if data.dvc changed; if so, run `dvc repro`.

**Q3:** What happens when two data scientists modify the same dataset simultaneously?  
*A3:* DVC follows Git merge semantics. When both push, Git merge resolves which `.dvc` file wins (or triggers conflict). Each version points to a different hash. The "main" branch has the canonical version; merge conflicts require manual reconciliation.

## 23. Cheat Sheet
```bash
# Initialize
dvc init
dvc remote add -d myremote s3://my-bucket/dvc-store

# Track data
dvc add data/raw/train.csv          # creates train.csv.dvc
git add train.csv.dvc .gitignore && git commit

# Push/Pull
dvc push
dvc pull

# Pipelines
dvc init                         # also sets up pipeline dirs
dvc run -n preprocess -p lr \
       -d src/preprocess.py \
       -o data/processed/ \
       python src/preprocess.py

dvc repro                        # reproduce pipeline
dvc dag                          # visualize DAG
dvc diff                         # show changes since last commit
dvc metrics diff                 # show metric deltas
```

## 24. Common Mistakes
1. **Adding data to Git instead of DVC**: Forgot to `.gitignore` — repo becomes huge
2. **`dvc push` before `git commit`**: Remote stores blobs that no commit references — orphaned
3. **Not using `dvc.lock`**: Pipeline outputs drift from dependencies without lock file
4. **Cross-filesystem symlinks**: Using `type=symlink` across mount points → broken links
5. **Committing secrets in `.dvc/config`**: Credentials leak — use `--local` flag
6. **Running `dvc repro` without pulling data**: Fails with file-not-found; order is `dvc pull && dvc repro`
7. **Ignoring `dvc gc`**: Cache grows unbounded; schedule `dvc gc -w -r` weekly
8. **Large single `.dvc` file**: Tracking 10K files in one `.dvc` → slow `dvc status`; split by subdirectory

## 25. Production Best Practices
- Always run `dvc pull` as first step in CI/CD pipelines
- Use `dvc repro --downstream` to only rebuild stages after a specific change point
- Keep `params.yaml` in Git — all hyperparameters versioned alongside code
- Set up DVC as a GitHub Action: `iterative/setup-dvc@v1` + `dvc pull` + `dvc repro`
- Enable `dvc metrics diff` in PR comments so reviewers see accuracy impact
- Use `dvc commit` instead of `dvc repro` when manually adding pre-trained models
- Store `dvc.lock` in Git — it freezes the exact dependency graph for reproducible runs
- Configure `core.remote_context` to enable per-branch remote isolation
- Run `dvc gc` weekly in CI to clean orphaned cache blobs
- Use DVC experiments (`dvc exp run`) for parameter sweeps without polluting Git history
