---
status: proposed
date: 2026-11-02
written-by: Marcel Frizler
---

# ADR-013: CI Owns Per-Run E2E vClusters

## Context

The E2E workflow in `naira-project/naira` creates a kind cluster on the GitHub runner for each run. The full scenario contains the catalog, plugins, Keycloak, LiteLLM and, as an option, OpenMetadata. This scenario uses almost all of the CPU, memory and disk of the runner. Thus, runs are slow and fail easily. We cannot add larger scenarios.

These ADRs apply to this decision:

- [ADR-007](./007-testbed-aws-rebuild.md) gives the sandbox EKS cluster with vCluster Platform.
- [ADR-010](./010-replace-cloudflare-warp-with-tailscale.md) makes Tailscale the only path for private access.
- [ADR-011](./011-manage-vcluster-lifecycle-through-gitops.md) makes the deployments repository the source of truth for vClusters. Changes go into this repository through pull requests. ArgoCD is the only writer that applies these changes to the platform.
- ADR-011 also creates one e2e vCluster for each pull request. It uses an ArgoCD `ApplicationSet` with the PullRequest generator.

The `naira` repository is public and accepts pull requests from forks. The infrastructure must run code only if these two conditions are true:

- The code comes from a branch of `naira-project/naira`.
- A reviewer approved the code.

The approval gate is the `e2e` GitHub deployment environment. GitHub stops each workflow run until a reviewer approves it. GitOps does not have a gate of this type. ArgoCD monitors the state of repositories and pull requests. ArgoCD does not monitor the approval of a run.

### Terms

| Term | Meaning |
| --- | --- |
| vCluster Platform, "the platform" | The service on the sandbox EKS cluster that creates and manages vClusters |
| `management.loft.sh` resources | The Kubernetes custom resources of the platform, such as `Project` and `VirtualClusterInstance`. A `VirtualClusterInstance` is one vCluster |
| Writer | The actor that creates, changes or deletes these resources |
| Platform access key | The credential that a platform user logs in with |
| Tailnet | The private Tailscale network. CI connects to the platform through this network |
| `e2e` environment | The GitHub deployment environment that has the reviewer approval gate and holds the platform access key |
| Job `e2e` | The GitHub Actions job that uses the `e2e` environment |
| Project `e2e` | The vCluster Platform project that holds the E2E vClusters |
| PR | A pull request |
| PR-controlled code | Code that the author of a PR can change, such as scripts, Go tests and their dependencies |
| Leaked vCluster | A vCluster that stays after its run ends |

## Decision drivers

- Only code from branches of `naira-project/naira` that a reviewer approved can go to the infrastructure. Code from forks can never go there.
- PR-controlled code cannot read a long-lived credential.
- Each run starts with a new test cluster. One run cannot see the state of a different run.
- Leaked vClusters cannot increase cost without a limit.
- CI can never connect to the EKS API.
- Local kind usage does not change.
- We build the smallest possible quantity of custom automation.

## Alternatives considered

### Option A: `ApplicationSet` with the PullRequest generator (ADR-011)

PRO

- ArgoCD stays the only writer.
- CI does not have a platform credential.

CON

- The generator creates the vCluster when a person opens the PR or adds a label. This occurs before a reviewer approves the run. The approval occurs in the GitHub Actions run, and the generator cannot see this run.
- The vCluster stays for the life of the PR, not for one run. Sequential runs share state.
- The generator polls for changes, so there is a delay between the PR event and a ready vCluster. CI must wait for a resource that it does not own.
- Teardown occurs when the PR closes or when a person removes the label. Teardown does not occur when the run ends.

### Option B: CI creates and deletes the vCluster with the `vcluster` CLI

PRO

- The vCluster exists only after approval and only for one job.
- Each run starts with a new state. This includes re-runs.
- Teardown is a part of the run (`if: always()`).

CON

- CI becomes a second writer of `management.loft.sh` resources, in addition to ArgoCD. This writer is limited to project `e2e`.
- CI has a platform access key during two steps.
- The deployments repository does not declare the vCluster. Thus, you cannot rebuild it from that repository, as you can with the other vClusters of ADR-011. A cluster that exists for one job does not need this.

### Option C: Keep kind on the runner

PRO

- This option needs no infrastructure and no credentials. It has no risk from forks.

CON

- This option keeps the problem that this ADR must solve: the resources of the runner limit the scenarios.

### Option D: One shared long-lived e2e vCluster, one namespace per run

PRO

- CI does not create and delete a vCluster for each run. Runs start faster.

CON

- Runs share cluster-scoped resources (CRDs, webhooks). One run can corrupt a different run.
- If a malicious change gets past review, it stays for all subsequent runs.

