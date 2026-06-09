---
title: "The KILL That Could Not Kill: A ClickHouse Distributed Query Cancellation Deadlock"
published: 2026-06-09
description: "A cluster full of distributed SELECTs that won't die, KILL QUERY itself wedged, and the ON CLUSTER DDL queue frozen. The root cause is one blocking epoll wait made while holding the executor mutex — an unfixed, upstream-open ClickHouse bug. A frame-by-frame trace from system.stack_trace down to the source line."
image: ''
tags: [ClickHouse, Big Data Engineering, Concurrency, Debugging, Distributed Systems, C++]
category: 'Coding'
draft: false
lang: 'en'
slug: clickhouse-distributed-query-cancellation-deadlock
---

## The symptom

A real-time alerting cluster — 12 shards × 2 replicas, ClickHouse 25.8 — accumulated a pile of distributed `SELECT`s that would not die. Every one was a fan-out over `*_local` tables behind a distributed table, every one carried `max_execution_time = 20`, and every one had been running for **thousands** of seconds. The timeout had clearly fired — and done nothing.

So we did the obvious thing and tried to kill them:

```sql
KILL QUERY WHERE query_id IN (...) ASYNC;
```

The `KILL` **also hung**. Then a plain `KILL QUERY ... ON CLUSTER` hung. Then a local `KILL` on a single node hung — `elapsed` ticking past 2,190 seconds on the kill itself. We now had three classes of stuck statement: the original SELECTs, every KILL aimed at them, and the distributed DDL queue that the `ON CLUSTER` kill rode in on.

The instinct is "a replica is down, queries are blocked on network." That was wrong — the nodes were healthy and serving other traffic fine. This post is the trace from that wrong instinct down to the actual source line, plus what the internet does and doesn't know about it. The short version: this is a **cancellation deadlock**. `KILL` is structurally incapable of clearing it, because `KILL` takes the exact lock that the wedged thread is holding. And as of 25.8 it is an **open, unfixed upstream bug**.

This is the same cluster family as [Debugging ClickHouse CANNOT_SCHEDULE_TASK](./debugging-clickhouse-cannot-schedule-task/); different failure mode entirely.

## First: confirm it's a lock, not a network wait

Before theorizing, get the stack traces. ClickHouse exposes live thread stacks through `system.stack_trace` (needs `allow_introspection_functions = 1`):

```sql
SET allow_introspection_functions = 1;

SELECT thread_id, query_id,
       arrayStringConcat(arrayMap(
         (s) -> demangle(addressToSymbol(s)), trace), '\n') AS stack
FROM system.stack_trace
WHERE query_id != ''
ORDER BY query_id;
```

The output sorted into **three** distinct shapes. Telling them apart is the whole investigation.

**Shape 1 — the lock owner (one per wedged query).** Stuck in `epoll`, *inside the cancellation path*:

```text
DB::Epoll::getManyReady(int, epoll_event*, int) const
DB::RemoteQueryExecutorReadContext::checkTimeout(bool)
DB::RemoteQueryExecutorReadContext::cancelBefore()
DB::AsyncTaskExecutor::cancel()
DB::RemoteQueryExecutor::tryCancel(char const*)
DB::RemoteQueryExecutor::cancel()
DB::RemoteSource::onCancel()
DB::ExecutingGraph::cancel(bool)          <-- holds processors_mutex
DB::PipelineExecutor::executeStepImpl(...)
```

**Shape 2 — the waiters (everything else trying to cancel).** Parked on a mutex:

```text
__lll_lock_wait
__GI___pthread_mutex_lock
std::__1::mutex::lock()
DB::ExecutingGraph::cancel(bool)          <-- blocked acquiring processors_mutex
DB::QueryStatus::cancelQuery(...)
DB::ProcessList::sendCancelToQuery(...)
DB::InterpreterKillQueryQuery::execute()
... (for ON CLUSTER) DB::DDLWorker::tryExecuteQuery / runMainThread
```

**Shape 3 — benign parked workers.** Idle pipeline/format threads in `pthread_cond_wait` → `PipelineExecutor::execute` or `ParallelFormattingOutputFormat::collectorThreadFunction`. These are **not** deadlocked; they're just waiting for work from the thread stuck in Shape 1. Red herrings — but you have to recognize them as such, or you'll chase ten "stuck" threads that are merely idle.

That `__lll_lock_wait` in Shape 2 is the proof it's a lock, not the network. Every `KILL`, the query's own handler, and the `DDLWorker` are all blocked on a `std::mutex`. The question is just: *which mutex, and who's holding it?* Shape 1 says: the thread that went into `ExecutingGraph::cancel` and never came back out of `epoll`.

:::note
Each wedged query has its **own** owner thread and its **own** executor mutex. This is N independent instances of one bug, not a single global lock freezing the server — which is exactly why the rest of the cluster kept serving traffic normally and only these specific query objects were stuck.
:::

## Tracing it to the source line

The checkout is `v25.8.11.1-lts`. Let's walk the chain top to bottom and watch the locks stack up.

