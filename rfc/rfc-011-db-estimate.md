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

Make an initial, rough estimate of how much data we may want to store, so we can decide:

- what limits to put on the data stored in the database,
- what limits to put on the data returned by plugins.

The estimate should be good enough for our first deployment and early scaling. The purpose of this document is to start a discussion and to see how much headroom we have.

## The idea in one minute

The database size is roughly:

> **size = number of rows x bytes per row**

We count two kinds of rows:

- a **node row** is a node together with one plugin claim on it (`nodes` + `node_plugin_claims`),
- a **relation row** is a relation together with one plugin claim on it (`relations` + `relation_plugin_claims`).

Counting a node and its claim together is a simplification, explained in the [Bytes per row rationale](#bytes-per-row-rationale) below.

### Bytes per row
| Row | Consists of | Bytes per row |
|---|---|---|
| node row | `nodes` + `node_plugin_claims` | 10^3 |
| relation row | `relations` + `relation_plugin_claims` | 5 x 10^2 |

The row sizes include the indexes and leave room for the JSON `properties` of a claim.

The row sizes were calculated from the proposed schema:

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
    node_id     UUID         NOT NULL REFERENCES nodes (id),
    plugin_name VARCHAR(40)  NOT NULL,
    snapshot_id UUID  NOT NULL,
    properties  JSONB NOT NULL DEFAULT '{}',
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
    relation_id UUID         NOT NULL REFERENCES relations (id),
    plugin_name VARCHAR(40)  NOT NULL,
    snapshot_id UUID  NOT NULL,
    properties  JSONB NOT NULL DEFAULT '{}',
    PRIMARY KEY (relation_id, plugin_name)
);
```

This schema is taken from [PR 247](https://github.com/naira-project/naira/pull/247). The original schema also includes operations, but their impact on database size is negligible compared to nodes and relations.

</details>

<a id="bytes-per-row-rationale"></a>

<details>

<summary>Bytes per row rationale</summary>

Only arithmetic, not a measurement.

**Assumptions** (1 B per character):

- Typical `kind` ≈ 20 B, `path` ≈ 100 B, `plugin_name` ≈ 20 B. The schema allows up to 40 / 300 / 40 (`VARCHAR`), but those are caps, not typical values. For reference, a `path` with two UUIDs written as text (2 x 36 characters) is already ≈ 100 B.
- `id`, `*_id`, `snapshot_id`: UUID columns, 16 B each (binary, not the 36-character text form).
- Overhead: a table row costs its data + ≈ 30 B (row header and pointer). An index entry costs its key + ≈ 20 B. A primary key or a unique constraint is an index, so its columns are stored twice: once in the table and once in the index.

**Cost of one row of each table** (typical values, without the JSON):

| Table | Table row | Indexes | Total |
|---|---|---|---|
| `nodes` | 16 + 20 + 100 + 30 ≈ 170 | PK: 16 + 20 ≈ 40; unique (`kind`, `path`): 20 + 100 + 20 ≈ 140 | ≈ 350 |
| `node_plugin_claims` | 16 + 20 + 16 + 30 ≈ 80, plus JSON | PK: 16 + 20 + 20 ≈ 60 | ≈ 140 + JSON |
| `relations` | 16 + 20 + 16 + 16 + 30 ≈ 100 | PK: 16 + 20 ≈ 40; unique (`kind`, `from`, `to`): 20 + 16 + 16 + 20 ≈ 70 | ≈ 210 |
| `relation_plugin_claims` | 16 + 20 + 16 + 30 ≈ 80, plus JSON | PK: 16 + 20 + 20 ≈ 60 | ≈ 140 + JSON |

**Why we count claims together with nodes and relations.** To keep the arithmetic simple, we take the extreme case: every `node_plugin_claims` row has its own `nodes` row, and every `relation_plugin_claims` row has its own `relations` row. In reality, different plugins report overlapping data, so there are fewer `nodes` and `relations` rows than claims. With this extreme case the number of `nodes` rows equals the number of claims, so we can add the sizes of the two rows and treat them as one row: a node row and a relation row.

**Node row: 10^3 B.** Typical: 350 (`nodes`) + 140 (claim without JSON) = 490 B. This leaves ≈ 500 B for the JSON `properties`.

At the schema caps (`kind` 40 B, `path` 300 B, `plugin_name` 40 B) the `nodes` row is 16 + 40 + 300 + 30 ≈ 390 in the table, plus ≈ 40 for the PK, plus 40 + 300 + 20 = 360 for the unique index, so ≈ 790 in total. The claim grows only by ≈ 40 B (longer `plugin_name`), to ≈ 180 B. Together that is ≈ 970 B, which leaves only ≈ 30 B for JSON.

**Relation row: 5 x 10^2 B.** Typical: 210 (`relations`) + 140 (claim without JSON) = 350 B. This leaves ≈ 150 B for JSON (2-3 small properties). We use a smaller budget than for nodes because we expect the JSON of relations to be significantly smaller than the JSON of nodes.

At the schema caps (`kind` 40 B, `plugin_name` 40 B) both rows are ≈ 40 B bigger, so ≈ 430 B in total, which leaves ≈ 70 B for JSON. `path` does not matter here, because relations only store ids.

</details>

## Assumptions

1. We estimate the **current state only**. Old snapshots are not stored. If we decide to keep them, every number below is multiplied by the number of retained snapshots.
2. If several plugins report the same object, there are fewer rows in `nodes` and `relations`. We ignore that (see the [Bytes per row rationale](#bytes-per-row-rationale)), so the estimate is an **upper bound**.
3. The row sizes from [Bytes per row](#bytes-per-row) are upper limits: they include ≈ 500 B of JSON per node and ≈ 150 B per relation (only ≈ 30 B and ≈ 70 B if `kind` and `path` are at their caps).

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

### How each size is calculated

```
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

