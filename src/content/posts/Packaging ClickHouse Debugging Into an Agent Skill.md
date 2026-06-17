---
title: "From War Stories to a Skill: Packaging ClickHouse Cluster Debugging for AI Agents"
published: 2026-06-17
description: "Every ClickHouse incident I've written up here followed the same shape: triage from the outside with Prometheus, drill in with read-only system.* queries, then confirm the root cause against the matched source tree. I turned that methodology into an installable Agent Skill — clickhouse-debug — so an AI agent can run the whole funnel without me, and without ever being able to OOM the node it's debugging. Here's what's in it, how it works, and how to install it."
image: ''
tags: [ClickHouse, Debugging, AI, Agent Skills, Prometheus, Observability]
category: 'Coding'
draft: false
lang: 'en'
slug: clickhouse-debug-agent-skill
---

## The pattern hiding in every incident

If you read back through the ClickHouse debugging posts on this blog, they look like different stories — [a node that "went down" but never rebooted](/posts/clickhouse-oom-debugging-up-metric/), [a `CANNOT_SCHEDULE_TASK` error storm](/posts/debugging-clickhouse-cannot-schedule-task/), [a node running 10× hotter on iowait because of a hidden SATA disk](/posts/clickhouse-iowait-sata-nvme-disk-tier/), [a live event that tripped three failures at once](/posts/live-event-three-compounding-clickhouse-failures/). Different symptoms, different root causes.

But the *method* was identical every single time:

1. **Start from the outside.** Prometheus is the only thing I can always reach. Is the host actually down, or did one exporter stop answering? OOM, or reboot? CPU, iowait, disk, network — localize *where* and *when* before touching the database.
2. **Drill into the inside.** Once Prometheus narrows it down, the cluster's own `system.*` tables tell you *what it was doing* and *who asked* — `errors`, `query_log`, `parts`, `merges`, `replicas`, thread-pool metrics.
3. **Confirm against the source.** Error codes, metric semantics, and throw sites are version-specific. The last step is always to stand in a ClickHouse source checkout *at the cluster's version* and confirm the mechanism — [reading the source removes the guesswork](/posts/reading-source-code-in-the-ai-era/).
4. **Route the fix.** Diagnosis and remedy are different jobs. Once I know it's an unbatched-insert part explosion, the fix is a known best practice — I shouldn't re-derive it from scratch.

