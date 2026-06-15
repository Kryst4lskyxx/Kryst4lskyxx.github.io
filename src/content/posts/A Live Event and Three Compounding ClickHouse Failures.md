---
title: 'A Live Event and Three Compounding ClickHouse Failures: Hedged-Request Crashes, FD Exhaustion, and Kafka Partition Skew'
published: 2026-06-15
description: 'A weekend of live-event traffic pushed a real-time ClickHouse cluster into a pod crash loop. Three failure modes — a hedged-request connection storm, a file-descriptor exhaustion that segfaulted the server, and Kafka consumer skew that starved half the cluster — turned out to be one chain. A source-level postmortem.'
image: ''
tags: [ClickHouse, Big Data Engineering, Kafka, Performance Tuning, Concurrency, Kubernetes, Debugging]
category: 'Coding'
draft: false
lang: 'en'
slug: live-event-three-compounding-clickhouse-failures
---

## The setup

A real-time analytics cluster serves per-minute alerting queries during live sports events. The shape, anonymized:

- **12 ClickHouse nodes** on Kubernetes, arranged as 6 shards × 2 replicas, plus 3 keeper nodes.
- **Ingestion:** each node runs Kafka-engine tables that consume from a set of upstream topics; a materialized view (MV) parses each raw message and writes it into wide "realtime" tables (hundreds of typed columns plus an open-ended JSON `tags` column).
- **Queries:** sub-second distributed `GROUP BY` rollups, fired once per minute per customer datasource, with a short rolling retention (data older than ~1 hour is dropped by partition).
- **Failover:** a second query backend exists for the same data; a small watermark service can cut individual datasources from ClickHouse to the fallback and back by flipping a per-datasource key.

Over one live-event weekend this cluster went from "fine" to a **pod crash loop** (containers exiting with code 139 and getting restarted by Kubernetes). What looked like three separate incidents — a connection storm, a server segfault, and Kafka consumer lag — turned out to be a single chain. This is the source-level walk-through.

## Day 1: the connection storm

The first symptom was vague: during a live game, the cluster "looked uncomfortable." TCP connection counts on the ClickHouse nodes spiked, query latency climbed, and the spike took an hour to drain. The pattern was reproducible: *whenever queries ran a little long, ClickHouse spawned more connections and then sat on them for an hour before garbage-collecting.*

Two settings explained the behaviour.

**Idle connections linger for an hour by default.** The server-side knob `idle_connection_timeout` defaults to **3600 seconds** (`src/Core/Settings.cpp`), and the TCP handler only closes a connection after it has been idle that long (`src/Server/TCPHandler.cpp`):

```cpp
if (elapsed_seconds > idle_connection_timeout)
{
    LOG_TRACE(log, "Closing idle connection");
    return;
}
```

So a burst of connections opened during a slow second persisted for a full hour. The first mitigation was to drop it sharply:

```xml
<idle_connection_timeout>120</idle_connection_timeout>  <!-- was 3600 -->
```

**Hedged requests were the connection multiplier.** ClickHouse's `use_hedged_requests` is **on by default**. For a distributed query it opens a connection to a *second* replica of a shard if the first hasn't established within `hedged_connection_timeout_ms` (default **50 ms**) or returned data within `receive_data_timeout_ms` (default **2000 ms**), then keeps whichever replica answers first (`src/Core/Settings.cpp`):

```cpp
DECLARE(Milliseconds, hedged_connection_timeout_ms, 50, ...)
DECLARE(Milliseconds, receive_data_timeout_ms, 2000, ...)
DECLARE(Bool, use_hedged_requests, true, ...)   // disabled by default only on Cloud
```

Hedging is a great tail-latency tool for long queries. But for *sub-second* real-time queries it is counterproductive: when a node is briefly slow (say it momentarily maxed out concurrent requests), it fails to respond inside 50 ms, so every coordinator immediately opens duplicate connections to replicas — a self-amplifying storm exactly when the cluster is already under load. The decision: **disable `use_hedged_requests` for the real-time path**. (Keep it only for the long-running hourly-rollup path, where the 50 ms timeout is noise.)

That stabilized Day 1. It also, unknowingly, removed the fuse for the Day 2 crash.

## Day 2: the crash loop

The next day, pods on the cluster began exiting with **code 139 (SIGSEGV)** and restarting every few hours. Restarts reset the symptom briefly, then it climbed back. The stack pointed at a destructor:

```
DB::HedgedConnectionsFactory::~HedgedConnectionsFactory()
  → std::terminate() → signal 11 → exit 139
```

Two clues stood out. First, the error that preceded the crash was `errno 24`, **"Too many open files"** (`CANNOT_OPEN_FILE`), on both inserts and selects. Second, the crash was in a *destructor* tied to hedged requests — the exact feature we'd just been discussing.

