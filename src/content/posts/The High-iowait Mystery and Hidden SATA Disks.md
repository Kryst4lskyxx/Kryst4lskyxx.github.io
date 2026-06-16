---
title: "10x iowait on 'Identical' Nodes: A ClickHouse Disk-Tier Detective Story"
published: 2026-06-16
description: "One rack series in a real-time ClickHouse cluster ran 10x the OS iowait of everything else. The disk-read latency metric pointed nowhere. The nodes doing the *most* I/O were the calmest. The answer was hiding in a Prometheus label nobody reads — and the twist was that 'these nodes use SATA' turned out to be true of almost the whole cluster. A metrics-level walk-through of debugging storage you can't SSH into."
image: ''
tags: [ClickHouse, Observability, Prometheus, Performance Tuning, Storage, Debugging]
category: 'Coding'
draft: false
lang: 'en'
slug: clickhouse-iowait-sata-nvme-disk-tier
---

## The report

> The `ch40x-*.dcB` machines on the real-time cluster consistently have high OS iowait. All the high-iowait machines are from the same `ch40x` series.

That was the whole bug report, plus a Grafana panel and one PromQL expression to "also check the disk IO":

```promql
max by (instance) (
  (rate(node_disk_read_time_seconds_total{cluster="realtime"}[2m]) * 1000)
  / rate(node_disk_reads_completed_total{cluster="realtime"}[2m])
)
```

Two things make this kind of report interesting. First, the cluster is supposed to be homogeneous — same role, same rack generation, same provisioning. "One series is different" should not be a thing. Second, I can't SSH into these boxes to run `iostat` and `smartctl`; all I have is whatever the node-exporter and ClickHouse already push to Prometheus.

So the first question wasn't about disks at all. It was: **can I drive Prometheus like an API, so the debugging loop is query → parse → query, instead of clicking around Grafana?**

## Prometheus is just an HTTP API

The Grafana URL everyone shares looks scary:

```
https://prometheus.internal/graph?g0.expr=rate(...)&g0.range_input=1w&...
```

But that `g0.expr` is the entire payload. The same query, over the same data, is one HTTP call:

```
GET https://prometheus.internal/api/v1/query?query=<promql>          # instant
GET https://prometheus.internal/api/v1/query_range?query=...&start=...&end=...&step=...
```

It returns JSON. Two operational gotchas bit me on the first call:

- The endpoint sits behind an internal CA, so `curl` needs `-k` (or the CA bundle).
- The first call returned `HTTP 000` — the agent sandbox blocks outbound network. Disabling the sandbox for that one command fixed it.

Once it returned `{"status":"success",...}`, I wrapped it in a throwaway helper so every query came back as a sorted table instead of a wall of JSON:

```bash
#!/usr/bin/env bash
# promq.sh 'PROMQL'                  -> instant, table sorted desc
# promq.sh 'PROMQL' range 24h 600s   -> range, per-series avg/max
PROM='https://prometheus.internal'
q="$1"; mode="${2:-instant}"
if [ "$mode" = "range" ]; then
  span="${3:-1h}"; step="${4:-60s}"
  num=${span%[smhd]}; unit=${span: -1}
  case $unit in s) m=1;; m) m=60;; h) m=3600;; d) m=86400;; esac
  now=$(date +%s); start=$((now - num*m))
  curl -sk -G "$PROM/api/v1/query_range" \
    --data-urlencode "query=$q" --data-urlencode "start=$start" \
    --data-urlencode "end=$now" --data-urlencode "step=$step" \
  | jq -r '.data.result[] | . as $s
      | ([.values[][1]|tonumber] | (add/length) as $a | max as $m | "\($a)\t\($m)")
      as $st | "\($st)\t\($s.metric|to_entries|map("\(.key)=\(.value)")|join(","))"' \
  | sort -rn -k2
else
  curl -sk -G "$PROM/api/v1/query" --data-urlencode "query=$q" \
  | jq -r '.data.result[] | "\(.value[1])\t\(.metric|to_entries|map("\(.key)=\(.value)")|join(","))"' \
  | sort -rn
fi
```

