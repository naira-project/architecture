# RFC 0002: Unified AI Asset Visualization — Graph & Tabular Views

| Field             | Value                                              |
|-------------------|----------------------------------------------------|
| RFC               | 0002                                               |
| Title             | Unified AI Asset Visualization — Graph & Tabular Views |
| Status            | Draft           |
Author  | @kocakkaan                                   |
| Type              | Feature                             |
| Created           | 2026-05-20                                         |

---

## Abstract

This RFC defines the UI/UX architecture for the **Naira platform's asset visualization layer**. Naira aggregates AI assets scattered across a company's ecosystem such as models, datasets, agents, MCP servers, skills, and more into a unified, source-agnostic representation via the Core API design. This RFC proposes two complementary visualization modes to surface that data:

1. **Graph View**: an interactive, filterable topology map for discovering lineage and relationships between AI assets across clusters and environments.
2. **Tabular View**: a detail-rich, filterable data grid for deep inspection of a specific asset class.

Both views are grounded in OpenMFP (Open Managed Frontend Platform), which provides the extensible UI shell, module federation, navigation, and runtime context into which both views are embedded as pluggable micro-frontends. The role-based, tag-filtered, and search-enabled design must remain consistent across both modes.

---

## Motivation

Naira's Core API design produces a source-agnostic, normalized representation of AI assets regardless of origin scanner or framework. This is necessary but not sufficient: engineers, security teams, and platform operators need intuitive, purpose-built interfaces to make sense of what can be hundreds or thousands of interrelated AI components distributed across projects, clusters, and deployment environments.

Two categories of insight are regularly needed:

**Relational / lineage insight.** Questions like "which downstream agents depend on this model?" or "what MCP servers does this agent fleet expose?" are fundamentally graph questions. A tabular list cannot convey topology.

**Detail insight.** Questions like "show me all models in production that are not pinned to a fixed version" or "what are the declared permissions of every MCP server registered this month?" are filtering and aggregation questions. A graph cannot convey tabular density.

---

## Goals

1. **Two-mode visualization.** Deliver a Graph View and a Tabular View as distinct, complementary interfaces within the same platform shell.
2. **OpenMFP-native embedding.** Both views are implemented as OpenMFP micro-frontend modules, conforming to its module federation contract and consuming the OpenMFP navigation and context APIs.
3. **Role-differentiated defaults.** App Developers, AI Engineers, and Platform Engineers see pre-filtered, role-appropriate views by default while retaining the ability to explore beyond those defaults.
4. **Tag-based organization as the primary filter primitive.** Entity coloring, graph filtering, and tabular filtering are all driven by a consistent tag system, user-customizable at runtime.
5. **Full-text search in both views.** Across entity properties and relationship metadata.
6. **Contextual help.** In-context tooltips and explanations prevent expert-only usability.
7. **Iterative UX improvement via user sessions.** Structured feedback loops inform the backlog.
8. **Specific subgraph view by clicking on one entity**: The design structure of graph view can be overwhelming for most end users as their AI assets would be scattered across a lot of resources that need to be unified. For mitigation, user can click one entity to open a new subgraph view.  
9. **Table and graph view updated in real-time**: Assets in a company change all the time, the changes could be retrieved in near-real-time by polling mechanisms.

---

## Non-Goals

- **Not a custom graph database.** The graph view is a visualization layer over the Core API's existing relationship model; it does not introduce a new persistence layer.
- **Not a custom AI asset registry**. AI assets are already present in the corresponding registries, we need to unify them under one visualization layer.

---

## Specification

### Terminology

| Term | Definition |
|------|------------|
| **Asset** | Any AI component tracked by Naira (model, dataset, agent, MCP server, skill, etc.) represented by the Core API schema. |
| **Relationship** | A directed or undirected edge between two Assets (e.g. `uses`, `trained-on`, `exposes`, `depends-on`). |
| **Tag** | A key-value label attached to an Asset or Relationship, sourced from the upstream scanner or applied manually within Naira. |
| **Subject** | A tracked entity (agent, fleet, project, deployment) |
| **OpenMFP** | The extensible UI shell that hosts Naira's micro-frontend modules. |
| **Graph View** | The topology-oriented visualization mode. |
| **Tabular View** | The detail-oriented, filterable data-grid mode. |
| **Role** | One of three platform persona: App Developer, AI Engineer, Platform Engineer. |

