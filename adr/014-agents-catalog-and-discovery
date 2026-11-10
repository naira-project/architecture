# Agent Catalog and Discovery/Ingest

## Context

How does the catalog represent an AI agent, and how does it relate the same agent when several sources report it? 

We don't want to answer in this ADR "How do we find out it's an agent?"; a short outlook will be added to the end of the ADR.

Naira has no agent concept yet. The shared kinds in `plugins/pkg/pluginapi/schema.go` cover `application`, `deployment`, `service`, `model`, `inference_endpoint`, `mcp_server`, `mcp_tool` and others, but no agent. That's because agents are not simple to identify, short living and distributed or stronlgy isolated.

But there are several ways to find agents, some plugin ideas are therefore: a Naira label on Kubernetes workloads (that would be a strong opinionation that we have to set), A2A Agent Cards, kagent and Kagenti resources, cloud agent registries and the Agent Server integration ([naira#309](https://github.com/naira-project/naira/issues/309)). 

A problem is also that one agent is usually visible in more than one source. Example: A kagent agent runs as a Deployment, may publish an Agent Card and may be listed in a registry.

No source-independent agent identifier exists. The Agent Card URL is the closest candidate, but adoption is low: a survey in July 2026 found 65 public cards across 22,341 hosts, 10 of them structurally valid ([API Evangelist](https://apievangelist.com/2026/07/29/most-published-agent-cards-are-not-actually-a2a/)). Other options are the OASF records and registry ARNs are scoped to their own system.

## Decision drivers

- Naira is a lens over sources, not a registry. It must not mint agent identities that owners then have to maintain.
- Show relationships: agent to workload, model and MCP server, and agent to the same agent in another source.
- Stable identity: links and relations survive when an agent gains a card or a hostname changes.
- Plugins stay independent and replaceable.

## Alternatives considered

### Option A: `agent` kind with source-scoped identity, correlated through shared anchor nodes

Each plugin emits `agent` nodes at its own source-scoped path. Plugins link an agent to anchors that have a natural shared identity: the `deployment` it runs as, and a new `agent_card` node whose path is the normalized card URL. Agents that share an anchor are related through it.

[Agent Cards from the A2A](https://agent2agent.info/docs/concepts/agentcard/).

PRO

- No core change. Uses node, relation and claim semantics as built.
- Agent identity never changes when a card appears, moves or disappears.
- A wrong or spoofed card reference is a visible relation attributed to one plugin.
- Works for agents without a card: label and kagent sources meet at the same `deployment` node.
- The card is modelled truthfully: the URL identifies the card document, and several plugins can add claims to it.

CON

- The same agent appears once per source in lists and counts until the UI groups agents by shared anchor.
- Agents with no shared anchor (a registry record without endpoint or card) stay unrelated.
- Plugins must agree on anchor paths: URL normalization and the `deployment` path scheme.

### Option B: `agent` kind with identity derived from the Agent Card URL

Agents with a card use the normalized card URL as path, so claims from all plugins merge into one node. Agents without a card use a source-scoped path.

PRO

- One node per agent for sources that know the card.
- Uses existing claim merging.

CON

- Identity changes when an agent gains a card or its host is renamed; the node is replaced and inbound relations break.
- Anyone who can set the card reference on a workload merges it into another team's agent.
- Covers few agents, given card adoption. The rest still duplicate, with no rule to relate them.
- Two identity schemes inside one kind.

### Option C: Alias and merge mechanism in the catalog core

Nodes carry a list of external identifiers. The core merges nodes that share one.

PRO

- One node per agent across all sources that share any identifier.
- Reusable for other kinds, for example models seen by several registries.

CON

- Merge conflicts, unmerge on withdrawal and transitive merges must be defined and tested.
- Same spoofing exposure as Option B, now in the core.
- Blocks every agent plugin until the core work is done.

### Option D: No agent kind; agent properties on `deployment` or `application` nodes

PRO

- No new kind. Smallest change for the Kubernetes label source.

CON

- Registry, card and workflow-platform agents have no workload node to attach to.
- An agent and its workload have different owners, versions and lifecycles in several sources.
- "List all agents" becomes a property filter across kinds.

## Decision

Option A.

It provides the cleanest approach, while gives us flexibility. At the same time we build on a "standard" schema, also the adoption is weak yet, I think it is better to use a schema that is already defined for agents than to come up with an own.

### Kinds

- `agent`: an agent as declared by one source. Path: `<PATH_PREFIX>/<source-native identifier>`, following the convention of existing plugins.
- `agent_card`: an A2A Agent Card document. Path: the normalized card URL without scheme. No `PATH_PREFIX`, so every plugin that knows the URL emits the same identity.

Both constants are added to `plugins/pkg/pluginapi/schema.go`.

### Relations

| Relation | From | To | Emitted when |
| --- | --- | --- | --- |
| `runs_as` | `agent` | `deployment` | the source knows the workload |
| `described_by` | `agent` | `agent_card` | the source carries a card URL |
| `uses_model` (existing) | `agent` | `model` | the source declares the model |
| `uses_mcp_server` | `agent` | `mcp_server` | the source declares the MCP server |

### Rules

- An agent is declared, not inferred. A plugin emits `agent` only when its source explicitly names an agent: a label, a card, an agent resource or a registry record.
- No plugin and no core logic merges or relates agents by name.
- Two `agent` nodes are treated as the same agent only when they share a `deployment` or an `agent_card` anchor. The catalog stores them as separate nodes. Grouping them is a presentation concern and gets a DDR.
- Card URL normalization: HTTPS only, lowercase host, default port removed, query and fragment removed, no trailing slash. One shared helper in `plugins/internal/`.
- Shared properties on `agent`, all optional except `name`: `name`, `provider`, `version`, `framework`, `endpoint`. Source-specific properties stay in the plugin's claim.
- A plugin that emits `described_by` does not need to fetch the card. Only the A2A plugin adds content claims to `agent_card`.

### Not decided here

- Whether client tools observed at an AI gateway are `agent` nodes. Until decided they are not.
- Keeping last known data when one target fails inside a successful run. This affects several plugins and needs its own decision.
- Skills and agent-to-agent relations.

## Consequences

### Positive

- The agent plugins can be built in parallel against the current core.
- rAIner and Peter can navigate from an agent to workload, card, models and MCP servers, and from an MCP server or model to every agent that depends on it.
- A false card claim is visible and attributable instead of destructive.
- The decision is additive. Kinds and relations can be extended without migration.

### Negative

- Until the grouping (projects/products) is implemented, lists and counts show one agent per source.
- In-cluster agents can have a service URL and an ingress URL for the same card. They yield two `agent_card` nodes unless the owner declares one.
- Registry records without endpoint, workload or card stay isolated.

### Neutral

- No lock-in to A2A. The card is an optional anchor; OASF and registry formats map onto the shared properties.
- If a core alias mechanism is wanted later for several kinds, Option C can be proposed as an RFC on top of this model.

# Outlook - How to find agents?

I wrote a couple of options up on https://liquid-naira.atlassian.net/wiki/spaces/Naira/pages/223215622/Agent+Catalog.
In short we have options of:
- Registries: AWS, Azure Google all introduced them
- Open source registries, like AGNTCY or agentregistry are options
- Read from runtimes e.g. agentsandbox, kagent.
- Read from framework/control plan servers: LangChain Server (proposed), GooseTown
- Read from Telemetry Data: OTEL Gen AI spans with gen_ai.agent.id, .name
