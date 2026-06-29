---
title: "Six Releases in Twelve Days: How I Keep Enhancing the ClickHouse Debug Skill"
published: 2026-06-29
description: "Packaging my ClickHouse debugging method into an Agent Skill was the start, not the finish. In twelve days it went from v0.1.0 to v0.6.0 — each release driven by a real incident or a real friction, kept honest by a fixture-replay eval harness, and the latest version built end-to-end with an AI pipeline of its own. Here's the loop: incidents feed the skill, evals keep it from regressing, and AI builds the next layer behind guardrails."
image: ''
tags: [ClickHouse, Debugging, AI, Agent Skills, Evals, Observability]
category: 'Coding'
draft: false
lang: 'en'
slug: evolving-clickhouse-debug-skill
---

## Shipping the skill was the easy part

A couple of weeks ago I [packaged my ClickHouse debugging method into an Agent Skill](/posts/clickhouse-debug-agent-skill/): triage from the outside with Prometheus, drill into `system.*` over a capped read-only loop, confirm the root cause against a version-matched source tree, and route the fix to ClickHouse's official best-practices skill. That post ended where most "I built a tool" posts end — with installation instructions.

But a skill isn't a blog post you publish once. It's closer to a piece of production software that happens to encode judgment. The day after I shipped `v0.1.0`, the next real incident found a gap. So did the one after that. Twelve days later the skill is at **`v0.6.0`** — six feature releases — and the more interesting story isn't any single feature. It's the *loop* that produced them:

1. A real incident or a real friction exposes a gap.
2. The gap becomes a rule, a probe, or a guardrail in the skill.
3. An eval harness makes sure the change actually helps and nothing regresses.
4. Increasingly, AI does the building — behind guardrails strict enough that I trust it to.

This post is about that loop.

## Every release is a scar

If you read the changelog as a narrative, it's a list of things that went wrong in production and what I did so they'd never cost me an hour again.

- **`v0.2.0` — the fleet hardening.** The first time I pointed the skill at a real large-fleet, chproxy-fronted cluster, it broke in unglamorous ways: a `clusterAllReplicas` scan multiplies rows by node count and blew past the per-query row cap, exported metrics came back as silent `NA`s, a transient TLS reset read as "the endpoint is down," and — the dumbest one — shell `export`s didn't survive between an agent's separate Bash calls, so every probe lost its connection config. None of that is a *diagnosis* problem. It's all friction that stops the agent before it can think. So `v0.2.0` was about making the skill survive contact with a 60-node fleet: fan-out-aware caps, transient-retry on the probes, and writing connection config to a sourced file instead of trusting `export`.

