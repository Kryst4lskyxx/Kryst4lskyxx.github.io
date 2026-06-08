# Reading Source Code in the AI Era — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Rewrite the existing `Navigating the ClickHouse Codebase with zread.ai` post into a methodology essay, "Reading Source Code in the AI Era: A ClickHouse Case Study", matching the blog's house style and covering the six requested content points.

**Architecture:** Single Markdown post in `src/content/posts/`. Rename the file via `git mv`, replace frontmatter, then rebuild the body section-by-section per the approved spec. The "test" for a prose deliverable is a style lint (grep checks) plus a successful `pnpm build` (which validates frontmatter against the Zod schema in `src/content/config.ts` and renders the `:::` admonition directives).

**Tech Stack:** Astro (Fuwari template), Markdown with custom rehype admonition directives (`:::note/:::tip/:::warning/:::caution`), pnpm, Biome.

---

## Reading note for the executor (prose deliverable adaptation)

This plan builds a blog post, not code. Two kinds of content appear in the tasks:

1. **Literal blocks** (frontmatter YAML, MCP config, the comparison table, SQL/error snippets, cross-link URLs). These are reproduced **verbatim** in the steps — copy them exactly.
2. **Narrative prose** (the paragraphs of each section). These are specified as a **detailed brief**: the exact section header, the points to make in order, the specific facts/figures to cite, and the source links. Write the prose at execution time from the brief, in the author's voice (first-person, source-grounded, opinionated, sentence-case headers). Do **not** paste the brief bullets as the final text.

**House-style rules (apply to every section):**
- No repeated `# H1` title, no tagline-under-title, **no `---` horizontal rules between sections**.
- Sentence-case `##` headers exactly as given.
- First-person, honest, concrete. Inline `code` for identifiers/paths.
- Use `:::warning` / `:::caution` / `:::tip` only where specified.
- Cross-links use `/posts/<slug>/` form.

**Confirmed sibling slugs (for cross-links):**
- `debugging-clickhouse-cannot-schedule-task`
- `diagnosing-clickhouse-concurrency-control`
- `clickhouse-operator-schema-policy`
- `moving-nodes-out-of-a-bare-metal-clickhouse-cluster`
- `scaling-up-clickhouse-on-kubernetes`

**Target length:** ~240–280 lines of Markdown.

---

## File Structure

- **Rename + rewrite:** `src/content/posts/Navigating the ClickHouse Codebase with zread.ai.md` → `src/content/posts/Reading Source Code in the AI Era.md` (single deliverable file).
- No other files change.

---

## Task 1: Rename file, replace frontmatter, lay down section skeleton

**Files:**
- Rename: `src/content/posts/Navigating the ClickHouse Codebase with zread.ai.md` → `src/content/posts/Reading Source Code in the AI Era.md`

- [ ] **Step 1: Rename the file with git mv (preserves history)**

```bash
git mv "src/content/posts/Navigating the ClickHouse Codebase with zread.ai.md" \
       "src/content/posts/Reading Source Code in the AI Era.md"
```

- [ ] **Step 2: Replace the entire file contents with new frontmatter + empty section skeleton**

Overwrite the file with exactly this (frontmatter is verbatim; the body is the section headers only, to be filled in later tasks):

```markdown
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

## The economics of reading code changed — the need didn't

## Reading code was never linear

## Why and when to reach for AI

## A workflow for reading with AI

## deepwiki vs zread.ai: when to use which

## Where AI falls short

## The trail in practice: CANNOT_SCHEDULE_TASK

## Takeaways

## Further reading
```

- [ ] **Step 3: Verify frontmatter builds (schema gate)**

Run: `pnpm build`
Expected: build succeeds, no Zod/content-collection error for the post. (A pure-skeleton post is valid.)

- [ ] **Step 4: Commit**

```bash
git add -A
git commit -m "refactor: rename zread post, reset to AI-era reading skeleton"
```

---

## Task 2: Write §1 and §2 (intro + reading is non-linear)