### Step 1: file descriptors, not connections

Checking the crashing pod:

```
FD soft limit:        1,048,576
open files (process): 1,032,449   → 98.5% of the limit
```

The pod was 25 minutes past a restart and already at 98.5%. The breakdown was decisive — these were **data-part files, not sockets**:

| FD type           | count    |
|-------------------|----------|
| part `.bin`       | 854,289  |
| part `.cmrk2`     | 107,902  |
| sockets           | 1,564    |

And the FD count tracked **shard age** exactly: the oldest shard (26 days of accumulated data) sat at ~1.03M open FDs and crashed; the youngest shard (5 days) sat at ~400K with headroom. Not a permanent difference — a **time bomb**: as the young shards age, they hit it too.

### Step 2: why a "too many open files" error crashes the whole server

This is the genuinely interesting bug. An `errno 24` *should* fail one query and move on. Instead it killed the process — because of a **destructor that throws during stack unwinding**.

When a hedged distributed query is torn down, `~HedgedConnectionsFactory()` runs (`src/Client/HedgedConnectionsFactory.cpp`):

```cpp
HedgedConnectionsFactory::~HedgedConnectionsFactory()
{
    stopChoosingReplicas();           // not noexcept, no try/catch
    pool->updateSharedError(shuffled_pools);
}
```

`stopChoosingReplicas()` removes file descriptors from an epoll set and resets timers:

```cpp
void HedgedConnectionsFactory::stopChoosingReplicas()
{
    for (auto & [fd, index] : fd_to_replica_index)
    {
        epoll.remove(fd);                                   // can throw
        replicas[index].connection_establisher->cancel();
    }
    for (auto & [timeout_fd, index] : timeout_fd_to_replica_index)
    {
        replicas[index].change_replica_timeout.reset();     // can throw
        epoll.remove(timeout_fd);                           // can throw
    }
    ...
}
```

`Epoll::remove()` throws when the underlying syscall fails (`src/Common/Epoll.cpp`):

```cpp
void Epoll::remove(int fd)
{
    --events_count;
    if (epoll_ctl(epoll_fd, EPOLL_CTL_DEL, fd, nullptr) == -1)
        throw ErrnoException(ErrorCodes::EPOLL_ERROR, "Cannot remove descriptor from epoll");
}
```

The chain:

1. The process is out of file descriptors. A query already throws `CANNOT_OPEN_FILE` and begins unwinding.
2. During unwinding, the in-flight `HedgedConnectionsFactory` is destroyed.
3. Its destructor calls `epoll.remove()` / `TimerDescriptor::reset()` — which **also** fail (you can't touch epoll/timerfd when you're out of FDs) and **throw a second exception while one is already propagating**.
4. A C++ destructor that throws during stack unwinding is `std::terminate()` → SIGSEGV → the whole server dies → Kubernetes restarts the pod.

This is why disabling `use_hedged_requests` is doubly valuable here: it doesn't just calm the connection storm, it **removes this object from the query path entirely**. Without it, FD exhaustion degrades to a clean per-query error instead of a process-wide segfault — "one query fails" instead of "pod restarts."

### Step 3: why the FDs exploded in the first place

ClickHouse's Wide-format parts store **one file per column substream** — a `.bin` (data) and a `.cmrk2` (marks) per stream, opened through a per-column reader (`src/Storages/MergeTree/MergeTreeReaderWide.cpp`). So files-per-part ≈ columns × 2.

The realtime tables are very wide on their own, but the real multiplier is the **JSON `tags` column**. When you write into a `JSON`/`Object` column, every distinct JSON path becomes its own dynamic subcolumn — its own stream, its own file pair — via `SerializationObject::enumerateStreams` (`src/DataTypes/Serializations/SerializationObject.cpp`), up to `DEFAULT_MAX_SEPARATELY_STORED_PATHS = 1024` (`src/DataTypes/DataTypeObject.h`), with overflow spilling into a single shared-data stream. Hence file names like `tags.<escaped.path>.cmrk2`.

Multiply it out:

```
files_per_part  ≈  (typed columns + distinct JSON tag paths) × 2 (.bin + .cmrk2)
total_open_fds  ≈  files_per_part × parts_per_table × number_of_tables
```

When ingestion volume roughly doubled during the event, parts-per-table rose (more inserts, more pressure on background merges) — and so did open files, linearly. The wide schema set the per-part cost; the part explosion did the rest.

## The real trigger: Kafka partition skew, amplified by an expensive MV

The FD exhaustion and the crash are the *mechanism*, but they don't explain why **specific** pods fell over while others were idle. That came from how Kafka consumers had been scaled.