That `node_disk_info` line near the end of this post is the punchline, and it only became reachable because the loop was now fast enough to ask a dozen questions in a row.

## Step 1: confirm the claim, and find the real boundary

OS iowait is `node_cpu_seconds_total{mode="iowait"}` — the fraction of CPU time stalled waiting on I/O. Averaged across cores, per host:

```promql
avg by (instance) (rate(node_cpu_seconds_total{cluster="realtime",mode="iowait"}[5m]))
```

| instance | iowait fraction |
|---|---|
| ch403-2.dcB | 0.0169 |
| ch402-2.dcB | 0.0115 |
| ch402-1.dcB | 0.0110 |
| ch401-2.dcB | 0.0106 |
| ch403-1.dcB | 0.0092 |
| ch401-1.dcB | 0.0086 |
| ch406-4.dcB | 0.0014 |
| ch405-4.dcB | 0.0013 |
| …rest of cluster | ~0.0003–0.001 |

The report was right, but slightly off on the boundary. It's not the whole `ch40x` series — it's specifically the **`-1` and `-2`** nodes (`ch401`/`ch402`/`ch403`). The **`-4`** nodes in the very same series (`ch404-4`, `ch405-4`, `ch406-4`) sit down with the quiet majority at ~0.001. So within one "identical" rack generation, the `-1`/`-2` replicas run **~10x** the iowait of the `-4` replicas.

A 24h range query confirmed it wasn't a blip — the `-1`/`-2` nodes held ~1.2% average and ~3.4% peak iowait all day, the `-4` nodes ~0.12%. Steady-state, structural.

## Step 2: the disk-read metric is a dead end

The bug report's suggested query was disk **read** latency. Run it and the hot nodes don't even appear:

| instance | ms per read |
|---|---|
| ch502-27.dcC | 5.0 |
| ch505-27.dcC | 4.0 |
| ch201-22c.dcA | 1.0 |
| …a few stragglers… | <0.3 |
| **ch40x-1/-2.dcB** | **nan** |

`nan`, because the denominator — `rate(node_disk_reads_completed_total)` — is **zero**. The high-iowait nodes are doing essentially **no disk reads at all**.

That's not as strange as it sounds for a real-time ingest cluster: retention is ~1 hour, so the entire working set is recent and lives in the page cache. Reads are served from RAM. All the disk pressure is **writes** — inserts landing as new parts, and background merges rewriting them. The read-latency panel was always going to point nowhere. We were looking at the wrong half of the I/O.

## Step 3: the counterintuitive part

So I pulled the write side and the disk-busy side together. This is where it stopped making sense — in the productive way:

```promql
# write throughput MB/s
sum by (instance) (rate(node_disk_written_bytes_total{cluster="realtime"}[5m]))/1048576
# write IOPS
sum by (instance) (rate(node_disk_writes_completed_total{cluster="realtime"}[5m]))
# disk busy fraction (utilisation)
max by (instance) (rate(node_disk_io_time_seconds_total{cluster="realtime"}[5m]))
# avg queue depth (weighted io time)
max by (instance) (rate(node_disk_io_time_weighted_seconds_total{cluster="realtime"}[5m]))
# write latency ms/op
max by (instance) ((rate(node_disk_write_time_seconds_total[5m])*1000)
                   / rate(node_disk_writes_completed_total[5m]))
```

Lining up the two groups of `dcB` nodes:

| Metric | `-1`/`-2` (high iowait) | `-4` (low iowait) |
|---|---|---|
| Write throughput | 310–337 MB/s | **350–395 MB/s** |
| Write IOPS | <36k (off the top-12) | **73k–84k** |
| Disk utilisation | **~25%** | ~11% |
| Avg queue depth | **~3,500–4,640** | ~960–1,200 |
| Write latency | **359–491 ms** | 93–96 ms |

