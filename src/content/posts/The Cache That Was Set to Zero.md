---
title: 'The Cache That Was Set to Zero (and Three Wrong Root Causes Before It)'
published: 2026-09-07
description: "56% of queries timing out at 13% CPU. Free threads, idle disks, and 99% of thread time in no counter at all. A four-day hunt through a thread pool that was never full, a merge storm that never mattered, and a throughput ceiling nobody had measured — ending at a config value I had printed on the first day and walked straight past, a metric the bug had fabricated to point me at the wrong subsystem, and an upstream fix that had already been merged four months earlier."
image: ''
tags: [ClickHouse, Performance, Debugging, Observability, Big Data Engineering, Concurrency, Caching]
category: 'Coding'
draft: false
lang: 'en'
slug: the-cache-that-was-set-to-zero
---

## A symptom that doesn't parse

> "We're seeing a lot of query failures during peak hours, but the cluster's CPU
> is only 15-20%. What's the bottleneck?"

Six bare-metal nodes, three shards, two replicas each. 128 cores and a terabyte
of RAM per node. During the daily peak, **56% of queries failed** with
`TIMEOUT_EXCEEDED`. During that same peak the CPU sat at **13%**, the disks at
**0.8%**, and the load average was **14.5** on machines with 128 cores.

That combination is not supposed to exist. A cluster that's out of capacity is
*busy*. This one was asleep.

The interesting thing about this investigation isn't the answer. It's that I got
the answer wrong three times first, and the decisive clue was sitting in my
terminal on day one — in the very first configuration dump I ran.

## The shape of the failure

First, the daily numbers. `query_log` only kept 30 days, but that was enough:

```
date         initial_q    timeouts    avg_ms    p99_ms
2026-08-29      18,663           0         2       ...
2026-08-31      17,542           0         2       ...
2026-09-01      78,318       9,484    11,325    96,143
2026-09-02     919,395     182,437    26,796   202,373
2026-09-03   1,092,753     189,882    22,346   167,097
```

This wasn't a slow regression. Through August the cluster served ~18,000 queries
a day at 2ms with **zero** timeouts — health-check traffic. Then production was
cut over on September 1st and the load went up **52×** in two days.

`metric_log` kept four months, and it confirmed the cleanest onset I've ever
seen: the queue of skip-index tasks was **exactly zero on all six nodes for
seventeen consecutive weeks**, then non-zero forever after.

:::tip
`query_log` and `metric_log` usually have different TTLs. I nearly published a
conclusion about a "long-standing pattern" from one night of `query_log` before
noticing `metric_log` went back to May. Check retention *before* you characterize
a trend.
:::

Within a single day the picture was stranger still:

```
hour    queries    data read    avg duration
15:00    62,976     16.78 TiB       2,051 ms
23:00    70,803      7.78 TiB      54,892 ms
```

12% more queries, **half the data read**, and 27× the latency. The cluster was
doing *less work* and taking *far longer* to do it. That rules out a workload
change and points at something structural.

## Wrong cause #1: the thread pool

One shard was doing all the suffering. Both replicas of shard 1 (`ch201-8` and
`ch201-10`) ran at ~36,000ms per sub-query while shards 2 and 3 sat at ~350ms.
Both replicas degraded *identically*, which rules out a single bad disk.

Digging into where the time went on those nodes:

```
FilteringMarksWithSecondaryKeysMicroseconds   448,615,811   <-- 448.6 s (91% of wall)
RealTimeMicroseconds                          494,835,642   <-- 494.8 s
UserTimeMicroseconds                              848,413   <--   0.85 s CPU
OSIOWaitMicroseconds                                    0
OSReadBytes                                       487,424   <-- basically nothing
```

91% of wall-clock inside skip-index filtering, with almost no CPU and no I/O. The
tables carried up to five `bloom_filter(0.001)` indexes at `GRANULARITY 1`, and
in the source the per-query index pool does this:

```cpp
// MergeTreeDataSelectExecutor.cpp
ThreadPool pool(..., num_threads);
/// Instances of ThreadPool "borrow" threads from the global thread pool.
/// We intentionally use scheduleOrThrow here to avoid a deadlock.
for (size_t part_index = 0; part_index < parts_with_ranges.size(); ++part_index)
    pool.scheduleOrThrow(...);
```

