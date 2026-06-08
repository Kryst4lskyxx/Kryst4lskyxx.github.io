# Design: Rewrite "Navigating the ClickHouse Codebase with zread.ai" → "Reading Source Code in the AI Era"

**Date:** 2026-06-08
**Status:** Approved (pending spec review)
**Type:** Blog post rewrite (content + style)

## Goal

Rewrite `src/content/posts/Navigating the ClickHouse Codebase with zread.ai.md` so that:

1. The **writing style** matches the blog's house style (the `CANNOT_SCHEDULE_TASK`, schemaPolicy, and bare-metal ClickHouse posts), replacing the current marketing/listicle voice.
2. The **content** is reframed from a single-tool (zread.ai) tutorial into a methodology essay — *how to read source code in the AI era* — covering six requested points, with ClickHouse as the running concrete example.

The six content points to cover (from the request):

1. How to read source code (grounded in aredridel's *how-to-read-code*).
2. Why and when you should read source code with AI.
3. How to read source code using AI.
4. How to read source code with deepwiki and zread.ai.
5. Why you should **not** read all the code with AI — the limitations (grounded in the ClickHouse *agentic-coding* post).
6. A real-world case: ClickHouse.

## Decisions (from brainstorming)

| Decision | Choice |
|---|---|
| Framing | **Reframe as methodology essay.** ClickHouse + zread/deepwiki are the worked example, not the subject. |
| Worked example (point 6) | **The `CANNOT_SCHEDULE_TASK` trail** — cross-links the existing error-439 post; demonstrates "AI gets you 80% there, then floats plausible-but-wrong hypotheses you must reject by hand." |
| Tooling depth | **Keep practical, drop promo.** Keep URL-swap, MCP config, tool roles, "ask why not where." Cut "Getting Started in 60 Seconds," "Pro Tips," emoji tables, marketing tone. |
| deepwiki vs zread | **Contrast them (when to use which).** |
| Structure | **Approach A** — linear methodology with ClickHouse threaded through every section, plus one dedicated anchor section for the worked example. |

## House style (must match)

Derived from the sibling posts (`Debugging ClickHouse CANNOT_SCHEDULE_TASK.md`, `How clickhouse-operator schemaPolicy Creates Schema.md`, `Moving Nodes Out of a Bare-Metal ClickHouse Cluster.md`):

- **No** repeated `# H1` title, **no** tagline-under-title line, **no** `---` horizontal rules between sections.
- Frontmatter has a rich one-sentence `description` and a `slug` field.
- Sentence-case, opinionated section headers ("Why this matters", "The symptom", "Wrong answer #1").
- First-person, source-grounded, honest voice. Real code/SQL/shell blocks with file paths.
- `:::note` / `:::tip` / `:::warning` / `:::caution` admonitions used where they earn their place.
- Cross-links to sibling posts via `/posts/<slug>/`.
- Closing `## Takeaways` bullet list; optional `Related:` footer line of links.
- Tables allowed but sparing and **without emoji**.

## Frontmatter

```yaml
---
title: 'Reading Source Code in the AI Era: A ClickHouse Case Study'
published: 2026-06-08
description: 'Source code is non-linear and AI changed the economics of reading it, not the need to. How to orient with deepwiki and zread.ai, when each tool wins, why you should never let AI read all the code for you — and the ClickHouse error-439 trail where AI got me to ThreadPool.cpp fast, then floated five plausible-but-wrong causes.'
image: ''
tags: [ClickHouse, Source Code Reading, AI Coding, Developer Tooling, Big Data Engineering]
category: 'Coding'
draft: false
lang: 'en'
slug: reading-source-code-in-the-ai-era
---
```

**File rename:** rename the post file to `Reading Source Code in the AI Era.md` (via `git mv`, to preserve history and match the convention that sibling files are named by title). The `slug` field is what actually controls the URL, making it `/posts/reading-source-code-in-the-ai-era/` regardless of filename.

**Trade-off acknowledged:** new slug + refreshed date means the old `/posts/navigating-the-clickhouse-codebase-with-zread-ai/` URL changes. Accepted because the content is substantially new.

## Section-by-section outline

### §1 — The economics of reading code changed — the need didn't
- Hook: ClickHouse ~1.5M lines of C++; orientation used to cost hours.
- Framing: for most engineers the dominant cost isn't writing new code, it's *understanding existing systems* (arXiv 2504.04553).
- Thesis: AI collapsed the **orientation** cost to minutes; it did **not** remove the obligation to read and verify. The verification cost is unchanged.

### §2 — Reading code was never linear
- aredridel core model: read by *purpose*; *categorize* the code (glue / interface / implementation / algorithmic / config / tasking); reading orders (entry point, biggest file, breakpoint-trace); forward/backward process entailment.
- Convergent multi-source method:
  - Clarify the goal first, keep it narrow (Jeremy Ong).
  - Overview → build system → module map (Sparkbox, understandlegacycode).
  - Trace a **happy path** from a real entry point; ignore error handling first; tests-as-documentation; search is not cheating (HN thread).
  - Jeremy Ong's **cyclical bottom-up/top-down** (execution callstacks ↔ data structures) as the get-unstuck move.
- Close: this is the human skill AI accelerates but can't replace.

### §3 — Why and when to reach for AI
- Where LLMs genuinely shine: initial comprehension — summarizing modules, explaining idioms, translating low-level → high-level intent (arXiv 2504.04553).
- Audit-style **global → local** comprehension order.
- When NOT to: you already know the area (ties to METR deep-familiarity finding in §6), security-critical paths, anything needing ground truth not a plausible summary.

### §4 — A workflow for reading with AI
- Three moves: (a) **orient** — get the map/guide first; (b) ask **"why," not just "where"**; (c) **always land in real source and verify** — map the AI's answer back to an aredridel category and read the actual file.
- Reinforcing practices: scope it, don't dump the repo; provide rich context; "one-shot exploration to map scope"; treat the model like a **junior engineer you mentor** (Addy Osmani, Honeycomb).
- Concrete ClickHouse hook: ask *"why are `IColumn` operations immutable?"* and verify against the architecture doc (`filter`/`permute`/`cut` map to WHERE/ORDER BY/LIMIT).

### §5 — deepwiki vs zread.ai: when to use which
- The `github.com → deepwiki.com` / `zread.ai` URL swap (both).
- Contrast (one small table, no emoji):
  - **DeepWiki** (Cognition/Devin): architecture-first "senior architect" framing, English/global, multi-model (Gemini + GPT-4o), free + no-auth, free MCP at `mcp.deepwiki.com/mcp` (tools: `ask_question`, `read_wiki_contents`, `read_wiki_structure`); **no** Issues/PR/commit context; snapshot lag.
  - **zread.ai** (Zhipu/GLM): guide + Ask AI + community **"Hot Discussions"** + multi-repo compare + private-repo support; MCP requires a paid Z.ai/GLM key.
- One-line note: **Sourcegraph / Greptile** are the enterprise code-search/intelligence tier.
- MCP config blocks for both (Claude Code):
  - DeepWiki: `claude mcp add -s user -t http deepwiki https://mcp.deepwiki.com/mcp` (no auth).
  - zread: the existing `claude mcp add ... --header "Authorization: Bearer <key>"` block.
- Data-flow caution for proprietary code (`:::caution`).
- Verdict: deepwiki = fast free first read; zread = community/issue context + multi-repo.

### §6 — Where AI falls short
Three independent evidence layers (not just one vendor blog):
1. **Practitioner** — ClickHouse's Milovidov (agentic-coding): hallucinated outdated/removed APIs, *plausible-and-wrong hypotheses you must reject first*, poor architectural calls, AI as **multiplier** (good engineers gain, bad ones do more harm), validate every change.
2. **Controlled study** — METR RCT (July 2025, arXiv 2507.09089): experienced devs **19% slower** with early-2025 AI, yet *felt* ~20% faster — the perception gap and verification overhead. Note the caveats (snapshot in time, wide CI).
3. **Mechanism** (why it's structural, not a bug): **"lost in the middle"** U-curve, 30%+ mid-context accuracy drop (Stanford/Berkeley/Samaya 2023); **context rot** across all 18 models tested, with *distractor interference* especially bad for code where functions look alike (Chroma/Morph); bigger context windows don't fix it (advertised vs effective gap).
- `:::warning`: **don't let AI read all the code for you** — it's a tool of thought, not a replacement for thinking.

### §7 — The trail in practice: CANNOT_SCHEDULE_TASK
- The anchor example; framed as METR/Milovidov made concrete.
- **The win:** paste error 439 + "what code path produces this," and the tool lands you on `ThreadPool.cpp::scheduleImpl` in minutes — the orientation win.
- **The limit:** ask *why* and it confidently offers the tempting-wrong "raise `max_thread_pool_size`" (a textbook plausible-and-wrong hypothesis), among others.
- **The truth** required reading the source by hand: two distinct error branches (`no free thread` vs `failed to start the thread`), a kernel `clone()` refusal, and a query-admission stampede proven from `system.metric_log`.
- Cross-link the full hand-investigation: `/posts/debugging-clickhouse-cannot-schedule-task/`.
- This *is* the thesis made concrete: AI for orientation, human for verification.

### §8 — Takeaways
~6 bullets distilling:
- Reading code is non-linear; categorize what you're reading before you read it.
- AI cuts orientation cost, not verification cost.
- Ask "why," not just "where"; always land in real source.
- deepwiki vs zread one-liner (free fast first read vs community/issue context).
- AI floats plausible-and-wrong hypotheses — reject them against source (METR 19% slower; lost-in-the-middle/context rot are structural).
- Never let AI read all the code for you.

### §9 — Further reading / Related
- **Related** (sibling posts): error-439, schemaPolicy, bare-metal, concurrency-control.
- **Further reading** (external canonical links): aredridel how-to-read-code, ClickHouse architecture doc, ClickHouse agentic-coding post, METR study, "lost in the middle."

## What gets cut from the original

Tagline-under-H1; all `---` rules; emoji tables; "Getting Started in 60 Seconds"; "Pro Tips for Big Data Engineers"; "From 'It's All C++' to Confident Understanding"; promo voice and the zread-only quick-reference table.

**Kept and reframed:** the URL-swap trick, MCP config, "ask why not where," the MergeTree/error-439 concrete hooks.

## Constraints

- Target length ~240–280 lines (in family with siblings; up slightly from the original 244 due to richer sourcing).
- Every external claim must be attributable to one of the sources below; every ClickHouse source claim (file paths, error branches) must match the existing error-439 post or the official architecture doc.
- Build must pass (`pnpm build`) — valid frontmatter against `src/content/config.ts` Zod schema, valid markdown, working admonition directives.

## Sources

- aredridel, *How To Read Source Code* — https://github.com/aredridel/how-to-read-code/blob/master/how-to-read-code.md
- ClickHouse, *Agentic coding* (Milovidov) — https://clickhouse.com/blog/agentic-coding
- ClickHouse, *Architecture Overview* — https://clickhouse.com/docs/development/architecture
- "Understanding Codebase like a Professional" — https://arxiv.org/html/2504.04553v2
- Jeremy Ong, *Grokking Big Unfamiliar Codebases* — https://www.jeremyong.com/game%20engines/2023/01/25/grokking-big-unfamiliar-codebases/
- Sparkbox, *How to Understand a Large Codebase* — https://sparkbox.com/foundry/how_to_understand_a_large_codebase
- Addy Osmani, *My LLM coding workflow going into 2026* — https://addyosmani.com/blog/ai-coding-workflow/
- Honeycomb, *How I Code With LLMs These Days* — https://www.honeycomb.io/blog/how-i-code-with-llms-these-days
- DeepWiki vs Zread — https://www.kdjingpai.com/en/deepwiki-vs-zread-a/
- Cognition, *DeepWiki* — https://cognition.ai/blog/deepwiki and MCP — https://cognition.ai/blog/deepwiki-mcp-server
- METR, *Measuring the Impact of Early-2025 AI...* — https://metr.org/blog/2025-07-10-early-2025-ai-experienced-os-dev-study/ (arXiv 2507.09089)
- "Lost in the Middle" — https://dev.to/thousand_miles_ai/the-lost-in-the-middle-problem-why-llms-ignore-the-middle-of-your-context-window-3al2
- Morph, *Context Rot* — https://www.morphllm.com/context-rot

## Out of scope

- No changes to other posts, config, or layout.
- No new components or plugins.
- No redirect handling for the old URL (GitHub Pages static; accepted as a known break).