Read that table twice. The calm `-4` nodes push **more** bytes and **far more** IOPS. The struggling `-1`/`-2` nodes push **less** work, yet sit at **double the utilisation, ~4x the queue depth, and ~4x the per-write latency.**

When a device absorbs more work while staying *less* busy, the device itself is faster. The conclusion writes itself: this isn't a workload problem. The `-1`/`-2` nodes are sitting on **slower storage** than their `-4` siblings in the same rack series.

A couple of metric details corroborated it before I'd confirmed any hardware:

- **Average write size.** Bytes ÷ IOPS gives ~10 KB/op on `-1`/`-2` vs ~4.7 KB/op on `-4`. That's the Linux block layer **coalescing** adjacent writes while they queue, because the slow device can't drain them fast enough. The fast device drains immediately, so writes stay small and uncoalesced. High average I/O size here is a symptom of saturation, not of a bulkier workload.
- **Merge backlog.** ClickHouse's `BackgroundMergesAndMutationsPoolTask` showed more concurrent active merges on the `-1`/`-2` nodes (7/5/4 vs 1). Tempting to call that the cause — "they merge more, so they iowait more." But causality runs the other way: each merge's write phase takes ~4x longer on slow disks, so merges *stay active longer* and pile up at any given snapshot. The backlog is the slow disk's shadow, not its source.

## Step 4: the label nobody reads

Metrics had proven *behaviour*. To prove *hardware* without shell access, I needed the device model — and node-exporter actually ships it, in a metric most people never query: `node_disk_info`, whose labels include `model` and `revision`.

```promql
node_disk_info{cluster="realtime",instance=~"ch403-1.*|ch405-4.*"}
```

```
ch403-1.dcB  dev=sda  model=HFS960G32FEH-BA10A   # SK Hynix 960GB SATA SSD
ch403-1.dcB  dev=sdb  model=HFS960G32FEH-BA10A
ch405-4.dcB  dev=sda  model=Dell_Ent_NVMe_CM     # Dell enterprise NVMe
ch405-4.dcB  dev=sdb  model=Dell_Ent_NVMe_CM
```

There it is. The high-iowait nodes are on **SK Hynix SATA SSDs**; the calm ones on **Dell enterprise NVMe**. Every number above falls out of that one line. SATA SSDs top out around ~550 MB/s on a shallow, ~32-deep AHCI queue; under sustained merge writes they saturate, queue up, coalesce, and stall the CPU on iowait. NVMe drains a far deeper queue at multiple GB/s, so the same — or heavier — workload barely registers. Two storage generations, racked into one "identical" cluster, distinguished by a label and nothing else.

I could have stopped here. That would have been the wrong place to stop.

## The twist: "these nodes use SATA" is almost cluster-wide

The natural follow-up: *is it only this series? What about the `20x`, `30x`, `50x`, `60x` machines?* So I censused `model` across every node:

```
20 × Dell NVMe ISE PS1030 1.6TB        (dcC, 50x/60x)
18 × HFS960G32FEH-BA10A (SK Hynix SATA)
14 × INTEL_SSDSC2KG960G8 (Intel SATA)
14 × INTEL SSDPE2KE032T8 (Intel P4610 NVMe 3.2TB)
12 × Dell Ent NVMe CM6 3.2TB
 8 × MZ7L3960HBLTAD3 (Samsung PM893 SATA)
 5 × Dell Ent NVMe PM1735a 6.4TB
 …
```

If you stop at "node has a SATA SSD → suspect," you'd now indict **most of the cluster** — there are SATA drives nearly everywhere. That would be wrong, and it's the trap.

Almost every box has **both** a SATA SSD *and* NVMe drives. The question was never "which nodes contain SATA," it's **"which device actually carries the ClickHouse write load."** So I asked per-device, one representative node per series:

```promql
rate(node_disk_io_time_seconds_total{cluster="realtime",
  instance=~"ch201-22b.*|ch201-24a.*|ch305-32.*|ch401-1.*|ch403-4.*|ch501-27.*",
  device=~"sd[ab]|nvme[0-9]n1"}[5m])
```