The main purpose of these limits is to guide plugin authors toward building plugins that scale easily, as well as to catch and reject plugin runs with logic errors that cause them to send an enormous amount of data.
We should set these limits in the codebase and reject snapshots that exceed them. When a plugin hits a limit, we should determine whether it's an unintended bug from the plugin, or if the catalog should increase or redesign the limits.

| Limit | Value | Reasoning |
|---|---|---|
| `kind` length | 40 B | The longest kind today has 21 characters. It is a string defined by the plugin author, so it is easy to fit. Maybe even set limit to 30 B. |
| `path` length | 300 B | The longest path today is a GitHub repo id (max ~100 characters) plus a prefix of ~10 characters. This value is a proposal and can change if a plugin justifies needing more. Plugin authors only decide what the identifier of a resource is, but its length depends on the source system—which is why there should be some buffer. |
| Whole plugin response (one plugin, one cluster) | 10 MB | Main limit, see below. |

We could set limits on things like:
- JSON `properties` per node claim
- JSON `properties` per relation claim
- Nodes per plugin run
- Relations per node
- Relations per plugin run

However it is problematic to find one correct limit for every JSON blob or number of relations. A small limit per relation JSON (for example 100 B) makes sense when we expect millions of relations. But it makes no sense for a plugin that returns only a few relations and wants to keep more data in each of them. The same logic applies to nodes.

So instead of tuning many separate limits, the proposal is to start with one limit on the **whole response of a plugin run: 10 MB = 10^7 B**. The same limit also answers the question "how many relations do we allow", so we do not need to find the "correct" number of relations.

### Why 10 MB

We derive the limit from the storage budget. Every (plugin, cluster) pair produces one run, and a run can never be larger than the limit `L`. So in the worst case:

> **database size ≤ P x C x L**

A byte of response costs roughly 1-2 bytes in the database, so treat this as an order of magnitude.

For `L = 10^7 B` (10 MB):

| Scenario | `P x C` | Worst case `P x C x 10^7 B` | Estimate from [Scenarios](#scenarios) |
|---|---|---|---|
| Today | 10^1 | 10^8 B | ~10^8 B |
| Early game | 10^3 | 10^10 B | ~10^10 B |
| Stretch | 10^4 | 10^11 B | ~10^11 B |

Early game fits within the ~50 GB many companies have by default. Stretch needs the "hundreds of GB" that large customers can give (see [Notes from Max](#notes-from-max)). For a different budget `B`, the limit per run is `B / (P x C)`.

The limit applies to the whole response, so it bounds everything inside it at once: the size of the JSON, the number of nodes and the number of relations. Plugin authors decide how to spend it.

**Transport.** The default maximum message size in gRPC is 4 MB ([grpc-go](https://github.com/grpc/grpc-go/blob/679b224109d3373eb0e4d80b6e41aafc8d295b6d/server.go#L448)). To accept responses of up to 10 MB, we would have to either raise this limit or send the plugin run data in chunks.

## Answers to the review questions

Questions from [@akavel-reply's comment](https://github.com/naira-project/naira/pull/247/changes#r4249078169):

- **How many plugins might we have?** At most about 100 (`P = 10^2`), and typically around 25 per cluster. See [Where the numbers come from](#where-numbers-come).
- **How many kinds of nodes could a plugin emit?** This is not needed for the estimate, only the number of nodes matters. We do not propose a limit.
- **How long a `kind` and a `path` do we allow?** 40 B and 300 B. See [Proposed limits](#proposed-limits).
- **How many nodes can a plugin emit, how much space do we allow per JSON `properties` blob, and how many relations do we allow?** We do not try to calculate exact values for these. One limit on the whole response of a plugin run (10 MB) bounds all of them together, see [Why 10 MB](#why-10-mb).
- **What if a single plugin needs more than the limit?** The limit can be configured per plugin, so a plugin that needs more can get an explicitly higher one.

## Notes

- **Plugin sizes in other tools.** OpenMetadata can produce a significant number of nodes. Its public benchmark was done with 10,000, 100,000 and 1.8M entities, representing a small, medium and large organization ([benchmark](https://blog.open-metadata.org/openmetadata-benchmarking-3120782f8a12)). I do not know if this is representative for us. Max what do you think? Models, MCP servers and deployments look like more limited sets of data.

## Notes from Max

Max's feedback helps put their customers scale into context:

- **Expected number of plugins.** Based on typical Backstage usage, we can usually expect around 10–15 plugins running on a cluster. Naira-specific plugins could add approximately 10 more. This suggests a typical total of roughly 20–25 plugins per cluster. The estimate uses `P = 10^2`, so the current model already includes a conservative margin above this expected range.

- **Reasonable database size.** In many companies, around 50 GB is available by default for a database. Large customers may reasonably give hundreds of gigabytes or even terabytes.

- **Filtering reduces the number of nodes.** We can filter out resources such as sidecars and observability containers. This should reduce the number of containers represented in the catalog and therefore reduce `N`, the number of nodes emitted by a plugin.

- **Large-customer deployment scenarios.** Max described two possible scenarios:
  - With a few clusters, a customer may typically have around 1,000 deployments, with a maximum of approximately 10,000 deployments.
  - With around 200 clusters, a customer may typically have hundreds of deployments per cluster, with a maximum of approximately 1,000 deployments per cluster.

- **Implication for the stretch scenario.** If the **Stretch (more clusters)** scenario uses `N = 10^4` instead of `N = 10^3`, its estimate increases by a factor of 10, from ~10^11 B (~100 GB) to ~10^12 B (~1 TB).
