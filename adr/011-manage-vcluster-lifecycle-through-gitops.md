---
status: accepted
date: 2026-10-06
written-by: Hossein Salahi
---

# ADR-011: Manage vCluster Lifecycle through GitOps

## Context

[ADR-007](./007-testbed-aws-rebuild.md) rebuilds the testbed on EKS with vCluster on the free tier and leaves lifecycle ownership to be defined during implementation. [RFC-009](../rfc/rfc-009-deployment-consolidation.md) makes ArgoCD the deployment mechanism and states that vClusters become ArgoCD destination clusters.

vCluster Platform manages virtual clusters through its own API and UI. Anything created there is stored in the platform's datastore, not in Git. The platform has no write-back to a repository.

The free tier has no automatic sleep and no auto-delete; manual sleep and wakeup through the platform API work. The team wants ephemeral environments stopped every day after **20:00**, while the dev/staging environment keeps running.

## Decision drivers

- One writer for environment state, with review and history.
- An environment can be rebuilt from Git alone.
- vClusters must be registered as ArgoCD destinations without custom code.
- Free tier only.
- A daily stop that cannot affect dev/staging.
- As little self-built automation as possible.

## Alternatives considered

### Option A: `VirtualClusterInstance` in Git, reconciled by ArgoCD, deployed by the platform (vCluster)

PRO

- The platform registers each vCluster in ArgoCD. Verified on the free tier.
- The ArgoCD destination URL is deterministic, so workload Applications need no lookup.
- No `ignoreDifferences` was needed; the Application stayed `Synced`.
- Deleting the Git entry removes the ArgoCD registration, Helm release, namespace and volume.
- The UI remains available for viewing and for kubeconfig access.
- ArgoCD integration in vCluster Platform, so the vCluster users can deploy their application via ArgoCD App.

CON

- The platform is a required component and needs a licence activated by hand on a fresh install.
- ArgoCD has no built-in health check for the resource.
- ArgoCD only reverts edits to fields that are declared in Git.

### Option B: vCluster Helm chart as a plain ArgoCD `Application`, no platform

PRO

- No licence and one component fewer.
- Direct control of the Helm release.

CON

- ArgoCD registration has to be built: a hook Job or a controller that turns the vCluster kubeconfig into an ArgoCD cluster secret.
- No UI, no project or access model.

### Option C: Platform UI or API as the writer

PRO

- Self-service works out of the box.

CON

- No review, no history, no declared state to detect drift against.
- Recovery depends on a backup of the platform datastore.
- No export to Git beyond a one-time manual `kubectl get -o yaml`.

### Option D: vCluster OSS with KRO or Crossplane for templating

