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

## Two ways to do this on 9.3

`kibana_sdp_amdb` currently exists **only on the local/CCS cluster**, not on `prod`. That is
the deciding fact, because a 9.3 remote `LOOKUP JOIN` joins against each queried cluster's own
copy of the lookup index. So there are exactly two routes, and both are provided here:

| | `10008_single_stage` (LOOKUP JOIN) | `10008_single_stage_enrich_variant` (ENRICH) |
|---|---|---|
| Needs on `prod` | `kibana_sdp_amdb` as a lookup-mode index, populated | **nothing** |
| Needs on local | nothing new | a keyword-reindexed enrich source + policy, executed |
| `hostname` handling | joins on the `.keyword` multi-field directly | needs a top-level `keyword` copy (reindex) |
| Config freshness | **live** — reads the index every run | snapshot — stale until reindex + policy re-executed |
| Join position | before `STATS`, on raw metric docs | after `STATS`, on breaching hosts only |
| Work done | joins every metric doc in the window | joins a handful of rows |

**Recommendation:** if you can get `kibana_sdp_amdb` onto `prod`, use the `LOOKUP JOIN` rule —
it reads live config, which is the property the original design was chosen for ("your
enrichment data changes frequently" is exactly the case for preferring `LOOKUP JOIN` over
`ENRICH`). If you cannot, or not soon, the `ENRICH` variant works today with no changes to
`prod`, at the cost of a bit more local setup and a re-execute whenever AMDB config changes.

One asymmetry, discovered the hard way (see `kibana_sdp_amdb_enrich_policy`): **enrich cannot
match on `hostname.keyword`.** The index maps `hostname` as `text` with a `.keyword`
multi-field, and enrich's `match_field` validation only walks `object` fields — it throws
*"The [hostname] field must be regular object but was [text]"* on a multi-field subfield. Nor
can it match on the bare `text` field, because enrich matches with a term query and an analyzed
hostname never matches whole. So the ENRICH route requires reindexing the three needed fields
into a small keyword-typed source first. **LOOKUP JOIN has no such limitation** — it resolves
`hostname.keyword` directly — which is a genuine point in the join route's favour beyond the
freshness argument.

The enrich route is still *cheaper at query time*: because coordinator-mode `ENRICH` is not
subject to the "no remote join after `STATS`" restriction, it runs after the aggregation and
enriches only the breaching hosts, whereas the join runs on every raw metric document in the
window.

Both emit a byte-identical event and both were verified the same way (see below).

## The consolidated query (LOOKUP JOIN)

```esql
FROM prod:metricbeat-*
| WHERE @timestamp > NOW() - 5 minutes
| EVAL event_dataset = event.dataset::keyword, cpu_pct = system.cpu.total.norm.pct::double
| WHERE event_dataset == "system.cpu" AND cpu_pct IS NOT NULL
| RENAME host.hostname AS hostname.keyword
| LOOKUP JOIN kibana_sdp_amdb ON hostname.keyword
| MV_EXPAND group
| WHERE group == "10008" AND alert_status == "enabled"
| STATS avg_cpu = AVG(cpu_pct) * 100 BY hostname.keyword, labels.client_code, labels.app_code
| WHERE avg_cpu > 98
| EVAL current_value = ROUND(avg_cpu, 2)
| DROP avg_cpu
| LIMIT 10000
```

## The ENRICH variant

```esql
FROM prod:metricbeat-*
| WHERE @timestamp > NOW() - 5 minutes
| EVAL event_dataset = event.dataset::keyword, cpu_pct = system.cpu.total.norm.pct::double
| WHERE event_dataset == "system.cpu" AND cpu_pct IS NOT NULL
| STATS avg_cpu = AVG(cpu_pct) * 100 BY host.hostname, labels.client_code, labels.app_code
| WHERE avg_cpu > 98
| ENRICH _coordinator:kibana_sdp_amdb_policy ON host.hostname WITH group, alert_status
| MV_EXPAND group
| WHERE group == "10008" AND alert_status == "enabled"
| EVAL current_value = ROUND(avg_cpu, 2)
| RENAME host.hostname AS hostname.keyword
| DROP avg_cpu
| LIMIT 10000
```

## The multi-typed field problem, and why the casts are there

