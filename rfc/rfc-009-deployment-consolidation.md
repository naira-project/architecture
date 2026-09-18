# RFC-009: One Deployment Path for Naira

| Field            | Value                                          |
|------------------|------------------------------------------------|
| RFC              | 009                                            |
| Title            | One Deployment Path for Naira                  |
| Author(s)        | @hosseinsalahi                                 |
| Status           | Proposed                                       |
| Type             | Feature                                        |
| Created          | 2026-09-11                                     |
| Updated          | 2026-09-18                                     |

## Summary

Naira has four ways to deploy itself and no single one an end user can consume. This RFC proposes:

- Two versioned Helm charts: `naira` (first-party components) and `test-dependencies`.
- Each published per release as an OCI Helm chart that ArgoCD consumes. An OCM component version carrying image digests, for signing and registry relocation, follows once OCM's Helm chart support is confirmed.
- ArgoCD deploys both from `test-dependencies`, pinned to chart versions.
- Tilt renders the same charts for the inner development loop.
- Flux is dropped completely.

## Current State

Four deployment paths exist, none a superset of another.

| Path | Location | Covers | Consumable by |
|---|---|---|---|
| Taskfile + raw manifests | `naira/deploy/dev/stacks/{core,llm-inference,mlops}` | full stack | local kind only |
| Helm charts | `naira/deploy/charts/{catalog,ui-poc}` | 2 components, neither completely | nothing, referenced by no workflow |
| Flux GitOps | `test-dependencies/infrastructure/` | OpenBao, ESO, MLflow, LiteLLM | shared testbed cluster |
| Tilt | `naira` branch `origin/tilt` (not merged) | core stack | one developer |

`task platform:deploy` is the only path producing a working Naira. It creates kind, builds 11 images locally, side-loads them with `kind load`, and applies raw manifests.

Component inventory the charts must account for:

| Tier | Components |
|---|---|
| First-party | catalog + 7 gRPC plugin sidecars (`litellm`, `mlflow`, `depl-calls-svc`, `depl-uses-litellm`, `fluxcd`, `openmetadata`, `mcp-servers`), ui, portal |
| Third-party | Keycloak, LiteLLM, MLflow, OpenMetadata, PostgreSQL |
| Dev-only | vLLM, llama.cpp, Prometheus/Grafana ServiceMonitors, `mcp-mock` |

## Problems

### P1: Local Kubernetes cluster and images

Eleven containers are referenced by `:local` tag across `deploy/dev/stacks/`: `catalog`, `ui`, `portal`, `mcp-mock` and the seven plugins. Ten have published equivalents; `mcp-mock` does not. Those tags exist only because the Taskfile builds them and `kind load` writes them into the node's containerd. Applied to any other cluster, all eleven fail to pull.

### P2: Helm charts

`deploy/charts/catalog/templates/deployment.yaml` renders one container. `deploy/dev/stacks/core/infra/kubernetes/catalog.yaml` runs eight: the catalog plus seven plugin sidecars, enumerated in a `plugin-config` ConfigMap. The chart has no concept of plugins. The two have drifted completely.

The charts are unreferenced by the Taskfile, by CI and by anything else.

### P3: OCI images

`dev-publish.yml` publishes 10 images. `release-please.yml` publishes 7. Missing from the release path:

- `naira-plugin-depl-uses-litellm`
- `naira-plugin-openmetadata`
- `naira-plugin-mcp-servers`

### P4: Kubernetes secrets

`test-dependencies` delivers LiteLLM's API key through an `ExternalSecret` backed by OpenBao via External Secrets Operator. The `naira` manifests carry literal `stringData` in six files:

| File | Values |
|---|---|
| `core/infra/kubernetes/catalog.yaml` | `LITELLM_API_KEY`, `OPENMETADATA_ADMIN_PASSWORD` |
| `core/infra/kubernetes/portal.yaml` | `client-secret` |
| `core/infra/keycloak/secret.yaml` | admin `username`, `password` |
| `llm-inference/infra/kubernetes/litellm.yaml` | `LITELLM_MASTER_KEY`, `OPENAI_API_KEY`, `ANTHROPIC_API_KEY`, `GEMINI_API_KEY` |
| `llm-inference/infra/kubernetes/postgres.yaml` | `POSTGRES_PASSWORD` |
| `mlops/infra/kubernetes/openmetadata-secrets.yaml` | MySQL and Airflow passwords |

### P5: Tilt vs. Taskfile

`task platform:deploy` costs a full image build, a node load and a rollout for every code change. Prohibiting Tilt without replacing what it provides recreates the same drift under a different name.

### P6: Three task runners

- `naira`: root `Taskfile.yml` delegating to `deploy/dev/Taskfile.yml` and eleven included task files.
- `test-dependencies`: `Makefile` with `testbed-*` and `platform-*` targets.
- `origin/tilt`: a Tiltfile.

Each encodes its own deployment order.

## Goals

- One definition of what a Naira deployment is, consumed by every environment.
- That definition is a versioned, immutable release artifact.
- An external adopter can install Naira into their own cluster from published artifacts, including into a private or air-gapped registry (via OCM, pending confirmation of its Helm chart support).
- The inner development loop stays fast and renders the same definition (via Tilt).
- Secrets are never committed in any path.
- One tool per job: Helm defines, ArgoCD reconciles, Tilt develops.

### CI requirements

Per pull request:

- Helm chart sanity checks.
- A script that checks whether code or Dockerfile changes are reflected in the chart, e.g. a new plugin missing from the chart's plugin list. Human review alone misses these.
- Deployment verified on a Kubernetes cluster.
- Compatibility check between the `naira` chart and `test-dependencies`. The PR vCluster surfaces breaking changes.

Pipelines: Naira core, test dependencies, staging, PR vCluster.

### Environments

All on vCluster. Free tier limit: 64 CPU.

| Environment | Purpose |
|---|---|
| PR | Per pull request verification |
| Dev | Hosts all components |
| Release-pinned | Fix against an older release, e.g. a bug fix for v1.1 while current is v1.2, runs on a vCluster at that version |
| Staging | Pre-release |

## Non-goals

- A single command that yields a fully working stack from nothing. The `naira` chart installs first-party components only. Dependencies are a separate umbrella chart with its own lifecycle.

## Proposal

### 1. Two Helm charts

`naira/deploy/charts/naira`: the product. Catalog including its plugin sidecars (plugins move to a separate repository later), ui, portal.

`test-dependencies/charts/test-dependencies`: the third-party tier. Keycloak, LiteLLM, MLflow, OpenMetadata, PostgreSQL declared as `Chart.yaml` dependencies at pinned upstream versions, vendored locally or mirrored to ECR as OCI.

The split follows lifecycle ownership. Naira's version moves with Naira's code; the dependency tier moves on upstream upgrade cycles unrelated to a Naira release.

OpenBao and External Secrets Operator also live in `test-dependencies`. They bootstrap the secret store, so they cannot be a dependency of something that needs secrets. They stay as separate ArgoCD Applications in earlier sync waves.

`deploy/dev/stacks/{core/infra/kubernetes,core/infra/keycloak,mlops}` is deleted once the charts reach parity.

`test-dependencies` carries every non-first-party component with an `enabled` flag: Keycloak, LiteLLM, PostgreSQL, MLflow, OpenMetadata, llama.cpp, vLLM, kube-prometheus-stack with its ServiceMonitors, and `mcp-mock`.

### 2. Values-driven plugin sidecars

The catalog chart takes a plugin list, not seven hardcoded container blocks:

```yaml
catalog:
  image:
    repository: ghcr.io/naira-project/naira-catalog
    tag: ""                    # empty -> .Chart.AppVersion
  pluginDefaults:
    resources:
      requests: { cpu: 50m, memory: 64Mi }
      limits: { cpu: 200m, memory: 128Mi }
  plugins:
    - name: litellm
      enabled: true
      image:
        repository: ghcr.io/naira-project/naira-plugin-litellm
        tag: ""
      port: 50051
      schedule: "0 0 * * *"
      env: []
    - name: depl-calls-svc
      enabled: true
      image:
        repository: ghcr.io/naira-project/naira-plugin-depl-calls-svc
        tag: ""
      port: 50053
      schedule: "0 * * * *"
      resources: {}            # optional; merged over pluginDefaults.resources
      rbac:
        rules:
          - apiGroups: [""]
            resources: ["namespaces", "services"]
            verbs: ["get", "list"]
          - apiGroups: ["apps"]
            resources: ["deployments"]
            verbs: ["get", "list"]
```

One template loop emits the containers and generates the `plugin-config` ConfigMap (`address: localhost:<port>`, `schedule`) from the same list. For each plugin with `rbac.rules`, it emits a ClusterRole and a ClusterRoleBinding to the catalog ServiceAccount. `depl-calls-svc`, `depl-uses-litellm` and `fluxcd` read cluster-scoped and cross-namespace resources, so a namespaced Role is insufficient. RBAC belongs to the plugin entry, not to a fixed template.

Plugin sidecars share the pod's network namespace. Ports must be unique; the chart fails rendering on a duplicate.

Each plugin's resources are `pluginDefaults.resources` with its own `resources` merged over them. The defaults match the current manifests; overrides are for heavier plugins. The pod's scheduling request is the sum across the catalog and every enabled plugin.

`enabled: false` drops a plugin. That is how an adopter runs a subset.

`tag: ""` falling back to `.Chart.AppVersion` removes the last hand-written tag from the repository.

### 3. OCI artifacts

| Artifact | Path | Consumer |
|---|---|---|
| 10 container images | `ghcr.io/naira-project/naira-*` | the charts |
| `naira` Helm chart | `ghcr.io/naira-project/charts/naira:X.Y.Z` | ArgoCD `source.chart`, `helm install` |
| `naira` OCM component version | `ghcr.io/naira-project/ocm/…` | adopters, signing, `ocm transfer` (gated, see below) |
| `test-dependencies` chart | `ghcr.io/naira-project/charts/test-dependencies:A.B.C` | ArgoCD and Tilt |

ArgoCD has no OCM source type; `source.chart` resolves OCI Helm only. Release CI runs `helm package` and `helm push`. ArgoCD, `helm install` and CI consume that chart without OCM.

The OCM component version is gated on confirmation from the OCM developers:

1. Can a component version carry an OCI Helm chart as a resource, by reference or as a local blob?
2. After `ocm transfer`, can `helm pull` consume the relocated chart?
3. Are the chart's image references localized to the target registry, or does the adopter override `image.repository`?

Once confirmed, release CI adds `ocm add component-version` (a descriptor referencing the chart and every image, each tag resolved to a digest) and `ocm transfer` to the release registry. ArgoCD deploys a tag-pinned chart; the OCM component, the artifact that is signed and relocated, is digest-pinned.

### 4. Tilt

A committed `Tiltfile` at the repository root, not under `deploy/dev/`, because it watches `catalog/`, `ui/`, `plugins/` and `naira-openmfp-portal/`:

- `helm_resource` on `charts/test-dependencies` from OCI with a dev values file.
- `helm()` on `deploy/charts/naira` with a dev values file overriding image repositories to local tags.
- `custom_build` per Go binary: host `go build` into a thin runtime image. The `go_image` helper on `origin/tilt` is the right shape.
- `k8s_yaml` for the dev-only manifests (`llm-inference`, `mcp-mock`).
- `local_resource` for the MLflow and OpenMetadata seed scripts, with `resource_deps` so ordering is declared rather than documented.
- `allow_k8s_contexts('kind-naira-idp')` retained as a guard.

Three root tasks are not deployment and move to `[tasks]` in `mise.toml`: `proto:generate`, `proto:lint`, `catalog:test`.