Not evaluated. [product#66](https://github.com/naira-project/product/issues/66) made it conditional on alternative 1 failing.

### Option E: Terraform for vClusters

PRO

- Terraform already owns the EKS layer; no additional tool.

CON

- Per-PR create and destroy is the wrong flow for Terraform state and locking.
- Adds a second reconciler directly below the layer ArgoCD owns.

### Daily stop

| Option | State kept | Verdict |
| --- | --- | --- |
| Platform sleep per vCluster, Karpenter consolidates the emptied nodes | Yes | Chosen. Native, per instance, no node-level machinery |
| Dedicated tainted NodePool scaled to zero at 20:00 | Yes | Works (verified), but needs a Terraform-managed NodePool, `limits` patching, forced termination, and node selectors on every ephemeral vCluster |
| CronJob that scales vCluster control planes to zero and deletes synced pods | Yes | Rebuilds what platform sleep does |
| Remove and re-add the Git entries daily | No | Two bot commits a day; everything re-seeded |
| Automatic sleep by schedule or inactivity | Yes | Outside the free tier |

## Decision

Git is the only writer for vCluster lifecycle. The platform UI is used for viewing and access, and end-user-managed application deployment via ArgoCD on vClusters.

### Mechanism

- `Project` and `VirtualClusterInstance` resources live in the deployments repository and are reconciled by ArgoCD on the host cluster.
- The platform deploys the vCluster and registers it in ArgoCD through the `integrations.argoCD` block.
- Workload Applications target `https://<loftHost>/kubernetes/project/<project>/virtualcluster/<name>`.

### Lifecycle classes

| Class | Existence driven by | Mechanism | Prune |
| --- | --- | --- | --- |
| Per-PR e2e | Pull request state | `ApplicationSet`, PullRequest generator | on |
| Per-developer | Entry in a list file | `ApplicationSet`, List generator | on |
| Dev/staging | Git, long-lived | plain `Application`, version-pinned | off |

**ApplicationSet & PullRequest generator**: An ArgoCD ApplicationSet with the PullRequest generator polls the GitHub API for open PRs (optionally only those with a given label) and renders one Application per PR from a template, with the PR number and head SHA as template variables.

Creating and changing an environment is a pull request. Deleting the vCluster directly is reverted by `selfHeal`, so deletion also starts in Git:

| Class | Deletion |
| --- | --- |
| Per-PR, per-developer | Removing the entry is the deletion. Expiry is a bot that removes the entry. |
| Dev/staging | Two steps. A pull request removes the entry; the instance stays and the Application shows it as requiring pruning. An operator then runs a manual sync with prune for that resource. |

### Deletion order

- The platform fixes the order on its side
- ArgoCD registration removed first, then the Helm release, then the namespace.
- Workload Applications can therefore lose their destination before they are removed.
- In the spike a workload Application in that state deleted cleanly. If that does not hold for the PullRequest generator, the ephemeral workload `ApplicationSet` sets `preserveResourcesOnDeletion`; the vCluster deletion removes the workloads anyway.

### Access

- Only ArgoCD's service account writes `management.loft.sh` resources.
- Human platform accounts get view-only project roles plus access to the vClusters they use.
- Local admin credentials are break-glass only.
- ArgoCD reverts only fields declared in Git, so this rule is what makes Git the single writer, not `selfHeal`.

### Health

- ArgoCD gets a custom health check for `VirtualClusterInstance`: `Healthy` only when `status.phase` is `Ready` and the `ArgoCDIntegrationSynced` condition is true; `Progressing` while pending; `Degraded` on failure. Without it a blocked instance reports `Healthy`.

### Daily stop

- A scheduled job puts every vCluster except dev/staging to sleep at 20:00 through the platform API (`vcluster platform sleep vcluster <name> --project <project>`), selecting instances by a label set in Git.
- The platform removes the vCluster's pods and keeps its volume.
- Karpenter then consolidates the emptied nodes; no NodePool is touched and Git is not changed.
- Wakeup is either the mirror job at 07:00 (`vcluster platform wakeup`) or the first API request to the vCluster, which the platform treats as activity.
- Sleep changes runtime state, not the declared spec, in the same way a scale operation does; ArgoCD does not revert it.
- The job runs on the host cluster with a platform access key scoped to sleep and wakeup.
- Dev/staging is protected by not carrying the label; a second safeguard is an explicit deny-list in the job.

### Verification

EKS Auto Mode 1.36, ArgoCD 3.5, vCluster Platform 4.12 (free tier), vCluster 0.37. Sleep tests on the shared sandbox, the rest on a throwaway cluster.

| Check | Result |
| --- | --- |
| Git to ArgoCD to vCluster to workload | Works |
| ArgoCD registration on the free tier | Works |
| Edit to a Git-declared field outside Git | Reverted in under 20 s |
| Edit to an undeclared field outside Git | Not detected |
| Platform sleep on the free tier | Works; `Sleeping` within 3 s, all host pods removed, volume kept |
| Argo CD wakes a sleeping vCluster | No. 15 min asleep with an Application targeting it, `last-activity` unchanged |
| Platform wakeup from the CLI | Works; `Ready` after 31 s, workloads back, etcd state intact |
| Fallback: tainted NodePool scaled to zero | Works; node gone in about 40 s, nothing rescheduled elsewhere, dev/staging pods untouched, restart in 63 s with the same volume |
| Delete by Git removal | Pruned in 21 s; platform cleaned up in about 20 s |
| Workload Application whose destination is already gone | Deleted cleanly within 20 s (ArgoCD 3.5.3, one run) |
| Leftovers after teardown | No volumes, no load balancers |

## Consequences

### Positive

- Environment state is reviewable and rebuildable from Git.
- ArgoCD registration is configuration, not code.
- Self-service ([product#65](https://github.com/naira-project/product/issues/65)) becomes a pull request against a list file.
- The daily stop is a platform API call per instance; no node pools, taints or forced termination.
- Dev/staging is excluded by selection; nothing shared with the stopped instances is touched.

### Negative

- The team owns two scheduled jobs and the expiry bot.
- Out-of-Git edits to undeclared fields are not detected; the access rule is the only control.
- Deleting dev/staging needs an operator step after the pull request.
- Sleep deletes running pods without draining; in-flight work in an ephemeral vCluster at 20:00 is lost.
- Argo CD keeps reporting the last known state (`Synced/Healthy`) for a sleeping vCluster; it does not notice the pods are gone.
- The job needs a platform access key; the platform is on the reconcile path for the stop as well as for provisioning.
- ArgoCD shows connection errors for stopped vClusters overnight unless a sync window is set.
- The platform registers clusters in ArgoCD with TLS verification off.
- Host contract additions: a default `ebs.csi.eks.amazonaws.com` StorageClass, `config.loftHost` resolvable in-cluster.
- `sync.toHost.pods.useSecretsForSATokens` must be enabled; otherwise tenant service-account tokens sit in host pod annotations.

## Related links

- [ADR-007: Testbed AWS rebuild](./007-testbed-aws-rebuild.md)
- [RFC-009: One deployment path for Naira](../rfc/rfc-009-deployment-consolidation.md)
- <https://github.com/naira-project/product/issues/66>
- <https://github.com/naira-project/product/issues/55>