Both queries cast `event.dataset` and `system.cpu.total.norm.pct` before using them. This is not
defensive styling — without it the rule does not run. Over `prod:metricbeat-*` both fields are
mapped with **conflicting types across indices**, so ES|QL resolves them to type `unsupported`,
and then:

- `event_dataset == "system.cpu"` fails with *"Invalid input types for `==`. Received
  (unsupported, keyword)"*.
- `AVG(system.cpu.total.norm.pct)` fails with *"Invalid input types for AVG. Received
  (unsupported)"* — `AVG` only accepts `aggregate_metric_double`, `double`,
  `exponential_histogram`, `integer` and `long`.

This is the same root cause as the `verification_exception` Stage 1 has been failing with. An
earlier revision here tried to fix it by excluding the legacy indices with
`prod:-metricbeat-7*`; that was **not sufficient**, because the original error named three 7.x
indices *"and [3] other indices"* whose names are not visible. Casting fixes it regardless of
which indices conflict, which is why the exclusion has been dropped in favour of casts — the
legacy indices are harmless now, and a 5-minute time filter skips them at the shard level
anyway.

Casting is the documented remedy for multi-typed fields ("union types"), and it works as long as
the *only* reference to the raw field is the conversion itself — hence the raw names appear
nowhere else in either query. You may still see a residual **warning** that the raw field
"cannot be retrieved, it is unsupported or not indexed"; that refers to the uncast field and is
expected. Warnings do not block; errors do.

`cpu_pct IS NOT NULL` is not strictly required — `AVG` skips nulls — but it drops non-CPU
documents before the join, which matters in the LOOKUP JOIN variant where the join runs on raw
documents.

`DIAGNOSTICS` has the `_field_caps` call that shows exactly which fields conflict and in which
indices, plus what to do if `host.hostname` or the `labels.*` fields turn out to conflict too
(they did not appear in the reported errors, so they are presumably consistent).

`_coordinator` forces the enrich onto the local cluster, which is what makes this work with the
policy only existing there. The trailing `RENAME` produces the same `hostname.keyword` column
the action template reads, so the action is unchanged. The enrich policy's `match_field` must be
`hostname.keyword` (see `kibana_sdp_amdb_enrich_policy`), because `hostname` is `text` in the
config index and an enrich match field has to be a keyword.

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
- **No collision guard is needed** — an earlier revision copied `labels.client_code` /
  `labels.app_code` to prefixed names across the join, because fields from the lookup index
  **override** same-named columns on the left and `client_code`/`app_code` looked like
  plausible AMDB field names. The actual mapping settles it: `kibana_sdp_amdb` has **no
  `labels.*` fields at all**. Its 51 join-visible columns are `alert_status`, `app_code`,
  `client`, `datacenter`, `domain`, `event_type`, `group`, `hostname`, `maintenance.*`,
  `node`, `os`, `query.*`, `record.*`, `script.*`, `svc-operator`, `svc-software-owner`,
  `svc_operator`, `svc_software_owner`, `target`, `type` and their `.keyword` siblings. It
  does have `app_code` and `client`, but those are different names from `labels.app_code` and
  `labels.client_code`, and ES|QL columns are flat dotted names, so nothing is shadowed. The
  guard was dropped and the `STATS` groups the metricbeat fields directly. The one genuine
  overlap is `hostname.keyword` itself, which is the join key — same value on both sides, so
  the override is a no-op.
- **`RENAME host.hostname AS hostname.keyword`** — unchanged from Stage 2, and the mapping
  confirms exactly why it is needed: `hostname` in `kibana_sdp_amdb` is `text` with a
  `.keyword` subfield, so the only keyword-typed join key available on the lookup side is
  `hostname.keyword`, and the left side must present a column with that literal name.
  `host.hostname` is `keyword` in metricbeat, the same type Stage 2 joined from, so join
  behaviour is identical.
- **`group` and `alert_status` are both `text` with `.keyword` subfields.** The rule filters
  on the `text` fields, exactly as Stage 2 does, so this is proven behaviour and was left
  alone. If the join turns out to be slow, switching to `MV_EXPAND group.keyword` /
  `WHERE group.keyword == "10008" AND alert_status.keyword == "enabled"` is the lever to try:
  `.keyword` has doc values and can be pushed down, whereas filtering `text` means loading
  from `_source`, and in this rule that filter now runs on raw metric documents rather than
  on a handful of pre-aggregated rows. That is an unmeasured optimisation, so it is not
  applied here — the `ENRICH` variant is the better answer if volume turns out to be a problem,
  since it filters after the aggregation.
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