- **`v0.3.0` — the cross-validation rule.** A Kafka-ingest investigation cost me real time because I trusted one number too long: an external Prometheus consumer-offset gauge looked authoritative, but it faked plateaus and overstated throughput ~2×, disagreeing with `part_log` rows-written by 2–6×. The lesson became a *rule* in the skill: Outside and Inside aren't a relay where each view is authoritative in turn — they're two instruments that must **agree** before you trust a rate. When they disagree, resolve to ground truth in a fixed order: `part_log` rows actually written first, an internal `system.*` counter second (and verify it even exists — there's no Kafka poll-time counter, and a guessed metric name reads exactly like zero), the external gauge last.

- **`v0.4.0` — the Keeper playbook.** A transfer test surfaced the one domain the skill couldn't walk end-to-end: a Keeper/ZooKeeper restart that left replicas stuck read-only. That became a whole cross-cutting reference file — `zookeeper_connection`, `replication_queue`, `distributed_ddl_queue`, the source mechanism for read-only, and the operator-side recovery ladder.

The pattern: the skill doesn't get smarter from me sitting down to brainstorm features. It gets smarter every time a real cluster embarrasses it. That's the same reason [I read source code instead of guessing](/posts/reading-source-code-in-the-ai-era/) — the truth is out there in the system, not in my memory of how I think it works.

## The unlock: making "did this actually help?" measurable

By `v0.4.0` I had a problem familiar to anyone who maintains a prompt or a skill: **I couldn't change it with confidence.** Every edit to a 500-line playbook is a bet that I improved the agent's behavior without quietly breaking a path that used to work. With prose, you can't run a test suite. You just hope.

`v0.5.0` fixed that with a **fixture-replay eval harness**, and it's the release that made everything after it possible.

The trick is small. The two probe scripts (`chq.sh` for ClickHouse, `promq.sh` for Prometheus) gained two env-gated modes. `CH_CAPTURE_DIR` records a probe's real output to a fixture file, keyed on a normalized hash of the query. `CH_REPLAY_DIR` returns that fixture instead of hitting the network. Both are completely inert when unset, so normal debugging is untouched.

That gives you a way to freeze a real incident — the exact Prometheus series and `system.*` rows from, say, a `CANNOT_SCHEDULE_TASK` storm — into a scenario folder, then replay it offline against the skill as many times as you want. A scenario is just a prompt, a folder of fixtures, and a rubric. Run it through the agent in replay mode, score the transcript against the rubric, and now "I improved the skill" is a claim with evidence behind it instead of a vibe. The seed scenario, a range-JOIN OOM, is proven to *pass* a correct diagnosis and *fail* a neutered one — so the eval can actually tell good from bad.

One hard rule came with it: those fixtures are captured from real clusters, so they carry real hostnames and tenant names. They never enter the public repo. The harness, the scenarios, and *sanitized* fixtures are committed; raw captures stay local and `.gitignore`d.

Once you can measure a skill, you can let something else edit it.

## v0.6.0: enhancing the skill with an AI pipeline

The latest release is where it gets recursive. `v0.6.0` added three things:

- **`preflight.sh`** — a step-0 readiness check that replaces a paragraph of manual prose. It confirms you're in a source tree, automatically diffs the source version against the live server's `version()` (the version-match step that the whole "confirm against source" method depends on), checks that Prometheus and ClickHouse actually answer, and reports cluster topology so fan-out caps can be sized. It runs every cluster read *through* `chq.sh`, so it inherits the caps for free.
- **Executable routing** — the symptom-to-specialist table used to live only as prose the agent recited from memory. Now it's a `routing.tsv` data file plus a `route.sh` lookup: paste an error code, get the reference playbook and the companion specialist to use. A test asserts the table and the data file can never drift apart.
- **A `curl`-guard hook** — the skill's number-one safety rule has always been "never hand-roll a raw `curl` to the cluster; go through the capped wrapper." `v0.6.0` makes that *enforced* for Claude Code plugin users: a `PreToolUse` hook blocks a raw, query-bearing `curl` to a ClickHouse port before it can run, and tells the agent to use `chq.sh` instead.

But the features matter less than how they were built. I didn't hand-write `v0.6.0`. I drove it through an AI pipeline, end to end, and just steered:

- **Brainstorm → spec.** A structured back-and-forth that turned "what could we enhance?" into a written, committed design doc — including the one decision that shaped everything: hooks only work for Claude Code plugin installs, not the cross-agent `npx` path, so the design is *layered*. A portable core (scripts that run everywhere) plus a plugin-only hook as a bonus enforcement layer. The "auto-detect which companion skills are installed" idea got sharpened too — that detection belongs to the *model* (it already has its skill list in context), not a brittle filesystem scan.
- **Spec → plan.** The design became a task-by-task implementation plan with the actual code in each step.
- **Plan → build.** A fresh sub-agent implemented each task with tests; a second sub-agent reviewed each one for spec compliance and quality before it counted as done; a final adversarial review read the whole branch.
- **Ship.** Push, PR, squash-merge, tag, release.

The whole thing — diagnosis, design, build, review — is now AI-driven, with me making the taste calls.

## The guardrails are what make fast safe

Letting an AI pipeline build your debugging tool sounds reckless until you look at what catches mistakes. Speed only works because each layer has a net under it:

- **The drift guard.** The routing data file and the human-readable table in the docs must name the exact same set of specialists; a unit test fails the build if they diverge. Two representations, one source of truth, enforced.
- **Fail-open by design.** The `curl`-guard blocks *only* the one precise dangerous case — a query-bearing `curl` to a ClickHouse port that skips the wrapper. A health-check `ping` passes. Anything ambiguous passes. A guard that over-blocks would just get disabled; a guard that blocks the one thing that once OOM-killed a node earns its place.
- **Eval replay.** `preflight.sh` and the routers are exercised offline against fixtures, so their behavior is pinned without a live cluster.
- **Adversarial review.** The final whole-branch review ran on the most capable model and was told to *try to break* the work. It earned its keep in an unexpected way: one of the per-task reviewers had hallucinated — it "found" problems in files that didn't exist. Because the process treats every review as an unverified claim to check against the actual diff, the hallucination got caught and the real diff was verified by hand and by the test suite before anything merged. The guardrail that mattered most wasn't catching a bug in the code. It was catching a bug in the *reviewer*.

That last point is the whole philosophy in miniature. The same instinct that runs through the skill itself — [don't trust one view, confirm against ground truth](/posts/clickhouse-debug-agent-skill/) — is the instinct that makes it safe to build the skill with AI. You don't trust the agent's report. You trust the diff, the test, and the source.

## The compounding loop

Step back and the twelve days have a shape:

- **Incidents feed the skill.** Every release from `v0.2.0` to `v0.4.0` is a lesson from a cluster that misbehaved, frozen into a rule so it never costs an hour twice.
- **Evals keep it honest.** `v0.5.0` turned "I think this is better" into "this scenario passes and that one fails," which is the precondition for changing a skill quickly without fear.
- **AI builds the next layer.** `v0.6.0` was designed, implemented, reviewed, and shipped through an AI pipeline, behind guardrails — a drift guard, a fail-open hook, eval replay, adversarial review — strict enough to make the speed trustworthy.

A blog post is something I write once and you read once. A skill is something an agent applies at 3 a.m. on the fifth cluster this week — and, increasingly, something an agent *improves* between those incidents. The method stopped being tribal knowledge in my head, became executable, and is now on a flywheel: each real failure makes it sharper, each eval keeps it from sliding back, and each release is a little more built by the same kind of system it's meant to debug.

The repo is open source (Apache-2.0) at **[github.com/Kryst4lskyxx/clickhouse-debug-skills](https://github.com/Kryst4lskyxx/clickhouse-debug-skills)** — `v0.6.0` is live, and the next scar is probably already on its way.
