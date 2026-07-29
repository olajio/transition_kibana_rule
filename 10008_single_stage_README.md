# 10008 — consolidating the two-stage rule into one ES|QL rule (Elastic 9.3)

## Why there were two rules

The original design (SharePoint page *Monitoring and Analytics → Kibana Alerts*, "Version 1")
was a single ES|QL rule: read the metric data, `LOOKUP JOIN` the AMDB config index to drop
hosts that don't have the alert enabled, emit the event.

That could not be built on 8.x, because **`LOOKUP JOIN` did not support cross-cluster
queries**: the metric data lives on the remote `prod` cluster, the config index
(`kibana_sdp_amdb`) lives on the local alerting cluster, and a join across the two was
rejected. Hence "Version 2", the two-rule workaround:

| | Rule | Query | Purpose |
|---|---|---|---|
| Stage 1 | `ITSMA Test 10008 - ... Stage 1` | `FROM prod:metricbeat-*` → `STATS AVG(cpu)` → `WHERE avg_cpu > 98` | detect the breach on the remote cluster, write a bare event into the local `kibana_threshold_alerts` index |
| Stage 2 | `ITSMA Test 10008 - ... Stage 2` | `FROM kibana_threshold_alerts` → `LOOKUP JOIN kibana_sdp_amdb` → filter `group`/`alert_status` | now that both indices are local, do the join, build the ticket event, write to `kibana_alerts` |

`kibana_threshold_alerts` existed only to move remote data onto the local cluster so the
join became local-to-local.

## What changed in 9.x

Remote `LOOKUP JOIN` is supported from **9.2.0**, so on **9.3** the join can be done in the
same query that reads `prod:metricbeat-*` — the intermediary index and the second rule are
no longer needed.

Two 9.3 constraints shape the query, and both are load-bearing:

1. **The lookup index must exist on every cluster being queried.** ES|QL resolves the
   lookup index on each cluster and each cluster joins against its *own* local copy — the
   same model as remote-mode `ENRICH`. `LOOKUP JOIN` also takes a concrete index name only:
   wildcards, aliases and `cluster:index` references are not accepted, so you cannot write
   `LOOKUP JOIN prod:kibana_sdp_amdb`.
   *Joining against a local copy while the data is remote ("coordinator mode") only arrives
   in 9.6 — it is not available to us on 9.3.*
2. **Cross-cluster `LOOKUP JOIN` cannot appear after `STATS`, `SORT`, `LIMIT`, or a
   coordinator-side `ENRICH`.** So the join has to happen on the raw metric documents,
   **before** the aggregation — the reverse of the Stage 1 / Stage 2 order.

## The consolidated query

```esql
FROM prod:metricbeat-*, prod:-metricbeat-7*
| WHERE event.dataset == "system.cpu" AND @timestamp > NOW() - 5 minutes
| EVAL metric_client_code = labels.client_code, metric_app_code = labels.app_code
| RENAME host.hostname AS hostname.keyword
| LOOKUP JOIN kibana_sdp_amdb ON hostname.keyword
| MV_EXPAND group
| WHERE group == "10008" AND alert_status == "enabled"
| STATS avg_cpu = AVG(system.cpu.total.norm.pct) * 100 BY hostname.keyword, metric_client_code, metric_app_code
| WHERE avg_cpu > 98
| EVAL current_value = ROUND(avg_cpu, 2)
| DROP avg_cpu
| RENAME metric_client_code AS labels.client_code, metric_app_code AS labels.app_code
| LIMIT 10000
```

Line by line:

- **`prod:-metricbeat-7*`** — excludes the legacy 7.x indices. This is the fix for the error
  Stage 1 is currently failing with:

  > `Cannot use field [event.dataset] due to ambiguities being mapped as [2] incompatible
  > types: [keyword] in [prod:.ds-metricbeat-8.11.0-…] and [321] other indices, [text] in
  > [prod:metricbeat-7.17.21, prod:metricbeat-7.17.22, prod:metricbeat-7.8.1] and [3] other
  > indices`

  ES|QL refuses a field mapped to conflicting types across the queried indices. Those 7.x
  indices are from 2023 and can never contribute a document to a 5-minute window, so
  excluding them costs nothing.
  *Fallback if the exclusion doesn't cover every legacy index:* cast the field instead —
  `| EVAL event_dataset = event.dataset::keyword | WHERE event_dataset == "system.cpu"`.
  Casting resolves the ambiguity as long as the only reference to the original field is the
  conversion itself. Note union-type casting is still a technical preview feature, which is
  why index exclusion is the primary approach here.
