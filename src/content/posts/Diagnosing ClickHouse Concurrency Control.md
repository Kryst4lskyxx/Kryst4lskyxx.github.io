---
title: Diagnosing ClickHouse Concurrency Control
published: 2026-05-29
description: 'A production broken-pipe incident, a 7-second thread creation stall, and a deep dive into how ClickHouse CPU slot concurrency control actually works.'
image: ''
tags: [ClickHouse, Big Data Engineering, Performance Tuning, Concurrency, Kubernetes]
category: 'Coding'
draft: false 
lang: 'en'
slug: diagnosing-clickhouse-concurrency-control
---

## The symptom

A real-time analytics cluster (distributed `GROUP BY GROUPING SETS` rollups, fired once per minute per customer) started logging the occasional query failure:

```
Code: 210. DB::NetException: I/O error: Broken pipe, while writing to socket
  (...:9000 -> ...:48370). (NETWORK_ERROR)
```

The user-facing query *succeeded* every time. So why the exception, and what was it telling us? The answer turned out to be a tour through ClickHouse's CPU-slot **concurrency control** — and a misconfiguration that was quietly throttling almost every query on the cluster.

This post walks the investigation and then explains the concurrency-control settings and metrics in detail, grounded in the 25.8 source.

> **Related:** a later incident on the same cluster — `CANNOT_SCHEDULE_TASK` thread-spawn failures under a query stampede — is dissected in [Debugging ClickHouse CANNOT_SCHEDULE_TASK](./debugging-clickhouse-cannot-schedule-task/).

## Step 1: the broken pipe is benign — it's hedged requests

The failing query was one shard-request of a distributed query. Shard 1 had two replicas. With `use_hedged_requests=1` and `load_balancing=random`, the initiator sent the query to replica `1-0` first. That replica stalled, so the initiator fired a hedged copy at replica `1-1`, which finished in **90 ms**. The initiator then dropped the connection to the slow replica → `1-0` got a *broken pipe* trying to write its results back.

The initiator's profile confirmed it: `HedgedRequestsChangeReplica=1`. So the exception row is the failover mechanism doing its job, not a real failure. The real question is: **why was replica `1-0` slow?**

## Step 2: the real anomaly — 7 seconds to create a thread

The failed replica's profile events held the smoking gun:

```
GlobalThreadPoolThreadCreationMicroseconds = 7140287   (~7.14 s)
LocalThreadPoolThreadCreationMicroseconds  = 7143334   (~7.14 s)
RealTimeMicroseconds                       = 7344008
```

Nearly the entire 7.2 s wall time was spent **waiting to create a single thread** — not doing query work (it scanned only ~20k rows). A `pthread_create` that normally takes microseconds taking 7 seconds means one thing: the process was **CPU-starved at the cgroup level**. These are Kubernetes pods, and the cgroup CFS quota was being throttled.

Corroborating signals on the *other* replicas at the same instant:

- `ConcurrencyControlSlotsDelayed=19`, `ConcurrencyControlQueriesDelayed=1` — concurrency control was throttling.
- `PartsLockWaitMicroseconds=666881` (~0.67 s waiting on a parts lock).

So the cluster was under heavy CPU contention; hedged requests masked it for the end user.

## Step 3: the configuration that caused it

The pods are large — `cpu: limit == request == 180` (Guaranteed QoS, hard CFS quota = 180 cores). And the thread-limit config said:

```xml
<!-- config.d/thread_limit.xml -->
<concurrent_threads_soft_limit_ratio_to_cores>4</concurrent_threads_soft_limit_ratio_to_cores>
<concurrent_threads_soft_limit_num>0</concurrent_threads_soft_limit_num>
```

ClickHouse reads the cgroup CPU quota for its core count (verified in `src/Common/getNumberOfCPUCoresToUse.cpp` — it reads `cpu.max` / `cpu.cfs_quota_us`). So the CPU-slot budget was:

> **4 × 180 = 720 concurrent query threads, on a box whose hard CFS quota is 180 cores.**

That's deliberate **4× oversubscription**. When enough queries pile on, 720 runnable threads burn the 180-core quota within a few ms of each 100 ms CFS period, and the kernel **throttles the entire cgroup** for the rest of the period. While throttled, even thread creation stalls — exactly the 7 s we saw.

## Step 4: the chronic problem — every query throttled

A second, *successful* sample (different rollup, all replicas finished in 64 ms–1.8 s) was actually more diagnostic. Every shard query showed the same thing:

| Node | Granted | Delayed | Competing acquired | Effective threads | Wanted (`max_threads`) |
|---|---|---|---|---|---|
| `3-1` | 1 | **19** | 0 | **1** | 20 |
| `2-1` | 1 | **19** | 0 | **1** | 20 |
| `1-0` | 1 | **19** | 1 | **2** | 20 |
| `0-0` (initiator) | 1 | **19** | 6 | **7** | 20 |

