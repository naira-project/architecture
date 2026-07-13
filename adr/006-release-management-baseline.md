---
status: accepted
date: 2026-07-13
written-by: Hossein Salahi
---

# Naira Release Management Baseline

## Context and Problem Statement

Naira is still early, but the work is already split across several repositories. These repositories cover product code, CI/CD workflows,
experiments, documentation, governance, and spike work. The documented repositories are `naira`, `naira-github-workflows`, `component-testbed`,
`architecture`, `community`, `.github`, `naira-project.github.io`, `repository-template`, and `spikes`. 
Without a clear release process, the project may create tags without release artifacts, apply hotfixes outside main,
treat plugins as stable before the API is stable, and publish documentation for workflows that do not exist.

The release process must first be repeatable and easy to check. It must stay simple enough for a small team to run,
but strict enough to prevent branch drift and invalid releases.

## Decision Drivers

- Release boundaries must follow how the software is built, released, and supported.
- A release version only matters if it points to immutable artifacts.
- The team needs a release process that does not depend on tribal knowledge.
- The project is `pre-1.0` and should avoid premature per-package versioning.
- Patch and hotfix handling must prevent divergence between released branches and main.
- Official plugins need an explicit compatibility boundary because the plugin API is not yet stable.
- Documentation must match released product behavior, not unreleased main changes.
- Automation should enforce policy, not compensate for missing policy.

## Considered Options
### Option A - One product repository, trunk-based development, release branches, tool-neutral release automation, and a separate plugin repository (Chosen option)

Pros:
- `naira` is the only repository that produces the core product release.
- Development targets main through pull requests.
- `release/vX.Y` branches exist only for patch and hotfix releases of already released minor versions.
- Official plugins move to a separate `naira-plugins` repository and are checked against supported Naira minor versions.
- The release process can use GitHub Actions automation, bot-driven workflows, or another proven approach.
- `component-testbed` remains a validation environment, not a product assembly or release repository.

Cons:
- Core product parts do not get separate version numbers.
- The naira repository stays broader in scope than a smaller split-repository model.
- The automation choice is still open, so some release steps may stay manual until the workflow is proven.

### Option B - Split release ownership across multiple repositories with GitFlow-style branching and broader independent versioning

Pros
- Separate product areas such as UI, catalog, plugins, and platform integration would evolve with independent release numbers and potentially separate repositories.
- A develop branch or GitFlow-style stabilization flow would be introduced.
- Teams could evolve some parts independently if those parts later become truly separate products.

Cons:
- It adds process overhead before the team has stable release boundaries, support commitments, or enough maintainers.
- It increases the risk of branch drift and hidden divergence.
- It encourages independent versioning before the components are independently deployable and supportable.

This option was rejected because it adds coordination cost and branch risk without solving a current problem.

## Decision Outcome

Naira will use this release baseline:

- `naira` is the single product release repository and bounded monorepo for the core deployable product.
- The project uses pull-request-based trunk-based development on main.
- `release/vX.Y` branches are created only after a minor release exists and are used only for patch and hotfix releases for that released minor version.
- Every valid release must produce immutable release artifacts, at minimum: catalog image, UI image, Git tag, GitHub Release notes, and deployment manifests pointing to immutable image versions.
- A Git tag without the required artifacts and release validation is not a valid release.
- Official plugins use a separate lifecycle in naira-plugins and must declare supported Naira version bounds.
- `component-testbed` is the release validation environment and not part of the product release boundary.
- Release automation is required, but the specific tool is not fixed yet.
- The selected automation must support release metadata, tagging, and artifact publication, or those steps must remain explicit and manual until proven.
- Published documentation must match released product versions and must not present unreleased main behavior as stable.
- Rollback must be possible to the previous known-good image set. If a release changes state or schema, the rollback steps must also be documented.

---

## Pros and Cons of the Options

Pros:
- The release boundary becomes clear. One core product release, one main integration branch, and limited maintenance branches.
- Invalid releases are easier to detect because tags, artifacts, validation, release records, and rollback steps are linked.
- Plugin instability is limited by a separate repository and explicit compatibility data.

Cons:
- This baseline does not allow separate versioning for subcomponents yet.
- The release automation is not fixed yet. That leaves room for uncertainty until the workflow is tested and enforced.
- The baseline depends on clear ownership for validation, release records, backports, and rollback. If ownership stays implicit, the process will fail.

---

## Technical Outlook / Future Direction

To evolve the release management beyond this baseline, we will focus on the following areas:

- **Automated Release Orchestration**: Evaluate and adopt tools like *Release Please* or *Semantic Release* to automate version bumping, changelog generation, and GitHub Release creation based on Conventional Commits.
- **Multi-Repository Compatibility**: Implement automated CI pipelines to test plugin repositories (e.g., `naira-plugins`) against multiple versions of `naira` core.
- **Promotion & Verification Pipelines**: Standardize release promotion stages using `component-testbed` as a gate for integration and end-to-end testing before final publication.
- **Versioned Documentation**: Build automation to publish version-specific documentation sites (e.g., matching git tags or major/minor branches) so docs match released behaviors.
- **Rollback and State Management**: Define clear procedures and rollback automations, specifically addressing backward-compatible database schema migrations and Kubernetes CRD updates.

---

## Links
