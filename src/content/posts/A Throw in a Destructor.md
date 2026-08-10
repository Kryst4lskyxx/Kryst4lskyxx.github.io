---
title: "A Throw in a Destructor: Why One ClickHouse Pod Out of Twelve Aborted Itself"
published: 2026-08-10
description: "A Kubernetes ClickHouse pod restarted. It wasn't OOMKilled, it wasn't evicted, and it wasn't crash-looping — it called `std::terminate()` on itself. Six of twelve nodes ran out of file descriptors that hour and threw the same error 33 times; exactly one of those throws landed inside a destructor, and destructors are implicitly `noexcept`. This is the walk from `reason=Error` down to `Epoll::Epoll()` in the matched source, why the node with the *highest* FD count survived, and how micro-batched inserts plus a merge pool pinned at 64/64 built a self-reinforcing feedback loop. Includes the metric that made me diagnose it backwards the first time."
image: ''
tags: [ClickHouse, Kubernetes, Debugging, File Descriptors, C++, Merges, Observability]
category: 'Coding'
draft: false
lang: 'en'
slug: clickhouse-fd-exhaustion-destructor-abort
---

## The report

> "We saw pod restarts on the alerting cluster earlier today."

Twelve nodes, six shards × two replicas, ClickHouse on Kubernetes, running an internal 25.8.11 LTS fork. That sentence contains two claims, and by the end of the first probe **both** were wrong: it was one pod, not "pods," and it wasn't a restart in any of the senses people usually mean.

The interesting part isn't the fix. It's that the crash required a coincidence — the same error fired 33 times across six nodes that hour, and 32 of them were harmless.

:::note
If `~HedgedConnectionsFactory()` sounds familiar, it should: [a live-event incident two months ago](/posts/live-event-three-compounding-clickhouse-failures/) hit the same destructor on a different cluster. That one exited **139 (SIGSEGV)** and the throw came from `Epoll::remove()` — `epoll_ctl(EPOLL_CTL_DEL)` failing on teardown. This one exits **134 (SIGABRT)** and the throw comes from `Epoll::Epoll()` — `epoll_create1()` failing because the destructor tries to *allocate* a new descriptor. Same destructor, opposite ends of the FD lifecycle. Twice in two months is no longer a coincidence; it's a design property, and Step 5 is about why.
:::

## Step 1: What actually restarted

`uptime()` fanned across the cluster is the cheapest possible question, and it re-scopes the incident immediately:

```sql
SELECT hostName() AS host, uptime() AS uptime_s,
       formatDateTime(now() - uptime(), '%F %T') AS started_at
FROM clusterAllReplicas('alert-ch', system.one)
ORDER BY uptime_s ASC
```

```
alert-ch-3-0-0    19849    2026-08-09 20:43:18   ← ~5.5h ago
alert-ch-0-0-0   280769    2026-08-06 20:14:38
alert-ch-4-1-0   330000    2026-08-06 06:34:07
…                                                 (9 more, all 06:18–06:34)
```

One pod bounced today. Ten share a tight Aug-6 window — a rolling restart. One is off-cycle from three days ago.

Now the trap. Pull the Kubernetes restart counters and they look alarming:

```promql
kube_pod_container_status_restarts_total{namespace="alert-ch"}
```

```
15  pod=alert-ch-3-1-0      8  pod=alert-ch-1-0-0
14  pod=alert-ch-0-0-0      7  pod=alert-ch-2-1-0
12  pod=alert-ch-3-0-0      7  pod=alert-ch-1-1-0
10  pod=alert-ch-2-0-0      6  pod=alert-ch-0-1-0
 2  … shards 4 and 5 (×4)
```

Fifteen restarts! Except that counter is **cumulative over the pod's lifetime**. It answers "how many times ever," not "what happened today." The question I actually asked was:

```promql
increase(kube_pod_container_status_restarts_total{namespace="alert-ch",container="clickhouse"}[24h]) > 0
```

```
1.0007   pod=alert-ch-3-0-0
```

