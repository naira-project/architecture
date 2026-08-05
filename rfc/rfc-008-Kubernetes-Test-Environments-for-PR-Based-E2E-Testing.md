# RFC-008: Ephemeral Kubernetes Test Environments for PR-Based E2E Testing
| Field            | Value                                                            |
|------------------|---------------------------------------------                     |
| RFC              | 008                                                              |
| Title            | Ephemeral Kubernetes Test Environments for PR-Based E2E Testing  |
| Author(s)        | @jagdeepm                                                        |
| Target Milestone | M1 - Foundation                                                  |
| Status           | Proposed                                                         |
| Type             | Feature                                                          |
| Created          | 2026-08-05                                                       |  


## Summary

This RFC proposes a lightweight approach for provisioning isolated Kubernetes test environments for pull request (PR) integration and end-to-end (E2E) testing.

The first iteration will use a **shell script, `envsubst`, and `kubectl`** to create and manage test environments. Application and infrastructure manifests will remain in the application source repository under `deploy/e2e/`. Each PR execution will create an isolated Kubernetes namespace, deploy the required application components and dependencies, initialize configuration and test data, run tests, and clean up the environment.

**The default is to deploy the entire stack per PR** (`--profile full`), giving a clean slate by construction: every dependency starts empty, so no seed-naming scheme, cleanup protocol, or concurrency gate is needed. `kubectl delete namespace` is the whole teardown.

A second profile (`--profile core`) shares the heavy dependencies to trade isolation for speed. It is documented here but should only be implemented if the `full` setup time proves painful in practice — it reintroduces five mechanisms whose sole purpose is simulating what `full` gets for free. See "Deployment Profiles" for the trade, and "Isolation Scope" for what each one actually guarantees.

Environments run either on a kind cluster created on the GitHub runner (Mode A) or on a **remote shared cluster (Mode B)**, which is the target — kind-on-runner is marginal at the `full` profile on CPU and disk. Mode B keeps namespace-per-PR isolation and requires two small plugin changes to stop PRs reading each other's Kubernetes state; see "Remote Cluster Deployment" and "Prerequisite Code Changes". Per-PR virtual clusters are deferred to a later iteration ("Future: vCluster").

Future iterations may evolve the solution toward a Kubernetes-native Environment Controller using technologies such as **Kubernetes Operators, kro, KubeVela, Crossplane, or GitOps tooling**, depending on the complexity and operational requirements observed during the initial implementation.

The initial implementation intentionally avoids introducing a custom controller or additional platform dependencies before the core workflow has been validated.

---

## Problem Statement

Integration and E2E tests often require multiple services and infrastructure components, such as:

- Core API Application
- Plugins
- Databases
- Test data and fixtures

Running these components directly on a GitHub-hosted runner may result in:

- CPU and memory contention
- Slow or unstable test execution
- Resource exhaustion
- Long workflow execution times
- Difficult debugging
- Limited support for parallel environments

A shared Kubernetes test cluster can provide reusable compute capacity. However, multiple PRs running on the same cluster require:

- Environment isolation
- Unique configuration
- Clean and deterministic test state
- Reliable readiness checks
- Resource limits
- Automatic cleanup
- Protection against concurrent test interference

The proposed solution addresses these requirements by provisioning an isolated namespace for each test execution.

---

## Goals

The first iteration must:

- Create an isolated Kubernetes namespace for each PR or test execution.
- Deploy the application **and its dependencies** per PR by default, using `envsubst` + `kubectl apply` and Helm.
- Support execution against:
    - A Kubernetes cluster created on a GitHub runner.
    - A remote shared Kubernetes test cluster.
- Inject environment-specific configuration.
- Initialize a clean and deterministic test dataset.
- Wait for application and dependency readiness.
- Provide the environment information required by the E2E test suite.
- Collect logs and diagnostic information when tests fail.
- Delete the environment after testing, leaving nothing behind.
- Support safe execution of multiple PR environments on the same cluster.

---

## Non-Goals

The first iteration will not implement:

- A custom Kubernetes Operator.
- A dedicated Environment Controller.
- A new REST API for environment provisioning.
- Multi-cluster scheduling.
- Automated cloud infrastructure provisioning.
- Advanced environment reuse or pooling.
- Full GitOps-based environment management.
- A persistent environment inventory database.

These capabilities may be considered in later iterations.

---

## Deployment Profiles

`create-environment.sh --profile <core|full>` selects how much of the stack each PR gets its own copy of. **`full` is the default.**

| | `full` *(default)* | `core` |
|---|---|---|
| Deployed per PR | everything | catalog, portal, ui, mlflow |
| Shared | nothing | litellm, litellm-db, inference, openmetadata |
| Clean slate | by construction | simulated via prefix naming + GC |
| Concurrent PRs | unlimited | 1 (serialised on OpenMetadata) |
| Setup time | ~7–13 min | ~1–2 min |
| Resource cost | ~3 CPU / ~7 GB | ~1 CPU / ~1.2 GB |

`core` exists to be implemented *later, if needed*. It reintroduces prefix-based seed naming, `hardDelete` semantics, a GC sweeper, a concurrency gate, and a namespace allowlist — five mechanisms whose failure modes are silent and concurrency-dependent. `full` has one failure mode: it is slow. Measure first; build `core` only if the wait proves intolerable.

### `full` — everything per PR

All of this lands in the PR's namespace and dies with it:

| Component | What it is |
|---|---|
| `catalog` Deployment | Go service + 6 plugin sidecar containers |
| `portal` Deployment | OpenMFP portal |
| `ui` Deployment | React UI |
| `mlflow` Deployment + PVC | MLflow tracking server (SQLite backend) |
| `litellm` Deployment + ConfigMap | AI gateway |
| `postgres` Deployment + PVC | LiteLLM backing store |
| `llama-dummy-model` Deployment + PVC | CPU inference backend (`replicas: 2`) |
| OpenMetadata (Helm) | Server + MySQL + OpenSearch, `airflow.enabled: false` |
| `catalog-secrets` Secret, `catalog` ServiceAccount | — |
| ClusterRoles + ClusterRoleBindings | Suffixed with `ENV_ID` to avoid cross-PR name collisions |

Every plugin DNS env var points at the PR's own namespace:

```
plugin-litellm           → litellm.${NAMESPACE}.svc.cluster.local:4000
plugin-depl-uses-litellm → litellm.${NAMESPACE}:4000
plugin-mlflow            → mlflow.${NAMESPACE}.svc.cluster.local:5000
plugin-openmetadata      → openmetadata.${NAMESPACE}.svc.cluster.local:8585
```

### `core` — shared dependencies

Under `core`, the four namespaces below are pre-deployed once on the cluster and the plugin DNS env vars point at them instead. This is the configuration that requires everything in "Clean Slate" and "Concurrency Gate".

| Namespace | Contents |
|---|---|
| `litellm` + `litellm-db` | LiteLLM and its PostgreSQL |
| `inference` | `llama-dummy-model` |
| `openmetadata` | OpenMetadata + MySQL + OpenSearch |
| `monitoring` | Prometheus + Grafana — skip in CI either way; enable for debugging |

### Excluded from CI under both profiles

- **vLLM** (`vllm-opt125m`) — requires a GPU node.
- **`llama-qwen25-05b`** — downloads a full-size model; use `llama-dummy-model` instead.

> **`llama-dummy-model` still performs a network download.** Its init container fetches
> `tiny-llama3-test-Q2_K.gguf` from HuggingFace whenever the PVC is empty
> (`deploy/dev/stacks/llm-inference/infra/kubernetes/llama.yaml:141-152`). Under `full` each PR
> gets a fresh PVC, so **every PR re-downloads it** — an external dependency on HuggingFace
> availability and rate limits. The model is small, but this is a real flakiness source. If it
> bites, bake the GGUF into a custom image or pre-seed it into the kind node image.

---

## Isolation Scope — What This Does and Doesn't Cover

**Iteration 1 delivers write isolation, not read isolation.** This is a deliberate, documented limit, not an oversight. Understanding it is a prerequisite for writing tests that pass reliably.

Prefix-based seeding (below) makes each PR's *writes* uniquely named. But every catalog plugin queries its backing service **globally, with no filter**. Verified in source:

| # | Plugin | Call site | Behaviour |
|---|---|---|---|
| 1 | `plugin-mlflow` | `plugins/cmd/mlflow/main.go:126` | `registered-models/search`, `max_results=1000`, no filter — returns every model in the registry |
| 2 | `plugin-openmetadata` | `plugins/cmd/openmetadata/main.go:207` | `/api/v1/tables`, `limit=100`, no service filter — returns tables from every service |
| 3 | `depl-calls-svc`, `depl-uses-litellm`, `fluxcd` | `plugins/internal/kubeutil/namespaces.go:23` | Lists **all** namespaces unconditionally, then iterates them listing Deployments and Services |
| 4 | `plugin-litellm` (AppIdentity) | `plugins/cmd/litellm/appidentity_provider.go:37` | `Namespace(metav1.NamespaceAll)`, no label selector — returns AppIdentity resources from every namespace |

Suffixing ClusterRole *names* with `ENV_ID` prevents apply-time collisions. It does not narrow the *permissions*, which remain cluster-wide list/get. Name uniqueness is not data isolation.

**Path 4 is dormant, not absent.** The `naira.io/v1alpha1` CRD does not exist in the repo, so the lookup currently fails and is swallowed (`plugins/cmd/litellm/main.go:89-93` logs and continues). The moment that CRD is installed on a shared cluster, this becomes a live cross-PR leak — and it does **not** route through `kubeutil.NamespacesAndClusterID`, so the namespace allowlist below does not cover it. See "Prerequisite Code Changes".

### What each profile actually guarantees

| Read path | `full`, cluster per PR | `full`, shared cluster | `core` |
|---|---|---|---|
| `plugin-mlflow` | clean | clean — empty registry | contaminated |
| `plugin-openmetadata` | clean | clean — 5 tables, ceiling never binds | contaminated + 100-table ceiling |
| Kubernetes-graph plugins (paths 3–4) | clean — one namespace set exists | **needs both code changes** | **needs both code changes** |

The Kubernetes plugins enumerate cluster-wide, so deploying every *service* per PR does not help them: on a shared cluster PR-123 still sees PR-124's namespace and Deployments. **Only an ephemeral cluster per PR closes every path with no code change.**

On a shared cluster, close paths 3 and 4 with the two changes in "Prerequisite Code Changes": a namespace allowlist for path 3, and scoping the AppIdentity list for path 4.

