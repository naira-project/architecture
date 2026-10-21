# RFC-011: Database size estimate and limits for plugin output

| Field | Value |
|---|---|
| RFC | 011 |
| Title | Database size estimate and limits for plugin output |
| Status | Draft, open for comments |
| Scope | First deployment and early scaling. This is not a long-term or hyperscale estimate. |
| Origin | Inspired by [@akavel-reply's comment](https://github.com/naira-project/naira/pull/247#discussion_r4249078169) requesting a database size estimation. |

Final estimates are rounded to powers of 10. They show orders of magnitude, not precise values.

## Goal

Separate two questions for the first deployment and early scaling:

- **Capacity planning:** estimate database size from the schema, row counts and representative payloads.
- **Plugin guardrails:** make it harder for one plugin run to send an unreasonable amount of data, for example because of a logic error.

## The idea in one minute

The database size is roughly:

> **size = number of rows x bytes per row**

We count two kinds of rows:

- a **node row** is a node together with one plugin claim on it (`nodes` + `node_plugin_claims`),
- a **relation row** is a relation together with one plugin claim on it (`relations` + `relation_plugin_claims`).

Counting a node together with its claim is a simplification, explained in [Bytes per row rationale](#bytes-per-row-rationale) below.

### Bytes per row

| Row | Consists of | Bytes per row |
|---|---|---|
| node row | `nodes` + `node_plugin_claims` | 10^3 |
| relation row | `relations` + `relation_plugin_claims` | 5 x 10^2 |

Row sizes include indexes and leave room for the JSON `properties` of the claim. They are calculated from the proposed schema:

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

- Typical `kind` ≈ 20 B, `path` ≈ 100 B, `plugin_name` ≈ 20 B. The schema allows up to 40 / 300 / 40 (`VARCHAR`), but those are caps, not typical values. For reference, a `path` with two UUIDs written as text (2 x 36 characters) is already ≈ 100 B.
- `id`, `*_id`, `snapshot_id`: UUID columns, 16 B each (binary, not the 36-character text form).
- Overhead: a table row costs its data + ≈ 30 B (row header and pointer). An index entry costs its key + ≈ 20 B. A primary key or unique constraint is an index, so its columns are stored twice: once in the table and once in the index.

**Cost of one row per table** (typical values, without JSON):

| Table | Table row | Indexes | Total |
|---|---|---|---|
| `nodes` | 16 + 20 + 100 + 30 ≈ 170 | PK: 16 + 20 ≈ 40; unique (`kind`, `path`): 20 + 100 + 20 ≈ 140 | ≈ 350 |
| `node_plugin_claims` | 16 + 20 + 16 + 30 ≈ 80, plus JSON | PK: 16 + 20 + 20 ≈ 60 | ≈ 140 + JSON |
| `relations` | 16 + 20 + 16 + 16 + 30 ≈ 100 | PK: 16 + 20 ≈ 40; unique (`kind`, `from`, `to`): 20 + 16 + 16 + 20 ≈ 70 | ≈ 210 |
| `relation_plugin_claims` | 16 + 20 + 16 + 30 ≈ 80, plus JSON | PK: 16 + 20 + 20 ≈ 60 | ≈ 140 + JSON |

**Why count claims together with nodes and relations.** In reality, different plugins report overlapping data, so there are fewer `nodes` and `relations` rows than claims. In the extreme case, the number of `nodes` rows equals the number of claims. To stay on the safe side, we add the sizes of the two tables and treat them as one row: a node row and a relation row. This is an upper bound with respect to overlap.

**Node row: 10^3 B.**
Typical: 350 (`nodes`) + 140 (claim without JSON) = 490 B. That leaves ≈ 500 B for JSON `properties`.
At the schema caps (`kind` 40 B, `path` 300 B, `plugin_name` 40 B), a `nodes` row is 16 + 40 + 300 + 30 ≈ 390 in the table, plus ≈ 40 for the PK, plus 40 + 300 + 20 = 360 for the unique index, so ≈ 790 in total. The claim grows only by ≈ 40 B (longer `plugin_name`), to ≈ 180 B. Together ≈ 970 B, which leaves only ≈ 30 B for JSON. So 10^3 B is a typical-values budget; a node with maximal identifiers and non-trivial JSON exceeds it.

**Relation row: 5 x 10^2 B.**
Typical: 210 (`relations`) + 140 (claim without JSON) = 350 B. That leaves ≈ 150 B for JSON. We use a smaller budget than for nodes because we expect the JSON on relations to be significantly smaller than on nodes.
At the schema caps (`kind` 40 B, `plugin_name` 40 B), both the relation and its claim are ≈ 40 B bigger, so ≈ 430 B in total, which leaves ≈ 70 B for JSON. `path` does not matter here, relations only store ids.

</details>

## Assumptions

1. The estimate covers **current graph state only** (`nodes`, `node_plugin_claims`, `relations`, `relation_plugin_claims`). Snapshot application replaces a plugin's previous claims and removes nodes and relations that no plugin claims anymore, so old snapshots are not stored. If we decide to keep them, the numbers below are multiplied by the number of retained snapshots.
2. The `operations` table is **not** included. It grows with run history, not with graph size.
3. If plugins report the same object, there are fewer rows in `nodes` and `relations`. We ignore this (see [Bytes per row rationale](#bytes-per-row-rationale)).
4. The row sizes from [Bytes per row](#bytes-per-row) are budgets for typical identifiers: they include ≈ 500 B of JSON per node and ≈ 150 B per relation, but only ≈ 30 B and ≈ 70 B when identifiers are at their caps.
6. All scenarios describe one catalog database that holds data from all `C` clusters.

## How many rows

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

### Where the numbers come from

- **`P` (plugins).** We have about 10 plugins today. Backstage has 258 active plugins. In practice a customer doesn't use every tool in a market per cluster (IIRC Max was saying one time that customers are usually happy with 3 or 4 specific plugins to their needs), so 100 plugins per cluster is a safe margin. We use `P = 10^2` as the maximum for this discussion.
- **`C` (clusters).**
- **`N` (nodes per plugin run).**
- **`R` (relations per node).** 

## Scenarios

Database size = node rows x 10^3 B + relation rows x 5 x 10^2 B, rounded to a power of 10. The row sizes come from [Bytes per row](#bytes-per-row); the calculation is shown below the table.

| Scenario | `P` | `C` | `N` | `R` | Node rows `P x C x N` | Relation rows `P x C x N x R` | Database size |
|---|---|---|---|---|---|---|---|
| Today | 10^1 | 1 | 10^3 | 10^1 | 10^4 | 10^5 | ~10^8 B |
| Early game | 10^2 | 10^1 | 10^3 | 10^1 | 10^6 | 10^7 | ~10^10 B |
| Stretch (more clusters) | 10^2 | 10^2 | 10^3 | 10^1 | 10^7 | 10^8 | ~10^11 B |
| Early game, denser graph | 10^2 | 10^1 | 10^3 | 10^2 | 10^6 | 10^8 | ~10^11 B |

### How the numbers are calculated

```text
node_bytes     = (P x C x N)     x 10^3 B
relation_bytes = (P x C x N x R) x 5 x 10^2 B
size           = node_bytes + relation_bytes, rounded to the nearest power of 10
```

| Scenario | `node_bytes` | `relation_bytes` | Sum | Rounded |
|---|---|---|---|---|
| Today | 10^4 x 10^3 = 10^7 | 10^5 x 5 x 10^2 = 5 x 10^7 | 6 x 10^7 B | **~10^8 B** |
| Early game | 10^6 x 10^3 = 10^9 | 10^7 x 5 x 10^2 = 5 x 10^9 | 6 x 10^9 B | **~10^10 B** |
| Stretch | 10^7 x 10^3 = 10^10 | 10^8 x 5 x 10^2 = 5 x 10^10 | 6 x 10^10 B | **~10^11 B** |
| Denser graph | 10^6 x 10^3 = 10^9 | 10^8 x 5 x 10^2 = 5 x 10^10 | 5 x 10^10 B | **~10^11 B** |

Stretch is the early game scenario with 10 times more clusters, so everything is 10 times larger. In the denser graph only `R` changed compared to the early game, and the relation rows now dominate.

## What we can read from this

- Size grows **linearly** with `P`, `C` and `N`. Ten times more clusters means ten times more storage.
- **Relations per node (`R`)** matters a lot. Relation rows (relations and their claims) are the most numerous, so they are the largest part of the database. A denser graph costs about as much as ten times more clusters.
- **The size of the plugin JSON** is another major factor. The data in `node_plugin_claims` heavily impacts storage, and the JSON payload in `relation_plugin_claims` can increase the database size even faster, because there can be many more relation claims than node claims.

## Proposed limits

These limits discourage excessive output from a single plugin; they do **not** enforce the database estimates above. Hitting a limit should prompt a review: is this a plugin bug, unnecessary data, or a legitimate need for more room? 

| Limit | Value | Reasoning |
|---|---|---|
| `kind` length | 40 B | The longest kind today has 21 characters. Plugin authors control this string. |
| `path` length | 300 B | The longest path today is a GitHub repo id (~100 characters) plus a prefix (~10). Leave headroom for identifiers from other source systems. |
| Plugin run response (one plugin, one cluster) | 4 MiB | Retain the existing gRPC receive limit; no additional storage budget (option B). |

Hitting limits reject the whole snapshot and keep the last successful one.

### Limit on plugin run size: options

**Proposal:** for now, the only limit on the size of a plugin run is the gRPC message size limit (option B). It does not guarantee the size in the database, and we accept that for the early game. If real plugin output shows the need, we can add finer guardrails (options A or C).

#### Option A: limit parts (JSON per claim, nodes per run, relations per node)

- **Pros:** simple checks; catches specific problems such as one huge JSON blob.
- **Cons:** rigid trade-offs. Small JSON limits suit many small relations but block a few rich ones; count limits do the reverse.

#### Option B: limit the response size with the gRPC message limit (proposed)

The catalog rejects any plugin response bigger than the gRPC receive limit. The default is **4 MiB** ([grpc-go default](https://github.com/grpc/grpc-go/blob/679b224109d3373eb0e4d80b6e41aafc8d295b6d/server.go#L448)). If a plugin needs more, we can raise the limit (a configuration value, possibly per plugin) or send the response in chunks.

- **Pros:** nothing new to implement or calibrate; oversized responses are rejected before decoding; plugin authors can freely trade the number of objects against the size of the payload.
- **Cons:** response size does not reliably predict database size. To compare them, we generated synthetic plugin responses with **5,000–20,000 nodes**, **1–20 relations per node** and varying sizes of kinds, paths and properties. We measured each response in its serialized protobuf form, applied it through the proposed schema, and compared it with the PostgreSQL size of the resulting tables and indexes. The database was ~**1–10 times** the response size after insertion. The highest ratios occur with short identifiers and empty properties, where the fixed row and index overhead dominates. Extrapolating these ratios, a 4 MiB response may need roughly 4–40 MB in the database. So the scenario estimates in this RFC are planning numbers, not guarantees.

#### Option C: estimate storage before inserting and limit the estimate

After decoding the response and before writing to the database, add up a charge per object and reject the snapshot above a threshold, for example 10 MB. A candidate rule is **400 B per object + 2 x its text bytes** (kind, path where present, plugin name and properties).

- **Pros:** closer to the real storage size than the wire size.
- **Cons:** extra accounting and maintenance. The fixed values must be revisited after schema or index changes, and validated against real data before we can rely on them.

### What the selected limit could mean for storage

At the largest ratio observed after one refresh, a full response would correspond to **4 MiB × 10 ≈ 40 MB** in the database per plugin–cluster pair. This is an illustrative high-end case from synthetic measurements.

If every plugin–cluster pair simultaneously reached that size:

| Scenario | Plugin–cluster pairs (`P × C`) | Illustrative database size at ~40 MB per pair |
|---|---:|---:|
| Today | 10 | ~0.4 GB |
| Early game (including denser graph) | 1,000 | ~40 GB |
| Stretch (more clusters) | 10,000 | ~400 GB |

### How big is a plugin response on the wire?

The database scenarios above estimate aggregate storage, but they do not show whether one plugin run fits within the current **4 MiB** gRPC message limit. This section estimates response sizes for representative node and relation counts, compares them with that limit, and shows which scenarios may require a higher limit or support for chunked responses.

The 4 MiB limit applies to **one gRPC message**: the response of one plugin run on one cluster. So `P` and `C` do not matter here, only `N` (nodes per run) and `R` (relations per node). To keep the arithmetic easy we round 4 MiB to **4 x 10^6 B** (the real limit is 4,194,304 B, about 5% more).

**Rules of thumb.** Protobuf adds a few bytes of tag and length to every field, so we count the text bytes and add a small fixed overhead:

```text
node     = kind + path + 10 B                      + node properties
relation = kind + 2 x (kind + path) + 20 B         + relation properties
property = key + value + 5 B

message  = N x node + N x R x relation
         = N x (node + R x relation)

max N    = 4 x 10^6 B / (node + R x relation)
```

Note that on the wire a relation carries the full `kind` and `path` of **both** ends, not UUIDs as in the database. This is why a relation is about twice as big as a node here.

**Two example shapes** (typical `kind` = 20 B, `path` = 100 B):

- **A:** no properties. node = 20 + 100 + 10 = **~130 B**, relation = 20 + 2 x 120 + 20 = **~280 B**.
- **B:** the JSON budgets from [Bytes per row](#bytes-per-row): ~500 B of properties per node (10 properties, 12 B key + 35 B value) and ~150 B per relation (3 properties, 8 B key + 35 B value). node = 130 + 500 = **~630 B**, relation = 280 + 150 = **~430 B**.

**How many nodes fit in 4 x 10^6 B** (`max N = 4 x 10^6 / (node + R x relation)`):

| Shape | `R` = 0 | `R` = 1 | `R` = 10 | `R` = 100 |
|---|---:|---:|---:|---:|
| A | ~3 x 10^4 | ~10^4 | ~1.4 x 10^3 | ~1.4 x 10^2 |
| B | ~6 x 10^3 | ~4 x 10^3 | ~8 x 10^2 | ~9 x 10^1 |

As soon as there are relations, they dominate.

**The scenarios from [How many rows](#how-many-rows) against the limit.**

| Shape | `N` | `R` | Calculation | Response size | Times the 4 x 10^6 B limit | Fits? |
|---|---:|---:|---|---:|---:|---|
| A | 10^3 | 10 | 10^3 x (130 + 10 x 280) = 10^3 x 2.9 x 10^3 | ~3 x 10^6 B | ~0.7x | yes |
| B | 10^3 | 10 | 10^3 x (630 + 10 x 430) = 10^3 x 4.9 x 10^3 | ~5 x 10^6 B | ~1.2x | **no** |
| A | 10^3 | 10^2 | 10^3 x (130 + 100 x 280) = 10^3 x 2.8 x 10^4 | ~3 x 10^7 B | ~7x | **no** |
| B | 10^3 | 10^2 | 10^3 x (630 + 100 x 430) = 10^3 x 4.4 x 10^4 | ~4 x 10^7 B | ~11x | **no** |
| A | 10^4 | 10 | 10^4 x (130 + 10 x 280) = 10^4 x 2.9 x 10^3 | ~3 x 10^7 B | ~7x | **no** |
| B | 10^4 | 10 | 10^4 x (630 + 10 x 430) = 10^4 x 4.9 x 10^3 | ~5 x 10^7 B | ~12x | **no** |

What we can read from this:

The 4 MiB limit does **not** cover the larger per-run scenarios shown above. If plugins need to send responses of those sizes, we will need to raise the limit or support chunked responses.

## Answers to the review questions

Questions from [@akavel-reply's comment](https://github.com/naira-project/naira/pull/247/changes#r4249078169):

- **How many plugins might we have?** At most about 100 (`P = 10^2`), and typically around 25 per cluster. See [Where the numbers come from](#where-numbers-come).
- **How many kinds of nodes could a plugin emit?** This is not needed for the estimate, only the number of nodes matters. We do not propose a limit.
- **How long a `kind` and a `path` do we allow?** 40 B and 300 B. See [Proposed limits](#proposed-limits).
- **How many nodes can a plugin emit, how much space do we allow per JSON `properties` blob, and how many relations do we allow?** We do not set separate limits for these. The current **4 MiB** gRPC message limit bounds the complete response from one plugin run; see [Proposed limits](#proposed-limits) and [How big is a plugin response on the wire?](#how-big-is-a-plugin-response-on-the-wire).
- **What if a single plugin needs more than the limit?** We can raise the limit, potentially per plugin, or support chunked responses.

## Notes from Max

Max's feedback helps put their customers scale into context:

- **Expected number of plugins.** Based on typical Backstage usage, we can usually expect around 10–15 plugins running on a cluster. Naira-specific plugins could add approximately 10 more. This suggests a typical total of roughly 20–25 plugins per cluster. The estimate uses `P = 10^2`, so the current model already includes a conservative margin above this expected range.

- **Reasonable database size.** In many companies, around 50 GB is available by default for a database. Large customers may reasonably give hundreds of gigabytes or even terabytes.

- **Filtering reduces the number of nodes.** We can filter out resources such as sidecars and observability containers. This should reduce the number of containers represented in the catalog and therefore reduce `N`, the number of nodes emitted by a plugin.

- **Large-customer deployment scenarios.** Max described two possible scenarios:
  - With a few clusters, a customer may typically have around 1,000 deployments, with a maximum of approximately 10,000 deployments.
  - With around 200 clusters, a customer may typically have hundreds of deployments per cluster, with a maximum of approximately 1,000 deployments per cluster.

- **Implication for the stretch scenario.** If the **Stretch (more clusters)** scenario uses `N = 10^4` instead of `N = 10^3`, its estimate increases by a factor of 10, from ~10^11 B (~100 GB) to ~10^12 B (~1 TB).
