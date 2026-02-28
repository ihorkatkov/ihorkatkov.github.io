---
layout: post
title: "Dashboards Are Dead: I Built a Calendar-Aware Health Agent with OpenClaw"
date: 2026-02-12
author: "Ihor Katkov"
description: "How I replaced my Whoop subscription with an AI health agent using OpenClaw and Apple Watch — calendar-aware recovery coaching for $5."
image: "" # TODO: add hero image before merge
tags: [ai-health-agent, autonomous-agents, apple-watch-automation, OpenClaw, health-automation, replace-whoop]
---

**⏱ 30–60 min to implement · 💵 $5 one-time · Stack: OpenClaw, Apple Watch, Health Auto Export, Tailscale**

---

Tuesday morning, Feb 12. My phone buzzes at 7:30 AM:

```
🌅 Morning Briefing — Feb 12

😴 Sleep: 7.62h (deep 0.89h, REM 1.76h) ✅
❤️ Resting HR: 49 bpm (baseline 52) — excellent
📊 HRV: 131 ms (baseline 120) — recovery above normal

Recovery: 🟢 Excellent — good day for intensity

📅 Today: Dutch lesson 12:00, WeFact 16:00, OpenClaw Builders 17:00
🏋️ Training window: 14:00-15:30 — strength training, 45–60 min
```

I'm out of bed before I've opened anything else.

Think of it like code review. The diff is already there — the health data. What you need is something that understands the context: your baselines, your schedule, your goals. That's an AI agent.

A year earlier I'd have gotten a score from Whoop. But Whoop didn't know my calendar. It couldn't tell me *when* to train, only that I *could*.

---

## Why this matters

This is a reference architecture for agentic product layers:

```
┌─────────────────────────────────────────────────────┐
│              AGENTIC LAYER PATTERN                   │
│                                                      │
│  Inputs      →  Interpretation  →  Actions          │
│                                                      │
│  sensors          baselines          schedule        │
│  calendar    →    heuristics    →    meeting cut     │
│  workload         context            intervention    │
│                        ↑                             │
│                   Feedback loop                      │
│              "Did that help?" → learns               │
└─────────────────────────────────────────────────────┘
```

The moat isn't data collection — it's **proprietary context + personalized decisioning**. Wearables are commoditized. The interpretation layer isn't. And unlike SaaS, the context your agent accumulates is yours, local, and compounds over time.

The full autonomy ladder: **advise → draft plan → schedule → act → learn.** This post covers the first two rungs. The rest follow naturally as trust builds.

---

## Contents