> **`clusterID` is shared on a shared cluster.** It is derived from the `kube-system` namespace UID (`namespaces.go:29-31`), which is identical for every PR on one host. Catalog node IDs therefore cannot distinguish environments by cluster. Tolerable while each PR reads only its own namespace, but it is a real limitation of namespace-based isolation — and one of the things a per-PR virtual cluster would fix properly (see "Future: vCluster").

### Consequences for test design

Under `full`, every dependency starts empty, so tests may assert freely on counts, completeness, and exact node sets.

Under `core`, only existence assertions on `ENV_ID`-scoped names are safe. Counts and completeness will see other PRs' data and fail intermittently. This asymmetry is the strongest practical argument for `full`: it is what lets the suite assert on what the catalog actually returns rather than merely that a row exists somewhere in it.

### Known ceilings

These bind under `core`. Under `full` each is either eliminated or never reached.

- **MLflow runs on SQLite.** `deploy/dev/stacks/mlops/infra/kubernetes/mlflow.yaml:54` uses `sqlite:////mlflow-data/backend/mlflow.db`, a single-writer lock. Concurrent seeds against one shared instance contend and fail with `database is locked`. Per-PR MLflow gives each PR its own database file — eliminated under both profiles, since `core` also deploys MLflow per PR.
- **OpenMetadata drops data past 100 tables.** `plugins/cmd/openmetadata/main.go:207` requests `limit=100` with no `order_by` and no pagination; the TODO at `main.go:205` confirms later pages are silently dropped. Under `core`, once tables across all PRs plus orphans exceed 100, **a PR's own seeded tables may be absent**, non-deterministically — worse than a dirty read, because it fails the PR's own assertions with no error. Under `full` each instance holds 5 tables and the ceiling is unreachable.
- **No caching.** Verified: the catalog has no cache layer, so plugin reads are live. No stale-read race between seed completion and test start, under either profile.

---

## Concurrency Gate — `core` only

Under `full` there is no shared mutable state, so PRs run in parallel with no gate. Cluster capacity is the only limit.

Under `core`, exactly one PR at a time may hold the shared OpenMetadata instance:

```yaml
concurrency:
  group: e2e-shared-openmetadata
  cancel-in-progress: false   # cancelling mid-run skips the Cleanup Job
```

This is the throughput cost that makes `core`'s faster setup partly illusory: environment build is quicker, but PRs queue behind one another. Whether `core` is actually faster end-to-end depends on how many PRs are in flight — with more than about three concurrent, `full` may well win despite the slower build.

---

## Clean Slate

### Under `full` — by construction

Every dependency is deployed fresh and empty for the PR. The seed scripts run unmodified, with their default entity names (`kind-standalone-demo`, `demo-model`, `naira_sample`), because no other PR shares the instance. Nothing needs prefixing, nothing needs cleaning up, and deleting the namespace removes every trace.

This is the entire clean-slate story for the default profile. The rest of this section applies only to `core`.

### Under `core` — simulated via prefix naming

Both seed scripts create entities with fixed global names, which collide inside a shared service:

| Script | What it creates |
|---|---|
| `register_dummy_model.py` | MLflow experiment `kind-standalone-demo`, model `demo-model` |
| `seed_sample_tables.py` | OpenMetadata service `naira_sample`, database `analytics`, 5 tables |

Pass `ENV_ID` in so each PR creates uniquely named entities:

| Script | Env var | Example for PR 123 |
|---|---|---|
| `register_dummy_model.py` | `MLFLOW_EXPERIMENT`, `MLFLOW_MODEL_NAME` (already exist) | `e2e-pr-123`, `e2e-pr-123-demo-model` |
| `seed_sample_tables.py` | `OPENMETADATA_SERVICE` (add one `os.getenv` line) | `naira_sample_pr123` |

This makes *writes* unique. It does nothing about the unfiltered *reads* documented in "Isolation Scope" — which is why `core` also needs the concurrency gate.

### LiteLLM

No seeding is required under either profile: LiteLLM is a gateway with a static model list in its ConfigMap, and today's tests only read that list.

Note the boundary. LiteLLM has a PostgreSQL store (`litellm-db`) holding keys, teams, and spend records. Under `full` that store is per-PR and the point is moot. Under `core` it is shared, so the moment a test exercises key creation or spend tracking it becomes shared mutable state with no isolation story.

---

## Environment Lifecycle

### `full` (default)

```
PR opened / commit pushed
         |
         v
GitHub Actions triggers workflow        (no concurrency gate)
         |
         v
create-environment.sh --profile full
  1. Derive ENV_ID = pr-<N>-<run-id>
  2. Create namespace + resource quota
  3. envsubst all manifests with NAMESPACE + ENV_ID + TAG → kubectl apply
  4. helm install openmetadata-deps + openmetadata --namespace ${NAMESPACE}
  5. Wait: openmetadata → litellm → mlflow → llama → catalog /healthz
         |
         v
Seed Job runs  — default entity names; the instances are empty
         |
         v
E2E tests run  (counts and completeness assertions are safe here)
         |
         v
destroy-environment.sh
  - kubectl delete namespace <ENV_ID>     ← takes everything with it
  - kubectl delete clusterrole/clusterrolebinding *-<ENV_ID>
```

No Cleanup Job, no GC sweeper, no orphan class. Namespace deletion is the entire teardown.

### `core` (if implemented)

