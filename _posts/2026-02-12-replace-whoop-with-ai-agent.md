---
layout: post
title: "Dashboards Are Dead: I Built a Calendar-Aware Health Agent with OpenClaw"
date: 2026-02-12
author: "Ihor Katkov"
tags: [AI, health, agents, openclaw, autonomous-agents]
---

**Time to implement:** 30–60 min · **Cost:** $5 one-time · **Stack:** OpenClaw, Apple Watch, Health Auto Export, Tailscale

---

Tuesday morning, Feb 12. My phone buzzes at 7:30 AM:

```
🌅 Morning Briefing — Feb 12

😴 Sleep: 7.62h (deep 0.89h, REM 1.76h) ✅
❤️ Resting HR: 49 bpm (baseline 52) — excellent
📊 HRV: 131 ms (baseline 120) — recovery above normal
🩸 SpO2: 99%

Recovery: 🟢 Excellent — good day for intensity

📅 Today: Dutch lesson 12:00, WeFact 16:00, OpenClaw Builders 17:00
🏋️ Training window: 14:00-15:30
   Recommendation: strength training 45-60 min
```

I'm out of bed before I've opened anything else.

This didn't come from Whoop. It came from an agent I built on top of my Apple Watch data — one that knows my calendar, my baselines, and my schedule.

Think of it like code review. The diff is already there (the health data). What you need is something that understands the context — your baselines, your schedule, your goals — to tell you what it means and what to do. That's an AI agent.

---

## Why this matters (if you're building products)

This is a reference architecture for agentic product layers: **ingest → normalize → baseline → decide → notify.**

The moat in this pattern isn't data collection — it's **proprietary context + personalized decisioning**. Wearables are already commoditized. The agent layer on top of them is not. And unlike a SaaS product, the context your agent accumulates about you compounds over time — it's yours, it's local, and no competitor can copy it.

Whoop, Oura, Apple Health all give you dashboards. None of them know your calendar, your workload, or your goals. The interpretation layer — the thing that takes your data and maps it to your actual day — is wide open. That's the product.

This post shows exactly how I built it, including the architecture, the prompts, and the pitfalls.

---

