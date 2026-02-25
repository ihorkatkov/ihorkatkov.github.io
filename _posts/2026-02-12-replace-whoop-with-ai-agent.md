---
layout: post
title: "I Canceled My Whoop Subscription and Built the Same Thing With an AI Agent"
date: 2026-02-12
author: "Ihor Katkov"
tags: [AI, health, automation, openclaw]
---

Tuesday morning, Feb 12. My phone buzzes at 7:30 AM with the daily briefing from my AI agent:

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

Think of it like code review. The diff is already there — the health data. What you need is something that understands the context: your baselines, your schedule, your goals. That tells you what it means and what to do next. That's an AI agent.

A year earlier I'd have gotten a score from Whoop. But Whoop didn't know my calendar. It couldn't tell me *when* to train, only that I *could*. And every metric it tracked, my Apple Watch was already collecting.

So I built my own interpretation layer. Using [OpenClaw](https://github.com/openclaw/openclaw) and Apple Health data, I get a personalized morning briefing, daytime stress alerts, and calendar-aware workout suggestions — from my Apple Watch, processed by an agent that knows my baselines and my schedule. This post is exactly how to replicate it.

## Two Years With Whoop

Whoop is a great product — I used it for two years and it changed how I understand my body. That's exactly why I canceled it.

The first two months I was obsessed. I tracked diet, journaled workouts, optimized for green recovery days. It felt excessive — but it was teaching me counterintuitive things. I had to walk more, not less, to recover. Go to bed earlier. I designed a specific gym routine that left me feeling better the *next* day instead of wrecked. Lessons I never would have found without the data.

After two years, I'd distilled everything into four rules: **sleep better, manage stress, actively restore, train.** Those rules are mine now. I don't need the app to remind me of them.

What I couldn't replace was something Whoop never had: active intelligence.

Whoop doesn't know my calendar. It can't see that I have back-to-back meetings until 18:00 and suggest training at 14:00 instead. It can't connect my morning HRV dip to my afternoon workload and recommend cutting a meeting. Its journal tracks what happened — but it doesn't shape what happens next.

You don't need a separate diary when you have an autonomous agent that already knows your calendar, your emails, your workload. An agent fine-tuned *specifically for you* — not for a generic user.

That's what I built.

## The Framework

Three parts:

**1. Data collection (automated)**
- Apple Watch tracks everything 24/7
- Health Auto Export app sends data to OpenClaw webhook every 3 hours
- Data stored as JSON files for the agent to analyze

**2. Morning briefing (cron job)**
- Agent reads latest health data at 7:30 AM
- Compares with personal baselines (resting HR ~52, HRV ~120, sleep >7h for me)
- Checks today's calendar
- Generates recovery assessment + workout recommendation

**3. Stress sentinel (heartbeat)**
- Every 30 minutes, agent checks for stress signals
- Alerts only if heart rate is elevated *and* HRV is low — real stress, not noise
- Frames through essentialism: "You have 3 meetings left today — worth skipping one?"

The agent doesn't report numbers. It knows my calendar, my baselines, my preferences. Coaching, not dashboards.

## Implementation: Building the System

### Prerequisites

- Apple Watch (any recent model with HR/HRV tracking)
- iPhone with [Health Auto Export](https://apps.apple.com/app/health-auto-export/id1115567461) app (~$5 one-time)
- OpenClaw running in Docker ([quick setup here](https://github.com/ihorkatkov/openclaw-agent-bootstrap))
- OpenClaw instance accessible via static IP or Tailscale
- ~30 minutes to set up

### Step 1: The Webhook Server

Health Auto Export can POST JSON to any HTTP endpoint. I built a simple Node.js webhook server inside the OpenClaw workspace.

Create `workspace/health/webhook-server.js`:

```javascript
const http = require('http');
const fs = require('fs');
const path = require('path');

const PORT = process.env.HEALTH_WEBHOOK_PORT || 8090;
const DATA_DIR = path.join(__dirname, 'data');

if (!fs.existsSync(DATA_DIR)) fs.mkdirSync(DATA_DIR, { recursive: true });

const server = http.createServer((req, res) => {
  // Health check
  if (req.method === 'GET' && req.url === '/health') {
    res.writeHead(200, { 'Content-Type': 'application/json' });
    return res.end(JSON.stringify({ status: 'ok', timestamp: new Date().toISOString() }));
  }

  // Receive health data
  if (req.method === 'POST' && (req.url === '/health-data' || req.url === '/api/data')) {
    let body = '';
    req.on('data', chunk => body += chunk);
    req.on('end', () => {
      try {
        // Strip BOM (Health Auto Export sometimes adds it)
        const cleaned = body.replace(/^\uFEFF/, '').trim();
        const data = JSON.parse(cleaned);
        
        const ts = new Date().toISOString().replace(/[:.]/g, '-');
        const filename = `health-${ts}.json`;
        fs.writeFileSync(path.join(DATA_DIR, filename), JSON.stringify(data, null, 2));
        
        // Daily log for time-series analysis
        const today = new Date().toISOString().slice(0, 10);
        fs.appendFileSync(
          path.join(DATA_DIR, `daily-${today}.jsonl`),
          JSON.stringify({ received: new Date().toISOString(), ...data }) + '\n'
        );
        
        console.log(`Received health data → ${filename} (${body.length} bytes)`);
        res.writeHead(200, { 'Content-Type': 'application/json' });
        res.end(JSON.stringify({ status: 'ok', file: filename }));
      } catch (e) {
        console.error('Parse error:', e.message);
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

server.listen(PORT, '0.0.0.0', () => {
  console.log(`Health webhook listening on :${PORT}`);
});
```

The error handling matters. Health Auto Export sometimes sends a UTF-8 BOM character at the start of JSON, which breaks parsing. The `body.replace(/^\uFEFF/, '')` line strips it. Trust but verify — I learned this the hard way after the first export failed silently.

Create a startup script at `workspace/health/start-server.sh`:

```bash
#!/bin/bash
SCRIPT_DIR="$(cd "$(dirname "$0")" && pwd)"
mkdir -p "$SCRIPT_DIR/data"
pkill -f "node.*webhook-server.js" 2>/dev/null || true
sleep 1
cd "$SCRIPT_DIR"
nohup node webhook-server.js > "$SCRIPT_DIR/server.log" 2>&1 &
echo "Health webhook started (PID $!) on port ${HEALTH_WEBHOOK_PORT:-8090}"
```

Expose port 8090 in your `docker-compose.yml` and auto-start:

```yaml
services:
  gateway:
    ports:
      - "18789:18789"
      - "8090:8090"
    command:
      - sh
      - -c
      - |
        /home/node/.openclaw/workspace/health/start-server.sh &
        node dist/index.js gateway --bind lan --port 18789
```

Restart and verify:

```bash
docker-compose restart
curl http://localhost:8090/health
# {"status":"ok","timestamp":"..."}
```

### Step 2: Configure Health Auto Export

Install [Health Auto Export](https://apps.apple.com/app/health-auto-export/id1115567461) on your iPhone ($5 one-time).

Open the app → **Automations** → **Add Automation**:

- **URL**: `http://YOUR_OPENCLAW_IP:8090/health-data` (Tailscale recommended: `http://100.x.x.x:8090/health-data`)
- **Interval**: Every 3 hours
- **Metrics**: HR, Resting HR, HRV, SpO2, Sleep Analysis, Respiratory Rate, Wrist Temperature, Step Count, Active Energy, Exercise Minutes

Your iPhone must reach the webhook. Tailscale is the cleanest solution for mobile → home server.

### Step 3: Know Your Baselines

Generic thresholds don't work. My resting HR of 49 bpm is excellent for me, but might read as worrying for someone else. Track for 1-2 weeks, then set your baselines. Mine (from Whoop historical data):

| Metric | My Baseline | Alert Threshold |
|--------|-------------|-----------------|
| Resting HR | 52 bpm | > 60 bpm |
| HRV (SDNN) | 120 ms | < 80 ms |
| SpO2 | 97% | < 95% |
| Sleep | 7.5h | < 7h |

### Step 4: Morning Briefing (The Core)

Every morning at 7:30 AM, the agent reads my health data, checks my calendar, and gives me a recovery score + workout recommendation. Here's the cron prompt:

```
MORNING BRIEFING. Do ALL of these:

1. SLEEP & HEALTH: Read latest health data from workspace/health/data/
   Extract: sleep (totalSleep, deep, rem, core, awake), resting_heart_rate,
   heart_rate_variability, blood_oxygen_saturation, respiratory_rate,
   apple_sleeping_wrist_temperature.
   Baselines: resting HR ~52 (alert >60), HRV ~120 (alert <80), sleep >7h.

2. CALENDAR: Check today's events.

3. RECOVERY SCORE: 🟢 Good / 🟡 Moderate / 🔴 Poor (rest day).

4. WORKOUT SUGGESTION: Type + time window that fits the calendar + duration.

5. ESSENTIALISM CHECK: If calendar is overloaded + recovery is poor, say it.

Be direct, personal, actionable.
```

Whoop gives a score. My agent tells me when to train and what to do.

### Step 5: Stress Sentinel

The sentinel runs every 30 minutes. Most of the time — silence. When something is off:

```
⚠️ HR elevated for the past hour (avg 95 bpm at rest).
HRV dropped to 65 ms.

You have 3 meetings left today — worth dropping one?

Essentialism: less, but better.
```

I canceled one of the meetings. HRV recovered within an hour.

### Step 6: Calendar-Aware Workout Suggestions

| Recovery | Calendar Gap | Suggestion |
|----------|-------------|------------|
| 🟢 High HRV, low resting HR | 1.5h+ free | Strength or HIIT |
| 🟡 Normal HRV | 1h free | Moderate cardio, padel |
| 🔴 Low HRV, high resting HR | Any | Mobility or rest |
| Any | No gaps | Skip today |

## Case Study: A Week of Use

**Monday**: HRV 131 ms. Suggested strength training at 14:00. Did it.

**Wednesday**: HRV 95 ms, back-to-back meetings. Agent suggested 20min mobility. I ignored it, went to padel anyway. Played poorly, felt exhausted after. The agent was right.

**Friday**: HRV 68 ms, resting HR 58 bpm. "Rest day." I listened. By Saturday, HRV back to 125 ms.

## Learnings

**Baselines are everything.** "HRV below 50 is bad" means nothing if your baseline is 80 vs 120. The agent needs YOUR numbers.

**Calendar integration makes the difference.** Whoop didn't know my schedule. My agent does. That's why it says "train at 14:00" instead of "today is a good day to train."

**Silence is golden.** The sentinel only alerts when something is wrong. Most health apps drown you in notifications. This one doesn't.

**What didn't work at first:**
- HRV/resting HR lag 2-3 days in Apple Health. I shifted focus to sleep quality and wrist temperature, which export immediately.
- Network: iPhone couldn't reach the webhook away from home. Tailscale fixed it.
- BOM parsing: Health Auto Export added a BOM. Webhook crashed silently. Added `body.replace(/^\uFEFF/, '')`. Always check your logs.

## Privacy & Security

Your health data doesn't leave your private network. Here's how I handle it:

- **Private network only** — the webhook is accessible via Tailscale, not the public internet. No cloud, no SaaS, no third-party storage.
- **Encryption at rest** — enable full-disk encryption on your server (FileVault on Mac, LUKS on Linux). The JSON files contain real biometrics.
- **Retention policy** — I keep 30 days of exports. Old files are deleted automatically. Your health history shouldn't accumulate indefinitely.
- **What goes to the LLM** — the agent sends parsed metrics (numbers, dates), not raw JSON dumps. Telegram briefings are summaries, not raw biometric data.
- **What stays local** — all raw files, all conversation history, all baselines. The only thing that leaves is the prompt to the LLM API.

If you're self-hosting an LLM (Ollama, local Llama), nothing leaves your machine at all.

## Architecture Reference

```
Apple Watch → iPhone (Health app)
                ↓
    Health Auto Export (automated, every 3h)
                ↓
        HTTP POST JSON
                ↓
    OpenClaw webhook (:8090) — private network / Tailscale
                ↓
        JSON files on disk (30-day retention)
                ↓
    Cron jobs parse + analyze
                ↓
    Summaries → LLM API → Morning briefing / alerts → Telegram
```

File structure:

```
workspace/
└── health/
    ├── webhook-server.js    # HTTP server
    ├── start-server.sh      # Auto-start
    └── data/
        ├── health-*.json    # Individual exports (30-day retention)
        └── daily-*.jsonl    # Daily aggregated log
```

## Where This Goes Next

This is version 1. What I'm building toward:

**Multi-signal fatigue model** — instead of single-point HRV, track the 7-day trend. A single low HRV reading is noise. Three consecutive days below baseline is a signal worth acting on.

**Context fusion** — merge health signals with other life signals: calendar density (how many meetings?), travel (timezone shifts), late meals, alcohol. The agent knows my calendar already. Adding the rest makes the model richer.

**Interventions library** — instead of "rest day", the agent suggests a specific protocol: 20min morning walk, sunlight before 9am, caffeine cutoff at 2pm, drop the 4pm meeting. Prescriptive, not descriptive.

**Feedback loop** — after each recommendation, the agent asks "did that help?" in the evening. Over time, it learns which interventions actually move my HRV, not just which ones are theoretically correct.

**Autonomy ladder** — right now the agent advises. Next: draft a weekly training plan. Then: schedule workouts in the calendar. Then: notify my training partner automatically when I'm in a recovery week. Each step is opt-in, each step requires me to verify the agent has enough context to act reliably.

The goal isn't automation. It's augmentation — an agent that knows me well enough to be right most of the time, and knows when to ask.

## Final Thoughts

Whoop gave me the rules. My agent applies them.

Two years of data distilled to four principles: sleep better, manage stress, actively restore, train. Once you have the rules internalized, you don't need the teacher. What you need is something that applies those rules to your actual life — your calendar, your specific Tuesday, your dinner at 19:00.

That's not a $30/month subscription. That's an agent that knows you.

$5 (Health Auto Export app) + an hour of setup. That's the whole cost.

---

**Get the full setup:**
→ [openclaw-agent-bootstrap](https://github.com/ihorkatkov/openclaw-agent-bootstrap) — docker-compose, webhook server, cron prompts, Health Auto Export config, baselines template
→ Built on [OpenClaw](https://github.com/openclaw/openclaw) — open-source agent framework

I'm building the agentic personal OS in public — one component per week.  
Follow [@ihor_katkov](https://x.com/ihor_katkov) to catch the next build.
