# Consolidating the 10008 two-stage rule — why we chose ENRICH over LOOKUP JOIN on 9.3

## Background

The 10008 CPU-utilization alert is currently split across two rules — **Stage 1** detects
breaches on the remote `prod` cluster and writes intermediary records into a local
`kibana_threshold_alerts` index; **Stage 2** reads those local records and joins them against
the local AMDB config index (`kibana_sdp_amdb`) to filter for hosts where 10008 is enabled.

This two-hop design existed because on Elasticsearch 8.x, `LOOKUP JOIN` could not cross
clusters — a join between remote metric data and a local config index was rejected. The
intermediary index was pure scaffolding to make the join local-to-local.

**On 9.3, that scaffolding is no longer necessary — but which one-rule replacement to build is
a real decision.**

## What 9.x actually changed

- **9.2.0 GA:** Remote `LOOKUP JOIN` supported in cross-cluster queries. Each cluster resolves
  the lookup index locally and joins against its own copy — the lookup index **must exist on
  every cluster being queried**.
  → [ES|QL `LOOKUP JOIN` → Cross-cluster support](https://www.elastic.co/docs/reference/query-languages/esql/esql-lookup-join#cross-cluster-support)
  → [ES|QL across clusters → LOOKUP JOIN across clusters](https://www.elastic.co/docs/reference/query-languages/esql/esql-cross-clusters#ccq-lookup-join)

- **9.6+ (not our version):** Coordinator mode arrives. `LOOKUP JOIN` can then join against a
  *local* copy while the metric data is remote. That's the mode that would let us keep
  exactly one AMDB copy on the CCS cluster while querying remote metric data.

- **Cross-cluster `LOOKUP JOIN` has one more constraint:** it can't appear after `STATS`,
  `SORT`, `LIMIT`, or a coordinator-side `ENRICH` — so the join has to happen on raw metric
  rows, before the aggregation.
  → [ES|QL `LOOKUP JOIN` → Limitations](https://www.elastic.co/docs/reference/query-languages/esql/esql-lookup-join#_limitations)

So on 9.3 there are two real routes, both a genuine improvement over the two-stage design:

| Route | Where the AMDB config index lives | Freshness | Extra ops step |
|---|---|---|---|
| `LOOKUP JOIN` | On every remote cluster we query (dev, qa, prod) | **Live** — reads the index every run | None |
| `ENRICH _coordinator:` | On the local (CCS) cluster only | Snapshot — re-execute after every change | `PUT _enrich/policy/…/_execute` |

## Why ENRICH, given our multi-cluster reality

Our rule reads `FROM dev:metricbeat-*, qa:metricbeat-*, prod:metricbeat-*` — three remote
clusters. On 9.3, `LOOKUP JOIN` therefore requires `kibana_sdp_amdb_enrich` **on all three
remotes**:

- **3× indices to create, sync, and monitor.** Each remote is authoritative for its own
  hosts, so it isn't one "master + replicas" pattern — it's three independently maintained
  indices.
- **3× write paths.** Config edits for dev hosts have to route to dev; qa to qa; prod to
  prod. Our current AMDB write path is single-target and would have to be split.
- **3× silent-failure surfaces.** Remote clusters default to `skip_unavailable: true` — a
  missing, empty, or stale copy on any one cluster is reported as a *skipped* cluster rather
  than an error, and the rule runs green with **zero alerts and no error**.
  → [ES|QL across clusters → Skipping problematic remote clusters](https://www.elastic.co/docs/reference/query-languages/esql/esql-cross-clusters#ccq-skip-unavailable-clusters)

The `ENRICH` route collapses all of that to a single local index:

- **`ENRICH _coordinator:kibana_sdp_amdb_policy`** forces the enrich to execute on the
  coordinating (local) cluster — one index, one policy, one place to maintain.
  → [ES|QL across clusters → Enrich with coordinator mode](https://www.elastic.co/docs/reference/query-languages/esql/esql-cross-clusters#esql-enrich-coordinator)
- **Cheaper at query time.** Coordinator-mode `ENRICH` is not subject to the "no remote join
  after `STATS`" restriction — it runs *after* the aggregation, enriching only the handful
  of breaching hosts rather than every raw metric document. The `LOOKUP JOIN` route has to
  happen on raw rows.
- **Same downstream event.** The emitted ticket written to `kibana_alerts` is byte-identical
  to what the `LOOKUP JOIN` route would produce — no change for anything consuming those
  alerts.

## The trade-off we accepted

`ENRICH` reads a **snapshot** the policy materializes at `_execute` time — not the live
source index — via a hidden `.enrich-*` system index that the `_execute` rebuilds each time.
This means:

- Every AMDB config change (host added, alert enabled/disabled) requires re-running
  `PUT _enrich/policy/kibana_sdp_amdb_policy/_execute`.
- Until it's re-executed, a newly enabled host won't alert and a newly disabled host will
  keep alerting.

**Mitigation:** wire the `_execute` into the AMDB maintenance path, or schedule it (hourly is
plenty for config that changes rarely). `LOOKUP JOIN` avoids this because it reads the source
live — but that's the one live-vs-snapshot property that comes with the three-copy cost above.

→ [Enrich processor & policies overview](https://www.elastic.co/docs/reference/enrich-processor)
explains the snapshot / execute model.

## When to re-evaluate

The moment we upgrade to **9.6**, coordinator-mode `LOOKUP JOIN` becomes available:

> *`applies_to` stack: ga 9.6+ — If the lookup index is missing from one or more remote
> clusters, use coordinator mode to join against a local cluster lookup index copy.*
>
> — [ES|QL across clusters → LOOKUP JOIN across clusters](https://www.elastic.co/docs/reference/query-languages/esql/esql-cross-clusters#ccq-lookup-join)

At that point, `LOOKUP JOIN` gives us live reads **and** a single local copy — strictly
better than `ENRICH` for this use case, and worth revisiting.

## References

- **ES|QL `LOOKUP JOIN` reference:** https://www.elastic.co/docs/reference/query-languages/esql/esql-lookup-join
- **ES|QL across clusters (cross-cluster search):** https://www.elastic.co/docs/reference/query-languages/esql/esql-cross-clusters
- **`ENRICH` command reference:** https://www.elastic.co/docs/reference/query-languages/esql/commands/enrich
- **Enrich processor / policy overview:** https://www.elastic.co/docs/reference/enrich-processor
- **9.2.0 release notes ("Add remote index support to `LOOKUP JOIN`"):** https://www.elastic.co/docs/release-notes/elasticsearch/9.2.0
- **Elasticsearch Labs — *ES|QL in 9.2: Smart Lookup Joins and time-series support*:** https://www.elastic.co/search-labs/blog/esql-elasticsearch-9-2-multi-field-joins-ts-command

## Related files in this repo

- `10008_single_stage_README.md` — full design rationale, verified against Elasticsearch and
  Kibana 9.3 source
- `10008_walkthrough.md` — step-by-step cutover instructions for the Kibana UI
- `10008_single_stage_enrich_variant` — the ENRICH rule chosen here
- `10008_single_stage_enrich_index` + `10008_single_stage` — the two `LOOKUP JOIN` variants
  kept for reference / future 9.6 migration
- `kibana_sdp_amdb_enrich_index`, `kibana_sdp_amdb_enrich_policy` — the local-cluster setup
