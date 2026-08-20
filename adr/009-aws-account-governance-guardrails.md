---
status: proposed
date: 2026-08-21
written-by: Hossein Salahi
---

# AWS Account Governance and Guardrails for the Naira Testbed

## Context and Problem Statement

The [testbed rebuild](https://liquid-naira.atlassian.net/wiki/x/GIDyBw) moves the shared
testbed onto AWS EKS and vCluster. It does not settle AWS account ownership, permissions,
or billing responsibility, and those must be agreed before implementation. This ADR
settles them.

It has to be settled first rather than alongside. An EKS control plane and a NAT gateway
bill by the hour whether or not anyone uses the cluster, so an account with no named owner
and no budget becomes an unbounded bill that nobody notices until the invoice.

The constraint that shapes every decision below is that **the account is handed to us by
SAP**. We do not create it, we do not choose whether it sits in an Organization, and we
should not assume Organization management access. Several things that look like design
decisions are facts to discover at handover. A governance model that assumes otherwise
will describe controls it cannot enforce.

Platform Engineer (PE) below means a member of the Naira platform engineering team holding
interactive access to the account.

## Decision Drivers

- Spend needs a named owner and an alert that reaches a PE who will act on it.
- Guardrails must be described at the strength they have. A control an account
  administrator can detach is a guardrail against accident, not against intent.
- No long-lived cloud credentials, for PEs or for CI.
- Terraform state must be locked. The current Scaleway backend has none, and CI works
  around that with a concurrency group.
- Phase 0 needs a written cost figure to be measured against.
- What we cannot enforce ourselves must become a reviewable request to SAP, not an
  assumption.

## Considered Options

### Option A - Design the account as if we held Organization privileges

Author Service Control Policies, activate cost allocation tags, stand up our own IAM
Identity Center, and decide the account structure ourselves.

Pros:

- Guardrails become unescapable. An SCP cannot be bypassed by an account administrator.
- Cost attribution works end to end without depending on anyone else.
- One coherent model, with no conditional branches to carry.

Cons:

- Requires Organization management access or delegated administrator rights we have no
  reason to expect. Every control depending on them silently does nothing if they are
  absent.
- The gap surfaces during a Phase 0 EKS run, which is the expensive place to find it.
- It documents controls we do not have, which is worse than documenting fewer controls
  accurately.

### Option B - Treat the account as handed over, and layer the controls (Chosen option)

Split controls by who can implement them: what we enforce in-account with IAM, and what we
request from SAP. Everything requiring Organization access exists as code behind a
default-off flag. The questions only handover can answer become an intake checklist, and
that checklist rather than this ADR is the precondition for Phase 0.

Pros:

- Applies unchanged whether the account is standalone or a member of an SAP Organization.
- The ask to SAP is a reviewable policy document, not a vague request for guardrails.
- The weaker in-account controls are labelled as weaker, so nobody over-trusts them.
- Rehearsable in an account we already have, because it assumes no privileges.

Cons:

- Two mechanisms for the same rule, which is more to maintain and to explain.
- The in-account guardrails stay detachable by an account administrator.
- Some decisions stay open until handover.

## Decision Outcome

Chosen option: **Option B**. It is the only option that survives contact with an account we
do not own, and it degrades honestly rather than silently.

### Ownership

A named account owner is accountable for spend against the envelope, guardrail changes, and
access grants. A deputy covers absence. Escalation is defined for a budget alert firing and
for spend running away. The SAP-side contact for billing, Organization changes, and quota
increases comes from handover.

### Account structure

Not our decision. Both branches, standalone and member of an SAP Organization, are recorded
with what each implies. The intake checklist resolves which applies. The Terraform applies
unchanged either way.

### Budget and alerting

**AWS Budgets, not CloudWatch billing alarms.** The `AWS/Billing` `EstimatedCharges` metric
is published only in `us-east-1` and only for the payer account. In a member account it is
absent, and an alarm on an absent metric sits in `INSUFFICIENT_DATA` looking like coverage.
The CloudWatch alarm exists behind a default-off flag, to be enabled only after confirming
the metric exists.

| Threshold | Basis | Purpose |
|---|---|---|
| 50% | Actual spend | Early signal |
| 80% | Actual spend | Action point |
| 100% | Actual spend | Envelope consumed |
| 100% | Forecast spend | The only alert that arrives before the money is spent |

Recipients are a shared mailbox, not an individual. A budget alerts, it does not cap.
Enforcement is the owner's responsibility.

### Guardrails

Permitted regions, permitted instance families, and required cost-attribution tags are
expressed as IAM policies applied in-account, and as Organizations policies behind a
default-off flag for the day we are granted delegated administrator.

Two properties:

- **The policies are attached to nobody by default.** They constrain nothing until attached
  to a permission set or a role. That attachment has a blast radius and belongs with the
  access decision, not as a side effect of applying a budget. The instance-family rule
  binds only the principal calling `RunInstances`, which for a managed node group or
  Karpenter is the node role.
- **The two packagings are not interchangeable.** A permissions boundary is a ceiling, so a
  boundary containing only `Deny` statements permits nothing and disables whatever it is
  attached to. One policy is deny-only, for direct attachment. The other carries an
  explicit `Allow` under the same denies, for use as a boundary.

The region deny exempts global service prefixes. Without that exemption it denies `iam:*`
everywhere and removes the ability to repair itself.

### PE access

IAM Identity Center, not IAM users. No long-lived access keys for people. Two permission
sets to start: an administrator set for the owner and deputy, a read-only set for everyone
else, broadened on demonstrated need. Which Identity Center instance, and who holds root,
are intake questions.

### CI access

GitHub OIDC into dedicated roles, never stored access keys. The current Scaleway pipeline
keeps long-lived credentials in repository secrets; that pattern does not carry over.

Two roles, because `terraform plan` executes provider code and a write-capable role
reachable from a pull request is a credential-exfiltration path:

| Role | Permissions | Trusted from |
|---|---|---|
| plan | Read-only apart from the state lock | Pull requests |
| apply | Write | A GitHub Environment, which holds the reviewer gate |

Trusted subjects are matched exactly. A wildcard subject trusts every ref in the
repository, including any branch a contributor can push.

The roles live in a Terraform root that CI validates but never plans or applies. A pipeline
that manages the credentials it authenticates with will eventually delete them; during
rehearsal it planned exactly that before the roots were separated.

### Terraform state

An S3 bucket with versioning, encryption, public access blocked, TLS enforced by bucket
policy, and superseded versions expired on a schedule. Locking uses the **S3-native
lockfile**, not a DynamoDB table. This requires Terraform 1.10 or newer; older versions
ignore the setting and run state unlocked, so the version is pinned.

### Cost envelope

| Scenario | Monthly, USD |
|---|---|
| Idle testbed | 300 to 330 |
| Normal working week | 440 to 480 |
| Proposed budget | 600 |

Unit prices from the AWS Price List. The budget is the working figure plus headroom for
ephemeral environments.

The comparison against the current single VM needs the actual Scaleway invoice figure and
is left open rather than guessed. It will not be flattering. A managed high-availability
control plane, a NAT gateway and a load balancer are real costs a single VM does not carry,
and the case for the rebuild is availability, elasticity and self-service, not cost.

### Verification

Rehearsed end to end against a sandbox member account without Organization management
access, reproducing the expected constraint. Confirmed:

- Organization policy authoring is denied.
- The billing metric is absent.
- Cost visibility and cost-allocation-tag activation are separate permissions, granted
  independently.
- State locking blocks a concurrent plan. Demonstrated, not assumed.
- Guardrail logic behaves as specified under policy simulation, including that a deny-only
  boundary permits nothing.

## Consequences

### Positive consequences

- Spend has an owner, a budget, and alerts that reach a shared mailbox before the invoice
  does.
- The controls are described at the strength they have.
- The model applies unchanged to a standalone or a member account, so handover cannot
  invalidate it.
- No long-lived credentials exist for PEs or CI.
- Terraform state is locked, retiring a risk carried by the current environment.
- Phase 0 has a cost figure to be measured against.
- What SAP must provide is a specific list rather than a general request.

### Negative consequences

- The in-account guardrails are detachable by an account administrator, and cannot be
  otherwise until we are made a delegated administrator.
- The same rule is expressed twice, in-account and at Organization level.
- The guardrail policies constrain nothing until deliberately attached, so applying the
  Terraform alone does not deliver enforcement.
- Ownership, escalation contact, alert recipient, and the agreed budget figure remain open,
  and several cannot close before handover.
- Cost attribution may produce well-tagged resources whose costs still cannot be broken
  down, because activating cost allocation tags belongs to the payer.

## Links

- [ADR-007: Rebuild the Naira Testbed on AWS EKS](007-testbed-aws-rebuild.md), which defers this decision
- [ADR-003: Naira Developer Lab](003-naira-dev-lab.md), the environment being replaced
- [Testbed rebuild](https://liquid-naira.atlassian.net/wiki/x/GIDyBw)
- Implementation, handover intake checklist and cost envelope: `aws/` in the
  `naira-testbed` repository