One task **per part**, borrowed from the global pool, ~265 parts per query, and a
`CANNOT_SCHEDULE_TASK` error in the logs saying `no free thread`. Thread pool
exhaustion. Obvious.

It was wrong, and two numbers say so.

First, that error appeared **twice** in a window with 224,469 timeouts. I'd built
a theory on a rounding error.

Second, when I actually measured the pool:

```
hour  concurrency  GlobalThread  Active   IDLE   in skip-index filter
14:00          11         1,898   1,005    893                     61
17:00         291         3,696   3,181    514                  1,916
20:00         555         5,069   4,552    516                  2,626
23:00         654         6,192   5,689    503                  3,843
```

`max_thread_pool_size` was 10,000. The pool never averaged above 6,192 and
**always kept ~500 idle threads in reserve**. Free threads at every moment.

The killer: per-query skip-index cost was *worst* at 17:00 (**49,577ms**) when
the pool was only **31% full**, and three times *better* at 22:30 when it was 73%
full. The cost didn't track pool occupancy in either direction.

## The measurement that should have ended it on day one

Frustrated, I did something I should have done immediately: added up every
time-valued counter ClickHouse exposes for a query and compared the sum to the
total.

```
RealTimeMicroseconds                     24,358 ms
  UserTime                                  207 ms
  SystemTime                                 29 ms
  DiskReadElapsed                             9 ms
  OSIOWait / OSCPUWait                        0 ms
  NetworkReceive / NetworkSend                0 ms
  ConcurrencyControlWait                      0 ms
  PartsLockWait                               0 ms
  WaitMarksLoad                               0 ms
  ─────────────────────────────────────────────────
  accounted                                 245 ms   1.0%
  UNACCOUNTED                            24,113 ms  99.0%
```

**99% of thread time was in no instrumented category at all.**

`trace_log` agreed independently: 189 million `Real` samples against 2.1 million
`CPU` samples — a 90:1 ratio, meaning ~99% of sampled thread time was off-CPU.
And with `node_load1` at 14.5 while 3,844 threads sat inside skip-index
filtering, those threads weren't competing for CPU. They were **asleep**.

This is the most useful thing in the whole investigation, and it's a technique
worth stealing: **budget the time**. If wall-clock vastly exceeds the sum of
everything you can name, stop hypothesizing about the things you *can* measure.
The answer is in a code path with no counter of its own.

I wrote that down. Then I ignored my own finding and went looking at merges.

## Wrong cause #2: merge load

Shard 1 ingested 44% more rows in 37% smaller batches — 258,052 parts a day
versus shard 3's 113,169 — and its merge threads ran at an **80.5% duty cycle**
against 49.5% elsewhere. In a "healthy" window its queries were 27% slower.
Background merges competing with queries. Case closed.

Wrong again, and this one embarrasses me more, because there was a natural
experiment sitting right there. Merge load varies by hour independently of query
load. So: hold concurrency near 1, sort the hours by merge duty cycle, and see if
latency follows.

```
hour  concurrency  s1 merge  s2/3 merge   s1 ms   s2/3 ms   ratio
  10            1     52.4%       39.2%     149       143    1.04
   9            1     57.1%       49.8%     152       147    1.03
  11            1     59.7%       40.5%     161       165    0.98
   6            1     86.8%       44.8%     175       165    1.06
   5            1    101.1%       68.5%     215       211    1.02
   4            2    131.5%       66.3%     314       281    1.12
```

At hour 4, shard 1 carries **double** the merge load of shard 3 and its queries
are 12% slower. At hour 5, near-doubled merge load costs 2%. **No correlation.**

So where did my 27% come from? I'd measured the "healthy window" as 12:00–14:00
— but concurrency there was already 2 to 11, not idle. That window's own ratios
were 1.05 at concurrency 2, 1.18 at 7, 1.39 at 11. **I had measured queueing
amplification and labelled it a baseline.** At genuine idle, the two shard groups
differed by 3%: 176ms against 171ms.