**For the ENRICH variant:** run `kibana_sdp_amdb_enrich_policy` on the local cluster. It is no
longer a one-liner — enrich cannot match on the `hostname.keyword` multi-field (see the file for
the full reason), so the steps are: reindex `hostname`/`group`/`alert_status` into a small
keyword-typed source (`kibana_sdp_amdb_enrich`), create the policy on that source, and execute
it. Nothing on `prod`. Skip to point 3.

**For the LOOKUP JOIN rule:**

1. **`kibana_sdp_amdb` must exist on the `prod` cluster** with `index.mode: lookup`.
   `kibana_sdp_amdb_prod_lookup_index` has the ready-to-run request, with the join-relevant
   fields typed to match the local copy, plus the sync options and their caveats. The short
   version: reindex-from-remote or dual-write are the straightforward routes; CCR is
   attractive but runs the wrong way round here (CCR pulls, so `prod` would be the follower
   and would need the local cluster configured as a remote on `prod`, which the existing
   local→`prod` CCS link does not give you).

   Only `hostname`, `group` and `alert_status` are actually read, so the copy does not need
   the other ~45 AMDB fields — and keeping it narrow removes any chance of a config field
   shadowing a metricbeat column. Verify with `GET kibana_sdp_amdb/_settings` on `prod`;
   lookup-mode indices are always single-sharded.

   **The copy must contain data, not just exist.** Every data row in this query comes from
   `prod`, so every join is evaluated against *prod's* copy — the local one is never consulted,
   because no local index appears in the `FROM`. An empty-but-present lookup index therefore
   left-joins every row to nulls, `WHERE group == "10008" AND alert_status == "enabled"` drops
   them all, and the rule runs green with **zero alerts and no error**. A missing index gives
   you a loud error; an empty one gives you silence, which is worse.

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

- `10008_single_stage` — the LOOKUP JOIN rule, as a Dev Tools console request. **`POST`**, not
  `PUT`: this creates a new rule, and `rule_type_id` / `consumer` are only accepted on create
  (they are immutable afterwards, and the `PUT` update API rejects them — which is why the
  existing `10008_stage_1` / `10008_stage_2` files, being updates to already-created rules,
  use `PUT`).
- `10008_single_stage.ndjson` — same rule as a Saved Objects import
  (*Stack Management → Saved Objects → Import*). Imported disabled (`enabled: false`); it
  references the existing connector `69fdac41-8a5e-483d-a506-19660d8550eb`
  ("ITSMA Test Kibana Alerts Final connector"), which must already be present.
- `kibana_sdp_amdb_prod_lookup_index` — creates the lookup-mode copy on `prod`, with sync
  options and caveats. Only needed for the LOOKUP JOIN rule.
- `10008_single_stage_enrich_variant` — the alternative rule that needs nothing on `prod`.
- `kibana_sdp_amdb_enrich_policy` — the keyword reindex + enrich policy the variant depends on,
  including why the multi-field forces the reindex and the re-execution requirement. Only needed
  for the ENRICH variant.
- `DIAGNOSTICS` — how to confirm each of the editor errors on your own cluster: the
  `_field_caps` call that identifies mapping conflicts, the `index.mode` checks for local and
  `prod`, and standalone `_query` calls that isolate the metricbeat half from the AMDB half.

Pick one rule or the other; they emit the same event to the same connector, so running both
enabled at once would double every ticket.

## Incidental: fields the AMDB index already has

Not changed here, because it is a scope decision rather than a migration concern, but worth
knowing now that the mapping is visible. The action currently hardcodes several tags to `"NA"`
that `kibana_sdp_amdb` actually carries per host: `svc-operator` / `svc_operator`,
`svc-software-owner` / `svc_software_owner`, and `app_code` (the config index's own view of the
app code, as opposed to `labels.app_code` from the metric document). There is also a
`maintenance` object with real `utc_start` / `utc_end` dates and an `sdp_ticket_id`, while
`hs:app:maintenance-window` is hardcoded `"true"`.