- **`EVAL metric_client_code = …`, and the closing `RENAME`** — `client_code` / `app_code`
  are plausible field names in an AMDB config index, and fields coming from the lookup index
  **override** same-named columns on the left. Copying the two metricbeat values to
  collision-proof names *before* the join, then renaming them back *after* the aggregation,
  guarantees the ticket carries the app/client code from the metric document and not from
  `kibana_sdp_amdb`. If you have confirmed `kibana_sdp_amdb` has no such fields, these two
  lines can be dropped and the `STATS` can group `BY hostname.keyword, labels.client_code,
  labels.app_code` directly.
- **`RENAME host.hostname AS hostname.keyword`** — unchanged from Stage 2. The join key has
  to exist under the same name on both sides; in `kibana_sdp_amdb` the usable keyword field
  is the `hostname.keyword` subfield, so the left side needs a column with that literal
  name. `host.hostname` is `keyword` in metricbeat, the same type Stage 2 was joining from,
  so join behaviour is identical.
- **`MV_EXPAND group`** — unchanged from Stage 2. `group` is a list in the config index and
  `WHERE group == "10008"` does not behave as intended on a multivalued field, so the list is
  expanded to one row per value first.
- **`STATS` after the join** — forced by constraint 2 above. The join is a left join, and
  the `WHERE group == … AND alert_status == "enabled"` immediately after it drops both
  non-matching rows (null) and disabled hosts, so only monitored hosts reach the aggregation.
  If a host matches several `kibana_sdp_amdb` documents its metric rows are duplicated, but
  every row for that host duplicates equally and the group is keyed by host, so `AVG` is
  unaffected.
- **`ROUND(avg_cpu, 2)` into `current_value`** — the threshold is compared against the true
  average (`WHERE avg_cpu > 98`) and only the *displayed* value is rounded, so rounding cannot
  move a host across the threshold. This matters because Kibana stringifies ES|QL values
  verbatim (`row[i].toString()`), and `AVG(scaled_float) * 100` produces IEEE-754 noise:
  without the rounding, `alarm_reason` reads *"…is 99.11999999999999"*. The two-stage flow hid
  this by round-tripping `avg_cpu` through `current_value`, a `float`-mapped field in
  `kibana_threshold_alerts`; a single-stage rule has no such round trip and has to round
  explicitly. `current_value` keeps the name the action template already reads, so the ticket
  body is unchanged.

## Prerequisites before enabling

1. **`kibana_sdp_amdb` must exist on the `prod` cluster**, with `index.mode: lookup`:

   ```console
   # on the prod cluster
   PUT kibana_sdp_amdb
   {
     "settings": { "index.mode": "lookup" },
     "mappings": { ... same mapping as the local copy ... }
   }
   ```

   Verify with `GET kibana_sdp_amdb/_settings` on `prod` and confirm `index.mode` is
   `lookup`. Indices in lookup mode are always single-sharded.

   For keeping it in sync, cross-cluster replication (already in use in this deployment) is
   the natural mechanism, with the local index as leader — confirm the follower keeps
   `index.mode: lookup`, since the join fails without it. A scheduled reindex/transform to
   `prod` works too.

2. **Watch out for the silent-failure mode.** Remote clusters default to
   `skip_unavailable: true`, which means a cluster is marked `skipped` — rather than failing
   the query — when it *does not have the requested index*. If `kibana_sdp_amdb` is missing
   or not in lookup mode on `prod`, the likely outcome is a rule that runs green and produces
   **zero alerts**. Validate with "Test query" in the rule editor and confirm rows come back
   before trusting it, and keep an eye on the rule's alert counts after the cutover.

3. **ES|QL cross-cluster search requires a matching subscription level on the local and
   every remote cluster** (the current docs state this as an Enterprise subscription), the
   `remote_cluster_client` role on the coordinating node, API-key based remote cluster
   security, and read privileges on the remote indices. Stage 1 already queries
   `prod:metricbeat-*`, so this is in place — the new requirement is read access to
   `kibana_sdp_amdb` *on prod* for the rule's API key.

