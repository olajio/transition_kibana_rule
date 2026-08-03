# 10008 walkthrough — cutting over from Stage 1 + Stage 2 to a single rule

A step-by-step you can hold your finger on. It uses **10008** as the concrete example; the
same procedure works for 10004, 10050, and every other AMDB rule once you have this one live.
Everything the UI covers has a Dev Tools equivalent, kept alongside — do it in the console if
you'd rather, but the goal here is the manual path.

## What you'll do

1. **Prep** the local cluster with `kibana_sdp_amdb_enrich` and an enrich policy (Dev Tools).
2. **Sanity-check** with three quick reads that all the prerequisites are in place.
3. **Create the consolidated rule** in the Kibana UI, leaving it disabled.
4. **Test-query** the rule to confirm it returns the right rows.
5. **Run it side-by-side** with Stage 2 for a cycle or two.
6. **Disable the old Stage 1 + Stage 2 pair.**
7. **Verify** nothing else broke.

This walkthrough uses the **ENRICH variant** (`10008_single_stage_enrich_variant`), which is
what fits your 3-remote-cluster setup best — nothing to create on `prod`, and one enrich
policy to maintain locally instead of three synced lookup indices. Notes at each step call
out how the two `LOOKUP JOIN` variants (`10008_single_stage` on original `kibana_sdp_amdb`,
`10008_single_stage_enrich_index` on the new keyword index) differ if you'd rather go that way
later. See `10008_single_stage_README.md` for the route trade-offs.

---

## Prerequisites — do these once

You need three things in place on the **local (CCS) cluster** before creating the rule:

- **The `kibana_sdp_amdb_enrich` index** (keyword-mapped, `index.mode: lookup`), populated
  with your AMDB config.
- **The enrich policy** `kibana_sdp_amdb_policy` built on that index.
- **The connector** the rule will use — `ITSMA Test Kibana Alerts Final connector`
  (`id: 69fdac41-8a5e-483d-a506-19660d8550eb`). Already exists in your Kibana; nothing to do.

Every command in this section runs against the **local** cluster.

### P1. Create `kibana_sdp_amdb_enrich`

In Kibana → **Dev Tools → Console**, paste and run everything in
`kibana_sdp_amdb_enrich_index`. The important bits:

```
PUT kibana_sdp_amdb_enrich
{
  "settings": { "index.mode": "lookup" },
  "mappings": {
    "properties": {
      "hostname":     { "type": "keyword" },
      "group":        { "type": "keyword" },
      "alert_status": { "type": "keyword" }
    }
  }
}

POST _reindex
{
  "source": { "index": "kibana_sdp_amdb",
              "_source": ["hostname", "group", "alert_status"] },
  "dest":   { "index": "kibana_sdp_amdb_enrich" }
}
```

Then confirm the copy landed:

```
GET kibana_sdp_amdb_enrich/_count
GET kibana_sdp_amdb_enrich/_search
{ "size": 3, "_source": ["hostname", "group", "alert_status"] }
```

- `_count` should match `kibana_sdp_amdb`.
- `hostname` values should look like real hostnames (`CS01-10VLCNPY24`, not tokens like
  `cs01`).
- `group` should be a JSON **array** where a host has multiple entries.

If any of that's off, stop here — the rule won't work if the source is wrong.

### P2. Create and execute the enrich policy

Run everything in `kibana_sdp_amdb_enrich_policy`:

```
PUT _enrich/policy/kibana_sdp_amdb_policy
{
  "match": {
    "indices": "kibana_sdp_amdb_enrich",
    "match_field": "hostname",
    "enrich_fields": ["group", "alert_status"]
  }
}

PUT _enrich/policy/kibana_sdp_amdb_policy/_execute
```

Confirm both:

```
GET _enrich/policy/kibana_sdp_amdb_policy
GET _enrich/_stats
```

The policy should list `enrich_fields: ["group", "alert_status"]`. `_stats` should show the
policy has been executed (a non-empty `executing_policies` history) and there should be a
`.enrich-kibana_sdp_amdb_policy-<timestamp>` index behind the scenes.

