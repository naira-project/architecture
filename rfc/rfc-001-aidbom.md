# RFC 0001: Continuous Agent Composition Tracking

| Field         | Value                                                    |
|---------------|----------------------------------------------------------|
| RFC           | 0001                                                     |
| Title         | Continuous Agent Composition Tracking                    |
| Author(s)     | @mkorbi                             |
| Target Milestone| M5 - Advanced & Plugind                           |
| Status        | Draft                                                    |
| Type          | Feature                  |
| Created       | 2026-05-19                                               |

## Abstract

This RFC proposes a new feature under Naira that
provides **continuous, versioned tracking of AI agent composition** over time.
The feature (mini-project) is not another AI or ML Bill of Materials generator. 
Some projects in this regards exists already but just for specific elements e.g. model + data set or MCP. 
In a first step, it is a **server and toolchain that ingests AI/ML/MCP-BOMs from any
compatible scanner, stores them versioned, diffs them across time and
deployments, correlates them against advisory feeds, and exposes a fleet
view**. 

The work is structured so that a future extension can add VEX-style
attestations for agent-specific risks (e.g. prompt injection, tool poisoning,
toxic flows). 

In a second step, this feature within Naira might be a holistic solution to create 
own "AI"BOMs, as we are anyhow connect all lose ends. 
Additionally, current project are in my point of view, 2-3 years behind the state 
of art. The future of AI is, also, Agentic, dynamic, self-managed.

Mapping the resulting record to EU AI Act Annex IV technical
documentation is explicitly out of scope for this RFC but is a downstream use
case the data model must not preclude.

## Motivation

Existing tooling in the AI supply-chain space, as of May 2026, covers different 
layers in various qualities.

**Well-covered: point-in-time scanning.** Tools such as Snyk AI-BOM, Cisco's
open-source `aibom`, Trusera/ai-bom, the OWASP AIBOM Generator, and several
commercial offerings can scan a codebase or environment and emit a CycloneDX
or SPDX document describing the AI components present: models, agents,
tools, MCP servers and clients, skills, datasets, guardrails. Coverage,
detection quality, and language support vary, but the category is healthy
and there is no need for another scanner. However, those are, and I have to 
heavy emphasize it, **point in time** snapshots, not build for the dynamics of agentic.

**Well-covered: MCP server security scanning and runtime guardrails.** Tools
including Invariant Labs / Snyk `mcp-scan`, Cisco MCP Scanner, Enkrypt,
Straiker, AQtive Guard, and others audit individual MCP servers for known
attack patterns (tool poisoning, prompt injection, toxic flows,
over-privilege) and, in some cases, proxy live MCP traffic. The category is
crowded.

**Poorly covered: the operational layer.** Despite this density of scanners,
there is no widely adopted open-source system that:

1. Ingests AIBOMs from heterogeneous sources (multiple scanners, multiple
   agent frameworks: Claude Code, Cursor, Windsurf, LangGraph, CrewAI, n8n,
   custom in-house) into a single store.
2. Versions them, so that the composition of a given agent or fleet at any
   past point in time can be reconstructed.
3. Computes and exposes **diffs** between versions e.g. what skills were added,
   which MCP servers' tool schemas changed, which library pins moved.
4. Correlates the inventory against vulnerability and risk advisory feeds
   (OSV.dev for traditional libraries; emerging advisory sources for MCP
   servers and skills).
5. Exposes this as a fleet view suitable for security, compliance, and
   platform-engineering teams.

The closest analogue in the software-supply-chain world is OWASP
Dependency-Track, which sits downstream of SBOM generators (Syft, cdxgen,
language-specific generators) and provides the continuous, versioned,
diff-aware view that those generators do not. No such system exists for
agent composition today.

This gap is relevant because agent composition is **substantially more dynamic
than traditional dependency composition**. Skills can be added or removed
without a rebuild and sometimes be executed and sometimes not; MCP server tool 
schemas can change between calls without
any version bump; system prompts and guardrail configurations are
edited in place. A point-in-time scan answers "what does this agent look
like right now?"; it cannot answer "what changed since the audit two weeks
ago, and did any of those changes introduce a known risk?" The latter
question is what security and compliance teams actually need to answer, and
it is the question the EU AI Act and similar regimes will increasingly
require organizations to answer on demand.

## Goals

1. **Be a sink, in a first run** Ingest AIBOMs produced by any
   compliant scanner.
