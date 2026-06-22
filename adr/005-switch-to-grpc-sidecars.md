---
status: accepted
date: 2026-06-22
written-by: Emil Libera
---

# Switch from go-plugin to gRPC Sidecars

## Context and Problem Statement

During the implementation of [ADR-004](004-backend-plugin-system.md), we found that HashiCorp `go-plugin` does not fit well into a Kubernetes-native environment. Running plugins as local sub-processes introduced operational overhead that we want to eliminate.

## Decision Drivers

- **K8s-Native Operations:** Utilizing standard container mechanisms instead of custom host-level tooling.
- **Observability:** Independent log streams per plugin.
- **Maintainability:** Simple packaging without binary management in Dockerfiles.

## Considered Options

- **HashiCorp go-plugin** (Previous choice - local sub-processes)
- **gRPC Sidecars** (Plugins as separate containers inside the core Pod)

## Decision Outcome

We chose **gRPC Sidecars**. Each plugin will run as a separate container within the Naira core Pod, communicating via gRPC over localhost. This leverages Kubernetes features for configuration and lifecycle management.

---

## Pros and Cons of the Options

### gRPC Sidecars

* **Good:** Native observability – logs are separated by container runtime out-of-the-box.
* **Good:** Native configuration – CLI/CMD flags and env vars are defined directly in the K8s deployment manifest.
* **Good:** Standard delivery – plugins are standard OCI images, removing `chmod +x` from Dockerfiles.
* **Bad (Resource Bloat):** Tight coupling of RAM/CPU. 10 plugins requiring 64Mi each adds 640Mi to a single Pod.
* **Bad (Pod Density):** Managing 10+ containers in one Pod definition can become unwieldy.

### HashiCorp go-plugin

* **Good:** Low resource footprint (single container deployment).
* **Bad:** Higher complexity – requires a custom mechanism in Catalog to proxy flags to sub-processes.
* **Bad:** Poor observability – stdout/stderr from all plugins is mixed into one container log stream.

---

## Technical Outlook / Future Direction

To solve the "many containers in one Pod" issue and high resource usage, our next step will be moving plugins to **independent Kubernetes Pods**. 

This will require considering:
- Secure inter-pod communication.

---

## Links

* Supersedes [ADR-004](004-backend-plugin-system.md)