---

### OpenMFP Integration

Naira's frontend is not a standalone application; it is a set of **OpenMFP micro-frontend modules** mounted into the shared platform shell. Each view (Graph, Tabular) is compiled as a separately deployable micro-frontend exposing a well-defined OpenMFP module manifest (`naira/graph`, `naira/tabular`). The shell loads these at runtime without a full redeployment cycle.

---

### Graph View

#### Purpose

The Graph View enables **relational discovery and lineage exploration** across all AI assets tracked by Naira. It answers topology questions: what depends on what, what changed, what is at risk due to a transitive dependency.

#### Entry point and role-based defaults

Each role arrives at a pre-filtered graph tailored to their primary concerns:

| Role | Default visible asset types | Default hidden |
|------|-----------------------------|----------------|
| AI Engineer | Models, Datasets, Skills, Guardrails | Application-layer assets (APIs, frontends) |
| App Developer | Agents, MCP Servers, APIs, Agent-facing Skills | Low-level ML assets (raw datasets, adapters) |
| Platform Engineer | Full fleet: all asset types and all relationships | Nothing hidden; cross-cluster edges visible |

These defaults are persisted per user in the OpenMFP preferences store and overridable at runtime.

#### Node representation

- Each node represents a single Asset. The node renders a compact badge: asset-type icon, asset name, and a one-line summary of the most salient attribute (e.g. model version, MCP server URL, dataset size).
- **Node size** scales with the node's degree (number of inbound + outbound edges), giving a natural visual weight to high-centrality assets such as a foundational model used by many agents or a shared MCP server.
- **Node color** is determined by the asset's primary tag category. The color-to-tag mapping is user-configurable in the filter panel. Color is applied consistently across the graph and the legend.
- **Edge styling** encodes relationship type: `uses` edges are solid; `trained-on` edges are dashed; `exposes` edges use a distinct arrowhead. Edge thickness optionally encodes call frequency (where runtime data is available).

#### Interaction model

- **Click on node → entity-scoped subgraph.** Clicking any node replaces the current graph canvas with a subgraph showing only the clicked asset and its direct and second-degree relationships. A breadcrumb trail allows navigating back.
- **Click on node → detail panel.** A side panel slides in alongside the subgraph showing the full Core API record for the asset such as all attributes represented as tags.
- **Click on edge → relationship detail.** A tooltip or panel shows relationship type and source scanner.
- **Zoom-based progressive disclosure.** At low zoom, only node name and type icon are visible. At medium zoom, one or two key attributes appear on-node. At high zoom, the full attribute badge is rendered in-place, reducing the need to click into the detail panel.
- **Subgraph navigation.** Clicking a node in the subgraph context drills into that node's subgraph, maintaining back-navigation history.

#### Filtering and search

- **Tag filter panel** (left sidebar): multi-select tags; the graph instantly hides nodes not matching the active filter set.
- **Asset type filter**: checkboxes for each tracked asset class; mirrors the role-based defaults but is fully user-adjustable.
- **Environment / cluster filter**: for multi-cluster deployments, nodes can be scoped to one or more registered environments.
- **Full-text search bar**: searches across all indexed asset properties (name, description, version, URL, tags, provider). Matching nodes are highlighted; non-matching nodes are dimmed. Search is debounced; results arrive without a page reload.

#### Layout

The default layout is a force-directed graph. Alternative layouts (hierarchical top-down for lineage-focused views, radial for single-subject subgraphs) are selectable from a toolbar. Layout state is preserved across sessions.

---

### Tabular View

#### Purpose

The Tabular View enables **deep inspection of a specific asset class**. It answers aggregation questions: how many, which version, which tag, which attributes.

#### Structure

Each Tabular View is scoped to a single asset class. Asset classes with a dedicated view at launch:

- Models
- Datasets
- Agents
- MCP Servers
- Skills

Additional asset classes can be registered as new OpenMFP micro-frontend sub-routes without modifying the core tabular framework.

#### Header metrics strip

At the top of every Tabular View, a horizontal strip of scalar KPI tiles summarizes the current filter state such as:

| Asset class | Example KPIs |
|-------------|--------------|
| Models | Total models, pinned vs. unpinned versions, providers |
| MCP Servers | Total servers, servers with open Findings |
| Agents | Total agents, agents with no guardrail attached |

KPI tiles react to the active filter and search state; they always reflect the visible subset, not the global total.

#### Data grid

- Columns are determined by the Core API schema for the asset class. Columns are reorderable, resizable, and hideable per user preference.
- Default sort is name of the asset in an alphabetical order.
- Multi-column sort is supported.
- Row click opens a detail drawer identical to the one used in the Graph View's node detail panel, maintaining a consistent information architecture across both views.
- Row selection (multi-select) enables bulk actions such as bulk tag assignment.

#### Filtering and search

- **Tag filter chip bar** above the grid: identical tag filtering primitive as the Graph View, ensuring consistent UX vocabulary. Active filters are rendered as removable chips.
- **Column-level filters**: type-aware filter controls per column (text contains, equals, date range, enum multi-select). Column filters compound with the tag filter.
- **Full-text search bar**: searches the same indexed property set as the Graph View. Results highlight matching cells within the grid.
- **Saved filter presets**: users can save a named combination of tag filters, column filters, and search terms as a preset, accessible via a dropdown.

#### Contextual help

Every column header provides an info icon (`ⓘ`). Hovering reveals:
- A plain-language description of the attribute.
- Its source in the Core API schema (field path).
- Where applicable, a link to the relevant scanner.

Complex status columns (e.g. "Finding severity", "Schema drift") include a legend tooltip explaining the status values.

---

### Navigation between views

The Graph View and Tabular View are complementary. Navigation between them is seamless:

- The node detail panel in the Graph View includes a **"View all [asset type]"** link that opens the corresponding Tabular View pre-filtered to the same asset class.
- Any row in the Tabular View includes a **"Show in graph"** action that opens the Graph View centered on that asset's subgraph.

---


## Rationale and Alternatives

**Why two distinct views rather than a unified view?**
Graph and tabular representations optimize for fundamentally different cognitive tasks. A graph excels at topology and lineage; a table excels at enumeration and filtering. The navigation bridge between the two views provides the integration without compromising either.

**Why tag-based filtering as the primary organizing primitive?**
Tags are the lowest-common-denominator organizational unit across all asset sources (scanner output, manual annotation, CI metadata). A tag-first filtering model works regardless of whether the operator uses project-based, team-based, environment-based, or risk-level-based organization schemes. Fixed category hierarchies would impose a single organizational model the platform cannot enforce.

**Why force-directed layout as the default?**
Force-directed layouts reveal clustering and centrality without requiring the operator to know the graph structure in advance. Alternative layouts (hierarchical, radial) are provided for users who already know what they are looking for and need a specific structural framing.

---

## Drawbacks

- **Graph performance at scale.** Force-directed layouts are computationally expensive for graphs with thousands of nodes. v1 mitigates this through server-side pre-computation of layout coordinates and a progressive rendering strategy (load visible viewport first). Very large fleets may require switching to a hierarchical or cluster-aggregated layout by default.
- **Tag consistency is user-dependent.** The power of tag-based filtering depends on consistent tagging discipline. Untagged or inconsistently tagged assets reduce the utility of both the color coding and the filter panel. Mitigation: the Tabular View surfaces an "untagged assets" KPI tile to create visibility into the gap.
- **Role defaults require ongoing calibration.** The pre-filtered views for each role are based on current assumptions about what each persona needs. These will need adjustment as real usage data is collected.

---

## Future Work (out of scope for this RFC)

- **Provide GraphRAG for graph view**: By providing a relational core API design, Naira enables users to interact with the graph in a flexible way. By ingesting the graph structure to a LLM, user will have flexible search which they can ask to Naira. This can be embedded as an AI assistant with a natural-language query interface exposed within the OpenMFP shell context.
- **User feedback sessions.**: After having a mature platform, structured 30–60 minute moderated sessions with representative users from each role to identify friction points can be organized. Findings feed a UI backlog maintained in alignment with the TSC.

---

## References

- Naira Core API Design
- OpenMFP documentation
- *Style and substance: design the perfect graph visualization* — YouTube
- *Designing for Success: UX Principles for Internal Developer Platforms* — Kirsten Schwarzer

---

## Changelog

- 2026-05-20: Initial draft