2. **Version everything.** Every ingested BOM is immutable and addressable.
   The composition of any tracked agent or fleet at any point in time MUST
   be reconstructible.
3. **Diff is a first-class output.** Between any two BOM versions of the
   same tracked entity, the system produces a structured, machine-readable
   diff covering added/removed/changed components, with particular fidelity
   for skills (content hash changes), MCP servers (tool schema deltas), and
   model references (version pins).
4. **Correlate, don't decide.** Cross-reference inventoried components
   against advisory feeds and surface findings. Whether a finding is
   actionable is left to humans (or, in v2, to attestations —> see Future
   Work).
5. **Self-hostable and telemetry-free by default.** The reference
   implementation MUST be runnable in an air-gapped environment without
   calling out to any vendor.
6. (future work, optional) **Have an AIBOM as core feature** Naira comes with 
    all information required to create AIBOMS at any point in time something happens 
    in the system. 

## Non-Goals

The following are explicitly out of scope for this RFC. They are listed not
to forbid them forever but to keep the v1 charter tight and avoid
duplicating existing work.

- **Not a runtime guardrail or proxy.** Live MCP traffic interception,
  policy enforcement at tool-call time, and prompt-injection blocking are
  the domain of existing runtime tools and are not duplicated here.
- **Not an EU AI Act compliance product.** The data model is designed so
  that Annex IV mappings can be built on top, but producing the binder
  itself is out of scope for v1.
- **Not yet another MCP server registry.** Tracking MCP server identity is
  in scope; curating a trusted registry is not.

## Specification

### Terminology

- **Component**: an item in a AI BOM (a model, an agent, an MCP
  server, a skill, a library, etc.).
- **Subject**: the thing the BOM describes, typically an agent, an agent
  fleet, a project, or a deployment.
- **BOM Version**: an immutable, content-addressed document
  ingested for a given Subject at a given point in time.
- **Diff**: a structured comparison between two BOM Versions of the same
  Subject.
- **Finding**: a correlation between a Component (in some BOM Version) and
  an external advisory or risk record.

### High-level architecture

```
  ┌──────────────┐    ┌──────────────┐    ┌──────────────┐
  │  Snyk aibom  │    │ Cisco aibom  │    │  mcp-scan    │  ... and others
  └──────┬───────┘    └──────┬───────┘    └──────┬───────┘
         │                   │                   │
         │   e.g. CycloneDX 1.6+ JSON (one BOM per scan)
         │                   │                   │
         └─────────┬─────────┴─────────┬─────────┘
                   │                   │
                   ▼                   ▼
            ┌────────────────────────────────┐
            │      Ingestion API (REST)      │
            └──────────────┬─────────────────┘
                           │
                  ┌────────▼─────────┐
                  │  BOM Version     │  immutable, content-addressed
                  │  Store           │
                  └────────┬─────────┘
                           │
            ┌──────────────┴───────────────┐
            │                              │
   ┌────────▼────────┐         ┌───────────▼────────────┐
   │ Diff Engine     │         │ Correlation Engine     │
   │ (subject-scoped)│         │ (advisory feeds)       │
   └────────┬────────┘         └───────────┬────────────┘
            │                              │
            └──────────────┬───────────────┘
                           │
                  ┌────────▼─────────┐
                  │  Query API +     │
                  │  Fleet UI        │
                  └──────────────────┘
```

### Ingestion

- Accept CycloneDX 1.6, SPDX and JSON.
- Each ingested document is stored verbatim and addressed by the SHA-256 of
  its canonical form.
- Ingestion is associated with a Subject identifier supplied by the client.
  Subject identifiers are opaque strings chosen by the operator (e.g.
  `prod/customer-support-agent`, `dev/claude-code/alice`).
- Optional fields on ingestion: scanner name and version, environment label,
  signed attestation envelope.

### Diff semantics

Diffs are computed per Subject between two BOM Versions. The diff format
extends standard SBOM-diff approaches with agent-specific awareness:

- **Library components**: standard added / removed / version-changed.
- **Model components**: track changes to model identifier, version, provider,
  and adapter chain (LoRAs etc.) as distinct fields.
- **MCP server components**: track changes to server identity (URL or
  command), declared version, and the hash of the declared
  tool schema. A tool-schema change without a version bump is a flag worth
  surfacing.
- **Skill components**: track content hash of the skill body and metadata
  separately. A typo fix and a tool-use-instruction rewrite produce
  different diff shapes.
