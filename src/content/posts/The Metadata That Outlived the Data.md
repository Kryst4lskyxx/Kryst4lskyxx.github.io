---
title: 'One-Hour Data, Eighty-Day Metadata: A ClickHouse Failure Chain'
published: 2026-10-26
description: 'Keeper kept 1.47 million partition entries after data expired. Session failures and thousands of JSON files per part turned recovery into repeated crashes.'
image: ''
tags: [ClickHouse, Keeper, Debugging, JSON, File Descriptors, Merges, Kubernetes]
category: 'Coding'
draft: false
lang: 'en'
slug: clickhouse-keeper-metadata-json-file-descriptors
---

The tables retained one hour of data. Their Keeper metadata included partition entries from about 80 days earlier.

That difference helped explain a production incident with 15 container restarts across four ClickHouse nodes. Keeper reached its memory limit and rejected work. Replica recovery then faced a second limit: thousands of open files for each JSON-heavy part.

I first traced how the processes stopped. The deeper explanation came from counting the objects that remained alive.

This investigation covered October 24, 2026. Follow-up measurements came from October 26. The cluster ran a ClickHouse 25.8.11.66 fork and Keeper 25.3.14.14. The examples use anonymous names. Source links point to the relevant upstream code for those versions.

## The timeline showed two kinds of failure

The cluster contained twelve ClickHouse nodes and three Keeper nodes. A shard stores one portion of a table. Each shard had two replicas, which stored copies of that portion.

Keeper provided coordination for the replicas. Clients repeatedly lost their Keeper sessions, which tracked their active connections. Several ClickHouse processes also aborted inside C++ destructors. These were different failures within the same incident.

A CPU quota limits processor time per scheduling period. CPU throttling pauses work that exceeds its CPU quota. The timeline includes both memory rejection and CPU throttling.

All times in the following timeline use UTC+8. The counts describe the measured scope, rather than application-level failures. I did not establish data loss or the number of failed user requests.

| Time | Observation |
| --- | --- |
| 01:00–01:10 | Clients logged 152,288 session-finalization events with `Hardware error: Out of Memory`. |
| 03:56–06:59 | Restart counters increased 15 times across four ClickHouse nodes. |
| 06:45 | One Keeper node reached 20.2% throttled scheduling periods. |
| 07:25 | One replica queue reached 1,152 operations. |
| 07:30 | One partition contained 1,023 parts. |
| 07:38–07:42 | Replacement Keeper containers started with larger resource allocations. |
| After 08:00 | Sampled Keeper connection failures stayed at zero. |

![Keeper memory, CPU throttling, open files, and replica queues before and after recovery.](/images/clickhouse-keeper-memory-json-files.svg)

Each line represents one node. The chart uses five-minute samples. Its memory panel shows the container working set, which differs from the process memory measurement used by Keeper.

## Keeper rejected requests at its soft memory limit

Keeper had a 2 GiB memory limit per container. Its default soft limit was 90% of that amount, or about 1.8 GiB. A soft limit rejects selected work before the process exhausts its container allocation.

The logs contained this exact reason for session finalization:

```text
Hardware error: Out of Memory
```

This message described a response from Keeper. It did not establish that the operating system killed a ClickHouse process for memory exhaustion.