```
         ... Concurrency gate: wait for the e2e-shared-openmetadata lane
         |
         v
create-environment.sh --profile core    (core + mlflow only)
         |
         v
Seed Job     → e2e-<ENV_ID> in per-PR MLflow
             → naira_sample_<ENV_ID> in SHARED OpenMetadata
         |
         v
E2E tests    (existence assertions only)
         |
         v
Cleanup Job  → DELETE naira_sample_<ENV_ID>?recursive=true&hardDelete=true
         |
         v
destroy-environment.sh  → delete namespace + cluster-scoped resources
         |
         v
[out of band] GC sweeper CronJob
  - Deletes orphaned naira_sample_* services older than N hours
  - Covers cancelled runs and crashes, where Cleanup never fired
```

---

## Repository Structure

```
deploy/
├── dev/                          ← existing local-dev, unchanged
│   └── stacks/core/infra/kubernetes/
│       ├── catalog.yaml
│       ├── portal.yaml
│       └── ui.yaml
└── e2e/                          ← new
    ├── catalog.yaml              ← envsubst template (NAMESPACE, ENV_ID, TAG)
    ├── portal.yaml
    ├── ui.yaml
    ├── mlflow.yaml               ← both profiles
    ├── litellm.yaml              ← full only
    ├── postgres.yaml             ← full only
    ├── llama.yaml                ← full only (dummy model, no qwen/vLLM)
    ├── openmetadata-values.yaml       ← full only, Helm overlay
    ├── openmetadata-deps-values.yaml  ← full only, Helm overlay
    ├── openmetadata-secrets.yaml      ← full only
    ├── quota.yaml
    ├── values-e2e.yaml           ← Helm values overlay (Mode B)
    ├── seed-job.yaml
    ├── cleanup-job.yaml          ← core only
    ├── gc-sweeper-cronjob.yaml   ← core only, deployed once to the cluster
    ├── ttl-sweeper-cronjob.yaml  ← Mode B, deployed once to the host cluster
    └── scripts/
        ├── create-environment.sh
        └── destroy-environment.sh

deploy/crds/
└── naira.io_appidentities.yaml   ← new; see Prerequisite Code Changes

deploy/charts/                    ← existing; the Mode B deploy vehicle
├── catalog/
└── ui-poc/
```

The `e2e/` manifests are parameterised copies of the `dev/` ones: `NAMESPACE` and `ENV_ID` are substituted at apply time, `ClusterRole`/`ClusterRoleBinding` names are suffixed with `ENV_ID`, and the plugin DNS env vars point at whichever instance the profile provides.

**Templating is not a copy-paste job.** Every resource across these files hardcodes its namespace (`idp-system`, `litellm`, `litellm-db`, `inference`, `mlflow`), and each file declares its own `Namespace` object. All must become `${NAMESPACE}`, including the `ClusterRoleBinding` subjects that reference the `catalog` ServiceAccount by namespace. Miss one and resources land in the shared namespace and collide across PRs.

For OpenMetadata under `full`, the two Helm releases take `--namespace ${NAMESPACE}`. Helm releases are namespace-scoped, so the release names need no suffix.

---

## Seed and Cleanup Jobs

### Seed Job (`deploy/e2e/seed-job.yaml`)

The same Job serves both profiles; only the endpoint and naming env vars differ.

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: seed-${ENV_ID}
  namespace: ${NAMESPACE}
spec:
  template:
    spec:
      restartPolicy: Never
      containers:
        - name: seed
          image: ghcr.io/naira-io/seed:${TAG}
          env:
            - name: MLFLOW_TRACKING_URI
              value: http://mlflow.${NAMESPACE}.svc.cluster.local:5000
            - name: OPENMETADATA_HOST_PORT
              value: ${OPENMETADATA_URL}
            # full: omit the name overrides — instances are empty, defaults are fine.
            # core: set MLFLOW_EXPERIMENT / MLFLOW_MODEL_NAME / OPENMETADATA_SERVICE
            #       to ENV_ID-scoped values (see "Clean Slate").
```

Under `full`, `OPENMETADATA_URL` resolves to `http://openmetadata.${NAMESPACE}.svc.cluster.local:8585`; under `core`, to the shared `openmetadata` namespace.

### Cleanup Job — `core` only

Under `full` nothing outlives the namespace, so there is no Cleanup Job.

Under `core`, the shared OpenMetadata entities need an API-level delete. Same image, `--cleanup` flag:

```
DELETE /api/v1/services/databaseServices/naira_sample_{ENV_ID}?recursive=true&hardDelete=true
```

**Both query parameters are required.** `recursive=true` cascades to tables and lineage; without `hardDelete=true` OpenMetadata performs a *soft* delete, leaving rows that still count toward the `limit=100` ceiling.

> If a future iteration shares MLflow too, note that `experiments/delete` is also a soft delete **and does not remove registered models**. That cleanup would additionally need `registered-models/delete` — the registry entry is exactly what `plugin-mlflow` reads, so skipping it leaves the contamination in place.

### GC Sweeper — `core` only

The Cleanup Job cannot be relied upon: a cancelled workflow, a runner OOM, or a hard crash skips it, and every skip leaves an orphan pushing the shared instance toward the 100-table ceiling.

Deploy a CronJob once to the cluster that lists `databaseServices` matching `naira_sample_*` and hard-deletes any older than a retention window — start at 4 hours, comfortably longer than any E2E run and short enough that orphans never accumulate.

Under `full` this whole class of problem does not exist.

