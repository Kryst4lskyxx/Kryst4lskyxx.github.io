---
title: "The RAM Disk That Changed Nothing: Why tmpfs Can't Beat the Page Cache in ClickHouse"
published: 2026-07-06
description: "The idea sounds obvious: put ClickHouse data on a tmpfs RAM disk so reads never touch the disk. I built it, ran the real production query patterns against a RAM-disk table and an NVMe table side by side, and the answer was zero improvement — for a reason the source code spells out. A walk through reading the read path before running the benchmark, the two confounds that faked a result, and the one thing the RAM disk *did* change: it nearly ran the node out of file descriptors."
image: ''
tags: [ClickHouse, Performance Tuning, Storage, Page Cache, Source Reading, Debugging]
category: 'Coding'
draft: false
lang: 'en'
slug: clickhouse-ram-disk-vs-page-cache
---

## The idea

> We have a node with 750 GB of RAM and the hot table is only ~20 GB. What if we mount a tmpfs RAM disk and put the ClickHouse data on it, so reads are served straight from memory instead of NVMe?

It's a reasonable-sounding optimization, and there are plenty of "how to create a RAM disk on Ubuntu" guides that make it a five-minute job:

```sh
mkdir -p /mnt/chram
mount -t tmpfs -o size=250G tmpfs /mnt/chram
```

Add it as a ClickHouse disk, point a table at it, and reads should fly. The plan was to prove it with the *real* workload: replay the top-10 most-frequent and top-10 most-costly production query patterns against a RAM-disk copy of the table and an NVMe copy, and measure the difference.

