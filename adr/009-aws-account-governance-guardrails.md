---
status: proposed
date: 2026-08-21
written-by: Hossein Salahi
---

# ADR-009: AWS Account Governance & Guardrails

## Context
[Nira testbed rebuild](https://liquid-naira.atlassian.net/wiki/x/GIDyBw) rebuilds the shared testbed on AWS EKS and vCluster, this ADR provides both rails and guardrails by setting AWS account ownership, permissions, and billing responsibility that must be agreed before implementation.

The constraint that shapes every decision below is that the account is handed to us by SAP. We do not create it, we do not choose whether it sits in an Organization, and we should not assume Organization management access. Several things that look like design decisions are therefore facts to discover at handover, and a governance model that assumes otherwise will describe controls it cannot actually enforce.

## Decision Drivers

- Spend needs a named owner and an alert that reaches a PE/SRE who will act on it.
- Guardrails must be described clearly.
- No long-lived cloud credentials, for both PE or for CI.
- Terraform state must be locked. 
- Phase 0 needs a written cost figure to be measured against.

What we cannot enforce ourselves must become a reviewable request to SAP, not an assumption.

## Alternatives considered

### Option A - Design the account as if it we have ORG level privileges
Author Service Control Policies, activate cost allocation tags, stand up our own IAM Identity Center, and decide the account structure ourselves.

#### Pros
- Guardrails become genuinely unescapable, since an SCP cannot be bypassed by an account administrator.
- Cost attribution works end to end without depending on anyone else.
- One coherent model, with no conditional branches to carry.

#### Cons
- It requires organization management access or delegated administrator rights that we have no reason to expect.
- It would require more efforts from PE to manage it.

### Option B - Treat the account as handed over, and layer the controls (Chosen option)
Split controls by who can implement them. What we can enforce in-account with IAM, and what has to be requested from SAP.

#### Pros
- Works on the first apply whether the account is standalone or a member of an SAP Organization.
- Offloads PE team from ORG level controls and operations
- Testable in an account we already have, because it does not assume privileges.

#### Cons
- Two mechanisms for the same rule, which is more to maintain and to explain.
- The in-account guardrails stay detachable by an account administrator.
- Some decisions stay open until handover.

## Decision

**Chosen option**: Option B, because it is the only one that survives contact with an account we do not own, and because it degrades honestly rather than silently.

### Ownership

A named account owner is accountable for spend against the envelope, guardrail changes, and access grants. Escalation is defined for the failure modes that actually occur, chiefly a budget alert firing and spend running away. The SAP-side contact for billing, Organization changes, and quota increases comes from handover.

### Budget and alerting

AWS Budgets, not CloudWatch billing alarms. The AWS/Billing EstimatedCharges metric is published only in us-east-1 and only for the payer account. In a member account it is absent, and an alarm on an absent metric sits in INSUFFICIENT_DATA looking like coverage while providing none. The CloudWatch alarm exists behind a flag that is off by default, to be enabled only after confirming the metric exists.

Alerts fire at 50, 80 and 100 percent of actual spend and at 100 percent of forecast spend. The forecast alert is the one with warning value; actual-100% arrives after the money is spent. Recipients should be a shared address, because an alert routed to one person's inbox fails when that person is on leave. A budget does not stop spending, it sends mail, which is why the ownership section above is load-bearing.

### Guardrails

Permitted regions, permitted instance families, and required cost-attribution tags are expressed as IAM policies we can apply in-account.

#### Two properties:

- The policies are attached to nobody by default. They constrain nothing until attached to a permission set or a role. That attachment has a blast radius and belongs with the access decision, not as a side effect of applying a budget. The instance-family rule binds only the principal calling RunInstances, which for a managed node group or Karpenter is the node role.

- The two packagings are not interchangeable. A permissions boundary is a ceiling, so a boundary containing only Deny statements permits nothing and disables whatever it is attached to. One policy is deny-only, for direct attachment. The other carries an explicit Allow under the same denies, for use as a boundary.

### CI 

GitHub OIDC into dedicated roles, never stored access keys. 
Two roles, because terraform plan executes provider code and a write-capable role reachable from a pull request is a credential-exfiltration path:

- a plan role, read-only apart from the state lock, trusted from pull requests;
- an apply role, trusted only from a GitHub Environment, which is where the reviewer gate lives.

Trusted subjects are matched exactly. A wildcard subject would trust every ref in the repository, including any branch a contributor can push.

### Terraform State
An S3 bucket with versioning, encryption, public access blocked.

### Verification
The model was tested end to end against a sandbox account that reproduces the expected constraint, being itself a member account without organization management access. The test confirmed that Organization policy authoring is denied, that the billing metric is absent, and that cost visibility and cost-allocation-tag activation are separate permissions granted independently. 

## Consequences

### Positive  

- Spend has an owner, a ceiling/quota, and alerts that reach a person/namespace before the invoice does.
- No long-lived credentials exist for user or CI.
- Terraform state is locked, retiring a risk carried by the current environment.
- Phase 0 has a cost figure to be measured against.

### Negative  

- The same rule is expressed twice, in-account and most probably at organization level.
- Ownership, escalation contact, alert recipient, and the agreed budget figure remain open, and several cannot close before handover.

## Links

- [ADR-007: Rebuild the Naira Testbed on AWS EKS](007-testbed-aws-rebuild.md), which defers this decision
- [ADR-003: Naira Developer Lab](003-naira-dev-lab.md), the environment being replaced
