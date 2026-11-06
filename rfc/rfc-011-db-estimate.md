# RFC-011: Database size estimate and limits for plugin output

| Field | Value |
|---|---|
| RFC | 011 |
| Title | Database size estimate and limits for plugin output |
| Status | Draft, open for comments |
| Scope | First deployment and early scaling. This is not a long-term or hyperscale estimate. |
| Origin | Inspired by [@akavel-reply's comment](https://github.com/naira-project/naira/pull/247#discussion_r4249078169) requesting a database size estimation. |

Final estimates are rounded **up** to one significant digit (for example 4.7 x 10^7 becomes 5 x 10^7), so they stay on the safe side. They show orders of magnitude, not precise values.

## Goal

Separate two questions for the first deployment and early scaling:

- **Capacity planning:** estimate database size from the schema, row counts and representative payloads.
- **Plugin guardrails:** make it harder for one plugin run to send an unreasonable amount of data, for example because of a logic error.

## Database size estimate

### The idea in one minute

The database size is roughly:

> **size = number of rows x bytes per row**

We count two kinds of rows:

- a **node row** is a node together with one plugin claim on it (`nodes` + `node_plugin_claims`),
- a **relation row** is a relation together with one plugin claim on it (`relations` + `relation_plugin_claims`).

Counting a node together with its claim is a simplification, explained in [Bytes per row rationale](#bytes-per-row-rationale) below.

#### Bytes per row

| Row | Consists of | Bytes per row |
|---|---|---|
| node row | `nodes` + `node_plugin_claims` | 10^3 |
| relation row | `relations` + `relation_plugin_claims` | 6 x 10^2 |

Row sizes include indexes and leave room for the JSON `properties` of the claim (~500 B per node and ~150 B per relation). This is the **baseline**. It covers the typical claims of plugins (see [Plugin properties](#plugin-properties)) with some margin. Concrete synthetic JSON examples at the baseline, JSON x3 and JSON x9 sizes are provided in [Appendix A](rfc-011-appendix-example-json/rfc-011-example-json.md).

To see what happens when plugins send more data, the scenarios also use three bigger variants. Only the JSON grows; the fixed part (identifiers, row overhead, indexes) stays the same, ~500 B per node and ~370 B per relation:

```text
node row     = 500 B + JSON per node          (rounded up)
relation row = 370 B + JSON per relation      (rounded up)
```

| Variant | JSON per node | JSON per relation | Node row | Relation row |
|---|---|---|---|---|
| **Baseline** | 5 x 10^2 B | 1.5 x 10^2 B | 10^3 B | 6 x 10^2 B (370 + 150 = 520) |
| **JSON x3** | 1.5 x 10^3 B | 4.5 x 10^2 B | 2 x 10^3 B (500 + 1,500) | 9 x 10^2 B (370 + 450 = 820) |
| **JSON x9** | 4.5 x 10^3 B | 1.35 x 10^3 B | 5 x 10^3 B (500 + 4,500) | 2 x 10^3 B (370 + 1,350 = 1,720) |
| **JSON x50** | 2.5 x 10^4 B | 7.5 x 10^3 B | 3 x 10^4 B (500 + 25,000 = 25,500) | 8 x 10^3 B (370 + 7,500 = 7,870) |

The row sizes are calculated from the proposed schema:

<a id="proposed-schema"></a>
<details>
<summary>Proposed schema</summary>

```sql
CREATE TABLE nodes (
    id   UUID PRIMARY KEY,
    kind VARCHAR(40) NOT NULL,
    path VARCHAR(300) NOT NULL,
    CONSTRAINT nodes_identity_key UNIQUE (kind, path)
);

CREATE TABLE node_plugin_claims (
    node_id     UUID        NOT NULL REFERENCES nodes (id),
    plugin_name VARCHAR(40) NOT NULL,
    snapshot_id UUID        NOT NULL,
    properties  JSONB       NOT NULL DEFAULT '{}',
    PRIMARY KEY (node_id, plugin_name)
);

CREATE TABLE relations (
    id           UUID PRIMARY KEY,
    kind         VARCHAR(40) NOT NULL,
    from_node_id UUID NOT NULL REFERENCES nodes (id),
    to_node_id   UUID NOT NULL REFERENCES nodes (id),
    CONSTRAINT relations_identity_key UNIQUE (kind, from_node_id, to_node_id)
);

CREATE TABLE relation_plugin_claims (
    relation_id UUID        NOT NULL REFERENCES relations (id),
    plugin_name VARCHAR(40) NOT NULL,
    snapshot_id UUID        NOT NULL,
    properties  JSONB       NOT NULL DEFAULT '{}',
    PRIMARY KEY (relation_id, plugin_name)
);
```

The schema is taken from [PR 247](https://github.com/naira-project/naira/pull/247). The original schema also includes operations, but their impact on database size is negligible compared to nodes and relations

`VARCHAR(n)` limits are in characters, not bytes. This RFC assumes ASCII identifiers (1 B per character).

</details>

<a id="bytes-per-row-rationale"></a>
<details>
<summary>Bytes per row rationale</summary>

Only arithmetic, not measurement.

**Assumptions** (1 B per character):

- Typical `kind` ≈ 20 B, `path` ≈ 100 B, `plugin_name` ≈ 20 B. The schema allows up to 40 / 300 / 40 (`VARCHAR`), but those are caps, not typical values. For reference, a `path` with two UUIDs written as text (2 x 36 characters) is already ~100 B.
- `id`, `*_id`, `snapshot_id`: UUID columns, 16 B each (binary, not the 36-character text form).
- Overhead: a table row costs its data + ~30 B (row header and pointer). An index entry costs its key + ~20 B. A primary key or unique constraint is an index, so its columns are stored twice: once in the table and once in the index.

**Cost of one row per table** (typical values, without JSON, every value rounded up):

| Table | Table row | Indexes | Total |
|---|---|---|---|
| `nodes` | `id` UUID (16) + `kind` (20) + `path` (100) + row overhead (30) ≈ 170 B | PK: `id` UUID (16) + index overhead (20) ≈ 40 B; unique: `kind` (20) + `path` (100) + index overhead (20) = 140 B | 170 (row) + 40 (PK) + 140 (unique) ≈ 350 B |
| `node_plugin_claims` | `node_id` UUID (16) + `plugin_name` (20) + `snapshot_id` UUID (16) + row overhead (30) ≈ 90 B, plus JSON | PK: `node_id` UUID (16) + `plugin_name` (20) + index overhead (20) ≈ 60 B | 90 (row) + 60 (PK) ≈ 150 B, plus JSON |
| `relations` | `id` UUID (16) + `kind` (20) + `from_node_id` UUID (16) + `to_node_id` UUID (16) + row overhead (30) ≈ 100 B | PK: `id` UUID (16) + index overhead (20) ≈ 40 B; unique: `kind` (20) + each node UUID (16 + 16) + index overhead (20) ≈ 80 B | 100 (row) + 40 (PK) + 80 (unique) ≈ 220 B |
| `relation_plugin_claims` | `relation_id` UUID (16) + `plugin_name` (20) + `snapshot_id` UUID (16) + row overhead (30) ≈ 90 B, plus JSON | PK: `relation_id` UUID (16) + `plugin_name` (20) + index overhead (20) ≈ 60 B | 90 (row) + 60 (PK) ≈ 150 B, plus JSON |

**Why count claims together with nodes and relations.** In reality, different plugins report overlapping data, so there are fewer `nodes` and `relations` rows than claims. In the extreme case, the number of `nodes` rows equals the number of claims. To stay on the safe side, we add the sizes of the two tables and treat them as one row: a node row and a relation row. This is an upper bound with respect to overlap.

**Node row: 10^3 B.**
Typical: 350 (`nodes`) + 150 (claim without JSON) = 500 B. That leaves ~500 B for JSON `properties`.
At the schema caps (`kind` 40 B, `path` 300 B, `plugin_name` 40 B), a `nodes` row is 16 + 40 + 300 + 30 ≈ 390 in the table, plus ~40 for the PK, plus 40 + 300 + 20 = 360 for the unique index, so ~790 in total. The claim grows only by ~40 B (longer `plugin_name`), to ~190 B. Together ~980 B, which leaves only ~20 B for JSON. So 10^3 B is a typical-values budget; a node with maximal identifiers and non-trivial JSON exceeds it.

**Relation row: 6 x 10^2 B.**
Typical: 220 (`relations`) + 150 (claim without JSON) = 370 B. With ~150 B for JSON this is 520 B, rounded up to 6 x 10^2 B. We use a smaller JSON budget than for nodes because we expect the JSON on relations to be significantly smaller than on nodes.
At the schema caps (`kind` 40 B, `plugin_name` 40 B), both the relation and its claim are ~40 B bigger, so ~450 B in total, which still leaves ~150 B for JSON within the 6 x 10^2 B. `path` does not matter here, relations only store ids.

</details>

<a id="plugin-properties"></a>

<details>
<summary>Plugin properties</summary>

Code-based inventory from 2026.10.23 of the properties emitted by each current plugin. Byte estimates use compact UTF-8 JSON and are rough planning examples, not limits. A useful approximation for a properties object is:

```text
JSON bytes ≈ 2 + Σ(key bytes + value bytes + 5) + (property count - 1)
```

The `5` accounts for quotes around the key and string value plus the colon; the `2` is `{}` and commas separate properties. Most keys are about **8–22 B**. Typical short values (names, statuses, IDs, counts) are about **5–40 B**; URLs often **40–100 B**. Descriptions, schemas, and serialized metadata can be hundreds of bytes to multiple KiB. Non-ASCII text and JSON escaping can increase byte size.

| Plugin | Node properties | Relation properties | Representative JSON size / notes |
|---|---|---|---|
| `bedrock` | Model: usually 5. Inference endpoint: roughly 4–10. | `serves_model`: 0. | Model properties are short names, modalities and provider. Endpoint values are region/model identifiers, status and optional numeric usage. Typically a few hundred bytes per node. |
| `depl_calls_svc` | Deployment and Service nodes: 0. | `calls`: 1 (`env`, the matching environment-variable name). | Usually a few dozen bytes per relation. |
| `depl_uses_litellm` | Deployment and model nodes: 0. | `uses_model`: 0. | Empty properties objects. |
| `depl_uses_thalamus` | Deployment and model nodes: 0. | `uses_model`: 0. | Empty properties objects. |
| `fluxcd` | Flux GitRepository and external repository nodes: 1 (`url`). Other emitted nodes: 0. | All relation kinds: 0. | A repository URL is commonly ~40–100 B; JSON properties around ~55–115 B for that one property. Repository nodes are a small subset of the graph. |
| `argocd` | Application: 6–10 properties (project/status/destination/source type, operation flag, and optional deploy/operation timestamps). Git repository: 1 (`url`). Managed resource: 0. | `describes`: 0. `sourced_from` and `deployed_from`: 0–1 (`paths`, a JSON-encoded list of source paths). | Application JSON is typically a few hundred bytes. The relation `paths` value grows with source paths and has no plugin-side cap; one short path adds roughly 15–30 B plus commas/quotes. |
| `github` | Repository: 1–3 (`url`, optional `language`, `homepage`). Deployment and owner: 0. | `built_from`, `owned_by`: 0. | Repository properties typically ~100–250 B, largely determined by URL/homepage length. |
| `litellm` | Model: 1 (`owned_by`). Inference endpoint: typically ~7–15 (up to 15 optional/conditional fields in the current mapper). MCP server/tool: variable; see note. | `serves_model`, `exposes`: 0. App `uses_model`: 1 (`litellm_virtual_key`). | Model: ~25–60 B. Endpoint: typically ~0.4–1.0 KiB for short scalar values. App edge includes the virtual key, whose length is externally controlled. MCP tool schemas/descriptions can be much larger and are not universally size-capped. |
| `mcp_servers` | Server: typically ~3–14. Tool: 0–9. | `exposes`: 0. | Server metadata is usually hundreds of bytes or less. Tool descriptions are clipped to 2,000 Unicode code points, but input/output schemas have no explicit size cap and may be KiB-sized. |
| `mlflow` | Model: 1 (`description`). Dataset: 2 (`digest`, `source_type`). | `trained_on`: 1 (`run_id`). | Dataset and relation properties are usually ~50–150 B. Model JSON size depends on description; there is no plugin-side clipping. |
| `openai_api_models` | Model: provider-dependent; zero or more properties. Extra `/v1/models` fields are retained, with nested object keys flattened. | None. | No representative fixed property count: provider response controls both count and content. A model can have KiB of properties, e.g. nested metadata or arrays. No plugin-side size cap. |
| `openmetadata` | Dataset/table: 4 base properties, typically 4–9 total (description, FQN, URL, optional types/tags/owners/columns). | `derived_from`: 0. | Often a few hundred bytes, but serialized columns can make properties KiB-sized. Text and columns are not clipped. |
| `tech_radar` | Radar: 8. Entry: 8. | None. | Entry is typically a few hundred bytes, with rationale capped at 2,000 code points.|
| `thalamus` | CRD model: typically 8–12. API model fallback: 1 (`owned_by`). Optional Hugging Face repository: 0. | `sourced_from`: 0. | Most CRD values are short, but serialized `engine_args` and `node_selector` can make the properties hundreds of bytes or more; neither is clipped. |

**What this means for the row budgets:**

- Most claims are small: Deployment and Service nodes and most relations have empty properties, and repositories and datasets are ~50–250 B. The baseline (~500 B per node, ~150 B per relation) covers these.
- Some node kinds are medium, a few hundred bytes to ~1 KiB: ArgoCD application, LiteLLM inference endpoint, `thalamus` CRD model, `tech_radar` entry, MCP server, Bedrock endpoint. They are between baseline budget and JSON x3.
- Some properties are **unbounded** (no plugin-side clipping) and can reach KiB: MCP tool schemas, `openai_api_models` metadata, `openmetadata` columns, `mlflow` model description, `thalamus` `engine_args` / `node_selector`. These are the claims that go beyond the baseline. The **JSON x3**, **JSON x9** and **JSON x50** variants show what happens if such claims become common. **JSON x50** (~25 KB per node, ~7.5 KB per relation) is an extreme what-if, far above anything in the inventory above, meant to show the upper end of the range.

</details>

### Assumptions

1. The estimate covers **current graph state only** (`nodes`, `node_plugin_claims`, `relations`, `relation_plugin_claims`). Snapshot application replaces a plugin's previous claims and removes nodes and relations that no plugin claims anymore, so old snapshots are not stored. If we decide to keep them, the numbers below are multiplied by the number of retained snapshots.
2. The `operations` table is **not** included. It grows with run history, not with graph size.
3. If plugins report the same node or relation, there are fewer rows in `nodes` and `relations`. We ignore this (see [Bytes per row rationale](#bytes-per-row-rationale)).
4. The row sizes from [Bytes per row](#bytes-per-row) are budgets for typical identifiers: they include ~500 B of JSON per node and ~150 B per relation, but only ~20 B per node (and still ~150 B per relation) when identifiers are at their caps.
5. The **JSON x3**, **JSON x9** and **JSON x50** variants multiply only the JSON part of the baseline (500 -> 1,500 -> 4,500 -> 25,000 B per node, 150 -> 450 -> 1,350 -> 7,500 B per relation). They are what-if variants.
6. All scenarios describe one catalog database that holds data from all `C` clusters.

### How many rows

The number of rows depends on four factors:

| Symbol | Meaning |
|---|---|
| `P` | plugins |
| `C` | clusters |
| `N` | nodes emitted by one plugin from one cluster |
| `R` | relations per node |

Each of the `P` plugins runs on each of the `C` clusters and emits `N` nodes, each with `R` relations. So:

- node rows = `P x C x N`
- relation rows = `P x C x N x R`

<a id="where-numbers-come"></a>

#### Where the numbers come from

- **`P` (plugins).** We have about 10 plugins today. Backstage has 258 active plugins. In practice a customer doesn't use every tool in a market per cluster (IIRC Max was saying one time that customers are usually happy with 3 or 4 specific plugins to their needs), so 100 plugins per cluster is a safe margin. We use `P = 10^2` as the maximum for this discussion.
- **`C` (clusters).**
- **`N` (nodes per plugin run).**
- **`R` (relations per node).** 

### Scenarios

Database size = node rows x node row bytes + relation rows x relation row bytes, rounded up. The row sizes for the four variants come from [Bytes per row](#bytes-per-row); the calculation is shown below the table.

| Scenario | `P` | `C` | `N` | `R` | Node rows `P x C x N` | Relation rows `P x C x N x R` | Baseline | JSON x3 | JSON x9 | JSON x50 |
|---|---|---|---|---|---|---|---|---|---|---|
| Today | 10^1 | 1 | 10^3 | 10^1 | 10^4 | 10^5 | 7 x 10^7 B | 2 x 10^8 B | 3 x 10^8 B | 2 x 10^9 B |
| Early game | 10^2 | 10^1 | 10^3 | 10^1 | 10^6 | 10^7 | 7 x 10^9 B | 2 x 10^10 B | 3 x 10^10 B | 2 x 10^11 B |
| Stretch (more clusters) | 10^2 | 10^2 | 10^3 | 10^1 | 10^7 | 10^8 | 7 x 10^10 B | 2 x 10^11 B | 3 x 10^11 B | 2 x 10^12 B |
| Early game, denser graph | 10^2 | 10^1 | 10^3 | 10^2 | 10^6 | 10^8 | 7 x 10^10 B | 10^11 B | 3 x 10^11 B | 9 x 10^11 B |

#### How the numbers are calculated

```text
size = node rows x node row bytes + relation rows x relation row bytes
```

**Baseline** (node row 10^3 B, relation row 6 x 10^2 B):

| Scenario | Node rows | `node_bytes` | Relation rows | `relation_bytes` | Sum | Rounded up |
|---|---|---|---|---|---|---|
| Today | 10^4 | 10^4 x 10^3 = 10^7 | 10^5 | 10^5 x 6 x 10^2 = 6 x 10^7 | 7 x 10^7 B | **7 x 10^7 B** |
| Early game | 10^6 | 10^6 x 10^3 = 10^9 | 10^7 | 10^7 x 6 x 10^2 = 6 x 10^9 | 7 x 10^9 B | **7 x 10^9 B** |
| Stretch | 10^7 | 10^7 x 10^3 = 10^10 | 10^8 | 10^8 x 6 x 10^2 = 6 x 10^10 | 7 x 10^10 B | **7 x 10^10 B** |
| Denser graph | 10^6 | 10^6 x 10^3 = 10^9 | 10^8 | 10^8 x 6 x 10^2 = 6 x 10^10 | 6.1 x 10^10 B | **7 x 10^10 B** |

**JSON x3** (node row 2 x 10^3 B, relation row 9 x 10^2 B):

| Scenario | Node rows | `node_bytes` | Relation rows | `relation_bytes` | Sum | Rounded up |
|---|---|---|---|---|---|---|
| Today | 10^4 | 10^4 x 2 x 10^3 = 2 x 10^7 | 10^5 | 10^5 x 9 x 10^2 = 9 x 10^7 | 1.1 x 10^8 B | **2 x 10^8 B** |
| Early game | 10^6 | 10^6 x 2 x 10^3 = 2 x 10^9 | 10^7 | 10^7 x 9 x 10^2 = 9 x 10^9 | 1.1 x 10^10 B | **2 x 10^10 B** |
| Stretch | 10^7 | 10^7 x 2 x 10^3 = 2 x 10^10 | 10^8 | 10^8 x 9 x 10^2 = 9 x 10^10 | 1.1 x 10^11 B | **2 x 10^11 B** |
| Denser graph | 10^6 | 10^6 x 2 x 10^3 = 2 x 10^9 | 10^8 | 10^8 x 9 x 10^2 = 9 x 10^10 | 9.2 x 10^10 B | **10^11 B** |

**JSON x9** (node row 5 x 10^3 B, relation row 2 x 10^3 B):

| Scenario | Node rows | `node_bytes` | Relation rows | `relation_bytes` | Sum | Rounded up |
|---|---|---|---|---|---|---|
| Today | 10^4 | 10^4 x 5 x 10^3 = 5 x 10^7 | 10^5 | 10^5 x 2 x 10^3 = 2 x 10^8 | 2.5 x 10^8 B | **3 x 10^8 B** |
| Early game | 10^6 | 10^6 x 5 x 10^3 = 5 x 10^9 | 10^7 | 10^7 x 2 x 10^3 = 2 x 10^10 | 2.5 x 10^10 B | **3 x 10^10 B** |
| Stretch | 10^7 | 10^7 x 5 x 10^3 = 5 x 10^10 | 10^8 | 10^8 x 2 x 10^3 = 2 x 10^11 | 2.5 x 10^11 B | **3 x 10^11 B** |
| Denser graph | 10^6 | 10^6 x 5 x 10^3 = 5 x 10^9 | 10^8 | 10^8 x 2 x 10^3 = 2 x 10^11 | 2.05 x 10^11 B | **3 x 10^11 B** |

**JSON x50** (node row 3 x 10^4 B, relation row 8 x 10^3 B):

| Scenario | Node rows | `node_bytes` | Relation rows | `relation_bytes` | Sum | Rounded up |
|---|---|---|---|---|---|---|
| Today | 10^4 | 10^4 x 3 x 10^4 = 3 x 10^8 | 10^5 | 10^5 x 8 x 10^3 = 8 x 10^8 | 1.1 x 10^9 B | **2 x 10^9 B** |
| Early game | 10^6 | 10^6 x 3 x 10^4 = 3 x 10^10 | 10^7 | 10^7 x 8 x 10^3 = 8 x 10^10 | 1.1 x 10^11 B | **2 x 10^11 B** |
| Stretch | 10^7 | 10^7 x 3 x 10^4 = 3 x 10^11 | 10^8 | 10^8 x 8 x 10^3 = 8 x 10^11 | 1.1 x 10^12 B | **2 x 10^12 B** |
| Denser graph | 10^6 | 10^6 x 3 x 10^4 = 3 x 10^10 | 10^8 | 10^8 x 8 x 10^3 = 8 x 10^11 | 8.3 x 10^11 B | **9 x 10^11 B** |

Stretch is the early game scenario with 10 times more clusters, so everything is 10 times larger. In the denser graph only `R` changed compared to the early game, and the relation rows now dominate.

### Implications of the estimate

- Size grows **linearly** with `P`, `C` and `N`. Ten times more clusters means ten times more storage.
- **Relations per node (`R`)** matters a lot. Relation rows (relations and their claims) are the most numerous, so they are the largest part of the database. A denser graph costs up to as much as ten times more clusters.
- **The size of the plugin JSON** is another major factor. The data in `node_plugin_claims` heavily impacts storage, and the JSON payload in `relation_plugin_claims` can increase the database size even faster, because there can be many more relation claims than node claims.
- More JSON grows the database less than proportionally, because the fixed part of each row stays the same. In the Today and Early game scenarios, JSON x3 gives a database ~1.6 times bigger than the baseline, JSON x9 ~4 times and JSON x50 ~16 times (compared on the unrounded sums from the calculation tables).

## Plugin output limits

### Proposed limits

These limits discourage excessive output from a single plugin; they do **not** enforce the database estimates above. Hitting a limit should prompt a review: is this a plugin bug, unnecessary data, or a legitimate need for more room? 

| Limit | Value | Reasoning |
|---|---|---|
| `kind` length | 40 B | The longest kind today has 21 characters. Plugin authors control this string. |
| `path` length | 300 B | The longest path today is a GitHub repo id (~100 characters) plus a prefix (~10). Leave headroom for identifiers from other source systems. |
| Plugin run response (one plugin, one cluster) | 4 MiB (current) | This is the gRPC receive limit we have today, not a target value. We keep it for now and expect to raise it later (option B). |

Hitting limits reject the whole snapshot and keep the last successful one.

### Plugin run size: options

**Proposal:** for now, we do not add a limit of our own on the size of a plugin run. The only limit is the gRPC message size limit that we already have (option B), which is 4 MiB. We do not treat 4 MiB as a value to aim for: it is the current default, and we expect to raise it, or to support chunked responses, once real plugin runs need more (see [Implications for the 4 MiB limit](#implications-for-the-4-mib-limit)). This limit does not guarantee the size in the database.

#### Option A: Limit parts (JSON per claim, nodes per run, relations per node)

- **Pros:** simple checks; catches specific problems such as one huge JSON blob.
- **Cons:** rigid trade-offs. Small JSON limits suit many small relations but block a few rich ones; count limits do the reverse.

#### Option B: Limit response size with the gRPC message limit (proposed for now)

Today the catalog rejects any plugin response bigger than the gRPC receive limit, which is at the default of **4 MiB** ([grpc-go default](https://github.com/grpc/grpc-go/blob/679b224109d3373eb0e4d80b6e41aafc8d295b6d/server.go#L448)). This is the limit we currently have, not a size we have chosen as right for plugins. When plugins need more, we will have to raise it (a configuration value, possibly per plugin) or send the response in chunks.

- **Pros:** nothing new to implement or calibrate; oversized responses are rejected before decoding; plugin authors can freely trade the number of objects against the size of the payload.
- **Cons:** response size does not reliably predict database size. To compare them, we generated synthetic plugin responses with **5,000–20,000 nodes**, **1–20 relations per node** and varying sizes of kinds, paths and properties. We measured each response in its serialized protobuf form, applied it through the proposed schema, and compared it with the PostgreSQL size of the resulting tables and indexes. The database was ~**1–10 times** the response size after insertion. The highest ratios occur with short identifiers and empty properties, where the fixed row and index overhead dominates. Extrapolating these ratios, a 4 MiB response may need roughly 5–50 MB in the database. So the scenario estimates in this RFC are planning numbers, not guarantees.

#### Option C: Estimate storage before inserting and limit the estimate

After decoding the response and before writing to the database, estimate how much space each node and relation will take and reject the snapshot if the total is above a threshold, for example 10 MB. A possible rule for one node or relation is **400 B + 2 x its text bytes** (kind, path where present, plugin name and properties). The 400 B is illustrative only; if we decide to pursue this approach, it will need to be calculated.

- **Pros:** closer to the real storage size than the wire size.
- **Cons:** extra accounting and maintenance. The fixed values must be revisited after schema or index changes, and validated against real data before we can rely on them.

### Potential storage impact of the current limit

At the largest ratio observed in Option B, a full response would correspond to **4 MiB × 10 ≈ 50 MB** (42 MB, rounded up) in the database per plugin–cluster pair. This is an illustrative high-end case from synthetic measurements. If the limit is raised in the future, these numbers grow in the same proportion.

If every plugin–cluster pair simultaneously reached that size:

| Scenario | Plugin–cluster pairs (`P × C`) | Illustrative database size at ~50 MB per pair |
|---|---:|---:|
| Today | 10 | ~0.5 GB |
| Early game (including denser graph) | 1,000 | ~50 GB |
| Stretch (more clusters) | 10,000 | ~500 GB |

### Plugin response size on the wire

The database scenarios in [scenarios](#scenarios) estimate aggregate storage, but they do not show whether one plugin run fits within the current **4 MiB** gRPC message limit. This section estimates response sizes for representative node and relation counts, compares them with that limit, and shows which scenarios may require a higher limit or support for chunked responses.

The 4 MiB limit applies to **one gRPC message**: the response of one plugin run on one cluster. So `P` and `C` do not matter here, only `N` (nodes per run) and `R` (relations per node). To keep the arithmetic easy we round 4 MiB to **4 x 10^6 B** (the real limit is 4,194,304 B, about 5% more).

**Rules of thumb.** In the response, nodes and relations are described by text, and protobuf adds a few bytes of tag and length to every field. So we count the text bytes and add a small fixed overhead:

```text
node     = kind + path + 10 B                  + node properties
relation = kind + 2 x (kind + path) + 20 B     + relation properties
property = key + value + 5 B

bytes per node = node + R x relation
response size  = N x bytes per node
max N          = (4 x 10^6 B) / bytes per node
```

On the wire a relation carries the full `kind` and `path` of **both** ends, not UUIDs as in the database. Without properties this makes a relation about twice as big as a node.

**Example shapes** (`kind` = 20 B, `path` = 100 B):

| Shape | Properties | node | relation |
|---|---|---|---|
| A | none | 20 + 100 + 10 = **130 B** | 20 + 2 x (20 + 100) + 20 = **280 B** |
| B | the budgets from [Bytes per row](#bytes-per-row): 10 properties per node and 3 per relation, each with a 10 B key and a 35 B value (50 B per property) | 130 + 10 x 50 = **630 B** | 280 + 3 x 50 = **430 B** |

**Maximum nodes that fit in 4 x 10^6 B** (`max N` is rounded down, because a smaller `N` is the safe side). For shape B with `R` = 10, the bytes per node are `630 + 10 x 430 = 4,930 B`. Substituting this into the `max N` formula above gives `4 x 10^6 B / 4,930 B ≈ 8 x 10^2` nodes.

| Shape | | `R` = 0 | `R` = 1 | `R` = 10 | `R` = 100 |
|---|---|---:|---:|---:|---:|
| A | bytes per node | 130 | 410 | 2,930 | 28,130 |
| A | max `N` | ~3 x 10^4 | ~9 x 10^3 | ~10^3 | ~10^2 |
| B | bytes per node | 630 | 1,060 | 4,930 | 43,630 |
| B | max `N` | ~6 x 10^3 | ~3 x 10^3 | ~8 x 10^2 | ~9 x 10^1 |

As soon as there are relations, they dominate.

**The scenarios against the limit.** Response size = `N` x bytes per node (from the table above). `P` and `C` do not matter for one run, so Today, Early game and Stretch (all with `N` = 10^3 and `R` = 10) give the same response size.

| Scenario | Shape | `N` | `R` | Bytes per node | Response size | Response / (4 x 10^6 B) | Fits? |
|---|---|---:|---:|---:|---:|---:|---|
| Today, Early game, Stretch | A | 10^3 | 10 | 2,930 | ~3 x 10^6 B | ~0.7 | yes |
| Today, Early game, Stretch | B | 10^3 | 10 | 4,930 | ~5 x 10^6 B | ~1.2 | **no** |
| Denser graph | A | 10^3 | 10^2 | 28,130 | ~3 x 10^7 B | ~7 | **no** |
| Denser graph | B | 10^3 | 10^2 | 43,630 | ~5 x 10^7 B | ~12 | **no** |
| 10k nodes + 10R | A | 10^4 | 10 | 2,930 | ~3 x 10^7 B | ~7 | **no** |
| 10k nodes + 10R | B | 10^4 | 10 | 4,930 | ~5 x 10^7 B | ~12 | **no** |

#### Implications for the 4 MiB limit

4 MiB is the limit we have today, and it does **not** cover the larger per-run scenarios shown above. Already shape B with `N` = 10^3 and `R` = 10 is above it, and the denser graph is about 10 times above it. So at some point the limit will have to be raised, or chunked responses supported, before plugins can send responses of those sizes.

## Additional context

### Answers to the review questions

Questions from [@akavel-reply's comment](https://github.com/naira-project/naira/pull/247/changes#r4249078169):

- **How many plugins might we have?** At most about 100 (`P = 10^2`; see [Where the numbers come from](#where-numbers-come)), and typically around 25 per cluster (see [Customer-scale context from Max](#customer-scale-context-from-max)).
- **How many kinds of nodes could a plugin emit?** This is not needed for the estimate, only the number of nodes matters. We do not propose a limit.
- **How long a `kind` and a `path` do we allow?** 40 B and 300 B. See [Proposed limits](#proposed-limits).
- **How many nodes can a plugin emit, how much space do we allow per JSON `properties` blob, and how many relations do we allow?** We do not set separate limits for these. For now, the **4 MiB** gRPC message limit that we have today bounds the complete response from one plugin run. It is not a target, and we expect to raise it in the future; see [Proposed limits](#proposed-limits) and [Plugin response size on the wire](#plugin-response-size-on-the-wire). How big the `properties` are in practice is shown in [Plugin properties](#plugin-properties).

### Customer-scale context from Max

Max's feedback helps put their customers scale into context:

- **Expected number of plugins.** Based on typical Backstage usage, we can usually expect around 10–15 plugins running on a cluster. Naira-specific plugins could add approximately 10 more. This suggests a typical total of roughly 20–25 plugins per cluster. The estimate uses `P = 10^2`, so the current model already includes a conservative margin above this expected range.

- **Reasonable database size.** In many companies, around 50 GB is available by default for a database. Large customers may reasonably give hundreds of gigabytes or even terabytes.

- **Filtering reduces the number of nodes.** We can filter out resources such as sidecars and observability containers. This should reduce the number of containers represented in the catalog and therefore reduce `N`, the number of nodes emitted by a plugin.

- **Large-customer deployment scenarios.** Max described two possible scenarios:
  - With a few clusters, a customer may typically have around 1,000 deployments, with a maximum of approximately 10,000 deployments.
  - With around 200 clusters, a customer may typically have hundreds of deployments per cluster, with a maximum of approximately 1,000 deployments per cluster.

## Summary

- For the nearest future, we plan to focus on the "Early game" scenario with:
    - `P` = 100 plugins max
    - `C` = 10 clusters max
    - 4 MiB default gRPC limit on the size of data in a single snapshot message from a Plugin
        - this should fit for example roughly:
            - 800 NodeClaims,
            - 10 Relations per each Node,
            - with on average ~500B max JSON per NodeClaim
            - and ~150B max JSON per Relation
            - (assuming on average ~10B per prop key and ~40B per prop value, this is ~10 props per Node and ~3 props per Relation).
- With the above, we estimate DB usage at <100GiB.

## Appendix

- [Appendix A: Example JSON `properties`](rfc-011-appendix-example-json/rfc-011-example-json.md) — synthetic claim payload examples and their byte-size breakdowns, with JSON fixtures. These illustrate the baseline, JSON x3 and JSON x9 estimates; JSON x50 is omitted as a boundary case.