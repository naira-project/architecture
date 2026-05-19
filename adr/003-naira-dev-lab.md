---
status: accepted
date: 2026-05-08
---

# Naira Developer Lab

## Context

The Naira project requires a shared Kubernetes environment where developers can test workloads, integrate services, and collaborate without needing to maintain a local cluster setup. Platform Mesh, the underlying stack Naira is built on, needs a multi-node Kind cluster with Podman, CoreDNS patching, TLS certificate injection, and Traefik ingress. Replicating this reliably across macOS (Intel and Apple Silicon), Linux, and Windows WSL2 is a significant maintenance burden for a small team.

A centralized, shared lab environment reduces setup friction and ensures all team members work against a consistent, reproducible cluster state.

## Decision Drivers

- Developers need a Kubernetes environment for testing Naira workloads without complex local setup
- Platform Mesh has non-trivial infrastructure requirements (multi-node Kind, Podman, CoreDNS patches, TLS) that are hard to replicate cross-platform
- Maintaining N different local setups is not practical
- Shared services (e.g. Keycloak, OpenFGA, Traefik) should be available to all developers without duplication
- Access must be secure without exposing cluster endpoints publicly
- Future target would be to host this lab environment by SAP infrastructure (Managed K8s/Bare-metal?)

## Alternatives Considered

- **Option A: Local dev clusters per developer** — Each developer runs Kind or k3d locally.

  |     |                                                                                       |
  | --- | ------------------------------------------------------------------------------------- |
  | ✅  | Full isolation, no shared state                                                       |
  | ❌  | High setup overhead, cross-platform inconsistencies, no shared service discovery      |
  | ❌  | Each developer must maintain the stack independently                                  |
  | ❌  | Sprawl of fragmented environments makes component interoperability impossible to test |

- **Option B: Managed Kubernetes (e.g. GKE/EKS/Scaleway Kapsule)** — Use a fully managed K8s cluster.

  |     |                                                                                                             |
  | --- | ----------------------------------------------------------------------------------------------------------- |
  | ✅  | Production-grade, less ops overhead                                                                         |
  | ❌  | Platform Mesh Helm charts and automation are deeply integrated with Kind — GKE proved incompatible early on |
  | ❌  | Higher cost, overkill for a dev/integration lab, less control over cluster config                           |

- **Option C: Single Scaleway VM with Kind (chosen)** — One VM hosts a multi-node Kind cluster. Developers access it via Cloudflare WARP.

  |     |                                                                                |
  | --- | ------------------------------------------------------------------------------ |
  | ✅  | Simple to maintain, consistent state, one environment to operate, low cost     |
  | ✅  | Dedicated resources (16 vCPUs / 32 GB RAM) without impacting developer laptops |
  | ✅  | Zero Trust access via WARP — no public ports except SSH (IP-allowlisted)       |
  | ❌  | Single point of failure; no high availability                                  |
  | ❌  | Workloads assumed stateless — Kind cluster or VM failure risks data loss       |

## Decision

We provision a single Scaleway VM (`POP2-HC-16C-32G`, Debian Bookworm) running a multi-node Kind cluster with Podman. The full Platform Mesh stack is bootstrapped automatically via cloud-init on first boot. Developers access the cluster and portal securely through Cloudflare WARP using a scoped team kubeconfig.

- Infrastructure is managed via Terraform.
- State is stored in Scaleway Object Storage.
- Access control is handled through Cloudflare Zero Trust (Access policies, WARP device profiles).

## Consequences

|     |                                                                                                               |
| --- | ------------------------------------------------------------------------------------------------------------- |
| ✅  | Onboarding reduces to: install WARP → receive kubeconfig → connect. One environment instead of N.             |
| ✅  | Shared services (Keycloak, OpenFGA, PostgreSQL) available to all developers without duplication.              |
| ✅  | Bootstrap is idempotent and version-pinned — reproducible across re-provisions.                               |
| ❌  | Single VM is a single point of failure — if it goes down, all developers lose access.                         |
| ❌  | Cloud-init changes require manual re-bootstrap on existing VMs (`ignore_changes` on cloud-init in Terraform). |
| ❌  | Scaleway Object Storage has no Terraform state locking — concurrent applies must be avoided.                  |

## Related Links

[Platform Mesh Helm Charts](https://liquid-naira.atlassian.net/wiki/spaces/Naira/pages/13467651/ADR+-+003+-+Naira+Developer+Lab#:~:text=Platform%20Mesh%20Helm%20Charts)

[Cloudflare WARP client](https://liquid-naira.atlassian.net/wiki/spaces/Naira/pages/13467651/ADR+-+003+-+Naira+Developer+Lab#:~:text=Cloudflare%20WARP%20client)
