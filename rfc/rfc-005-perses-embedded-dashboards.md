# RFC 005: Perses as Naira's Embedded Dashboard Foundation

| Field         | Value                                                    |
|---------------|----------------------------------------------------------|
| RFC           | 005                                                     |
| Title         | Perses as Naira's Embedded Dashboard Foundation          |
| Author(s)     | @mkorbi                                                  |
| Target Milestone | M2 - Plugin System & Reference Integrations           |
| Status        | Draft                                                    |
| Type          | Feature                                                  |
| Created       | 2026-05-19                                               |

## Abstract

This RFC proposes adopting **[Perses](https://perses.dev/)**, a CNCF Sandbox project, as Naira's
**native, embedded dashboard solution** for visualizing AI and platform metrics directly within
the OpenMFP UI. Perses is a modern, GitOps-first, Kubernetes-native dashboard tool whose architecture
treats dashboards as first-class declarative resources (CRDs) and whose UI components are explicitly
designed to be embedded into third-party applications.

Perses fills a gap: **dashboards that Naira itself ships and renders in its own UI**.
Today, every "metric view" embedded in Naira (asset detail pages, registry browsers, observability
panels) must be hand-built as bespoke React components or rendered via Grafana iframe embeds. I think both
of which have significant drawbacks. Perses gives us a third path: declarative dashboards-as-code
that ship with the platform, version-controlled alongside our other CRDs, and embedded as native
micro-frontend components.

The work is scoped to **embedded visualization for platform-shipped dashboards**. Operator-grade
exploratory dashboarding remains the domain of Grafana (via the existing plugin) or whatever
observability stack the operator already runs.

## Motivation

Naira's positioning as a "single pane of glass" creates a specific
requirement: **the platform must render meaningful, AI-asset-contextual visualizations directly in
its UI**, not just link out to external tools.

The existing options for getting visualizations into Naira's OpenMFP UI all have major downsides:

### Option 1: Hand-built React components hitting Prometheus directly

Pros: full control, native look-and-feel, no extra dependencies.

Cons: every chart is bespoke code; new visualizations require frontend engineering work; PromQL
queries are scattered across the codebase; non-developers (Platform Peter, AI Engineer rAIner)
cannot author or modify dashboards; there is no versioned source of truth for "what does Naira
show?"; refactoring the underlying queries breaks every consumer.

### Option 2: Grafana iframe embedding

Pros: leverages an existing tool many teams already run; rich panel library; mature query editor.

Cons: iframes break the OpenMFP micro-frontend model (no shared auth, no shared theming, no
deep linking into Naira context, awkward responsive behavior); Grafana's panel embedding requires
anonymous-access configuration or token forwarding that is operationally painful; dashboards live in
Grafana, not in Git alongside the Naira manifests that depend on them; users see Grafana's UI
chrome inside Naira's UI chrome, breaking the "one tool" experience.

### Option 3: Grafana Scenes / dashboard-as-code SDK

Pros: programmatic dashboard composition, JSX-like API.

Cons: tied to Grafana's runtime; still doesn't solve the iframe/embedding problem cleanly; not
GitOps-native; no Kubernetes-native resource model.

### What we actually need

A dashboard tool that is:

1. **Embeddable as a true micro-frontend component**: not an iframe and must compose with OpenMFP's
   shell, share auth context and respect Naira's theming.
2. **Declarative and Kubernetes-native**: dashboards defined as CRDs (or CRD-equivalent YAML)
   so they live alongside the rest of Naira's manifests in Git and benefit from Kubernetes' RBAC and audit machinery.
3. **Multi-datasource**: must speak PromQL today (against Prometheus/Mimir) and ideally LogQL,
   TraceQL, and other backends tomorrow.
4. **GitOps-first**: dashboards are versioned artifacts reviewed via PR, not clicked together
   in a UI and exported as JSON blobs.
5. **Open governance, permissive license**: aligned with NeoNephos / IPCEI-CIS positioning.

### Option 4: Perses
It is a CNCF Sandbox project (joined late 2024), maintained largely
by ex-Grafana and observability-domain engineers, licensed Apache 2.0, with explicit project goals
around being a "dashboard standard" rather than a competing product. Its key architectural choices
align almost exactly with what Naira needs:

- A formal **dashboard schema** defined as protobuf/JSON Schema, intentionally portable
- **First-class CRDs** for `PersesDashboard`, `PersesDatasource`, `PersesProject` (via the
  perses-operator), making dashboards a Kubernetes-native concept
- **Static dashboards-as-code** workflow (CUE-based or YAML-based) with strong validation
- A **React component library** (`@perses-dev/dashboards`) explicitly designed for third-party
  embedding
- **Headless rendering** capability for SSR or testing
- **Plugin architecture** for datasources, panel types and variables

The trade-offs (Perses is younger than Grafana, has fewer panel types, smaller community) are
real but acceptable for our use case: we don't need the full Grafana feature surface; we need a
small, well-defined set of embedded visualizations that we control end-to-end.

## Goals

1. **Embedded-first.** Adopt Perses specifically as the foundation for dashboards that render
   **inside Naira's OpenMFP UI**. Every asset detail page that needs a chart gets it via a Perses
   component, not a hand-built React chart and not a Grafana iframe.

2. **Dashboards-as-code.** All platform-shipped dashboards live as `PersesDashboard` resources
   (CRDs or static YAML, depending on deployment mode) committed to Git, reviewed via PR,
   reconciled by Flux. No clickops dashboard editing in production paths.

3. **Datasource abstraction.** Naira's UI components reference dashboards by name, not by
   datasource. Operators configure `PersesDatasource` resources once per environment; switching
   from Prometheus to Mimir, or pointing at a different Loki, is a manifest change with no UI
   code impact.

4. **OpenMFP micro-frontend wrapping.** A Naira-maintained MFE (`naira-perses-mfe`) wraps the
   Perses React components, handles auth pass-through, applies Naira's theme, and is registered
   as an OpenMFP extension point so any other Naira MFE can render a dashboard by referencing
   its name.

5. **Coexistence with Grafana.** Perses and the existing Grafana LGTM plugin solve different
   problems and ship in parallel. Naira ships platform dashboards via Perses; users who want
   deep exploratory analysis still get one-click deep links to Grafana via the existing plugin.

6. **A bundled set of starter dashboards.** Ship Naira with a small, opinionated set of
   `PersesDashboard` manifests covering the M2 asset types: Inference Endpoint health, Model
   Registry overview, AI Gateway consumption, Webhook ingestion rate. These prove the embedding
   pattern and give plugin authors something to copy.

## Non-Goals

The following are explicitly out of scope for this RFC:

- **Replacing Grafana.** This RFC does not propose deprecating, removing, or competing with the
  Grafana LGTM stack.
- **Replacing Naira's existing OpenMFP UI components for non-dashboard views.** Perses is for
  visualization panels, not for general-purpose UI.
- **Authoring tooling for end-user dashboards.** v1 ships platform dashboards only.
  Self-service "let an AI Engineer build their own dashboard in Naira" is a follow-up explicitly
  deferred to later.
- **Replacing or wrapping Prometheus/Mimir/Loki/Tempo.** Perses is a dashboarding tool; it
  queries existing observability backends. The observability stack itself is operator-provided.
- **Alerting.** Perses does not own alerting; we route alerts through the existing observability
  stack (Alertmanager, Grafana OnCall, or whatever the operator runs).


## Specification

### Terminology

- **PersesDashboard**: a Perses dashboard resource, defined as a Kubernetes CRD by the Perses
  operator, containing panel definitions, variables, and datasource references.
- **PersesDatasource**: a Perses datasource resource (e.g., a Prometheus endpoint) referenced
  by dashboards.
- **PersesProject**: a Perses-native grouping primitive for dashboards and datasources, used
  here to map to Naira workspaces / tenants.
- **Naira Perses MFE**: a Naira-maintained OpenMFP micro-frontend that wraps the Perses React
  component library, applied to render dashboards inside Naira's UI.
- **Embedded panel**: a single Perses panel rendered standalone (without surrounding dashboard
  chrome) inside a Naira asset detail page.
- **Embedded dashboard**: a full Perses dashboard rendered inside Naira (with row/grid layout
  preserved) inside a dedicated Naira view.

### Dashboard Definition Pattern

Each `PersesDashboard` is a Kubernetes manifest committed to Git, reconciled by Flux, and applied
to the Perses Operator. Plugin authors add new dashboards by submitting PRs to this directory.

### Auth & RBAC

- **End users** authenticate to Naira via Keycloak (per the existing IAM story). The Naira BFF
  forwards a service-to-service token to Perses, scoped to read-only dashboard access.
- **Perses itself** is not directly exposed to end users. There is no Perses login UI in the
  browser; all access flows through Naira.
- **OpenFGA policies** (per the existing access-control story) determine which workspaces a
  user can see; the Naira BFF translates this into the appropriate Perses project scoping.
- **Dashboard authoring permissions** live in Git/GitHub (PR review on `dashboards/`), not in
  Perses' own RBAC. This is the natural consequence of GitOps-first.

### Ideas for Starter Dashboards (M2 Deliverable)

Shipping the following dashboards with v1, each as a `PersesDashboard`:

1. **`inference-endpoint-health`**: QPS, latency p50/p95/p99, error rate, GPU utilization (if
   available). Embedded into the Inference Endpoint detail page.
2. **`model-registry-overview`**: model count by stage, recently-promoted models, version churn.
   Embedded into the Model Registry browser.
3. **`ai-gateway-consumption`**: token consumption by route, request count by upstream
   provider, rate-limit hit counts. Embedded into the AI Gateway / LiteLLM plugin's view.

Each dashboard references the same `default-prometheus` `PersesDatasource`. Operators override
the datasource at deployment time to point at their own Prometheus/Mimir.

### Plugin Author Workflow

A plugin author who wants to ship a dashboard with their plugin:

1. Authors a `PersesDashboard` YAML in their plugin's `dashboards/` directory.
2. PR is reviewed by the maintainers and the Platform Engineering team (the latter
   because the dashboard becomes part of the platform's data surface).
3. On merge, e.g. Flux applies the new dashboard CR; the Perses Operator reconciles it; the
   dashboard becomes immediately addressable from Naira's UI via its name.
4. The plugin's MFE uses `<NairaPersesDashboard name="my-plugin-dashboard" />` to render it.

No Perses-side configuration is required beyond the YAML manifest. No JSON-blob export/import.
No iframe wrangling.

## Rationale and Alternatives

### Why Perses specifically, and not just commit to Grafana?

Grafana is the dominant tool for operator-grade exploratory dashboarding and we are not
displacing it. But Grafana was never designed to be embedded as a micro-frontend component:

- **Iframe-only embedding** in practice. Grafana's "scenes" and panel-embedding APIs are
  improving but still assume Grafana itself is the host environment. Theming, auth, and
  context-variable propagation are awkward.
- **JSON dashboards aren't truly GitOps-native.** Grafana's provisioning supports YAML/JSON
  dashboard sources, but the canonical experience is "click in the UI, export, commit." The
  result is dashboards that drift between environments and are painful to review in PR diffs.
- **Operator-first, embedded-second.** Grafana's UX optimizes for operators exploring data, not
  for application developers embedding small visualizations into their own UI.

Perses inverts these priorities. It is **embedded-first, GitOps-first, declarative-first**.
For our specific use case a platform that needs to show dashboards inside its own UI Perses
is the better fit.

### Why not build our own native React chart library?

Pros: maximum control, no extra runtime dependency. 
Cons: every new visualization is engineering work; we'd reinvent variable substitution, time range pickers,
PromQL query building and panel layout all of which Perses already provides; non-engineers
can't contribute dashboards; we lose the Perses ecosystem's future contributions.

### Why not Grafana Scenes (programmatic dashboard composition)?

Grafana Scenes is genuinely interesting and partially addresses the embedding pain. But it
remains tied to Grafana's runtime, is not GitOps-native (Scenes definitions are TypeScript
code, not declarative manifests), and does not give us the Kubernetes CRD model we want for
dashboards as platform artifacts.

### Why not contribute to an existing project (e.g., extend Backstage's catalog visualizations)?

Backstage has its own visualization story (TechInsights, custom plugins) which is bespoke React
work, the same problem we're trying to avoid.

### Why CNCF Sandbox and not a more mature tool?

Perses is backed by
Red Hat, Amadeus and core observability-domain contributors; the project is actively developed
and has clear architectural direction. The risk is real but bounded: if Perses stagnates, the
dashboards we author are still standard CycloneDX-style declarative artifacts that could be
migrated to a successor tool. We are not locking ourselves into a vendor; we are betting on a
standard.

## Drawbacks

- **Maturity gap with Grafana.** Perses has fewer panel types, less ecosystem tooling, fewer
  community dashboards and a smaller user base than Grafana. Mitigation: scope v1 to a small
  set of platform-shipped dashboards we control end-to-end; revisit feature gaps with later major versions.

- **Operational footprint.** Adds another platform component (Perses Operator + server) to
  install, maintain, upgrade, and monitor.Resource footprint is modest.

- **Two dashboarding tools to explain to users.** Operators will reasonably ask "why both
  Perses and Grafana?" Mitigation: "Perses for what
  Naira ships and renders; Grafana for what you explore and incident-respond against."

- **Naira Perses MFE becomes a critical path.** If the MFE breaks, every dashboard in Naira
  breaks. Mitigation: standard frontend resilience patterns (error boundaries, graceful
  degradation to "View in Grafana" deep link); the MFE is small and well-scoped; integration
  tests cover the embedding contract.

- **Theming integration work.** Perses has its own theming system; mapping Naira's design tokens
  onto it is real frontend work.

## Future Work (out of scope for this RFC)

- **Self-service dashboard authoring inside Naira.** Allow AI Engineer rAIner and
  Application Developer Abdel to compose dashboards in-Naira using Perses' authoring UI,
  saved as `PersesDashboard` CRs in a tenant-scoped Git location, reviewed by their workspace
  owners.

- **Cross-dashboard navigation primitives.** Naira-aware drill-down: click a model in a
  dashboard panel, navigate to the model's Naira detail page; click an endpoint, jump to its
  AI Gateway view. Requires Perses extension points that don't fully exist yet but are on the
  project's roadmap.

## References

- Perses project: https://perses.dev/
- Perses GitHub organization: https://github.com/perses
- Perses Operator: https://github.com/perses/perses-operator
- Perses dashboard schema documentation: https://perses.dev/perses/docs/api/dashboard
- Perses React components: https://github.com/perses/perses/tree/main/ui
- CNCF Sandbox announcement (Perses): https://www.cncf.io/projects/perses/
- Naira existing Grafana LGTM reference integration:
  [`product management/features/2026-05-05-reference-integration-observability-grafana-lgtm.md`](../../../product%20management/features/2026-05-05-reference-integration-observability-grafana-lgtm.md)
- Naira OpenMFP architecture (ADR-0001 § OpenMFP / Luigi):
  [`product/Milestone-1/adr-0001-naira-platform-design-foundations.md`](../../../product/Milestone-1/adr-0001-naira-platform-design-foundations.md)
- Naira OpenBao platform component (similar always-on platform pattern):
  [`product management/features/2026-04-27-openbao-testbed-infrastructure.md`](../../../product%20management/features/2026-04-27-openbao-testbed-infrastructure.md)

## Changelog

- 2026-05-19: Initial draft.