With the `LOOKUP JOIN` rule these are already joined in and could be carried into the ticket by
adding them to the `STATS ... BY` (or via `VALUES()`); with the `ENRICH` variant they would need
adding to the policy's `enrich_fields`. Worth a separate conversation about which source of
truth the ticketing process should use — this change deliberately keeps the emitted event
identical to Stage 2's so the migration is comparable.

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
- **Coordinator-mode `ENRICH` carries none of the remote restrictions**, which is what makes
  the variant legal after `STATS`: in `Enrich.java`, `postAnalysisVerification` and
  `checkForPlansForbiddenBeforeRemoteEnrich` both gate on `mode == Mode.REMOTE`, so
  `Mode.COORDINATOR` is unconstrained.

**Executed, using Kibana 9.3's own code:** the ES|QL ANTLR parser (`@kbn/esql-ast`), the
alert-ID derivation (`getAlertIdFields`/`getEsqlQueryHits` from the `stack_alerts` plugin) and
the action renderer (`renderMustacheObject` from the `actions` plugin) were bundled out of the
`9.3` branch and run against this rule:

- Both queries parse clean, as stored and with Kibana's appended `| limit 1000`.
- Each construct was isolated and parsed: dotted `RENAME` targets, multi-assignment `EVAL`,
  arithmetic on an aggregate inside `STATS`, `ROUND` around an aggregate, dotted `BY` fields,
  and the `prod:-metricbeat-7*` exclusion (which parses as prefix `prod` + index
  `-metricbeat-7*`).
- `getAlertIdFields` returns `["hostname.keyword","labels.client_code","labels.app_code"]` for
  **both** variants, confirming Kibana's parser follows a trailing `RENAME` through to the
  final column names (the `ENRICH` variant relies on this for `host.hostname` →
  `hostname.keyword`), and that two breaching hosts yield two distinct alert IDs with no
  duplicate-ID warning.
- The collision analysis was run mechanically against the transcribed mapping: of the 51
  columns `kibana_sdp_amdb` contributes to the join, the only one that overlaps a column the
  rule depends on is `hostname.keyword`, the join key itself. No `labels.*` field exists in the
  config index, which is what allowed the guard columns to be removed.
- Kibana builds `_source` **flat**, with literal dotted keys
  (`{"hostname.keyword": "...", "labels.client_code": "..."}`) — but the action renderer runs
  `expandDottedKeys` first, turning those into nested objects, so
  `{{context.hits.0._source.hostname.keyword}}` resolves correctly. This is worth knowing
  because it is the renderer, not the rule, that makes the dotted template paths work.
- The full action document renders, for both variants, with **no unresolved `{{...}}` tags and
  no empty fields**, and identical output. The rendered ticket is in the commit message.

The two defects this turned up — the `alarm_reason` float noise, and Stage 2's
`.app_code.keyword` path rendering empty — were fixed before committing and the simulation
re-run clean.

**Kibana's own validator was later run against the rule** (`validateQuery` from
`@kbn/esql-ast`, same 9.3 branch), which reproduced the three reported editor errors exactly,
including the received types:

```
Line 2: Invalid input types for ==.  Received (unsupported, keyword)
Line 4: "kibana_sdp_amdb" is not a valid JOIN index. Please use a "lookup" mode index.
Line 7: Invalid input types for AVG. Received (unsupported)
```

Isolating them: lines 2 and 7 are caused solely by the two multi-typed fields, and adding the
casts takes the LOOKUP JOIN rule from **2 errors to 0** under the identical harness.

**Line 4 is accurate and independent — it is reporting that `kibana_sdp_amdb` is not a lookup
index _on `prod`_.** An earlier revision of this file suggested it might be a false positive
about the local index; that was wrong. Traced through the Kibana 9.3 source:

```
getJoinIndices(query)
  -> getRemoteClustersFromESQLQuery(query)          => ["prod"]
  -> GET /internal/esql/autocomplete/join/indices?remoteClusters=prod
  -> EsqlService.getIndicesByIndexMode("lookup", "prod")
       resolves index.mode=lookup over ["*", "prod:*"]
  -> getListOfCCSIndices(["prod"], indices)
       keeps only "cluster:index" entries for the listed clusters;
       entries without a colon -- every LOCAL index -- are skipped outright
```

