---
status: proposed
date: 2026-11-02
written-by: Marcel Frizler
---

# ADR-013: CI Owns Per-Run E2E vClusters

## Context

The E2E workflow in `naira-project/naira` builds a kind cluster on the GitHub runner on every run. The full scenario (catalog, plugins, Keycloak, LiteLLM, optionally OpenMetadata) barely fits the runner's CPU, memory and disk, so runs are slow and fragile, and heavier scenarios can't be added.

[ADR-007](./007-testbed-aws-rebuild.md) provides the sandbox EKS cluster with vCluster Platform. [ADR-010](./010-replace-cloudflare-warp-with-tailscale.md) makes Tailscale the only private access path. [ADR-011](./011-manage-vcluster-lifecycle-through-gitops.md) makes Git the only writer for vCluster lifecycle and assigns per-PR e2e vClusters to an `ApplicationSet` with the PullRequest generator.

The `naira` repo is public and accepts fork PRs. The infrastructure must never run code that hasn't come from a branch of `naira-project/naira` and been approved by a reviewer. The approval gate is the `e2e` GitHub deployment environment. It exists per workflow run. Git has no equivalent.

## Decision drivers

- Only reviewer-approved code from branches of `naira-project/naira` reaches the infrastructure. Fork code never does.
- PR-controlled code (scripts, Go tests, their dependencies) can't read a long-lived credential.
- Each run starts from a fresh test cluster. Runs can't see each other's state.
- Leaked vClusters can't accumulate cost.
- The EKS API is never reachable from CI.
- Local kind usage keeps working unchanged.
- As little self-built automation as possible.

## Alternatives considered

### Option A: `ApplicationSet` with the PullRequest generator (ADR-011)

PRO

- Keeps Git as the single writer.
- No platform credential in CI.

CON

- The generator creates the vCluster when the PR is opened or labelled, before anyone approves. The approval gate lives in GitHub Actions, which the generator doesn't see.
- The vCluster lives as long as the PR, not the run. Consecutive runs share state.
- Polling delay between the PR event and a ready vCluster. CI has to wait on a resource it doesn't own.
- Teardown depends on the PR closing or the label being removed, not on the run ending.

### Option B: CI creates and deletes the vCluster with the `vcluster` CLI

PRO

- The vCluster exists only after approval and only for one job.
- Fresh state per run, including re-runs.
- Teardown is part of the run (`if: always()`).

CON

- A second writer for `management.loft.sh` resources, limited to project `e2e`.
- CI holds a platform access key during two steps.
- Not reproducible from Git. For a throwaway cluster this doesn't matter.

### Option C: Keep kind on the runner

PRO

- No infrastructure, no credentials, no fork risk.

CON

- The problem this ADR exists to solve: runner resources cap the scenarios.

### Option D: One shared long-lived e2e vCluster, one namespace per run

PRO

- No create/delete per run. Faster start.

CON

- Runs share cluster-scoped resources (CRDs, webhooks) and can corrupt each other.
- A malicious change that slips past review persists for every later run.

## Decision

Per-run e2e vClusters are created and deleted by CI with the `vcluster` CLI. They are not GitOps-managed. The `e2e` deployment environment approval is the gate. The vCluster lives for one job.

This replaces ADR-011's per-PR e2e lifecycle class. The per-developer and dev/staging classes stay as ADR-011 defines them.

### Workflow shape

- Job **`authorize`**: no `environment`, no secrets, no `id-token`. Read-only `contents` and `pull-requests`.
  - Unlabelled PRs skip without asking a reviewer (`run-e2e` label opt-in).
  - Validates the PR number as a positive integer before using it anywhere.
  - Checks through the GitHub API that the PR is open and its head repo is this repo.
  - A fork `pull_request` ends as skipped with a notice. A fork or closed-PR `workflow_dispatch` fails.
  - Outputs only the validated number. Downstream jobs use only that output.
- Job **`e2e`**: needs `authorize`. `environment: e2e` with required reviewers and self-review prevented. 45-minute timeout. Permissions `contents: read`, `packages: write`, `id-token: write`.
- Per-PR concurrency with cancel-in-progress.

### Step order

1. Check out the PR merge ref without persisting credentials.
2. Build and push all images to GHCR. No tailnet or platform credential exists yet.
3. Join the tailnet as an ephemeral `tag:ci` node through GitHub OIDC. No stored secret.
4. **Provision**: log in with the platform access key, create the vCluster in project `e2e` from `vcluster-e2e-template`, obtain a kubeconfig with only a short-lived token, log out, delete the platform CLI config.
5. Deploy the E2E environment and run the assertions with that kubeconfig. Catalog and Keycloak are reached by port-forward through the platform proxy.
6. **Teardown** (`if: always()`): collect diagnostics, log in again, delete the vCluster, log out.

The platform access key is passed only through the `env:` of the provision and teardown steps. PR-controlled steps hold only the short-lived kubeconfig.

If the pinned `vcluster` CLI can't produce a kubeconfig that works without the platform login, the implementation stops. Keeping the key on disk is not a fallback.

### Identifiers