The matched [Keeper dispatcher](https://github.com/ClickHouse/ClickHouse/blob/84d6b30ad528e77d787ab7a2437406c1e2a5887a/src/Coordination/KeeperDispatcher.cpp#L165-L177) rejects requests that increase memory when the soft limit is reached. It returns `ZOUTOFMEMORY`. The [Keeper startup code](https://github.com/ClickHouse/ClickHouse/blob/84d6b30ad528e77d787ab7a2437406c1e2a5887a/programs/keeper/Keeper.cpp#L337-L359) sets the default ratio to `0.9`.

The ClickHouse client classifies `ZOUTOFMEMORY` as a hardware error. Its [response handler](https://github.com/ClickHouse/ClickHouse/blob/v25.8.11.66-lts/src/Common/ZooKeeper/ZooKeeperImpl.cpp#L1113-L1115) immediately finalizes the session after such an error. It does not wait for the normal session timeout.

That behavior explains the rapid reconnects. Keeper rejected work, clients closed sessions, and replicas tried to activate again. The next memory-increasing request met the same limit.

The reconnects also consumed CPU. Keeper had a two-core CPU limit during the incident. Short bursts exhausted that quota even when the five-minute average stayed below two cores.

The 20.2% value counted scheduling periods that experienced throttling. It did not mean that Keeper lost exactly 20.2% of its CPU capacity. CPU throttling described part of the pressure, while `ZOUTOFMEMORY` identified a separate memory guard.

## Data expiration did not remove partition metadata

A partition groups rows by a key. A TTL rule sets when data expires. The sampled tables created partitions every 30 minutes and expired data one hour after each partition started.

With a generic column name, the policy looked like this:

```sql
PARTITION BY toStartOfInterval(event_time, INTERVAL 30 MINUTE)
TTL toStartOfInterval(event_time, INTERVAL 30 MINUTE) + INTERVAL 1 HOUR
```

Keeper also stored metadata for each partition. A znode is one entry in the Keeper tree. The relevant path had this form:

```text
/clickhouse/tables/<shard>/<database>/<table>/block_numbers/<partition_id>
```

ClickHouse used these entries to allocate ordered block numbers. The [allocation code](https://github.com/ClickHouse/ClickHouse/blob/v25.8.11.66-lts/src/Storages/StorageReplicatedMergeTree.cpp#L7157-L7174) explicitly created each partition entry with `CreateMode::Persistent`.

The word `Persistent` mattered. A session ending did not remove that entry. Data expiration also left the sampled partition entries in place.

I counted immediate children under known table paths. The count covered 420 table/shard paths without recursively reading the entire Keeper tree. It found 1,468,298 entries under `block_numbers`.

| Follow-up measurement | Result |
| --- | ---: |
| Total znodes on the sampled Keeper node | About 1.81 million |
| Partition entries across known table paths | 1,468,298 |
| Share of total znode count | About 81% |
| Partition entries for sampled table A, one shard | About 3,862 |
| Partition entries for sampled table B, one shard | About 3,590 |
| Creation time of the oldest inspected entries | August 6, 2026 |

The creation timestamps established their age. The oldest inspected entries also had zero children. The persistent parents remained even when they had no children.

At continuous activity, 30-minute partitioning introduces up to 48 partition entries per table/shard each day. That becomes 3,840 entries over 80 days. The sampled counts matched that scale.

A one-hour data TTL therefore did not bound this metadata history. The source also contained a separate [`forgetPartition` operation](https://github.com/ClickHouse/ClickHouse/blob/v25.8.11.66-lts/src/Storages/StorageReplicatedMergeTree.cpp#L7881-L7895) that removed a partition-number entry. Its existence did not make blanket deletion safe.

Partition numbers protect ordering and replication. A cleanup design must account for remaining parts, outstanding work, and possible writes to old partitions. I did not delete Keeper entries during this investigation.

## Node count explained growth, but not every memory byte

After recovery, the sampled Keeper process still used about 1.10 GiB of resident memory. Resident set size, or RSS, measures process memory in RAM. Keeper reported about 487 MB of approximate tree data and 79 MB in its latest-log cache.

Those values described different measurements. The approximate tree size did not include every process allocation. The znode count did not describe the exact memory share of each category.

The old container left about 1.8 GiB before the soft guard rejected work. The accumulated partition entries established a source of persistent growth. Caches, temporary allocations, and allocator retention also consumed headroom.

I did not recover the old process memory breakdown. The metadata accounted for about 81% of node count, not a measured 81% of RSS. The evidence did not assign every pre-restart byte to the tree or establish a memory leak.

## One JSON column opened thousands of files per part

A part stores a batch of rows. A merge combines parts into a new part. A file descriptor identifies an open operating-system resource.

ClickHouse needs descriptors for data files, sockets, pipes, and timers. The process limit was 1,048,576. Incident logs repeatedly reported this error:

```text
errno: 24, strerror: Too many open files
```

That error established process descriptor exhaustion. Counting files on disk alone does not establish the same condition. The relevant quantity is how many files remain open at the same time.

The affected tables stored dynamic event attributes in a `tags JSON` column. A JSON path identifies a field within a JSON value. In this storage format, dynamic paths produce separate data and metadata streams.

The source [enumerates dynamic paths separately](https://github.com/ClickHouse/ClickHouse/blob/v25.8.11.66-lts/src/DataTypes/Serializations/SerializationObject.cpp#L190-L201). A path can produce several files for values, type information, and marks. Marks locate positions inside compressed data.

A follow-up listing of descriptors on one affected node showed the scale:

| Descriptor category | Approximate count |
| --- | ---: |
| All sampled descriptors | 89,917 |
| Descriptors pointing into part storage | 80,962 |
| Descriptors for `tags.*` streams | 80,881 |
| Sockets | 1,224 |
| Pipes | 1,791 |

JSON streams accounted for about 90% of all sampled descriptors. One event table held about 61,700 descriptors across 21 part directories. Several ordinary input parts each held about 3,080 distinct open files.

Two temporary merge outputs held 8,221 and 5,407 distinct open files. Each sampled filename appeared once within those directories. The large count came from many physical streams, rather than thousands of opens of one filename.

The listing captured a running process, so it was not atomic. It also came after recovery. It established the current source of descriptor demand, while the incident logs established actual exhaustion during the failure.

## Concurrent merges multiplied the demand

A merge reads several input parts and writes an output part. Its readers and writers keep streams open during the operation. Queries and new inserts also need descriptors while that work runs.

The cost grows with simultaneous input parts, streams per part, and open output streams. JSON increased the streams-per-part term. Background recovery increased the amount of simultaneous work.

Historical metrics showed up to 64 executing merges on one node. Another sample showed 705 source parts participating in merges on one node. Those peaks were separate measurements, so I did not combine them into one synthetic instant.

At 03:35, one affected node reported both of these gauges:

```text
OpenFileForRead     810,894
OpenFileForWrite    242,978
Total            1,053,872
Process limit    1,048,576
```

A gauge reports a sampled current value. A file buffer manages file reads or writes. These ClickHouse gauges count file-buffer objects, so their sum is not an atomic operating-system descriptor census. Concurrent changes and object lifetimes affect the comparison.

The comparison still explained the scale. Reading alone consumed hundreds of thousands of file buffers. Writers added hundreds of thousands more, before sockets and other resources. The kernel errors established that the process actually reached its limit.

Keeper session failures interrupted normal replica activity. Replication and part backlogs grew during the same period. The largest observed replica queue reached 1,152 operations, and one partition reached 1,023 parts.

This sequence supports a feedback loop. Coordination failures delayed useful progress, while recovery increased demand for resources. The evidence did not attribute every open descriptor to a particular recovery task.

## Exceptions turned resource errors into process crashes

A destructor runs when C++ destroys an object. Destructors normally must not let exceptions escape. An escaping exception can call `std::terminate()` and stop the entire process.

Hedged requests race connections to multiple replicas. Retained crash stacks from three nodes included this path:

```text
HedgedConnectionsFactory::~HedgedConnectionsFactory()
std::terminate()
terminate_handler()
```

The [destructor](https://github.com/ClickHouse/ClickHouse/blob/v25.8.11.66-lts/src/Client/HedgedConnectionsFactory.cpp#L50-L63) called `stopChoosingReplicas()` without an exception boundary. That cleanup path performed timer and descriptor operations. That path can throw during resource exhaustion.

The stack established the destructor path. It did not identify the exact throwing instruction for every crash. I covered another instance of this failure family in [A Throw in a Destructor](/posts/clickhouse-fd-exhaustion-destructor-abort/).

The fourth node also hit a different destructor failure. During a replica fetch, an HTTP connection destructor tried to determine whether a response was complete. That operation read from the socket and timed out.

The [HTTP pool destructor](https://github.com/ClickHouse/ClickHouse/blob/v25.8.11.66-lts/src/Common/HTTPConnectionPool.cpp#L439-L450) called `HTTPChunkedInputStream::isComplete()`. That [method enabled a device read](https://github.com/ClickHouse/ClickHouse/blob/v25.8.11.66-lts/base/poco/Net/include/Poco/Net/HTTPChunkedStream.h#L85). The [completion routine](https://github.com/ClickHouse/ClickHouse/blob/v25.8.11.66-lts/base/poco/Net/src/HTTPChunkedStream.cpp#L143-L159) caught `MessageException`, but the observed socket timeout escaped.

The node later exited with code 137 during startup. Its previous log showed repeated DNS failures and a termination signal. That last record did not establish an operating-system OOM kill or identify who initiated termination.

The retained records therefore showed several failure paths. They did not establish one identical cause for all fifteen restarts. SIGABRT reports a process abort. SIGSEGV reports an invalid memory access. Counting SIGABRT and SIGSEGV records as separate restarts also overstated the failures, because one abort episode logged both signals.

## The resize cleared the storm, with two changes at once

The team increased each Keeper container from two to eight CPU cores and from 2 GiB to 4 GiB of memory. Replacement containers started over about four minutes. Keeper connection failures then disappeared from the sampled monitoring.

From 08:00 on October 24 through 10:00 on October 26, every inspected half-hour sample showed zero Keeper hardware exceptions. Each sample used a five-minute counter rate. This sampling did not cover every second between samples.

Open-file pressure and replica queues also fell. A read-only replica rejects new writes. Three table replicas remained read-only on one node during the follow-up. The evidence did not establish complete application recovery from those resource measurements alone.

The deployment changed memory, CPU, and process lifetime together. The evidence did not separate the benefit of larger memory from the memory reclaimed by restarting Keeper. The existing partition entries also survived that restart.

## Match each fix to the resource that grows

The resize provided immediate headroom. Persistent metadata and wide JSON parts remained. A durable response needs separate work for each resource:

1. Preserve enough Keeper memory for its tree, caches, temporary work, and process overhead.
2. Design a safe lifecycle for partition metadata that outlives the data TTL.
3. Reduce simultaneous file demand from wide JSON parts and concurrent merges.
4. Make destructor cleanup tolerate descriptor exhaustion and socket timeouts.
5. Monitor process descriptors and Keeper memory separately from host-wide resource totals.

[Partition lifecycle guidance](https://clickhouse.com/docs/best-practices/choosing-a-partitioning-key) helps match partitioning to data retention. [Insert batching guidance](https://clickhouse.com/docs/best-practices/selecting-an-insert-strategy) helps control part creation. [JSON guidance](https://clickhouse.com/docs/best-practices/use-json-where-appropriate) helps distinguish dynamic attributes from fields with stable types.

A lower merge limit trades fewer simultaneous files for slower background progress. A JSON representation with fewer separate streams trades file demand against other costs. The workload needs measurements before either change becomes a production recommendation.

The data lived for one hour. Partition metadata survived for about 80 days. During recovery, each wide JSON part brought thousands of open files. Capacity planning needed to account for all three lifetimes.