Once the query reads from a remote cluster, the editor's list of valid JOIN indices contains
only lookup indices that exist **on that remote cluster**. The local copy is excluded by
design, mirroring what Elasticsearch does: each cluster joins against its own copy. So a local
`index.mode: lookup` — which is correctly set — cannot satisfy this check, and the error clears
only when the index exists on `prod` (or when you switch to the ENRICH variant, which has no
join-index check at all).

One validator gap worth knowing, since it affects the ENRICH variant: `validateQuery` does not
add a policy's `enrich_fields` to its column list, so it reports `Unknown column "group"` after
the `ENRICH` even when the policy resolves. Confirmed against Kibana's own test fixtures — a
deliberately wrong policy name adds an `Unknown policy` error, so policy resolution works while
the enrich columns are simply never registered. Treat that message as noise and rely on
**Test query** / `POST _query`; `DIAGNOSTICS` has the calls.

Still unverified, and only a live cluster can settle it:

- that `kibana_sdp_amdb` on `prod` resolves and joins as expected (LOOKUP JOIN route), or that
  the enrich policy resolves in `_coordinator` mode (ENRICH route);
- that ES|QL equality against the `text`-mapped `group` / `alert_status` behaves as it does in
  Stage 2 today. Stage 2 runs the identical predicates against the identical fields, so this is
  as close to proven as it gets without a cluster, but it is inherited rather than tested here;
- the cost of joining raw metric documents at your fleet's volume, which is the one thing that
  might push you to the ENRICH variant on performance grounds rather than on the prod-copy
  question.

All of these are covered by the cutover steps below.

## Suggested cutover

0. Decide the route (see the table above). Everything below is the same either way apart from
   step 1.
1. Set up the dependency:
   - *LOOKUP JOIN:* run `kibana_sdp_amdb_prod_lookup_index` on `prod`, load the data, and put
     the sync in place.
   - *ENRICH:* run `kibana_sdp_amdb_enrich_policy` on the local cluster, including the
     `_execute`, and schedule the re-execution.
2. Create the rule (`10008_single_stage` or `10008_single_stage_enrich_variant`), leave it
   disabled.
3. Open it in the rule editor and run **Test query**; confirm it returns the breaching,
   AMDB-enabled hosts you expect. **An empty result with no error is the signature of a missing
   dependency** — a lookup index that is absent or not in lookup mode on `prod`, or an enrich
   policy that was never executed. Do not read "no rows" as "no breaches" until you have
   confirmed the dependency resolves; cross-check against a host you know is breaching.
4. Enable it and compare its output in `kibana_alerts` against Stage 2's for a cycle or two.
   Expect one round of "new" alerts, since alert identities do not carry over.
5. Check the rule's execution duration in *Stack Management → Rules* before trusting it at
   steady state — for the LOOKUP JOIN route this is where excessive join volume would show up.
6. Disable Stage 1 and Stage 2. Once the pattern is proven here, repeat for 10004, 10050 and
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
- **Coordinator-mode `ENRICH`** (the basis of the variant) — [ES|QL across clusters → Enrich with coordinator mode (9.3)](https://github.com/elastic/elasticsearch/blob/9.3/docs/reference/query-languages/esql/esql-cross-clusters.md#esql-enrich-coordinator):
  *"{{esql}} provides the enrich `_coordinator` mode to force {{esql}} to execute the enrich
  command on the local cluster. Use this mode when the enrich policy is not available on the
  remote clusters or maintaining consistency of enrich indices across clusters is
  challenging."* — which is precisely the situation here. The same page documents that it is
  `_remote` enrich, not `_coordinator`, that cannot follow `STATS`.
- **`LOOKUP JOIN` vs `ENRICH` trade-off** — [Join data with LOOKUP JOIN → Compare with ENRICH (9.3)](https://github.com/elastic/elasticsearch/blob/9.3/docs/reference/query-languages/esql/esql-lookup-join.md#compare-with-enrich):
  prefer `LOOKUP JOIN` when *"your enrichment data changes frequently"* and you *"want to avoid
  index-time processing"* — the reason the original design reached for it, and the reason the
  enrich variant is the fallback rather than the default.
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
