---
status: accepted
date: 2026-06-09
written-by: Emil Libera
---

# Design a Scalable Plugin Architecture for Naira

## Context and Problem Statement

Naira need a plugin system that allows us (and eventually others) to add new integrations without breaking the core engine. Frontend is already modular thanks to openMFP.

## Decision Drivers

- Developer Experience: It must be very easy and "friendly" to write a new plugin.
- Speed to Market: We don't want to spend weeks setting up complex infrastructure before writing the actual logic.
- Evolutionary Growth: The system should start simple and only get more complex when the requirements (like security or external contributors) actually demand it. 

## Considered Options

* **WebAssembly (via Extism):** Running sandboxed plugins as local WASM modules.
* **HashiCorp go-plugin:** Running plugins as separate local processes communicating via gRPC.
* **Plugins as Internal Packages:** Organizing integrations as native Go packages inside the core Naira repository.
* **Network API (REST/gRPC):** Fully decoupling plugins by exposing an ingestion API over the network.

## Decision Outcome

We will implement HashiCorp go-plugin, because it balances the need for process isolation, multi-language support, and native performance without the heavy infrastructure overhead of a full Network API or the strict limitations of WASM runtimes at this stage.

---

## Pros and Cons of the Options

### HashiCorp go-plugin

* **Good:** Battle-tested architecture (powers the ecosystem for Terraform and Vault).
* **Good:** Plugins run as separate processes, meaning a plugin crash cannot take down the core Naira engine.
* **Good:** Supports multiple languages through an open RPC/gRPC protocol, while allowing native Go plugins to access standard libraries easily.
* **Good:** Plugins can be loaded dynamically. Catalog can watch a directory for new binaries and register them at runtime.
* **Bad:** Slightly more complex than local packages due to IPC communication, lifecycle management, and error tracing between the main app and the plugin process.
* **Bad:** It's not k8s native. Plugins with catalog will be in one pod, which doesn't allow for easy scaling, and doesn't allow for granular RBAC

### Plugins as Internal Packages

* **Good:** Easiest way to start with absolutely zero operational or communication overhead.
* **Good:** Developers can write standard Go code and directly utilize the core engine's utility packages.
* **Bad:** Adding, updating, or fixing a plugin requires a full code change and redeployment of the Naira core application.
* **Bad:** Forces all plugins to be written in Go and live in the same repository.

### WebAssembly (with Extism)

* **Good:** Highly secure, isolated, and sandboxed execution environment.
* **Good:** Extremely low resource usage and fast startup times.
* **Good:** Good polyglot support (plugins can be written in Rust, Go, TypeScript, etc.).
* **Good:** Plugins install don't require catalog restart. Modules are loaded/unloaded dynamically into the runtime memory.
* **Bad:** Introduces too much setup overhead and "magic" for an early-stage project.
* **Bad:** Managing granular permissions (e.g., giving a specific WASM module network access) requires custom host-function logic that we would need to build.

### Network API (REST / gRPC)

* **Good:** Complete deployment flexibility. Plugins can run as long-lived Kubernetes Pods, ephemeral K8s Jobs (to save resources during syncs), or even local ad-hoc bash scripts feeding data via `curl`.
* **Good:** Infrastructure-friendly security. Platform engineers can enforce access control using standard Kubernetes RBAC and Network Policies.
* **Good:** Fully language-agnostic and trivial to manage via GitOps pipelines.
* **Good:** Plugins install don't require catalog restart. Plugins are independent services, core simply updates its internal routing table.
* **Bad:** Demands a highly mature, strictly versioned Ingestion API from day one, including complex authentication and bulk/asynchronous Job APIs for large snapshots.

---

Related references:

* HashiCorp go-plugin: https://github.com/hashicorp/go-plugin
* Extism (WASM Framework): https://extism.org/
* OpenMFP Core: https://github.com/openmfp
* Helm Plugin Architecture Reference (HIP-0026): https://github.com/helm/community/blob/main/hips/hip-0026.md