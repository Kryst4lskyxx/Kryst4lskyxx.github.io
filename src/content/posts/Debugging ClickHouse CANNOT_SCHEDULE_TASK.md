---
title: 'Debugging ClickHouse CANNOT_SCHEDULE_TASK: The Thread That Could Not Spawn'
published: 2026-06-01
description: 'A 2400-error storm of CANNOT_SCHEDULE_TASK, five OS limits ruled out one by one, and the metric_log query that finally caught the real culprit — a query-admission stampede, not a thread-pool ceiling.'
image: ''
tags: [ClickHouse, Big Data Engineering, Performance Tuning, Concurrency, Kubernetes, Debugging]
category: 'Coding'
draft: false
lang: 'en'
slug: debugging-clickhouse-cannot-schedule-task
---

## The symptom

A real-time alerting cluster (distributed queries with `GROUP BY GROUPING SETS`, fired per minute per customer) started throwing error 439 in bursts. An aggregation over two days surfaced ~2400 of them:

```
Code: 439. DB::Exception: Cannot schedule a task:
  failed to start the thread (threads=3, jobs=3). (CANNOT_SCHEDULE_TASK)

Code: 439. DB::Exception: Cannot schedule a task:
  no free thread (timeout=0) (threads=10000, jobs=10000). (CANNOT_SCHEDULE_TASK)
```

Two things were strange immediately:

1. The **dominant** message was `failed to start the thread` with *tiny* counts — `threads=0`, `threads=3`, `threads=9` — not the `threads=10000` you would expect from an exhausted pool.
2. The errors arrived in tight per-second clusters (e.g. every entry stamped `08:37:59`–`08:38:00`), then nothing for hours.

This post is the investigation. The short version: it was **not** the thread-pool limit, **not** memory, and **not** any OS quota. It was a query-admission stampede, and the proof came from ClickHouse's own `metric_log`. The fix is one config line. Getting there took ruling out five wrong answers, which is the more useful story.

The cluster: 4 shards × 2 replicas, Kubernetes pods at `cpu: 180`, `memory: 680Gi` (Guaranteed QoS), ClickHouse 25.8. This is the same cluster from [Diagnosing ClickHouse Concurrency Control](./diagnosing-clickhouse-concurrency-control/); different failure mode.

## Reading the error: two failures hiding under one code

Error 439 comes from the thread pool's scheduler, `ThreadPoolImpl::scheduleImpl` in `src/Common/ThreadPool.cpp`. There are **two distinct branches** that both produce `CANNOT_SCHEDULE_TASK`, and telling them apart is the whole game.

**Branch A — the pool's own limit (`no free thread`):**

```cpp
// src/Common/ThreadPool.cpp  (~line 331)
if (!job_finished.wait_for(lock, std::chrono::microseconds(*wait_microseconds), pred))
    return on_error(fmt::format("no free thread (timeout={})", *wait_microseconds));
```

The `(threads=N, jobs=N)` in this message is `threads.size()` and `scheduled_jobs`. `threads=10000` means the global pool is genuinely at `max_thread_pool_size` (default 10000) with the queue full. This is "ClickHouse said no."

**Branch B — the OS refused (`failed to start the thread`):**

```cpp
// src/Common/ThreadPool.cpp  (~lines 300-308)
try
{
    new_thread = std::make_unique<ThreadFromThreadPool>(*this);
}
catch (...)
{
    remaining_pool_capacity.fetch_add(1, std::memory_order_relaxed);
    return on_error("failed to start the thread");   // <-- our error
}
```

And the constructor that throws:

```cpp
// src/Common/ThreadPool.cpp  (~line 614)
ThreadPoolImpl<Thread>::ThreadFromThreadPool::ThreadFromThreadPool(...)
{
    thread = Thread(&ThreadFromThreadPool::worker, this);   // std::thread ctor -> pthread_create
}
```

For the global pool, `Thread` is `std::thread`. Its constructor calls `pthread_create`. If the **kernel** refuses, the constructor throws `std::system_error`, the `catch` fires, and you get `failed to start the thread`. The `threads=3` here is just how many threads *that particular pool* held when the OS said no — and it can be tiny, because the pool was nowhere near its own limit.

So the dominant error was telling us, unambiguously: **the operating system refused `pthread_create` while ClickHouse's pool was almost empty.** The rare `threads=10000` variant is a secondary mode (a pool that occasionally did saturate). The 2400 errors were Branch B.

The first call into this path for any SELECT returning to a client is the async pipeline driver:

```cpp
// src/Processors/Executors/PullingAsyncPipelineExecutor.cpp  (~line 105)
data->thread = ThreadFromGlobalPool(std::move(func));
```

That single driver thread is what fails first when the kernel is refusing threads.

