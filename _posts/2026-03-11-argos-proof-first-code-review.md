---
layout: post
title: "I Was Tired of Chasing Ghosts. So I Built Argos."
date: 2026-03-11
author: "Ihor Katkov"
description: "AI code reviewers hallucinate. I built a multi-hunter agent that proves every bug before reporting it."
tags: [ai-agents, code-review, opencode, gemini, engineering]
---

I closed the tab. Third time this week.

Claude Code's review plugin had flagged a potential null pointer in the reconciliation service. I pulled up the code, traced the call path, checked the guard conditions. The bug didn't exist. The guard was right there. The model just... invented it.

I opened a new file and typed: `argos.md`.

## The Hallucination Problem

Claude Code's review plugin is genuinely useful. That's what makes this frustrating — if it were bad across the board, I'd just stop using it. But it catches real things. It notices the subtle stuff. It asks the right questions about edge cases.

And then, one in three findings, it makes something up.

Not "slightly wrong" — completely fabricated. A race condition that can't happen given the transaction isolation level. A missing validation that's handled two layers up the call stack. An injection risk on a parameterized query.

The problem isn't the model. The problem is that I have no way to distinguish real findings from hallucinated ones without manually investigating each one. Every review becomes a ghost hunt. Forty minutes of my time to debunk two imaginary bugs and confirm one real one.

I needed a verification layer. Not a better prompt. Not a smarter model. A forcing function.

## The Pantheon Already Existed

My OpenCode setup at Neno runs on a set of specialized agents. Each one has a distinct role, a tuned prompt, and a specific job that it does and nothing else:

- **Prometheus** — reads the task, maps the codebase, writes an implementation plan
- **Vulkanus** — test-driven development, stubborn about coverage
- **Oracle** — architecture review, trade-off analysis, structural concerns
- **Zeus** — orchestrator, delegates to specialists, synthesizes results

The system works because each agent carries only what it needs in context. Zeus doesn't hold Vulkanus's test output — it gets a summary. Separated concerns, focused agents.

Code review was the missing piece. Every other phase of the dev cycle had a specialist. Review was still ad hoc.

## Meet Argos

The hundred-eyed giant of Greek myth. Set to watch over Io, he never slept — because while some eyes rested, others stayed open. Every angle, always covered.

I built Argos as an orchestrator that spawns seven specialist hunters, each examining the code from a single angle, running in parallel:

1. **Security** — injection, auth, data exposure
2. **Data Integrity** — transaction boundaries, constraint violations, edge cases in persistence
3. **Simplicity** — unnecessary complexity, abstractions that add weight without value
4. **Test Coverage** — gaps in the test suite, untested edge cases, missing failure paths
5. **Comments** — misleading docs, outdated comments, missing context on non-obvious logic
6. **Performance** — N+1 queries, unindexed columns, unnecessary computation in hot paths
7. **Types** — type safety gaps, implicit any, coercions that hide bugs

Seven eyes. Each one focused. Each one running simultaneously.

## The Proof Rule

Hunters cannot just flag issues. They must write a failing test that proves the problem exists.

Not a passing test. Not a description of what a test might look like. A test that, when you run it right now against the current codebase, fails — because the bug is real and the code doesn't handle it yet.

No test = no finding. Auto-deleted.

If a hunter can't write the failing test, the finding doesn't make it to the review.

This is the forcing function that eliminates hallucinations. You can't write a failing test for a bug that doesn't exist. The act of requiring a test proof filters out everything the model invented, because invented bugs don't produce test failures — they produce tests that pass, which Argos immediately discards.

It shifts the model's job from "describe what might be wrong" to "prove what is wrong." That's a fundamentally different cognitive task, and LLMs handle it differently. They can't hallucinate their way through a failing test.

## Argos as Orchestrator

After all seven hunters finish, Argos does the following — in sequence:

1. **Collects** all proposed proof tests from every hunter
2. **Runs** them against the actual codebase
3. **Keeps** only the tests that fail (these are real bugs)
4. **Discards** every test that passes (hallucinations, false positives, model noise)
5. **Passes** the verified failing tests to Oracle for architecture implications
6. **Returns** a final review containing only findings with proof

Oracle's role at the end is important. A verified failing test tells you the bug is real. It doesn't tell you whether fixing it requires a one-line patch or a refactor of the service boundary. That's an architecture judgment, and Oracle is better at it than the security hunter.

The output is a structured review where every single finding has an attached failing test. You can run it yourself. It either fails or it doesn't. No interpretation required.

---

**Sidebar: The Gemini Temperature Bug**

I chose Gemini 2.5 Pro for Argos. Deliberate choice — I wanted the hunters to be on a different model from the rest of the stack (mostly Claude). External perspective, different training distribution, cross-checks against my primary toolchain.

First run: infinite thinking loop. Argos just... spun. No output. The model was thinking, elaborating, second-guessing, revising — and never stopping.

Cause: temperature 0.1.

At low temperature, Gemini collapses into a single reasoning path it can't escape. It finds a direction, commits to it hard, and the extended thinking mode amplifies this into a loop. It keeps refining the same thought instead of terminating.

Fix: temperature 1.0.

That was it. One parameter. The model snapped out of it immediately on the next run and produced clean output.

This isn't intuitive — you'd think lower temperature means more reliable, more deterministic behavior. For most models, it does. For Gemini with extended thinking enabled, it's a trap. High temperature gives the thinking process enough variance to actually reach a conclusion and stop.

Different models have different failure modes. Document them when you find them.

---

## First Real Run

I pointed Argos at a PR in the Neno reconciliation service. The PR touched transaction matching logic — the code that links bank transactions to invoice records.

Two findings survived the proof filter.

**Finding 1 (yellow): Empty `invoiceNumber` collapses the SQL filter**

The query used `ILIKE '%' || invoiceNumber || '%'` to find matching invoices. When `invoiceNumber` is an empty string, this becomes `ILIKE '%%'` — which matches every row in the table. Reconciliation score inflation. Any transaction with an empty invoice number would match every invoice record and return an artificially high confidence score.

The failing test: create a transaction with `invoiceNumber: ''`, run the matcher, assert that it returns zero matches. It returned hundreds. Test failed. Bug confirmed.

**Finding 2 (red): `bankTransactionId` not scoped to workspace**

The query looked up bank transactions by ID without filtering by workspace. In a multi-tenant environment, this means a user could provide a `bankTransactionId` belonging to another tenant's data and the query would return it. Cross-tenant data leak.

The failing test: create a transaction in workspace A, attempt to look it up from a workspace B context, assert that the lookup returns empty. It returned the transaction. Test failed. Bug confirmed.

Both real. Both in production code. Both with failing tests attached.

My teammate's reaction in the GitHub PR: *"Great catch Argus!"*

Close enough.

## 1.5 days to build. Planning to open-source the approach.

The architecture is simple enough that it fits in a single agent config file and a few prompt templates. The insight — requiring proof tests as a gating mechanism — is the part that transfers to any multi-agent review setup, regardless of what orchestration framework you're using.

If you're running AI code review and you're not requiring the model to prove its findings, you're just adding noise to your process. More sophisticated noise than `# TODO: fix this`, but noise.

Does the "proof test" requirement change how you think about using LLMs for other kinds of analysis? I'm starting to think the pattern generalizes beyond code review.
