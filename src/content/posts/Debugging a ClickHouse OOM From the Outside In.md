---
title: "A Node 'Went Down' for Three Minutes: Reverse-Engineering a ClickHouse OOM From the Outside In"
published: 2026-06-16
description: "A monitoring alert said a ClickHouse node was down. It wasn't — the host had 152 days of uptime. This is the methodology for turning a vague `up < 1` into a precise root cause: triaging host-vs-exporter-vs-network, catching the moment the observability stack itself became the casualty, and cracking a 636 GB memory disappearance where the biggest logged query was 2 GB. Metrics for the timeline, `query_log` for the culprit, and the discipline to keep correcting your own first conclusion."
image: ''
tags: [ClickHouse, Observability, Prometheus, Debugging, Memory, OOM]
category: 'Coding'
draft: false
lang: 'en'
slug: clickhouse-oom-debugging-up-metric
---

## The report

The whole thing started with one alert expression:

```promql
up{cluster="olap-default", instance=~".*:9100"} < 1
```

Translation: *the node-exporter on one host stopped answering Prometheus a few minutes ago.* The human version of the ticket was shorter — "this node went down, why?" — and pointed at a single instance, `chNNN-4.dcX:9100`.

That phrasing contains the first trap of the whole investigation. `up < 1` does **not** mean "the host is down." It means **one scrape target failed to respond**. A host can be perfectly alive while its exporter is dead; a whole rack can vanish while every process keeps running. Before touching disks, memory, or ClickHouse, the job is to figure out *which kind of "down" this is.*

I can't SSH into these boxes on a whim. What I have is the same thing the alert has: whatever node-exporter and ClickHouse already push to Prometheus, queryable as an HTTP API. (I [wrote up that query-as-API loop separately](/posts/clickhouse-iowait-sata-nvme-disk-tier/) — a tiny `promq.sh` wrapper that turns `query` / `query_range` into sorted tables. Everything below leans on it.)

## Step 1: Is the host actually down?

The single most useful query in this entire investigation was also the dumbest. The alert watched port `9100` (node-exporter). So ask what *every other* target on the same host is doing:

```promql
up{instance=~"chNNN-4.dcX.*"}
```

```
1   instance=chNNN-4.dcX:9363   # ClickHouse metrics endpoint
1   instance=chNNN-4.dcX:9100   # node-exporter
```

By the time I looked, both were back to `1`. So whatever happened had already recovered — this was a post-mortem, not a live incident. Good to know before I start paging people.

Now the question that actually decides everything: **did the host reboot, or did it just go unreachable?** node-exporter answers this directly:

```promql
node_boot_time_seconds{instance=~"chNNN-4.dcX.*"}
```

The value was ~152 days in the past. The host had **not** rebooted, crashed, or power-cycled. Whatever took the exporter offline for a few minutes, the kernel underneath it never stopped counting.

That single fact kills a whole branch of hypotheses (hardware fault, kernel panic, power event, reboot loop) before I've spent any real effort on them. Always check boot time first; it's one query and it eliminates the scariest causes for free.

## Step 2: Pin the exact window

`query_range` plus a filter for the zero samples gives the precise outage, target by target:

```bash
curl -sk -G "$PROM/api/v1/query_range" \
  --data-urlencode 'query=up{instance=~"chNNN-4.dcX.*"}' \
  --data-urlencode "start=$((now-7200))" --data-urlencode "end=$now" --data-urlencode "step=15s" \
| jq -r '.data.result[].values[] | select(.[1]=="0")
         | "\(.[0]|strftime("%H:%M:%S")) \(.metric.instance)"'
```

```
08:38:06 … :9100   node-exporter down
…                  (1.5 min)
08:39:51 … :9100   node-exporter back

08:39:06 … :9363   ClickHouse down
…                  (1.5 min)
08:40:51 … :9363   ClickHouse back
```

This is more informative than it looks. **Both** scrape targets on the host failed, in an overlapping ~08:38–08:41 window, and both recovered on their own. They went down *staggered* (different scrape offsets) but their dead-time overlapped. So this isn't "node-exporter crashed" or "ClickHouse crashed" in isolation — it's something that took out *the whole host's ability to answer*, briefly, with no reboot.

Two daemons silent together + host never rebooted = a host-level stall or a host-level network blip. Those are very different root causes, so the next step is to separate them.

## Step 3: One host, or many?

If a network fabric hiccuped, or Prometheus itself had a bad few minutes, neighbours would have blipped at the same instant. Count the down targets across the whole cluster, over time:

```promql
count(up{cluster="olap-default"} == 0)
```

```
08:38:02   2      ← exactly this host's two targets
08:38:17   2
…
08:39:47   2
08:40:02   1
```

It peaks at **2**, and those two are `:9100` and `:9363` on the *same* host. No neighbour ever joined them. So:

- Not a Prometheus-side problem (it was scraping everyone else fine).
- Not a rack/fabric-wide network event (only one host).
- Whatever this was, it was **local to this single machine.**

That narrows it to two survivors: a transient network blip on this host's NIC, or the host stalling hard enough to stop servicing anything. Time to look at what the host was *doing*.

## Step 4: The first conclusion — and why it was wrong

Here's where it's worth being honest about the debugging process, because my first write-up of this incident was **wrong**.

I checked load (`node_load1`) around the edges of the gap: ~20–57 on a 192-core box. Trivial. No CPU saturation. Combined with "both targets down, host never rebooted, no neighbours affected," the tidy story was: *a transient network blip to a single host; self-recovered; nothing to do.*

That conclusion was clean, plausible, and didn't survive one more query. The lesson that generalizes: **a hypothesis that explains the symptom is not the same as a hypothesis you've tried to disprove.** I hadn't yet asked the host *why* it might have stalled. So I asked the two questions a brief host stall always demands — was the OOM killer involved, and what was memory doing:

```promql
# did the kernel OOM-kill anything?
node_vmstat_oom_kill{instance=~"chNNN-4.dcX.*"}
# available memory over the window
node_memory_MemAvailable_bytes{instance=~"chNNN-4.dcX.*"} / 1024^3
```

The "network blip" theory died instantly:

| Time (UTC) | `MemAvailable` | `oom_kill` |
|---|---|---|
| ≤ 08:36 | ~640 GB | 0 |
| 08:37:00 | **448 GB** | 0 |
| 08:38:00 | **3.99 GB** | 0 |
| 08:40:00 | — | **0 → 1** |
| 08:41:00 | **746 GB** | 1 |

Available memory fell off a cliff — **640 GB to 4 GB in about two minutes** — the kernel fired the OOM killer at 08:40, and immediately afterward ~742 GB came back. This was never a network event. The host stalled because it was **thrashing on memory exhaustion**, the exporters couldn't get scheduled to answer scrapes, the OOM killer reaped the biggest process, and the freed memory snapped back. The "blip" was the *shadow* of an OOM kill.

## Step 5: Confirm what died

The OOM killer reaps the largest memory consumer. On a ClickHouse data node, that's almost always `clickhouse-server` itself. Confirm with its own uptime metric:

```promql
ClickHouseAsyncMetrics_Uptime{instance=~"chNNN-4.dcX.*"}
```

```
08:37   13,133,312   (152 days — climbing normally)
   …    (gap)
08:42           43   ← restarted ~08:41:17
08:43          103
```

The counter resets to a few seconds and starts climbing again. `clickhouse-server` was OOM-killed around 08:40 and auto-restarted by its supervisor at ~08:41:17. The host's uptime never moved (Step 1); only ClickHouse's did. The "node down" alert was, all along, a **ClickHouse OOM crash** wearing a network costume.

A quick pattern check before going deeper — is this chronic?

```promql
increase(node_vmstat_oom_kill{cluster="olap-default"}[7d]) > 0
resets(ClickHouseAsyncMetrics_Uptime{instance=~"chNNN-4.dcX.*"}[7d])
```

One OOM, one restart, one host, in seven days. A single isolated event — which means I'm hunting one specific cause, not a leak or a systemic over-commit.

## Step 6: The observability blind spot

Now the obvious next question: *what consumed 636 GB?* And here the investigation hits a wall that's worth naming, because it recurs in every crash post-mortem.

**The thing that failed is the thing that monitors.** During 08:38–08:41 the exporters weren't being scraped — that's *why* `up` went to zero. So Prometheus has a hole exactly where the interesting data is. Worse, when I went to ClickHouse's own high-resolution `system.metric_log` (which records per-second `MemoryTracking` and concurrent query counts), the first row available for the whole day was timestamped **08:40:50** — after the crash. The pre-crash buffer never made it to disk: `metric_log` flushes in batches, and under the memory storm the flush thread was starved, so the in-memory buffer was simply lost when the process was killed.

So both of my time-series sources are blind across the peak. This is the pivot every crash investigation eventually makes: **when the timeline goes dark at the moment of death, stop asking the metrics what happened and start asking the survivor what it was doing.** `system.query_log` records a row when a query *starts*, not just when it finishes — so the queries running *into* the crash left a `QueryStart` footprint even though they never logged a `QueryFinish`. That asymmetry is the entire crack in the case.

A practical aside: I connected over the **HTTP interface** with `curl`, not the native client. The native `clickhouse-client` aborted at startup trying to resolve the *laptop's own* hostname — a local quirk that has nothing to do with the prod box. The HTTP endpoint (`POST` the SQL as the request body, or `-G --data-urlencode 'query=...'`) sidesteps that entirely and behaves exactly like the Prometheus loop. When one client is being weird, the other interface is right there.