I'd just finished [a related experiment — trying `min_bytes_to_use_mmap_io`](/posts/clickhouse-mmap-read-setting/) to memory-map the same table's reads — which had gone nowhere for a subtle reason (mmap only engages when a single contiguous per-column read exceeds the threshold, and this table's selective, many-part reads never do). That experience set the habit that saved this one: **read the read path in the source before you benchmark it.**

## Read the source first

The question is narrow: for a MergeTree read, does *where the bytes live* (tmpfs vs NVMe) change anything, given the working set fits in RAM? The read path answers it.

A column read ultimately calls plain `pread`:

```cpp
// src/IO/ReadBufferFromFileDescriptor.cpp:71
res = ::pread(fd, to + bytes_read, to_read, offset + bytes_read);
```

`pread` on a file whose pages are already in the OS page cache is a copy from a cached RAM page into the query buffer. **A tmpfs read is the same copy from the same kind of RAM page** — tmpfs *is* page cache. When the data is warm, the two are byte-for-byte identical work.

Three more checks confirm there's no hidden fast path a RAM disk could unlock:

- **tmpfs is just a `local` disk.** It registers via `factory.registerDiskType("local", …)` (`src/Disks/DiskLocal.cpp:810`). ClickHouse has no "in-memory disk" that behaves differently.
- **No direct I/O.** `min_bytes_to_use_direct_io = 0`, so reads go through the page cache, not `O_DIRECT`. (tmpfs can't even do `O_DIRECT`.)
- **Writes don't wait on the device.** fsync is off by default (`src/Storages/MergeTree/MergeTreeSettings.cpp:455-459`: `fsync_after_insert=false`, `fsync_part_directory=false`, `min_*_to_fsync_after_merge=0`). New parts are dirty page-cache pages the kernel flushes lazily; tmpfs only removes asynchronous background writeback, which is never on the critical path.

And the part that seals it: the data on disk is **compressed** (`CODEC(ZSTD(1))`, ~5× ratio). tmpfs stores the same compressed bytes, so every read still does `pread → ZSTD decompress → aggregate`. The dominant cost of these queries is CPU decompression, and **no storage medium can touch that** — the only layer that avoids re-decompressing is the uncompressed cache, which lives in RAM regardless of where the part sits.

So the source predicts: when the working set fits in RAM (20 GB into 750 GB — it does), a RAM disk changes nothing on reads. Time to try to prove the source wrong.

## The metric that settles it

There's exactly one number that isolates "did we touch the physical device": `ProfileEvents['OSReadBytes']` — bytes read from the block device, straight from `/proc`. It's the same metric that [caught an earlier read regression that turned out to be a write problem](/posts/clickhouse-read-regression-write-problem/): if it's zero, the query never hit the disk, no matter how it feels.

Running a warm production query against the NVMe table:

```text
read_bytes (logical):  22.20 MiB
OSReadBytes (device):  0.00 B      ← zero physical I/O
MarkCacheHits / Misses: 210 / 0
query_duration_ms:     30
```

The NVMe table is already served **100% from the page cache**. There is no disk I/O for a RAM disk to remove. That single line is the whole conclusion — everything after it is me trying, and failing, to find a scenario where it isn't true.

## The A/B that faked a result — twice

I set up the honest comparison: same node, two tables, identical schema, both ingesting live — one on NVMe (`ssd_in_order`), one on the tmpfs `ram_only` policy. Then I replayed the 20 production query patterns against each, warm, `use_uncompressed_cache=0` to isolate the read layer.

The raw medians *looked* like a RAM-disk win on the heavy queries:

| pattern | NVMe | RAM disk | rows read (NVMe / RAM) |
|---|---|---|---|
| c3 (costly) | 501 ms | 316 ms | 20.6 M / 19.2 M |
| c8 (costly) | 489 ms | 320 ms | 20.6 M / 19.2 M |
| f1 (frequent) | 69 ms | 38 ms | 0.21 M / 0.11 M |

But `OSReadBytes` was **0 on both**, every query. If neither touches the disk, the medium can't be the cause of a 1.5× gap. Something else was different. Two something-elses, in fact.

**Confound #1 — merge state.** The two tables were fed by separate ingestion, and their part counts had drifted apart. For the same time window:

| window `08:00–08:16` | NVMe | RAM disk |
|---|---|---|
| **parts in window** | **385** | **131** |
| rows in those parts | 43.5 M | 42.7 M |

Same rows, but the NVMe table had **3× more parts**. The read pool builds one reader per part (`src/Storages/MergeTree/MergeTreeReadPoolBase.cpp:135`, `per_part_infos.reserve(parts_ranges.size())`), so 385 parts means 3× the per-part setup, file opens, and mark lookups, plus more boundary-granule read amplification — which is exactly why NVMe's `read_rows` was ~2× for the "same" query. That's part-count overhead, storage-independent, masquerading as a storage result.

**Confound #2 — the data wasn't even identical.** When I finally checked instead of assuming:

| | NVMe | RAM disk |
|---|---|---|
| total rows | 92.4 M | 124.0 M |
| TTL | rolling ~1 h | none (accumulating) |
| rows in `08:00–08:15` | 32.12 M | 31.25 M |

Different TTLs, different totals, ~2.7% different even inside the shared window. The two ingestion streams simply weren't producing the same rows. The only signal immune to all of this drift was, again, `OSReadBytes = 0`.

Normalized per row and controlled for parts, the latency difference evaporates — consistent with the source: reads come from RAM either way.

## The one thing the RAM disk *did* change: file descriptors

Here's the twist that made the experiment worth it anyway. Partway through, a wide query started failing:

```text
Code: 76. DB::Exception: Cannot open file
/mnt/chram/store/.../switch_compensationTimeMs.cmrk2:
errno: 24, strerror: Too many open files
```

`errno 24` is `EMFILE` — the process hit its open-file limit. The node was holding **~490,000 file descriptors open**, right at the ceiling.

Why so many? This table is wide — 208 columns — and each column in each part is stored as two files (`.bin` + `.cmrk2`), before the JSON column's dynamic subcolumns add more. At ~820 active parts across the two tables, that's on the order of **370k files on disk**, and ClickHouse keeps a large fraction of their descriptors open: an `OpenedFileCache` for reuse on the read path (`src/Disks/IO/createReadBufferFromFileBase.cpp:97`, `ReadBufferFromFilePReadWithDescriptorsCache`), plus every running merge opening all of its input parts' columns at once — there were 30 merges in flight.

The critical nuance: **a file costs one descriptor whether it lives on tmpfs or NVMe.** The RAM disk didn't use more FDs per file. What blew the budget was running a *second full copy* of a wide, high-part-count, JSON table in the same process — the FD footprint is per-table (parts × columns × merges × queries), so a second equally-hungry table roughly *sums* onto the first. When the process crept past its `RLIMIT_NOFILE`, the next `open()` returned `EMFILE`:

```cpp
// src/IO/ReadBufferFromFile.cpp:49
errno == ENOENT ? FILE_DOESNT_EXIST : CANNOT_OPEN_FILE, file_name, "Cannot open file {}"
```

So the RAM disk delivered no read benefit and, as deployed *alongside* NVMe, doubled a scarce resource until queries started failing. (Dropping `max_threads` from 8 to 4 halved the per-query FD burst and let the run finish — a workaround, not a fix.)

## When a RAM disk *would* help

To be fair to the idea, tmpfs isn't useless — it's just mismatched to this workload. It pays off when:

- the **working set is larger than RAM** and a hot subset keeps getting evicted from the page cache (cache thrashing);
- the backing disk is **genuinely slow** (SATA/HDD) *and* cold reads dominate — not the case here, where it's NVMe and reads are warm (see [the SATA-tier detective story](/posts/clickhouse-iowait-sata-nvme-disk-tier/) for what a real disk-tier problem looks like);
- the data is **truly ephemeral** scratch space where losing everything on restart is fine.

None of those describe a 20 GB hot table on a 750 GB-RAM NVMe box. There, the page cache already *is* the RAM disk — an adaptive, reclaimable, durable one — and tmpfs only takes away those three properties (volatility, a RAM-capped ceiling, non-reclaimable memory) in exchange for nothing.

## Lessons

1. **Read the read path before you benchmark the read path.** Ten minutes in `ReadBufferFromFileDescriptor` and `DiskLocal` predicted the entire result. The benchmark only confirmed it.
2. **`OSReadBytes` is the ground truth for "did we touch the disk."** Latency lies; the `/proc` counter doesn't. Zero physical reads means no storage change can help.
3. **A/B tests on live tables drift.** Different TTLs, different merge states, different ingestion — my two "identical" tables were neither identical in data nor in part count. If the isolating metric hadn't been storage-independent, I'd have shipped a wrong conclusion.
4. **The most interesting finding is often a side effect.** The RAM disk answered its own question with a flat "no," but the file-descriptor math it forced me to do is the part I'll actually reuse — wide JSON tables plus high part counts sit close to the FD ceiling, and any scheme that keeps a second copy needs a higher `nofile` limit, fewer parts, or both.

The verdict was the same one the [mmap experiment](/posts/clickhouse-mmap-read-setting/) reached from a different angle: the data is already in RAM. The fastest storage tier for a warm ClickHouse workload is the one the kernel gives you for free.