### What changed

The upstream topic has **60 partitions**. The original layout was **5 consumers per node × 12 nodes = 60 consumers** — a clean 1:1, where every consumer owns exactly one partition and **every node owns exactly 5**. To add headroom, operators added 3 consumers per node → **8 × 12 = 96 consumers for 60 partitions**. The balance collapsed: some nodes ended up doing all the ingestion, others none. The overloaded nodes crashed.

### Why ClickHouse can't keep it balanced

ClickHouse does **not** assign partitions itself. Each `kafka_num_consumers` instance becomes a real librdkafka consumer (`src/Storages/Kafka/StorageKafka.cpp`), they all share one `group.id` (from `kafka_group_name`), and ClickHouse just calls `subscribe()` and accepts whatever the Kafka group coordinator returns. It even logs the over-provisioned case (`src/Storages/Kafka/KafkaConsumer.cpp`):

```cpp
if (topic_partitions.empty())
    LOG_INFO(log, "Got empty assignment: Not enough partitions in the topic for all consumers?");
```

Assignment is done by Kafka's **partition assignor**, and the assignor's universe is a *flat list of anonymous consumers* — it has no concept of a host or node. ClickHouse doesn't set `partition.assignment.strategy`, so librdkafka's default applies (`range,roundrobin`), and **range** is used. Its logic is brutally simple, per topic (`contrib/librdkafka/src/rdkafka_range_assignor.c`):

```c
numPartitionsPerConsumer    = partition_cnt / member_cnt;   // 60 / 96 = 0
consumersWithExtraPartition = partition_cnt % member_cnt;   // 60 % 96 = 60
for (i = 0; i < member_cnt; i++) {
    int length = numPartitionsPerConsumer + (i + 1 > consumersWithExtraPartition ? 0 : 1);
    if (length == 0) continue;                              // members 61..96 get NOTHING
    ...assign one partition...
}
```

With 96 consumers and 60 partitions, the **first 60** members (in sorted order) get exactly one partition; the **last 36 get zero** (an empty assignment).

Now the detail that turns "uneven" into "whole nodes starve." The assignor sorts members **lexicographically by `member_id`** (`contrib/librdkafka/src/rdkafka_assignor.c`), and `member_id` is prefixed by the client's `client.id`. ClickHouse's default `client.id` embeds the **hostname** (`src/Storages/Kafka/StorageKafkaUtils.cpp`):

```cpp
// "ClickHouse-<hostname>-<db>-<table>"
return fmt::format("{}-{}-{}-{}", VERSION_NAME, getFQDNOrHostName(), db, table);
```

So all 8 consumers of a node sort into one **contiguous block**, and the "first 60 / last 36" cut falls on node boundaries:

```
sorted members ≈ grouped by hostname:
  node1[1..8] node2[9..16] ... node8[57..64] ... node12[89..96]

first 60 members each get 1 partition, the rest get 0:
  nodes 1–7   → all 8 consumers active → 8 partitions each   (56)
  node 8      → 4 of 8 active          → 4 partitions
  nodes 9–12  → 0 active               → 0 partitions
                                          ──────────────
                                          60 partitions
```