## Step 7: The 2 GB / 636 GB paradox

Top finished queries by memory, across the whole hour around the crash:

```sql
SELECT event_time, formatReadableSize(memory_usage) AS mem, user,
       substring(query, 1, 60) AS q
FROM system.query_log
WHERE event_time BETWEEN '...08:00:00' AND '...08:42:00'
  AND type IN ('QueryFinish', 'ExceptionWhileProcessing')
ORDER BY memory_usage DESC LIMIT 10
```

The biggest query that *finished* used **2.03 GiB**. Everything else trailed below it. That's the paradox: ~636 GB vanished, and the largest tracked query is 2 GB. No single logged query explains the crash. Either it was death-by-a-thousand-queries (concurrency), or the real culprit never wrote a `QueryFinish` row — because it was the one the OOM killer interrupted.

The query that kills the server is, by definition, the one that *doesn't finish.* So invert the search: find queries that started in the window but have no terminal record.

```sql
SELECT event_time, user, query_id, substring(query,1,80) AS q
FROM system.query_log
WHERE event_time BETWEEN '...08:34:00' AND '...08:41:00' AND type = 'QueryStart'
  AND query_id NOT IN (
    SELECT query_id FROM system.query_log
    WHERE event_time BETWEEN '...08:34:00' AND '...08:45:00' AND type != 'QueryStart')
ORDER BY event_time
```

Sixteen queries started and never finished. Fifteen of them started *after* 08:37:24 — i.e. after memory was already collapsing. Those are **victims**: they got stuck because the box was already starving. Exactly one started *before* the collapse:

```
08:36:13   user=default   query_id=0463…   WITH windows AS ( SELECT '6/14 fast' AS w …
```

Memory began its dive at 08:37, one minute after this query started. By timing alone, this is the suspect. The discipline here: in a cascade, **timestamp ordering separates cause from casualty.** The earliest unfinished query that predates the symptom is the one to read.

## Step 8: Read the query

```sql
WITH windows AS (
  SELECT '6/14 fast' AS w, toDateTime('…13:26:11') AS t0, toDateTime('…13:28:30') AS t1
  UNION ALL SELECT '6/15 slow',  toDateTime('…14:10:14'), toDateTime('…16:05:47')
  UNION ALL SELECT '6/15 win3',  toDateTime('…13:00:03'), toDateTime('…14:08:43')
)
SELECT w.w, maxIf(a.value, a.metric='LoadAverage1') AS load1_max, …
FROM windows w
JOIN system.asynchronous_metric_log a
  ON a.event_time >= w.t0 AND a.event_time <= w.t1 AND a.event_date = toDate(w.t0)
GROUP BY w.w
```

There's the bomb, and it's the `JOIN` condition:

```sql
ON a.event_time >= w.t0 AND a.event_time <= w.t1 AND a.event_date = toDate(w.t0)
```

Every condition is an **inequality** (or a per-window constant). There is no equality key joining the two sides. ClickHouse can't build a hash table on `>=` / `<=`, so a range/non-equi join degrades to an effective **cross product**: it pairs the window rows against `system.asynchronous_metric_log` — a table that records *every* server metric *every* second, i.e. billions of rows — and only *then* filters by the time predicates. The intermediate set is what eats the machine.

The smoking-gun corroboration was in the *finished* versions of the same query, run minutes earlier by the same user. With tiny 2-minute windows they completed in under 400 ms using ~2 GB. The fatal run used **multi-hour** windows (`6/15 slow` spans ~2 hours). Same query shape, same table, windows three orders of magnitude wider — and the cross-product scaled right along with them until it consumed the host. The 2 GB ceiling on *finished* queries and the 636 GB *unfinished* explosion were always the same query at two different window sizes.

## Step 9: Why one query could take down the whole node

A pathological query should fail *itself*, not the server. That it killed the node is a configuration finding, not just a bad-query finding:

```sql
SELECT name, value FROM system.settings
WHERE name IN ('max_memory_usage', 'max_memory_usage_for_user');
-- max_memory_usage           = 0      (no per-query cap)
-- max_memory_usage_for_user  = 0      (no per-user cap)
```

`max_memory_usage = 0` means **a single query has no memory ceiling** — it is allowed to allocate until the server's global limit stops it. The server-wide limit *was* set (~0.9 × RAM), but the spike drove real RSS to ~751 GB — about **71 GB past** that global tracker. That overshoot is the tell: a chunk of the cross-product's memory was **untracked** by ClickHouse's `MemoryTracker`, so the soft global limit never tripped, and the kernel's OOM killer won the race. With a per-query `max_memory_usage`, this query would have died alone with `MEMORY_LIMIT_EXCEEDED` and the node would have stayed up.

