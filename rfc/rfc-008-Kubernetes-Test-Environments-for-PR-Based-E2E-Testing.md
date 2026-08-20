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

Every pull request that touches Naira should be able to run a full integration and end-to-end test suite against a real, isolated copy of the platform — not a shared environment that other PRs might be mutating at the same time.

We propose giving each PR its own Kubernetes namespace, with the entire application and its dependencies deployed fresh into it. Tests run against that isolated copy, and the whole thing is deleted afterward. No shared state to reason about, no cleanup protocol to get wrong.

This is deliberately the simplest version of the idea. It costs more time and compute per PR than sharing dependencies would, but it avoids an entire category of intermittent, hard-to-diagnose test failures. We'd rather ship something slow and trustworthy than something fast and occasionally wrong.

## The Problem

Naira's E2E tests need several services running together: the core catalog API, its plugins, a model registry, an AI gateway, a metadata catalog, and an inference backend. Running all of that directly on a GitHub Actions runner strains its CPU, memory, and disk, and different PRs running at the same time can't share a runner without stepping on each other.

The usual fix is a shared test cluster. But a shared cluster only helps if concurrent PRs can't see or corrupt each other's data. That guarantee turns out to be harder than it sounds — some of our services aren't currently written to be aware of the difference between "the whole cluster" and "the resources I care about." A cost-saving instinct — share what's shared everywhere else, like a database or a cache — actively works against test reliability here, because it reopens exactly the isolation problem we're trying to close.

## Goals

The first iteration must:

- Create an isolated Kubernetes namespace for each PR or test execution.
- Deploy the application and its dependencies per PR by default.
- Work first against a throwaway cluster created inside the CI job, with a design that also supports a persistent remote shared cluster once the prerequisite isolation fixes in "What Still Needs Fixing" land — remote-cluster support is the target, but not considered in first iteration
- Inject environment-specific configuration.
- Initialize a clean and deterministic test dataset.
- Wait for application and dependency readiness.
- Provide the environment information required by the E2E test suite.
- Collect logs and diagnostic information when tests fail.
- Delete the environment after testing, leaving nothing behind.
- Support safe execution of multiple PR environments on the same cluster.


## The Approach

Give each PR its own namespace, and deploy everything that PR's tests need into it: the catalog, its plugins, the AI gateway, the model registry, the metadata service, and the inference backend. Nothing is shared with any other PR's environment.

Because every dependency starts empty and privately owned, there's no need to invent naming schemes to keep one PR's test data from colliding with another's, no cleanup jobs that might silently fail, and no waiting in line behind another PR for a shared resource. Deleting the namespace when the tests finish removes everything, with nothing left over to garbage-collect later.

This deliberately reuses what already exists rather than building a parallel implementation. The Kubernetes definitions mirror the ones developers already run locally, and the step that seeds a starting dataset runs the same seed scripts developers already use on their own machines, unchanged — just packaged to run as a one-off Kubernetes job instead of from a local checkout. There's one definition of "what a working Naira deployment looks like," not two that can quietly drift apart.

The tradeoff is time and machine cost: standing up a full copy of the platform for a single PR takes several minutes and a meaningful chunk of CPU and memory, dominated by one particularly heavy dependency. A lighter alternative — sharing the expensive pieces across PRs and just faking isolation with careful naming — was considered and is documented as a fallback below, under "Sharing the heavy dependencies across all PRs" in **Alternatives We Considered** — but it was not chosen as the starting point.

## Where This Runs

In the near term, environments run on a throwaway Kubernetes cluster created fresh inside the GitHub Actions job itself. That's simple to reason about and needs no external infrastructure, but it's tight on the runner's available CPU and disk once the full platform is deployed — workable, but with little headroom.

The target is a small, persistent, shared Kubernetes cluster that CI authenticates into, rather than building one from scratch every run. That removes the resource ceiling and lets container images stay cached between runs instead of being re-downloaded every time. Moving to a shared cluster does mean some of our plugins need small fixes first, because a few of them currently look at cluster-wide state instead of just their own namespace — see "What Still Needs Fixing" below.

A further step — giving each PR not just its own namespace but its own miniature Kubernetes control plane, using a tool called **vCluster** — is the **planned next iteration**, once the shared-cluster migration above is running in CI. It would close a couple of isolation gaps that namespaces alone can't, at a modest extra cost per PR. It's a deliberate follow-on, not part of this proposal.

## What Still Needs Fixing

Isolating each PR's namespace protects the data it *writes*. It doesn't yet protect what its services *read*. A few of Naira's plugins currently query their backing services or the cluster itself without limiting the query to their own environment — so on a cluster shared by multiple PRs, one PR's catalog could see another PR's data.

This is a real, known gap, not a hidden one. Two small, contained code changes close it, and they're the only code changes this proposal requires. Until they land, running on a genuinely shared cluster isn't safe — but running on a throwaway per-PR cluster (today's approach) sidesteps the problem entirely, since there's nothing else on the cluster to see.

## Decision

Proceed with: one namespace per PR, the full platform deployed into it by default, running on a throwaway cluster today and migrating to a shared cluster once the two isolation fixes land. Treat sharing dependencies across PRs — see "Sharing the heavy dependencies across all PRs" below — as a fallback to build only if the full-deployment approach proves too slow in practice, not as the starting design. Once the shared cluster is running in CI, giving each PR its own miniature control plane via vCluster (see "Where This Runs") is the next planned step, not a maybe.

## Environment Lifecycle

When a PR is opened or updated, CI creates a fresh namespace, deploys the full platform into it, and waits until every service reports healthy before seeding it with a starting dataset. Tests then run against that environment. Whether the tests pass or fail, the namespace and everything in it is deleted afterward — there's no separate cleanup step to remember or get wrong.


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

## Alternatives We Considered

- **A custom controller or operator from day one** — too much upfront complexity before we've validated that the basic workflow works.
- **Existing platform-orchestration tools** (KubeVela, kro, Argo CD) — all solve problems (multi-cluster delivery, GitOps, resource composition) that we don't have yet. Worth revisiting once the manual approach shows real strain.
- **Running everything directly on the GitHub runner with no Kubernetes cluster at all** — doesn't scale past a small environment; runner limits bite hard.
- **Sharing the heavy dependencies across all PRs** — cheaper and faster to set up, but it trades away the guarantee that a red test result means something, because it reopens the same read-isolation gap described in "What Still Needs Fixing." We expect to revisit this later as a cost-reduction step — for example, running one shared copy of the AI gateway and the metadata catalog that every PR's environment points to — but only once per-PR cost is a proven problem, not as part of this proposal.

## Success Looks Like

- Any PR can get a fully isolated test environment on demand.
- The environment starts with a clean, deterministic dataset every time.
- Tests can assert on exact results, not just "does this row exist somewhere" — a direct consequence of full isolation.
- Running the whole suite dozens of times back to back produces zero cross-contamination between runs.
- Failed environments leave behind logs and diagnostics, then clean up completely.
- The same scripts work whether the environment lives on a throwaway runner cluster or a shared remote one.