- **Agent components**: track changes to system prompt hash, declared
  permissions, and the set of attached skills/MCP servers.

The diff format is itself emitted as JSON. A canonical schema is part of
the deliverable for this RFC.

### Correlation

v1 correlates components against:

- **OSV.dev** for traditional library components (already covered by every
  SBOM tool; included here for completeness).
- **A pluggable advisory source interface** for MCP-server and skill-level
  risks. v1 ships with Nairas natural capability to consume any other input source.

### Deployment

- Single-binary reference implementation .
- Backing store: SQLite for single-node; Postgres for multi-node.
- No outbound network calls in the default configuration. Advisory feeds
  are pull-on-demand with explicit operator consent.

## Rationale and Alternatives

**Why a separate Naira feature rather than a feature in an existing scanner?**
Scanners are detection engines; their value is in their detection quality
and language coverage. Bolting a versioned store and diff engine onto each
of them duplicates work and fragments the operator experience. The
Dependency-Track precedent, a single downstream system fed by many SBOM
generators, is well established and is the right shape for this layer and fits 
100% to how Naira is designed.

**Why not extend OWASP Dependency-Track directly?** Dependency-Track is
deeply oriented around library-style components and CVE-style findings. The
diff semantics needed for skills, MCP tool schemas, and system prompts
don't map cleanly onto its current data model, and the maintainers have
their hands full with the SBOM workload. A dialogue with that project
about eventual convergence is worth having, but the v1 work is better done
as a focused sub-project.

**Why not define a new "AgentBOM" format instead of riding CycloneDX 1.6+?**
The ecosystem has already converged. CycloneDX 1.6 has component types
covering models and ML assets; 1.7 and the OWASP AIBOM working group are
filling remaining gaps. Inventing a parallel format would split the
ecosystem and slow adoption. Where CycloneDX is genuinely insufficient (for
example, the diff format itself, or the v2 attestation schema), we should extend
via the CycloneDX property taxonomy or propose upstream changes, we do
not fork.

## Drawbacks

- **Ecosystem dependency.** The value of this sub-project is bounded by the
  quality of upstream AIBOM scanners. If they emit inconsistent or
  low-fidelity documents, downstream diffs and correlations inherit those
  problems. Mitigation: define a minimum BOM-quality profile the system
  expects on ingestion, with clear errors when it isn't met.
- **Skill and prompt diffing is genuinely hard.** Distinguishing a benign
  edit from a security-relevant change in a skill or system prompt is an
  open problem. v1 reports the diff; it does not classify it. Operators
  may find this insufficient. v2 attestation work is the planned answer.
- **Advisory feed sparsity.** Library CVEs are abundant; MCP-server and
  skill advisories are still rare and not consistently formatted. The
  pluggable adapter interface absorbs this but doesn't solve it.
- **Competitive landscape risk.** Commercial vendors (Snyk, Cisco, Enkrypt,
  Straiker) may extend their existing platforms into this layer. The
  open-source, self-hostable, telemetry-free positioning is the
  differentiation; if that erodes, the project's reason to exist erodes
  with it.

## Future Work (out of scope for this RFC)

- **v2: Agent-VEX.** A VEX-like attestation format for agent-specific
  risks, allowing operators to record "this finding exists in our BOM but
  is mitigated by guardrail X" with cryptographic provenance.
- **v3: Annex IV / NIST AI RMF mappings.** Tooling that takes a versioned
  Subject and produces a regulator-ready technical documentation package.
- **v4: AIBOM Generator.** Create, based on Nairas sum of all dependencies at any action a AIBOM. 

We might consider to think/work with other communities on this. E.g. it might make no sense to 
create entire BOMs, but having a foundation and addtions. This reduces the size of stored 
information and let focus on actual changes.

## References

- CycloneDX specification: https://github.com/CycloneDX/specification
- CycloneDX ML-BOM: https://cyclonedx.org/capabilities/mlbom/
- OWASP AIBOM Project: https://owasp.org/www-project-aibom/
- OWASP Dependency-Track: https://dependencytrack.org/
- Cisco AI Defense `aibom`: https://github.com/cisco-ai-defense/aibom
- Invariant Labs / Snyk `mcp-scan`: https://github.com/snyk/agent-scan
- OSV.dev: https://osv.dev/
- EU AI Act, Article 11 and Annex IV (technical documentation requirements
  for high-risk AI systems; applicable from 2 August 2026).

## Changelog

- 2026-05-19: Initial draft.