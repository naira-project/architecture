

## Context and Problem Statement

Naira need a plugin system that allows us (and eventually others) to add new integrations without breaking the core engine. Frontend is already modular thanks to openMFP. More about the need for a plugin system https://liquid-naira.atlassian.net/wiki/spaces/Naira/pages/11993089/ADR+-+002+-+Custom+vs.+existing+product+for+Catalog#Context  

## Decision drivers

- Developer Experience: It must be very easy and "friendly" to write a new plugin.
- Speed to Market: We don't want to spend weeks setting up complex infrastructure before writing the actual logic.
- Evolutionary Growth: The system should start simple and only get more complex when the requirements (like security or external contributors) actually demand it. 


Alternatives considered

Document the alternatives evaluated. The chosen option is highlighted in green.

WebAssembly (with Extism) – s / s  

: secure and sandboxed

: plugins can't crash the main app

: small resource usage

: supports many languages

: too much "magic" and setup overhead for the start

: how to set up permissions per plugin might not be obvious for platform engineers. We'll need to build the logic to manage plugin permissions.

 it might be overkill while we are still figuring out the basic requirements

Hashicorp go-plugin 

: battle-tested (used by Terraform)

: plugins run as separate processes, so they can be written in native Go and have access to standard libraries

: supports many languages, open protocol https://github.com/hashicorp/go-plugin/blob/main/docs/guide-plugin-write-non-go.md 

: Slightly more complex than local code because it involves communication between the main app and the plugin process.

Plugins as Internal Packages (The Winner for now)

: Easiest way to start. No extra overhead. Just write a new Go package inside the Naira repository and call it.

: Adding a new plugin requires code changes in Naira

The above solutions (Internal Packages, Hashicorp go-plugin, and WebAssembly) all share a common trait: they assume the plugin runs as a local process described here https://liquid-naira.atlassian.net/wiki/spaces/Naira/pages/4227073 

However, as Naira grows, we may need to expose a Network API. This decouples the plugin from the core engine's lifecycle and allows for more flexible data ingestion.

Data Flow Strategies

I see two possible flows.  

The first, where the plugin simply sends data to the catalog whenever it wants. This API could be used by a CI/CD job or serve as a CRUD for the UI. “Push” mode

And the second, where the catalog triggers plugins to run a sync job. “Pull” mode

Both solutions require Ingestion API, which accepts entities from remote plugins (potentially each require different API):

“Pull” - we would need to think if we accept only full snapshot, or delta is acceptable. If we need guarantee of delivery etc. 

“Push”- we would need to implement CRUD API on entities.

Deployment Flexibility

By using a Network API (REST or gRPC), the "where" and "how" of plugin deployment becomes almost limitless:

Kubernetes Native: Plugins can run as long-lived Pods (grouped by permission levels), or as ephemeral K8s Jobs that spin up, sync, and die to save resources.

Automation-Centric: Plugins can be part of a CI/CD pipeline.

Ad-hoc: A developer could even run a custom bash script from their local machine to feed data into the Catalog.

Summary

network API - communication over gRPC/REST between plugin/catalog

: language agnostic

 : friendly permissions management. Platform engineers can use standard k8s RBAC and Network Policies, to restrict exactly what a plugin pod can touch.

 : if plugins are k8s deployments, it allows to use GitOps for plugins deployment

: harder to trace the error between plugin and catalog, may require distributed tracing

: needs more careful API versioning, as we cannot in one go update all plugins, plugins could exist outside of Naira

: require separate API. If we require a full snapshot, we'll need a Job API for the asynchronous upload of data that is too large for a single request. https://developer.salesforce.com/docs/atlas.en-us.api_asynch.meta/api_asynch/create_job.htm  

 : require authentication system for plugins

Decision

We will take a phased approach to keep things simple but scalable:

Phase 1 (Current): Write plugins as internal Go packages. This lets us move fast and build the core integrations ourselves.

Phase 2 (Growth): Once we want to allow external people to write plugins, we will move to Hashicorp go-plugin. This allows them to ship their own binaries.

Consequences

Positive: We don't over-engineer the system on day one. The complexity of the architecture only grows as the actual needs of the platform grow.

Negative: We will eventually have to refactor the internal packages into the go-plugin format, which creates some "planned" technical debt.

Related links

Mentioned stack:

https://github.com/hashicorp/go-plugin

https://extism.org/

https://github.com/openmfp

Reference ADR regarding the choice of plugin system:

https://github.com/helm/community/blob/main/hips/hip-0026.md 

Notes & review comments

Future directions:

Choosing one of the plugin systems now doesn't lock us out of switching to another system in the future. Thanks to the main ingestion interface, we can create adapters for additional plugin types (choosing go-plugin doesn't rule out using WASM down the road)