## Wrong answer #1: "the pool is too small, raise max_thread_pool_size"

The tempting fix. It is wrong, and there is clean evidence it is wrong.

A community report on the identical error ran with `max_thread_pool_size=20000` and **still failed at ~15000 threads**. If this were ClickHouse's pool limit, it would fail at exactly 20000 with `no free thread (threads=20000)`. Failing *below* the limit, with `failed to start the thread`, is only possible if the kernel refused. Raising the ClickHouse ceiling cannot fix a kernel refusal. Scratch that idea before touching it.

So the real question became: **why does the kernel refuse `pthread_create`?** `pthread_create` fails with `EAGAIN` or `ENOMEM`, and the causes are a finite list. On a Kubernetes pod the suspects are:

- cgroup `pids.max` (pod task limit)
- `vm.max_map_count` (each thread stack is mapped memory)
- `RLIMIT_NPROC` (`ulimit -u`)
- `kernel.threads-max` / `kernel.pid_max`
- cgroup memory hard limit (no room for the stack)
- strict overcommit refusing the stack reservation

Time to check them, on the pod, one at a time.

## Wrong answers #2–#6: ruling out every limit

```text
$ cat /sys/fs/cgroup/pids.current ; cat /sys/fs/cgroup/pids.max
5413
629145                       # not close

$ cat /proc/sys/vm/max_map_count
1048576                      # already raised; default 65536 would have been the classic culprit

$ PID=$(pidof clickhouse-server); cat /proc/$PID/maps | wc -l
12260                       # vs a 1,048,576 cap — irrelevant
```

Three down. Then the limits file, which is authoritative for what the *running server* actually has:

```text
$ cat /proc/$PID/limits
Max processes             unlimited   unlimited   processes   # RLIMIT_NPROC: ruled out
Max open files            1048576     1048576     files       # fine
Max stack size            8388608     unlimited   bytes       # 8 MB default thread stack

$ cat /proc/sys/kernel/threads-max
6184551                    # ruled out
$ cat /proc/sys/kernel/pid_max
4194304                    # ruled out
```

And memory:

```text
$ cat /sys/fs/cgroup/memory.events
low 0
high 0
max 0                      # the cgroup memory limit was NEVER hit
oom 0
oom_kill 0

$ cat /sys/fs/cgroup/memory.stat | grep -E '^(anon|file) '
anon 156424089600          # 145 GiB unreclaimable  (of 680 GiB)
file 340788776960          # 317 GiB page cache (reclaimable, harmless)

$ cat /proc/sys/vm/overcommit_memory
1                          # 1 = always overcommit; the kernel NEVER refuses on commit
```

That `memory.events: max 0` is strong: the process had been up for `etimes ≈ 262704s` (~3 days), spanning **every** incident, and the cgroup memory cap was never touched once. Combined with `overcommit_memory=1`, both memory mechanisms are dead.

Every standard limit is ruled out. Yet `std::thread`'s constructor provably throws. At this point you have to admit the **method** is wrong, not the next guess: every check above was run on an **idle** pod (`pids.current=5413` is the resting baseline), but the failures happen only at the *peak* of 30-second bursts. Idle snapshots cannot see a transient exhaustion.

Stop probing live idle state. ClickHouse already recorded the failure moment.

## The smoking gun: query the failure second

ClickHouse logs per-second gauges to `system.metric_log` and `system.asynchronous_metric_log`. Pull the window around a known incident:

```sql
SELECT event_time,
       CurrentMetric_GlobalThread        AS gthread,
       CurrentMetric_GlobalThreadActive  AS gactive,
       CurrentMetric_Query               AS queries,
       CurrentMetric_TCPConnection       AS tcp,
       formatReadableSize(CurrentMetric_MemoryTracking) AS mem
FROM system.metric_log
WHERE event_time BETWEEN '2026-06-01 08:37:55' AND '2026-06-01 08:38:03'
ORDER BY event_time;
```

```text
event_time            gthread  gactive  queries  tcp    mem
2026-06-01 08:37:58   1785     890      2        1171   65.98 GiB
2026-06-01 08:37:59   2009     1308     279      1171   82.43 GiB   <-- errors fire here
2026-06-01 08:38:00   2009     1668     307      1157   77.88 GiB
2026-06-01 08:38:01   2009     1492     97       1142   79.58 GiB
2026-06-01 08:38:02   2009     1107     21       1144   72.34 GiB
```

And the memory view at the same instant confirmed the cgroup was at ~75% with hundreds of GiB free:

```text
CGroupMemoryTotal  730144440320   (680 GiB)
CGroupMemoryUsed   538204315648   (~501 GiB, ~74%)  at 08:38:00
OSMemoryAvailable  625301966848   (~582 GiB free)
```

