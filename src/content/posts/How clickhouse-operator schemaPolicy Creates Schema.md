---
title: 'How clickhouse-operator''s schemaPolicy Creates Schema on New Hosts'
published: 2026-06-02
description: 'A source-grounded look at what schemaPolicy actually does when a new ClickHouse pod comes up: which engines it copies (it is engine-agnostic), why creation is not dependency-ordered, how the 10-pass retry loop self-heals a Kafka→MV→local→Distributed pipeline, and the one failure mode that gets silently marked "done".'
image: ''
tags: [ClickHouse, Kubernetes, clickhouse-operator, Kafka, Materialized Views, Big Data Engineering, DevOps]
category: 'Coding'
draft: false
lang: 'en'
slug: clickhouse-operator-schema-policy
---

## Why this matters

When you add a shard or replace a replica on an operator-managed ClickHouse cluster, the new pod comes up **empty**. Whether it ends up with your tables — and whether the right tables, in a state that actually works — is decided by one field in your `ClickHouseInstallation` (CHI): `schemaPolicy`.

In the [scale-up post](/posts/scaling-up-clickhouse-on-kubernetes/) I treated `schemaPolicy` as a black box: `None` means "no schema", `All` means "the operator copies it." This post opens the box. It answers three questions that decide whether you can trust it for a real ingestion pipeline:

1. **What does it actually create?** (Spoiler: more than you'd think — it's engine-agnostic.)
2. **Does it create in dependency order?** (No.)
3. **So what happens when a Materialized View is created before its source/target tables exist?**

