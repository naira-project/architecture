---
status: accepted
date: 2026-09-08
---

# Replace Cloudflare WARP with Tailscale

## Context

[ADR-007](./007-testbed-aws-rebuild.md) keeps the EKS API private and uses Cloudflare WARP for developer access. In the current testbed implementation, exposing cluster workloads also requires us to operate a Cloudflare tunnel alongside Route 53, ExternalDNS, cert-manager, and an ingress controller. This is more infrastructure than we need for a private development environment.

The Tailscale Kubernetes Operator can provide private access to both the Kubernetes API and cluster workloads. For workloads exposed through a Tailscale `Ingress`, it also provides a MagicDNS name and manages the TLS certificate.

Cloudflare WARP currently costs us nothing. Tailscale's free plan is limited to six users and is intended for personal use, so it does not fit our team of ten developers. The Standard plan costs $8 per user per month. At the current team size, this adds $80 per month, or $960 per year.

## Decision drivers

- Reduce the number of components required to provide private testbed access.
- Keep the EKS API and private workloads inaccessible from the public internet.
- Make private workload access, DNS, and TLS straightforward for developers and operators.
- Use access controls that map to team identities and can be managed centrally.
- Keep the ongoing cost proportionate to a ten-person development team.

## Decision

We will replace Cloudflare WARP with Tailscale for private access to the Naira testbed.

We will install the Tailscale Kubernetes Operator and use its API server proxy for Kubernetes access. Private workloads will be exposed through Tailscale `Ingress` resources using MagicDNS names under the tailnet's `ts.net` domain.

This replaces the Cloudflare tunnel and removes the need for Route 53, ExternalDNS, cert-manager, and a separate ingress controller for these private endpoints. Access will be controlled through Tailscale identities, tags, and grants.

Custom domains and public endpoints are outside the scope of this decision. They may require separate DNS, certificate, and ingress components.

## Alternatives considered

### Keep Cloudflare WARP

**Pros**

- Avoids migration work and an additional subscription cost.
- Retains the existing developer access setup.

**Cons**

- Requires a Cloudflare tunnel in addition to Route 53, ExternalDNS, cert-manager, and an ingress controller.
- Leaves more components to configure, secure, upgrade, and troubleshoot for private endpoints.

### Use Tailscale

**Pros**

- Provides private connectivity, DNS, and TLS for the testbed through the Tailscale Kubernetes Operator.
- Reduces the number of components needed for private Kubernetes API and workload access.
- Uses Tailscale identities, tags, and grants to control access.

**Cons**

- Adds an estimated cost of $80 per month for the current team.
- Requires every developer to join the tailnet and run the Tailscale client.
- Introduces reliance on Tailscale's operator, control plane, MagicDNS, and `ts.net` names.

## Consequences

- The testbed access path becomes smaller and easier to operate.
- Tailscale adds an estimated subscription cost of $80 per month for the current team, while Cloudflare WARP is currently free for us.
- Developers must join the Naira tailnet and run the Tailscale client.
- We depend on Tailscale's operator, control plane, MagicDNS, and `ts.net` names.
- Tailnet access policies and operator credentials become security-critical configuration.
- Services that need custom domains or public access will need a separate design.

## Related links

- [ADR-007: Rebuild the Naira Testbed on AWS EKS](./007-testbed-aws-rebuild.md)
- [Tailscale Kubernetes Operator](https://tailscale.com/docs/kubernetes-operator)
- [Tailscale cluster ingress](https://tailscale.com/docs/kubernetes-operator/ingress)
- [Tailscale pricing](https://tailscale.com/pricing)