That's a *procedure*. And anything that's a stable procedure is something an AI agent can run — if you hand it the funnel, the guardrails, and the institutional memory of what each signature means. So I packaged it as an [Agent Skill](https://agentskills.io): **`clickhouse-debug`**.

## What an Agent Skill actually is

A skill is a folder an agent loads on demand. It's progressive: the agent always sees a short description (so it knows *when* to reach for the skill), pulls the main `SKILL.md` into context when the skill triggers, and only opens the deeper reference files when it needs them. That keeps the agent's context lean while still giving it a 500-line playbook the moment a ClickHouse incident shows up.

`clickhouse-debug` is laid out like this:

```
skills/clickhouse-debug/
├── SKILL.md                      # the triage funnel + routing
├── metadata.json                 # version, abstract, references
├── references/
│   ├── cluster-state.md          # the Prometheus playbook (outside view)
│   └── query-state.md            # the system.* playbook (inside view)
└── scripts/
    ├── chq.sh                    # read-only ClickHouse HTTP query, capped
    └── promq.sh                  # Prometheus query helper
```

The description is deliberately *pushy* about when to trigger, because the failure mode of skills is under-triggering — not firing when they'd help. It fires whenever someone is investigating a running cluster: nodes down or flapping or OOM-killed, pods crash-looping, `TOO_MANY_PARTS`, `CANNOT_SCHEDULE_TASK`, `MEMORY_LIMIT_EXCEEDED`, `FILE_DOESNT_EXIST`, replication lag, thread-pool or FD exhaustion — even when the user just pastes an error code or a node name and asks "what's wrong." It explicitly *doesn't* fire for writing application SQL or designing schemas; those are different skills.

## The one rule that makes it safe to point at production

Here's the thing that kept me up about automating cluster debugging: **a debugging probe must never become the incident.** It's trivially easy to ask `system.query_log` an innocent-looking question that scans hundreds of GB and pushes an already-stressed node over its memory limit. The cure can't be worse than the disease.

So every query the skill issues goes through `chq.sh`, which bakes resource caps into *every* request — they're not optional, and the agent can't forget them:

```bash
curl -sk "$CH_URL/" \
  --data-urlencode "max_memory_usage=${CH_MAX_MEM}" \        # 20 GiB per-query
  --data-urlencode "max_execution_time=${CH_MAX_TIME}" \     # 30 s wall clock
  --data-urlencode "max_estimated_execution_time=60" \       # reject before it runs
  --data-urlencode "max_rows_to_read=${CH_MAX_ROWS}" \       # the real guardrail
  --data-urlencode "max_bytes_to_read=${CH_MAX_BYTES}" \
  --data-urlencode "max_result_rows=100000" \
  --data-urlencode "result_overflow_mode=break" \
  --data-urlencode "max_threads=4" \
  --data-urlencode "readonly=1" \
  --data-urlencode "query=${sql}"
```

A probe that would exceed these aborts with `MEMORY_LIMIT_EXCEEDED` / `TIMEOUT_EXCEEDED` instead of taking down the node. `readonly=1` means the agent *cannot* mutate the cluster even if it tried. The work is read-only by construction; the only way a read can still hurt is by consuming too much, and the caps close that door. (These mirror the official `agent-query-safety` rule from ClickHouse's own skills — more on that below.)

Why HTTP and not `clickhouse-client`? From a laptop or VPN the native client often can't resolve its own hostname or reach the native port; HTTP on 8123/8443 is reliable and works cleanly with a read-only user.

## The playbooks: institutional memory, not just queries

The two reference files are where the war stories live. They're not generic "here's how to query `system.parts`" docs — they encode the *judgment* that took real incidents to learn. A few examples of the kind of thing baked in:

- **`up < 1` doesn't mean the host is down.** It means one scrape target failed to answer. Check `node_boot_time_seconds` first — if the box has 152 days of uptime, you've killed the entire "hardware/reboot/panic" branch of hypotheses for the cost of one query.
- **A failing-merge retry storm looks like a part pile-up but isn't.** A partition pinned at exactly `parts_to_throw_insert` for hours, with merge launches high but `MergesTimeMilliseconds ≈ 0` and `part_log` showing the same merge erroring instantly on a missing `.cmrk2`/`.bin`, is usually a **dead disk underneath**. The skill says, in so many words: don't `DROP PARTITION` to "fix" it — that's a band-aid that won't hold while the disk is broken.
- **`CANNOT_SCHEDULE_TASK` with `GlobalThread` far below its limit is not pool saturation.** It's the OS transiently refusing `clone()` under a one-second admission stampede. The lever is `max_concurrent_queries`, *not* `max_thread_pool_size` — and the skill points at `src/Common/ThreadPool.cpp` to confirm the throw site in your version.
- **Query attribution.** `address=::1` and a plain-UUID `query_id` is a human in a local shell; `client_name='ClickHouse server'` is routine shard fan-out, not an attacker. The skill reads these fields together so it doesn't blame the wrong query.

This is the part you can't get from documentation — it's the difference between an agent that runs queries and one that *reaches the right conclusion in the right order.*

## How we use it

The workflow is the same one I follow by hand, just driven by the agent.

**1. Check out ClickHouse at the cluster's version** and `cd` into it. The source tree is the skill's confirmation oracle, so the version has to match.

**2. Point the skill at your telemetry** with a few environment variables:

```bash
export PROM='https://prometheus.example.com'      # Prometheus base URL
export CH_URL='http://node-1.example.com:8123'     # or https://…:8443
export CH_USER='readonly_user'
export CH_PASS='…'                                 # optional
```

**3. Describe the problem.** That's it. For example:

> Our cluster is throwing Code 439 `CANNOT_SCHEDULE_TASK` 'failed to start the thread' in per-minute bursts, ~2400 a minute. Prometheus and a read-only HTTP user are set up, it's on k8s. What's the root cause and how do I fix it?

The skill gathers what it still needs (it'll *ask* whether it's bare metal or k8s if you didn't say), then runs the funnel: it pulls the admission-rate signature from Prometheus, confirms `GlobalThread` is nowhere near its ceiling, checks pod restart counts to rule out a crash loop, opens `src/Common/ThreadPool.cpp` to confirm the throw comes from a failed `std::thread` constructor, and lands on "lower `max_concurrent_queries`, and no, raising `max_thread_pool_size` won't help." Every probe along the way is time-scoped to the failure window and capped.

The output is a root-cause writeup: what happened, the evidence chain, the mechanism confirmed against source, and the fix.

### It composes with the official ClickHouse skills

Diagnosis is this skill's job; the *remedy* canon belongs to ClickHouse's own [`clickhouse-best-practices`](https://github.com/ClickHouse/agent-skills) skill. So `clickhouse-debug` deliberately doesn't restate fix guidance — when the diagnosis is, say, a genuine over-insertion part explosion, it routes to the relevant best-practices rules (`insert-batch-size`, `insert-async-small-batches`, the `schema-partition-*` family) and cites them. Install both and they hand off cleanly: one tells you *what broke and why*, the other tells you *how to do it right*. If the companion skill is missing, `clickhouse-debug` tells you to install it before recommending fixes.

## How to install

The skill ships through two channels.

**Via `npx skills`** (works with Claude Code, Cursor, Copilot, and other agents):

```bash
npx skills add Kryst4lskyxx/clickhouse-debug-skills
```

The CLI auto-detects your installed agents and asks where to put it.

**Via the Claude Code plugin marketplace:**

```text
/plugin marketplace add Kryst4lskyxx/clickhouse-debug-skills
/plugin install clickhouse-debug@clickhouse-debug-skills
```

Then add the official companion skill so the fix-routing works:

```bash
npx skills add clickhouse/agent-skills
```

The repo is open source (Apache-2.0) at **[github.com/Kryst4lskyxx/clickhouse-debug-skills](https://github.com/Kryst4lskyxx/clickhouse-debug-skills)** — issues and PRs welcome, especially new incident signatures. One ground rule for contributors: never commit real cluster telemetry. The eval fixtures that drive the skill's testing contain real hostnames and cluster names, so they stay local and `.gitignore`d; everything public uses placeholders like the `node-1.example.com` you see above.

## Why bother packaging it at all

I could keep these methods in my head and in blog posts. But a blog post is something *I* read and then apply. A skill is something an *agent* applies — at 3 a.m., on the fifth cluster this week, without me. The institutional memory of "check boot time first" or "a pinned partition with zero merge time is a dead disk, not a part problem" stops being tribal knowledge and becomes executable.

The interesting shift is that the source tree is no longer just documentation for humans — it's the agent's ground truth. Pair a version-matched checkout with a capped read-only loop and a playbook of hard-won signatures, and you get something that debugs the way a careful engineer does: outside-in, evidence first, conclusions confirmed against the code, and never, ever knocking over the patient to take its pulse.
