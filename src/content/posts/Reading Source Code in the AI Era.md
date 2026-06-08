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

ClickHouse's source is roughly 1.5 million lines of C++ spread across thousands of files. The hard part was never typing — it was orientation. Which file do I even open first? Where does this error message come from? What calls this function? For most engineers the dominant cost of development is understanding existing systems, not writing new code. That orientation phase used to take hours. AI collapsed it to minutes.

But AI did not remove the obligation to read and verify. Orientation got cheap. Verification did not. The tools will hand you a file, a function, a plausible explanation. You still have to confirm it is correct, that the explanation matches what the code actually does, that the edge cases are real. This is how to use AI to read code well — where the tools help, where they actively mislead, and a real ClickHouse trail that shows both.

## Reading code was never linear

You don't read code front-to-back like a book. This is core to Aria Stewart's [*How To Read Source Code*](https://github.com/aredridel/how-to-read-code/blob/master/how-to-read-code.md): you read by purpose — to comprehend, to find a bug, to see the interface — and you read non-linearly. The most useful move is to categorize what you're looking at before you read it. Glue code connects components. Interface-defining code sets contracts. Implementation code does the work. Algorithmic code optimizes the work. Configuration code parameterizes behavior. Tasking code orchestrates execution. Each kind demands a different question.

Reading orders that work: start at the entry point and trace forward. Open the biggest file first to get a sense of scale. Set a breakpoint and walk the call stack backward. Experienced engineers converge on similar advice. Clarify your goal first and keep it narrow — you are not reading the entire codebase, you are answering one question. Get the overview first: the README, then the build system, then the module map. Trace a happy path from a real entry point and ignore error handling on the first pass. Read the tests as documentation — they show actual usage. Stop feeling guilty about search. Search is essential, not a crutch. Alternate bottom-up (follow execution call stacks) and top-down (study the data structures), and switch modes whenever you get stuck.

Every one of these moves is something AI can accelerate. But the judgment in them is the part AI cannot do for you.

## Why and when to reach for AI

LLMs genuinely help with the friction of initial comprehension. Summarizing what a module does. Explaining an unfamiliar idiom you've never seen. Translating low-level implementation into high-level intent. These are all first-pass orientation tasks where you need a plausible narrative to get started, not ground truth. The model gives you that narrative. You then verify it.

The right comprehension order mirrors how professional code auditors work: global to local. Start with the project overview, then the structure and business logic, then the local functions. Prompt in that order. Ask for the architecture before you ask about a single function. Ask what a subsystem does before you ask how a method works. This is how you orient without getting lost in implementation details that only make sense once you have the map.

When to reach for it: you are facing a large unfamiliar codebase. You need to know where something actually happens. You want a narrative map of a subsystem before you start reading files. You want to surface community footguns before you touch a subsystem. The value is highest when you have no prior context. The model shortens the time from zero knowledge to rough orientation.

When not to: you already know the area well. Experts gain little and can even lose time — we'll cover this in the limitations section. Anything security-critical where you need ground truth rather than a plausible summary. Anything where the cost of being wrong is high and the cost of reading the real source is low. The model is a shortcut to orientation, not a replacement for verification.

## A workflow for reading with AI

Three moves work reliably. **Orient first.** Get the generated map before you start reading. Don't open a random file and ask the model what it does. Ask for the overview, the module structure, the control flow. Then drill down. You want the narrative before the details. **Ask "why," not just "where."** "Where is the merge logic" gets you a file path. "Why does ClickHouse use a greedy merge selector instead of a cost-based one" gets you the reasoning that actually teaches you the system. The second question is harder to answer but far more valuable. It forces the model to explain trade-offs, not just locate code. **Always land in real source and verify.** The AI's answer is a hypothesis, not a citation. Map the summary back to a code category — glue, interface, implementation, whatever — and read the actual file. If the model says "the executor schedules tasks in `ThreadPool.cpp`," open `ThreadPool.cpp` and confirm that `schedule` does what the summary claims.

A few reinforcing practices. Scope it. Never dump the whole repo into the context window. Feed the relevant modules, not everything. The model will happily summarize irrelevant code if you let it. Treat the model like a junior engineer you mentor. Give it bite-size asks. You hold the judgment. The model proposes, you verify. This keeps you in the driver's seat.

Here's a concrete ClickHouse illustration. You ask "why are `IColumn` operations immutable?" The model tells you it's because ClickHouse's query execution model maps SQL operators directly to column transformations, and immutability prevents accidental sharing bugs. You then verify against the [official ClickHouse architecture doc](https://clickhouse.com/docs/development/architecture) and confirm that `IColumn::filter`, `IColumn::permute`, and `IColumn::cut` map directly to `WHERE`, `ORDER BY`, and `LIMIT` operators. This answer is checkable against real method names in minutes. That's exactly why the workflow works. The model gives you the high-level intent. You confirm it against the implementation. You learn faster than if you had read `IColumn.h` cold, and you learn correctly because you verified.

## deepwiki vs zread.ai: when to use which

Both tools turn a GitHub repo into an AI-readable map, and both share the same entry trick: swap the host in the URL. `github.com/ClickHouse/ClickHouse` becomes `deepwiki.com/ClickHouse/ClickHouse` or `zread.ai/ClickHouse/ClickHouse`. No clone, no signup for public repos. They are not the same tool, and the difference decides which to open.

| | DeepWiki | zread.ai |
|---|---|---|
| Maker | Cognition (the Devin team) | Zhipu AI (GLM models) |
| Shape | Auto-generated wiki + chat, architecture-first | Structured guide + Ask AI |
| Community context | None — no Issues, PRs, or commit history | "Hot Discussions" mined from Issues & PRs |
| Multi-repo / private | Public repos, cloud-only | Multi-repo compare; private-repo support |
| MCP server | Free, no auth: `mcp.deepwiki.com/mcp` | Requires a paid Z.ai / GLM key |
| Freshness | Snapshot, regenerated on a schedule (can lag main) | Re-analyzes on demand |

For daily work, wire either one into your AI IDE over MCP so the assistant reads real docs instead of hallucinating. DeepWiki's server is free and needs no auth; zread's needs a paid key.

```bash
# DeepWiki MCP — free, no auth
claude mcp add -s user -t http deepwiki https://mcp.deepwiki.com/mcp
```

```bash
# zread MCP — requires a Z.ai API key + GLM Coding Plan
claude mcp add -s user -t http zread \
  https://api.z.ai/api/mcp/zread/mcp \
  --header "Authorization: Bearer <your_zai_api_key>"
```

DeepWiki's MCP exposes three tools: `ask_question`, `read_wiki_contents`, `read_wiki_structure`.

:::caution
Before you point any of these at proprietary code, check the data flow. The public tiers are cloud services that index your repo on their servers. For closed-source work, prefer a private-repo plan or a self-hosted option.
:::

Reach for DeepWiki when you want a fast, free, architecture-first first read. Reach for zread.ai when you want community/issue context or to compare repos. Sourcegraph and Greptile are the heavier, enterprise code-search/intelligence tier if that's what you actually need.

## Where AI falls short

The orientation story is appealing. It is also partial. AI genuinely accelerates the initial map-making — but that is a low-stakes task. The moment you ask the model to deeply understand a large codebase on your behalf, you enter territory where the tools systematically fail and the failure is hard to detect.

**The practitioner's report.** Alexey Milovidov, creator of ClickHouse, has written about [agentic coding on the ClickHouse codebase](https://clickhouse.com/blog/agentic-coding). On a big C++ base the agent gets lost. It hallucinates command-line parameters that no longer exist. It reintroduces architectural patterns that were deliberately refactored out years ago, because its training data is stale. It makes poor design decisions and is best confined to small, well-scoped tasks. The key observation: during a hard bug hunt the agent generated "a lot of false but convincing hypotheses" that you have to spend time rejecting before you can find the real cause. And AI is a multiplier — a good engineer who validates every change becomes more productive, a careless one who trusts the output does more harm. The discipline that makes it work is unforgiving: validate every change, lean on your test suite and CI, and remember that the agent does, you review.

**The controlled study.** METR ran a [randomized controlled trial](https://metr.org/blog/2025-07-10-early-2025-ai-experienced-os-dev-study/) with 16 experienced open-source developers working on repositories they had contributed to for about 5 years on average. With early-2025 AI tools they were 19% slower than without the tools — while they believed they were about 20% faster. That perception gap is the lesson. The time goes into verifying and cleaning up AI output, and experts on familiar code have little room to gain because they already know where things are. The study is a snapshot of early-2025 tooling and the confidence interval is wide, but the finding is humbling: the people who should benefit most — experts on their own codebases — are slowed down, and they do not notice.

**The mechanism.** This is not a prompt-engineering problem you can fix. LLMs predict the next token. When the answer is not well represented in the training data they produce something plausible rather than refusing to answer. Two documented effects make this worse on large codebases. First, "lost in the middle": a 2023 Stanford/Berkeley study showed that attention is U-shaped. Models attend strongly to the start and end of the context window and weakly to the middle. Relevant material buried in the middle can drop accuracy by 30% or more. Second, "context rot": across many models tested, every one degrades as context size grows. Distractor interference is especially nasty for code, where many functions and variable names look alike and actively mislead the model. Bigger context windows do not fix this — the effective context is far below the advertised token limit.

Put the layers together. Orientation is exactly the low-stakes, easily verifiable task AI is good at. "Read and understand all of this for me" is exactly the high-stakes, hard-to-verify task it is bad at. The model will give you an answer either way. The difference is whether you can catch the error before it costs you time.

:::warning
Don't let AI read *all* the code for you. It is a tool of thought, not a replacement for thinking. Treat every AI explanation as a hypothesis to confirm against the source — not a citation.
:::

## The trail in practice: CANNOT_SCHEDULE_TASK

Here is the whole thesis on one real bug — where AI saved hours, and where trusting it would have cost a day.

A production ClickHouse cluster started throwing bursts of error 439. The error text was cryptic:

```
Code: 439. DB::Exception: Cannot schedule a task:
  failed to start the thread (threads=3, jobs=3). (CANNOT_SCHEDULE_TASK)
```

This is exactly where the AI tools earn their keep. I pasted the error into deepwiki's Ask interface and asked "what code path produces `CANNOT_SCHEDULE_TASK`?" Within a minute I had the answer: `ThreadPool.cpp`, the `ThreadPoolImpl::scheduleImpl` function. That is the file I would have otherwise grepped thousands of C++ files to find, and the model got me there in one shot. Orientation phase complete.

Then I asked the model why this happens. It confidently offered the textbook fix: raise `max_thread_pool_size`. This is a perfect example of a plausible-and-wrong hypothesis. It is wrong because a community report failed at around 15000 threads with the ceiling already set to 20000 — failing below the configured limit means the limit is not the constraint. No amount of raising a ClickHouse setting fixes a kernel that is refusing to create threads.

The truth only came from reading the source and the data by hand. Error 439 is really two different errors sharing one code: `no free thread` means ClickHouse's pool is genuinely full; `failed to start the thread` means the kernel refused `pthread_create` or `clone()`. The small `threads=3` count is the tell — the pool was nearly empty. The real cause was a query-admission stampede: hundreds of distributed queries landing in the same second, each demanding pipeline and connection threads. That was proven not by intuition but by querying `system.metric_log` for the failure second, where the only column that moved was the concurrent `Query` count.

I wrote up the full hand-investigation — five wrong answers ruled out one at a time — in [Debugging ClickHouse CANNOT_SCHEDULE_TASK](/posts/debugging-clickhouse-cannot-schedule-task/). The AI got me to the right file in a minute. It could not have gotten me to the right answer. That division of labor — AI for orientation, human for verification — is the whole game.

## Takeaways

## Further reading