Read the table. The thread pool failed `clone()` while sitting at **2009 global threads** — against a 10000 limit, on a process that runs **5538 threads at rest**, with memory at 74% and every quota miles away. The absolute thread count was never the constraint.

**The only column that moves is `queries`: 2 → 279 → 307 in one second.**

That is a query-admission stampede. 279 distributed queries landed in the same second. Each one needs a pipeline driver thread plus remote-connection threads plus local pools — none of which are governed by the CPU-slot soft limit. Hundreds of queries demanding thread creation in the same instant asked the kernel to `clone()` several thousand threads *right now*, and the kernel could not deliver them fast enough. The scattered tiny counts in the original errors (`threads=0..19`) are exactly this: hundreds of separate pools each trying to expand in the same second, some losing the race.

## Root cause

It is a thread-creation **rate** failure under a query stampede, not a quota ceiling. With every `EAGAIN` source ruled out (`RLIMIT_NPROC` unlimited, `threads-max` 6.1M, `pid_max` 4.2M) and `overcommit_memory=1` ruling out commit-refusal, the `clone()` failure is almost certainly transient `ENOMEM` — the kernel briefly cannot satisfy thread-stack and kernel-stack allocations during a burst of thousands of clones per second under heavy allocator churn (`jemalloc.mapped` swung ~50 GiB across these seconds). The proximate trigger, provable from the data, is that **`max_concurrent_queries` was set to 5000** — effectively unlimited — so nothing throttled the 279-query stampede before it hit the kernel.

A note on honesty: I did not capture the exact `errno` (`EAGAIN` vs `ENOMEM`). The evidence makes `ENOMEM`-under-burst overwhelmingly likely, but the only way to be certain is `dmesg -T` around an incident (look for "page allocation failure" / "fork") or `perf trace -e clone,clone3` on a canary during a burst. The fix below does not depend on which it is, because both are cured by not creating thousands of threads in one second.

## The fix

**1. Throttle admission — the primary fix, targeting the proven trigger.**

```xml
<!-- config.xml -->
<!-- was 5000 (effectively unlimited) -->
<max_concurrent_queries>250</max_concurrent_queries>
```

Baseline concurrency on this cluster is single digits; the stampedes spike to ~280–307. A cap of 250 sheds the pathological burst as a retryable `TOO_MANY_SIMULTANEOUS_QUERIES` (a clean per-query error) instead of letting it become a pod-wide `clone()` storm that 439s everything, including in-flight queries. It is a config reload, no restart. Tune by watching `system.errors` for `TOO_MANY_SIMULTANEOUS_QUERIES` and raising if real load is rejected.

**2. Shrink the per-query thread/connection avalanche** so each admitted query contributes fewer simultaneous `clone()`s:

- Lower `max_threads` (these sub-second queries do not need 15–20 threads).
- Reconsider `use_hedged_requests=1` — it opens connections to *both* replicas per shard, doubling remote-connection threads during a burst.
- Collapse multi-way `UNION ALL` over several distributed tables; each branch multiplies the remote fan-out.

**3. Make the kernel robust to spawn bursts** (node-level, since these are non-namespaced sysctls — set via a privileged DaemonSet on the ClickHouse nodes):

- `transparent_hugepage/enabled = madvise` — `always` under fragmentation triggers compaction stalls (visible as a spike in *system* CPU during the errors) and can fail stack allocations.
- raise `vm.min_free_kbytes` to keep a contiguous-memory reserve for thread-stack allocations.

**Do not** raise `max_thread_pool_size` (proven useless) or any `ulimit` (none are hit).

Roll out #1 alone first and confirm the 439 storms stop. Add #2 if they persist during stampedes; add #3 only if `dmesg` shows allocation/compaction failures.

## Takeaways

- **Error 439 is two errors.** `no free thread (threads=10000)` is ClickHouse's pool limit; `failed to start the thread (threads=<small>)` is the *kernel* refusing `pthread_create`. The small count is the tell — read it before you tune anything.
- **Failing below your configured limit means the limit is not the problem.** Raising `max_thread_pool_size` cannot fix a kernel refusal.
- **Idle snapshots lie about bursty failures.** Every OS limit looked healthy at rest. The answer was only visible at the failure second — and `system.metric_log` / `asynchronous_metric_log` already had it recorded, no live reproduction needed.
- **Correlate, do not assume.** The failure tracked exactly one variable — concurrent `Query` count — not threads, not memory, not any quota. A query-admission stampede plus an unbounded `max_concurrent_queries` was the whole story.
- **`pthread_create` can fail with free memory.** Under a high enough spawn *rate*, transient `ENOMEM` from stack allocation (fragmentation, THP compaction) refuses threads while hundreds of GiB sit available. The cure is to lower the rate, not to add memory.