:::caution
Two metric traps bit me here. `CurrentMetric_Merge` is a **concurrency gauge**,
not a duty cycle — it reads a flat `1` on every node even when one is doing 2.6×
the merging. Use `sum(duration_ms)` from `part_log`. And `CurrentMetric_Query`
counts **coordinator** queries too, so a node showing "238 concurrent" had ~4
local sub-queries and 234 coordinators blocked on a different shard.
:::

## The shape underneath: a throughput ceiling

Having been wrong twice by reasoning about mechanisms, I switched to measuring
behaviour. If something is serializing, throughput should plateau regardless of
how much you throw at it.

Counting **all** terminal events — completions *and* failures, so a rising
failure rate can't hide a rising throughput:

```
node      concurrency   finish/s   fail/s   TOTAL departures/s
ch201-8             7        9.2      0.0                  9.2
ch201-8            50       12.9      0.1                 13.0
ch201-8           318       11.5      2.1                 13.5
ch201-8           653        8.1      4.8                 13.0
```

Concurrency rises **90×** and the departure rate does not move. It sits between
9 and 13.5 the entire time.

Deriving *local* arrival rate from `query_log` rather than trusting the
coordinator-contaminated gauge, the cliff came into focus:

```
hour       ch201-8 (shard 1)              ch202-10 (shard 2)
      arr/s    avg_ms   local_conc     arr/s   avg_ms   local_conc
10      3.4       146          0.5       3.4      133          0.4
13      9.2       392          3.6       9.0      320          2.9
15     11.5       536          6.2      11.4      416          4.7
16     13.0     2,380         30.9      12.6      747          9.4   <-- cliff
17     13.5    16,992        230.1      10.6      392          4.1
23     13.0    36,541        474.0       8.8      442          3.9
```

The two nodes are **identical** up to 11.5 arrivals/sec. The cliff sits between
**12.6 and 13.0**. Past it, shard 1's arrival rate pins at 12.7–13.5 — that *is*
its service rate — so utilization hits 1.0 and the queue grows without bound.

And a detail that reframed the whole incident: shard 2 wasn't healthy by design.
It touched 12.6 arrivals/sec — within 3% of the cliff — and fell back only
because shard 1 collapsed first and blocked the coordinators. **Shard 1's failure
was accidentally protecting the other two shards.**

So the claim became quantitative and falsifiable: *each node has a hard ceiling
of ~13 sub-queries/sec — about 77ms of serialized time per sub-query — that no
measured resource explains.* Much better than "an unidentified lock, by
elimination."

I still didn't know what it was.

## The answer

It was `index_uncompressed_cache_size`, and it was set to **0**.

```xml
<index_uncompressed_cache_policy>SLRU</index_uncompressed_cache_policy>
<index_uncompressed_cache_size>34359738368</index_uncompressed_cache_size>  <!-- 32 GiB, was 0 -->
<index_uncompressed_cache_size_ratio>0.5</index_uncompressed_cache_size_ratio>
```

`index_uncompressed_cache_size` **defaults to 0** — disabled. For most workloads
that's harmless. For a workload built on bloom skip indexes at `GRANULARITY 1`,
where 98% of sub-queries consult a skip index, it is catastrophic: every index
granule gets decompressed under the cache mutex on **every single access**.
Concurrent queries pile up inside `filterPartsByPrimaryKeyAndSkipIndexes` while
the CPU idles.

That's the ~77ms of serialized time. That's the 13/sec ceiling. That's the 99%
of thread time in no counter — because "blocked on a cache mutex" doesn't have a
ProfileEvent.

One file. Hot-reloadable. No restart.

And it's worse than simply "off," which the source makes clear:

```cpp
// Context.cpp — called unconditionally at startup, whatever the size
void Context::setIndexUncompressedCache(const String & cache_policy, size_t max_size_in_bytes, double size_ratio)
{
    ...
    shared->index_uncompressed_cache = std::make_shared<UncompressedCache>(
        cache_policy, ..., max_size_in_bytes, size_ratio);
}
```

