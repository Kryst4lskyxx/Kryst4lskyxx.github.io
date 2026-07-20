---
title: 'Migrating ClickHouse Keeper Two Ways (and When Each One Betrays You)'
published: 2026-07-20
description: 'Moving replication metadata between Keeper ensembles: the live reconfig membership dance, the SYSTEM RESTORE REPLICA rebuild, why zkcopy silently corrupts your data, and the version mismatch that segfaulted a production ensemble into switching methods mid-migration.'
image: ''
tags: [ClickHouse, ClickHouse Keeper, ZooKeeper, Raft, Big Data Engineering, DevOps, Replication]
category: 'Coding'
draft: false
lang: 'en'
slug: migrating-clickhouse-keeper-two-ways
---

## The job nobody scopes correctly

"Move Keeper off the old hosts" sounds like a hardware swap. It isn't. ClickHouse
Keeper doesn't hold *your* data — it holds the **metadata that makes replicas
agree**: which replicas exist, the replication log, part block numbers,
mutations, quorum state. A lot of that lives in **sequential znodes**, whose
monotonically increasing counter is maintained by Keeper itself. Move that
metadata carelessly and you don't get an outage — you get *silent divergence*,
which is much worse.

I recently migrated three ClickHouse environments off their old Keeper ensembles:
one dev cluster across clouds, and two production clusters onto new datacenter
hardware. I used **two different methods**, because halfway through I learned —
the expensive way — that the elegant method has sharp edges. This is the writeup
I wish I'd had going in.

All hostnames, ports, and roots below are genericized; the failure modes,
commands, and Keeper internals are verbatim from the real runs.

## First, the rule that saves your data

:::warning
**Do not use `zkcopy` (or any generic tree-copy tool) to migrate ClickHouse
Keeper.** It copies znodes as plain key/values and does **not** preserve the
*sequential* counter behind ClickHouse's replication paths. Restore those naively
and two replicas can mint the **same block number** — instant, silent data
divergence. This is the single most important sentence in this post.
:::

Everything below exists to either **preserve** those counters (Method 1) or make
ClickHouse **re-derive** them safely from the parts on disk (Method 2). Never to
copy them by hand.

## The two methods at a glance

