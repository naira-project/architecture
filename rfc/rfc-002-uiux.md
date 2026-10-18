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

This RFC defines the UI/UX architecture for the **Naira platform's asset visualization layer, specifically concentrating itself on the asset aggregation**. Naira aggregates AI assets scattered across a company's ecosystem such as models, datasets, agents, MCP servers, skills, and more into a unified, source-agnostic representation via the Core API design. This RFC proposes three complementary visualization modes to surface that data:

1. **Graph View**: an interactive, filterable topology map for discovering lineage and relationships between AI assets across clusters and environments.
2. **Tabular View**: a detail-rich, filterable data grid for deep inspection of a specific asset class.
3. **Merged View**: a merged view which wraps around a tabular view that concentrates upon both relationships across different entities as well as details of each entity.

All views are grounded in OpenMFP (Open Micro Frontend Platform), which provides the extensible UI shell, module federation, navigation, and runtime context into which all views are embedded as pluggable micro-frontends. Property-filtered and and search-enabled design must remain consistent across both modes.

---

## Motivation

Naira's Core API design produces a source-agnostic, normalized representation of AI assets regardless of origin scanner or framework. This is necessary but not sufficient: engineers, security teams, and platform operators need intuitive, purpose-built interfaces to make sense of what can be hundreds or thousands of interrelated AI components distributed across projects, clusters, and deployment environments.

Three categories of insight are regularly needed:

**Relational / lineage insight.** Questions like "which downstream agents depend on this model?" or "what MCP servers does this agent fleet expose?" are fundamentally graph questions. A tabular list cannot convey topology.

**Detail insight.** Questions like "show me all models in production that are not pinned to a fixed version" or "what are the declared permissions of every MCP server registered this month?" are filtering and aggregation questions. A graph cannot convey tabular density.

**Alternative merged insight.** User may feel overwhelmed by filtering different views back and forth just to reach out to some relationships or get some details. This is why relationships in a graph view can also be provided via property-based filtering in a tabular view as an alternative.

---

## Goals

1. **Three-mode visualization.** Deliver a Graph View, Tabular View, and a Merged View as distinct, complementary interfaces within the same platform shell.
2. **OpenMFP-native embedding.** Both views are implemented as OpenMFP micro-frontend modules, conforming to its module federation contract and consuming the OpenMFP navigation and context APIs.
3. **Rich dashboard ecosystem with transparent switching between different dashboard structures.** Users can pick up provided dashboards or they can customize their own dashboard based on existing templates (such as "list of models", "list of agents"...). Users will also have the ability to switch between dashboards flexibly, they won't stick to one rigid dashboard. 
4. **Property-based organization as the primary filter primitive.** Entity coloring, graph filtering, and tabular filtering are all driven by a consistent property system, user-customizable at runtime.
5. **Full-text search in both views.** Across entity properties and relationship metadata.
6. **Contextual help.** In-context tooltips and explanations prevent expert-only usability.
7. **Specific subgraph view by clicking on one entity**: The design structure of graph view can be overwhelming for most end users as their AI assets would be scattered across a lot of resources that need to be unified. For mitigation, user can click one entity to open a new subgraph view.  
8. **Table and graph view updated in real-time**: Assets in a company change all the time, the changes could be retrieved in near-real-time by polling mechanisms.

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
| **Property** | A key-value label attached to an Asset or Relationship, sourced from the upstream scanner. |
| **Tag** | A plain string label attached to an Asset or Relationship |
| **OpenMFP** | The extensible UI shell that hosts Naira's micro-frontend modules. |
| **Graph View** | The topology-oriented visualization mode. |
| **Tabular View** | The detail-oriented, filterable data-grid mode. |

---

### OpenMFP Integration

Naira's frontend is not a standalone application; it is a set of **OpenMFP micro-frontend modules** mounted into the shared platform shell. Each view (Graph, Tabular) is compiled as a micro-frontend exposing a well-defined OpenMFP content configuration (`naira/graph`, `naira/tabular`, `naira/merged`). The shell loads these at runtime without a full redeployment cycle.

---

### Graph View

#### Purpose

The Graph View enables **relational discovery and lineage exploration** across all AI assets tracked by Naira. It answers topology questions: what depends on what, what changed, what is at risk due to a transitive dependency.

#### Entry point and role-based defaults

Each user will have the right to pick or customize their dashboard based on AI assets avaiable in Naira. This will provide dynamic graphs that will be pre-filtered tailored to each user's primary concerns.

#### Node representation

- Each node represents a single Asset. The node renders a compact badge: asset-type icon, asset name, and a one-line summary of the most salient attribute (e.g. model version, MCP server URL, dataset size). The salient attribute must be determined by the plugin itself by default, later enabling customization for the user. User would be welcomed with a screen for each AI asset at the beginning, and can select which attribute they want to see right away for each AI asset that they have access to. 
- **Node size** scales with the node's degree (number of inbound + outbound edges), giving a natural visual weight to high-centrality assets such as a foundational model used by many agents or a shared MCP server.
- **Node color** is determined by the asset's primary tag category. The color-to-tag mapping is user-configurable in the filter panel. Color is applied consistently across the graph and the legend.


#### Interaction model