## Decision

**Chosen option**: Option B, CI creates and deletes the vCluster with the `vcluster` CLI.

CI creates the vCluster after a reviewer approves the run in the `e2e` environment. CI deletes the vCluster when the job ends. GitOps does not manage E2E vClusters.

This decision replaces the per-PR e2e row in the lifecycle classes table of ADR-011. Per-developer and dev/staging vClusters do not change. They stay as ADR-011 defines them.

### Workflow shape

- Job **`authorize`**: This job has no `environment`, no secrets and no `id-token`. It has read-only access to `contents` and `pull-requests`.
  - If a PR has the label `skip-e2e`, the E2E run does not occur. The job does not ask a reviewer.
  - All other PRs from this repository request the run.
  - The job makes sure that the PR number is a positive integer. It does this before all other uses of the number.
  - The job uses the GitHub API to make sure that the PR is open and that its head repository is this repository.
  - A `pull_request` run from a fork ends as skipped, with a notice. A `workflow_dispatch` run for a fork PR or a closed PR fails.
  - The only output of the job is the validated number. Subsequent jobs use only this output.
- Job **`e2e`**: This job starts after `authorize`.
  - The job uses `environment: e2e`. This environment has required reviewers and prevents self-review.
  - The timeout is 45 minutes.
  - The permissions are `contents: read`, `packages: write` and `id-token: write`.
- Concurrency is per PR, with cancel-in-progress. A new run for a PR cancels the current run for that PR.

### Step order

1. Check out the PR merge ref. Do not persist credentials.
2. Build all images and push them to GHCR. At this step, no tailnet credential or platform credential exists.
3. Join the tailnet as an ephemeral `tag:ci` node through GitHub OIDC. This step does not use a stored secret.
4. **Provision**:
   1. Log in with the platform access key.
   2. Create the vCluster in project `e2e` from `vcluster-e2e-template`.
   3. Get a kubeconfig that contains only a short-lived token.
   4. Log out.
   5. Delete the platform CLI config.
5. Deploy the scenario into the vCluster and run the assertions with that kubeconfig. To connect to Catalog and Keycloak, use a port-forward through the platform proxy. This proxy sends requests to the Kubernetes API of the vCluster.
6. **Teardown** (`if: always()`):
   1. Collect diagnostics.
   2. Log in again.
   3. Delete the vCluster.
   4. Log out.

The workflow gives the platform access key only through the `env:` of the provision and teardown steps. PR-controlled steps have only the short-lived kubeconfig.

If the pinned `vcluster` CLI cannot give a kubeconfig that operates without the platform login, stop the implementation. Do not keep the key on disk as a fallback.

### Identifiers

| Identifier | Value | Why |
| --- | --- | --- |
| Image tag | Commit hash of the checked-out PR merge ref | This hash identifies exactly the code under test. Do not use `github.sha`, because on a dispatch it is the SHA of main. Identical code uses identical tags |
| Image reference | `ghcr.io/naira-project/naira-e2e:<component>-<tag>` | All images use one public GHCR package. Thus, there is one visibility setting and pull secrets are not necessary |
| `ENV_ID` | `pr-<n>-<run_id>-<run_attempt>` | This value names the vCluster and the namespace in it. A re-run gets a new name, so a leftover from a failed delete cannot block it. The value must stay DNS-1123 |

### Cleanup

| Layer | Mechanism |
| --- | --- |
| Normal | The teardown step deletes the vCluster by `ENV_ID`. "Not found" is a success. All other failures make the job fail, so you can see leaked vClusters |
| Crashed runner | The inactivity auto-delete in `vcluster-e2e-template` deletes the vCluster |
| Too many concurrent runs | The `e2e` project quota limits the number of vClusters |

Do not build a sweeper.

### Access (amends ADR-011)

- The platform user `github-e2e` can write `management.loft.sh` resources only in project `e2e`. The access key of this user expires. Only an `e2e` environment secret holds this key.
- ADR-011 says that only the service account of ArgoCD writes these resources. This rule stays for all other projects.

### Tailnet identity

With `environment: e2e`, the default GitHub OIDC `sub` claim is `repo:naira-project/naira:environment:e2e`. This claim does not name the workflow file or the ref. Thus, each workflow in the repository that declares `environment: e2e` matches this claim.

- Customize the OIDC subject template of the repository to append `job_workflow_ref`. Then the subject names the workflow file.
- The Tailscale federated identity trusts `repo:naira-project/naira:environment:e2e:job_workflow_ref:naira-project/naira/.github/workflows/e2e.yml@*`.
- The ref stays a wildcard, because PR runs use `refs/pull/<n>/merge`. The workflow path stays pinned.

