---
title: "Zero mmap Buffers: The ClickHouse Read Setting That Never Fired"
published: 2026-07-04
description: "min_bytes_to_use_mmap_io promises faster reads by memory-mapping large files instead of copying them through the kernel. I turned it on for a real production workload and it did literally nothing — CreatedReadBufferMMap stayed at 0 for every query. The source explains why the setting almost never triggers for selective analytics, why a synthetic benchmark makes it look great, and why the one time it does fire it can turn a disk error into a server crash."
image: ''
tags: [ClickHouse, Performance Tuning, Storage, Source Reading, Page Cache, Debugging]
category: 'Coding'
draft: false
lang: 'en'
slug: clickhouse-mmap-read-setting
---

## The idea

ClickHouse has a setting that sounds like free performance:

```sql
SET local_filesystem_read_method = 'mmap',
    min_bytes_to_use_mmap_io = 67108864;   -- 64 MiB
```

The pitch: instead of `pread`-ing a large column file into a userspace buffer (a kernel→userspace copy), `mmap` it and let the query read straight from the mapped pages. No copy. The docs even suggest ~64 MiB as the threshold. On a node where the hot table is already resident in the page cache, "skip the copy" felt like it should be a measurable win.

The plan was the same as always: prove it against the *real* workload, not a microbenchmark. Mine the top-10 most-frequent and top-10 most-costly production query patterns from `system.query_log`, replay them with mmap on and off, and measure.

It didn't take a benchmark to find the first surprise. It took one profile-event counter.

## The setting that never engaged

I turned mmap on, ran the production patterns, and checked whether mmap was actually used:

```sql
SELECT ProfileEvents['CreatedReadBufferMMap']    AS mmap_bufs,
       ProfileEvents['CreatedReadBufferOrdinary'] AS ordinary_bufs
FROM system.query_log
WHERE type = 'QueryFinish' AND query_id = '…';
```

`mmap_bufs` was **0**. Not small — zero. Across all ten frequent patterns. Every read went through the ordinary `pread` path despite the settings being set. Raising and lowering the threshold, re-checking the settings actually applied — still zero. The optimization I'd enabled was inert.

The costly patterns were no different. Even the heaviest one — a query reading **330 MiB across 26 million rows** — showed `CreatedReadBufferMMap = 0` at the 64 MiB threshold. Whatever gates mmap, "big query" wasn't enough to trip it.

## Why: the threshold is compared against the wrong thing you'd assume

Time to read the source. Two facts settle it.

First, **both** settings are required, not either one. The mmap branch in the read-buffer factory only fires when all of these hold:

```cpp
// src/Disks/IO/createReadBufferFromFileBase.cpp:51-55
if (!existing_memory
    && settings.local_fs_method == LocalFSReadMethod::mmap   // local_filesystem_read_method='mmap'
    && settings.mmap_threshold                                // min_bytes_to_use_mmap_io > 0
    && settings.mmap_cache
    && estimated_size >= settings.mmap_threshold)             // file/read >= threshold
```

Second — and this is the whole story — what is `estimated_size`? It is **not** the size of the query, nor the size of the column file. It's the size of the *mark ranges this reader is about to read*:

```cpp
// src/Storages/MergeTree/MergeTreeReaderStream.cpp:57
auto [max_mark_range_bytes, sum_mark_range_bytes] = estimateMarkRangeBytes(all_mark_ranges);
```

`sum_mark_range_bytes` becomes the read hint that's compared to the threshold. So mmap engages per **(part, column)** only when the *contiguous slice of that one column, in that one part, that this query needs* is at least the threshold.

Now overlay the workload. These are selective analytics queries: a narrow time window plus a customer filter on the sort key. Each one reads a handful of granules per column. And the table is a real-time table under constant ingestion, so a "big" query's rows are scattered across **hundreds of small parts**. Put those together and every per-part, per-column read is a few tens of KB — nowhere near 64 MiB, or even 1 MiB. The 330 MiB heavy query reads 330 MiB *in aggregate*, fragmented into thousands of tiny per-part slices, none of which crosses the bar. So mmap never fires. Not because the setting is broken — because the workload's reads never look like the one shape mmap is gated for.

