---
title: "0 MB Read, 174 ms Stalled: The ClickHouse Read Regression That Was a Write Problem"
published: 2026-06-18
description: "At a live event, SELECT p99 on a real-time ClickHouse cluster jumped ~6x. The obvious suspects were extra Kafka consumers, a query-thread change, or hedged requests. The first mechanism I wrote down — slow SATA reads and merge-driven cache eviction — was wrong. Reads never touched the disk at all. A walk through a diagnosis that had to correct itself twice, and the verification habits that caught it."
image: ''
tags: [ClickHouse, Observability, Prometheus, Performance Tuning, Storage, Kafka, Debugging]
category: 'Coding'
draft: false
lang: 'en'
slug: clickhouse-read-regression-write-problem
---

## The report

> Query response time on the real-time cluster has a marked step up from the start of the live event. Is this the extra consumers we started? Could also be the change in query-thread count, or the hedged-requests setting. Can we narrow down which one?

That was the brief, with a Grafana panel showing a clean step: the gateway-measured response time sat flat at ~100–200 ms for a week, then jumped the evening play started and stayed elevated, with spikes into the seconds. The reporter was leaning toward **hedged requests** — meaning straggler hosts.

I've debugged this cluster before — it's the same real-time analytics fleet from [the live-event crash-loop postmortem](/posts/live-event-three-compounding-clickhouse-failures/), and the same one with [a hidden SATA disk tier](/posts/clickhouse-iowait-sata-nvme-disk-tier/). Both of those turned out to matter here too, but not in the way I first wrote down.

This post is mostly about being wrong. The first mechanism I committed to paper was clean, plausible, and contradicted by the cluster's own metrics. So was the second version of one number. The interesting content isn't the final answer — it's the verification steps that kept overturning it.

:::note
Everything here is read-only: Prometheus over its HTTP API, and `system.*` tables over a read-only HTTP user, fanned across the fleet with `clusterAllReplicas`. Every root-cause claim is checked against the matched ClickHouse source tree (25.8.x). Hostnames, customers, and the event are anonymized.
:::

## The shape

- ~60 nodes, **31 shards × 2 replicas**, fronted by a load-balancing proxy.
- Three hardware tiers across data centers: a CPU-heavy rack series, a fast NVMe series, and — crucially — an older **SATA-SSD series** (call it `ch40x`) that hosts one replica of shards 26–31.
- **Ingestion:** Kafka-engine tables → materialized views → wide "realtime" `_local` tables (200+ typed columns), 30-minute partitions, short retention.
- **Queries:** per-minute distributed `GROUP BY` rollups, sub-second, fanned to all 31 shards.

## Framing: get a timestamp, then a baseline

The first job is always to turn "it's slow" into a number with a clock on it. Fleet-wide SELECT latency by day from `system.query_log`:

| Period | p50 | p90 | p99 |
|---|---|---|---|
| baseline | 36 ms | 94 ms | **192 ms** |
| onset day | 48 ms | 138 ms | 605 ms |
| after | 54–68 ms | 150–276 ms | **900–1187 ms** |

p99 up ~6x; p50 up ~1.7x. The tail moved far more than the median — the signature of **stragglers**, not a uniform slowdown. And query *volume fell* over the same window (~11.5M → 7.4M initiator SELECTs/day), which immediately kills "more traffic" as a cause.

:::caution[First correction, before I even finished framing]
My very first latency pull ran against `system.query_log` *without* `clusterAllReplicas` — so through the proxy it hit one random node, which happened to be a fast one. That gave "p50 flat at 25 ms, p99 12–15x." Pretty, and wrong. Fanned across all nodes the real numbers are above: p50 is *not* flat. **A proxy in front of `system.*` means every bare query samples one arbitrary backend.** Always wrap in `clusterAllReplicas(<cluster>, system.<table>)` and group by `hostName()`.
:::

## The three suspects

**1. Extra consumers.** To handle the event, the team had added Kafka consumers. The consumer count is in Prometheus, and the step is unambiguous:

```
KafkaConsumers:  992 (steady for a week)  →  1088  →  ramp to 1230 (+24%)
```

The first increase lands in the same hour as the latency step. And in that same hour, concurrent merges jump `84 → 181` and stay ~80% elevated. So ingestion scaling and the latency step are time-locked.