**One restart. One pod. In 24 hours.** Not a crash loop. `uptime()` and the counter agree, which is the cross-validation that lets me stop worrying about the other eleven.

:::note
`uptime()` only ever shows you the *most recent* restart — it can't distinguish "restarted once" from "restarted forty times." The Kubernetes counter can, but only as a delta. You need both: the counter for frequency, `uptime()` for the timestamp.
:::

## Step 2: Rule out the obvious killer

A ClickHouse pod on Kubernetes that dies is memory until proven otherwise. Kubernetes will tell you directly:

```promql
kube_pod_container_status_last_terminated_reason{namespace="alert-ch"} > 0
```

```
reason=Error      × 8   (shards 0–3)
reason=Completed  × 4   (shards 4–5)
```

No `OOMKilled`. Anywhere in the namespace. That one query kills the entire memory branch — no cgroup limit chase, no `MemoryTracking` archaeology, no arguing about `max_server_memory_usage_to_ram_ratio`.

`Error` means the container's main process exited non-zero. Something inside ClickHouse decided to die. And the `Completed` / `Error` split along shard lines (0–3 vs 4–5) is a hint I'll come back to.

## Step 3: The gold mine, and a lazily-created table

Two server-side tables matter for a crash. I checked both:

```sql
SELECT hostName(), count() FROM clusterAllReplicas('alert-ch', system.tables)
WHERE database='system' AND name='text_log' GROUP BY hostName()
```

`text_log` — enabled on all twelve. `crash_log` — the distributed query failed outright:

```
There is no table `system`.`crash_log` on server: alert-ch-4-0:9000
```

:::tip
`system.crash_log` is created **lazily, on the first fatal signal**. Its absence is not a broken config — it's evidence that *this node has never crashed*. Don't let a missing-table error abort your query; it's a finding. (Practically: query it per-node, or accept that one `clusterAllReplicas` call will fail on the healthy nodes.)
:::

That leaves `text_log`, and it persisted across the restart — continuous rows from 18:00 through 22:00 on the pod that died. Everything below comes from it.

## Step 4: The pod killed itself

Filtering `text_log` to the two minutes around 20:43 on the dead pod, with the query loggers excluded:

```
20:42:40  Fatal  BaseDaemon  ########## Short fault info ############
20:42:40  Fatal  BaseDaemon  Signal description: Aborted
20:42:40  Fatal  BaseDaemon  Code: 573. DB::ErrnoException: Cannot open epoll descriptor: ,
                             errno: 24, strerror: Too many open files. (EPOLL_ERROR)
…
20:43:18  Information  Application  Starting ClickHouse 25.8.11.x, PID 1
20:43:19  Information  Application  Ready for connections.
```

Thirty-eight seconds of downtime, self-recovered, shard still served by its other replica the whole time.

But `Signal description: Aborted` is SIGABRT — **the process aborted itself.** Not killed by the kernel, not killed by Kubernetes. And the stack trace is the whole story:

```
 3. DB::Epoll::Epoll()
 4. DB::TimerDescriptor::drain() const
 5. DB::TimerDescriptor::reset() const
 6. DB::HedgedConnectionsFactory::stopChoosingReplicas()
 7. DB::HedgedConnectionsFactory::~HedgedConnectionsFactory()     ← destructor
 8. ?
 7. std::terminate()
 6. std::__terminate(void (*)())
 5. terminate_handler()
```

An exception was thrown while a destructor was running. In C++11 and later, destructors are **implicitly `noexcept`** — an escaping exception doesn't propagate anywhere, it goes straight to `std::terminate()`. Which calls `abort()`. Which is SIGABRT. Which is exit code 134. Which Kubernetes reports as `reason=Error`.

The whole chain, front to back, in one line: *the server ran out of file descriptors at the exact moment it was tearing down a connection object.*

## Step 5: Confirming it against the source you're actually running

This is the step I refuse to skip, because a plausible mechanism read off a stack trace is still a guess. With the matched source tree checked out, every frame resolves:

```cpp
// src/Client/HedgedConnectionsFactory.cpp:50
HedgedConnectionsFactory::~HedgedConnectionsFactory()
{
    /// Stop anything that maybe in progress,
    /// to avoid interference with the subsequent connections.
    stopChoosingReplicas();          // ← no try/catch. none.

    pool->updateSharedError(shuffled_pools);
}
```

```cpp
// src/Client/HedgedConnectionsFactory.cpp:353
    for (auto & [timeout_fd, index] : timeout_fd_to_replica_index)
    {
        replicas[index].change_replica_timeout.reset();   // ← :355
        epoll.remove(timeout_fd);
    }
```

`TimerDescriptor::reset()` (`src/Common/TimerDescriptor.cpp:51`) ends by calling `drain()` on line 63. And `drain()` contains the detail that makes this whole bug possible:

```cpp
// src/Common/TimerDescriptor.cpp:66
void TimerDescriptor::drain() const
{
    …
    /// Due to a bug in Linux Kernel, reading from timerfd in non-blocking mode
    /// can be still blocking. Avoid it with polling.
    Epoll epoll;                       // ← :82 — allocates a NEW fd, unconditionally
    epoll.add(timer_fd);
```

```cpp
// src/Common/Epoll.cpp:18
Epoll::Epoll() : events_count(0)
{
    epoll_fd = epoll_create1(0);
    if (epoll_fd == -1)
        throw ErrnoException(ErrorCodes::EPOLL_ERROR, "Cannot open epoll descriptor");
}
```

And `M(573, EPOLL_ERROR)` at `src/Common/ErrorCodes.cpp:474` closes the loop with the code from the log line.

Sit with the irony for a second. A **workaround for a Linux kernel bug** — polling a timerfd instead of trusting non-blocking reads — means that *releasing* a timer requires *acquiring* a file descriptor. So the cleanup path allocates. When you are out of FDs, the code that runs to give resources back is the code that can't get started. And it runs inside a destructor, where throwing is fatal by language rule.

:::caution
Any resource acquisition on a teardown path is a latent abort. It works fine right up until the moment the resource is exhausted — which is precisely the moment teardown is running most often.
:::

## Step 6: The part that surprised me — it's a race, not a threshold

My first instinct was "find the node with the worst FD pressure, that's your victim." That instinct was wrong, and proving it wrong is the most transferable thing in this post.

The only FD metric exposed here is OS-wide:

```promql
ClickHouseAsyncMetrics_OSOpenFiles{namespace="alert-ch"}
```

| Pod | 8h max (spans the incident) | 1h max (now) |
|---|---|---|
| alert-ch-2-0-0 | **1,042,592** | 437,024 |
| alert-ch-3-1-0 | **1,035,680** | 412,832 |
| alert-ch-0-0-0 | **1,031,072** | 500,768 |
| **alert-ch-3-0-0** (died) | **959,648** | 411,296 |
| … shards 4–5 | 564K–676K | ~290–350K |

Baseline is 300–500K. During the incident three pods pressed against **1,048,576 — 2²⁰**, the classic `nofile` ceiling. And the pod that died had the *fourth* highest peak. Three nodes were under **more** FD pressure and survived.

The error counts say the same thing even more clearly:

```sql
SELECT hostName(), count(), min(event_time), max(event_time)
FROM clusterAllReplicas('alert-ch', system.text_log)
WHERE event_time BETWEEN '2026-08-09 20:00:00' AND '2026-08-09 22:00:00'
  AND message LIKE '%Code: 573%'
GROUP BY hostName() ORDER BY count() DESC
```

```
alert-ch-0-0-0   11   20:10:38 → 20:46:27
alert-ch-3-1-0    8   20:10:38 → 20:34:22
alert-ch-3-0-0    7   20:42:40 → 20:48:49     ← its FIRST one was the fatal one
alert-ch-5-1-0    4   20:34:22 → 20:46:27
alert-ch-1-0-0    2   20:19:53
alert-ch-2-0-0    2   20:48:49
```