**Files:**
- Modify: `src/content/posts/Reading Source Code in the AI Era.md`

- [ ] **Step 1: Write §1 — "The economics of reading code changed — the need didn't"**

Brief (write prose from this, in order):
- Open on the stakes: ClickHouse's source is ~1.5M lines of C++ across thousands of files; the hard part was never typing, it was *orientation* — finding the right file to even start reading.
- State the framing that for most engineers the dominant cost of development is **understanding existing systems**, not writing new code (source: arXiv 2504.04553, https://arxiv.org/html/2504.04553v2).
- Thesis of the whole essay (say it plainly): AI collapsed the orientation cost from hours to minutes — but it did **not** remove the obligation to read and verify. Orientation got cheap; verification did not.
- One sentence setting up the rest: this post is how to use AI to read code well, where the tools (deepwiki, zread.ai) help, where they actively mislead, and a real ClickHouse trail that shows both.

- [ ] **Step 2: Write §2 — "Reading code was never linear"**

Brief (write prose from this, in order):
- Core idea (source: aredridel, https://github.com/aredridel/how-to-read-code/blob/master/how-to-read-code.md): you don't read code front-to-back like a book. You read by **purpose** (comprehend / find a bug / see the interface) and you read non-linearly.
- The most useful move from aredridel: **categorize what you're looking at before you read it** — glue, interface-defining, implementation, algorithmic, configuration, tasking — because each kind demands a different question. (List them briefly with a one-clause gloss each.)
- Reading orders that work: start at the entry point; open the biggest file first; or set a breakpoint and trace the call stack.
- Convergent advice from people who do this for a living (weave, don't list mechanically):
  - Clarify your goal first and keep it narrow (source: Jeremy Ong, https://www.jeremyong.com/game%20engines/2023/01/25/grokking-big-unfamiliar-codebases/).
  - Get the overview first — README, then the build system, then the module map (sources: Sparkbox https://sparkbox.com/foundry/how_to_understand_a_large_codebase ; understandlegacycode https://understandlegacycode.com/getting-into-large-codebase/).
  - Trace a **happy path** from a real entry point and ignore error handling on the first pass; read the tests as documentation; and stop feeling guilty about search — it is essential, not a crutch.
  - Jeremy Ong's cyclical method: alternate **bottom-up** (follow execution call stacks) and **top-down** (study the data structures), and switch modes whenever you get stuck.
- Close the section with the bridge to AI: every one of these moves is something AI can *accelerate* — but the judgment in them is the part AI can't do for you.

- [ ] **Step 3: Style-lint the file**

Run: `grep -nE '^---$' "src/content/posts/Reading Source Code in the AI Era.md" | tail -n +2`
Expected: no output beyond the two frontmatter fences at the very top (i.e. NO `---` rules between sections). If any horizontal-rule `---` appears in the body, delete it.

- [ ] **Step 4: Commit**

```bash
git add "src/content/posts/Reading Source Code in the AI Era.md"
git commit -m "docs: add intro and non-linear reading sections"
```

---

## Task 3: Write §3 and §4 (why/when AI + the workflow)

**Files:**
- Modify: `src/content/posts/Reading Source Code in the AI Era.md`

- [ ] **Step 1: Write §3 — "Why and when to reach for AI"**

Brief (write prose from this, in order):
- Where LLMs genuinely help: the *friction of initial comprehension* — summarizing a module, explaining an unfamiliar idiom, translating low-level implementation into high-level intent (source: arXiv 2504.04553).
- The right comprehension order mirrors how professional code auditors work: **global → local** (project overview → structure/business logic → local functions). Prompt in that order.
- When to reach for it: a large unfamiliar codebase; "where does X actually happen?"; getting a narrative map of a subsystem; surfacing community footguns before you touch a subsystem.
- When NOT to: when you already know the area well (forward-reference §6's METR finding — experts gain little and can lose time); security-critical paths; anything where you need ground truth rather than a plausible summary.

- [ ] **Step 2: Write §4 — "A workflow for reading with AI"**

Brief (write prose from this, in order):
- Give three moves, named:
  1. **Orient** — get the generated map/guide first; don't start in a random file.
  2. **Ask "why," not just "where."** "Where is the merge logic" gets a path; "why does ClickHouse use a greedy merge selector instead of a cost-based one" gets the reasoning that actually teaches you. (This is the one genuinely good tip salvaged from the original post.)
  3. **Always land in real source and verify.** Map the AI's answer back to an aredridel category and read the actual file. The AI's summary is a hypothesis, not a citation.
- Reinforcing practices from practitioners (weave in):
  - **Scope it; never dump the whole repo.** Feed the relevant modules, not everything (sources: Addy Osmani https://addyosmani.com/blog/ai-coding-workflow/ ; Honeycomb https://www.honeycomb.io/blog/how-i-code-with-llms-these-days).
  - Treat the model like a **junior engineer you mentor** — bite-size asks, you hold the judgment.
- Concrete ClickHouse illustration to ground the section: ask *"why are `IColumn` operations immutable?"*, then verify against the official architecture doc (https://clickhouse.com/docs/development/architecture) — that `IColumn::filter`, `IColumn::permute`, and `IColumn::cut` map directly to the `WHERE`, `ORDER BY`, and `LIMIT` operators. Note the nice property that the answer is checkable against real method names in minutes — which is exactly the point.

- [ ] **Step 3: Style-lint the file**

Run: `grep -nE '^---$' "src/content/posts/Reading Source Code in the AI Era.md" | tail -n +2`
Expected: no output. (No body `---` rules.)

- [ ] **Step 4: Commit**

```bash
git add "src/content/posts/Reading Source Code in the AI Era.md"
git commit -m "docs: add why/when-AI and read-with-AI workflow sections"
```

---

## Task 4: Write §5 (deepwiki vs zread.ai)

**Files:**
- Modify: `src/content/posts/Reading Source Code in the AI Era.md`

- [ ] **Step 1: Write the §5 intro prose — "deepwiki vs zread.ai: when to use which"**

Brief:
- Both tools turn a GitHub repo into an AI-readable map, and both share the same entry trick: **swap the host in the URL.** `github.com/ClickHouse/ClickHouse` becomes `deepwiki.com/ClickHouse/ClickHouse` or `zread.ai/ClickHouse/ClickHouse`. No clone, no signup for public repos.
- They are not the same tool, though, and the difference decides which to open.

- [ ] **Step 2: Insert the comparison table verbatim (no emoji)**

```markdown
| | DeepWiki | zread.ai |
|---|---|---|
| Maker | Cognition (the Devin team) | Zhipu AI (GLM models) |
| Shape | Auto-generated wiki + chat, architecture-first | Structured guide + Ask AI |
| Community context | None — no Issues, PRs, or commit history | "Hot Discussions" mined from Issues & PRs |
| Multi-repo / private | Public repos, cloud-only | Multi-repo compare; private-repo support |
| MCP server | Free, no auth: `mcp.deepwiki.com/mcp` | Requires a paid Z.ai / GLM key |
| Freshness | Snapshot, regenerated on a schedule (can lag main) | Re-analyzes on demand |
```

(Sources for these facts: DeepWiki vs Zread https://www.kdjingpai.com/en/deepwiki-vs-zread-a/ ; Cognition https://cognition.ai/blog/deepwiki and https://cognition.ai/blog/deepwiki-mcp-server .)

- [ ] **Step 3: Write the MCP setup prose + insert both config blocks verbatim**

Prose: for daily work, wire either one into your AI IDE over MCP so the assistant reads real docs instead of hallucinating. DeepWiki's server is free and needs no auth; zread's needs a paid key.

DeepWiki (Claude Code) — verbatim:

````markdown
```bash
# DeepWiki MCP — free, no auth
claude mcp add -s user -t http deepwiki https://mcp.deepwiki.com/mcp
```
````

zread (Claude Code) — verbatim:

````markdown
```bash
# zread MCP — requires a Z.ai API key + GLM Coding Plan
claude mcp add -s user -t http zread \
  https://api.z.ai/api/mcp/zread/mcp \
  --header "Authorization: Bearer <your_zai_api_key>"
```
````

One line on the DeepWiki MCP tools: `ask_question`, `read_wiki_contents`, `read_wiki_structure`.

- [ ] **Step 4: Add the proprietary-code caution and the verdict**

Insert this admonition verbatim:

```markdown
:::caution
Before you point any of these at proprietary code, check the data flow. The public tiers are cloud services that index your repo on their servers. For closed-source work, prefer a private-repo plan or a self-hosted option.
:::
```

Verdict prose: reach for **DeepWiki** when you want a fast, free, architecture-first first read; reach for **zread.ai** when you want community/issue context or to compare repos. One sentence noting that **Sourcegraph** and **Greptile** are the heavier, enterprise code-search/intelligence tier if that's what you actually need (source: https://rywalker.com/research/code-intelligence-tools).

- [ ] **Step 5: Verify build (renders code blocks + caution directive)**

Run: `pnpm build`
Expected: build succeeds; the `:::caution` renders without a directive error.

- [ ] **Step 6: Commit**

```bash
git add "src/content/posts/Reading Source Code in the AI Era.md"
git commit -m "docs: add deepwiki vs zread comparison and MCP setup"
```

---

## Task 5: Write §6 (where AI falls short — three evidence layers)

**Files:**
- Modify: `src/content/posts/Reading Source Code in the AI Era.md`

- [ ] **Step 1: Write the §6 prose — "Where AI falls short"**

Brief (write prose from this, in order — three independent layers so it isn't one vendor's opinion):

Layer 1 — **the practitioner.** ClickHouse's own Alexey Milovidov on agentic coding (source: https://clickhouse.com/blog/agentic-coding): on the big C++ base the agent "gets lost"; it hallucinates outdated command-line params and reintroduces patterns that were refactored out; it makes poor architectural decisions and is best on small contained tasks. The line to quote/paraphrase: during a hard bug hunt it generated "a lot of false but convincing hypotheses" you have to reject first — and AI is a **multiplier**, so a good engineer gets better and a careless one does more harm. Their discipline: validate every change, lean on tests/CI, "agent does, you review."

Layer 2 — **the controlled study.** METR's randomized trial (July 2025, source: https://metr.org/blog/2025-07-10-early-2025-ai-experienced-os-dev-study/ , arXiv 2507.09089): 16 experienced open-source devs on repos they averaged ~5 years on were **19% slower** with early-2025 AI tools — while *believing* they were ~20% faster. The gap is the lesson: the time goes to verifying and cleaning AI output, and experts on familiar code have little room to gain. Note the honest caveats: it's a snapshot of early-2025 tools and the confidence interval is wide.

Layer 3 — **the mechanism** (why this is structural, not a bug you can prompt around). LLMs predict the next token; when the answer isn't well represented they produce something *plausible*. Two well-documented effects make this worse on big codebases:
- **"Lost in the middle"** (Stanford/Berkeley/Samaya, 2023; source: https://dev.to/thousand_miles_ai/the-lost-in-the-middle-problem-why-llms-ignore-the-middle-of-your-context-window-3al2): attention is U-shaped — strong at the start and end of the context, weak in the middle; relevant material in the middle can drop accuracy 30%+.
- **Context rot** (Chroma, 18 models; source: https://www.morphllm.com/context-rot): every model tested degrades as context grows, and **distractor interference** is especially nasty for code, where many functions and variables look alike and actively mislead the model. Bigger context windows do not fix this — effective context is far below the advertised number.

- [ ] **Step 2: Add the headline warning admonition verbatim**

Place this near the end of §6:

```markdown
:::warning
Don't let AI read *all* the code for you. It is a tool of thought, not a replacement for thinking. Treat every AI explanation as a hypothesis to confirm against the source — not a citation.
:::
```

- [ ] **Step 3: Style-lint the file**

Run: `grep -nE '^---$' "src/content/posts/Reading Source Code in the AI Era.md" | tail -n +2`
Expected: no output.

- [ ] **Step 4: Commit**

```bash
git add "src/content/posts/Reading Source Code in the AI Era.md"
git commit -m "docs: add AI limitations section with practitioner, RCT, and mechanism evidence"
```

---

## Task 6: Write §7 (the CANNOT_SCHEDULE_TASK trail)

**Files:**
- Modify: `src/content/posts/Reading Source Code in the AI Era.md`

- [ ] **Step 1: Write the §7 prose — "The trail in practice: CANNOT_SCHEDULE_TASK"**

Brief (write prose from this, in order — this is the essay's payoff: AI for orientation, human for verification):

- Framing sentence: here is the whole thesis on one real bug — where AI saved hours, and where trusting it would have cost a day.
- **The win (orientation).** A production cluster threw bursts of error 439. Paste the error into deepwiki/zread Ask, ask "what code path produces `CANNOT_SCHEDULE_TASK`," and within a minute you land on `ThreadPool.cpp` and `ThreadPoolImpl::scheduleImpl` — the file you'd otherwise have grepped thousands of C++ files to find. That is exactly what these tools are for.

- Insert the error snippet verbatim:

````markdown
```
Code: 439. DB::Exception: Cannot schedule a task:
  failed to start the thread (threads=3, jobs=3). (CANNOT_SCHEDULE_TASK)
```
````

- **The limit (verification).** Ask the AI *why*, and it confidently offers the textbook fix: raise `max_thread_pool_size`. That is a perfect example of a **plausible-and-wrong hypothesis** (callback to §6). It's wrong: a community report failed at ~15000 threads with the ceiling set to 20000 — failing *below* the limit means the limit isn't the constraint. No amount of raising a ClickHouse setting fixes a kernel that is refusing `pthread_create`.
- **The truth — which only came from reading the source and the data by hand:** error 439 is really *two* errors sharing one code — `no free thread` (ClickHouse's pool is full) versus `failed to start the thread` (the kernel refused `clone()`); the small `threads=3` count is the tell. The real cause was a query-admission stampede — hundreds of distributed queries landing in one second, each demanding pipeline/connection threads — proven not by intuition but by querying `system.metric_log` for the failure second, where the only column that moved was the concurrent `Query` count.
- Close with the cross-link, e.g. "I wrote up the full hand-investigation — five wrong answers ruled out one at a time — in [Debugging ClickHouse CANNOT_SCHEDULE_TASK](/posts/debugging-clickhouse-cannot-schedule-task/)." Land the point: the AI got me to the right file in a minute; it could not have gotten me to the right *answer*. That division of labor is the whole game.

- [ ] **Step 2: Verify build**

Run: `pnpm build`
Expected: build succeeds; the cross-link renders.

- [ ] **Step 3: Commit**

```bash
git add "src/content/posts/Reading Source Code in the AI Era.md"
git commit -m "docs: add CANNOT_SCHEDULE_TASK worked-example section"
```

---

## Task 7: Write §8 Takeaways + §9 Further reading/Related, final verification

**Files:**
- Modify: `src/content/posts/Reading Source Code in the AI Era.md`

- [ ] **Step 1: Write §8 — "Takeaways" (bullet list, ~6 bullets)**

Write as a `-` bullet list, each bullet one tight sentence, bolded lead-in like the sibling posts. Cover, in order:
- Reading code is non-linear — **categorize** what you're reading before you read it.
- AI cut the **orientation** cost, not the **verification** cost.
- Ask **"why," not just "where,"** and always land in the real source.
- **deepwiki vs zread**: free fast architecture-first first read, vs community/issue context and multi-repo.
- AI floats **plausible-and-wrong hypotheses** — experts went 19% slower trusting them (METR); "lost in the middle" and context rot make it structural, not a prompting problem.
- **Never let AI read all the code for you.**

- [ ] **Step 2: Write §9 — "Further reading" + a Related line**

Under the `## Further reading` header, an external-links bullet list (verbatim targets):
- aredridel, *How To Read Source Code* — https://github.com/aredridel/how-to-read-code/blob/master/how-to-read-code.md
- ClickHouse, *Architecture Overview* — https://clickhouse.com/docs/development/architecture
- ClickHouse, *Agentic coding* — https://clickhouse.com/blog/agentic-coding
- METR, *Measuring the Impact of Early-2025 AI on Developer Productivity* — https://metr.org/blog/2025-07-10-early-2025-ai-experienced-os-dev-study/

Then a final italic Related line linking siblings (verbatim):

```markdown
*Related: [Debugging ClickHouse CANNOT_SCHEDULE_TASK](/posts/debugging-clickhouse-cannot-schedule-task/) · [How clickhouse-operator schemaPolicy Creates Schema](/posts/clickhouse-operator-schema-policy/) · [Moving Nodes Out of a Bare-Metal ClickHouse Cluster](/posts/moving-nodes-out-of-a-bare-metal-clickhouse-cluster/)*
```

- [ ] **Step 3: Full style-lint sweep**

Run: `grep -nE '^---$' "src/content/posts/Reading Source Code in the AI Era.md" | tail -n +2`
Expected: no output (only the two top frontmatter fences exist).

Run: `grep -nE '🧱|🌲|⚡|📦|🗄️|📨|🔁|🧮|🔍|🧩|🐛|📋|🔗|⚙️|💡|ℹ️' "src/content/posts/Reading Source Code in the AI Era.md" || echo "no emoji"`
Expected: `no emoji` (the original's decorative emoji must be gone).

Run: `grep -cE '^## ' "src/content/posts/Reading Source Code in the AI Era.md"`
Expected: `9` (the nine section headers).

- [ ] **Step 4: Final build + type check (integration gate)**

Run: `pnpm build`
Expected: build succeeds, post is generated, pagefind index builds without error.

Run: `pnpm check`
Expected: no new errors attributable to the post.

- [ ] **Step 5: Read-through for voice**

Open `pnpm dev` and read the rendered post at `http://localhost:4321/posts/reading-source-code-in-the-ai-era/`. Confirm: no marketing tone, no leftover "Pro Tips"/"60 Seconds"/tagline, admonitions render, all links resolve, length ~240–280 lines (`wc -l` on the file).

- [ ] **Step 6: Final commit**

```bash
git add "src/content/posts/Reading Source Code in the AI Era.md"
git commit -m "docs: add takeaways and further-reading; finalize AI-era reading post"
```

---

## Self-Review (completed by plan author)

**1. Spec coverage** — every spec section maps to a task:
- Frontmatter/rename → Task 1.
- §1 economics, §2 non-linear → Task 2.
- §3 why/when, §4 workflow → Task 3.
- §5 deepwiki vs zread (table + MCP + caution + verdict) → Task 4.
- §6 limitations (3 layers + warning) → Task 5.
- §7 CANNOT_SCHEDULE_TASK trail → Task 6.
- §8 Takeaways, §9 Further reading/Related → Task 7.
- "What gets cut" → enforced by the emoji/`---` style-lints in Tasks 2–7.
- Six requested content points: #1 (read source code)→§2; #2 (why/when with AI)→§3; #3 (how with AI)→§4; #4 (deepwiki+zread)→§5; #5 (don't read all with AI / limits)→§6; #6 (ClickHouse case)→§7 (plus threaded in §1/§4).

**2. Placeholder scan** — the only `<...>` is `<your_zai_api_key>`, which is an intentional reader-supplied value, verbatim from the source config. No TBD/TODO/"handle edge cases."

**3. Type/fact consistency** — section header strings in Task 1 skeleton match the headers referenced in Tasks 2–7 and the `grep -c '^## '` expected count of 9. Slugs match the confirmed list. MCP endpoints/flags match the spec's Sources. Error code (439), file (`ThreadPool.cpp`/`scheduleImpl`), and the `max_thread_pool_size` 20000/15000 detail match the existing `debugging-clickhouse-cannot-schedule-task` post.