**Step 1 — `ExecutingGraph::cancel()` takes `processors_mutex` and holds it across the whole loop.**

```cpp
// src/Processors/Executors/ExecutingGraph.cpp:404
void ExecutingGraph::cancel(bool cancel_all_processors)
{
    {
        std::lock_guard guard(processors_mutex);          // line 409  ← THE contended lock
        uint64_t num_processors = processors->size();
        for (uint64_t proc = 0; proc < num_processors; ++proc)
        {
            ...
            IProcessor * processor = processors->at(proc).get();
            processor->cancel();                          // line 421  ← descends into RemoteSource
        }
    }
}
```

This is the mutex in both Shape 1 (held) and Shape 2 (waited on). Everything funnels through `processor->cancel()` while `processors_mutex` is held.

**Step 2 — `RemoteQueryExecutor::cancel()` takes `was_cancelled_mutex`.**

```cpp
// src/QueryPipeline/RemoteQueryExecutor.cpp:850
void RemoteQueryExecutor::cancel() {
    LockAndBlocker guard(was_cancelled_mutex);   // 852
    cancelUnlocked();                            // 856 → tryCancel() (940) → read_context->cancel() (948)
}
```

**Step 3 — `AsyncTaskExecutor::cancel()` takes `fiber_lock`, then calls `cancelBefore()`.**

```cpp
// src/Common/AsyncTaskExecutor.cpp:48
void AsyncTaskExecutor::cancel() {
    std::lock_guard guard(fiber_lock);           // 50
    is_cancelled = true;
    {
        SCOPE_EXIT({ destroyFiber(); });         // 53  ← the famous fix; see below
        cancelBefore();                          // 54  ← never returns
    }
    cancelAfter();
}
```

**Step 4 — `cancelBefore()` blocks forever draining the in-flight packet.**

```cpp
// src/QueryPipeline/RemoteQueryExecutorReadContext.cpp:126
void RemoteQueryExecutorReadContext::cancelBefore()
{
    if (!is_timer_alarmed) {
        ...
        /// Wait for current pending packet, to avoid leaving connection in unsynchronised state.
        while (is_in_progress.load(std::memory_order_relaxed)) {   // 139
            checkTimeout(/* blocking= */ true);                    // 141  ← blocks here
            resumeUnlocked();
        }
    }
    ...
}
```

and `checkTimeout(true)` is a wait with **no timeout**:

```cpp
// src/QueryPipeline/RemoteQueryExecutorReadContext.cpp:98
size_t num_events = epoll.getManyReady(3, events, blocking ? -1 : 0);
//                                                          ^^ -1 = wait indefinitely
```

There it is. The Shape-1 thread is parked in `epoll.getManyReady(..., -1)` **while holding all three locks**: `processors_mutex`, `was_cancelled_mutex`, and `fiber_lock`. Every other cancellation — each `KILL`, the query's own `PullingAsyncPipelineExecutor::cancel`, the `DDLWorker` — slams into `processors_mutex` at `ExecutingGraph.cpp:409` and joins the `__lll_lock_wait` pile. They will never acquire it, because the owner is blocked on network I/O that may never arrive.

## Why it blocks "forever"

`cancelBefore()` does a deliberate, **blocking** drain — its own comment says *"Wait for current pending packet, to avoid leaving connection in unsynchronised state."* The idea is sound: if you abandon a half-read packet, the TCP stream is desynchronized and the connection can't be reused. So on cancel it tries to finish reading the in-flight packet first.

The flaw is *where* it does this. The drain runs:

- with **no timeout** on the epoll wait (`-1`), bounded only by the socket receive timeout, which can be very long; and
- while **`processors_mutex` is held**.

On a healthy-but-saturated replica that simply hasn't flushed the in-flight packet yet, that wait can stretch for thousands of seconds — and for the entire time, the node's cancellation machinery is frozen behind one lock. A slow socket is silently promoted into a **node-wide cancellation deadlock**. It's a design flaw — blocking network I/O under an executor mutex — not a leak or a corruption.

This also explains the cascade we saw: the original SELECT's `max_execution_time=20` fired the cancel path → owner thread wedged in the drain holding `processors_mutex` → the query's own handler thread blocked → our `KILL` blocked → the `ON CLUSTER` `KILL` blocked → the `DDLWorker` running it blocked → the whole distributed DDL queue stalled. One blocking `epoll` call took the entire cancellation subsystem hostage.

## What the internet knows (and a fix that looks right but isn't yours)

Searching the obvious frames turns up a well-known fix that is *adjacent* to this and frequently misidentified as the cure:

::github{repo="ClickHouse/ClickHouse"}

**PR #55516 — "Destroy fiber in case of exception in cancelBefore in AsyncTaskExecutor"** (closes issue #55185). That's the `SCOPE_EXIT({ destroyFiber(); })` you saw at `AsyncTaskExecutor.cpp:53`. It shipped in **23.10** and fixes a *different* failure: if `cancelBefore()` **throws**, the boost fiber used to leak, taking its lock with it. The `SCOPE_EXIT` guarantees teardown on the way out.