That is the observed asymmetry: a handful of nodes carrying ~8 partitions while a third of the cluster does nothing. (Round-robin, the fallback strategy, splits the same 60/36 over the same sorted list; cooperative-sticky can't conjure partitions for surplus consumers either. No standard assignor saves you when consumers > partitions.)

**Why 60/60 was perfect:** `60 / 60 = 1`, remainder `0` → every consumer gets exactly one partition, so every 5-consumer node gets exactly 5. Even per-node distribution is automatic **only when the partition count is an exact multiple of the total consumer count** — every member gets the identical share. Break the divisibility (96 ∤ 60) and the remainder concentrates onto whichever hosts sort last.

### Why skew became a crash: the MV runs on the consuming node

Skew alone wouldn't crash a pod — the per-message work has to be heavy, and it has to run *where the partition landed*. Both are true. The MV transform executes inline on the consuming node (`src/Storages/Kafka/StorageKafka.cpp`, `streamToViews` → `CompletedPipelineExecutor`), and the transform is expensive per row. In schematic form:

```sql
CREATE MATERIALIZED VIEW realtime_mv TO realtime_local
AS
WITH
  (JSONExtract(msg, 'Tuple(/* ~200 typed fields */)') AS parsed,   -- parse #1
   JSONExtractKeysAndValues(msg, 'String')           AS kvs)       -- parse #2
SELECT
  tupleElement(parsed, 'field_1')   AS field_1,
  /* ... ~200 tupleElement projections ... */
  mapFilter((k, v) -> startsWith(k, 'tags.') AND NOT has(promoted_keys, k),
            mapFromArrays(kvs.1, kvs.2)) AS tags          -- written into a JSON column
FROM realtime_kafka
SETTINGS type_json_skip_duplicated_paths = 1, json_type_escape_dots_in_keys = 1;
```

Per row, this:

1. parses the raw message **twice** — once into a ~200-field typed tuple, once for all key/values. Each `JSONExtract*` call parses the *entire* document independently (`src/Functions/FunctionsJSON.cpp`); they don't share work.
2. runs ~200 `tupleElement` projections,
3. builds and filters a map of leftover tags, and
4. writes a `JSON` column whose every distinct path becomes a subcolumn file at write time.

So a node that drew extra partitions gets hit on **two axes at once**: CPU (double JSON parse + ~200 extractions per row, scaled by row count) *and* file descriptors (per-path subcolumn files × the extra parts it now produces). The hottest nodes saturate CPU and march toward the FD limit faster — and the FD limit is where the hedged-destructor segfault waits. The three "separate" incidents are one chain:

> **partition skew → a few hot nodes → more rows + more parts on those nodes → CPU saturation + FD exhaustion → `errno 24` inside a hedged query → throwing destructor → SIGSEGV → pod restart.**

## The fixes

**Stop the crash loop (immediate):**

- **Disable hedged requests on the real-time path** (`use_hedged_requests = 0`). Removes the connection storm *and* the segfault path; FD exhaustion degrades to a clean per-query error.
- **Raise the FD limit** (e.g. to 2M) on the StatefulSet. FDs are cheap on a large-memory box; this buys time while the real fixes land.

**Fix the ingestion skew (root cause):**

- **Keep total consumers an exact divisor of the partition count, equal per node.** For 60 partitions on 12 nodes, **5/node (= 60)** is optimal. Never set total consumers *above* the partition count — the surplus consumers are idle *and* every membership change triggers a group-wide stop-the-world rebalance (`set_revocation_callback` revokes everyone's partitions), which itself pauses ingestion.
- **Align shard and partition counts to shared factors** (e.g. multiples of 30/60) so rebalances stay even cluster-wide.

**Reduce the per-part and per-row cost:**

- **Cut part count:** make sure background merges keep up under peak ingestion (fewer parts = fewer files), and keep retention/TTL tight so rolling tables don't hold more than the queries need.
- **Shrink the MV transform:** parse the message once instead of twice, and reconsider materializing an open-ended `JSON` column at full width — it is the dominant contributor to both per-row CPU and per-part file count.

**Add the alerts that would have caught this early:** open file descriptors per pod, TCP connections, partitions-assigned-per-node and messages-consumed-per-node (the engine already exposes `KafkaAssignedPartitions` / `KafkaConsumersWithAssignment`), and concurrent-request counts.

**Longer term:** the throwing-destructor segfault is a server bug worth tracking upstream — a destructor that calls `epoll_ctl`/`timerfd_settime` (both can fail) is unsafe by construction; newer ClickHouse releases may handle FD exhaustion gracefully.

## Takeaways

- **A "too many open files" error should never crash the server.** It did here because `~HedgedConnectionsFactory()` throws during stack unwinding (`epoll.remove()` fails when you're out of FDs) → `std::terminate()`. Turning off the feature on the hot path converted a process-wide segfault into a survivable per-query error.
- **Wide schemas and open-ended JSON columns are FD amplifiers.** Files-per-part ≈ (columns + distinct JSON paths) × 2; multiply by parts and tables and a doubling of ingestion volume walks you straight into a 1M-FD wall. The oldest shard hits it first — it's a time bomb, not a stable difference.
- **Kafka balances consumers, not nodes.** The partition assignor sees a flat list of anonymous members; it has no concept of a host. Even per-node distribution is guaranteed *only* when partition count is an exact multiple of total consumer count. Over-provisioning consumers past the partition count doesn't add headroom — it starves whichever hosts sort last and adds rebalance churn.
- **The default hostname in `client.id` makes skew land on node boundaries.** Because the range assignor sorts by `member_id` (prefixed by `client.id`), a node's consumers cluster together, so the "have/have-not" cut falls on whole nodes — turning a statistical imbalance into "these pods do everything, those do nothing."
- **Where the materialized view runs matters.** The MV transform executes on the *consuming* node, so partition skew becomes CPU and FD skew on that node. A cheap MV tolerates imbalance; an expensive one (double JSON parse + a wide JSON write per row) converts it into crashes.
- **Mitigation order beats root-cause order during an incident.** Disabling hedged requests, dropping the idle timeout, tightening TTL, and cutting datasources over to the fallback backend bought the hours needed to fix the ingestion layout calmly — even though the *cause* was the consumer scaling, not any of those knobs.