## Behaviour differences from the two-rule setup

Worth knowing before the cutover — none of these are bugs, but they are changes:

- **Detection latency drops** from up to ~9 minutes (Stage 1's 4-minute schedule plus
  Stage 2's 5-minute schedule) to one 5-minute cycle.
- **Repeat notifications drop.** Stage 2 re-read a 30-minute window every 5 minutes, so a
  single breach could re-fire roughly six times. The consolidated rule evaluates a 5-minute
  window every 5 minutes, so a sustained breach fires once per cycle. If the downstream
  ticketing process was relying on that repetition, widen the window or add throttling.
- **The window is 5 minutes, not Stage 1's 4.** This makes the `alarm_reason` text
  ("Average CPU utilization over the last 5 minutes") accurate, and matches the documented
  house default of 5 minutes.
- **`kibana_threshold_alerts` is no longer written.** It was scaffolding, but it also
  happened to record *every* threshold breach, including hosts with the alert disabled. If
  that audit trail is wanted, add a second action to this rule using the existing
  "ITSMA Test Kibana Alerts Threshold Breached connector" — note it would then only record
  breaches that survived the AMDB filter.
- **`hs:std:app-code` now reads `{{context.hits.0._source.labels.app_code}}`**, not
  `labels.app_code.keyword`. Stage 2 read `labels.app_code` out of `kibana_threshold_alerts`,
  where it is mapped as `text` with a `.keyword` subfield that ES|QL surfaces as its own
  column; here the column is produced by the query itself and is plain `keyword`. Rendering
  Stage 2's `.app_code.keyword` path against the new column set produces an **empty string**,
  so this change is required, not cosmetic. Every other field in the emitted event is
  byte-for-byte Stage 2's.
- **The alert identity is now `hostname.keyword, labels.client_code, labels.app_code`** — the
  `BY` fields of the last `STATS`, which is how Kibana derives the alert ID for row-grouped
  ES|QL rules. Stage 2's query had no `STATS`, so its alert ID fell back to *every* returned
  column, which is both unstable and long enough to risk Kibana's "alert ID is very long"
  warning. This is an improvement, but it does mean alert identities do not carry over from
  Stage 2 — expect one round of "new" alerts at cutover.
- If a host has no `labels.app_code`, that field drops out of the alert ID (identity becomes
  host + client code) and `hs:std:app-code` renders as an empty string. Stage 2 behaved the
  same way for missing values.
- `size` is left at 100 to match the existing rules, but **it has no effect on an ES|QL rule** —
  `params.size` is only read on the Query DSL path. The real cap is the alerting framework's
  max-alerts value, which Kibana appends to the query as `| limit <alertLimit>` (1000 by
  default). So the effective ceiling is 1000 breaching hosts per run, not 100, and the
  trailing `| LIMIT 10000` in the query is superseded by that appended limit.
- `threshold: [0]` / `thresholdComparator: ">"` are also inert here: for row-grouped rules the
  executor sets `met = true` unconditionally and never calls the comparator. The `WHERE
  avg_cpu > 98` in the ES|QL is the only threshold, which is why it must stay in the query.

## Files

- `10008_single_stage` — Dev Tools console request. **`POST`**, not `PUT`: this creates a new
  rule, and `rule_type_id` / `consumer` are only accepted on create (they are immutable
  afterwards, and the `PUT` update API rejects them — which is why the existing
  `10008_stage_1` / `10008_stage_2` files, being updates to already-created rules, use `PUT`).
- `10008_single_stage.ndjson` — same rule as a Saved Objects import
  (*Stack Management → Saved Objects → Import*). Imported disabled (`enabled: false`); it
  references the existing connector `69fdac41-8a5e-483d-a506-19660d8550eb`
  ("ITSMA Test Kibana Alerts Final connector"), which must already be present.

## What was verified, and how

A live two-cluster test was not possible in the environment this was written in
(`docker.elastic.co` blobs and `artifacts.elastic.co` are both unreachable there), so the
design was checked against the actual 9.3 source of Elasticsearch and Kibana, and the Kibana
half was executed for real. What that established:

**Against `elastic/elasticsearch` @ `9.3`:**

- The cross-cluster restriction is enforced in `Join.checkRemoteJoin`
  (`x-pack/plugin/esql/.../plan/logical/join/Join.java`), which fails the query for any
  `PipelineBreaker` or coordinator-only node *before* a remote join:
  `"LOOKUP JOIN with remote indices can't be executed after [...]"`. The only
  `PipelineBreaker` implementations are `Aggregate`, `TopN` and `Limit` — i.e. exactly
  `STATS`, `SORT` and `LIMIT`, as documented. **`MvExpand` is not a pipeline breaker**, which
  is what makes `LOOKUP JOIN → MV_EXPAND → WHERE → STATS` legal.
- **Kibana's injected time filter does not break the join.** Kibana passes the time window as
  a Query DSL `filter` on the ES|QL request, and `kibana_sdp_amdb` has no `@timestamp` — if
  that filter reached the lookup index the join would match nothing and the rule would run
  green with zero alerts. It does not, on either path: at resolution,
  `preAnalyzeLookupIndices` is called without the request filter (only
  `preAnalyzeMainIndices` receives it); at execution, `LocalMapper` wraps the join's right
  side in a `FragmentExec`, which is a `LeafExec`, so the
  `transformUp(EsSourceExec.class, …)` that applies the filter in `PlannerUtils.localPlan`
  cannot reach inside it.
- For a remote join, `Mapper.mapBinary` wraps the *whole* join in one `FragmentExec` shipped
  to the remote cluster — consistent with the "lookup index must exist on every cluster"
  requirement.

**Executed, using Kibana 9.3's own code:** the ES|QL ANTLR parser (`@kbn/esql-ast`), the
alert-ID derivation (`getAlertIdFields`/`getEsqlQueryHits` from the `stack_alerts` plugin) and
the action renderer (`renderMustacheObject` from the `actions` plugin) were bundled out of the
`9.3` branch and run against this rule:

- The query parses clean, both as stored and with Kibana's appended `| limit 1000`.
- Each construct was isolated and parsed: dotted `RENAME` targets, multi-assignment `EVAL`,
  arithmetic on an aggregate inside `STATS`, `ROUND` around an aggregate, dotted `BY` fields,
  and the `prod:-metricbeat-7*` exclusion (which parses as prefix `prod` + index
  `-metricbeat-7*`).
- `getAlertIdFields` returns `["hostname.keyword","labels.client_code","labels.app_code"]`,
  confirming Kibana's parser follows the trailing `RENAME` through to the final column names,
  and that two breaching hosts yield two distinct alert IDs with no duplicate-ID warning.
- Kibana builds `_source` **flat**, with literal dotted keys
  (`{"hostname.keyword": "...", "labels.client_code": "..."}`) — but the action renderer runs
  `expandDottedKeys` first, turning those into nested objects, so
  `{{context.hits.0._source.hostname.keyword}}` resolves correctly. This is worth knowing
  because it is the renderer, not the rule, that makes the dotted template paths work.
- The full action document renders with **no unresolved `{{...}}` tags and no empty fields**.
  The rendered ticket is in the commit message for this change.

The two defects this turned up — the `alarm_reason` float noise, and Stage 2's
`.app_code.keyword` path rendering empty — were fixed before committing and the simulation
re-run clean.

Still unverified, and only a live cluster can settle it: that `kibana_sdp_amdb` on `prod`
resolves and joins as expected, and the real shape of that index's mapping (see the collision
note above). Both are covered by the cutover steps below.