`kind create` / `kind delete` become README one-liners. `kind-config.yaml` declares a single control-plane node with a pinned image and no port mappings, so it needs no wrapper. `tilt down` removes resources, not the cluster, so the delete command is needed regardless.

### 5. ArgoCD deploys from `test-dependencies`

`test-dependencies` holds the `test-dependencies` chart source, ArgoCD `Application` and `ApplicationSet` definitions, per-environment values, and the OpenBao/ESO bootstrap. Each Application pins a chart version:

```yaml
source:
  repoURL: ghcr.io/naira-project/charts
  chart: naira
  targetRevision: 0.2.3
```

Sync waves:

| Wave | Contents |
|---|---|
| 0 | Namespaces, ESO CRDs |
| 1 | ESO controller, OpenBao |
| 2 | OpenBao init and seed |
| 3 | `ClusterSecretStore`, `ExternalSecret`s |
| 4 | `test-dependencies` |
| 5 | `naira` |

### 6. Repository naming

Two repositories both said "testbed" and neither said which job it does.

| Before | Holds | After |
|---|---|---|
| `naira-testbed` | Terraform and cluster provisioning | `test-infrastructure` |
| `component-testbed` | External charts and Applications deployed onto a cluster | `test-dependencies` |

The `naira*` prefix is reserved for Naira core. All repositories follow this convention.

## Consumers

| Audience | Path |
|---|---|
| Adopter | `helm install oci://ghcr.io/naira-project/charts/naira`, their own ArgoCD, or `ocm transfer` into a private registry first (after phase 8) |
| Naira developer, inner loop | Tilt |
| Naira developer, verifying the GitOps path | ArgoCD on kind against published images |
| CI | `helm install`, non-interactive |

ArgoCD on kind validates the release path. ArgoCD pulls from a registry and cannot see images side-loaded by `kind load`. Local ArgoCD with local builds would need `containerdConfigPatches` in `kind-config.yaml` plus a local registry container, neither of which exists. Out of scope.

## Prerequisites

In order:

1. Add `naira-plugin-depl-uses-litellm`, `naira-plugin-openmetadata` and `naira-plugin-mcp-servers` to `release-please.yml` (P3). `dev-publish.yml` already builds all three.
2. Values-driven plugin sidecars in the `naira` chart (P2). Everything else depends on the chart describing a working catalog.
3. Chart CI: `helm lint`, `ct lint`, `ct install`, OCI push.
4. Remove plaintext secrets from the charts' default path (P4). The `ExternalSecret` shape in `test-dependencies` is the model.

## Next Steps

| Phase | Outcome | Verified by |
|---|---|---|
| 0 | Three missing plugin images published by the release path | Ten images resolve from ghcr at one tag |
| 1 | `naira` chart reaches parity; chart CI green | Catalog pod's eight containers, plus ui and portal, healthy on kind from the chart alone; `ct install` passes |
| 2 | `test-dependencies` chart published | `helm install` yields Keycloak, LiteLLM, MLflow, OpenMetadata, PostgreSQL |
| 3 | Root Tiltfile; `deploy/dev/stacks/{core/infra/kubernetes,core/infra/keycloak,mlops}` and both Taskfiles deleted; chores moved to `mise.toml` | Developer edits a plugin and sees it live; no second manifest set; `task` removed from `mise.toml` |
| 4 | Release CI pushes OCI chart | `helm pull oci://…` works |
| 5 | ArgoCD installed; Naira synced from the OCI chart | Stack healthy; images pulled from ghcr, not side-loaded |
| 6 | OpenBao, ESO, MLflow, LiteLLM migrated off Flux; Makefile retired | `ExternalSecret`s still resolve; Flux uninstalled |
| 7 | Repositories renamed (done on GitHub) | No stale source URLs in cluster resources |
| 8 | Release CI pushes OCM component; gated on the OCM questions in section 3 | `ocm get component-version` lists the chart and ten image digests; `ocm transfer` then `helm install` from the target registry works |

Phases 0-4 and 8 are `naira` work; phases 5-7 are `test-dependencies` and cluster work.