- [Two Years With Whoop](#two-years-with-whoop)
- [The architecture](#the-architecture)
- [Why OpenClaw](#why-openclaw)
- [Setup](#setup)
- [The prompts](#the-prompts)
- [Comparison](#comparison)
- [Founder lens](#founder-lens)
- [Lessons learned](#lessons-learned)
- [Where this goes next](#where-this-goes-next)
- [Privacy & security](#privacy--security)

---

## Two Years With Whoop

Whoop is a great product — I used it for two years and it taught me things about my body I couldn't have learned otherwise. That's exactly why I canceled it.

The first two months I was obsessed. Chasing green recovery days, tracking diet, logging workouts. It felt like too much — but the data was teaching me counterintuitive things: walk more to recover, go to bed earlier, design a gym routine that leaves you better the *next* day. I never would have found those patterns alone.

After two years, I had four rules: **sleep better, manage stress, actively restore, train.** Those are mine now. I don't need the app to remind me.

What I couldn't replace was something Whoop never had: active intelligence. It doesn't know my calendar. It can't connect my HRV dip to my afternoon workload and suggest cutting a meeting. Its journal tracks what happened — it doesn't shape what happens next.

That gap is the product.

---

## The Architecture

```
Apple Watch → Health app
                ↓
    Health Auto Export (every 3h)
                ↓
        HTTP POST JSON
                ↓
    OpenClaw webhook (:8090)   ← Tailscale / private only
                ↓
        JSON files on disk     ← 30-day retention
                ↓
    OpenClaw agent             ← reads calendar + health
                ↓
    Morning briefing + alerts → Telegram
```

Three components:

| Component | What it does |
|---|---|
| **Webhook server** | Receives Apple Watch data from Health Auto Export |
| **Morning briefing** | Cron at 7:30 AM — recovery score + workout window |
| **Stress sentinel** | Every 30 min — alerts only on combined HR spike + HRV drop |

---

## Why OpenClaw

Most people build agents as stateless chat prompts. That's not an agent — it's autocomplete. Agents need loops, memory, and boundaries. I wrote about why I built OpenClaw to solve exactly this in [The Lethal Trifecta and Why I Built OpenClaw](/2026/02/16/the-lethal-trifecta-and-why-i-built-openclaw).

OpenClaw gives you that:

| Capability | Why it matters here |
|---|---|
| **Long-running processes with state** | Baselines, context, and history persist across sessions |
| **Scheduled loops built-in** | Morning briefing and stress sentinel are first-class cron jobs |
| **Composable concerns** | Briefing agent and sentinel are separate — different prompts, different schedules |
| **Context fusion as workflow** | Calendar + health + email in one agent, one prompt |
| **Local-first by default** | Nothing leaves your network unless you configure it to |

**The intelligence flywheel.** The longer this runs, the better it gets. Three months in, the agent has seen patterns in your data that you haven't noticed yet. No SaaS product can replicate that — because that data is yours, and it lives locally.

---

## Setup

The full setup lives in the companion repo: **[github.com/ihorkatkov/health-agent](https://github.com/ihorkatkov/health-agent)**

The repo is written for agents, not humans. Point your OpenClaw agent at it:

> *"Set up the health agent from github.com/ihorkatkov/health-agent. Read AGENTS.md first, then follow CHECKLIST.md step by step. My Tailscale IP is [YOUR_IP]."*

The agent reads the spec and applies it to your setup. No copy-pasting.

**What's in the repo:**

| File | Purpose |
|---|---|
| `AGENTS.md` | Agent contract — how to read and execute |
| `CHECKLIST.md` | Sequential setup steps |
| `PROMPTS/morning-briefing.txt` | Ready-to-use cron prompt |
| `PROMPTS/stress-sentinel.txt` | Sentinel prompt |
| `TEMPLATES/webhook-server.js` | Webhook server (Node.js) |
| `TEMPLATES/baselines.json` | Baselines template — edit with your values |
| `TEMPLATES/retention-cron.sh` | 30-day cleanup script |
| `TESTS/smoke-test.sh` | Post-setup verification |
| `SECURITY.md` | Hardening guide |

---

## The Prompts

The prompts are what make this a coach, not a dashboard.

**Morning briefing** (excerpt from `PROMPTS/morning-briefing.txt`):
```
MORNING BRIEFING. Execute all sections:

1. SLEEP & HEALTH: read latest health data, compare to baselines
2. CALENDAR: check today's events
3. RECOVERY SCORE: 🟢 Good / 🟡 Moderate / 🔴 Poor
4. WORKOUT SUGGESTION: type + time window + duration
5. ESSENTIALISM CHECK: if overloaded + poor recovery, name one thing to cut

Be direct, personal, no filler.
```

**Stress sentinel** (excerpt from `PROMPTS/stress-sentinel.txt`):
```
If HR > threshold AND HRV < threshold: alert with one concrete suggestion.
Otherwise: SENTINEL_OK (silent, no message).
```

The sentinel is silent by default. That's intentional. No noise.

---

## Comparison

| Feature | Whoop | Apple Watch + OpenClaw |
|---|---|---|
| Recovery score | ✅ | ✅ derived |
| Calendar-aware planning | ❌ | ✅ |
| Proactive suggestions | Limited | ✅ |
| Custom baselines | Partial | ✅ fully yours |
| Email / workload context | ❌ | ✅ |
| Intelligence flywheel | ❌ | ✅ compounds over time |
| Data ownership | Vendor | You |
| Cost | $30/mo | ~$5 one-time |

---

## Founder Lens

Why this is a product wedge:

- **Data capture is commodity.** Apple Watch, Oura, Garmin — all collect the same signals. The hardware moat is gone.
- **Context fusion is moat.** No wearable knows your calendar, your workload, your inbox. Only your personal agent does.
- **Autonomy ladder drives retention.** Users who reach "advise" stay. Users who reach "schedule" become advocates.
- **Local-first reduces compliance drag.** No HIPAA scope, no SOC 2, no data processor agreements. The data never leaves the user's machine.
- **Agent templates become ecosystems.** Ship a spec repo like this one. Others fork it. The agent layer multiplies.

---

## Lessons Learned

**Baselines are everything.** "HRV below 50 is bad" means nothing if your baseline is 80 vs 120. The agent needs your numbers.

**Calendar integration makes the difference.** Whoop can't say "train at 14:00." My agent can.

**Silence is golden.** The sentinel only fires when something is actually wrong.

> ⚠️ **Pitfall: BOM** — Health Auto Export sometimes prefixes JSON with a UTF-8 BOM (`\uFEFF`). Without stripping it, exports parse silently as invalid. The webhook template handles this.

> ⚠️ **Pitfall: Apple Health lag** — HRV and resting HR write to Apple Health with a 2-3 day delay. Use sleep quality and wrist temperature for daily decisions; HRV for weekly trends.

**Wednesday that went wrong:** HRV 95 ms (below baseline), packed calendar. Agent suggested mobility. I went to padel anyway. Played poorly. The agent was right.

---

## Where This Goes Next

**Multi-signal fatigue model** — 7-day HRV trend instead of single readings. Three days below baseline is a signal; one bad night is noise.

**Context fusion** — merge calendar density, travel days, late meals. The agent already knows the calendar. The rest follows.

**Interventions library** — instead of "rest day": 20min walk, sunlight before 9am, caffeine cutoff at 2pm, drop the 4pm meeting.

**Feedback loop** — evening check-in: "did that help?" The agent learns which interventions move your HRV.

**Autonomy ladder** — advise → draft weekly training plan → schedule workouts → notify training partner. Each step opt-in.

The longer this runs, the better it gets. Six months from now this agent will know my rhythms better than any wearable — because it has context no wearable can access. That's the flywheel. That's the product.

---

## Privacy & Security

- **Private network only** — webhook runs behind Tailscale, not the public internet
- **Encryption at rest** — enable full-disk encryption on your server
- **30-day retention** — automated cleanup via `retention-cron.sh`
- **What goes to the LLM** — parsed summaries, not raw JSON; no biometric dumps in messages
- **Self-hosted option** — run Ollama locally and nothing leaves your machine

Full hardening guide: `SECURITY.md` in the repo.

*Health disclaimer: recovery scores are coaching heuristics, not medical advice. If metrics look consistently off, talk to a clinician.*

---

## Final Thoughts

Whoop gave me the rules. My agent applies them.

Two years of data distilled to four principles: sleep better, manage stress, actively restore, train. Once internalized, you don't need the teacher. You need something that applies those rules to your actual Tuesday — your calendar, your dinner at 19:00, your four meetings.

That's not a subscription. That's an agent that knows you.

$5 and an hour. That's it.

---

**If you're building agentic products, steal this pattern.**  
**If you're building OpenClaw workflows, fork the repo.**  
**If you care about local-first autonomy, follow along — I'm shipping one component per week.**

→ [github.com/ihorkatkov/health-agent](https://github.com/ihorkatkov/health-agent) — agent-executable spec: webhook + prompts + baselines + retention + smoke tests  
→ Built on [OpenClaw](https://github.com/openclaw/openclaw)  
→ [@ihor_katkov](https://x.com/ihor_katkov) on X
