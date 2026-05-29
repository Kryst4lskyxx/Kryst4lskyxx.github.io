---
title: ClickHouse on K8s — What Triggers a Rolling Restart
published: 2026-05-29
description: 'A source-grounded tour of the clickhouse-operator restart policy: which config changes roll your pods, which apply live, and how to control it.'
image: ''
tags: [ClickHouse, Kubernetes, clickhouse-operator, Big Data Engineering, DevOps]
category: 'Coding'
draft: false 
lang: 'en'
---

## Why this matters

When you run ClickHouse on Kubernetes via the [Altinity clickhouse-operator](https://github.com/Altinity/clickhouse-operator), every edit to your `ClickHouseInstallation` (CHI) goes through a reconcile loop. Some edits are applied **live** — the operator rewrites a ConfigMap and ClickHouse reloads it in place. Others force a **rolling restart** — each pod is shut down and recreated, one at a time.

Knowing which is which is the difference between a free config tweak and an unintended cluster-wide bounce. This post maps the exact rules, grounded in the operator source (paths from a recent `master`).

## Two independent restart mechanisms

A pod gets restarted by **either** of two paths:

1. **StatefulSet pod-template change** — the operator regenerates the StatefulSet; if the *pod spec* differs, Kubernetes' `RollingUpdate` strategy recreates the pods. (`pkg/model/common/creator/stateful-set.go` sets `RollingUpdateStatefulSetStrategyType`.)
2. **Force restart** — the operator itself decides a restart is required and issues a SQL shutdown so the pod is recreated. This is `shouldForceRestartHost()` in `pkg/controller/chi/worker.go`.

Config changes that *don't* require a reboot go through neither: the ConfigMap is updated and ClickHouse picks the change up on its own.

## Mechanism 1: pod-template changes (always restart)

Anything that changes the **pod template** of the StatefulSet rolls the pods, because Kubernetes restarts pods whenever their spec changes. This includes:

- **Container image / ClickHouse version** (`templates.podTemplates...image`)
- **CPU / memory `resources`** (requests or limits)
- **Environment variables**, command, args
- **Volume mounts / `volumeClaimTemplates`** topology, `podTemplate` changes
- **Affinity, tolerations, nodeSelector, securityContext, serviceAccount**
- Labels/annotations that are part of the pod template

If you change the `cpu` limit on a pod (relevant if you're tuning concurrency — see my [concurrency-control post](/posts/diagnosing-clickhouse-concurrency-control/)), that's a pod-template change and the cluster *will* roll.

## Mechanism 2: config changes — the reboot decision engine

This is the interesting part. The entry point is `IsConfigurationChangeRequiresReboot(host)` in `pkg/model/chop_config.go`. It diffs the previous (`ancestor`) and desired configuration across these sections and asks, per changed path, "does this require a reboot?":

- **Zookeeper** — *any* change requires reboot (`isZookeeperChangeRequiresReboot` = "not equal").
- **Profiles** (global) — rule-based.
- **Quotas** (global) — rule-based.
- **Settings** (global + per-host local) — rule-based.
- **Files** (global + local) — rule-based, **with the `users` section filtered out**.

Note what is **absent**: there is no check for the **`users`** section at all. Changes to users — passwords, `networks`, `grants`, profile assignment — **never** trigger a reboot. They land in `users.xml`, which ClickHouse reloads live.

### How a rule decides

For each changed path (e.g. `settings/max_concurrent_queries`), the operator walks the configured rules, scoped by ClickHouse version, and the **last matching rule wins** (`getLatestConfigMatchValue`). The logic (`isListedChangeRequiresReboot`):

- A matching rule with value **`"yes"`** → reboot required.
- A matching rule with value **`"no"`** → this path does *not* force a reboot.
- **No matching rule at all** → does *not* force a reboot (default is safe/no-restart).

So a path reboots **only if its last matching rule is explicitly `"yes"`.**

### The default rules (`config/config.yaml`)

```yaml
configurationRestartPolicy:
  rules:
    - version: "*"
      rules:
        - settings/*: "yes"                 # ALL config.xml settings restart by default...

        # ...except these (runtime-reloadable):
        - settings/access_control_path: "no"
        - settings/dictionaries_config: "no"
        - settings/max_server_memory_*: "no"
        - settings/max_*_to_drop: "no"
        - settings/max_concurrent_queries: "no"
        - settings/models_config: "no"
        - settings/user_defined_executable_functions_config: "no"
        - settings/logger/*: "no"
        - settings/macros/*: "no"
        - settings/remote_servers/*: "no"
        - settings/user_directories/*: "no"
        - settings/display_secrets_in_show_and_select: "no"

        - zookeeper/*: "yes"

        - files/*.xml: "yes"                 # raw files restart by default...
        - files/config.d/*.xml: "yes"
        - files/config.d/*dict*.xml: "no"    # ...except dictionaries
        - files/config.d/*no_restart*: "no"  # ...and an explicit opt-out by filename

        # profiles do NOT restart by default — only these two patterns do:
        - profiles/default/background_*_pool_size: "yes"
        - profiles/default/max_*_for_server: "yes"
    - version: "21.*"
      rules:
        - settings/logger: "yes"
```

### Summary: triggers vs. no-ops

| Change | Restart? | Why |
|---|---|---|
| `settings/*` (most `config.xml` server settings) | ✅ Yes | `settings/*: "yes"` |
| `settings/max_concurrent_queries`, `max_server_memory_*`, `max_*_to_drop` | ❌ No | explicit `"no"` (runtime-reloadable) |
| `settings/macros/*`, `logger/*`, `remote_servers/*`, `user_directories/*`, `dictionaries_config` | ❌ No | explicit `"no"` |
| **Zookeeper** connection config | ✅ Yes | `zookeeper/*: "yes"` (and any inequality) |
| Raw **files** under `files/...` and `files/config.d/*.xml` | ✅ Yes | `files/*.xml: "yes"` |
| Files matching `*dict*` or `*no_restart*` | ❌ No | explicit `"no"` |
| **Profiles** (e.g. `max_threads`, query-cache, most profile settings) | ❌ No | no matching `"yes"` rule |
| `profiles/default/background_*_pool_size`, `profiles/default/max_*_for_server` | ✅ Yes | explicit `"yes"` |
| **Quotas** | ❌ No (default) | no matching `"yes"` rule |
| **Users**: password, `networks`, `grants`, profile assignment | ❌ No | users section not checked at all |
| Pod **image / version** | ✅ Yes | StatefulSet pod-template change |
| Pod **CPU/memory resources**, env, volumes, affinity | ✅ Yes | StatefulSet pod-template change |

## Mechanism 3: explicit and special cases

`shouldForceRestartHost()` also restarts in these situations (and *suppresses* restarts in others):

- **`spec.restart: "RollingUpdate"`** — a manual, operator-wide roll of every host (`IsRollingUpdate()`), regardless of whether anything changed. Set it, reconcile, then unset it.
- **Crashed pod with unknown version** — if a pod is in `CrashLoopBackOff` and its version is unknown (often a bad config it can't start with), the operator restarts it to recover.
- **Suppressed** — a host that is **stopped** (`IsStopped`) or in **troubleshoot** mode (`IsTroubleshoot`), or a **brand-new** host (no ancestor), is never "restarted" by this logic.

## Customizing the policy

The rules live in the **operator's** config (`clickhouse.configurationRestartPolicy.rules`), not the CHI. Two practical levers:

1. **Add a `"no"` rule** for a server setting you know is runtime-reloadable, scoped to a version if needed. The comment in `config.yaml` even points at the canonical source of truth:
   > to be replaced with `select * from system.server_settings where changeable_without_restart = 'No'`

2. **The `*no_restart*` filename trick.** Any raw file whose name matches `files/config.d/*no_restart*` is applied **without** a restart. So if you keep a setting in a raw XML file that ClickHouse *can* reload live, name it accordingly.

### A concrete tie-in

In my [concurrency-control investigation](/posts/diagnosing-clickhouse-concurrency-control/), the fix lived in `config.d/thread_limit.xml` (`concurrent_threads_soft_limit_ratio_to_cores`). That setting is **runtime-reloadable** in ClickHouse — but as a raw file it matches `files/config.d/*.xml: "yes"`, so editing it through the operator would **roll the whole cluster**. Two ways to avoid the bounce:

- Rename it to `config.d/thread_limit_no_restart.xml` so it matches `files/config.d/*no_restart*: "no"`, **or**
- Move the value into the CHI `settings` section and add a `settings/concurrent_threads_soft_limit_*: "no"` rule.

Either way you get the live reload without a restart you didn't need.

## Takeaways

- **Two restart paths:** pod-template (StatefulSet) changes always roll; config changes roll only if the reboot-policy engine says so.
- **Default bias:** `settings/*`, `zookeeper/*`, and raw `files` restart; **profiles, quotas, and users do not** (with a couple of explicit profile exceptions).
- **Last matching rule wins**, and **no match = no restart**.
- **Users-section edits are always live** — never a restart.
- You can tune the policy in the operator config, and the `*no_restart*` filename is a handy escape hatch for live-reloadable raw files.
