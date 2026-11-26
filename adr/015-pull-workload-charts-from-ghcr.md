---
status: proposed
date: 2026-11-24
---

# Pull Workload Charts directly from GHCR

## Context

[RFC-009](../rfc/rfc-009-deployment-consolidation.md) §5 deploys the Naira stack through Argo CD from published OCI charts, on both the EKS sandbox and Kind.

The sandbox Argo CD installs charts from two kinds of source:

- **Platform Charts**: third-party infrastructure charts shared across environments, such as Argo CD, External Secrets, OpenBao, the Tailscale operator and vCluster Platform. They are mirrored into a private ECR repository from a pinned list (`gitops/charts.txt` in `test-infrastructure`), so bootstrapping the cluster does not depend on external HTTP chart repositories.
- **Workload Charts**: charts the Naira project publishes to deploy its own stack, `naira` and `test-dependencies`. Their releases are public OCI artifacts on `ghcr.io/naira-project/charts`, which the project already owns.

Until now, the Workload Charts were deployed from commit-pinned Git paths, because no release contained the needed content. Releases now exist (`naira` 0.1.2, `test-dependencies` 0.1.0), so we have to choose where Argo CD pulls them from.

## Decision drivers

- Deploy the same chart artifacts on the sandbox and on Kind, which has no AWS access.
- Make a chart upgrade a single, reviewable version change.
- Avoid per-chart infrastructure such as ECR repositories and mirror runs.
- Keep cluster bootstrap independent of registries we do not control.

## Decision

We will pull Workload Charts directly from `ghcr.io/naira-project/charts` as OCI Helm charts, pinned by version in each Argo CD Application's `targetRevision`. Both the sandbox and Kind will use this source.

Platform Charts stay on the ECR mirror. `gitops/charts.txt` and the Helm mirror workflow cover Platform Charts only.

Charts are never sourced from Git paths in other repositories. A Git-path source is allowed only for manifests and for Local Charts in `test-infrastructure` itself, which template cluster manifests from Git data and are never published. CI enforces this.

## Alternatives considered

### Mirror Workload Charts into ECR

**Pros**

- One chart registry for every sandbox Application.
- The sandbox would not depend on GHCR availability.

**Cons**

- Kind cannot use ECR, so the sandbox and Kind would pull from different sources.
- Every Workload Chart needs an ECR repository provisioned in Terraform.
- Every release becomes a multi-file change, gated on a mirror workflow run.

### Keep deploying from commit-pinned Git paths

**Pros**

- No release is needed to deploy unreleased chart changes.

**Cons**

- What is deployed is not a versioned, published artifact.
- Argo CD has to build chart dependencies from several HTTP repositories at sync time.
- CI has to clone each chart repository to render it.

### Pull Workload Charts directly from GHCR

**Pros**

- The sandbox and Kind deploy identical artifacts from the same source.
- An upgrade is a one-line `targetRevision` change.
- No new infrastructure: the registry is already owned by the Naira project, and the charts are public, so no credentials are needed.

**Cons**

- Syncing Workload Charts on the sandbox depends on GHCR availability.
- The sandbox uses two chart registries instead of one.

## Consequences

- Sandbox Argo CD registers `ghcr.io/naira-project/charts` as a public OCI Helm repository alongside the ECR one.
- Workload Chart upgrades are one-line changes. Unreleased chart changes have to be released before they can be deployed.
- A GHCR outage blocks Workload Chart syncs on the sandbox; running workloads are not affected.
- `gitops/charts.txt` and the Helm mirror workflow stay scoped to Platform Charts.
- CI rejects any Application that sources a chart from a Git path outside `test-infrastructure`.

## Related links

- [RFC-009: One Deployment Path for Naira](../rfc/rfc-009-deployment-consolidation.md)
- [ADR-007: Rebuild the Naira Testbed on AWS EKS](./007-testbed-aws-rebuild.md)
- [naira-project/test-infrastructure#65](https://github.com/naira-project/test-infrastructure/pull/65): implementation
- [Argo CD: Helm OCI repositories](https://argo-cd.readthedocs.io/en/stable/user-guide/helm/#oci-helm-repositories)
