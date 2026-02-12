---
layout: post
title: "I Canceled My Whoop Subscription and Built the Same Thing With an AI Agent"
date: 2026-02-12
author: "Ihor Katkov"
tags: [AI, health, automation, openclaw]
---

## Introduction

For over a year, I paid $30/month for a Whoop subscription. Every morning, I'd check my recovery score, read the sleep analysis, and get workout recommendations based on my HRV and resting heart rate. It was genuinely useful — until I realized I was paying for an interpretation layer on top of data my Apple Watch already collects.

The hardware wasn't the value. The value was the morning briefing: "Your recovery is 76%, you had good REM sleep, today is a good day for intensity." That contextual coaching that tells you not just what happened, but what to do about it.

So I built that layer myself. Using OpenClaw (an open-source AI agent framework I've been working with) and Apple Health data, I now get a personalized morning health briefing, real-time stress monitoring, and calendar-aware workout suggestions — all from my Apple Watch, processed by an AI that knows my baselines and my schedule.

Here's the journey, the framework, and exactly how to replicate it.

## Background: The Whoop Realization

I started using Whoop because I wanted objective recovery metrics. As someone who trains regularly (mostly strength work and padel), I needed to know when to push and when to back off. Whoop delivered that — but at a cost.

The breaking point came when I looked at the data sources. Everything Whoop measured — heart rate variability, resting heart rate, sleep stages, respiratory rate, wrist temperature — my Apple Watch already tracked. The only difference? Whoop gave me a score and a recommendation. Apple gave me raw numbers in the Health app.

That's when I realized: I don't need better sensors. I need a better interpretation layer.

Think about it like code review. The diff is already there (the health data). What you need is someone who understands the context (your baselines, your schedule, your goals) to tell you what it means and what to do next. That's exactly what an AI agent can do.

## The Framework

Here's the three-part system I built:

**1. Data collection (automated)**
- Apple Watch tracks everything 24/7
- Health Auto Export app sends data to OpenClaw webhook every 3 hours
- Data stored as JSON files for the agent to analyze

**2. Morning briefing (cron job)**
- Agent reads latest health data at 7:30 AM
- Compares with your baselines (resting HR ~52, HRV ~120, sleep >7h for me)
- Checks today's calendar
- Generates recovery assessment + workout recommendation

**3. Daytime monitoring (heartbeat)**
- Every 30 minutes, agent checks for stress signals
- Alerts only if heart rate elevated + HRV low (real stress, not noise)
- Frames through essentialism: "You have 3 meetings left today — worth skipping one?"

The key insight: the agent doesn't just report numbers. It knows my calendar, my baselines, and my preferences. It gives me coaching, not dashboards.

## Implementation: Building the System

### Prerequisites

- Apple Watch (any recent model with HR/HRV tracking)
- iPhone with [Health Auto Export](https://apps.apple.com/app/health-auto-export/id1115567461) app (~$5 one-time)
- OpenClaw running in Docker ([quick setup here](https://github.com/ihorkatkov/openclaw-agent-bootstrap))
- OpenClaw instance accessible via static IP or Tailscale
- ~30 minutes to set up

### Step 1: The Webhook Server

First, I needed a way to receive health data from my iPhone. Health Auto Export can POST JSON to any HTTP endpoint, so I built a simple Node.js webhook server inside my OpenClaw workspace.

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
        // Save raw body for debugging
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

The error handling here is important. Health Auto Export sometimes sends a UTF-8 BOM character at the start of JSON, which breaks parsing. The `body.replace(/^\uFEFF/, '')` line strips it. Trust but verify — I learned this the hard way after the first export failed silently.

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

```bash
chmod +x workspace/health/start-server.sh
```

Expose port 8090 in your `docker-compose.yml`:

```yaml
services:
  gateway:
    # ... existing config ...
    ports:
      - "18789:18789"
      - "18790:18790"
      - "8090:8090"  # Health webhook
```

Auto-start the webhook with your agent by updating the docker-compose `command`:

```yaml
command:
  - sh
  - -c
  - |
    /home/node/.openclaw/workspace/health/start-server.sh &
    node dist/index.js gateway --bind lan --port 18789
```

Restart your container:

```bash
docker-compose restart
```

Verify it's running:

```bash
curl http://localhost:8090/health
# Should return: {"status":"ok","timestamp":"..."}
```

### Step 2: Configure Health Auto Export

Install [Health Auto Export](https://apps.apple.com/app/health-auto-export/id1115567461) on your iPhone. This is the only paid component ($5 one-time).

Open the app → **Automations** → **Add Automation**:

- **URL**: `http://YOUR_OPENCLAW_IP:8090/health-data`
  - If using Tailscale (recommended): `http://100.x.x.x:8090/health-data`
  - If on local network: your machine's LAN IP
- **Interval**: Every 3 hours (I use 3h, but 6h works too)
- **Metrics to export**:
  - ✅ Heart Rate
  - ✅ Resting Heart Rate
  - ✅ Heart Rate Variability (HRV)
  - ✅ Blood Oxygen Saturation (SpO2)
  - ✅ Sleep Analysis
  - ✅ Respiratory Rate
  - ✅ Wrist Temperature
  - ✅ Step Count
  - ✅ Active Energy
  - ✅ Exercise Minutes

Save and run once manually to test.

**Important**: Your iPhone must be able to reach the webhook URL. If OpenClaw runs at home, your phone needs to be on the same network or use Tailscale. I use Tailscale — it's the easiest setup for mobile → server communication.

Verify data is arriving:

```bash
ls workspace/health/data/
# Should show: health-2026-02-12T09-37-46-453Z.json
```

### Step 3: Know Your Baselines

Before the AI can tell you "your recovery is low," it needs to know what's normal for you. This is critical — generic thresholds don't work. My resting heart rate of 49 bpm is excellent for me, but might be worrying for someone else.

Track for 1-2 weeks, then calculate your personal baselines. My baselines (I had Whoop historical data to reference):

| Metric | My Baseline | Alert Threshold |
|--------|------------|-----------------|
| Resting HR | 52 bpm | > 60 bpm |
| HRV (SDNN) | 120 ms | < 80 ms |
| SpO2 | 97% | < 95% |
| Sleep | 7.5h | < 7h |

If you don't have Whoop history, just track Apple Watch data for 2 weeks and ask your agent to calculate averages. It can read the JSON files and compute the stats.

### Step 4: Morning Briefing (The Core)

This is where the system comes alive. Every morning at 7:30 AM, the agent reads my latest health data, compares it with my baselines, checks my calendar, and gives me a recovery score + workout recommendation.

In OpenClaw, I configured a cron job. Here's the prompt I use:

```
MORNING BRIEFING. Do ALL of these:

1. SLEEP & HEALTH: Read latest health data from workspace/health/data/
   (find most recent health-*.json). Extract: sleep (totalSleep, deep, rem,
   core, awake, sleepStart, sleepEnd), resting_heart_rate, heart_rate_variability,
   blood_oxygen_saturation, respiratory_rate, apple_sleeping_wrist_temperature.
   Compare with baselines: resting HR ~52 (alert >60), HRV ~120 (alert <80),
   sleep >7h.

2. CALENDAR: Check today's events. List the schedule.

3. RECOVERY SCORE: Based on sleep + HRV + resting HR, rate recovery:
   🟢 Good (train hard), 🟡 Moderate (light training ok), 🔴 Poor (rest day).

4. WORKOUT SUGGESTION: Based on recovery + calendar gaps, suggest:
   - Type (strength/cardio/mobility/rest)
   - Time window that fits the calendar
   - Duration

5. ESSENTIALISM CHECK: If calendar is overloaded + recovery is poor, say it.
   Recommend what to defer.

Be direct, personal, actionable.
```

Here's an actual morning briefing I received today:

```
🌅 Ранковий брифінг — 12 лютого

😴 Сон: 7.62 год (deep 0.89h, REM 1.76h) ✅
❤️ Resting HR: 49 bpm (baseline 52) — відмінно
📊 HRV: 131 ms (baseline 120) — recovery вище норми
🩸 SpO2: 99%

Recovery: 🟢 Відмінна — можна тренуватись інтенсивно

📅 Сьогодні: Dutch 12:00, WeFact 16:00, OpenClaw Builders 17:00
🏋️ Вікно для тренування: 14:00-15:30 (між Dutch і WeFact)
   Рекомендація: strength training 45-60 хв

💡 Calendar load помірний. Recovery дозволяє повну активність.
```

This is the difference between a dashboard and a coach. Whoop would give me a score. My agent tells me exactly when to train and what to do.

### Step 5: Daytime Stress Monitoring

The heartbeat check runs every 30 minutes and focuses on real-time stress signals. Most of the time, it's silent — no news is good news. But when something is off, it alerts me.

The heartbeat prompt:

```
Daytime health check = stress/battery only:
- Heart rate: elevated >90 at rest = stress signal
- Resting HR: alert if >60
- HRV: alert if <80 (low battery/recovery)

Do NOT report sleep during the day.
If stress indicators are high, frame through essentialism:
high stress = probably doing too much.
```

Example alert I received last week:

```
⚠️ HR підвищений останню годину (avg 95 bpm at rest).
HRV знизився до 65 ms.

Це може означати стрес або перевтому.
У тебе 3 мітинги залишилось сьогодні — варто один скасувати?

Есенціалізм: менше, але краще.
```

I canceled one of the meetings. My HRV recovered within an hour. That's actionable coaching.

### Step 6: Calendar-Aware Workout Suggestions

This is where the system becomes more than just health tracking. The agent knows my recovery state AND my schedule. It finds gaps in my calendar and matches them with workout recommendations based on how I'm feeling.

| Recovery | Calendar Gap | Suggestion |
|----------|-------------|------------|
| 🟢 High HRV, low resting HR | 1.5h+ free | Strength or high-intensity intervals |
| 🟡 Normal HRV | 1h free | Moderate cardio, padel, swimming |
| 🔴 Low HRV, high resting HR | Any | Mobility, walking, or full rest |
| Any | No gaps | "Skip today, don't force it" |

If you're using Google Calendar (I use `gog` CLI for integration), the agent checks events automatically:

```bash
gog calendar events your@email.com --days 1
```

It sees your meetings, finds free windows, and suggests workout timing. No more "I'll train when I have time" — the agent tells you exactly when.

## Case Study: A Week of Use

Here's what the system caught in one week:

**Monday**: Recovery 🟢, HRV 131 ms. Suggested strength training at 14:00 (1.5h gap before client call). Did it. Felt great.

**Wednesday**: Recovery 🟡, HRV 95 ms (dropped from baseline). Calendar showed back-to-back meetings. Agent suggested skipping gym, doing 20min mobility instead. I ignored it, went to padel anyway. Played poorly, felt exhausted after.

**Friday**: Recovery 🔴, HRV 68 ms, resting HR 58 bpm. Agent said "rest day, your body is asking for it." I listened. By Saturday, HRV back to 125 ms.

That Wednesday was the learning moment. The agent was right — my body was already stressed from the meeting load. Adding intense exercise made it worse. I should have listened.

## Learnings

### What Worked

**1. Baselines are everything**
Generic thresholds are useless. "HRV below 50 is bad" means nothing if your baseline is 80 vs 120. The agent needed MY numbers to give useful advice.

**2. Calendar integration is the killer feature**
Whoop didn't know my schedule. My agent does. That's why it can say "train at 14:00" instead of just "today is a good day to train."

**3. Silence is golden**
The heartbeat check only alerts when something is wrong. No unnecessary notifications. I learned this from my phone — most alerts are noise. The agent respects my attention.

### What Didn't Work (At First)

**1. HRV and resting HR lag**
Apple Health exports HRV and resting HR with a 2-3 day delay in some configurations. Fresh sleep and activity data arrive immediately, but HR metrics lag. I adjusted the morning briefing to focus more on sleep quality and wrist temperature (which export immediately).

**2. Network connectivity**
The first week, my iPhone couldn't reach the webhook when I was out of the house. I added Tailscale to both iPhone and the server — solved instantly.

**3. JSON parsing failures**
Health Auto Export added a BOM character at the start of JSON. The webhook crashed silently. I added `body.replace(/^\uFEFF/, '')` to strip it. Trust but verify — always check your logs.

## The Result

For $5 (Health Auto Export app) and an hour of setup, I replaced a $360/year Whoop subscription with something more useful. The AI doesn't just give me a number. It tells me what to do, when to do it, and when to back off.

Whoop said: "Your recovery is 76%."  
My agent says: "Your recovery is great, you have a free window at 14:00, do strength training, but remember you have a dinner at 19:00 so don't go too hard."

That's the difference between a dashboard and a coach.

## Architecture Reference

For implementation details, here's the full architecture:

```
Apple Watch → iPhone (Health app)
                ↓
    Health Auto Export app (automated)
                ↓
        HTTP POST (JSON)
                ↓
    OpenClaw webhook server (:8090)
                ↓
        Stored as JSON files
                ↓
    Cron jobs read & analyze
                ↓
    Morning briefing / alerts → Telegram
```

File structure:

```
workspace/
└── health/
    ├── webhook-server.js    # HTTP server for receiving data
    ├── start-server.sh      # Auto-start script
    ├── server.log           # Server logs
    └── data/
        ├── health-*.json    # Individual exports
        └── daily-*.jsonl    # Daily aggregated log
```

Health Auto Export JSON field names:
- `resting_heart_rate` — daily resting HR
- `heart_rate_variability` — HRV (SDNN)
- `blood_oxygen_saturation` — SpO2
- `sleep_analysis` — sleep stages (includes `totalSleep`, `deep`, `rem`, `core`, `awake`, `sleepStart`, `sleepEnd`)
- `heart_rate` — individual HR readings (array)
- `respiratory_rate` — breathing rate
- `apple_sleeping_wrist_temperature` — overnight wrist temp

Storage: Each export is 500KB-5MB depending on metrics. At 3 exports/day, that's ~5-15MB/day. I keep 30 days and periodically clean up.

## What's Next

This is version 1. Here's what I want to add:

**1. Trend analysis**
Track HRV/HR over weeks, detect overtraining patterns before they become problems.

**2. Workout logging**
Log what I actually did, correlate with recovery. Did that padel session actually hurt my HRV? Let's see the data.

**3. Nutrition integration**
If I track food, add it to the morning briefing. "You ate late last night — that's why your deep sleep was low."

**4. Multi-device support**
Add data from gym equipment, running watch, whatever. More data, better coaching.

## Final Thoughts

Building this system taught me something about AI agents: they're most powerful when they know your context. Generic tools give generic advice. Personalized systems — with your baselines, your calendar, your goals — give coaching that actually works.

This isn't just about saving $360/year. It's about building tools that fit your life, not adapting your life to fit the tools.

The best part? I control the system. I can adjust the baselines, tweak the prompts, add new features. Whoop gave me a black box. I built a white box.

If you're already using OpenClaw (or considering it), this is a great first project. You'll learn about webhooks, cron jobs, data analysis, and prompt engineering — all while building something genuinely useful.

And if you're still paying for Whoop — well, now you know there's another way.

---

*Built with [OpenClaw](https://github.com/openclaw/openclaw). Quick setup guide: [openclaw-agent-bootstrap](https://github.com/ihorkatkov/openclaw-agent-bootstrap).*