---

## Resource Cost

Declared requests summed from the repo manifests. Rows marked *(est.)* are upstream Helm chart defaults — the OpenMetadata values files override only `airflow.enabled: false`, so everything else is chart default and must be measured (see "Verification").

| Group | Component | CPU | Memory | PVC |
|---|---|---|---|---|
| **Core** | catalog (main) | 100m | 128Mi | — |
| | 6 plugin sidecars | 300m | 384Mi | — |
| | portal | 100m | 128Mi | — |
| | ui | 50m | 64Mi | — |
| | mlflow | 250m | 512Mi | 10Gi |
| | *`core` subtotal* | **800m** | **1216Mi** | **10Gi** |
| **Inference** | litellm | 250m | 512Mi | — |
| | postgres | 100m | 256Mi | 5Gi |
| | llama-dummy-model ×2 | 100m | 128Mi | 3Gi |
| **OpenMetadata** | server *(est.)* | ~1000m | ~2Gi | — |
| | MySQL *(est.)* | ~250m | ~1Gi | ~8Gi |
| | OpenSearch *(est.)* | ~500m | ~2Gi | ~10Gi |
| | **`full` TOTAL** | **~3 CPU** | **~7.1 GB** | **~36 Gi** |

**OpenMetadata alone is ~1.75 CPU / ~5 GB / ~18 Gi — about 60% of the `full` footprint for one dependency.** Everything else combined is ~1.25 CPU / ~2 GB.

On kind the local-path provisioner is hostPath-backed, so actual disk consumed is real usage (~1 GB), not the 36 Gi declared. On a cloud cluster with dynamic provisioning, 36 Gi per PR is genuinely allocated and billed.

### Setup time

The existing taskfile timeouts are the honest signal for how slow these are:

| Step | Budget in repo | Realistic |
|---|---|---|
| OpenMetadata deps (MySQL + OpenSearch), `--wait` | 15 min | 3–5 min |
| OpenMetadata server, `--wait` | 15 min | 2–4 min |
| LiteLLM rollout | 15 min | 1–3 min |
| MLflow / catalog rollout | 3 min each | <1 min |

Sequential worst case is ~40 min budgeted; **realistic is 7–13 min** for `full`, versus ~1–2 min for `core` against warm shared services.

### Quotas

```yaml
# full
requests.cpu: "3"
requests.memory: 8Gi
limits.cpu: "8"
limits.memory: 14Gi
pods: "25"
persistentvolumeclaims: "6"
```

```yaml
# core
requests.cpu: "1"
requests.memory: 1536Mi
limits.cpu: "4"
limits.memory: 3Gi
pods: "12"
persistentvolumeclaims: "2"
```

### Runner capacity

Standard `ubuntu-latest` is 4 vCPU / 16 GB RAM / ~14 GB free disk, and `deploy/dev/stacks/core/infra/kind/kind-config.yaml` declares a **single control-plane node**. Against `full`:

- **RAM** — ~7 GB of 16 GB. Comfortable.
- **CPU** — ~3 of 4 requested, leaving ~1 core for kubelet, containerd, and the test process. Expect slow tests and timeout flakiness.
- **Disk** — upstream images (OpenMetadata ~2 GB, OpenSearch ~1.2 GB, LiteLLM ~1.5 GB, MLflow ~1 GB, MySQL ~600 MB, postgres ~400 MB, llama.cpp ~200 MB) plus nine locally-built images total ~9–10 GB against ~14 GB free. **Add a free-disk-space step** to the workflow.

`deploy/dev/stacks/mlops/infra/helm/openmetadata-deps-values.yaml` records that Airflow was disabled specifically to relieve "node pressure that was OOMKilling OpenSearch" — with a single copy of the stack. Treat that as evidence this is genuinely at the edge on a runner.

On a remote cluster, 5 concurrent `full` PRs need ~15 CPU / ~35 GB / ~180 Gi. That is a dedicated cluster.

> **`model-cache-pvc` is ReadWriteOnce with `replicas: 2`.** Both pods co-schedule on single-node kind, so this works there. On a multi-node remote cluster the second replica may fail to schedule — set `replicas: 1` for E2E or switch the volume to `emptyDir`.

---

## Image Strategy

**Locally built** (change with each PR):

| Image | Source |
|---|---|
| `catalog` | `./catalog` |
| `mlflow` plugin | `./plugins/cmd/mlflow` |
| `litellm` plugin | `./plugins/cmd/litellm` |
| `depl-calls-svc` | `./plugins/cmd/depl_calls_svc` |
| `depl-uses-litellm` | `./plugins/cmd/depl_uses_litellm` |
| `fluxcd` | `./plugins/cmd/fluxcd` |
| `openmetadata` plugin | `./plugins/cmd/openmetadata` |
| `portal` | `./naira-openmfp-portal` |
| `ui` | `./ui-poc` |

```bash
# Mode A — kind on runner
docker build -t catalog:${TAG} ./catalog
kind load docker-image catalog:${TAG} --name naira-idp

# Mode B — remote cluster: push to a registry instead
docker build -t ghcr.io/naira-project/naira-catalog:${TAG} ./catalog
docker push ghcr.io/naira-project/naira-catalog:${TAG}
```

`kind load` has no remote equivalent. The Helm charts already default to `ghcr.io/naira-project/naira-catalog`, so Mode B needs `image.tag` set per run and an `imagePullSecrets` entry for private images.