Here's the trap: **that fix is already present in your 25.8 build**, and it does not help, because our thread never throws and never leaves `cancelBefore()`. It is stuck *inside* the blocking drain at line 141. `SCOPE_EXIT` only runs when the scope exits; an `epoll(-1)` that never returns never exits the scope. Confirming the fix is present is what rules it out — and stops you from "upgrading to get #55516" when you already have it.

Two more relatives, both also already in 25.8 and both *not* this bug:

- **PR #82160 / #82157** — a deadlock between `RemoteQueryExecutor::was_cancelled_mutex` and `process_list.mutex` under memory pressure (`OvercommitTracker` in the stack). Shipped 25.7. Our stack has no `OvercommitTracker`.
- **PR #41343 / #41467 / #47161** — the 2022–2023 wave of distributed-cancellation lock fixes (`async_socket_for_remote`/hedged + parallel `KILL`, lock-free cancellation, `QueryStatus` deadlock). Long since merged.

The one that actually matches — same `RemoteQueryExecutor::cancel → ExecutingGraph::cancel → epoll` signature — is:

**Issue #66351 — "Possible deadlock using clusterAllReplicas against system tables."** Reported on 24.5. **Still open. No fix PR. No fix version. Labeled only `potential bug`.** The drain code itself hasn't been meaningfully touched since — the most recent commits to `RemoteQueryExecutorReadContext.cpp` are refactors (FiberStack relocation, packet header/body split), not a change to the blocking wait.

:::caution
**Bottom line: this is an unfixed, upstream-known-but-open bug as of 25.8.11.** There is no newer release that fixes it. Anyone telling you to "just upgrade" is pattern-matching on #55516, which you already have. Don't burn a maintenance window on it.
:::

## The fix (mitigation, because there's no patch)

**1. Stop issuing `KILL`.** Every `KILL` just adds another waiter to the `processors_mutex` pile and freezes more of the DDL queue. It cannot win against a held lock.

**2. Restart `clickhouse-server` on the nodes holding owners**, one at a time so the replica keeps serving. `SIGTERM` first, then `SIGKILL` — and expect the graceful path to hang, because shutdown drains queries through the *same* cancel path. The locks die only when the process holding them dies. (Safe here: these were read-only SELECTs.)

**3. Prevent recurrence by avoiding the deadlocking code path entirely:**

```sql
-- in the profile used by these distributed queries
SET async_socket_for_remote = 0;   -- skips the RemoteQueryExecutorReadContext epoll path,
                                   -- so cancelBefore() no longer does the blocking drain
SET use_hedged_requests   = 0;     -- avoids the HedgedConnections variant of the same path (#66351)
```

`async_socket_for_remote = 0` is the load-bearing one: with the async read context gone, there is no `cancelBefore()` epoll drain to block in. The trade-off is losing the async remote-read optimization, which for short fan-out queries is negligible.

**Do not** rely on `max_execution_time` to clean these up — it is precisely what *fires* the broken path. And don't expect any quota or `KILL` mode to help; the bug is below all of them.

**4. Push the fix upstream.** Issue #66351 is open and stale, reported on 24.5 with thin stacks. A current `25.8` repro with the frame-by-frame `processors_mutex` analysis above is more than what's on the issue today. The real fix needs to either make the `cancelBefore()` drain **non-blocking** (bounded epoll timeout + bail) or move it **out from under `processors_mutex`** so a slow socket can't freeze node-wide cancellation.

## Takeaways

- **When `KILL` itself hangs, suspect the cancellation path, not the query.** A `KILL` that won't return is a lock-ordering symptom: `KILL` calls `ExecutingGraph::cancel`, which is exactly the function the wedged thread is stuck inside.
- **`system.stack_trace` sorts the problem into owners, waiters, and red herrings.** One thread in `epoll` inside `…::cancel` holding the mutex; everyone else in `__lll_lock_wait` on it; idle `pthread_cond_wait` workers that look stuck but aren't. Classify before you act.
- **Blocking network I/O under a lock is the whole bug.** `cancelBefore()` drains the in-flight packet with an untimed `epoll` wait while holding `processors_mutex`. A slow-but-healthy replica becomes a node-wide cancellation deadlock.
- **A present fix can be the thing that rules itself out.** `SCOPE_EXIT`/PR #55516 is right there in `AsyncTaskExecutor::cancel` — and confirming it's present is what proves the bug is elsewhere. Don't "upgrade to get" a fix you already shipped.
- **Match the stack, not the keywords.** Three different merged PRs share frames with this deadlock; none of them are it. The actual match — issue #66351 — is still open. "There's a fix" and "your bug is fixed" are different claims.
- **The cure is to not enter the path.** With no upstream patch, `async_socket_for_remote = 0` (plus `use_hedged_requests = 0`) sidesteps the deadlocking drain; restart-one-at-a-time clears the already-wedged nodes.