I confirmed the negative directly: forcing the threshold down to **4 KiB** finally made `CreatedReadBufferMMap` climb — and the latency difference was noise, single-digit milliseconds either way, sometimes worse. At any *sane* threshold, mmap is a no-op for this workload.

(Two related dead-ends the source ruled out: `storage_file_read_method='mmap'` is `clickhouse-local`-only and ignored by the server, and merges/part-checks force plain `pread` regardless — so mmap can only ever touch the normal SELECT path.)

## The benchmark that would have lied

There *is* a query shape where mmap wins. A full-column scan —

```sql
SELECT count(), sum(length(streamUrl)) FROM events;
```

— reads one column end-to-end, so each part's slice of that column is large and mmap engages. Warm, it came out meaningfully faster with lower CPU and memory:

| method | median | CPU (user) | memory |
|---|---|---|---|
| mmap | **2159 ms** | 15,346 ms | 49.9 MiB |
| pread_threadpool | 2778 ms | 17,600 ms | 61.6 MiB |

~22% faster. If I'd started with *that* benchmark, I'd have shipped mmap cluster-wide and declared victory. But no production query has that shape — the dashboards issue selective, filtered aggregations, not full-column scans. This is the trap: **the microbenchmark measures the one case the real workload never hits.** The profile-event counter (`CreatedReadBufferMMap = 0`) is what kept the synthetic number honest. It's the same discipline as trusting `OSReadBytes` over wall-clock in [the read regression that was actually a write problem](/posts/clickhouse-read-regression-write-problem/) — one internal counter, checked, beats a plausible-looking latency chart.

## The reason it's off by default: a read error becomes a crash

Even if mmap *had* helped, there's a reason it's flagged experimental. With `pread`, a disk read error surfaces as a normal, catchable ClickHouse exception. With mmap, a read fault is delivered as a **signal**:

```text
// src/Storages/StorageFile.cpp:281-282
//  - concurrent modifications of a file will result in SIGBUS;
//  - IO error from the device will result in SIGBUS;
```

and `SIGBUS` is registered in the fatal signal handler:

```cpp
// src/Common/SignalHandlers.cpp:624
addSignalHandler({SIGABRT, SIGSEGV, SIGILL, SIGBUS, SIGSYS, SIGFPE, …}, signalHandler, true);
```

So a bad block under an mmap'd read doesn't fail the query — it takes down the whole server. That's a bad trade for a warm workload: you'd be swapping `pread`'s catchable errors for a crash path, in exchange for a speedup the source says you won't even get.

## Verdict and takeaways

For a selective, high-part-count, real-time analytics workload, **don't enable mmap.** It won't engage at any reasonable threshold, forcing it on buys noise, and the one regime it activates in carries a server-crash risk.

The lessons generalize past this one setting:

1. **Check that an optimization actually engaged before you measure it.** `CreatedReadBufferMMap = 0` ended the investigation in one query — no A/B needed to know the knob did nothing.
2. **Read what the threshold is compared against.** "64 MiB" sounds like "big queries use mmap." The source says it's per-part, per-column *mark-range* bytes — a completely different quantity once ingestion fragments your data into hundreds of parts.
3. **Beware the benchmark that fits the feature instead of the workload.** A full-column scan flatters mmap by ~22%; your dashboards never run one.
4. **"Off by default" usually encodes a hazard.** Here it's `SIGBUS` → crash. The default is often the accumulated scar tissue of everyone who tried the obvious thing first.

This turned out to be the warm-up for a bigger version of the same question — if the data's already in RAM and copies aren't the bottleneck, would a RAM disk help? [Spoiler: it changed nothing either](/posts/clickhouse-ram-disk-vs-page-cache/), for the same underlying reason: the page cache already handed us the memory speed we were trying to buy.