`ThreadCreation` was sub-5 ms this time (less loaded), yet **every query still had all 19 competing slots delayed**. The cluster was *continuously* at its concurrency limit — most queries ran at 1–7 threads despite requesting 20. They were only fast because the data was small.

To understand why, we need the concurrency-control model.

---

## How ClickHouse concurrency control works

### Why it exists at all

The [official architecture docs](https://clickhouse.com/docs/development/architecture#concurrency-control) frame the problem cleanly. A parallelizable query uses `max_threads`, whose default is deliberately chosen so that **one query can fully saturate every CPU core**. That's great in isolation — but the moment several queries run concurrently, each still asks for enough threads to use all cores. They oversubscribe the CPU, the OS steps in to enforce fairness by context-switching, and you pay a penalty. As the docs put it, `ConcurrencyControl` exists to *"deal with this penalty and avoid allocating a lot of threads."*

This is exactly the trap behind the incident above: `max_threads=20` was the per-query multiplier, and nothing reconciled the sum of those demands with the real core count.

### The model: a slot = the right to run one thread

ClickHouse maintains a single server-wide pool of **CPU slots**. A slot is, in the docs' words, *"a unit of concurrency"*: to run a thread, a query must **acquire a slot in advance and release it when the thread stops**. A query wanting N threads must acquire N slots. Each slot is *"an independent state machine"* with a lifecycle (`src/Common/ConcurrencyControl.h`):

```
free  ->  granted  ->  acquired  ->  free
```

- **free** — *"available to be allocated by any query."*
- **granted** — *"allocated by specific query, but not yet acquired by any thread."* A short, transitional state.
- **acquired** — *"allocated by specific query and acquired by a thread."* The thread is running.
- back to **free** when the thread finishes

### How one query requests slots

In `src/Processors/Executors/PipelineExecutor.cpp`:

```cpp
const auto master_threads = 1uz;              // always 1
const auto worker_threads = num_threads - master_threads;
...
return ConcurrencyControl::instance().allocate(master_threads, num_threads);  // allocate(min=1, max=max_threads)
```

Every query calls `allocate(min = 1, max = max_threads)`:

- **min = 1** → the *master* slot, granted immediately so the query can always make progress (never fully starves).
- **max = `max_threads`** → desired parallelism; the other `max_threads − 1` are *competing* slots it may have to wait for.

This is the crux: **`max_threads` directly sets how many competing slots each query demands.** A query with `max_threads=20` demands **19** competing slots.

The docs distill the whole mechanism into a three-function API:

1. **Allocate** — `auto slots = ConcurrencyControl::instance().allocate(1, max_threads);` reserves at least 1 and at most `max_threads` slots. The first is granted immediately; the rest may arrive later. This is what makes the limit *soft* — every query is guaranteed its one thread.
2. **Acquire** — `while (auto slot = slots->tryAcquire()) spawnThread(...);` a thread picks up a granted slot and starts. `tryAcquire` is non-blocking: if no slot is available, the query simply runs with fewer threads rather than waiting.
3. **Resize** — `ConcurrencyControl::setMaxConcurrency(concurrent_threads_soft_limit_num)` changes the total pool size **at runtime, without a restart** — which is why the ratio/limit settings below are reloadable.

### The two schedulers — and oversubscription

Set by `concurrent_threads_scheduler`. The difference is whether the master slot counts against the limit:

**`round_robin`** (old default) — the master slot *is* counted:

```cpp
SlotCount granted = std::max(min, std::min(max, available));
state.cur_concurrency += granted;     // min IS counted
```

Because each query gets its master slot unconditionally, the total can **exceed** the limit → oversubscription possible. Also unfair: a flood of `max_threads=1` queries each grab a guaranteed slot.

**`fair_round_robin`** (current default) — master slots are *free*:

```cpp
SlotCount limit = max - min;          // = max_threads - 1 competing slots
SlotCount granted = std::min(limit, available);
state.cur_concurrency += granted;     // min is NOT counted
```

Only competing slots count, so total acquired can never exceed the limit → no oversubscription at the ClickHouse layer, and `max_threads=1` queries need zero competing slots (perfectly fair).

> **Key caveat:** When the header comment in `ConcurrencyControl.h` says of `fair_round_robin` *"There is no oversubscription: total amount of allocated slots CANNOT exceed `setMaxConcurrency(limit)`"*, it means oversubscription **of the slot budget** — slot *accounting*, not CPU cores. (For `round_robin` the same comment says the opposite: *"Oversubscription is possible … because `min` amount of slots is allocated for each query unconditionally."*) That guarantee says nothing about whether the budget itself is sane: set it above the core count (ratio 4 → 720 slots on 180 cores) and `fair_round_robin` will faithfully keep acquired slots ≤ 720 while the **OS** is hopelessly oversubscribed. The kernel, not ClickHouse, then does the throttling (the 7 s `pthread_create`).

### The settings

| Setting | Scope | Default | Meaning |
|---|---|---|---|
| `concurrent_threads_soft_limit_num` | server | `0` (unlimited) | Absolute pool size. |
| `concurrent_threads_soft_limit_ratio_to_cores` | server | `0` | Pool size = ratio × cores (cores from the **cgroup** quota). |
| `concurrent_threads_scheduler` | server | `fair_round_robin` | `round_robin` vs `fair_round_robin`. Runtime-changeable. |
| `use_concurrency_control` | per-query | `true` | If `false`, the query bypasses the pool entirely (no counting). |

"Soft" means the limit caps *competing* threads, but the master slot is always granted, so a query never deadlocks — it just runs with fewer threads. When both `num` and `ratio` are 0, the limit is `UnlimitedSlots` and the mechanism is a no-op.

### The per-query profile events (in `query_log.ProfileEvents`)

| Event | When | How to read it |
|---|---|---|
| `ConcurrencyControlSlotsGranted` | `+= min` at allocate | **Always = 1** with the fair scheduler. *Not* "got 1 of 20 threads" — it's "the master slot was reserved." Commonly misread. |
| `ConcurrencyControlSlotsDelayed` | `+= (limit − granted)` at allocate | Competing slots **not available at start**. `=19` means the budget was full the instant the query began. |
| `ConcurrencyControlQueriesDelayed` | `+= 1` if any delay | This query had to wait for ≥1 competing slot. |
| `ConcurrencyControlSlotsAcquiredNonCompeting` | `+= 1` when a thread takes the master slot | The master thread started. Essentially always 1. |
| `ConcurrencyControlSlotsAcquired` | `+= 1` per **competing** slot a thread picks up | **The real measure of extra parallelism the query got.** |

So **effective parallelism ≈ 1 (master) + `ConcurrencyControlSlotsAcquired`**, and the gap between that and `max_threads` is your throttling.

### The current (gauge) metrics (in `system.metrics`)

| Metric | Meaning |
|---|---|
| `ConcurrencyControlAcquired` | Competing slots currently held across all queries. Pinned at the limit = saturated. |
| `ConcurrencyControlAcquiredNonCompeting` | Master slots currently held ≈ number of running queries. |

A quick health check:

```sql
SELECT metric, value FROM system.metrics
WHERE metric LIKE 'ConcurrencyControl%';
```

If `ConcurrencyControlAcquired` sits pinned at the soft limit, the cluster is continuously slot-starved — exactly the `Delayed=19`-on-every-query pattern from Step 4.

---

## The fix

The "broken pipe + 7 s thread creation" incident and the "every query throttled to ~1 thread" pattern are the **same root cause**: too many concurrent threads requested against the CPU budget, with `max_threads=20` as the multiplier.

**1. Lower `max_threads` (the #1 lever, per-query / profile).** These real-time queries scan tens of thousands of rows and finish sub-second; 20 threads is wasteful. Each query at `max_threads=20` demands 19 competing slots, so a burst exhausts the budget instantly. Dropping to **6–8** both shrinks per-query demand *and* raises effective concurrency:

| Config | Slots | Competing/query | Max fully-parallel queries | OS oversubscription |
|---|---|---|---|---|
| ratio 4, mt=20 | 720 | 19 | ~37 | **4×** (CFS throttle) |
| ratio 2, mt=8 | 360 | 7 | **~51** | 2× |
| ratio 1, mt=8 | 180 | 7 | ~25 | **1× (no throttle)** |

**2. Lower `concurrent_threads_soft_limit_ratio_to_cores` 4 → 2** so the budget matches real cores and the kernel stops CFS-throttling the cgroup. Don't go to 1 *until* `max_threads` is reduced, or you worsen slot starvation. Runtime-reloadable.

**3. Reduce query volume / fan-out** — chronic slot starvation means structural demand exceeds capacity. Lean on the query cache, reduce rollups fired per preload, or scale replicas.

> A subtlety worth remembering: lowering the ratio *reduces* the slot budget, so on its own it makes `Delayed` worse. That's why `max_threads` reduction comes first — it's what makes a smaller, non-oversubscribed pool actually sufficient.

## Takeaways

- A "Broken pipe" `NETWORK_ERROR` on a distributed query is often **hedged requests** cancelling a slow replica — benign, but a signal that some replica was slow.
- Multi-second **thread creation** time is a fingerprint of **cgroup CFS throttling**, i.e. CPU oversubscription.
- `ConcurrencyControlSlotsGranted=1` does **not** mean "1 thread granted" — with the fair scheduler it's always the master slot. Read `ConcurrencyControlSlotsAcquired` for real parallelism, and `ConcurrencyControlSlotsDelayed` for throttling.
- On Kubernetes, set `concurrent_threads_soft_limit_ratio_to_cores` so the CPU-slot budget matches the pod's CFS quota — and size `max_threads` to the workload, not the box.
