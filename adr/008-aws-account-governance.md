---
status: proposed
date: 2026-08-20
written-by: Hossein Salahi
---

# AWS Account Governance for the Naira Testbed

## Context and Problem Statement

[ADR-007](007-testbed-aws-rebuild.md) rebuilds the shared testbed on AWS EKS and vCluster, and closes by
naming what it does not settle: "AWS account ownership, permissions, and billing responsibility must be
agreed before implementation." This ADR settles that.

It has to be settled first rather than alongside. An EKS control plane and a NAT gateway bill by the hour
whether or not anyone uses the cluster, so an account with no named owner and no budget is the ordinary way
a testbed becomes an unbounded bill, and nobody notices until the invoice.

The constraint that shapes every decision below is that **the account is handed to us by SAP**. We do not
create it, we do not choose whether it sits in an Organization, and we should not assume Organization
management access. Several things that look like design decisions are therefore facts to discover at
handover, and a governance model that assumes otherwise will describe controls it cannot actually enforce.

## Decision Drivers

- Spend needs a named owner and an alert that reaches a human who will act on it.
- Guardrails must be described honestly. A control an account administrator can detach is a guardrail
  against accident, not against intent.
- No long-lived cloud credentials, for humans or for CI.
- Terraform state must be locked. The current Scaleway backend has none, and CI works around that with a
  concurrency group.
- Phase 0 needs a written cost figure to be measured against.
- What we cannot enforce ourselves must become a reviewable request to SAP, not an assumption.

## Considered Options

### Option A - Design the account as if it were ours

Author Service Control Policies, activate cost allocation tags, stand up our own IAM Identity Center, and
decide the account structure ourselves.

Pros:

- Guardrails become genuinely unescapable, since an SCP cannot be bypassed by an account administrator.
- Cost attribution works end to end without depending on anyone else.
- One coherent model, with no conditional branches to carry.

Cons:

- It requires Organization management access or delegated administrator rights that we have no reason to
  expect, and every control depending on them silently does nothing if they are absent.
- Applying it discovers the gap during a Phase 0 EKS run, which is the expensive place to find out.
- It would document controls we do not have, which is worse than documenting fewer controls accurately.

### Option B - Treat the account as handed over, and layer the controls (Chosen option)

Split controls by who can implement them: what we can enforce in-account with IAM, and what has to be
requested from SAP. Everything requiring Organization access exists as code behind a flag that is off by
default. Record the questions that only handover can answer as an intake checklist, and treat that
checklist rather than this ADR as the real precondition for Phase 0.

Pros:

- Works on the first apply whether the account is standalone or a member of an SAP Organization.
- The ask we make of SAP is a reviewable policy document rather than a vague request for guardrails.
- The weaker in-account controls are labelled as weaker, so nobody over-trusts them.
- Rehearsable in an account we already have, because it does not assume privileges.

Cons:

- Two mechanisms for the same rule, which is more to maintain and to explain.
- The in-account guardrails are detachable by an account administrator and always will be.
- Some decisions stay open until handover, so this ADR does not close everything it would like to.

## Decision Outcome

Chosen option: **Option B**, because it is the only one that survives contact with an account we do not
own, and because it degrades honestly rather than silently.

### Ownership

A named account owner is accountable for spend against the envelope, guardrail changes, and access grants.
A deputy covers absence: a single owner with no deputy is the single point of failure this epic exists to
remove. Escalation is defined for the failure modes that actually occur, chiefly a budget alert firing and
spend running away. The SAP-side contact for billing, Organization changes, and quota increases comes from
handover.

### Account structure

Not our decision. Both branches, standalone and member of an SAP Organization, are recorded with what each
implies, and the intake checklist resolves which one applies. The Terraform applies unchanged either way.

### Budget and alerting

**AWS Budgets, not CloudWatch billing alarms.** The `AWS/Billing` `EstimatedCharges` metric is published
only in `us-east-1` and only for the payer account. In a member account it is absent, and an alarm on an
absent metric sits in `INSUFFICIENT_DATA` looking like coverage while providing none. The CloudWatch alarm
exists behind a flag that is off by default, to be enabled only after confirming the metric exists.

Alerts fire at 50, 80 and 100 percent of actual spend and at 100 percent of forecast spend. The forecast
alert is the one with warning value; actual-100% arrives after the money is spent. Recipients should be a
shared address, because an alert routed to one person's inbox fails when that person is on leave. A budget
does not stop spending, it sends mail, which is why the ownership section above is load-bearing.

### Guardrails

Permitted regions, permitted instance families, and required cost-attribution tags are expressed as IAM
policies we can apply in-account, and as Organizations policies behind a default-off flag for the day we
are granted delegated administrator.

Two properties of this are easy to misread and are therefore stated explicitly:

- **The policies are attached to nobody by default.** They constrain nothing until attached to a permission
  set or a role, which has a blast radius and belongs with the access decision rather than happening as a
  side effect of applying a budget. The instance-family rule only bites on the principal that calls
  `RunInstances`, which for a managed node group or Karpenter is the node role, not a human.
- **The guardrail ships in two packagings that are not interchangeable.** A permissions boundary is a
  ceiling, so a boundary containing only `Deny` statements permits nothing and disables whatever it is
  attached to. One policy is deny-only, for attaching directly; the other carries an explicit `Allow` under
  the same denies, for use as a boundary.

The region deny exempts global service prefixes. Without that exemption it denies `iam:*` everywhere and
removes the ability to repair itself.

### Human access

IAM Identity Center, not IAM users: no long-lived access keys for people. Two permission sets to start, an
administrator set for the owner and deputy and a read-only set for everyone else, broadened on demonstrated
need. Which Identity Center instance, and who holds root, are intake questions.

### CI access

GitHub OIDC into dedicated roles, never stored access keys. The current Scaleway pipeline keeps long-lived
credentials in repository secrets, and that pattern does not carry over.

Two roles rather than one, because `terraform plan` executes provider code and a write-capable role
reachable from a pull request is a credential-exfiltration path:

- a plan role, read-only apart from the state lock, trusted from pull requests;
- an apply role, trusted only from a GitHub Environment, which is where the reviewer gate lives.

Trusted subjects are matched exactly. A wildcard subject would trust every ref in the repository, including
any branch a contributor can push.

The roles live in a Terraform root that CI validates but never plans or applies. A pipeline that manages
the credentials it authenticates with will eventually delete them, and during rehearsal it planned exactly
that before the roots were separated.

### Terraform state

An S3 bucket with versioning, encryption, public access blocked, TLS enforced by bucket policy, and
superseded versions expired on a schedule. Locking uses the **S3-native lockfile**, not a DynamoDB table,
which requires Terraform 1.10 or newer; older versions ignore the setting and run state unlocked, so the
version is pinned. This retires the missing-locking risk the Scaleway backend carries rather than papering
over it with a CI concurrency group.

### Cost envelope

Roughly 300 to 330 USD per month for an idle testbed and 440 to 480 USD for a normal working week, from
unit prices taken from the AWS Price List. A monthly budget of 600 USD is proposed for the target account,
being the working figure plus headroom for ephemeral environments.

The comparison against the current single VM is deliberately not filled in: it needs the actual Scaleway
invoice figure, and guessing it would be worse than leaving it open. It will not be flattering. A managed
high-availability control plane, a NAT gateway and a load balancer are real costs a single VM does not
carry, and per ADR-007 the case for the rebuild is availability, elasticity and self-service, not cost.

### Validation

The model was rehearsed end to end against a sandbox member account that reproduces the expected
constraint, being itself a member account without Organization management access. The rehearsal confirmed
that Organization policy authoring is denied, that the billing metric is absent, and that cost visibility
and cost-allocation-tag activation are separate permissions granted independently, which is the finding
that turned a single intake item into two. State locking was demonstrated by blocking a concurrent plan
rather than assumed. Guardrail logic was verified by policy simulation, including empirical confirmation
that a deny-only boundary permits nothing.

## Consequences

### Positive consequences

- Spend has an owner, a ceiling, and alerts that reach a person before the invoice does.
- The controls are described at the strength they actually have.
- The model applies unchanged to a standalone or a member account, so handover cannot invalidate it.
- No long-lived credentials exist for humans or CI.
- Terraform state is locked, retiring a risk carried by the current environment.
- Phase 0 has a cost figure to be measured against.
- What SAP must provide is a specific list rather than a general request.

### Negative consequences

- The in-account guardrails are detachable by an account administrator, and cannot be otherwise until we
  are made a delegated administrator.
- The same rule is expressed twice, in-account and at Organization level.
- The guardrail policies constrain nothing until deliberately attached, so applying the Terraform alone
  does not deliver enforcement.
- Ownership, escalation contact, alert recipient, and the agreed budget figure remain open, and several of
  them cannot close before handover.
- Cost attribution may produce well-tagged resources whose costs still cannot be broken down, because
  activating cost allocation tags belongs to the payer.

## Links

- [ADR-007: Rebuild the Naira Testbed on AWS EKS](007-testbed-aws-rebuild.md), which defers this decision
- [ADR-003: Naira Developer Lab](003-naira-dev-lab.md), the environment being replaced
- [Product epic 55](https://github.com/naira-project/product/issues/55)
- [Product issue 56](https://github.com/naira-project/product/issues/56), including the acceptance criteria this ADR does not by itself satisfy
- [Product issue 58](https://github.com/naira-project/product/issues/58), Phase 0, blocked by this
- Implementation, handover intake checklist and cost envelope: `aws/` in the `naira-testbed` repository