**Thirty-three EPOLL_ERRORs across six nodes. One crash.** The other 32 hit call sites where a throw is just an error — a query fails, a merge aborts, life goes on. The node with eleven of them shrugged them all off.

So the crash isn't "FD usage crossed a line." It's "an FD failure happened to land inside `~HedgedConnectionsFactory()`." Same pressure, same error, same version — one bad draw. Which means the blast radius scales with how *often* you tear down hedged connections:

```sql
SELECT name, value, changed FROM system.settings WHERE name='use_hedged_requests'
```
```
use_hedged_requests   1   1     ← changed=1: written explicitly in this cluster's config
```

`changed=1` only tells you it was set explicitly — it happens to match the shipped default, which is `true` (`src/Core/Settings.cpp:288`). That's the uncomfortable part: **this teardown path is active on essentially every ClickHouse deployment doing distributed queries.** Nobody opted into it. You have to opt *out*.

## Step 7: Why the FDs ran out

FD exhaustion is a symptom. Working backwards, the same hour was full of this:

```
Code: 252. DB::Exception: Too many parts (3006 with average size of 99.04 KiB)
in table 'default.alert_thresholds_local'.
Merges are processing significantly slower than inserts
```

3000 is not a coincidence — it's `parts_to_throw_insert`, default at `src/Storages/MergeTree/MergeTreeSettings.cpp:672`, thrown at `src/Storages/MergeTree/MergeTreeData.cpp:5447-5452`, and the cluster runs the default. Three thousand active parts, averaging **99 KiB**, each with its own column files, all open. That's where a million file descriptors went.

The storm is sharply bounded:

| 10-min bucket | `Too many parts` errors |
|---|---|
| 20:10 | 577 |
| 20:30 | 1707 |
| 20:40 | 941 (crash at 20:42:40) |
| 21:50 | 752 |
| **22:00 onward** | **0** |