**Upstream** (LiteLLM, llama.cpp, MLflow server, OpenMetadata, MySQL, OpenSearch, PostgreSQL): pulled from public registries. On a remote cluster these stay warm in the node image cache between runs — the main reason a cold `full` build is materially faster there than on a runner, where all ~9–10 GB is re-pulled every time.

---

## Remote Cluster Deployment (Mode B)

Kind-on-runner is marginal at the `full` profile: ~3 CPU of requests against 4 vCPU, ~9–10 GB of images against ~14 GB of free disk, on a single node. A remote shared cluster removes both limits and gives warm image caches. It keeps namespace-per-PR isolation — per-PR virtual clusters are deferred (see "Future: vCluster").

Moving there changes four things: image delivery (above), packaging, authentication, and read isolation.

### Packaging: use the existing Helm charts

`deploy/charts/catalog` and `deploy/charts/ui-poc` already exist and already do what this RFC otherwise describes doing by hand:

- ClusterRole names are **release-scoped** via `catalog.appIdentitiesRoleName` → `catalog.fullname`, so the manual `ENV_ID` suffixing is unnecessary for chart-managed resources
- `image.repository` / `image.tag`, `imagePullSecrets`, `serviceAccount`, and `rbac.*.create` are all parameterised

`envsubst` was a local-kind convenience. For a remote cluster the charts are the correct vehicle and remove most of the templating hazard flagged under "Repository Structure".

**Gap:** charts cover only `catalog` and `ui-poc`. Portal, MLflow, LiteLLM, postgres, and llama remain raw manifests. Run a hybrid path for iteration 1 — `helm install` for the two, `envsubst | kubectl apply` for the rest — and author the missing charts later.

### Host cluster prerequisites

| Requirement | Detail |
|---|---|
| Default StorageClass, RWO dynamic provisioning | MLflow 10Gi, postgres 5Gi, llama cache 3Gi, plus OpenMetadata's MySQL and OpenSearch |
| Capacity | ~3 CPU / ~7.1 GB / ~36 Gi per PR; 5 concurrent ≈ 15 CPU / 35 GB / 180 Gi |
| OIDC federation from GitHub Actions | Preferred over a long-lived kubeconfig secret |
| Registry pull access | `imagePullSecrets` for private GHCR images |
| Flux toolkit CRDs | `kustomize.toolkit.fluxcd.io/v1`, `helm.toolkit.fluxcd.io/v2`, `source.toolkit.fluxcd.io/v1` — only if the fluxcd plugin path is under test |
| Gateway API CRDs *(optional)* | Only if `ui-poc` sets `httpRoute.enabled`; in-cluster tests do not need it |
| metrics-server *(optional)* | For `kubectl top` during capacity tuning |

### Teardown

Namespace deletion does **not** remove ClusterRoles or ClusterRoleBindings. `helm uninstall` cleans up chart-managed ones; anything applied raw needs an explicit labelled delete.

Add a **TTL sweeper CronJob** on the host that deletes namespaces carrying `naira.io/environment: e2e` older than N hours. Cancelled workflows and runner crashes skip the teardown step, and an orphaned namespace holds its full quota reservation indefinitely — on a shared cluster that starves other PRs rather than merely leaving clutter.

---

## GitHub Actions Integration

```yaml
# No concurrency group under `full` — nothing is shared, so PRs run in parallel.
# Under `core`, add:
#   concurrency:
#     group: e2e-shared-openmetadata
#     cancel-in-progress: false   # cancelling mid-run skips the Cleanup Job

jobs:
  e2e:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      # ~9-10 GB of images against ~14 GB free; reclaim preinstalled toolchains first.
      - name: Free disk space
        run: |
          sudo rm -rf /usr/share/dotnet /usr/local/lib/android /opt/ghc
          df -h /

      - name: Create kind cluster
        run: kind create cluster --name naira-idp

      - name: Build and load images
        run: |
          docker build -t catalog:${{ github.sha }} ./catalog
          kind load docker-image catalog:${{ github.sha }} --name naira-idp
          # repeat for plugins, portal, ui

      - name: Create E2E environment
        run: |
          ./deploy/e2e/scripts/create-environment.sh \
            --profile full \
            --pr ${{ github.event.pull_request.number }} \
            --tag ${{ github.sha }}

      # Mode B replaces the three steps above with:
      #
      #   - name: Authenticate to host cluster
      #     uses: <cloud>/get-credentials-action    # OIDC, no stored kubeconfig
      #
      #   - name: Build and push images
      #     run: |
      #       docker build -t ghcr.io/naira-project/naira-catalog:${{ github.sha }} ./catalog
      #       docker push  ghcr.io/naira-project/naira-catalog:${{ github.sha }}
      #       # repeat for plugins, portal, ui
      #
      #   - name: Create E2E environment
      #     run: |
      #       ./deploy/e2e/scripts/create-environment.sh \
      #         --profile full --mode remote \
      #         --pr ${{ github.event.pull_request.number }} \
      #         --tag ${{ github.sha }}
      #
      # The "Free disk space" step is unnecessary in Mode B.

      - name: Run E2E tests
        run: go test ./tests/e2e/... -env-id ${{ env.ENV_ID }}

      - name: Collect diagnostics
        if: failure()
        run: |
          kubectl logs -n ${{ env.ENV_ID }} -l app=catalog
          kubectl get events -n ${{ env.ENV_ID }} --sort-by=.lastTimestamp
          kubectl describe pods -n ${{ env.ENV_ID }}   # OOMKills surface here

      # core only — under `full` the namespace delete below is the whole teardown.
      # - name: Clean up shared OpenMetadata entities
      #   if: always()
      #   run: kubectl wait --for=condition=complete job/cleanup-${{ env.ENV_ID }} -n ${{ env.ENV_ID }} --timeout=120s

      # Deletes the namespace AND the cluster-scoped resources it leaves behind.
      # Backed by the host TTL sweeper for runs that never reach this step.
      - name: Destroy environment
        if: always()
        run: ./deploy/e2e/scripts/destroy-environment.sh --env-id ${{ env.ENV_ID }}
```