| node (series) | busy device | type | result |
|---|---|---|---|
| ch201-22b (20x) | `nvme1n1` 1.1% (Intel P4610) — `sd*` 0.45% | NVMe data, SATA = boot | fine |
| ch201-24a (20x) | `nvme0n1` 6.4% (Dell CM6) — `sd*` 0.35% | NVMe data, SATA = boot | fine |
| ch305-32 (30x) | `sd*` 0.36% — **`nvme*` 0.0%** | SATA data, NVMe **idle** | fine *(but…)* |
| **ch401-1 (40x -1/-2)** | **`sd*` ~26%** (SK Hynix), no NVMe present | **SATA data** | **HOT** |
| ch403-4 (40x -4) | `sd*` 9.2% = Dell NVMe (enumerated as `sd`) | NVMe data | fine |
| ch501-27 (50x) | `nvme*` ~7.3% (Dell PS1030) | NVMe data | fine |

This reframed everything:

- On the **20x / 50x / 60x** nodes the SATA SSD is just the **boot disk**, idling near 0.4%. The data — and all the write pressure — lives on NVMe. SATA present, SATA irrelevant.
- The **40x `-4`** node's NVMe happens to *enumerate as `sda`* behind its controller. Device name lied; the `model` label didn't.
- The **40x `-1`/`-2`** nodes are the **only** ones in the cluster running ClickHouse **data on SATA under real load** — and they have **no NVMe at all** to migrate to. That, precisely, is the bug.

And one freebie the census surfaced: the **30x** nodes *also* keep their data on SATA, while their **6.4TB Dell PM1735a NVMe drives sit completely idle (0 MB/s).** They escape the iowait only because that shard currently takes almost no writes (0.03–0.67 MB/s). They're a latent repeat of the same incident *and* a pile of unused fast hardware — a near-free win waiting for someone to move the data onto the NVMe already in the chassis.

## What actually fixed it / what to do

- **Real fix:** the six `ch40[123]-{1,2}.dcB` nodes need NVMe to match their `-4` siblings, or their data tier moved off SATA. They are the cluster's weakest I/O links and will saturate first under any growth — and slow merges there feed straight into part-count and file-descriptor problems elsewhere.
- **Free fix:** point the `30x` nodes' ClickHouse storage at the idle 6.4TB NVMe they already have.
- **Stopgap:** until hardware moves, cap concurrent merges / write amplification on the SATA nodes (`background_pool_size`, merge sizing) so the slow disks aren't running as many parallel rewrites.
- **Placement rule:** don't schedule extra shards/replicas or I/O-heavy work onto data-on-SATA nodes.

## Lessons that generalize

1. **iowait is a relative signal.** 1.2% is "fine" in a vacuum; at 10x the rest of an identical cluster it's a flashing light. Compare across peers, not against an absolute threshold.
2. **Debug the half of I/O that's actually happening.** The report (and the obvious panel) pointed at read latency. Reads were zero. For short-retention real-time ingest, the disk only ever sees *writes* — page cache eats the reads.
3. **Utilisation + queue depth + latency beat throughput.** Throughput said the hot nodes were *less* busy. The device that does more work at lower utilisation is simply faster; that single inversion was the whole diagnosis.
4. **`node_disk_info` is the cheapest hardware audit you have.** No SSH, no `smartctl`. The `model`/`revision` labels distinguish SATA from NVMe across an entire fleet in one query. Almost nobody graphs it.
5. **Per-node hides; per-device reveals.** "This node has SATA" was true of most of the cluster and meaningless. "This node's *data device* is SATA *and* under load" was true of six nodes and was the bug.
6. **"Identical" clusters aren't.** Mixed procurement generations get racked under one role and one dashboard. The label is the only thing that remembers they're different.

The browser URL that started it all was a week-long iowait graph nobody could explain. The answer was one `node_disk_info` query — reachable only once Prometheus was being driven as an API instead of a dashboard.
