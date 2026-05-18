---
status: accepted
date: 2026-05-06
---

# Build a Custom Catalog Core for Naira

## Context and Problem Statement

Naira needs a way to track complex relationships between technical entities. Current tools like Backstage or ORD often require developers to manually update manifest files. This creates a high barrier to entry and is not easy to maintain.

While Naira will support static files, our main goal is automatic discovery. We want to simplify onboarding through integrations. Once set up, Naira will automatically pull data into a central catalog.

Naira is not a "source of truth." It is a unified lens. It gathers data from specialized tools that already exist, such as:
- Models: MLflow
- Code & Teams: GitHub
- Deployments: ArgoCD / Flux
- Inference: LiteLLM
- Datasets: OpenMetadata

We need a core architecture that is consistent and handles data from many sources. It must also be ready for new types of entities like MCP servers, Skills, and Agents.

## Decision Drivers

- Foundational Role: The catalog is the "heart" of Naira; every other feature will be built on top of it.
- Flexibility: The system must allow to interact with and modify the metadata schema easily
- Integration: The ability to pull data from various external plugins reliably.

## Considered Options

* Build a custom catalog core
* Use DataHub
* Use OpenMetadata
* Use a database adapter hybrid
* Use Cartography

## Decision Outcome

We will build our own custom core. This system will focus on accepting and managing Nodes and Relationships to power Naira's views and future features.

Instead of building on DataHub, OpenMetadata, we will treat them as "plugins of plugins". This means Naira will ingest aggregated data from these systems rather than being replaced by them

### Consequences

* Good, because Naira gain full control over the roadmap and schema flexibility.
* Bad, because the team will need to invest more time upfront in building the catalog.

## Pros and Cons of the Options

### Build a custom catalog core

* Good, because it provides full independence.
* Good, because it avoids unnecessary bloat or complexity from third-party tools.
* Good, because it can be built specifically for Naira's requirements.
* Bad, because it requires significant effort to build and maintain from scratch.

### Use DataHub

* Good, because it is ready to use out of the box.
* Good, because it supports OpenLineage.
* Good, because it offers high data consistency via Kafka.
* Good, because it supports near real-time streaming updates.
* Good, because it can simplify onboarding for users already using this ecosystem.
* Good, because it provides a high number of available plugins.
* Bad, because it is not possible to change schemas through API, DataHub requires PDL files for new entitiy types.
* Bad, because it creates a heavy dependency whose license and roadmap may diverge from Naira's needs.

### Use OpenMetadata

* Good, because it is ready to use out of the box.
* Good, because it supports OpenLineage.
* Good, because it can simplify onboarding for users already using this ecosystem.
* Good, because it provides a good REST API and a Go SDK.
* Good, because it provides a high number of available plugins.
* Bad, because changing schemas through the API is not possible, and extending the model may require code-level forks.
* Bad, because it creates a heavy dependency whose license and roadmap may diverge from Naira's needs.

### Use a database adapter hybrid

* Good, because it could allow switching between a custom core and existing systems such as DataHub.
* Bad, because it introduces too much complexity for the initial phase of the project.

### Use Cartography

* Good, because it relies on Neo4j and one service, which is lighter than DataHub or OpenMetadata.
* Good, because its architecture is similar to what Naira wants to build.
* Good, because it is part of CNCF.
* Bad, because it does not provide a straightforward way to edit nodes and relationships from the UI.
* Bad, because Neo4j is a hard dependency and ties the solution to its licensing model.
* Bad, because ingestion is Python-based.
* Neutral, because it still needs more investigation and may remain a useful reference if Naira continues with a custom core.

## More Information

Cartography is a useful reference point for node unification and schema design if Naira continues building its own catalog core.

Related references:

* Cartography: https://github.com/cartography-cncf/cartography
* DataHub: https://datahubproject.io/
* OpenMetadata: https://open-metadata.org/
* OpenMetadata Standards: https://docs.open-metadata.org/latest/main-concepts/metadata-standard/schemas