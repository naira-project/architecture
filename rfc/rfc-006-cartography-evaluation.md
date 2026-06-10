# RFC-006: Cartography Evaluation as a Connector Framework

| Field         | Value                                                    |
|---------------|----------------------------------------------------------|
| RFC           | 006                                                      |
| Title         | Cartography Evaluation as a Connector Framework          |
| Author(s)     | @Daviidg                                                 |
| Target Milestone | M2 - Plugin System & Reference Integrations           |
| Status        | Draft                                                    |
| Type          | Feature                                                  |
| Created       | 2026-06-10                                               |


## Summary

This RFC documents the evaluation of [Cartography](https://github.com/cartography-cncf/cartography) as a potential connector/integration framework for Naira. The goal is to determine whether Cartography can be integrated, adapted, or used as a reference point for Naira's integration layer.


## Motivation

Naira needs to discover and surface infrastructure and AI assets (models, datasets, MCP servers, …) from a wide range of external systems (cloud providers, ML platforms, identity providers, and Kubernetes clusters). Building connectors for each source from scratch is costly.

The core question is: **Can we use Cartography, an existing, well-maintained connector library rather than building everything ourselves?**


## What is Cartography?

Cartography is an open-source Python tool (Apache 2.0, originally created at Lyft) that extracts infrastructure and identity assets and the relationships between them from cloud and SaaS providers, and consolidates them into a single [Neo4j](https://neo4j.com/) graph database. 

Cartography has no UI of its own. Any visualization is provided directly by Neo4j's tooling. Once the data is in the graph, it is queried with Cypher to answer questions that span across services, accounts, and providers.

It is a [CNCF Sandbox project](https://www.cncf.io/sandbox-projects/), written entirely in Python, with an active community of contributors.

### Plugin System and Sync Process

The sync process is **unidirectional and read-only**. Cartography pulls data from each integration's source API and writes it into Neo4j. It never writes back to or modifies those sources.

Every intel module (plugin) implements the same four-phase contract:

| Phase | Description |
|---|---|
| **GET** | Calls the provider API and returns a plain `list[dict]`. This get function is intentionally kept simple, with no data handling or retries. |
| **TRANSFORM** | Reshapes raw dicts into the shape the graph wants. |
| **LOAD** | The standardized, reusable core. Modules **do not write Cypher by hand**. Instead, they declare the target shape using Python dataclasses (`CartographyNodeSchema`, `CartographyNodeProperties`, `CartographyRelSchema`) and call a generic `load()` function. |
| **CLEANUP** | Removes stale data without diffing — anything not written in the current run is deleted. |

This contract is consistent across all 30+ intel modules, making it straightforward to add new connectors. The process to submit and review plugins is also structured and well-documented.

### Node and Relationship Model

Cartography extracts unique nodes and relationships for each intel module independently. It has no concept of grouped or shared entity types. The resulting graph is fragmented per module:

- AWS → `:AWSAccount`, `:EC2Instance`, `:IAMUser`, …
- Azure → `:AzureSubscription`, `:AzureTenant`, `:AzureVirtualMachine`, …
- Okta → `:OktaUser`; Entra → `:EntraUser`

There is no shared parent schema across providers. This means cross-provider queries (e.g. "list all cloud accounts I own") require knowing all module-specific node labels in advance.

Cartography offers two potential mechanism to correlate heterogeneous data across modules:

#### Matchlinks

Cross-module links can be established via [MatchLinks](https://cartography-cncf.github.io/cartography/dev/matchlinks.html), which allow connecting resources from separate sources by matching on shared identifiers. However, this still requires manual work.

#### Ontology 

Another option for the fragmented per-module entity problem is Cartography's recently added [ontology support](https://github.com/cartography-cncf/cartography/discussions/1579), which sits on top of the per-module graph and unifies entities across modules two ways:

**1. Abstract ontology nodes** — provider-agnostic nodes (e.g., `:User`, `:Device`, `:PublicIP`) created separately from module nodes. An `EntraUser` and an `OktaUser` both link to the same `:User` node, which normalises identity data across sources. These are created by the `intel.ontology` module after the per-module sync phases.

**2. Semantic labels** — extra Neo4j labels applied directly to module nodes (e.g., `:EntraUser:UserAccount`). No separate node is created; the node carries the label plus normalised `_ont_*` properties (`_ont_email`, `_ont_source`, etc.) for consistent querying.


## Adoption Assessment

### Pros

- **30+ read-only integrations out of the box.** AWS, Azure, GCP, Okta, GitHub, Kubernetes, and more. All seem to be actively maintained.
- **Cross-module relations.** MatchLinks and the ontology layer link resources across sources via shared identifiers, enabling unified cross-provider queries. This however, still requires manual relationship definitions and is not a focus of the tool.
- **CNCF Sandbox, Apache 2.0.** Well-governed, open-source, and safe to adopt or reference.
- **Consistent plugin contract.** The GET → TRANSFORM → LOAD → CLEANUP pattern is clean and followed by all plugins. 

### Cons

- **Neo4j is non-optional.** Cartography is tightly coupled to Neo4j. Adopting it means committing to a graph database, which conflicts with Naira's current relational-based storage approach (see RFC-004).
- **Cartography assumes sole ownership of the Neo4j instance.** The CLEANUP phase deletes anything it did not write in the current run. Running Cartography alongside other writers on the same instance would result in data loss.
- **Python-only**. Cartography is written entirely in Python, so adopting its runtime would force Naira to take on a Python dependency in its core stack 
- **Security-centric view.** Cartography is built primarily around security posture (IAM, network exposure, identities), whereas Naira's focus is more focused on the AI domain. 
- **Minimal AI/model coverage.** Cartography has almost no AI-asset modeling. The only AI-specific pieces are the recently added [AIBOM plugin](https://cartography-cncf.github.io/cartography/modules/aibom/schema.html) and a thin `AIModel` ontology type (a wrapper over cloud-provider model registries). There is no rich schema for models, datasets, MCP servers, or other AI assets central to Naira.


## Recommendations

In the current initial phase, Cartography **should not** be adopted as a dependency for Naira. The mandatory Neo4j coupling, Python-only runtime and mismatched entity model make it incompatible with Naira's current architecture.

That said, this decision is tied to Naira's current architecture. It would be worth revisiting Cartography if either of the following changes:

- **Naira moves to Neo4j.** The single biggest blocker is the mandatory graph-database coupling. If Naira's storage strategy shifts toward a graph model, Cartography's would become easy to incorporate.
- **Naira wants to offer a security-focused view.** Cartography's entity model and 30+ integrations are oriented around security posture (IAM, network exposure, identity attack paths). If Naira's scope expands to include a security or attack-path perspective alongside its AI-asset view, Cartography's existing schema and analysis jobs would be a strong head start rather than a mismatch.


## References

- Cartography GitHub: https://github.com/cartography-cncf/cartography
- Cartography docs: https://cartography-cncf.github.io/cartography
- Cartography MatchLinks: https://cartography-cncf.github.io/cartography/dev/matchlinks.html
- Cartography ontology support: https://github.com/cartography-cncf/cartography/discussions/1579
- Cartography AIBOM: https://cartography-cncf.github.io/cartography/modules/aibom/schema.html
- Cartography Kubernetes schema: https://cartography-cncf.github.io/cartography/modules/kubernetes/schema.html
- RFC-004 (Core API v1 Design): [rfc/rfc-004-core-api-v1-design.md](rfc-004-core-api-v1-design.md)