## Suggested cutover

1. Create `kibana_sdp_amdb` as a lookup-mode index on `prod` and sync it.
2. Create the new rule from `10008_single_stage`, leave it disabled.
3. Open it in the rule editor and run **Test query**; confirm it returns the breaching,
   AMDB-enabled hosts you expect. An empty result with no error is the signature of a
   missing/misconfigured lookup index on `prod` (see prerequisite 2).
4. Enable it and compare its output in `kibana_alerts` against Stage 2's for a cycle or two.
5. Disable Stage 1 and Stage 2. Once the pattern is proven here, repeat for 10004, 10050 and
   the rest, then retire `kibana_threshold_alerts`.

## References

Version-specific documentation for the 9.x behaviour this rule depends on:

- **Remote `LOOKUP JOIN`, GA 9.2.0** — [`LOOKUP JOIN` → Cross-cluster support (9.3 docs)](https://github.com/elastic/elasticsearch/blob/9.3/docs/reference/query-languages/esql/esql-lookup-join.md#cross-cluster-support):
  *"Remote lookup joins are supported in cross-cluster queries. The lookup index must exist
  on all remote clusters being queried, because each cluster uses its local lookup index
  data. This follows the same pattern as remote mode Enrich."* — marked
  `applies_to: stack: ga 9.2.0`.
- **The `STATS` / `SORT` / `LIMIT` restriction** — [`LOOKUP JOIN` → Limitations (9.3 docs)](https://github.com/elastic/elasticsearch/blob/9.3/docs/reference/query-languages/esql/esql-lookup-join.md#limitations):
  *"Cross-cluster `LOOKUP JOIN` can not be used after aggregations (`STATS`), `SORT` and
  `LIMIT` commands, and coordinator-side `ENRICH` commands."* Same page:
  *"Cross cluster search is unsupported in versions prior to `9.2.0`. Both source and lookup
  indices must be local for these versions."* — this is the sentence that justified the
  two-rule workaround on 8.x.
- **9.2.0 release note, "Add remote index support to `LOOKUP JOIN`"** —
  [Elasticsearch 9.2.0 release notes](https://github.com/elastic/elasticsearch/blob/9.3/docs/release-notes/index.md)
  ([PR #129013](https://github.com/elastic/elasticsearch/pull/129013)).
- **Concrete index name only** — [`LOOKUP JOIN` command reference (9.3)](https://github.com/elastic/elasticsearch/blob/9.3/docs/reference/query-languages/esql/_snippets/commands/layout/lookup-join.md):
  *"This must be a specific index name — wildcards, aliases, and remote cluster references
  are not supported. Indices used for lookups must be configured with the `lookup` index
  mode."*
- **Coordinator mode is 9.6+** — [ES|QL across clusters → LOOKUP JOIN across clusters (current docs)](https://github.com/elastic/elasticsearch/blob/main/docs/reference/query-languages/esql/esql-cross-clusters.md#lookup-join-across-clusters):
  *"`applies_to` stack: ga 9.6+ — If the lookup index is missing from one or more remote
  clusters, use coordinator mode to join against a local cluster lookup index copy."*
  Confirms that on 9.3 the copy on `prod` is mandatory.
- **CCS prerequisites and `skip_unavailable` behaviour** — [ES|QL across clusters (9.3 docs)](https://github.com/elastic/elasticsearch/blob/9.3/docs/reference/query-languages/esql/esql-cross-clusters.md).
- **Multi-typed fields / union types** — [Using ES|QL to query multiple indices (9.3 docs)](https://github.com/elastic/elasticsearch/blob/9.3/docs/reference/query-languages/esql/esql-multi-index.md#union-types)
  and [ES|QL limitations (9.3)](https://github.com/elastic/elasticsearch/blob/9.3/docs/reference/query-languages/esql/limitations.md)
  (also the source of the 10,000-row `LIMIT` ceiling).
- **Rule API create vs update** — [Create a rule](https://www.elastic.co/docs/api/doc/kibana/operation/operation-post-alerting-rule-id)
  / [Update a rule](https://www.elastic.co/docs/api/doc/kibana/operation/operation-put-alerting-rule-id).
- **Elasticsearch Labs, "ES|QL in 9.2: Smart Lookup Joins and time-series support"** —
  https://www.elastic.co/search-labs/blog/esql-elasticsearch-9-2-multi-field-joins-ts-command
  (narrative overview: *"When Lookup Join went GA in 8.19 and 9.1, it lacked Cross-Cluster
  Search support… in 9.2, LOOKUP JOIN now seamlessly integrates with CCS"*).

The GitHub links point at the `9.3` branch of `elastic/elasticsearch`, which is the source of
truth for the rendered pages at `elastic.co/docs` for our version — use those if you want the
exact wording as of 9.3 rather than whatever `current` has moved on to.