I checked that the zeros were real and not a logging gap — `text_log` keeps writing 12–44K rows per 30 minutes after 22:00, with `Too many parts` at exactly zero. Merges caught up. (Whether it *stays* that way is a different question, and I get to it near the end — it doesn't.)

### The reading I got wrong

I assumed an insert spike. `part_log` appeared to say otherwise:

| 30-min bucket | new parts | avg rows/insert | merges | failed merges |
|---|---|---|---|---|
| 19:00 | 43,945 | 1086 | 13,846 | 45 |
| 19:30 | 43,742 | 1092 | **12,688** | 185 |
| 20:30 | 38,878 | 1111 | **13,046** | 329 |
| 22:30 | 42,183 | 950 | 16,746 | 439 |
| 00:30 | 41,160 | 861 | **18,733** | 23 |
| 01:30 | 42,169 | 792 | **19,640** | 34 |

New parts flat at 40–45K per 30 minutes; merges down to ~12–13K during the incident and back to ~19K once it cleared. I wrote that up as "not an ingestion spike — a **merge dip**."

That was wrong, and it's worth showing exactly how, because the numbers above are all *correct*. The mistake was in what they measure.

`part_log`'s `MergeParts` rows count **completed** merges. The fleet-wide gauge counts **running** ones — and it moved the other way:

```promql
sum(ClickHouseMetrics_Merge{namespace="alert-ch"})
```

```
18:00   394
19:00   487
19:30   612      ← doubled, not dipped
20:30   595
21:30   608
23:00   377
00:00   252
```

Running merges **doubled** — 300–330 before, **614** at the peak. Merges weren't idle, they were saturated. And `InsertedRows` shows there *was* a ramp after all:

```
16:00  1,070,853  →  20:30  1,367,487  (+28%)  →  23:00  1,220,393
```

Meanwhile `MergedRows` stayed roughly flat at ~5M per interval despite twice the concurrency. So each merge got slower, completions fell, and the *completed*-merge count dropped — which is precisely what made `part_log` look like a dip when the pool was actually pinned.

Pinned against what? The per-node ceiling is `background_pool_size` × `background_merges_mutations_concurrency_ratio`:

```
background_pool_size                            32
background_merges_mutations_concurrency_ratio    2     → 64 merge tasks/node
```

Peak concurrent merges per pod during the episode:

```
alert-ch-0-0-0 … alert-ch-3-1-0   peak=64   ← all 8 pods of shards 0–3, AT CEILING
alert-ch-5-0-0                    peak=40
alert-ch-5-1-0                    peak=39
alert-ch-4-1-0                    peak=37
alert-ch-4-0-0                    peak=33
```

Exactly 64 on all eight pods of shards 0–3. Not "near" the ceiling — *at* it, flat against it. Shards 4–5 had headroom and were never in trouble.

And that single number retro-explains every asymmetry in this incident: it's the same shards that threw `TOO_MANY_PARTS` locally, the same shards showing `reason=Error` instead of `Completed`, and the same shards whose part counts blew past 3000 while 4–5 stayed under 79.

So the real mechanism is **merge-pool saturation under a diurnal insert ramp**: chronic micro-batching sets a high part-creation floor, a +28% evening ramp pushes the loaded shards to 64/64, completion falls behind creation, and parts sail past 3000. CPU was ~17% average the whole time — this is **pool-limited, not CPU-limited**, so more cores would have changed nothing.

:::caution
`part_log` counts merges that **finished**; `system.metrics` / `ClickHouseMetrics_Merge` counts merges **running right now**. Under saturation these move in *opposite* directions — completions fall precisely because concurrency is maxed and each merge is slower. Read the completion count alone and you will diagnose an idle merge subsystem that is in fact flat-out.
:::

### The feedback loop

Then the merge failure reasons closed the circle:

```sql
SELECT error, count(), substring(any(exception),1,120)
FROM clusterAllReplicas('alert-ch', system.part_log)
WHERE event_time BETWEEN '2026-08-09 19:00:00' AND '2026-08-09 22:30:00'
  AND event_type='MergeParts' AND error > 0
GROUP BY error ORDER BY count() DESC
```

```
234   13178   No active replica has part … (cannot execute queue-… MERGE_PARTS)
 76     203   Cannot open file /data/lib/clickhouse/store/…/tags…bin:
              errno: 24, strerror: Too many open files
236     166   Cancelled merging parts
```

There it is. Code 76 is the **same `errno 24`**. FD exhaustion was breaking the merges — 203 of them — and the merges were the only thing that could reduce the part count that was consuming the FDs.

```
micro-batched inserts → parts pile up → FDs consumed → merges fail on EMFILE
        ↑                                                        │
        └────────────── part count climbs further ←──────────────┘
```

A self-reinforcing loop that only broke when the insert pressure eased enough for merges to win the race.

### The actual driver

```sql
SELECT count() AS inserts, round(avg(written_rows)) AS avg_rows,
       quantile(0.5)(written_rows) AS p50, max(written_rows) AS max_rows
FROM clusterAllReplicas('alert-ch', system.query_log)
WHERE event_time > now() - INTERVAL 2 HOUR AND type='QueryFinish'
  AND query_kind='Insert'
  AND arrayExists(t -> t LIKE '%alert_thresholds%', tables)
```

```
inserts   avg_rows   p50   max_rows
170531        760     34     108602
```

**23.7 inserts/second. Median batch: 34 rows.** The mean of 760 is doing a lot of hiding — the distribution is dominated by tiny writes with occasional large ones. Per `insert-batch-size` in `clickhouse-best-practices`, the *minimum* sane batch is 1,000 rows and the ideal range is 10K–100K. A p50 of 34 is roughly 30× below the floor. `async_insert` was `0`.

:::note
Report `p50` alongside `avg` for insert sizes, always. An average of 760 rows looks merely suboptimal. A median of 34 tells you the pipeline is writing one part per handful of rows, which is the actual pathology.
:::

## Step 8: Scope — and what the shard split meant

That `Error` / `Completed` split from Step 2 turns out to be real signal. Locally-originated `TOO_MANY_PARTS` throws, separated from ones merely *forwarded* by the distributed insert queue:

```
host              local_throw   forwarded
alert-ch-0-1-0        2358          68
alert-ch-0-0-0        2098          47
alert-ch-3-0-0        1298         178
alert-ch-3-1-0         606         220
alert-ch-2-1-0         586         258
alert-ch-2-0-0         210         175
alert-ch-1-*, 4-*, 5-*   0        ~320 each
```

Shards 0, 2, 3 were the source. Everyone else just relayed the error. `MaxPartCountForPartition` agrees — peaks of 4466 / 3694 / 3059 / 3027 / 3000 / 2964 / 2826 / 1517 on shards 0–3, and **never above 79** on shards 4–5.

Without splitting local from forwarded, that first query said "all 12 nodes affected" and I'd have chased a cluster-wide cause. Distributed tables propagate errors, so an error appearing on a node is not evidence the node caused it.

## Step 9: Is this the first time? (it isn't)

A single incident tells you what broke. It doesn't tell you whether you're looking at a freak event or the third instance of something with a schedule. `text_log` couldn't answer that — it only retains days. But there's a counter for exactly this, incremented at the throw site (`MergeTreeData.cpp:5449`, one line above the `throw`):

```cpp
ProfileEvents::increment(ProfileEvents::RejectedInserts);
throw Exception(ErrorCodes::TOO_MANY_PARTS, …);
```

Which Prometheus scrapes, and retains for a month:

```promql
sum(increase(ClickHouseProfileEvents_RejectedInserts{namespace="alert-ch"}[1h]))
```

Before trusting the zeros, I checked the metric was actually *present* on every pod for the whole window — `count(...)` returns 12 for all 30 days, so the gaps are real gaps and not missing scrapes. Every non-zero hour in 30 days:

| Episode (UTC) | Rejected inserts | Peak parts | Pod abort? |
|---|---|---|---|
| Sun 07-19 21:30–22:30 | 1,738 | 3,117 | no |
| Thu 08-06 09:30 | 222 | — | no |
| **Thu 08-06 18:30–22:30** | **14,386** | 4,710 | yes |
| **Sun 08-09 20:30–22:30** | **3,190** | 4,504 | yes (this post) |

Three real episodes, **every one in the 18:30–23:00 UTC band**, and the gap is closing: 18 days, then 3. **Two of the three produced exactly one pod abort each** — and the one that didn't is the smallest. That's the race model making a prediction and the historical data agreeing with it: more EMFILE throws, more chances one lands in the destructor.

The baseline is the part that reframes the whole thing:

```
daily peak MaxPartCountForPartition, all other days: 73 – 136
during episodes:                                     3,117 – 4,710
```

Normal operation sits at ~90 parts against a throw threshold of 3000. This is **not** a system slowly drifting into its ceiling — it's flat, healthy, boring, and then a 30×+ excursion inside an hour. Which means part count is a *terrible* early-warning signal: there's no ramp to alert on. `RejectedInserts > 0` and merge concurrency ≥ 90% of `64/node` both fire before anything is user-visible; part count only tells you after you've already lost.

## What I ruled out, so nobody re-investigates it

- **OOM / memory pressure.** No `OOMKilled` in the namespace; `reason=Error`. Not a memory incident at all.
- **Crash loop.** Exactly one restart in 24h. The scary cumulative counters are lifetime totals.
- **CPU / IO starvation.** ~17% average CPU across the window.
- **Keeper.** No `KEEPER_EXCEPTION`; the keeper pod's single restart was `reason=Completed`.
- **`Address already in use: 0.0.0.0:8123` at startup.** This appears in the restart log and looks damning — the new process apparently failing to bind its own ports. It's benign: `[::]:8123` binds first and with `IPV6_V6ONLY=0` already covers IPv4, so the subsequent explicit IPv4 bind is redundant and fails. The proof isn't the theory, it's that the node logged `Ready for connections` one second later and has served every query I've thrown at it since.

## Fixes, in priority order

1. **Batch the writes.** Per `insert-batch-size`: 10K–100K rows per INSERT, 1,000 minimum. A p50 of 34 rows is the root cause; everything else in this post is downstream of it.
2. **If the client can't batch, buffer server-side.** Per `insert-async-small-batches`: `async_insert=1` with `wait_for_async_insert=1`, scoped to the writing user via `ALTER USER … SETTINGS` rather than globally.
3. **Raise `background_pool_size` on the loaded shards.** 32 (→ 64 merge tasks) is what they pin against during every episode, on 192-logical-core boxes idling at 17% CPU. That's conservative for the hardware. This is the binding constraint *at the moment of failure* — though note it buys headroom rather than removing the need for it.
4. **Consider `use_hedged_requests=0`.** This doesn't fix FD exhaustion — it removes the *abort path*, converting this failure class from a crash into an ordinary error. The trade-off is real (you lose hedged-request tail-latency protection), so it's a judgement call, not an obvious win.
5. **Raise `nofile` on the pod spec.** Widens the margin. Treats the symptom.
6. **Alert on `RejectedInserts > 0`,** not on part count — see Step 9 for why part count gives you no warning.

Note the ordering, because these fix different things. #1 removes the cause. #3 raises the ceiling the cause runs into. #4 changes the *consequence* from a crash to an error without touching either. All three are worth doing; conflating them is how you end up raising a limit and calling it a fix.

## What I'd take to the next incident

- **Cumulative counters answer the wrong question.** `restarts_total` said 15; `increase(...[24h])` said 1. Always delta a counter over the window you actually care about.
- **`reason` before anything else.** One query separates OOMKilled from Error from Completed, and each sends you down a completely different path. It costs nothing and it's the highest-information probe available on Kubernetes.
- **A missing `crash_log` is data.** It's created on first fatal signal. Absence means "never crashed here," not "misconfigured."
- **`text_log` is worth its retention cost.** Every load-bearing fact in this investigation — the fatal stack, the storm boundaries, the errno, the local-vs-forwarded split — came from it. Without it I'd have had a pod restart and no explanation, and the pre-crash evidence would have died with the process.
- **Distinguish originated from forwarded errors.** Distributed tables relay exceptions. "All 12 nodes show the error" meant three nodes had the problem.
- **`avg` hides ingestion pathology; `p50` exposes it.** 760 vs 34.
- **"Completed" and "running" counters move in opposite directions under saturation.** `part_log` said merges were dwindling; `ClickHouseMetrics_Merge` said they'd doubled and were pinned at the pool ceiling. Both true, and only the second one is the diagnosis. When a subsystem looks *idle* during an incident, check whether you're reading its throughput instead of its concurrency.
- **A counter at the throw site outlives your logs.** `RejectedInserts` sits one line above the `throw` and lives in Prometheus for a month, which turned "an incident" into "the third episode, and they're accelerating." Look for the ProfileEvent next to the exception; it's the cheapest history you'll ever get.
- **A healthy baseline can be the alarming part.** ~90 parts on ordinary days against a 3000 threshold sounds like enormous headroom. It actually means there's no ramp to alert on — the system goes from fine to failing inside an hour.
- **Frequency of a failure ≠ severity of a failure.** Thirty-three identical errors, thirty-two harmless. The node with the most of them was fine; the node with the fewest died. When a failure's consequence depends on *where* it lands, "how bad is the pressure" is the wrong question — ask "how many places can this land, and is any of them fatal?"

And the one that generalizes furthest, past ClickHouse entirely: **teardown paths that allocate are latent aborts.** A destructor that needs a resource in order to release a resource is fine forever, and then it is a SIGABRT — at exactly the moment the system is under the pressure that makes destructors run most.

---

*Direct prequel: [a live event and three compounding failures](/posts/live-event-three-compounding-clickhouse-failures/) — the first time this destructor took down a cluster, via the other end of the FD lifecycle. Also from the same corner of the world: [reverse-engineering an OOM from the outside in](/posts/clickhouse-oom-debugging-up-metric/), [a read regression that was a write problem](/posts/clickhouse-read-regression-write-problem/), and [packaging this workflow into an agent skill](/posts/clickhouse-debug-agent-skill/).*
