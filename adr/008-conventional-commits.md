---
status: accepted
date: 2026-08-24
written-by: Marcel Frizler
---

# Enforcing Conventional Commits

This is an Architecture Decision Record for the Naira Platform. Fill in each section, set the status, and link related ADRs or Github issue tickets.

## Context

Naira uses Release Please to automate versioning, release pull requests, and changelog generation. Release Please derives release intent from commit messages on the target branch, particularly from Conventional Commit types such as `feat`, `fix`, and breaking-change markers.

The repository currently supports both squash merges and normal merge commits. Supporting both strategies requires different commit-message validation:

- With a squash merge, the pull request title normally becomes the subject of the resulting commit on the target branch. The pull request title must therefore follow the Conventional Commits specification.
- With a normal merge, the individual feature-branch commits remain in the target-branch history. Those commits must therefore also follow the Conventional Commits specification.

This results in Conventional Commit validation for both pull request titles and all individual commits. For developers who use squash merges, validating intermediate feature-branch commits creates additional effort even though those commits do not remain in the target-branch history.

The team is reconsidering whether supporting multiple merge strategies provides sufficient value to justify this additional validation and complexity. A consistent merge strategy could simplify enforcement while continuing to provide Release Please with reliable release information.

We need an enforceable policy that produces valid Conventional Commits on the target branch, supports Release Please, and minimizes unnecessary restrictions on development within feature branches.

## Decision drivers

- Reliable semantic-version calculation and changelog generation by Release Please.
- A consistent and readable target-branch history.
- One authoritative description of the change represented by a pull request.
- Simple rules that are easy for developers to understand and follow.
- Freedom to use informal or work-in-progress commit messages on feature branches.
- Server-side enforcement that cannot be bypassed locally.
- Low maintenance overhead and compatibility with repository automation.
- Clear remediation when a pull request title is invalid.

## Alternatives considered

### 1. Allow only squash merges and enforce Conventional Commits for pull request titles

**Pros**

- Ensures every commit added to the target branch follows the Conventional Commits specification.
- Provides Release Please with reliable release intent.
- Allows developers to use informal work-in-progress commits on feature branches.
- Avoids rebasing or rewriting feature-branch history solely to correct commit messages.
- Establishes the pull request title as the single authoritative description of the merged change.
- Requires less tooling and configuration than validating every individual commit.

**Cons**

- CON Normal merges and rebase merges cannot be used.
- CON Pull request titles may need to be updated before merging.
- CON Large pull requests containing unrelated changes may be difficult to describe with a single Conventional Commit type.

### 2. Enforce Conventional Commits for pull request titles and individual commits

**Pros**

- Produces structured commit messages on feature branches as well as the target branch.
- Would support normal merges if they were enabled again.

**Cons**

- Adds restrictions without improving the resulting target-branch history when all pull requests are squash-merged.
- Developers may need to rewrite and force-push feature-branch history to fix invalid commit messages.
- Requires additional CI validation and potentially local commit hooks.
- Creates unnecessary friction for work-in-progress commits that do not reach the target branch.

### 3. Enforce only individual commit messages

**Pros**

- Produces consistent branch and target-branch history for normal merges.
- Gives Release Please structured input from individual commits.

**Cons**

- Does not ensure that a squash merge produces a valid final commit message.
- A non-conforming pull request title can still break or reduce the quality of release automation.

### 4. Allow multiple merge strategies

**Pros**

- Gives contributors flexibility in choosing how pull requests are merged.

**Cons**

- Requires different validation rules for different merge strategies.
- Normal merges retain all feature-branch commits and therefore require every commit to conform.
- Produces a less consistent target-branch history.
- Increases tooling, documentation, and maintenance complexity.

### 5. Do not enforce a commit convention

**Pros**

- No additional tooling or developer constraints.

**Cons**

- Release Please cannot reliably infer release types or generate consistent changelogs.
- The target-branch history may contain ambiguous or inconsistent commit subjects.
- Release corrections become more manual and error-prone.

## Decision

The repository will allow **squash merges only**.

Normal merge commits and rebase merges will be disabled in the repository settings. Each merged pull request will create exactly one commit on the target branch.

The subject of the squash commit must be derived from the pull request title. Repository settings must therefore be configured so that the pull request title is used as the squash commit subject.

Pull request titles must follow the Conventional Commits specification. This rule will be validated by an automated GitHub status check, and the check will be required by the target branch’s ruleset or branch protection configuration.

A pull request with an invalid title cannot be merged. The validation should provide a clear error message and examples of valid titles.

Individual commits on feature branches are not required to follow the Conventional Commits specification. Local commit hooks and CI checks for individual feature-branch commit messages are therefore not required.

The effective flow is:

1. Developers may create work-in-progress commits using any meaningful commit message.
2. The pull request title describes the complete change in Conventional Commit format.
3. An automated required check validates the pull request title.
4. GitHub squash-merges the pull request.
5. The pull request title becomes the subject of the resulting target-branch commit.
6. Release Please evaluate that commit to determine release intent.

Pull requests should contain one coherent change that can be accurately represented by a single Conventional Commit title. Unrelated changes should be split into separate pull requests.

## Consequences

### Positive

- Every commit added to the target branch has a Conventional Commit subject.
- Release Please receive consistent and reliable release information.
- The target-branch history contains one concise commit per pull request.
- Developers can freely create _fixup_ and _work-in-progress_ commits on feature branches.
- Contributors do not need to rewrite feature-branch history solely to satisfy commit-message validation.
- Validation and maintenance are simplified because only the pull request title must be checked.
- The pull request title, target-branch commit, changelog entry, and release intent remain aligned.

### Negative

- Normal merges and rebase merges are no longer available.
- The quality of the final commit message depends on the quality of the pull request title.
- Contributors may need to correct the pull request title before merging.
- A pull request containing multiple unrelated changes cannot always be represented accurately by a single Conventional Commit type.
- Repository settings and branch rules must prevent alternative merge strategies and require the title-validation check.

## Related links

- [Conventional Commits specification](https://www.conventionalcommits.org/)
- [Release Please documentation](https://github.com/googleapis/release-please)
- [GitHub documentation: required status checks](https://docs.github.com/en/repositories/configuring-branches-and-merges-in-your-repository/managing-protected-branches/about-protected-branches)
- [GitHub documentation: merge queue checks](https://docs.github.com/en/repositories/configuring-branches-and-merges-in-your-repository/configuring-pull-request-merges/managing-a-merge-queue)