Read `10008_single_stage_README.md` → *"how does this work" answer* for why we route through
a policy instead of querying the index directly. Short version: `ENRICH` reads a snapshot the
policy materializes at `_execute` time. This means: **every time AMDB config changes, re-run
`PUT _enrich/policy/kibana_sdp_amdb_policy/_execute`** — otherwise the rule keeps using stale
config. Wire that into whatever maintains AMDB, or schedule it.

---

## Sanity checks — read only, run these each time before you create a new consolidated rule

These are quick, cheap, and catch 90% of what would otherwise be an unexplained empty result.

### S1. `event.dataset` and the fields the rule uses are usable in ES|QL

```
GET prod:metricbeat-*/_field_caps?fields=event.dataset,system.cpu.total.norm.pct,host.hostname,labels.client_code,labels.app_code
```

Each field's response block should show **one** `types` entry (e.g. just `keyword`, or just
`float`). More than one type block on a field means it's mapped inconsistently across the
matched indices — the query will fail with *"Cannot use field ... due to ambiguities"* unless
that field is cast up front. The current query already casts the five known offenders; if
`_field_caps` shows a new one, extend the EVAL block the same way (see
`10008_single_stage_README.md` → *"The multi-typed field problem, and why the casts are there"*).

### S2. The enrich source has this host

Pick any host you know is currently alerting via Stage 2 and expect to see in the new rule:

```
GET kibana_sdp_amdb_enrich/_search
{
  "query": { "term": { "hostname": "CS01-10VLCNPY24" } }
}
```

Should return exactly one hit with `group` containing `"10008"` and `alert_status: "enabled"`.
If no hit, the reindex missed it. If the hit's hostname doesn't exactly match what
`prod:metricbeat-*` emits as `host.hostname`, the enrich will silently fail to match — check
casing and short-name-vs-FQDN.

### S3. The ES|QL query returns rows *before* you wire it into a rule

```
POST _query?format=txt
{
  "query": "FROM prod:metricbeat-* | WHERE @timestamp > NOW() - 5 minutes | EVAL event_dataset = event.dataset::keyword, cpu_pct = system.cpu.total.norm.pct::double, hostname = host.hostname::keyword, client_code = labels.client_code::keyword, app_code = labels.app_code::keyword | WHERE event_dataset == \"system.cpu\" AND cpu_pct IS NOT NULL | STATS avg_cpu = AVG(cpu_pct) * 100 BY hostname, client_code, app_code | WHERE avg_cpu > 98 | ENRICH _coordinator:kibana_sdp_amdb_policy ON hostname WITH group, alert_status | MV_EXPAND group | WHERE group == \"10008\" AND alert_status == \"enabled\" | LIMIT 10"
}
```

Three outcomes:

- **Rows come back** — good, the query works; the values in `hostname`, `client_code`,
  `app_code`, `avg_cpu` are what will end up in the tickets. Move on to the UI.
- **No rows, no error** — either no host is actually breaching 98% right now (plausible;
  run it again in a few minutes, or drop the threshold to `> 0` temporarily as a smoke
  test), or the enrich didn't match anything (S2 would already have caught the common cause).
  If no CPU breach in prod is available for testing, this is what "no rows" should look like
  and you can still proceed to create the rule.
- **Any error** — stop and read the message. If it says *"Cannot use field ... due to
  ambiguities"* on a new field, cast that field in the EVAL block and re-run. `DIAGNOSTICS`
  has more.

---

## Create the rule in the Kibana UI

Everything below is on the **local (CCS) cluster**'s Kibana, in whatever space you keep alerts.

### Step 1. Open the rule editor

- **Stack Management → Alerts and Insights → Rules**
- Top right: **Create rule**
- Pop-up **Select rule type** → search or scroll to **Elasticsearch query** (under *Stack Alerts*)
- **Create rule** on the *Elasticsearch query* card

### Step 2. Rule definition

Fill in this tab first — the fields are:

1. **Query type toggle** — switch from *Query DSL* / *KQL/Lucene* to **ESQL** (bottom-right of
   the query-type row).

2. **ES|QL query** — paste the query from `10008_single_stage_enrich_variant`:

   ```esql
   FROM prod:metricbeat-*
   | WHERE @timestamp > NOW() - 5 minutes
   | EVAL event_dataset = event.dataset::keyword,
          cpu_pct       = system.cpu.total.norm.pct::double,
          hostname      = host.hostname::keyword,
          client_code   = labels.client_code::keyword,
          app_code      = labels.app_code::keyword
   | WHERE event_dataset == "system.cpu" AND cpu_pct IS NOT NULL
   | STATS avg_cpu = AVG(cpu_pct) * 100 BY hostname, client_code, app_code
   | WHERE avg_cpu > 98
   | ENRICH _coordinator:kibana_sdp_amdb_policy ON hostname WITH group, alert_status
   | MV_EXPAND group
   | WHERE group == "10008" AND alert_status == "enabled"
   | EVAL current_value = ROUND(avg_cpu, 2)
   | RENAME hostname    AS hostname.keyword,
            client_code AS labels.client_code,
            app_code    AS labels.app_code
   | DROP avg_cpu
   | LIMIT 10000
   ```

   The editor may draw red squiggles under `group` and `alert_status` saying *Unknown
   column*. That is a **known Kibana validator bug** — it doesn't track fields that `ENRICH`
   adds. Ignore it; the server executes fine. Verified via S3 above.

   > *LOOKUP JOIN variants:* use the query from `10008_single_stage` or
   > `10008_single_stage_enrich_index` instead. Same tab, same box.

3. **Test query** button (right of the query box) — click it. It runs the query with your
   time-window setting applied. A summary count should appear. If S3 came back with rows, this
   should too.

4. **Select a time field** — pick **`@timestamp`** from the dropdown.