- **Click on node → entity-scoped subgraph.** Clicking any node replaces the current graph canvas with a subgraph showing only the clicked asset and its direct and second-degree relationships. A breadcrumb trail allows navigating back.
- **Click on node → detail panel.** A side panel slides in alongside the subgraph showing the full Core API record for the asset such as all attributes represented as properties.
- **Click on edge → relationship detail.** A tooltip or panel shows relationship type and source scanner.
- **Zoom-based progressive disclosure.** At low zoom, only node name and type icon are visible. At medium zoom, one or two key attributes appear on-node. At high zoom, the full attribute badge is rendered in-place, reducing the need to click into the detail panel.
- **Subgraph navigation.** Clicking a node in the subgraph context drills into that node's subgraph, maintaining back-navigation history. Back-navigation history will be enabled through breadcrumbs, where clicking on a subgraph leads to the path shown as breadcrumb on top of the graph. By clicking on the previous path, user can go to the previous graph view.

#### Filtering and search

- **Tag filter panel** (left sidebar): multi-select tags; the graph hides nodes not matching the active filter set.
- **Asset type filter**: checkboxes for each tracked asset class; they are fully user-adjustable. 
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
- Row click opens a detail drawer similar to the one used in the Graph View's node detail panel, maintaining a consistent information architecture across both views.
- Row selection (multi-select) enables bulk actions such as separate detail pages.

#### Filtering and search

- **Property filter chip bar** above the grid: identical property filtering primitive as the Graph View, ensuring consistent UX vocabulary. Active filters are rendered as removable chips.
- **Column-level filters**: type-aware filter controls per column (text contains, equals, date range, enum multi-select). Column filters compound with the property filter.
- **Full-text search bar**: searches the same indexed property set as the Graph View. Results highlight matching cells within the grid.
- **Saved filter presets**: users can save a named combination of property filters and search terms as a preset, accessible via a dropdown.

#### Contextual help

Every column header provides an info icon (`ⓘ`). Hovering reveals:
- A plain-language description of the attribute.
- Its source in the Core API schema (field path).
- Where applicable, a link to the relevant scanner.


---

### Navigation between views

The Graph View, Tabular View and Merged View are complementary. Navigation between them is seamless:

- The node detail panel in the Graph View includes a **"View all [asset type]"** link that opens the corresponding Tabular View pre-filtered to the same asset class.
- Any row in the Tabular View includes a **"Show in graph"** or **Show in Merged View** action that opens the Graph View centered on that asset's subgraph.
- User will have the possibility deviating from Merged View with the **Show in graph** or **Show this asset in Tabular View** actions.
---


## Rationale and Alternatives

**Why three distinct views rather than a unified view or two views?**
Graph and tabular representations optimize for fundamentally different cognitive tasks. A graph excels at topology and lineage; a table excels at enumeration and filtering. The navigation bridge between the two views provides the integration without compromising either. A merged view is added for understanding if a unified view of relationships and details would be sufficient. If yes, then one/some of the unused views could be discarded and their features would be embedded into remaining views.

**Why property-based filtering as the primary organizing primitive?**
Properties are the lowest-common-denominator organizational unit across all asset sources (scanner output, manual annotation, CI metadata). A property-first filtering model works regardless of whether the operator uses project-based, team-based, environment-based, or risk-level-based organization schemes. Fixed category hierarchies would impose a single organizational model the platform cannot enforce.

**Why force-directed layout as the default?**
Force-directed layouts reveal clustering and centrality without requiring the operator to know the graph structure in advance. Alternative layouts (hierarchical, radial) are provided for users who already know what they are looking for and need a specific structural framing.

---

## Drawbacks

- **Graph performance at scale.** Force-directed layouts are computationally expensive for graphs with thousands of nodes. v1 mitigates this through server-side pre-computation of layout coordinates and a progressive rendering strategy (load visible viewport first). Very large fleets may require switching to a hierarchical or cluster-aggregated layout by default. In this RFC, the mitigation for this would be customized dashboards by the user, where a user selects specific AI assets that they need for showing and making actions with. This provides limited assets for each user by default, providing performance gains for the graph view. However, this technique must be improved and combined with other methodologies.
- **Property mechanism is user-dependent.** The power of property-based filtering depends on key-value pairing discipline. Assets without any properties reduce the utility of both the color coding and the filter panel. Mitigation: the Tabular View surfaces an "assets without properties" KPI tile to create visibility into the gap.

---

## Future Work (out of scope for this RFC)

- **Provide graph grounded LLM retrieval for graph view**: By providing a relational core API design, Naira enables users to interact with the graph in a flexible way. By ingesting the graph structure to a LLM, user will have flexible search which they can ask to Naira. This can be embedded as an AI assistant with a natural-language query interface exposed within the OpenMFP shell context.
- **User feedback sessions.**: After having a mature platform, structured 30–60 minute moderated sessions with representative users from each role to identify friction points can be organized. Findings feed a UI backlog maintained in alignment with the TSC.
- **Edge styling** is a possible extension to be used inside Naira Graph View. Since there will be a lot of relationship types, using different styles for each edge would be a lot overwhelming Instead we can concentrate on edge thickness where edges could be optionally encoded based on call frequency (where runtime data is available). However, this should not be on the main features.
- **Multi-column sort** will be supported in the later stages based on how much possible value it would provide.
---

## References

- Naira Core API Design
- OpenMFP documentation
- *Style and substance: design the perfect graph visualization* — YouTube
- *Designing for Success: UX Principles for Internal Developer Platforms* — Kirsten Schwarzer

---

## Changelog

- 2026-05-20: Initial draft
- 2026-05-21: First iteration based on feedback from @mkorbi and @akavel-reply
- 2026-05-27: Second iteration based on feedback from @libera13 and @Daviidg