| Identifier | Value | Why |
| --- | --- | --- |
| Image tag | Commit hash of the checked-out PR merge ref | Identifies exactly the code under test. Not `github.sha`, which is main's SHA on dispatch. Identical code reuses identical tags |
| Image reference | `ghcr.io/naira-project/naira-e2e:<component>-<tag>` | One public GHCR package, one visibility setting, no pull secrets |
| `ENV_ID` | `pr-<n>-<run_id>-<run_attempt>` | Names the vCluster and the namespace inside it. A re-run gets a new name, so a leftover from a failed delete doesn't block it. Must stay DNS-1123 |

### Cleanup

| Layer | Mechanism |
| --- | --- |
| Normal | Teardown step deletes the vCluster by `ENV_ID`. "Not found" is success. Any other failure fails the job, so leaks are visible |
| Crashed runner | Inactivity auto-delete in `vcluster-e2e-template` |
| Burst | `e2e` project quota on the number of vClusters |

No sweeper is built.

### Access (amends ADR-011)

- The platform user `github-e2e` may write `management.loft.sh` resources in project `e2e` only. Its access key expires and is stored only as an `e2e` environment secret.
- ADR-011's rule that only ArgoCD's service account writes these resources holds for every other project.

### Threats and controls

| Threat | Control |
| --- | --- |
| Fork PR runs on the infrastructure | `authorize` rejects fork heads before `e2e` requests the environment. Fork `pull_request` runs get neither secrets nor an OIDC token anyway. `pull_request_target` is never used |
| Maintainer dispatches with a fork or closed PR number | `authorize` checks head repo and state for dispatches too |
| Input injection through the PR number | Validated as a positive integer in `authorize`. Downstream uses only the validated output |
| Same-repo PR runs unreviewed code with infra access | `e2e` environment requires reviewer approval, self-review prevented |
| Stolen Tailscale credential | None exists. Federated identity trusts only `repo:naira-project/naira:environment:e2e` |
| Lateral movement on the tailnet | `tag:ci` reaches only `tag:vcluster-platform` on tcp/443, enforced by ACL tests. The EKS API is never reachable |
| PR-controlled code reads the platform key | Key lives only in the provision and teardown steps' `env:`. Platform CLI config is deleted before PR-controlled steps run |
| Build-time code steals credentials | Images are built before any tailnet or platform credential exists |
| Malicious change slips past review | PR-controlled steps hold a short-lived token scoped to one vCluster |
| Stolen platform access key | Dedicated user `github-e2e`, member of project `e2e` only, key expires |
| Workload escapes the vCluster | Pod Security `baseline`. NetworkPolicy blocks IMDS and in-VPC egress. Project quota |
| Two runs share images on cached EKS nodes | Image tag is the merge-ref commit, not main's SHA |
| Credentials in public logs | No `set -x` in steps with secrets in `env:`. Nothing secret is echoed |
| Leaked vClusters (cost) | Teardown, template auto-delete, project quota |

### Accepted residual risks

- `pull_request` runs the workflow file from the PR. A same-repo PR that edits the workflow or `e2e/` runs those edits with the `e2e` identity once approved. Approvers must review those paths. The blast radius is project `e2e`.
- `GITHUB_TOKEN` has `packages: write` on `naira-e2e` for the whole job.

The workflow header names both and links this ADR.

## Consequences

### Positive

- Scenarios are no longer limited by runner resources.
- No vCluster exists before a reviewer approves the run.
- Each run, including each re-run, gets a fresh test cluster.
- Fork code never reaches the infrastructure. No long-lived credential is usable from a fork.
- No ArgoCD, no PR-polling, no expiry bot for e2e.
- Local kind usage is unchanged.

### Negative

- Two writers for vCluster lifecycle: Git for everything else, CI for project `e2e`.
- E2E vClusters are not visible in Git or ArgoCD.
- The platform is on the critical path of every E2E run.
- The platform access key must be rotated before it expires.
- The GHCR package `naira-e2e` must be public. Until it is made public after the first push, pods hit `ImagePullBackOff`.
- Cleanup after a crashed runner depends on the template's auto-delete.

## Verification

To be checked during implementation, against vCluster Platform 4.12 / vCluster 0.37:

- The `vcluster` CLI can produce a short-lived kubeconfig that works without the platform login.
- Port-forward works through the platform proxy.
- Inactivity auto-delete works in `vcluster-e2e-template` on the current licence. ADR-011 records that the free tier has no auto-delete.
- A cancelled run still deletes its vCluster.
- "Re-run failed jobs" creates a vCluster with a new name.
- No platform CLI config remains on the runner after the provision step.

## Related links

- [ADR-007: Testbed AWS rebuild](./007-testbed-aws-rebuild.md)
- [ADR-010: Replace Cloudflare WARP with Tailscale](./010-replace-cloudflare-warp-with-tailscale.md)
- [ADR-011: Manage vCluster lifecycle through GitOps](./011-manage-vcluster-lifecycle-through-gitops.md)
- [RFC-008: Kubernetes test environments for PR-based E2E testing](../rfc/rfc-008-Kubernetes-Test-Environments-for-PR-Based-E2E-Testing.md)
- `naira-project/naira`: `docs/e2e-vcluster.md`, `docs/e2e-vcluster-workflow-spec.md`
