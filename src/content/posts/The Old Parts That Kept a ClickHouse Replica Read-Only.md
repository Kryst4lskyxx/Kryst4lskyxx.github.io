---
title: 'The Old Parts That Kept a ClickHouse Replica Read-Only'
published: 2026-10-27
description: 'A read-only replica matched a known ClickHouse startup bug fixed by PR #103537. The running fork still used the affected code.'
image: ''
tags: [ClickHouse, Debugging, Replication, Keeper, TTL]
category: 'Coding'
draft: false
lang: 'en'
slug: clickhouse-readonly-replica-old-parts
---

One table stayed read-only after a ClickHouse restart. Its Keeper session stayed active. Other tables on the same server accepted writes.

The error named two parts from June. A part stores a batch of rows and related files. Removing the old files while the server was stopped allowed the table to start again.

The failure matched a known startup bug involving overlapping empty unexpected parts. Upstream fixed that bug in [PR #103537](https://github.com/ClickHouse/ClickHouse/pull/103537). The running fork did not contain the fix.

The investigation also found a reason to preserve the rest of that server's data. Its peer lacked several recent parts. Rebuilding the failed copy from that peer risked losing recoverable data.

After recovery, a separate audit found 25.10 TB of expired active data across the cluster. A separate configuration limit blocked automatic deletion of some expired parts. Recovery of one table did not resolve that backlog.

The incident occurred on October 27, 2026. The cluster ran a fork of ClickHouse `25.8.11.66`. Host and table names below are anonymous. The quoted error and measured counts remain unchanged. The source links point to upstream code that matched the relevant fork implementation.

## The alert named one table replica

A shard holds one portion of a table. A replica stores a copy of that portion. The cluster had ten shards with two replicas each, for twenty servers.

Prometheus reported `ReadonlyReplica=1` on one server. That metric counted read-only table replicas, not read-only servers. The affected server contained many other replicated tables.

Keeper held metadata, records that described the stored parts. Each replica used those records to coordinate its work. The investigation compared those records with the local part state.

The SQL view identified one affected table. I will call it `analytics.events_a_local` on `replica-a`. Its peer on the same shard was `replica-b`.

| Field on the affected table | Value |
| --- | --- |
| `is_readonly` | `1` |
| `is_session_expired` | `0` |
| `active_replicas` / `total_replicas` | `1` / `2` |
| `queue_size` | `0` |
| `last_queue_update` | `1970-01-01 00:00:00` |
| `readonly_start_time` | `NULL` |
| `zookeeper_exception` | Empty |

A session represents an active client connection. The affected server had an active Keeper session. The evidence did not point to a current loss of that connection.

## An empty queue did not mean a healthy replica

A replication queue contains work that a replica must apply. An empty queue usually looks reassuring. Here, the timestamp showed that the table never initialized its queue.

The date in 1970 was an unset timestamp. It did not mean that the table was decades behind. The replica never reached the stage that updated the value.

This distinction also applied to `absolute_delay`. A very large value on an uninitialized table did not measure a real backlog of that age. I used the peer's queue to understand the missing work instead.

The following example reads the table on `replica-a`. It uses the anonymous database and table names from this post. The account must permit only read operations.

```sql
SELECT
    database,
    table,
    is_readonly,
    is_session_expired,
    active_replicas,
    total_replicas,
    queue_size,
    last_queue_update,
    last_queue_update_exception,
    zookeeper_exception
FROM system.replicas
WHERE database = 'analytics'
  AND table = 'events_a_local'
LIMIT 1
SETTINGS
    max_memory_usage = 1073741824,
    max_execution_time = 15,
    timeout_before_checking_execution_speed = 0,
    max_estimated_execution_time = 30,
    max_rows_to_read = 1000000,
    max_bytes_to_read = 1073741824,
    max_result_rows = 1000,
    max_threads = 2,
    use_query_cache = 0;
```

The investigation used bounded queries throughout. A query across every replica multiplies the amount of metadata read. I used compact results and explicit limits rather than scanning application rows.

## The error named two old parts

A merge combines parts into a new part. Merge history affects the level in each part name. The error counter contained this message:

```text
Part 20260625_0_33900_8_33890 intersects part 20260625_30295_33449_8_33890.
It is a bug or a result of manual intervention in the ZooKeeper data.
```

The final sentence listed possible explanations. It did not prove that someone edited Keeper. I treated the part names as evidence and left their original cause unresolved.

These names contained the partition date, block range, merge level, and mutation version. A block number identifies a position in the partition's write sequence. A mutation changes existing data.

| Part | First block | Last block | Merge level | Mutation version |
| --- | ---: | ---: | ---: | ---: |
| `20260625_0_33900_8_33890` | 0 | 33,900 | 8 | 33,890 |
| `20260625_30295_33449_8_33890` | 30,295 | 33,449 | 8 | 33,890 |

The first numeric range enclosed the second. ClickHouse still did not consider the first part a valid containing part. Both parts had merge level 8, but their block ranges differed.

The version's [containment rule](https://github.com/ClickHouse/ClickHouse/blob/02ec3a1ea1e08fb18d2d9638f00b2b557fa1cc1c/src/Storages/MergeTree/MergeTreePartInfo.h#L83-L99) required more than numeric enclosure. Equal-level parts with different block ranges did not satisfy that rule. The [part set](https://github.com/ClickHouse/ClickHouse/blob/02ec3a1ea1e08fb18d2d9638f00b2b557fa1cc1c/src/Storages/MergeTree/ActiveDataPartSet.cpp#L140-L164) rejected their intersection with `LOGICAL_ERROR`.

Neither replica listed June 25 parts in its Keeper part list. The cluster's `system.parts` table also contained no June 25 entries for this table. That evidence pointed toward leftover files outside the registered part set.

It did not prove the contents of those files. Unexpected files can remain on disk without appearing in `system.parts`. I did not inspect the removed directories or recover a complete startup stack.

## A startup error left the table read-only

The source contained a matching failure path. During startup, [`checkPartsImpl`](https://github.com/ClickHouse/ClickHouse/blob/02ec3a1ea1e08fb18d2d9638f00b2b557fa1cc1c/src/Storages/StorageReplicatedMergeTree.cpp#L1846-L1858) built a set of empty unexpected parts. An unexpected part is a local part absent from Keeper's expected list.

The loop accepted non-broken parts with zero rows that no other unexpected part contained. It added their names to `ActiveDataPartSet`. That structure rejected intersections without valid containment.

The [attachment thread](https://github.com/ClickHouse/ClickHouse/blob/02ec3a1ea1e08fb18d2d9638f00b2b557fa1cc1c/src/Storages/MergeTree/ReplicatedMergeTreeAttachThread.cpp#L57-L113) treated this error as a failed initialization. It retried selected coordination failures and replica-status changes. It did not schedule another attempt for this logical error.

That behavior explained the persistent state. Keeper stayed connected, but the table never started replication. A low error count did not mean that the problem disappeared.

Two earlier `SYSTEM RESTART REPLICA` commands returned successful SQL responses. The table still remained read-only afterward. The source [explicitly separates a startup attempt from successful replication](https://github.com/ClickHouse/ClickHouse/blob/02ec3a1ea1e08fb18d2d9638f00b2b557fa1cc1c/src/Storages/StorageReplicatedMergeTree.cpp#L5613-L5619).

## The bug and the merged fix

The bug was the use of a strict part set for empty unexpected parts. Two such parts can overlap without valid containment. `ActiveDataPartSet::add` then threw `LOGICAL_ERROR`, and the attachment thread left the table read-only.

Upstream merged [PR #103537: Fix `LOGICAL_ERROR` "Part X intersects previous part Y" during ReplicatedMergeTree startup](https://github.com/ClickHouse/ClickHouse/pull/103537). The [merged change](https://github.com/ClickHouse/ClickHouse/commit/4adfb717cc2f27b45b619c99fab95b2a3eae7d8b) introduced `OverlappingPartCovering`. It kept overlapping empty markers available for later coverage decisions.

The new structure kept a primary index for non-overlapping markers and a separate list for overlapping markers. A lookup used both groups. An overlapping marker therefore still covered other unexpected parts.

An early approach skipped the second marker after an intersection. That lost its coverage and risked a later `TOO_MANY_UNEXPECTED_DATA_PARTS` failure. The final patch preserved coverage instead of dropping the marker from the lookup.

The running fork still contained the old implementation. Its source and the observed state strongly matched this bug. The missing file inspection limited the diagnosis to that evidence. The offline recovery removed the immediate obstruction without installing the upstream fix.

## The writable peer did not contain every recent part

The peer accepted writes, but its replication queue contained ten stalled operations. Several reported `NO_REPLICA_HAS_PART`. The message meant that no active replica supplied the requested part or a covering part.

The inactive replica still stored those parts. One merged part contained 160,411 rows. Six more active parts brought the inspected total to 248,707 rows.

Those seven parts did not overlap. Their existence showed that useful data remained on the failed replica. It did not establish a complete count of unique data or prove application-level data loss.

Deleting the failed replica and copying everything from its peer put that data at risk. The peer was already asking for data that the failed replica retained. The recovery plan therefore preserved the existing copy.

## We recovered without deploying a code change

The operator stopped the affected ClickHouse server, removed the implicated part files from disk, and started the server again. The table recovered. No code deployment occurred during that recovery.

The later measurements covered all twenty servers. Every server reported zero read-only replicas and zero expired Keeper sessions. Replication delay was zero to one second, and Prometheus independently reported zero read-only replicas.

This was the observed recovery, not a general file-deletion procedure. I did not observe a backup, inspect the removed files, or establish their row counts. The successful restart supported the link to the local files without answering those questions.

Do not remove files from a running ClickHouse server. Active parts also have records in Keeper. Removing their directories directly can leave the remaining replica unable to fetch data.

If an offline repair requires file changes, preserve the original directories before any removal. Make sure that the selected files are obsolete before the repair. Keep current parts that the peer still needs.

## Restoring Keeper metadata addressed a different problem

`SYSTEM RESTORE REPLICA` rebuilds missing Keeper metadata from local data. The affected replica's metadata still existed. Its replica path and host record remained present throughout the investigation.

The running version [rejected restoration when the replica path existed](https://github.com/ClickHouse/ClickHouse/blob/02ec3a1ea1e08fb18d2d9638f00b2b557fa1cc1c/src/Storages/StorageReplicatedMergeTree.cpp#L6830-L6850). Detaching and attaching the table also left the conflicting files in place. Neither operation removed the condition that blocked startup.

Do not delete a replica's Keeper path to force this recovery route. That changes the metadata problem and can change how ClickHouse handles local parts. In this incident, preserving recent data on the failed server took priority.

## The TTL audit found another backlog

A TTL rule sets when data expires. Both affected tables used a daily partition and a 31-day TTL. With an anonymous time column, the policy looked like this:

```sql
PARTITION BY toYYYYMMDD(event_time)
TTL toDate(event_time) + INTERVAL 31 DAY
SETTINGS ttl_only_drop_parts = 1
```

On October 27 UTC, September 26 and earlier were fully expired. September 27 expired at midnight on October 28 UTC. The date conversion in the expression determined this boundary.

I counted active parts whose latest delete-TTL timestamp was already in the past. This measured fully expired parts without scanning the stored rows. The result included both replica copies.

| Anonymous table | Affected nodes | Expired parts | Date span | Physical size |
| --- | ---: | ---: | --- | ---: |
| Table A | 20 | 1,306 | August 24–September 26 | 20.23 TB |
| Table B | 20 | 984 | July 26–September 26 | 4.87 TB |

The spans contained gaps. The total covered 65 distinct table/date pairs, with 2,290 parts and 25.10 TB. It was not 25.10 TB of unique logical data.

The other thirty populated business tables contained no fully expired regular parts in that snapshot. Some patch parts lacked delete-TTL timestamps. A patch stores changes that ClickHouse applies to existing rows.

Every such patch part belonged to September 28 or September 29. Those dates remained inside the retention window. I kept these parts separate from the expired-part count.

The TTL backlog did not establish how the June files formed. Active expired parts and unregistered leftover files are different sets of objects. Removing one startup obstruction did not prove that retention worked across the cluster.

## The merge limit also restricted TTL deletion

Table A used a 20 GiB merge-size limit. Table B used a 5 GiB limit. Both tables also used `ttl_only_drop_parts=1` and a minimum twelve-hour interval between TTL merge attempts for a partition.

That configuration removed whole expired parts rather than deleting expired rows inside a part. The [selector for whole-part deletion](https://github.com/ClickHouse/ClickHouse/blob/02ec3a1ea1e08fb18d2d9638f00b2b557fa1cc1c/src/Storages/MergeTree/Compaction/MergeSelectorApplier.cpp#L65-L84) still received the table's fixed merge-size limit. The [range builder](https://github.com/ClickHouse/ClickHouse/blob/02ec3a1ea1e08fb18d2d9638f00b2b557fa1cc1c/src/Storages/MergeTree/Compaction/MergeSelectors/TTLMergeSelector.cpp#L14-L18) rejected a part above that limit.

The name `max_bytes_to_merge_at_max_space_in_pool` described the relevant configuration value. It also affected this deletion path, even though dropping an expired part did not rewrite its rows. The row-deletion fallback was disabled.

Of the expired parts, 1,064 exceeded their table's limit. They occupied 13.57 TB. Waiting for another TTL cycle did not remove the size restriction.

Another 1,226 expired parts remained below their limits and occupied about 11.53 TB. Their oldest dates were September 23 for Table A and September 24 for Table B. The size restriction alone did not explain their continued presence.

I did not trace every scheduler decision or inspect the history of TTL operations. The data supported a specific blocked set. It did not support one explanation for every remaining byte.

For active data that expires by whole days, partition operations preserve ClickHouse's metadata coordination. The [partitioning guidance](https://clickhouse.com/docs/best-practices/choosing-a-partitioning-key) describes this use of partitions for data retention. This cleanup belongs in SQL rather than another round of direct file removal.

## Detached files needed separate review

A detached part sits outside the active table. TTL selection does not normally process those files. The audit found 262 detached entries across nineteen servers, with a total size of 1.11 TB.

These entries included broken parts, parts covered by broken parts, ignored parts, and archive files. Their names included dates from April through September. Some entries existed for recovery, so their age alone did not justify deletion.

System logs formed another separate category. `system.query_log` occupied about 5.65 TB across the cluster and had no table TTL. The intended retention for those logs remained unspecified.

An expired active part, a detached recovery copy, and an unregistered directory require different treatment. `system.parts` describes registered parts. It does not provide a complete inventory of every file on a server.

The recovered table accepted writes again. The remaining work was explicit: review the expired partitions, correct the TTL restrictions, and decide which recovery files and logs to retain.
