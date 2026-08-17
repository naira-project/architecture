---
status: accepted
date: 2026-08-17
written-by: Marcel Frizler
---

# Rebuild the Naira Testbed on AWS EKS

## Context

[ADR 003 - Naira Developer Lab](./003-naira-dev-lab.md) introduced a shared Naira developer lab on a single Scaleway VM. The VM runs a multi-node Kind cluster and is accessed through Cloudflare WARP.

This was maily necessary because Platform Mesh depended heavily on Kind and required several cluster-specific workarounds. Platform Mesh is now being removed, so this constraint no longer applies. The Naira application and its dependencies can run on a standard Kubernetes cluster.

The current lab also has limitations:

- The VM is a single point of failure.
- Capacity is fixed and cannot scale with demand.
- A VM or Kind failure can result in data loss.
- Maintenance and upgrades requires manual work.
- The environment does not provide isolated, self-service test environments.

Also, we have to move from Scaleway to AWS as SAP will give us an AWS project.

We therefore need to reconsider the infrastructure used for the shared Naira testbed.

## Decision drivers

The new testbed should:

- Provide a reliable, managed Kubernetes control plane.
- Scale with developer and CI demand.
- Give developers and CI jobs isolated environments.
- Support automatic creation and deletion of test environments.
- Remain reasonably portable to other Kubernetes platforms.
- Provide secure, scoped access.
- Keep infrastructure cost visible and controlled.
- Avoid operating our own Kubernetes control plane.

## Alternatives considered

### Option A: Keep the single Scaleway VM with Kind

**Pros**

- It already exists.
- It is inexpensive.
- The team understands how to operate it.

**Cons**

- It remains a single point of failure.
- Capacity is fixed.
- Changes to VM leads to redeployment or restart and because of Kind, data loss will happen.
- Recovery, maintenance, and upgrades remain manual.

### Option B: Every developer has its own local Kind cluster

**Pros**

- Developers have isolated environments.
- Local clusters are useful for fast development.

**Cons**

- Every developer must maintain the stack.
- Setup differs across operating systems.
- Shared integration and interoperability testing become difficult.
- Sharing work will be difficult.

### Option C: Run self-managed Kubernetes on AWS EC2

**Pros**

- It would provide full control.
- It could be designed for high availability.

**Cons**

- We would have to operate the control plane, etcd, upgrades, backups, and recovery.
- The operational effort is too high for a development testbed.

### Option D: Create a separate EKS cluster for every environment

**Pros**

- It provides strong isolation and full Kubernetes fidelity.
- It supports tests that depend on dedicated worker nodes.

**Cons**

- Creating a cluster takes several minutes.
- It is too slow and expensive for the default CI path.

### Option E: AWS EKS with vCluster

**Pros**

- EKS provides a managed, highly available control plane.
- vCluster provides isolated Kubernetes environments on shared worker nodes.
- Environments can be created for developers, pull requests, staging, and CI runs.
- Worker capacity can scale with demand.
- Access can be limited to an individual vCluster.
- AWS-specific infrastructure can be kept behind a common platform contract.

**Cons**

- It could lead to cost more than the existing VM.
- It introduces additional components to operate.
- vClusters share worker nodes and cannot reproduce every node-level behavior.
- Some useful vCluster lifecycle and identity features require a paid licence. First we have to stick to the free version of vCluster.

## Decision

Naira will rebuild the shared testbed using AWS EKS and vCluster.

The architecture will have three layers:

1. **AWS EKS substrate**  
   EKS will provide the managed Kubernetes control plane and scalable worker capacity. Infrastructure will be managed with Terraform, using remote state with locking.
2. **Host-cluster contract**  
   The host cluster will provide common storage, ingress, DNS, certificates and AWS identity integration  
   AWS-specific details such as CSI parameters, load-balancer annotations, Route 53 configuration and IAM trust policies will remain in this layer rather than appearing in Naira workload manifests.
3. **vCluster environments**  
   vCluster will provide: - Development envs for developers or pull requests. - A long-lived stating environment for the latest release candidate. - Temporary environments for end-to-end CI runs.

Each vCluster may contain the Naira application and standalone dependencies such as OpenFGA, Keycloak and PostgreSQL.

CI will create an ephemeral vCluster, deploy the required workloads, run the tests and delete the vCluster afterward. A TLL-based cleanup process will remove abandoned environments.

The EKS API will be private. Developer access should continue to use Cloudflare WARP where practical. Developers and CI jobs will receive scoped vCluster kubeconfigs rather than host-cluster credentials.

The initial implementation will use the vCluster free tier. Because the free tier does not provide native automatic sleep, TTL cleanup or SSO, we will provide our own cleanup automation and use local platform accounts or tokens.

A paid vCluster licience can be reconsidered if SSO, audit capabilities, compliance support or native lifecycle management becomes necessary.

Tests that require DaemonSets, dedicated nodes, cross-node network policies or per-cluster load balancers may use a real temporary EKS cluster. This is an exception, not the normal CI path.

Some supporting services, such as LiteLLM and OpenMetadata, may be shared by multiple ephemeral vClusters instead of being deployed for every test run. Shared services should run either in the host cluster or in a dedicated, long-lived services vCluster and be exposed through stable internal endpoints. Direct vCluster-to-vCluster communication should only be used when host-to-vCluster access cannot meet the requirement. The final sharing model, isolation rules, and lifecycle ownership will be defined during implementation.

The migration will happen gradually:

- Establish EKS and the host-cluster services.
- Install vCluster and migrate Naira.
- Move end-to-end testing to ephemeral vClusters.
- Move staging and retire the Scaleway VM.

When this ADR is accepted, ADR-003 becomes deprecated and is superseded by ADR-009. The existing Scaleway environment will only be removed after the replacement has been proven stable.

Implementation details are maintained in the linked testbed rebuild design document.

## Consequences

### Positive consequences

- The Kubernetes control plane becomes managed and highly available.
- The testbed no longer depends on one VM.
- Capacity can scale with demand.
- Developers and CI jobs receive isolated environments.
- Test environments become reproducible and disposable.
- AWS-specific configuration is kept separate from Naira workloads.
- Naira remains portable to other Kubernetes platforms.
- Scoped access reduces the need to distribute host-cluster credentials.

### Negative consequences

- AWS EKS will cost more than the existing VM.
- The team must operate EKS integrations, vCluster, GitOps, and cleanup automation.
- vClusters do not provide complete node-level isolation.
- The free vCluster tier requires us to build parts of the lifecycle and login solution ourselves.
- AWS account ownership, permissions, and billing responsibility must be agreed before implementation.

## Related links

- [ADR-003: Naira Developer Lab](./003-naira-dev-lab.md)
- [Naira Playground repository](https://github.com/naira-project/naira-playground)
- [Product issue 55](https://github.com/naira-project/product/issues/55)