**Contents**
- [Two Years With Whoop (and why I canceled)](#two-years-with-whoop)
- [The architecture](#the-architecture)
- [Why OpenClaw](#why-openclaw)
- [Setup: step by step](#setup-step-by-step)
- [The prompts](#the-prompts)
- [Comparison: Whoop vs the agent](#comparison)
- [Lessons learned](#lessons-learned)
- [Where this goes next](#where-this-goes-next)
- [Privacy & security](#privacy--security)

---

## Two Years With Whoop

Whoop is a great product. I used it for two years and it taught me things about my body that I couldn't have learned otherwise.

The first two months I was obsessed — tracking diet, journaling workouts, chasing green recovery days. It felt like too much. But the data was teaching me counterintuitive things: walk more to recover, go to bed earlier, design gym routines that leave you better the *next* day, not wrecked. I never would have found those patterns on my own.

After two years, I'd distilled it to four rules: **sleep better, manage stress, actively restore, train.** Those rules are mine. I don't need the app to remind me of them.

So I canceled.

What I couldn't replace was something Whoop never had: **active intelligence**. Whoop doesn't know my calendar. It gives me a score and a suggestion — but it can't say "your HRV is low *and* you have four meetings today, train tomorrow instead." Its journal tracks what happened. It doesn't proactively shape what happens next.

That gap is the product.

---

## The Architecture

Three components:

**1. Data collection (automated)**
Apple Watch → Health Auto Export → OpenClaw webhook → JSON files on disk

**2. Morning briefing**
Cron job at 7:30 AM → reads health data → checks calendar → generates recovery score + workout recommendation → Telegram

**3. Stress sentinel**
Heartbeat every 30 min → checks HR + HRV → alerts only when something is actually wrong

```
Apple Watch → iPhone (Health app)
                ↓
    Health Auto Export (every 3h, automated)
                ↓
        HTTP POST JSON
                ↓
    OpenClaw webhook (:8090) ← Tailscale / private network
                ↓
        JSON files (30-day retention)
                ↓
    OpenClaw agent: parse + baseline + decide
                ↓
    Morning briefing + stress alerts → Telegram
```

The agent doesn't report numbers. It knows my calendar, my baselines, my preferences. Coaching, not dashboards.

---

## Why OpenClaw

I could have built this with a cron script and a curl call to the Claude API. I didn't — and the difference matters.

**Stateful workspace** — OpenClaw agents have a persistent workspace. Health files, baselines, context — all accessible across sessions. No database setup, no state management code.

**Scheduling built-in** — morning briefing, stress sentinel, cleanup jobs are all first-class cron jobs in the config. I didn't write a scheduler.

**Connectors** — calendar (via `gog` CLI), Telegram, email are all connected to the same agent. The morning briefing crosses sources naturally: health + calendar + workload in one prompt.

**Composable agents** — the briefing agent, stress sentinel, and cleanup agent are separate concerns. Each has its own prompt and schedule. No code required to compose them.

**Memory as first-class concept** — baselines, user preferences, and learned patterns live in workspace files the agent can read and update. Persists across restarts.

If you're building any kind of personal agent, OpenClaw is the platform — not a script host.

**The intelligence flywheel.** Generic health products collect your data but know nothing else about you. OpenClaw agents accumulate proprietary context: your calendar patterns, your baseline trends, your past decisions, your communication style, your goals. That context compounds. The agent that's been running for three months knows you better than one running for a week — and no generic product can replicate that, because that data is yours and it lives locally.

This is the moat that wearables can't copy. Not the sensors. The context.

---

## Setup: Step by Step

### Prerequisites

- Apple Watch (any recent model with HR/HRV tracking)
- iPhone with [Health Auto Export](https://apps.apple.com/app/health-auto-export/id1115567461) ($5 one-time)
- OpenClaw running in Docker ([bootstrap repo](https://github.com/ihorkatkov/openclaw-agent-bootstrap))
- Tailscale on both iPhone and server

### Step 1: The Webhook Server

Create `workspace/health/webhook-server.js`:

```javascript
const http = require('http');
const fs = require('fs');
const path = require('path');

const PORT = process.env.HEALTH_WEBHOOK_PORT || 8090;
const DATA_DIR = path.join(__dirname, 'data');

if (!fs.existsSync(DATA_DIR)) fs.mkdirSync(DATA_DIR, { recursive: true });

const server = http.createServer((req, res) => {
  if (req.method === 'GET' && req.url === '/health') {
    res.writeHead(200, { 'Content-Type': 'application/json' });
    return res.end(JSON.stringify({ status: 'ok', timestamp: new Date().toISOString() }));
  }

  if (req.method === 'POST' && (req.url === '/health-data' || req.url === '/api/data')) {
    let body = '';
    req.on('data', chunk => body += chunk);
    req.on('end', () => {
      try {
        const cleaned = body.replace(/^\uFEFF/, '').trim(); // Strip BOM
        const data = JSON.parse(cleaned);
        const ts = new Date().toISOString().replace(/[:.]/g, '-');
        const filename = `health-${ts}.json`;
        fs.writeFileSync(path.join(DATA_DIR, filename), JSON.stringify(data, null, 2));
        const today = new Date().toISOString().slice(0, 10);
        fs.appendFileSync(
          path.join(DATA_DIR, `daily-${today}.jsonl`),
          JSON.stringify({ received: new Date().toISOString(), ...data }) + '\n'
        );
        res.writeHead(200, { 'Content-Type': 'application/json' });
        res.end(JSON.stringify({ status: 'ok', file: filename }));
      } catch (e) {
        const ts = new Date().toISOString().replace(/[:.]/g, '-');
        fs.writeFileSync(path.join(DATA_DIR, `raw-${ts}.txt`), body);
        res.writeHead(400, { 'Content-Type': 'application/json' });
        res.end(JSON.stringify({ error: 'Invalid JSON' }));
      }
    });
    return;
  }
  res.writeHead(404);
  res.end('Not found');
});

server.listen(PORT, '0.0.0.0', () => console.log(`Health webhook :${PORT}`));
```

> ⚠️ **Pitfall: BOM** — Health Auto Export sometimes prefixes JSON with a UTF-8 BOM character (`\uFEFF`). Without the strip, your webhook silently saves unparseable files. The `body.replace(/^\uFEFF/, '')` line handles it.

Add to `docker-compose.yml`:

```yaml
ports:
  - "8090:8090"
command:
  - sh
  - -c
  - |
    /home/node/.openclaw/workspace/health/start-server.sh &
    node dist/index.js gateway --bind lan --port 18789
```

### Step 2: Configure Health Auto Export

Open app → **Automations** → **Add Automation**:
- **URL**: `http://YOUR_TAILSCALE_IP:8090/health-data`
- **Interval**: Every 3 hours
- **Metrics**: HR, Resting HR, HRV, SpO2, Sleep Analysis, Respiratory Rate, Wrist Temperature, Step Count, Active Energy

> ⚠️ **Pitfall: Apple Health lag** — HRV and resting HR are written to Apple Health with a 2-3 day delay in some configurations. Focus your morning briefing on sleep quality and wrist temperature, which export immediately. Use HRV for trend analysis, not daily decisions.

### Step 3: Set Your Baselines

Track for 1-2 weeks. Mine (from Whoop historical data):

| Metric | My Baseline | Alert Threshold |
|--------|-------------|-----------------|
| Resting HR | 52 bpm | > 60 bpm |
| HRV (SDNN) | 120 ms | < 80 ms |
| SpO2 | 97% | < 95% |
| Sleep | 7.5h | < 7h |

---

## The Prompts

### Morning Briefing

```
MORNING BRIEFING. Do ALL of these:

1. SLEEP & HEALTH: Read latest health data from workspace/health/data/
   (find most recent health-*.json). Extract: sleep (totalSleep, deep, rem,
   core, awake), resting_heart_rate, heart_rate_variability,
   blood_oxygen_saturation, apple_sleeping_wrist_temperature.
   Baselines: resting HR ~52 (alert >60), HRV ~120 (alert <80), sleep >7h.

2. CALENDAR: Check today's events. List the schedule.

3. RECOVERY SCORE: 🟢 Good / 🟡 Moderate / 🔴 Poor (rest day).

4. WORKOUT SUGGESTION: Type + time window that fits calendar + duration.

5. ESSENTIALISM CHECK: If calendar is overloaded + recovery is poor, say it.

Be direct, personal, actionable.
```

*Not medical advice — treat recovery scores as coaching heuristics. If metrics look consistently off, talk to a clinician.*

### Stress Sentinel (heartbeat, every 30 min)

```
Daytime check — stress signals only:
- Resting HR >60 = alert
- HRV <80 = alert (only if also elevated HR)
- No sleep stats during the day.
If stress indicators are high: frame through essentialism —
high stress = probably doing too much. Suggest one concrete action.
```

---

## Comparison

| Feature | Whoop | Apple Watch + OpenClaw |
|---------|-------|------------------------|
| Recovery score | ✅ | ✅ (derived) |
| Calendar-aware planning | ❌ | ✅ |
| Proactive suggestions | Limited | ✅ |
| Custom baselines | Partial | ✅ fully yours |
| Email / workload context | ❌ | ✅ |
| Data ownership | Vendor | You |
| Cost | $30/mo | ~$5 one-time + compute |

---

## Lessons Learned

**Baselines are everything.** "HRV below 50 is bad" means nothing if your baseline is 80 vs 120. The agent needs YOUR numbers.

**Calendar integration makes the difference.** Whoop can't say "train at 14:00." My agent can.

**Silence is golden.** The sentinel only alerts when something is actually wrong. No noise.

**Case study — Wednesday that went wrong:** HRV 95 ms (below my baseline), back-to-back meetings. Agent suggested 20min mobility. I ignored it, played padel. Played poorly, felt exhausted. Lesson: the agent was right. I should have listened.

---

## Where This Goes Next

**Multi-signal fatigue** — 7-day HRV trend instead of single-point readings. Three consecutive days below baseline is a signal. One bad reading is noise.

**Context fusion** — merge health with workload signals: calendar density, travel days, late meals. The agent already knows my calendar. Adding the rest makes the model richer.

**Interventions library** — instead of "rest day", prescribe: 20min walk, sunlight before 9am, caffeine cutoff at 2pm, drop one meeting. Specific, not vague.

**Feedback loop** — evening check-in: "did that help?" Over time the agent learns which interventions actually move my HRV.

**Autonomy ladder** — advise → draft weekly training plan → schedule workouts → notify training partner. Each step opt-in, each step requires trust earned.

The goal isn't automation. It's augmentation — an agent that knows me well enough to be right most of the time, and knows when to ask.

The longer this runs, the better it gets. Three months from now my agent will have seen patterns in my data that I haven't noticed yet. Six months from now it will understand my rhythms better than any wearable ever could — because it has context no wearable can access. That's the flywheel. That's the product.

---

## Privacy & Security

Your health data is sensitive. Here's how this setup handles it:

- **Private network only** — the webhook runs behind Tailscale. Not exposed to the public internet. No cloud, no third-party storage.
- **Encryption at rest** — enable full-disk encryption on your server (FileVault / LUKS). The JSON files contain real biometrics.
- **30-day retention** — old exports are deleted automatically. Health history shouldn't accumulate indefinitely.
- **What goes to the LLM** — parsed metrics (numbers, dates), not raw JSON dumps. Telegram briefings are summaries.
- **What stays local** — all raw files, conversation history, baselines. Only the prompt reaches the LLM API.
- **Self-hosted option** — run a local Ollama/Llama model and nothing leaves your machine at all.

---

## Final Thoughts

Whoop gave me the rules. My agent applies them.

Two years of data distilled to four principles: sleep better, manage stress, actively restore, train. Once those are internalized, you don't need the teacher. You need something that applies those rules to your actual Tuesday — your calendar, your dinner at 19:00, your four meetings.

That's not a subscription. That's an agent that knows you.

$5 and an hour. That's it.

---

**Get the full setup:**
→ [openclaw-agent-bootstrap](https://github.com/ihorkatkov/openclaw-agent-bootstrap) — docker-compose, webhook, cron prompts, Health Auto Export config, baselines template
→ Built on [OpenClaw](https://github.com/openclaw/openclaw) — open-source agent framework

I'm building the agentic personal OS in public — one component per week.  
Follow [@ihor_katkov](https://x.com/ihor_katkov) to catch the next build.
