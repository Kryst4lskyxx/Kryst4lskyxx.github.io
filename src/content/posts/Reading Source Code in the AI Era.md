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

## A workflow for reading with AI

## deepwiki vs zread.ai: when to use which

## Where AI falls short

## The trail in practice: CANNOT_SCHEDULE_TASK

## Takeaways

## Further reading
