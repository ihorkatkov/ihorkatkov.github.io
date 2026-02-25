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

A year earlier, I'd have gotten the same information from Whoop — $30/month for a recovery score and a workout suggestion. But Whoop didn't know my calendar. It couldn't tell me *when* to train, only that I *could*. And every metric it tracked, my Apple Watch was already collecting.

So I built my own interpretation layer. Using [OpenClaw](https://github.com/openclaw/openclaw) and Apple Health data, I get a personalized morning briefing, daytime stress alerts, and calendar-aware workout suggestions — from my Apple Watch, processed by an agent that knows my baselines and my schedule. This post is exactly how to replicate it.

## Two Years With Whoop

Let me be clear: Whoop is a great product. I used it for two years and it genuinely changed how I think about my body.

The first two months I was obsessed. Every morning I checked my recovery score and worked to get green days. I tracked diet, journaled workouts, logged everything. It felt like too much — but the data was teaching me things that were counterintuitive. I had to walk more, not less, to recover. Go to bed earlier. I designed a specific gym routine that left me feeling better the *next* day instead of wrecked. Things I never would have figured out on my own.

After two years, I'd distilled it all down to a simple set of rules: **sleep better, manage stress, actively restore, train.** Those rules are mine now. I don't need the app to teach them to me anymore.

So I canceled.

The Apple Watch metrics aren't as precise — the Whoop strap is designed specifically for health tracking, the Watch isn't. But "good enough" is fine when you already know your baselines and what to do with them.

What I couldn't replace was something Whoop never actually had: **active intelligence**.

Whoop doesn't know about my calendar. It can't see that I have back-to-back meetings until 18:00 and suggest training at 14:00 instead of skipping entirely. It can't connect my morning HRV to my afternoon energy crash and recommend cutting the day short. Its diary feature tracks what happened — but it doesn't proactively shape what happens next.

One of Whoop's best features is the journal. You answer questions every day, it learns patterns. But you don't need a separate diary when you have an autonomous agent that already knows your calendar, your health data, your emails, your workload. An agent that's been fine-tuned *specifically for you* — not for a generic user.

That's what I built.

Think of it like code review. The diff is already there (the health data). What you need is something that understands the context — your baselines, your schedule, your goals — to tell you what it means and what to do next.

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

**3. Daytime monitoring (heartbeat)**
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

Before the AI can tell you "your recovery is low," it needs to know what's normal for you. Generic thresholds don't work. My resting heart rate of 49 bpm is excellent for me, but might be worrying for someone else.

Track for 1-2 weeks, then calculate your personal baselines. Mine (from Whoop historical data):

| Metric | My Baseline | Alert Threshold |
|--------|------------|-----------------| 
| Resting HR | 52 bpm | > 60 bpm |
| HRV (SDNN) | 120 ms | < 80 ms |
| SpO2 | 97% | < 95% |
| Sleep | 7.5h | < 7h |

If you don't have Whoop history, track Apple Watch data for 2 weeks and ask your agent to calculate averages. It can read the JSON files and compute the stats.

### Step 4: Morning Briefing (The Core)

Every morning at 7:30 AM, the agent reads my latest health data, compares it with my baselines, checks my calendar, and gives me a recovery score + workout recommendation.

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

Whoop would give me a score. My agent tells me exactly when to train and what to do.

### Step 5: Daytime Stress Monitoring

The heartbeat check runs every 30 minutes. Most of the time — silence. But when something is off:

```
⚠️ HR elevated for the past hour (avg 95 bpm at rest).
HRV dropped to 65 ms.

This could mean stress or overexertion.
You have 3 meetings left today — worth dropping one?

Essentialism: less, but better.
```

I canceled one of the meetings. My HRV recovered within an hour.

### Step 6: Calendar-Aware Workout Suggestions

| Recovery | Calendar Gap | Suggestion |
|----------|-------------|------------|
| 🟢 High HRV, low resting HR | 1.5h+ free | Strength or high-intensity intervals |
| 🟡 Normal HRV | 1h free | Moderate cardio, padel, swimming |
| 🔴 Low HRV, high resting HR | Any | Mobility, walking, or full rest |
| Any | No gaps | "Skip today, don't force it" |

## Case Study: A Week of Use

**Monday**: Recovery 🟢, HRV 131 ms. Suggested strength training at 14:00 (1.5h gap before client call). Did it.

**Wednesday**: Recovery 🟡, HRV 95 ms (dropped from baseline). Calendar showed back-to-back meetings. Agent suggested skipping gym, doing 20min mobility instead. I ignored it, went to padel anyway. Played poorly, felt exhausted after.

**Friday**: Recovery 🔴, HRV 68 ms, resting HR 58 bpm. Agent said "rest day, your body is asking for it." I listened. By Saturday, HRV back to 125 ms.

That Wednesday was the learning moment. The agent was right — my body was already stressed from the meeting load. Adding intense exercise made it worse. I should have listened.

## Learnings

**Baselines are everything.** Generic thresholds are useless. "HRV below 50 is bad" means nothing if your baseline is 80 vs 120. The agent needed MY numbers to give useful advice.

**Calendar integration makes the difference.** Whoop didn't know my schedule. My agent does. That's why it can say "train at 14:00" instead of just "today is a good day to train."

**Silence is golden.** The heartbeat check only alerts when something is wrong. No unnecessary notifications. I learned this from my phone — most alerts are noise. The agent respects my attention.

**What didn't work at first:**

- HRV and resting HR lag 2-3 days in Apple Health in some configurations. I adjusted the morning briefing to focus more on sleep quality and wrist temperature, which export immediately.
- Network connectivity: the first week, my iPhone couldn't reach the webhook when I was out of the house. I added Tailscale to both iPhone and the server — solved instantly.
- JSON parsing failures: Health Auto Export added a BOM character at the start of JSON. The webhook crashed silently. I added `body.replace(/^\uFEFF/, '')` to strip it. Always check your logs.

## The Result

$5 (Health Auto Export app) + an hour of setup = replaced a $360/year Whoop subscription.

Whoop said: "Your recovery is 76%."  
My agent says: "Your recovery is great, you have a free window at 14:00, do strength training, but remember you have dinner at 19:00 so don't go too hard."

That's the difference between a dashboard and a coach.

## Architecture Reference

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

Storage: each export is 500KB-5MB depending on metrics. At 3 exports/day, that's ~5-15MB/day. I keep 30 days and periodically clean up.

## Final Thoughts

Whoop is a great product. It taught me everything I needed to know about my body — and then I outgrew it.

Two years of data gave me my personal rules: sleep better, manage stress, actively restore, train. Once you have the rules, you don't need the teacher anymore. What you need is something that applies those rules to your actual life — your calendar, your workload, your specific day.

That's not a $30/month subscription. That's an agent that knows you.

Whoop gave me the rules. My agent applies them.

$5 (Health Auto Export app) + an hour of setup. That's the whole cost.

---

*Built with [OpenClaw](https://github.com/openclaw/openclaw). Quick setup guide: [openclaw-agent-bootstrap](https://github.com/ihorkatkov/openclaw-agent-bootstrap).*
