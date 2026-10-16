# RFC-011: Database size estimate and limits for plugin output

| Field | Value |
|---|---|
| RFC | 011 |
| Title | Database size estimate and limits for plugin output |
| Status | Draft, open for comments |
| Scope | First deployment and early scaling. This is not a long-term or hyperscale estimate. |
| Origin | Inspired by [@akavel-reply's comment](https://github.com/naira-project/naira/pull/247#discussion_r4249078169) requesting a database size estimation. |

Numbers are rounded to powers of 10. They show orders of magnitude, not precise values.

## Goal

Make an initial, rough estimate of how much data we may want to store, so we can decide:

- what limits to put on the data stored in the database,
- what limits to put on the data returned by plugins.

The estimate should be good enough for our first deployment and early scaling. The purpose of this document is to start a discussion and to see how much headroom we have.

## The idea in one minute

For each of the four tables, the database size is roughly:

> **size = number of rows x bytes per row**

The row sizes were calculated from the schema in the [PR](https://github.com/naira-project/naira/pull/247), with one difference: instead of repeated text for ids we use `BIGINT` for nodes and relations. Because of that, relations and claims only carry small numbers.

| Table | One row per | Bytes per row |
|---|---|---|
| `nodes` | distinct node | 10^2 |
| `node_plugin_claims` | (node, plugin), includes the JSON `properties` | 10^3 |
| `relations` | distinct relation | 10^2 |
| `relation_plugin_claims` | (relation, plugin), includes the JSON `properties` | 10^2 |

`node_plugin_claims` is one order of magnitude bigger on purpose: it leaves room for the JSON `properties` a node claim can carry. Assuming that `relation_plugin_claims` JSON will be significantly smaller

## Assumptions

1. We estimate the **current state only**. Old snapshots are not stored. If we decide to keep them, every number below is multiplied by the number of retained snapshots.
2. If several plugins report the same object, there are fewer rows in `nodes` and `relations`. We ignore that, so the estimate is an **upper bound**.
3. The row sizes from section 2 are upper limits; for example, for nodes, it gives around 800B for JSON. Up to now, plugins have sent less data in properties.

## How many rows

Start from one plugin running on one cluster and multiply:

| Symbol | Meaning |
|---|---|
| `P` | plugins |
| `C` | clusters |
| `N` | nodes emitted by one plugin from one cluster |
| `R` | relations per node |

Then:

- node claims = `P x C x N`
- relations and relation claims = `P x C x N x R`

### Where the numbers come from

- **`P` (plugins).** We have about 10 plugins today. Backstage has 258 active plugins. In practice a customer doesn't use every tool in a market per cluster (IIRC Max was saying one time that customers are usually happy with 3 or 4 specific plugins to their needs), so 100 plugins per cluster is a safe margin. We use `P = 10^2` as the maximum for this discussion.
- **`C` (clusters).**
- **`N` (nodes per plugin run).**
- **`R` (relations per node).** 

## Scenarios

| Scenario | `P` | `C` | `N` | `R` | Database size |
|---|---|---|---|---|---|
| Today | 10^1 | 1 | 10^3 | 10^1 | ~10^7 B |
| Early game | 10^2 | 10^1 | 10^3 | 10^1 | ~10^9 B |
| Stretch (more clusters) | 10^2 | 10^2 | 10^3 | 10^1 | ~10^10 B |
| Early game, denser graph | 10^2 | 10^1 | 10^3 | 10^2 | ~10^10 B |

### How each size is calculated

For every scenario we do the same three steps:

1. **Rows** per table, from the formulas in section 4.
2. **Size per table** = rows x bytes per row.
3. **Total** = sum of the four tables

#### Today: `P`=10^1, `C`=1, `N`=10^3, `R`=10^1

| Table | Rows | Bytes per row | Size |
|---|---|---|---|
| `nodes` | 10^1 x 1 x 10^3 = 10^4 | 10^2 | 10^6 B |
| `node_plugin_claims` | 10^4 | 10^3 | 10^7 B |
| `relations` | 10^4 x 10^1 = 10^5 | 10^2 | 10^7 B |
| `relation_plugin_claims` | 10^5 | 10^2 | 10^7 B |

Total: about 3 x 10^7 B, so **~10^7 B**.

#### Early game: `P`=10^2, `C`=10^1, `N`=10^3, `R`=10^1

| Table | Rows | Bytes per row | Size |
|---|---|---|---|
| `nodes` | 10^2 x 10^1 x 10^3 = 10^6 | 10^2 | 10^8 B |
| `node_plugin_claims` | 10^6 | 10^3 | 10^9 B |
| `relations` | 10^6 x 10^1 = 10^7 | 10^2 | 10^9 B |
| `relation_plugin_claims` | 10^7 | 10^2 | 10^9 B |

Total: about 3 x 10^9 B, so **~10^9 B**.

#### Stretch (more clusters): `P`=10^2, `C`=10^2, `N`=10^3, `R`=10^1

This is the early game scenario with 10 times more clusters, so everything is 10 times larger: about 3 x 10^10 B, so **~10^10 B**.

#### Early game, denser graph: `P`=10^2, `C`=10^1, `N`=10^3, `R`=10^2

| Table | Rows | Bytes per row | Size |
|---|---|---|---|
| `nodes` | 10^2 x 10^1 x 10^3 = 10^6 | 10^2 | 10^8 B |
| `node_plugin_claims` | 10^6 | 10^3 | 10^9 B |
| `relations` | 10^6 x 10^2 = 10^8 | 10^2 | 10^10 B |
| `relation_plugin_claims` | 10^8 | 10^2 | 10^10 B |

Total: about 2 x 10^10 B, so **~10^10 B**.
Only `R` changed compared to the early game, and the two relation tables now
dominate.

## What we can read from this

- Size grows **linearly** with `P`, `C` and `N`. Ten times more clusters means ten times more storage.
- **Relations per node (`R`)** matters a lot. Relations and their claims have the highest number of rows, so they are the largest tables. A denser graph costs about as much as ten times more clusters.
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

So instead of tuning many separate limits, proposition is to start with one limit on the **whole response of a plugin run: 10 MB**. The same limit also answers the question "how many relations do we allow", so we do not need to find the "correct" number of relations.

### Does 10 MB fit the estimate?

Take a snapshot: 1000 nodes, each with ~600 B of JSON, and 10 relations per node.

| Table | Rows | Bytes per row | Size |
|---|---|---|---|
| `nodes` | 10^3 | 10^2 | 10^5 B |
| `node_plugin_claims` | 10^3 | 10^3 | 10^6 B |
| `relations` | 10^3 x 10^1 = 10^4 | 10^2 | 10^6 B |
| `relation_plugin_claims` | 10^4 | 10^2 | 10^6 B |

Total: about 3 x 10^6 B. The mentioned run uses about 3 MB, comparing to it 10 MB doesn't look as a particularly high limit.
However, this run with 600 B JSON per node represents an upper bound, and it will probably take a long time before we actually see a run like that on real data.
That's why I think 10 MB will be enough for the time being

## Notes

- **Plugin sizes in other tools.** OpenMetadata can produce a significant number of nodes. Its public benchmark was done with 10,000, 100,000 and 1.8M entities, representing a small, medium and large organization ([benchmark](https://blog.open-metadata.org/openmetadata-benchmarking-3120782f8a12)). I do not know if this is representative for us. Max what do you think? Models, MCP servers and deployments look like more limited sets of data.
- **Number of `kinds` per plugin.** Not needed for the estimate only the number of nodes matters. I'm not proposing a limit here. A large number of nodes would simply be a signal that it could be split into several smaller plugins, which would also make it easier to test.

## Notes from Max

Max's feedback helps put their customers scale into context:

- **Expected number of plugins.** Based on typical Backstage usage, we can usually expect around 10–15 plugins running on a cluster. Naira-specific plugins could add approximately 10 more. This suggests a typical total of roughly 20–25 plugins per cluster. The estimate uses `P = 10^2`, so the current model already includes a conservative margin above this expected range.

- **Reasonable database size.** In many companies, around 50 GB is available by default for a database. Large customers may reasonably give hundreds of gigabytes or even terabytes.

- **Filtering reduces the number of nodes.** We can filter out resources such as sidecars and observability containers. This should reduce the number of containers represented in the catalog and therefore reduce `N`, the number of nodes emitted by a plugin.

- **Large-customer deployment scenarios.** Max described two possible scenarios:
  - With a few clusters, a customer may typically have around 1,000 deployments, with a maximum of approximately 10,000 deployments.
  - With around 200 clusters, a customer may typically have hundreds of deployments per cluster, with a maximum of approximately 1,000 deployments per cluster.

- **Implication for the stretch scenario.** If the **Stretch (more clusters)** scenario uses `N = 10^4` instead of `N = 10^3`, its estimate increases by a factor of 10, from ~10 GB to ~100 GB.
