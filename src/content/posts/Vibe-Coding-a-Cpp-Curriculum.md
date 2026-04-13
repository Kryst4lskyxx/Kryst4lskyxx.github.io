---
title: I Vibe-Coded a C++ Curriculum — Here's What the Process Taught Me
published: 2026-04-13
description: 'How I built cpp-self-learning — a structured, test-driven C++ curriculum — through AI collaboration, and what that process revealed about pedagogy and working with AI on serious projects.'
image: ''
tags: [C++, Vibe-Coding, Developer Tooling, Learning]
category: 'Coding'
draft: false
lang: 'en'
---

## The Apparent Contradiction

There's a perception that vibe coding — letting an AI drive most of the implementation while you direct at a high level — produces throwaway code. Quick prototypes. Demos that don't scale.

[cpp-self-learning](https://github.com/Kryst4lskyxx/cpp-self-learning) is my attempt to challenge that assumption. It's a structured, test-driven C++ curriculum for experienced programmers — 9 foundation modules, a mini search engine project, and optional expert tracks — built almost entirely through AI collaboration.

But this post isn't really about the curriculum. It's about what the *process* of building it taught me about working with AI on something that requires sustained design discipline.

---

## What I Built 🏗️

Before we get into the how, here's what exists:

| Section | What's Inside |
|---|---|
| **Foundations (Modules 01–09)** | Sequential modules from tooling → templates → C++20 concepts |
| **Project 01: Mini Search Engine** | Inverted index, term frequency scoring, CLI interface |
| **Expert Tracks** | Optional advanced material: move semantics, performance profiling |

The curriculum is designed for developers who already know how to program — Python, Go, JavaScript — and want to pick up modern C++. It assumes you can read code; it teaches you to think in C++ idioms.

Every exercise is validated with CTest. The canonical loop:

```bash
cmake -S . -B build
cmake --build build --target module_XX_tests
ctest --test-dir build -R module_XX_name
```

No ad hoc compiler commands. Learners know exactly what to run.

---

## The Shape of the Thing 📐

The first design decision was also the most important: every module must have the same structure.

```
module-XX/
├── README.md          # Entry point: prerequisites, file list, verification steps
├── lesson.md          # Short, code-oriented concept notes
├── assignment/        # Starter code with explicit checkpoints
├── drills/            # Optional focused practice
├── tests/             # CTest validation
└── expert/            # Optional depth (never required)
```

This sounds obvious in retrospect. But maintaining it across nine modules — where each introduces different C++ concepts, different code shapes, different test patterns — is where things get interesting.

**A human developer drifts.** Module 01 gets perfect docs. By Module 06 you're moving fast and the README is thin. The `drills/` folder is empty because you didn't have time. The `expert/` section got folded into the main assignment because separating it felt like overhead.

The AI didn't drift. Every module I shipped had the same contract. When something was thin, I noticed it immediately because the shape was wrong — the AI made the gap visible. The consistent structure became a quality bar that was easy to enforce precisely because the AI held it without effort.

> **Insight:** When you vibe code with AI, consistency is free. The AI doesn't get tired or impatient. Establish structural contracts early, and let the AI maintain them across every iteration.

---

## Making the Contract Visible 📋

The second design decision runs through the whole curriculum: **make implicit assumptions explicit**.

In C++, this looks like using C++20 `requires` clauses to express constraints in the function signature rather than letting them surface as incomprehensible compiler errors:

```cpp
template <typename Range, typename Selector>
    requires std::invocable<Selector, std::ranges::range_value_t<Range>>
auto group_by(const Range& values, Selector selector)
    -> std::map<std::invoke_result_t<Selector, std::ranges::range_value_t<Range>>,
                std::vector<std::ranges::range_value_t<Range>>>;
```

Before you read the body of this function, the signature tells you: the selector receives elements from the range, and the result is a map from selector-return-type to vectors. The contract is the interface.

In the curriculum's assignments, this principle shows up as **checkpoint questions** embedded in the starter code:

```cpp
// Checkpoint: What type does the selector receive?
// Checkpoint: Can you describe the return shape without reading the implementation?
```

These questions force learners to understand the interface before implementing it. They shift the exercise from "figure out what the test wants" to "translate the contract into code."

This design emerged from collaboration. I kept seeing modules where learners would get stuck — not because the concept was hard, but because the assignment left too much implicit. The AI helped identify every place where an assumption was buried, and we surfaced it into a checkpoint.

> **Insight:** "Make the contract visible" is good C++ advice. It's also good pedagogy. When collaborating with AI, ask it to find the implicit assumptions in your design — it's surprisingly good at this.

---

## The Collaboration Dynamic 🤝

Here's how the division of labor actually felt:

| What I Brought | What the AI Brought |
|---|---|
| Vision for the curriculum arc | Implementation of every module to spec |
| Pedagogical instincts ("this doesn't feel right") | Consistency across all 9 modules |
| "Nope, redo this" judgment | Stamina to iterate without getting impatient |
| The decision about *what* to teach | The ability to teach it clearly in code |

The key dynamic: I held the *why*, the AI held the *what*. When I said "Module 07 should teach the tradeoffs between `std::map` and `std::unordered_map` through a real indexing task," I got back a lesson, an assignment with checkpoints, drills isolating the specific tradeoffs, and tests that validated the indexing logic. Correctly. On the first pass.

What I couldn't have predicted going in: how much the AI would force me to sharpen my own thinking. To instruct it well, I had to be precise about what I wanted. "Teach templates" is too vague. "Build a `filter_values` function template that works on any range type, using `std::copy_if` with a caller-supplied predicate, returning a new vector without mutating the input" — that's something the AI can work with.

The friction of writing precise instructions made the curriculum better.

---

## The Moment That Surprised Me 🎯

I was deep in the main path — Modules 01 through 09, the mini search engine — focused on shipping the core curriculum. The AI had been executing module after module to spec. I wasn't planning anything beyond the main path.

Then, after we finished Module 09 (C++20 concepts and `requires` clauses), the AI proposed something I hadn't asked for.

It noted that the main path was intentionally scope-limited — no move semantics, no performance profiling — but that motivated learners would hit a ceiling and not know where to go next. It proposed **optional expert tracks**: separate modules on move semantics and value categories, and on performance profiling with a benchmark harness. Clearly scoped as optional. Never blocking the main path. Available for learners who wanted a higher ceiling.

I hadn't planned this. The concept didn't exist in my mental model of the curriculum.

But when I saw the proposal, it was obviously right. The main path was coherent and complete for its intended audience. The expert tracks gave motivated learners a place to go without bloating the core. The "never required" constraint was the key design insight — it kept both halves honest.

This is the moment I think about when someone asks me whether vibe coding can produce serious work. The AI, having internalized the curriculum's design philosophy across nine modules of context, proposed an architectural extension that served learners better than my original plan.

It didn't exceed its instructions. It understood them well enough to extrapolate.

> **Insight:** If the AI has enough context about your design philosophy, it will sometimes propose things you didn't ask for — and they'll be good. This is the signal that the collaboration is working.

---

## The Result 📦

Here's where the project stands today:

**Foundations (Modules 01–09):** Complete and testable. Each module builds on the last:
- 01–04: Tooling, strings/streams, classes, templates
- 05–09: Iterators, algorithms, associative containers, callables, C++20 concepts

**Project 01: Mini Search Engine:** ~190 lines. Inverted index, term frequency scoring, CLI. A synthesis exercise that combines everything from the foundations.

**Expert Tracks:** Two optional modules — move semantics and performance profiling. The benchmark harness in Expert 02 measures cache effects directly: you'll see the difference between row-major and column-major matrix traversal in nanoseconds.

If you want to work through it:

```bash
git clone https://github.com/Kryst4lskyxx/cpp-self-learning
cmake -S . -B build
cmake --build build --target module_01_tests
ctest --test-dir build -R module_01
```

Start at Module 01. Read the `README.md`, then `lesson.md`, then open the assignment. Work through the checkpoints before looking at the tests.

---

## Takeaways for Building Learning Resources with AI 💡

If you want to try this approach, here's what I'd pass forward:

| Lesson | What It Means in Practice |
|---|---|
| **Establish structural contracts early** | Define your module shape before writing Module 01. The AI will hold it forever. |
| **Be precise in your instructions** | "Teach templates" → nothing useful. "Build X using Y, returning Z without mutating input" → correct first pass. |
| **Use the AI to surface implicit assumptions** | Ask it to review your assignment prompts and find anything a learner might not understand without you explaining it. |
| **Watch for unsolicited proposals** | When the AI proposes something you didn't ask for, pay attention. If it has enough context, these are often right. |
| **The friction is the value** | Writing precise instructions to the AI sharpens your own thinking. The curriculum is better because I had to articulate it clearly enough to be instructable. |

---

The curriculum is on GitHub at [Kryst4lskyxx/cpp-self-learning](https://github.com/Kryst4lskyxx/cpp-self-learning). If you're an experienced developer who's been meaning to pick up modern C++, it's designed for exactly that.

And if you're building learning resources of your own — in any language, on any topic — vibe coding is a serious tool for this kind of work. The key is giving the AI enough design context to extrapolate from, and trusting it when it does.