The cache object is **always constructed**, and `getIndexUncompressedCache()`
returns a valid non-null pointer. So size 0 doesn't bypass the cache — it creates
a **live cache with zero capacity**. Every index granule read still enters
`CacheBase::getOrSet`, takes the global mutex, always misses, holds a per-key
token mutex across the decompression, and re-takes the global mutex to insert —
only to evict immediately. Two global-mutex acquisitions and a
decompression-length hold, per granule, at a 0% hit rate *by construction*.

A disabled cache would have been fine. A zero-capacity cache is a pure mutex.

## It's a known bug, fixed four months earlier

After all that, it turns out this is
[ClickHouse PR #104063](https://github.com/ClickHouse/ClickHouse/pull/104063) —
"Bypass index uncompressed cache when its size is zero" — merged **four months
before our incident**. The PR describes the failure better than I did:

> …every skip-index block read acquired the global `CacheBase` mutex twice,
> evicted itself via `removeOverflow`, and unconditionally incremented
> `UncompressedCacheMisses` and `UncompressedCacheWeightLost`. With
> small-`GRANULARITY` skip indexes (`ngrambf_v1`, `bloom_filter`, etc.) this can
> produce hundreds of thousands of contended mutex acquisitions per query and
> tens of GB of phantom `UncompressedCacheWeightLost`, with zero
> `UncompressedCacheHits`.

We were on a patch release that predated the backport, in the same LTS line. A
version bump would have fixed it.

The regression story is the interesting part. From the PR: the original 2021 code
only constructed the cache when the configured size was non-zero. That `if (size)`
guard was later **removed so the cache could be resized at runtime** via
`SYSTEM RELOAD CONFIG`. The *data* uncompressed cache survived that change,
because `MergeTreeReadPoolBase` independently gates it on
`pool_settings.use_uncompressed_cache`. The index-read path had no equivalent
guard, so it quietly lost its protection and nobody noticed for years — because
you only feel it with low-granularity skip indexes under real concurrency.

The fix adds an explicit `index_uncompressed_cache_enabled` flag and returns
`nullptr` from the getter, so `MergeTreeReaderStream::init` takes the `else`
branch it was always supposed to take.

:::note
Worth saying plainly: the diagnosis was right, the mechanism was right, and the
fix worked — and all of it was still four months of rediscovering something
already fixed upstream. Reading the changelog for the versions between yours and
current is cheaper than any of this.
:::

## The proof

Everything above is inference from counters. Three days after the fix I finally
got a database user with introspection privileges, which made `system.trace_log`
symbolizable — and `trace_log` on this cluster retains **four months**, so the
pre-fix stacks were still sitting there.

Aggregating off-CPU (`trace_type='Real'`) samples from the collapse window:

```
frame                                                        samples
filterPartsByPrimaryKeyAndSkipIndexes(...)::$_0                77,144
DB::ReadBuffer::next()                                         76,315
DB::MergeTreeIndexReader::read(...)                            73,820
DB::MergeTreeIndexGranuleBloomFilter::deserializeBinary(...)   73,697
DB::CachedCompressedReadBuffer::nextImpl()                     72,951   <-- calls getOrSet()
```

**95% of the skip-index-filter samples are inside
`CachedCompressedReadBuffer::nextImpl`** — the function that calls
`cache->getOrSet()`. Skip-index filtering, into the index reader, into bloom
granule deserialization, into the cached read buffer, into the mutex. Measured,
not reasoned.

:::tip
`trace_type='Real'` samples wall-clock time including off-CPU, which is exactly
what you want for blocked threads. Plain `perf record` and `trace_type='CPU'`
sample on-CPU time and would have shown a mostly-idle machine. If your threads
are asleep, profile the sleeping.
:::

## Why I walked past it

Here's the part worth writing down. On day one I dumped the cache configuration
and got this back:

```
name                            value
index_mark_cache_size           5.00 GiB
index_uncompressed_cache_size   0.00 B     <-- right there
mark_cache_size                10.00 GiB
uncompressed_cache_size        50.00 GiB
```

I saw it. I even speculated in my notes that a zero-sized cache might pay the
full mutex cost for a 0% hit rate. And then I chased the **data** uncompressed
cache instead — because that one had a visibly terrible 8.5% hit rate and was
evicting 1.29 GB per query.

The reason that misled me is worse than shared counters, and the upstream PR
spells it out: the zero-capacity index cache **unconditionally incremented
`UncompressedCacheMisses` and `UncompressedCacheWeightLost`** on every single
read. Those events are shared between the two caches, so the "terrible data
cache" I was staring at was **phantom accounting fabricated by the bug itself**.

That's an unusually nasty failure mode. The broken subsystem didn't just hide —
it generated a plausible, quantitative, *wrong* signal that pointed at its
neighbour. I followed the evidence, and the evidence was manufactured. The 1.29 GB
"evicted per query" never happened; nothing was ever cached to evict.

I ran a test, found that disabling the data cache made queries 27% faster, and
recommended it. That was real — turning it off partly avoided the broken path.
It was also **the opposite direction from the fix** and roughly 150× smaller in
effect.

:::caution
A measured improvement is not proof that you understand the mechanism behind it.
My 27% win was genuine, reproducible, and pointed the wrong way.
:::

The lesson I'd actually hand to someone: **a configuration value of `0` is a
first-class suspect, not a curiosity.** Zero means "this subsystem is turned off,"
and a subsystem that's off does not appear in any of the metrics you'd normally
use to find it. It generates no hit rate, no eviction count, no wait time. It is
invisible precisely because it isn't running.

## The verification

The fix went out at 09:07 UTC on a Friday. The hourly data makes the causality
unusually clean:

```
hour     queries   timeouts   avg_ms   skip-index ms/sub-query
07:00     22,431        396    7,675                    33,153
08:00     23,984      1,940    9,235                    15,312
09:00     18,251        171    1,479                       387   <-- deploy 09:07
10:00     19,026          0      212                        76
15:00     62,346          1      222                        86
```

A 200–400× drop, inside the hour of the deployment.

Then the weekend — and critically, **Saturday was the busiest day of the entire
period**, which rules out "traffic just went down":

```
date          initial_q   timeouts   fail%   avg_ms   p99_ms
2026-09-03    1,092,753    189,882   21.2%   22,346  167,097
2026-09-05    1,309,381         28    0.0%      260    1,834
```

20% more queries. 189,882 timeouts became **28**.

The mechanism check is my favourite table of the whole investigation:

```
                        before      after
skip-index time      17,967 ms   106.9 ms    -99.4%
RealTime             24,358 ms     862 ms      -96%
CPU time                207 ms     652 ms     +215%   <-- up!
RealTime : CPU          118 : 1     1.3 : 1
cache hit rate            6.2%      98.5%
sub-queries/hour         58,776    103,841      +77%
```

**CPU time went up 215% while wall time fell 96%.** That's the proof. A change
that merely reduced work would have lowered both. Instead the queries stopped
sleeping and started computing. The ratio went from 118:1 to 1.3:1 — from
"almost entirely blocked" to "almost entirely working."

The ceiling moved too: every node now sustains **18.8 arrivals/sec at ~85ms**,
against the old hard stop of 13.5 at 17,000–42,000ms.

And the same stack measurement, before and after, on one node in comparable
20-minute peak windows:

```
                        total blocked samples    in the mutex path    share
pre-fix   09-02 16:40              330,728              72,952       22.1%
post-fix  09-06 22:40               20,214                 222        1.1%
```

Absolute blocked thread-time fell **16×**. Time in that one code path fell
**329×**.

## Which node? The question I couldn't answer

One loose end nagged at me. The broken setting was cluster-wide — every node had
the same zero-capacity cache. So why did one shard eat the failures while the
others stayed at 300ms?

I tested five explanations and every one collapsed. Intrinsic node speed: equal
at matched idle load, and the ranking *flips between days* — on the day the
"victim" shard first collapsed, it was the cheapest of the three. Data volume,
merge load, initial-query routing, hidden coordinator-local reads: all even, and
several pointed at a *different* shard than the one that failed.

Then I looked at minute resolution instead of daily averages, and the premise
fell apart:

```
time     s1_ms    s2_ms    s3_ms
15:40    7,498      807    4,127   <-- s1 and s3 both spike
15:50   15,059      377      327   <-- s1 runs away
16:20   23,754      277      766
16:50      317      287   19,560   <-- s3 collapses, s1 RECOVERS
```

**The victim moves.** Daily averages had hidden it completely. And because a
shard holds the role for hours once it tips, a peak is one to three independent
events, not sixty — so "three days running" is about four coin flips, not a
pattern.

The symbolized stacks then explained why nothing could predict it. At the
*pre-cliff* window, when every node still looked healthy at 400–500ms:

```
node               blocked samples    in the mutex path    share
ch201-10 (shard 1)         114,353              62,969     55.1%
ch202-10 (shard 2)         113,352              60,554     53.4%
ch201-8  (shard 1)         100,656              50,774     50.4%
ch205-10 (shard 3)         102,151              49,423     48.4%
ch204-10 (shard 3)          92,284              39,834     43.2%
```

**Between 43% and 55% of all blocked thread time on every node was already in
the failing path before anything visibly broke.** And the ranking doesn't order
the outcomes — `ch202-10` sat at 53.4% and never collapsed once; `ch204-10` sat
lowest at 43.2% and did.

Six machines sitting uniformly at the edge of one shared limit. Which one went
over was noise. What kept it there was that nothing in the stack routes away
from a slow replica: load balancing was `random`, and hedged requests — which I
briefly thought would have helped — only fire when a replica goes **silent** for
two seconds. A replica that is slow but still streaming progress packets
explicitly *disables* replica switching. It would also have redirected work onto
the other replica of the same saturated shard.

And the cleanest epilogue: the ingest skew that I'd blamed in round two is
**completely unchanged** — shard 1 still runs a 118% merge duty cycle against
shard 3's 75%. Same skew, and its queries now answer in 79–98ms against shard
3's 71–100ms. Identical. A better controlled experiment than the one I ran
myself.

## What I'd do differently

**Budget the time first.** The 99%-unaccounted measurement took one query and
would have redirected the entire investigation on day one. I ran it on day three.

**Treat zeros as suspects.** Any capacity, size, or limit reading `0` deserves an
explicit "is this intentional, and what does off cost us?" before you look
anywhere else.

**Know which counters are shared — and which are fabricated.** I spent a day on
the data uncompressed cache because a broken subsystem was writing invented
misses and evictions into counters I attributed to its neighbour. Shared metrics
are bad enough; metrics a bug actively manufactures are worse, because they look
like exactly the kind of hard evidence you're taught to trust.

**Read the changelog between your version and current.** The whole thing was
fixed upstream four months before it hit us, in a patch release of the same LTS
line. That's a boring lesson and it would have saved all of it.

**Match on load before claiming a baseline.** Two of my three wrong answers came
from measuring "idle" behaviour in a window that wasn't idle.

**Prefer measuring behaviour to reasoning about mechanism.** I was wrong twice
reasoning from architecture — thread pools fan out, merges compete for CPU, both
plausible. The throughput-ceiling measurement made no assumption about mechanism
and was right the first time.

**Profile the sleeping.** When threads are blocked, on-CPU profiling shows an
idle machine. `trace_type='Real'` (or `offcputime` from BCC) samples wall-clock
including off-CPU, and it named the function in a single query. I spent three
days inferring what one properly-scoped profile would have told me — I just
didn't have the grant, and didn't think to ask for it early.

**Distrust daily averages on anything that can flip.** "It's always shard 1" was
an artifact of aggregation. At minute resolution the victim moved, and that one
fact dissolved a question I'd burned two rounds on.

And one on temperament. This investigation produced three confident, wrong root
causes before the right one. Every one was plausible, supported by real numbers,
and would have survived a casual review. What killed each wasn't a better theory
— it was a measurement designed specifically to falsify the theory I already
liked. The thread-pool answer died to "how full was the pool, actually." The
merge answer died to "hold concurrency constant and vary merges." The
shard-1 answer died to "stop averaging by day."

The uncomfortable part is that the final answer was visible in the first hour, in
a config dump I'd already read. Debugging isn't usually short of data. It's short
of the willingness to take your own evidence literally.
