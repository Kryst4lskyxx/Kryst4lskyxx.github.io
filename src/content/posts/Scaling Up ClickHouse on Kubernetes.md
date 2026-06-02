---
title: 'Scaling Up ClickHouse on Kubernetes — Adding a Shard Without Downtime'
published: 2026-06-02
description: 'Adding nodes to an operator-managed ClickHouse cluster: why "two more nodes" means one shard, why the layout count and your Distributed cluster are two separate things, and the create-then-wire ordering that adds capacity with zero query errors.'
image: ''
tags: [ClickHouse, Kubernetes, clickhouse-operator, Kafka, Sharding, Big Data Engineering, DevOps]
category: 'Coding'
draft: false
lang: 'en'
slug: scaling-up-clickhouse-on-kubernetes
---

## Why this is trickier than "bump the replica count"

Scaling a ClickHouse cluster up on Kubernetes *looks* like a one-line change: raise the shard count in your `ClickHouseInstallation` (CHI) and let the [Altinity clickhouse-operator](https://github.com/Altinity/clickhouse-operator) reconcile. The pods do appear. But two things that the operator does **not** do for you decide whether the new capacity is actually used — and whether you take query errors on the way there:

1. **The operator does not move existing data.** A new shard starts empty; ClickHouse never rebalances historical rows onto it. Config controls *routing*, not *placement* — the same load-bearing fact from the [scale-*down* version of this work](/posts/moving-nodes-out-of-a-bare-metal-clickhouse-cluster/).
2. **The operator may not create your schema.** With `schemaPolicy: None`, the new pods come up with no databases or tables. If your production Distributed cluster starts routing to a shard whose local tables don't exist yet, those queries throw.

This is the inverse of [decommissioning a shard](/posts/moving-nodes-out-of-a-bare-metal-clickhouse-cluster/), and just like that operation, the safety lives entirely in the sequencing. All hostnames, cluster names, and table names below are generic.

## "Two more nodes" means one shard

A ClickHouse cluster is `shards × replicas`. The arithmetic constrains what "add N nodes" can mean. Starting from **4 shards × 2 replicas = 8 pods**:

| Action | Result | Node delta |
|---|---|---|
| Add a **shard** | 5 × 2 = 10 | **+2** |
| Add a **replica** layer | 4 × 3 = 12 | +4 |

So "two more nodes" on a 2-replica cluster is **one new shard** (`5 × 2`), not a third replica. The choice is not just arithmetic, though — it's about what you're scaling *for*:

- **Add a shard** when the bottleneck is **per-shard CPU/memory or write throughput**. It spreads query fan-out and ingest across more nodes — but only for *new* data (see below).
- **Add a replica** when the bottleneck is **read concurrency or HA**. No rebalance needed; every replica already holds the full shard. But that's `+shards` nodes, not two.

The rest of this post assumes the shard case: going from 4 to 5 shards, adding `ch-e01` and `ch-e02`.

## Will it cause downtime?

**No cluster-wide downtime** — if you sequence it. Adding a shard is *additive*, and nothing about it forces the existing pods to restart.

Recall the operator's restart rules ([full breakdown here](/posts/clickhouse-on-k8s-rolling-restart-policy/)): a pod rolls only on a **pod-template change** (image, resources, volumes, affinity…) or when the **config reboot engine** says a changed path needs it. Adding a shard does neither to shards 0–3:

- The new shard is **new StatefulSets** — Kubernetes creates them; it does not touch the existing ones.
- The operator's own auto-generated cluster definition gains a shard, but `remote_servers/*` is an explicit **`"no"`** in the restart policy — it's **hot-reloaded**, not restarted.
- With 2 replicas per shard, even if a pod *did* bounce, its peer keeps serving.

There are two non-downtime effects to expect:

:::note
When the new nodes' Kafka engine tables join the consumer group, Kafka **rebalances partitions** across all consumers — a few seconds of ingest pause cluster-wide, not query downtime.
:::

:::warning
The real risk is **query errors, not downtime**: if the new shard is added to your production Distributed cluster *before* its local tables exist, fan-out queries hit a shard with missing tables and throw `UNKNOWN_TABLE`. Note that `skip_unavailable_shards` only skips *unreachable hosts* — it will **not** save you from a *missing table* on a reachable host. The ordering below avoids this entirely.
:::

## The two definitions that are not the same definition

This is the scale-up echo of the [two-numbers problem](/posts/moving-nodes-out-of-a-bare-metal-clickhouse-cluster/). There are **two** places a shard count lives, and keeping them decoupled is what makes a zero-error rollout possible.

### Definition 1: the operator layout (`shardsCount`)

```yaml
# ClickHouseInstallation
clusters:
  - name: events
    layout:
      shardsCount: 4      # <-- bumping this CREATES PODS
      replicasCount: 2
```

`shardsCount` controls how many StatefulSets exist and feeds the operator's *auto-generated* cluster (named after the CHI). Raise it and pods appear; lower it and pods are deleted.

### Definition 2: your production cluster (`<remote_servers>`)

Many real deployments don't query the operator's auto-cluster. They route through a **hand-maintained** cluster in a config file, so they can control `internal_replication`, subsets, and naming:

```xml
<!-- config.d/cluster.xml -->
<clickhouse>
  <remote_servers>
    <events_all>
      <shard>
        <internal_replication>true</internal_replication>
        <replica><host>ch-a01</host><port>9000</port></replica>
        <replica><host>ch-a02</host><port>9000</port></replica>
      </shard>
      <!-- ...shards b, c, d... -->
    </events_all>
  </remote_servers>
</clickhouse>
```

Your `Distributed` tables point at `events_all` — **this file**, not `shardsCount`. So bumping `shardsCount` alone creates the pods but leaves `events_all` at four shards: the new nodes exist but take no production traffic.

:::tip
That decoupling is usually a footgun (forget to edit the file and the new shard sits idle) — but here it's the **feature** that makes the rollout safe. You can bring the pods up, fully populate them *outside* the read path, and only wire them in once they're ready.
:::

## The ordering: create, then wire

Three phases. Phase 1 creates idle pods, Phase 2 fills them, Phase 3 puts them in the path.

### Phase 1 — Bump `shardsCount` only

Raise `shardsCount: 4 → 5`. **Do not touch `cluster.xml` yet.**

```diff
   layout:
-    shardsCount: 4
+    shardsCount: 5
     replicasCount: 2
```

The operator creates `ch-e01` / `ch-e02`, empty. Because `events_all` still lists four shards, **zero production reads or writes reach them.** No impact, nothing to undo if you pause here.

:::note
Before this, make sure the cluster has somewhere to *put* the pods. If your ClickHouse pods are pinned with a `nodeSelector` + taint and `ShardAntiAffinity` (one big pod per node), you need **two new Kubernetes worker nodes** labeled and tainted to match — provisioned first, or the new pods sit `Pending`.
:::

### Phase 2 — Create the schema on the new shard

With `schemaPolicy: None`, you do this yourself. Create tables on **both** new nodes, in **dependency order**:

```sql
-- 1. local table first — everything else points at it
CREATE TABLE events ON ...        -- ReplicatedMergeTree('/clickhouse/tables/{shard}/events', '{replica}')
-- 2. Kafka engine source
CREATE TABLE events_kafka ...
-- 3. materialized view (needs both the Kafka source and the local target)
CREATE MATERIALIZED VIEW events_mv TO events AS SELECT ... FROM events_kafka;
-- 4. the Distributed table (so the node can also serve as a query initiator)
CREATE TABLE events_dist AS events ENGINE = Distributed(events_all, currentDatabase(), events, rand());
```

Two things make this work:

- Run the DDL **directly on `ch-e01` and `ch-e02`**, *not* `ON CLUSTER events_all` — the static cluster doesn't include the new shard yet, so `ON CLUSTER` wouldn't reach it. Create on both replicas; they share a `{shard}` macro (the operator assigns it from the layout position) and different `{replica}` values, so their `ReplicatedMergeTree` ZK paths are unique and they replicate to each other.
- The moment `events_kafka` + `events_mv` exist, the new shard **joins the consumer group and starts ingesting** — exactly what you want. This is the brief rebalance from earlier.

:::caution
Verify the `{shard}` macro on the new pods (`SELECT * FROM system.macros`) before creating Replicated tables. If two nodes accidentally share the same `{shard}` *and* `{replica}`, they'd collide on one ZK path instead of forming a new shard. The operator normally gets this right from the layout; confirm it anyway.
:::

### Phase 3 — Wire the shard into the production cluster

Now add the fifth `<shard>` block to `cluster.xml`:

```xml
      <shard>
        <internal_replication>true</internal_replication>
        <replica><host>ch-e01</host><port>9000</port></replica>
        <replica><host>ch-e02</host><port>9000</port></replica>
      </shard>
```

`remote_servers` is hot-reloaded — **no restart**. `events_all` is now five shards, every one of which already has tables and is consuming. Distributed reads and writes safely include the new shard.

### Verify at each step

```sql
-- new consumers joined, partitions assigned
SELECT hostName(), * FROM clusterAllReplicas('events_all', system.kafka_consumers);

-- new data landing on the new shard
SELECT hostName(), count() FROM clusterAllReplicas('events_all', currentDatabase(), events)
GROUP BY hostName();

-- cluster topology reflects 5 shards
SELECT shard_num, host_name FROM system.clusters WHERE cluster = 'events_all';
```

## Where the new shard's data actually comes from

A shard you just added is empty, and ClickHouse will **not** backfill it. How it fills depends on your write path:

- **Kafka per-node ingestion** (`Kafka → MV → local`, the pattern above): distribution is governed by **Kafka consumer-group partition assignment**, *not* the Distributed sharding key. The gate is the **topic's partition count** — every consumer beyond `partition_count` sits idle. Going from 8 to 10 consumers with only 8 partitions means your two new nodes pull *nothing*. Check the partition count before you scale, and add partitions if needed.
- **`INSERT` into the Distributed table**: distribution follows the **sharding key**. `rand()` or `cityHash64(key)` spreads over five shards immediately; a key hardcoded to `% 4` would strand the new shard. Widening the cluster changes the modulus for *new* rows only.

Either way, **historical data stays on the original four shards.** Queries over old data don't get lighter per-shard until the new shard accrues its share over time. If your tables carry a short **TTL**, the imbalance self-corrects as old data ages out — the cleanest path. If you need relief on historical queries *now*, that's a separate, explicit rebalance (part moves or `INSERT … SELECT` redistribution), not something scaling does for free.

:::tip
If you scaled to relieve **query CPU/memory** pressure, adding aggregate nodes helps — but it does **not** change per-node admission limits. Keep guardrails like `max_concurrent_queries` in place; revisit them after scaling rather than assuming more nodes removed the need. (See the [`CANNOT_SCHEDULE_TASK` post](/posts/debugging-clickhouse-cannot-schedule-task/) for how admission storms bite under burst.)
:::

## Takeaways

- **`shards × replicas` constrains the move:** on a 2-replica cluster, "two more nodes" is **one shard**, not a third replica. Add a shard for per-shard CPU/throughput; add a replica for HA/read concurrency.
- **No downtime, by sequencing.** An additive shard creates new StatefulSets and hot-reloads `remote_servers` — existing pods don't roll. The only failure mode is *query errors* from wiring the shard in before its tables exist.
- **Two shard definitions:** the operator `shardsCount` (creates pods) and your static `<remote_servers>` (carries production traffic) are independent. Their decoupling is what lets you populate a shard out-of-band.
- **Order is load-bearing:** bump `shardsCount` → create tables directly on the new nodes (local → kafka → mv → dist) → add the shard to `cluster.xml`. Never the reverse.
- **The operator moves no data and, under `schemaPolicy: None`, creates no schema.** New capacity only helps *new* data; rebalancing historical rows is your job (or your TTL's).
- For Kafka ingestion, the real distribution gate is the **topic partition count**, not the sharding key — scale partitions alongside shards.

Scaling up and [scaling down](/posts/moving-nodes-out-of-a-bare-metal-clickhouse-cluster/) are mirror images: the engine will happily let you create idle capacity or orphan data, so in both directions the correctness lives in the sequencing, not the YAML.
