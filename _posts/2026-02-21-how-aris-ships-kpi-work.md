---
layout: post
title: "Autonomous AI Agents in Practice: How Aris Ships KPI Work via Heartbeat + Oracle"
date: 2026-02-21
author: "Ihor Katkov"
tags: [ai, agents, autonomous-agents, openclaw, automation, productivity, kpi]
description: "A real case study of an autonomous AI agent workflow: heartbeat priorities, Oracle alignment, and KPI-first execution that shipped a concrete content artifact without a direct prompt."
image: /assets/img/aris-kpi-loop-hero.jpg
---

![Autonomous AI Agents in Practice: Heartbeat + Oracle KPI Loop](/assets/img/aris-kpi-loop-hero.jpg){: .img-fluid .rounded .z-depth-1 }

Saturday morning, before I opened my editor, Aris had already shipped a Typefully-ready KPI draft.

I didn’t prompt “write me a post” — the heartbeat policy triggered the work. We had configured strict heartbeat priorities, and rule #1 was simple: KPI output first.

This is a real autonomous AI agent case study: Aris checked context, consulted Oracle, confirmed there was no urgent calendar pressure, and shipped a tangible KPI artifact based on our HackerNoon Top Story moment.

I only reacted after the fact.

That’s the point.

## The Setup

I run Aris on [OpenClaw](https://github.com/openclaw/openclaw) with explicit heartbeat instructions and a strict priority order. The heartbeat doesn’t just “notify.” It executes a decision loop:

1. Quick checks (calendar, inbox, context)
2. Priority 1: KPI content work
3. Priority 2: backlog execution
4. Oracle alignment check

This is not generic assistant behavior. It’s an operating model.

A simplified heartbeat policy artifact:

```yaml
priorityOrder:
  - phase1_quick_checks
  - kpi_content_first
  - beads_only_if_no_kpi
  - oracle_alignment_check
kpiRule:
  requireArtifactEachCycle: true
```

## What Happened in Practice

In one morning window, this sequence happened:

- Oracle recommended KPI-first focus and warned against maintenance drift
- Aris generated a concrete content artifact (Top Story + security lesson)
- The draft was saved and then polished into a publish-ready variant
- I reviewed and gave one strategic edit for the hook

The hook change was important:

> Fully autonomous agents can’t be made 100% safe — that’s impossible today.

Then we kept the rest practical: blast radius control, permission model, architecture before autonomy (same principles I described in [Living With the Lethal Trifecta](/blog/2026/the-lethal-trifecta-and-why-i-built-openclaw/)).

## Why This Matters

Most “AI productivity” demos stop at chat quality.

What matters is the closed loop: **priority → action → artifact**.

No hero prompt.
No manual orchestration per step.
No “assistant, remind me later” theater.

Just policy + context + execution.

## The Oracle Role (and Why It Exists)

I added Oracle as a second perspective to reduce local blind spots.

Aris handles execution. Oracle pressure-tests direction.

That separation matters because agents drift toward easy maintenance work by default: cleanup, docs, small refactors that feel productive but don’t move KPI.

Oracle keeps the system honest:

- Is current focus aligned with top objective?
- Is this work producing visible output?
- Should we pivot, or keep shipping?

## Guardrails That Make This Work

Autonomy without boundaries scales mistakes faster.

The system works because constraints are explicit:

- KPI-first priority order in heartbeat
- Human approval for consequential external actions
- Tool policies and clear capability boundaries
- Logged execution trail for review

This is the same pattern I use everywhere with agents:

**tight scope + clear objective + visible artifact**.

## What I’d Recommend If You Want to Replicate It

Start with one loop, not ten.

- Define one primary objective (for me: KPI content output)
- Encode a hard priority order
- Require one concrete artifact per cycle
- Add a second agent/reviewer for strategic alignment
- Review outcomes weekly and cut any “busywork automation”

Architecture beats model choice for this kind of autonomous work.

A focused system with clear policies on a solid model will outperform a chaotic autonomous setup on the best model.

## Final Thought

People ask me which model to use for autonomous agents.

Wrong first question.

Ask this instead:

**What does your agent ship when you’re not asking it directly?**

If the answer is “real KPI artifacts,” you’re onto something.
If the answer is “more internal busywork,” redesign the loop.