Everything below is grounded in the [Altinity clickhouse-operator](https://github.com/Altinity/clickhouse-operator) source (paths from a recent `master`). All names are generic.

## A worked example: the pipeline everyone runs

The canonical streaming-ingest pattern in ClickHouse is four objects per dataset:

```sql
-- 1. source: a Kafka engine table
CREATE TABLE default.events_kafka  ( ... ) ENGINE = Kafka SETTINGS ...;

-- 2. storage: a ReplicatedMergeTree
CREATE TABLE default.events_local  ( ... )
  ENGINE = ReplicatedMergeTree('/clickhouse/tables/{shard}/default/events_local','{replica}') ...;

-- 3. pump: a Materialized View that reads Kafka, writes to storage
CREATE MATERIALIZED VIEW default.events_mv TO default.events_local
  AS SELECT * FROM default.events_kafka;

-- 4. fan-out: a Distributed table over the local storage
CREATE TABLE default.events
  ENGINE = Distributed('my_cluster','default','events_local', rand());
```

The MV has two hard dependencies: it can't be created unless **both** `events_kafka` (its `FROM`) and `events_local` (its `TO`) already exist. Hold onto that — it's the whole story.

## schemaPolicy: values, defaults, and gating

The field is per-cluster, with two independent knobs (`pkg/apis/clickhouse.altinity.com/v1/type_cluster.go`):

```yaml
clusters:
  - name: my_cluster
    schemaPolicy:
      replica: All   # None | All
      shard:   All   # None | All | DistributedTablesOnly
```

The normalizer defaults **both to `All`** — so unless you set `None` explicitly, schema management is on. Two gates decide whether anything runs for a given new host:

- **Replicated objects** (`pkg/model/chi/schemer/replicated.go`, `shouldCreateReplicatedObjects`): runs when `shard == All` and the cluster has ≥2 hosts, or when `replica != None` and the shard has ≥2 replicas. (You need at least one *other* host to copy from.)
- **Distributed objects** (`pkg/model/chi/schemer/distributed.go`, `shouldCreateDistributedObjects`): runs whenever `shard != None` and the cluster has ≥2 hosts.

## Myth-buster: the copy is engine-agnostic

A common belief is that `schemaPolicy` only copies `ReplicatedMergeTree` and `Distributed` tables. **That's wrong.** Look at the actual query that drives the "replicated objects" path (`pkg/model/chi/schemer/sql.go`, `sqlCreateTableReplicated`):

```sql
SELECT DISTINCT tables.name,
  replaceRegexpOne(create_table_query,
    'CREATE (TABLE|VIEW|MATERIALIZED VIEW|DICTIONARY|LIVE VIEW|WINDOW VIEW)',
    'CREATE \1 IF NOT EXISTS')
FROM clusterAllReplicas('my_cluster', system.tables) tables
LOCAL JOIN system.databases databases ON databases.name = tables.database
WHERE database NOT IN ('system','information_schema','INFORMATION_SCHEMA')
  AND databases.engine IN ('Ordinary','Atomic','Memory','Lazy')
  AND create_table_query != ''
  AND name NOT LIKE '.inner.%' AND name NOT LIKE '.inner_id.%'
SETTINGS skip_unavailable_shards=1, show_table_uuid_in_table_create_query_if_not_nil=1
```

There is **no filter on table engine**. The only exclusions are system databases and the `.inner.*` internal storage of implicit MVs. So a `Kafka` table, a `MaterializedView`, a plain `MergeTree`, a `Dictionary` — all of them are copied, verbatim, with `CREATE` rewritten to `CREATE … IF NOT EXISTS`. For our example pipeline, `events_kafka`, `events_local`, and `events_mv` all come through this one query. The Distributed path (`sqlCreateTableDistributed`) then adds `events` and re-asserts its local table.

Key consequence: schemaPolicy reads from **live cluster state** via `clusterAllReplicas`, *not* from your `.sql` files. It propagates whatever currently exists on the running replicas. If a table was never applied to the live cluster, it won't appear on the new host — the files on disk are irrelevant to this path.

## The order it builds in

Two layers of ordering exist, and one critical gap.

**Across groups** — `getReplicatedObjectsSQLs` concatenates the batch as databases → tables → functions (`replicated.go`), and `HostCreateTables` runs the **entire replicated batch first, then the distributed batch** (`pkg/model/chi/schemer/schemer.go`):

```go
err1 = s.ExecHost(ctx, host, replicatedCreateSQLs,  ...SetRetry(true))  // databases, tables, functions
err2 = s.ExecHost(ctx, host, distributedCreateSQLs, ...SetRetry(true))  // distributed + their locals
```

So databases precede tables, and the distributed fan-out comes after local storage. The distributed query even has an explicit `ORDER BY order` so local tables sort before the Distributed tables that reference them.

**Within the tables group** — here's the gap. `sqlCreateTableReplicated` is `SELECT DISTINCT tables.name …` with **no `ORDER BY`**. The order your `events_kafka`, `events_local`, and `events_mv` come back in is arbitrary. **The MV can absolutely be attempted before its source and target.**

## So what happens when the MV is created first?

It fails on the first try — and then the retry loop fixes it. Execution is wrapped in a multi-pass retry (`pkg/model/clickhouse/cluster.go`, `exec`):

```go
r.Retry(ctx, opts.Tries /* =10 */, "Applying sqls", ...
  func() error {
    var errors []error
    for i, sql := range queries {
      if len(sql) == 0 { continue }          // already-done statements are skipped
      err := conn.Exec(ctx, sql, opts)
      // ... (CREATE TABLE → ATTACH TABLE fallback for codes 253 / 57) ...
      if err == nil || strings.Contains(err.Error(), "ALREADY_EXISTS") {
        queries[i] = ""                       // success → blank it out for next pass
      } else {
        errors = append(errors, err)          // failure → keep it, retry next pass
      }
    }
    if len(errors) > 0 { return errors[0] }   // any error → run the closure again
    return nil
  })
```

`SetRetry(true)` sets `Tries = defaultMaxTries = 10` (`pkg/model/clickhouse/query_options.go`). Each pass blanks out the statements that succeeded, so only failures carry over. Walk it through for the worst case where the MV sorts first:

- **Pass 1**
  - `CREATE MATERIALIZED VIEW events_mv …` → **fails** (`UNKNOWN_TABLE`: `events_kafka`/`events_local` don't exist yet). Not `ALREADY_EXISTS`, so it **stays** in the list.
  - Later in the *same* pass, `events_kafka` and `events_local` run → **succeed** → blanked.
  - Pass ends with one error (the MV) → retry.
- **Pass 2**
  - Only the MV remains; its dependencies now exist → **succeeds** → blanked.
  - No errors → retry loop stops. Done.

A wrong initial order costs one extra pass, not a failure. With a dependency depth of 2–3 and a 10-pass budget, ordering problems within a batch are self-healing. As a bonus, the loop special-cases a replica already registered in Keeper, rewriting `CREATE TABLE` → `ATTACH TABLE` on error codes 253 and 57 — the standard "data dir survived but metadata didn't" recovery.

## The one failure mode that bites silently

The retry only rescues **ordering** problems where the dependency is *in the same batch*. It cannot conjure a dependency that isn't there. If `events_kafka` doesn't exist on the source cluster you're copying from, the MV fails all 10 passes — and then this happens (`pkg/controller/chi/worker-migrator.go`, `migrateTables`):

```go
if err := w.ensureClusterSchemer(host).HostCreateTables(ctx, host); err != nil {
    // ... logs an Error event ...
}
// ↓ runs regardless of the error above
host.GetCR().IEnsureStatus().PushHostTablesCreated(...)
return nil   // <- error is NOT propagated
```

The error is logged but **not returned**, and the host is **marked as "tables created" anyway**. On the next reconcile, the gate `shouldMigrateTables` sees `host.HasData()` is true — which really means "this FQDN is in `status.hostsWithTablesCreated`", not "the disk has data" (`pkg/apis/.../type_host.go`) — and **skips** the host. Net result: a partial schema, frozen, with no automatic retry.

So `schemaPolicy: All` is not fire-and-forget for a Kafka/MV pipeline. Transient ordering heals itself; a genuinely missing dependency leaves you broken *and* marked done. After adding hosts, verify with `system.tables` on the new pod and watch the operator's events — don't assume green just because the pod is Running.

## Two more facts worth internalizing

- **It only touches empty hosts.** Your existing, populated pods are skipped (`HasData()` gate), so flipping `None → All` on a live cluster does nothing to current data and triggers no restart — `schemaPolicy` isn't part of host config, so it never rolls pods (see [what triggers a rolling restart](/posts/clickhouse-on-k8s-rolling-restart-policy/)).
- **Kafka tables start consuming the moment they're created.** Copying `events_kafka` + `events_mv` onto a new replica makes it join the consumer group immediately and rebalance partitions across the cluster. That's a live data-plane effect, not just DDL — intended for added capacity, but worth expecting.

## Takeaways

- `schemaPolicy` defaults to `All`/`All`; it copies **all** non-system tables and views by engine-agnostic DDL from live replicas, plus Distributed tables and their locals — not just `Replicated*`.
- Creation is **not** dependency-ordered within the tables batch, so an MV can precede its source/target.
- A **10-pass retry loop** blanks successes and re-runs failures, so in-batch ordering issues converge on their own.
- A **truly missing dependency** fails permanently — and the operator **swallows the error and marks the host done**, so verify after the fact.
- For pipelines where correctness and ordering must be explicit, an init-script bootstrap (`/docker-entrypoint-initdb.d` with numbered `*.sql`) driven by your own DDLs is the deterministic alternative; `schemaPolicy: All` remains excellent for `ReplicatedMergeTree` replica self-healing.

*Related: [Scaling Up ClickHouse on Kubernetes](/posts/scaling-up-clickhouse-on-kubernetes/) · [What Triggers a Rolling Restart](/posts/clickhouse-on-k8s-rolling-restart-policy/) · [Moving Nodes Out of a Bare-Metal ClickHouse Cluster](/posts/moving-nodes-out-of-a-bare-metal-clickhouse-cluster/)*