| | **Method 1: `reconfig`** | **Method 2: `SYSTEM RESTORE REPLICA`** |
|---|---|---|
| Idea | New nodes join the *live* ensemble over Raft, old nodes leave | Stand up a *new empty* ensemble; ClickHouse rebuilds metadata into it |
| Write freeze | None | Short freeze during cutover |
| Version sensitivity | **High** (target ≥ source, both ≥ 23.3, don't mix) | None |
| Cross-DC / cross-cloud | Risky (quorum spans both sites) | Safe |
| Touches the source? | **Yes** — mutates the live source ensemble | No — source untouched |
| Rollback | Staged, gets harder over time | Trivial (repoint back) |

The short version: **default to Method 2.** Reach for Method 1 only when you
truly can't take a write freeze *and* source/target are the same datacenter on
compatible, modern versions. Here's why.

## Method 1: the live membership dance (`reconfig`)

ClickHouse Keeper is a Raft state machine. Its `reconfig` command changes Raft
*membership* at runtime. The trick is beautiful on paper:

1. Configure the new nodes with a **transition membership** — the Raft config
   lists **both** the old (source) nodes *and* the new (target) nodes.
2. Start each new node. It doesn't form its own ensemble; it sits waiting to be
   admitted.
3. From a live member, `reconfig add` the new nodes **one at a time**. Each new
   node receives the leader's snapshot + log over Raft — sequential counters and
   all — and becomes a synced follower.
4. Repoint ClickHouse's `zk.xml` at the new nodes.
5. `reconfig remove` the old nodes, one at a time.

No freeze, because the state is never dumped and reloaded — it's *replicated*
live through Raft. When it works, it's the cleanest possible migration.

### The essential config

Every node needs an explicit identity and, crucially, reconfiguration turned on:

```xml
<keeper_server>
    <tcp_port>2185</tcp_port>
    <server_id>11</server_id>
    <log_storage_path>/data/keeper/coordination/log</log_storage_path>
    <snapshot_storage_path>/data/keeper/coordination/snapshots</snapshot_storage_path>
    <storage_path>/data/keeper/state</storage_path>

    <enable_reconfiguration>true</enable_reconfiguration>

    <raft_configuration>
        <!-- transition: source ids 1-3 AND target ids 11-15 together -->
        <server><id>1</id><hostname>keeper-old-01</hostname><port>9444</port></server>
        <server><id>2</id><hostname>keeper-old-02</hostname><port>9444</port></server>
        <server><id>3</id><hostname>keeper-old-03</hostname><port>9444</port></server>
        <server><id>11</id><hostname>keeper-new-01</hostname><port>9444</port></server>
        <!-- ...12-15... -->
    </raft_configuration>
</keeper_server>
```

:::note
Target `server_id`s must **not** collide with source ones. Source is usually
`1–3`; I use `11–15` for the new nodes so there's no ambiguity when I later remove
members by id.
:::

### The add loop

`reconfig` takes **one member per call** — NuRaft only supports single-member
changes. Add, then *verify the node is a synced follower* before the next:

```bash
clickhouse keeper-client -h keeper-old-01 -p 2185 \
  -q 'reconfig add "server.11=keeper-new-01:9444;participant"'

# verify before moving on
clickhouse keeper-client -h keeper-old-01 -p 2185 -q "get /keeper/config"
echo mntr | nc -w2 keeper-new-01 2185 | grep -E 'zk_server_state|zk_synced_followers'
```

Add as `participant` (it joins the voting set — required, since the source is
leaving). A `learner` syncs data but can't be promoted in place; you'd have to
remove and re-add it. Don't bother — go straight to `participant`.

After all five new nodes are in, you briefly have an **eight-node ensemble** (3
old + 5 new), one leader and seven synced followers, all with identical
`zk_znode_count`. That's your green light to cut ClickHouse over.

:::caution
While the combined ensemble spans both host sets, **every write needs a quorum
that spans both sites.** Inside one datacenter that's fine. Across a WAN it's a
latency tax on every single write — which is the first reason Method 1 is a bad
fit for cross-cloud moves.
:::

### Then it segfaulted a production ensemble

Here's where the paper elegance met a five-year-old binary.

One production source ran an **older Keeper build (23.5.2.7)**. The migration
plan required `enable_reconfiguration` on the source, so I added the flag and
did the rolling restart. All three source nodes came back up… and immediately
began a crash loop:

```
<Fatal> BaseDaemon: ########## short fault info ############
RaftInstance: Trying to rollback invalid ZXID ...
status=11/SEGV
```

The flag had tripped a Raft log/snapshot path that this build couldn't handle,
and it corrupted its own coordination state on restart. I reverted the flag,
restarted, and the ensemble stabilized — but the lesson was final:

:::warning
`reconfig` is **version-fragile**. `enable_reconfiguration` on an old source
build can segfault the whole ensemble via a bad ZXID rollback. Never enable it in
prod without testing on a **non-prod copy of the exact source build** first. If
the source is old, don't use Method 1 at all.
:::

That single failure is what turned a "one method" migration into a two-method
one. Two of my three environments ended up on Method 2 — one for a version
mismatch, one because it was cross-cloud.

## Method 2: rebuild into an empty ensemble (`SYSTEM RESTORE REPLICA`)

The insight: ClickHouse can **reconstruct its Keeper metadata from the parts
already on local disk.** So instead of migrating state, you throw the old state
away and let ClickHouse rebuild it into a fresh ensemble. The source is never
touched, which makes this both safer and version-agnostic.

1. Provision a **brand-new, independent** ensemble — its Raft config lists **only
   the new nodes**.
2. Bootstrap it clean and verify it's *empty*.
3. Freeze writes.
4. Repoint `zk.xml` at the new ensemble and restart ClickHouse. Every replicated
   table comes up **read-only** — expected, the new Keeper has no metadata yet.
5. Run `SYSTEM RESTORE REPLICA` per table, on every node.
6. Verify, unfreeze, done.

### Verify the new ensemble is actually empty

This step bit me, so it gets its own warning. A "fresh" five-node ensemble came
up reporting **eight members** (`zk_synced_followers=7`), because earlier failed
join attempts had left stale Raft state on disk. Config alone doesn't fix it:

:::caution
To truly bootstrap a clean ensemble you must **wipe the on-disk Raft state**, not
just fix the config. Stop every node, confirm the config lists only the intended
members, then delete the coordination log, snapshots, **and** the state dir
before restarting:

```bash
systemctl stop clickhouse-keeper       # on ALL new nodes first
rm -rf /data/keeper/coordination/log/* \
       /data/keeper/coordination/snapshots/* \
       /data/keeper/state/*
systemctl start clickhouse-keeper      # then bring them all up
```
:::

A clean ensemble shows exactly one leader, `N-1` synced followers, and `ls /`
returning only `keeper`.

### The rebuild

After repointing `zk.xml` and restarting, let ClickHouse tell you what's broken
and generate its own fix. This is my favorite trick — never hand-type the table
list:

```sql
-- generate the restore statements from the read-only set, on each node
SELECT concat('SYSTEM RESTART REPLICA `', database, '`.`', table, '`;')
FROM system.replicas WHERE is_readonly;

SELECT concat('SYSTEM RESTORE REPLICA `', database, '`.`', table, '`;')
FROM system.replicas WHERE is_readonly;
```

Then run the emitted pairs **on every node** — `RESTORE REPLICA` is node-local;
it re-registers the replica and rebuilds its metadata from local parts:

```sql
SYSTEM RESTART REPLICA `default`.`events_local`;
SYSTEM RESTORE REPLICA `default`.`events_local`;
```

### The gotcha nobody warns you about

Once you've repointed to the empty Keeper, **`ON CLUSTER` DDL stops working** —
there's no cluster/DDL metadata in the new ensemble yet. So the write-freeze you
do *before* cutover (detaching Kafka-engine tables) has to be done **node-locally**
if you do it after repointing:

```sql
-- BEFORE repoint you can use ON CLUSTER; AFTER repoint you cannot.
SELECT concat('DETACH TABLE IF EXISTS `', database, '`.`', name, '`;')
FROM system.tables WHERE engine = 'Kafka';
```

:::tip
Sequence it so the freeze and detaches happen while the *old* Keeper is still
wired up (so `ON CLUSTER` still works), then repoint, then restore. If you slip
and detach after repointing, just do it per-node — no `ON CLUSTER`.
:::

## The tooling potholes

Two things cost me time that have nothing to do with Raft.

**`clickhouse keeper-client` may not have `reconfig`.** On several builds it
errors with `Service not found` or a syntax error. The fix is to drive Keeper
from Python `kazoo`, whose `reconfig()` maps to the ZooKeeper-compatible
reconfiguration API:

```python
# add one member to the source ensemble
from kazoo.client import KazooClient
zk = KazooClient(hosts="keeper-old-01:2185,keeper-old-02:2185,keeper-old-03:2185")
zk.start(timeout=15)
res = zk.reconfig(joining="server.11=keeper-new-01:9444;participant",
                  leaving=None, new_members=None)
print(res[0].decode() if isinstance(res, tuple) else res)
zk.stop(); zk.close()
```

`kazoo` also handles the chroot creation cleanly (`zk.ensure_path("/my-root")`)
when a cluster uses a `<root>` — which, incidentally, lives in ClickHouse's
`zk.xml`, **not** in the Keeper server config. Install with
`apt-get install python3-kazoo`.

**`SYSTEM SYNC REPLICA` "timing out" is usually a lie.** After the ECO cutover I
watched a `SYSTEM SYNC REPLICA` sit for 300 seconds and throw:

```
Code: 159. DB::Exception: ... SYNC REPLICA ... command timed out.
```

Nothing was wrong. The plain (STRICT) form blocks until the replication queue
hits **empty**, and under live ingestion the queue *never* hits empty, so it
burns `receive_timeout` and throws — even though every replica was perfectly in
sync (`is_readonly=0`, `absolute_delay=0`, no exceptions).

:::note
Under live ingestion, use `SYSTEM SYNC REPLICA db.table LIGHTWEIGHT` (waits only
for fetch/attach entries), or skip it entirely and trust the snapshot:

```sql
SELECT hostName(), database, table, is_readonly, zookeeper_exception
FROM clusterAllReplicas(my_cluster, system.replicas)
WHERE is_readonly = 1 OR zookeeper_exception != '';   -- 0 rows == healthy
```

Health is a *snapshot*, not a command's exit code.
:::

One more small one that costs 24.8 users a night: **`<storage_path>` is
mandatory on Keeper 24.8.** Omit it and 24.8.x quietly falls back to
`/var/lib/clickhouse-keeper` and dies with `Permission denied` under the
`clickhouse` user, even when your log/snapshot paths are perfect.

## Choosing, in practice

Here's the decision tree the three migrations actually followed:

- **Cross-cloud dev cluster** → Method 2. WAN quorum during a combined ensemble
  is a non-starter, and the target ran a newer Keeper anyway.
- **Production, old source build (23.5.2.7)** → Method 2, *after* Method 1
  segfaulted it. Version mismatch + fragile source = rebuild, don't merge.
- **Production, same DC, matching modern versions** → Method 1. No write freeze,
  clean live membership hand-off. This is the one case where the elegant method
  earned its keep.

## Takeaways

- **Never `zkcopy` ClickHouse Keeper.** It breaks sequential counters and
  silently diverges replicas. Preserve (Method 1) or rebuild (Method 2) — never
  copy.
- **Method 1 (`reconfig`)** is the zero-freeze, live-membership path. It's
  gorgeous *only* when source/target share a datacenter and compatible, modern
  (≥ 23.3, target ≥ source) versions.
- **Method 2 (`SYSTEM RESTORE REPLICA`)** is the safe default: version-agnostic,
  cross-cloud-safe, source untouched, trivial rollback. The price is a short
  write freeze.
- **`enable_reconfiguration` can segfault an old source build** via a bad ZXID
  rollback. Test on a non-prod copy of the exact build, or don't use Method 1.
- **A "clean" ensemble isn't clean until you wipe the on-disk Raft state** — log,
  snapshots, *and* the state dir. Stale state makes five nodes look like eight.
- **`ON CLUSTER` dies the moment you point at an empty Keeper.** Freeze while the
  old Keeper is still wired, or go node-local.
- **`SYSTEM SYNC REPLICA` timeouts under ingestion are false alarms.** Use
  `LIGHTWEIGHT` or read `system.replicas`.
- When `keeper-client` lacks `reconfig`, **`kazoo` is your friend.**

The recurring theme, same as every bare-metal ClickHouse job: the engine will
happily let you corrupt yourself, so all the safety lives in the sequencing and
the verification between steps. Migrate one environment at a time, keep the old
ensemble as a backstop through a 24–72h soak, and never trust a command's exit
code over a `system.replicas` snapshot.