---

## Namespace Labels

```yaml
metadata:
  labels:
    naira.io/environment: e2e
    naira.io/pr: "${PR_NUMBER}"
    naira.io/run-id: "${RUN_ID}"
    naira.io/managed-by: environment-script
```

---

## Readiness Check

Dependencies must be ready before the catalog, since its plugin sidecars connect on startup. Under `full` the script waits in this order:

1. `helm ... --wait` for OpenMetadata deps, then the server — the long pole, 5–9 min
2. `kubectl rollout status deploy/litellm -n ${NAMESPACE} --timeout=900s`
3. `kubectl rollout status deploy/mlflow -n ${NAMESPACE} --timeout=180s`
4. `kubectl rollout status deploy/llama-dummy-model -n ${NAMESPACE} --timeout=600s` — includes the HuggingFace download
5. `kubectl rollout status deploy/catalog -n ${NAMESPACE} --timeout=180s`
6. `curl -f http://<catalog-service>/healthz`
7. `kubectl wait --for=condition=complete job/seed-${ENV_ID} -n ${NAMESPACE}`

Steps 2–4 are independent and can run in parallel to claw back a few minutes. Under `core`, steps 1, 2, and 4 drop out.

No fixed sleeps. The catalog has no cache layer, so a completed seed Job is sufficient — reads are live and no settling delay is needed.

---

## Alternatives Considered

### Custom Environment Controller from the Beginning

Rejected for the first iteration because it introduces significant implementation and maintenance complexity before validating the environment lifecycle.

### KubeVela from the Beginning

Deferred because the first iteration does not require application workflows, multi-cluster delivery, or the full KubeVela platform model.

### kro from the Beginning

Deferred because the initial use case can be implemented directly using `envsubst` + `kubectl` and namespace-scoped Kubernetes resources.

kro may become useful when reusable, higher-level Kubernetes APIs are required.

### Argo CD Preview Environments

Deferred because the initial prototype does not require a GitOps control plane. It may be evaluated when environment definitions and deployment workflows become more complex.

### Running All Services Directly on GitHub Runners

Not preferred for larger environments because runner CPU, memory, disk, and execution limits may lead to unstable or slow E2E testing.

### Sharing All Dependencies (the original `core`-only design)

Rejected as the default. Sharing LiteLLM, OpenMetadata, and the inference backend saves ~2 CPU / ~6 GB per PR and ~10 minutes of setup, but it is the sole reason the design would need prefix-based seed naming, `hardDelete` semantics, a GC sweeper, a concurrency gate, and a namespace allowlist.

Those five mechanisms fail *silently* and *only under concurrency* — cross-PR reads, a soft delete that looks successful, a pagination ceiling that drops a PR's own data with no error. Those are precisely the bugs that erode trust in an E2E suite, and they are expensive to diagnose because they do not reproduce serially.

`full` has one failure mode: it is slow. Slow is visible on every run, measurable, and improves with parallelism. For a first iteration whose purpose is establishing that the suite can be trusted, that is the better trade. `core` remains specified as the escape hatch if setup time proves intolerable.

---

## Success Criteria

The first iteration will be considered successful when it can:

- Provision an isolated environment for a PR.
- Deploy the application and all its dependencies.
- Inject PR-specific configuration.
- Initialize a clean test dataset.
- Complete readiness checks without fixed sleep delays.
- Run integration and E2E tests.
- Support multiple environments in parallel on the same cluster.
- Collect useful diagnostics on failure.
- Clean up every environment resource.
- Operate using the same scripts against both a GitHub-runner cluster and a remote Kubernetes cluster.
- **Run the suite 20 times consecutively with zero cross-run contamination**, including assertions on counts and completeness — the practical test that clean slate actually holds.
- **Complete a cold `full` build in under 15 minutes.** If it exceeds that, `core` becomes worth building.

---

## Open Questions

| Question | Status |
|---|---|
| Which runner mode — kind on runner, or remote cluster? | Remote is the target; kind-on-runner is marginal at `full`. Both code changes in "Prerequisite Code Changes" are required for remote |
| Which host cluster / cloud? | Open — decides the OIDC auth action and StorageClass specifics |
| Does a cold `full` build fit in ~8 min on the host? | Open — measure; warm image caches should beat the 7–13 min runner figure |
| Are the OpenMetadata chart-default resources accurate? | Open — measure with `kubectl top`, see Verification |
| Author charts for portal, mlflow, litellm, postgres, llama, or keep a hybrid apply path? | Open — hybrid is fine for iteration 1 |
| TTL sweeper retention window on the host? | Open; start at 4h and tune |
| Should `llama-dummy-model`'s GGUF be baked into an image? | Open — depends on whether HuggingFace flakiness materialises |
| Max concurrent environments? | `full`: host capacity only. `core`: 1 |
| Add pagination to `plugin-openmetadata`? | Not needed under `full`; required if `core` is ever built |

