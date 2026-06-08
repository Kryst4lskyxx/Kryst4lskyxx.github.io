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

## Where AI falls short

## The trail in practice: CANNOT_SCHEDULE_TASK

## Takeaways

## Further reading