**2. Query threads (`max_threads`).** Pulled `Settings['max_threads']` out of `query_log` across the 14-day window: unchanged. (A footgun here: my own capped debug wrapper injects `max_threads=4`, which I briefly mistook for the production value. Don't read your own probe's settings back as the cluster's.) And a parallelism change would shift the *whole* distribution; only the tail moved.

**3. Hedged requests.** `use_hedged_requests=1`, and it had never changed. The reporter's straggler instinct was right about the *symptom* — but a config that didn't change isn't the *trigger*. (More on the hedged metric below; my first reading of it was also wrong.)

So the trigger is the consumers. The interesting question is **how** more consumers turned into a tail-latency event.

## How more consumers became more parts

The "extra consumers" weren't extra *data*. Each new stream was a second Kafka table — `<table>_kafka_2` — reading the **same topic with the same consumer group** as the original. That's pure consumer parallelism to drain a topic faster, not new rows.

But it fragments writes. The tell is parts vs. rows on the busiest rollup table, comparing equal windows before and after:

| | rows ingested | parts created | avg part size |
|---|---|---|---|
| before | 777 M | 29.7 k | 9.3 MB |
| after | 775 M (**flat**) | 119.9 k (**4x**) | 2.8 MB |

Same rows, 4x the parts. Why? Because each Kafka table is an independent flush pipeline. From the matched source, `StorageKafka.cpp`:

```cpp
auto task_count = thread_per_consumer ? num_consumers : 1;   // line 207
...
auto pipe = Pipe::unitePipes(std::move(pipes));              // line 686
```

With `kafka_thread_per_consumer=1` (set on every table here), each consumer is its own background task with its own `streamToViews()` → its own INSERT → its own parts. A second `_kafka_2` table doubles the independent flush pipelines. And the default flush is **time-based** (`stream_flush_interval_ms`, 7.5 s), so a low-volume stream emits a part every 7.5 s regardless of how few rows arrived — one low-traffic table was minting ~470 parts/min at **~300 rows each**.

More parts → ~80% more merges. That's the lever. The question is why it hurt *one tier* so much.

:::tip
Two Kafka tables of one consumer each is **not** the same as one table of two consumers. With `thread_per_consumer=0`, `unitePipes` merges the consumers into a *single* insert per flush. With separate tables (or `thread_per_consumer=1`), you get N independent inserters, and MergeTree writes ≥1 part per insert per partition. Part count tracks *flush pipelines*, not consumers or rows.
:::

## The wrong turn

Here's the mechanism I wrote down first, and it sounded great:

> The hot table is 200+ columns with a sort key that doesn't lead with the time column, so a 4x part count means each query reads from 4x more parts — more marks, more granules, more random I/O. On the slow SATA disks, that random read I/O is brutal; and the merge storm evicts the page cache, turning former cache hits into SATA reads. Hence the tail.

Every clause of that is plausible. The reviewer pushed back with one sentence: *these machines have plenty of RAM for page cache.* So I checked, instead of defending.

```
SATA tier:  810 GB RAM,  707 GB free,  ~158 GB page cache
active dataset per node:  ~120 GB
```

The entire active dataset fits in cache several times over, with hundreds of GB free. There is no cache pressure. So "merges evict the cache" can't be load-bearing. But the decisive test is whether reads touch the disk at all — and ClickHouse exposes exactly that, two counters that differ only by the page cache:

- `OSReadChars` — bytes returned by `read()` syscalls (cache included)
- `OSReadBytes` — bytes actually pulled from the block device

```
per tier:   readChars 268–1110 MB/s     diskRead = 0.0 MB/s
```

**Zero.** Including at peak. Every SELECT read is served from RAM; none touches the SATA disk for data. My read-amplification story — random reads, cache eviction, the lot — was about a disk the reads never visit. Dead.

:::important
When you can't `iostat` the box, `OSReadBytes` vs `OSReadChars` *is* your cache-hit test. If `diskRead ≈ 0` while `readChars` is high, the read path is RAM-bound and any "slow disk reads" hypothesis is already falsified — go look at writes.
:::

## The real mechanism: reads waiting behind writes

If reads are cached and CPU isn't saturated, where does a 774 ms segment go? `query_log` stores per-query ProfileEvents, so I broke down slow SELECT *segments* (the per-replica sub-queries of a distributed query) by tier:

| slow segment on… | duration | CPU | **block-I/O wait** | disk read | rows |
|---|---|---|---|---|---|
| SATA (`ch40x`) | 870 ms | 254 ms | **174 ms** | **0 MB** | 1.1 M (light) |
| NVMe tier | 1582 ms | 1374 ms | 3 ms | 0 MB | 5.1 M (heavy) |

Two different animals. The NVMe tier's slow segments are *genuinely heavy* — lots of CPU, lots of rows — normal big-query tail. The SATA segments are **light** (1 M rows, 254 ms CPU) yet sit **174 ms in block-I/O wait having read zero bytes from disk.**

A thread that waits on block I/O while reading nothing isn't waiting for its own reads. The source confirms what that counter is (`ThreadProfileEvents.cpp`):

```cpp
profile_events.increment(ProfileEvents::OSIOWaitMicroseconds,
    safeDiff(prev.blkio_delay_total, curr.blkio_delay_total) / 1000U);  // line 163
```

It's the Linux taskstats `blkio_delay_total` — documented in `ProfileEvents.cpp:501` as *"real IO that doesn't include page cache."* The read thread is stalled behind the **write** congestion on the shared device: merge and replication writeback saturating the SATA disk's queue, and a cache-served read caught behind it.

So the chain is:

```
+_kafka_2 streams → 3–4x parts (flat data) → +80% merges + replication writes
   → SATA disk queue saturates → cache-served reads stall in blkio wait → tail p99
```

A **write** bottleneck wearing a read mask. The reporter's stragglers are real; they just straggle on writes, not reads.

## Why SATA, when it ingests the *least*

The intuitive guess is that SATA nodes straggle because they ingest the most. The opposite is true:

| tier | consumers/node | Kafka msgs/node | own inserts (`NewPart`) |
|---|---|---|---|
| CPU tier (shard siblings) | 20 | **3433 M** | — |
| NVMe | 19 | 2623 M | — |
| **SATA `ch40x`** | **16** | **862 M** | **1.4 GB/hr** |

The SATA nodes carry the *fewest* consumers and 3–4x *less* Kafka volume — someone already drained them. Their disk is busy with everything *except* their own ingestion. Write composition per SATA node, one hour from `part_log`:

| event | GB/hr | p99 duration | what |
|---|---|---|---|
| `MergeParts` | 33.7 | 14.6 s | merging the fragmented parts |
| `DownloadPart` | 12.1 | 1014 ms | replication fetched from busy sibling |
| `NewPart` | 1.4 | 848 ms | own Kafka inserts (tiny) |

The shard-26–31 siblings (on the CPU tier) are the heavy ingesters; they replicate parts *down* to the SATA replica. So the SATA disk eats merges of fragmented parts plus a 12 GB/hr replication stream — and barely any of its own ingestion.

One more nuance worth stating, because it corrects an over-claim I almost made. Merge *duration* is similar across tiers (SATA p99 14.6 s ≈ NVMe 13 s; the CPU tier is actually worst at 26 s). SATA handles big *sequential* merge writes fine. What singles it out is **small/concurrent** I/O: replication fetches at 1014 ms p99 vs ~340 ms on NVMe, and read threads stalling 174 ms vs 3 ms. That's the textbook SATA-SSD-vs-NVMe profile — comparable sequential throughput, far worse random IOPS and queue depth.

:::caution[A dropped recommendation]
My early fix list said "drain consumers off the SATA nodes." The consumer data shows that's *already done* — and it didn't help, because the killer load is merge + replication of the fragmented data, which lands on the SATA replica no matter where the consumers run. Always check whether your proposed fix is already in place before recommending it.
:::

## Are the sub-queries on SATA actually slower? Control for weight.

A fair objection: maybe SATA just gets heavier queries. So I compared segment latency by tier, then restricted to **light** segments (`read_rows < 2M`) to hold the work constant:

| tier | p50 | p99 (all) | p99 (light, equal work) |
|---|---|---|---|
| NVMe | 6 | 50 | 38 |
| CPU tier | 9 | 100 | 82 |
| **SATA** | 6 | **192** | **137** |

At the **median, SATA is identical** to NVMe (5–6 ms) — typical sub-queries are cache-served and disk-independent. At the **tail, SATA is ~4x slower for the same work** (p99 137 vs 35–38). So it's the node, not the workload. A distributed query waits for its slowest shard; with a SATA replica on shards 26–31, most queries hit one, and that segment's fat tail *becomes* the user-visible p99.

## The clincher: was SATA always like this?

If SATA were chronically slow, this would be a hardware story, not an event story. So I compared the same metric, same night hours, **before vs after** the event — light segments only:

| tier | p99 before | p99 after | change |
|---|---|---|---|
| NVMe | 26 ms | 28 ms | flat |
| CPU tier | 28 ms | 54 ms | ~2x |
| **SATA** | **20 ms** (fastest!) | **347 ms** | **~17x** |

Before the event, **SATA had the lowest tail of any tier.** Nothing was wrong with these nodes a week earlier. And the linchpin metric, block-I/O wait on light SATA segments:

| | avg io-wait | % of sub-queries with any io-wait |
|---|---|---|
| SATA before | 0.3 ms | 0.1% |
| SATA after | 3.2 ms | 0.8% |

`io_wait` was ~0 before, rose only on SATA, only after the onset — and it's a *burst/tail* effect (only ~0.8% of reads get caught, but those stall ~174 ms). That is the cleanest proof of a **latent fault line**: the disk had headroom, and the event's write storm consumed it.

## Confounder check: did anything else change?

The onset is pinned to one minute. So any alternative cause must land there. From `system.tables` `metadata_modification_time` over the surrounding days, the **only** DDL changes are the `_kafka_2` tables and their views — the first at exactly the onset minute. No `ALTER` to any target table. `system.mutations`: zero since well before. Server config (pools, concurrency) can only change on restart, and the fleet last restarted *two days after* the onset — so it can't be the trigger.

:::note
Honest limit: ClickHouse `system.*` keeps no settings *history*, so "no server-config change" is bounded by restart timing, not a changelog. For full closure you'd check config-management/git history. But every signal the cluster itself retains points to the `_kafka_2` additions as the sole coincident change.
:::

## The hedged-requests metric, and a second sampling lesson

Back to the reporter's favorite suspect. My first pass reported `HedgedRequestsChangeReplica` as "flat ~0 for 14 days." Then a later instant read showed **130/s**. Both were "true" — and both misleading. The rate is **bursty**:

```
baseline: 0     onset hour: 2575/s     between: ~0     later spike: 130/s
```

My 12-hour Prometheus step had landed *between* the bursts. (Same trap bit the OS `iowait` series: a coarse step reads ~0; `max_over_time` over each bucket shows the real spikes.) So hedged requests fire hard exactly at the straggler peaks — when a SATA replica is so write-starved it goes *silent* past `receive_data_timeout` (2 s) and the coordinator switches (`HedgedConnections.cpp:243,414,491`). Most of the time the straggler merely *trickles* slowly, never goes silent, and hedging stays blind — which is why the tail persists. Hedging is a **symptom** here, not the cause, and not a config that changed.

:::caution
A range query at a coarse step samples the instantaneous value at each grid point — it does **not** average the interval. For bursty counters (hedged switches, iowait), that silently hides the spikes. Use a fine step, or `max_over_time((...)[window])`, when you care about peaks.
:::

## The ultimate cause

Collapse the "why" ladder and it's the collision of two things:

- **Trigger:** the event was scaled by adding *parallel* `_kafka_2` consumer streams, which fragmented ingestion into 3–4x more parts → +80% merge and replication writes. Remove this and the write load drops; the incident ends.
- **Latent root:** shards 26–31 have only two replicas — one SATA, one CPU-bound — and **no fast replica**. A routine +24% ingestion bump should be a non-event; it became a ~6x latency regression *only* because it ran into a tier with no I/O headroom and no escape hatch. Remove this and the same change is absorbed silently.

Both are necessary. Neither alone produces the incident — which is exactly why SATA was the *fastest* tier the week before.

## Fixes, in priority order

1. **De-fragment ingestion.** Set `kafka_thread_per_consumer=0` + an explicit `kafka_max_block_size` so consumers unite into one insert per flush, then fold each `_kafka_2` back into its base table. Fewer, larger parts → fewer merges *and* less replication churn reaching SATA. (See `clickhouse-best-practices`: `insert-batch-size`, `insert-async-small-batches`.)
2. **Give shards 26–31 a fast replica.** The only change that removes the no-escape-hatch tier instead of trading disk-slow for CPU-slow.
3. **Raise the flush interval / enable `async_insert`** on the trickle tables to stop the 300-rows-per-part churn.
4. ~~Drain consumers off SATA~~ — already done; ineffective, because the load is merge + replication, not local ingestion.

## What I'd take to the next incident

The technical answer is "a write bottleneck on a weak disk tier surfaced as a read tail." The transferable part is the verification habits, every one of which overturned something I'd written:

- **Proxy + `system.*` = sample one node.** Fan out with `clusterAllReplicas`, or your baseline is one arbitrary backend (my "p50 flat" mistake).
- **`OSReadBytes` vs `OSReadChars`** is the cache-hit test you can run without SSH. `diskRead ≈ 0` falsifies every slow-read hypothesis in one query.
- **`OSIOWaitMicroseconds` is `blkio_delay`.** High io-wait + zero `OSReadBytes` = a read stalled behind writes. That single pair reframed the whole diagnosis.
- **Control for query weight.** "Light segments only" separates a slow *node* from a heavy *workload*.
- **Before-vs-after on the same metric** separates chronic from triggered — and turned "SATA is slow" into "SATA was the fastest, until it wasn't."
- **Coarse Prometheus steps hide bursts.** Two of my "flat ~0" readings were sampling artifacts.

And the meta-lesson: the first clean mechanism you write down is a hypothesis, not a finding. The cluster's own counters will tell you it's wrong if you let them — the job is to keep asking the question that could falsify it, not the one that confirms it.

---

*Companion posts on the same cluster: [a hidden SATA disk tier](/posts/clickhouse-iowait-sata-nvme-disk-tier/) and [three compounding live-event failures](/posts/live-event-three-compounding-clickhouse-failures/).*