## Step 10: Attribution and the concurrency red herring

Who, and was the load itself a factor? `query_log` keeps the client metadata:

```sql
SELECT user, address, interface, os_user, client_name
FROM system.query_log WHERE query_id = '0463…' AND type = 'QueryStart';
-- user=default  address=::1  interface=1(native)  os_user=root  client_name=ClickHouse client
```

`::1`, native protocol, `os_user=root`: the query came from a **local interactive `clickhouse-client` shell on the box**, run as root — not the application, not over the network. (Production app queries carry gateway-prefixed query IDs; this was a plain UUID.) It was a human running an ad-hoc diagnostic directly on a production data node.

But was the node *also* just overloaded? Counting queries started in the 4-minute window looked alarming at first:

```sql
SELECT user, count() FROM system.query_log
WHERE event_time BETWEEN '...08:35:00' AND '...08:39:00' AND type='QueryStart'
GROUP BY user;
-- default  18808
-- reader     832
```

18,808 queries in four minutes — until you look at *where* they came from. Grouping by `address` and `client_name` showed ~24 distinct peers with `client_name = 'ClickHouse server'` over HTTP: these are **distributed sub-queries from sibling shards** — the normal fan-out of a distributed cluster, ~750–850 each. Routine. And at the peak instant (08:37:00), the ~70 concurrently-active queries summed to a *tracked* total of just **408 MiB**.

So the concurrency was a red herring as a *cause*: the box served this exact traffic at 640 GB free right up until 08:36. It mattered only as *context* — heavy fan-out left little headroom, so the cross-product had less slack to chew through before the kernel stepped in. **636 GB of untracked single-query memory was the cause; 18,808 routine sub-queries were the weather.** Distinguishing the two is the difference between "add a per-query cap" and the wrong fix, "throttle the cluster."

## What actually fixed it

- **Real fix:** set a per-query `max_memory_usage` (e.g. ~200 GB) on the interactive/default profile. This single setting converts "runaway query crashes the node" into "runaway query gets a `MEMORY_LIMIT_EXCEEDED`." It is the one change that would have prevented the whole incident.
- **Defense in depth:** set `max_memory_usage_for_user` too, so a *burst* of heavy queries can't sum past RAM even when each is individually capped.
- **Headroom:** lower `max_server_memory_usage_to_ram_ratio` (~0.8) so the tracker aborts with more margin before the OS OOM killer can win the race against untracked allocations.
- **Fix the query:** never non-equi-`JOIN` a huge system table. Filter `asynchronous_metric_log` to the overall min/max time span in a subquery (with `event_date` partition pruning), then bucket rows into windows with `multiIf` — a single bounded scan instead of a cross product.
- **Process:** ad-hoc heavy diagnostics shouldn't run as root in a local shell against a production data node. Route them through a memory-capped profile.

## Lessons that generalize

1. **`up < 1` is a question, not an answer.** It means "a scrape failed," nothing more. Triage host-vs-exporter-vs-network *before* theorizing. `node_boot_time_seconds` is the cheapest, highest-value first query: it eliminates reboot/crash/power in one shot.
2. **Two daemons silent together, host still up = a host-level event, not a per-process one.** And `count(up == 0)` across the cluster instantly separates "one host" from "the network/Prometheus."
3. **A plausible conclusion is not a verified one.** My first answer — "transient network blip" — explained every symptom and was still wrong. The fix was one more query (`oom_kill`, `MemAvailable`) aimed at *disproving* it. Always ask the host *why* it would have stalled.
4. **The monitor is often the casualty.** When the failure is what stops the scrapes, your time series goes blind exactly when it matters. Pivot from "what do the metrics show" to "what was the survivor doing" — and know which of your logs survive a hard kill (`query_log`'s `QueryStart` did; `metric_log`'s in-memory buffer didn't).
5. **The query that kills the server is the one with no `QueryFinish`.** Don't rank by `memory_usage` of *finished* queries — invert and hunt the *unfinished* ones, then let timestamp order separate the cause from its casualties.
6. **Tracked memory can lie about totals.** A 2 GB ceiling on logged queries next to a 636 GB collapse means the real consumer was untracked and uncommitted — i.e. the one that died. The gap between RSS and `MemoryTracker` is itself a clue.
7. **Separate the cause from the weather.** 18,808 queries looked like the story; they were ambient distributed fan-out. Attribution (`address`, `interface`, `os_user`) and per-instant tracked totals kept the fix pointed at the real lever — a per-query cap — instead of throttling innocent traffic.

The alert said a node went down. The host had 152 days of uptime, the "outage" was three minutes of memory thrashing, and the root cause was a single hand-written `JOIN` with no equality key. None of it required SSH — just the metrics for the *when*, the `query_log` for the *what*, and the discipline to keep disproving the comfortable answer.