### Threats and controls

| Threat | Control |
| --- | --- |
| A fork PR runs on the infrastructure | `authorize` rejects fork heads before `e2e` requests the environment. Fork `pull_request` runs also get no secrets and no OIDC token. Never use `pull_request_target` |
| A maintainer starts a dispatch with a fork PR number or a closed PR number | `authorize` also examines the head repository and the state for dispatches |
| An attacker injects input through the PR number | `authorize` makes sure that the number is a positive integer. Subsequent jobs use only the validated output |
| A same-repo PR runs unreviewed code with infrastructure access | The `e2e` environment has a reviewer approval gate and prevents self-review |
| An attacker steals a Tailscale credential | This credential does not exist. The federated identity trusts only the subject of the E2E workflow in environment `e2e` (see [Tailnet identity](#tailnet-identity)) |
| A different workflow declares `environment: e2e` to join the tailnet | The subject pins `job_workflow_ref` to the E2E workflow file. Other workflow files get a subject that does not match |
| An attacker moves laterally on the tailnet | `tag:ci` can connect only to `tag:vcluster-platform` on tcp/443. ACL tests enforce this. CI can never connect to the EKS API |
| PR-controlled code reads the platform key | The key is only in the `env:` of the provision and teardown steps. The workflow deletes the platform CLI config before PR-controlled steps start |
| Code at build time steals credentials | The workflow builds the images before a tailnet credential or platform credential exists |
| A malicious change gets past review | PR-controlled steps have only a short-lived token for one vCluster |
| An attacker steals the platform access key | The key belongs to the dedicated user `github-e2e`. This user is a member of project `e2e` only, and the key expires |
| A workload escapes the vCluster | Pod Security `baseline` applies. NetworkPolicy blocks IMDS and in-VPC egress. The project quota applies |
| A run pulls the image of a different run from the cache of an EKS node | The image tag is the merge-ref commit, not the SHA of main. Thus, different code never has the same tag |
| Credentials go into public logs | Do not use `set -x` in steps that have secrets in `env:`. Do not echo secret values |
| Leaked vClusters increase cost | Teardown, the auto-delete of the template and the project quota |

### Accepted residual risks

- `pull_request` runs the workflow file from the PR. A same-repo PR can change the workflow or `e2e/`. After approval, these changes run with the `e2e` identity. Approvers must review those paths. The maximum damage area is project `e2e`.
- `GITHUB_TOKEN` has `packages: write` on `naira-e2e` for the full job.

The workflow header names the two risks and links to this ADR.

## Consequences

### Positive

- Runner resources do not limit the scenarios.
- No vCluster exists before a reviewer approves the run.
- Each run, and each re-run, gets a new test cluster.
- Code from forks never goes to the infrastructure. A fork cannot use a long-lived credential.
- E2E does not need an `ApplicationSet`, PR polling or an expiry bot.
- Local kind usage does not change.

### Negative

- vCluster lifecycle has two writers: ArgoCD for all other projects and CI for project `e2e`.
- The deployments repository does not declare E2E vClusters. They do not show in ArgoCD.
- All E2E runs fail if the platform is not available.
- You must rotate the platform access key before it expires.
- The GHCR package `naira-e2e` must be public. Pods get `ImagePullBackOff` until you make the package public after the first push.
- After a runner crash, cleanup depends on the auto-delete of the template.

## Verification

During implementation, make sure that these items are true for vCluster Platform 4.12 and vCluster 0.37:

- The `vcluster` CLI can give a short-lived kubeconfig that operates without the platform login.
- Port-forward operates through the platform proxy.
- Inactivity auto-delete operates in `vcluster-e2e-template` on the current licence. ADR-011 records that the free tier has no auto-delete.
- A cancelled run still deletes its vCluster.
- "Re-run failed jobs" creates a vCluster with a new name.
- No platform CLI config stays on the runner after the provision step.
- The Tailscale federated identity accepts the pinned subject with a ref wildcard. It rejects a token from a different workflow file in environment `e2e`.

## Related links

- [ADR-007: Testbed AWS rebuild](./007-testbed-aws-rebuild.md)
- [ADR-010: Replace Cloudflare WARP with Tailscale](./010-replace-cloudflare-warp-with-tailscale.md)
- [ADR-011: Manage vCluster lifecycle through GitOps](./011-manage-vcluster-lifecycle-through-gitops.md)
- [RFC-008: Kubernetes test environments for PR-based E2E testing](../rfc/rfc-008-Kubernetes-Test-Environments-for-PR-Based-E2E-Testing.md)
- `naira-project/naira`: `docs/e2e-vcluster.md`, `docs/e2e-vcluster-workflow-spec.md`