5. **Select alert group** — pick **Create an alert for each row**. (The alternative "Create
   an alert if matches are found" fires one aggregate alert instead of one per host, which
   isn't what we want.)

6. **Time window** — **5 minutes**. Match the query's `NOW() - 5 minutes` window; a mismatch
   here doesn't break the query (Kibana applies its own filter *on top*) but can hide bugs.

### Step 3. Rule schedule

- **Check every** — **5 minutes**.
- **Advanced options** → **Alert delay** — set *Alert active on* to **1** consecutive
  check. This suppresses one-shot spikes; unchanged from Stage 2.

### Step 4. Actions

1. **Add action** → search for and select **ITSMA Test Kibana Alerts Final connector**
   (the connector `id: 69fdac41-8a5e-483d-a506-19660d8550eb`). If it isn't listed, that
   connector hasn't been created in this space — stop and reconcile.

2. **Action frequency** — **On each check** (default) with **Run when: Query matched**.

3. **Documents to index** — this is the ticket body. Paste **exactly** this JSON (it's the
   same object as in `10008_single_stage_enrich_variant`):

   ```json
   [
     {
       "target": "{{context.hits.0._source.hostname.keyword}}",
       "event_type": "10008",
       "event_uuid": "on-prem_10008_{{context.hits.0._source.hostname.keyword}}",
       "alarm_name": "10008 - CPU Utilization > 98%",
       "trace": { "id": "{{alert.uuid}}" },
       "tags_to_exclude": "NA",
       "cloud": { "region": "NA", "account": { "name": "NA", "id": "NA" } },
       "labels": { "client_code": "{{context.hits.0._source.labels.client_code}}" },
       "@timestamp": "{{date}}",
       "amdb_link": "https://hedgeservcorp.sharepoint.com/sites/globaltechnology/amdb/sitepages/10008---system-cpu-utilization---98-.aspx?web=1",
       "dashboard_link": "https://287d86a4b1184182b340bd5074cdfd7e.us-east-1.aws.found.io:9243/s/information-technology/app/r/s/wmCdY",
       "service": { "type": "kibana_alerts", "name": "cpu_utilization" },
       "alarm_tags": {
         "hs:app:maintenance-window": "true",
         "hs:std:app-name": "NA",
         "hs:app:amdb": "10008",
         "hs:app:sdp-priority": "2 - High",
         "hs:std:svc-operator": "NA",
         "hs:std:svc-software-owner": "NA",
         "hs:app:monitored": "true",
         "hs:std:app-code": "{{context.hits.0._source.labels.app_code}}",
         "hs:app:ticket-group": "Monitoring and Analytics - Testing"
       },
       "alarm_reason": "Average CPU utilization over the last 5 minutes is {{context.hits.0._source.current_value}}"
     }
   ]
   ```

   Two easy-to-miss things:

   - The outer `[ … ]` is a JSON array. The action expects an array of documents; here it's
     just one.
   - `{{context.hits.0._source.hostname.keyword}}` looks odd because there is no *literal*
     `.keyword` column any more (the trailing `RENAME` in the query produces one named
     literally `hostname.keyword`). Kibana's mustache renderer expands dotted keys, so
     `._source.hostname.keyword` resolves correctly. Don't try to "fix" this to `.hostname`
     — that would render empty.

   > *If you're using `10008_single_stage_enrich_index` instead:* the action's `target` and
   > `event_uuid` templates use `_source.hostname` (no `.keyword`), because that variant
   > doesn't add the `.keyword` suffix at query time. Every other field in the action is
   > identical.

### Step 5. Details

- **Name** — `ITSMA Test 10008 - System CPU Utilization > 98%` (matches Stage 2's name minus
  " Stage 2", so it sorts next to the ones it replaces).
- **Tags** — `ITSMA`, `Test`, `Threshold rule`, `10008`, `CPU`.

### Step 6. Save — DISABLED

**Uncheck the "Enable" toggle at the bottom.** You want the rule created but not firing yet,
so you can validate before it starts writing to `kibana_alerts`.

**Click Save.**

### Step 7. Confirm the rule was created

- **Stack Management → Rules** — the new rule should appear, with status *Disabled*.
- Click into it — the rule detail page should show your query, the schedule, and the action
  with the connector name.

---

## Verify with Test query (again, from inside the rule)

- Open the rule → **Definition** tab → **Test query**.

That runs your query with the rule's *actual* time-window and shows the count. This is the
important check — it tests exactly what the scheduled runs will see, including permission
scoping the rule inherits.

**What you're looking for:**

- A count of `0` is OK if no host is currently breaching CPU >98% in the last 5 minutes and
  is enabled for 10008 in AMDB. Don't take that at face value — run S3 again to distinguish
  *"no breach right now"* from *"something's misconfigured"*.
- Any error message — fix before enabling. Common: *"policy not found"* (P2 wasn't run),
  *"index does not exist"* (the enrich source name is wrong).

---

## Enable and run side-by-side with Stage 2

- **Rule detail → Enable** toggle at the top.

For a cycle or two (5–10 minutes), **both** the new consolidated rule and the old
`Stage 1 + Stage 2` pair are firing. That's fine — they write to different documents in
`kibana_alerts` (the new rule uses a fresh `trace.id`, and `event_uuid` is
`on-prem_10008_<hostname>` in both). If you have any downstream deduplication on `event_uuid`,
this will show up as duplicate tickets briefly; if that's a problem, do the side-by-side out
of hours.

**Compare in Discover** (`kibana_alerts` index pattern):

```
KQL: event_type: "10008" AND @timestamp >= now-15m
```

For each host that appears, you should see one entry from the old Stage 2 rule and one from
the new one, with identical values for `target`, `labels.client_code`,
`alarm_tags.hs:std:app-code`, and `alarm_reason` (the number may differ by rounding — the
new rule rounds to 2dp; Stage 2 stored an unrounded `current_value`).

If any field is empty or different, stop and investigate before disabling the old rules.

---

## Disable Stage 1 and Stage 2

Once you're satisfied the new rule matches Stage 2:

- **Stack Management → Rules** — find both:
  - `ITSMA Test 10008 - System CPU Utilization > 98% Stage 1`
  - `ITSMA Test 10008 - System CPU Utilization > 98% Stage 2`
- On each, toggle **Enable → off**.

Leave them **disabled but not deleted** for now. Deleting is a separate decision — the audit
trail in `kibana_threshold_alerts` (populated by Stage 1) may be useful for a while, and
re-enabling is a single click if something surprising shows up.

---

## After: verify continuity

Watch `kibana_alerts` for the next few 5-minute cycles. The new rule alone should be
producing 10008 events for the same hosts you saw during the side-by-side. Signal that the
cutover is complete:

- **New tickets appear on schedule** (a host either breaches or it doesn't; if you have a
  reliable test host, use it).
- **No gaps** for hosts you'd expect to see.
- **No errors** on the rule (**Rule detail → Execution history** — every run should be *OK*
  or *Warning*, not *Failed*).

---

## Ongoing operations — the one thing that must not be forgotten

**Every time AMDB config changes** (a host is added, or an alert enabled/disabled), re-execute
the policy:

```
PUT _enrich/policy/kibana_sdp_amdb_policy/_execute
```

Until you do, the rule keeps filtering against the previous snapshot:

- A **newly enabled** host at 10008 will not alert until the next `_execute`.
- A **newly disabled** host at 10008 will keep alerting until the next `_execute`.

Wire this into whatever process maintains `kibana_sdp_amdb_enrich`, or schedule it. For
config that changes hourly-ish, an hourly cron is plenty. `DIAGNOSTICS` has an `_enrich/_stats`
call that shows the last execute time.

*(This step exists only for the ENRICH route. The `LOOKUP JOIN` variants read the source
index live and skip it.)*

---

## Rollback

Anything wrong at any point:

- **Disable the new rule.** Toggle off in the rule detail page.
- **Re-enable Stage 1 and Stage 2.** They were only disabled, not deleted.

No data cleanup needed — the tickets the new rule wrote to `kibana_alerts` are indexed as any
other alerts and can be filtered on `trace.id` or the timestamp range if the downstream
process needs to reconcile.

---

## Dev Tools alternative — skip the UI entirely

Once you're comfortable with the process, the whole rule creation collapses to one Dev Tools
call. Open `10008_single_stage_enrich_variant`, copy the request body, and:

```
POST kbn:/api/alerting/rule/<your-uuid>
{ …the body from 10008_single_stage_enrich_variant… }
```

The file uses a specific UUID as an example; substitute any fresh UUID (or let Kibana
generate one by `POST`ing to `/api/alerting/rule` with no `id`). The rule is created
**disabled** because `enabled` isn't set. To enable via API:

```
POST kbn:/api/alerting/rule/<uuid>/_enable
```

Or import the ndjson (`10008_single_stage_enrich_variant` doesn't have an ndjson counterpart;
the two LOOKUP JOIN variants do — `10008_single_stage.ndjson` and
`10008_single_stage_enrich_index.ndjson`) via **Stack Management → Saved Objects → Import**.

---

## The same procedure for other AMDB rules

For 10004, 10050, etc., the differences from the above walkthrough are usually:

- The event_type / group filter (`group == "10004"`).
- The metric being aggregated (`system.memory.actual.used.pct` for memory, etc.). Check
  `_field_caps` on the new metric — if it's multi-typed, cast it in the EVAL block.
- The threshold value.
- The connector body (`event_type`, `event_uuid`, `alarm_name`, `amdb_link`, `alarm_reason`).

The rest of the pipeline — the cast block, the `ENRICH _coordinator:kibana_sdp_amdb_policy`,
the `MV_EXPAND group`, the `WHERE alert_status == "enabled"`, the trailing `RENAME`, and the
action shape — is boilerplate you can copy verbatim.

Every rule uses the **same** enrich policy and the **same** `kibana_sdp_amdb_enrich` index.
You don't need a per-rule policy — one execute refreshes the config for all of them.

Once you've cut over every consolidated rule and no rule references `kibana_sdp_amdb` any
more, that original index can be retired. Until then, leave it be.