---

## Prerequisite Code Changes

| Change | File | Size | Needed for |
|---|---|---|---|
| Namespace allowlist via `NAIRA_PLUGIN_NAMESPACES` — closes read path 3 | `plugins/internal/kubeutil/namespaces.go` + 3 call sites | ~15 lines | Any shared cluster |
| Scope the AppIdentity list away from `NamespaceAll` — closes read path 4 | `plugins/cmd/litellm/appidentity_provider.go:37` | ~5 lines | Any shared cluster, **before** the CRD is installed |
| Author the missing `naira.io/v1alpha1` `AppIdentity` CRD | *new* `deploy/crds/naira.io_appidentities.yaml` | new file | Testing the app-identity path at all |
| `OPENMETADATA_SERVICE` env var override | `.../tools/openmetadata/seed_sample_tables.py` | 1 line | `core` profile only |

The first two are what make a shared cluster safe; both are required, since the allowlist does not cover path 4. With a cluster per PR neither is needed — isolation comes from topology rather than code.

### The missing AppIdentity CRD

`naira.io/v1alpha1 appidentities` is consumed at `plugins/cmd/litellm/appidentity_provider.go:35` and granted in chart RBAC (`deploy/charts/catalog/templates/rbac.yaml:9-10`), but **no CRD manifest exists anywhere in the repo**.

Its absence degrades silently — `plugins/cmd/litellm/main.go:89-93` logs the failure and returns successfully. **That means the app-identity code path is untested today, and a green E2E run proves nothing about it.** Any test asserting on app-identity relations must install the CRD, seed an instance, and assert on *presence* of the expected relations rather than merely absence of error.

Order matters: land the path-4 scoping change before installing the CRD on a shared cluster, or the CRD turns a dormant leak into a live one.

---

## Verification

The OpenMetadata figures are chart defaults and the least certain input to the profile decision. Measure before committing:

```bash
task platform:deploy
task --taskfile deploy/dev/Taskfile.yml mlops:deploy

kubectl top pods -A --sum        # actual steady-state vs declared requests
kubectl get pvc -A               # real consumption vs declared capacity
```

Then time a cold `full` environment build end to end. Under ~8 minutes and `full` as default is comfortable — `core` may never need to exist. Over ~15 minutes and `core` becomes the default fast path, with `full` reserved for nightly and release branches.

For Mode B, two additional checks matter more than the timings:

1. **Prove isolation holds.** Run two PR environments concurrently and assert from PR-A's catalog API that no node references PR-B's namespace, Deployments, or Services. This is the entire premise of namespace-based isolation on a shared cluster — verify it before trusting the suite.
2. **Prove the app-identity path is exercised.** Install the CRD, seed an instance, and assert the catalog returns the expected relations. Without the CRD this silently passes while covering nothing, so the assertion must be on presence, not on absence of error.

Teardown check: after a run, no namespace carrying `naira.io/environment: e2e` remains, no orphaned PVCs on the host, and no ClusterRoles matching the environment label.

---

## Future: vCluster

Deferred to a later iteration, recorded so the reasoning is not re-derived.

[vCluster](https://www.vcluster.com/) gives each PR a real API server running as pods inside a host namespace — cluster-per-PR isolation at roughly namespace-per-PR cost. For this codebase it would:

- **Remove both prerequisite code changes.** A vCluster's namespace list is naturally its own, so read paths 3 and 4 close without the allowlist or the AppIdentity scoping.
- **Give each PR a distinct `clusterID`,** since every vCluster has its own `kube-system` namespace and therefore its own UID — fixing the limitation noted under "Isolation Scope".
- **Allow per-PR CRDs,** making CRD schema changes testable and stopping the AppIdentity CRD from being shared state.
- **Remove ClusterRole name collisions entirely,** since each vCluster has its own RBAC space.

Cost is roughly an extra 0.3–0.5 CPU / 0.5–1 GB per PR *(estimate — validate against a real vCluster before sizing)*, plus a new platform dependency to version and upgrade, and a layer of indirection when debugging a failed environment.

One point in its favour specific to Naira: **no plugin reads Kubernetes `Node` or `PersistentVolume` objects** — verified by grep; the `nodes` in this codebase are catalog graph nodes. Node virtualization is where vCluster diverges most from a real cluster, so the fidelity gap here is close to zero.

Revisit when per-PR CRDs are needed, or when the two isolation code changes start reading as workarounds rather than fixes.

---

## What's Out of Scope (for now)

- **vCluster / per-PR virtual clusters** — see above; namespace-per-PR first
- **`core` profile implementation** — specified here, built only if `full` proves too slow
- Charts for portal, mlflow, litellm, postgres, llama — hybrid `helm` + `kubectl apply` for iteration 1
- Pagination fix in `plugin-openmetadata` — only matters under `core`
- vLLM — requires a GPU runner
- `llama-qwen25-05b` — full-size model download
- Monitoring stack in CI — deploy alongside the cluster for debugging, never per PR
- Kustomize overlays — `envsubst` plus the existing Helm charts are sufficient
- Environment controller / CRD — revisit after iteration 1 is validated
