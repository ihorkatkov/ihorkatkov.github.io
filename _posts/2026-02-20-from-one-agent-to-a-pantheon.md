---
layout: post
title: "From One Agent to a Pantheon: Building AI Systems That Compound"
date: 2026-02-20
author: "Ihor Katkov"
tags: [ai, agents, opencode, openclaw, automation]
description: "How I went from a single AI assistant to a multi-agent system that researches its own source code, submits PRs, and analyzes its own behavior — and why true recursion still doesn't exist."
---

Last month, my personal AI assistant found a bug in her own voice messages, researched the open-source platform she runs on, wrote a fix, and submitted a pull request. PR #21193. Two files changed, eight additions. I didn't write a single line.

How we got there — and why it took six agents, two frameworks, and a Greek mythology naming convention.

## The Problem With One Agent

I've been working with coding agents since April 2024. Back then I wrote about the "stdlib approach" — treating agent context like a standard library, giving it structured knowledge instead of hoping it figures things out. The industry is just now catching up to that idea.

When I joined Neno, an AI fintech, in October 2024, I brought those practices with me. AI-first development. Agents writing code, agents reviewing code, agents planning work. It worked. For a while.

The ceiling hit around month three. A single agent with a single context window can't hold an entire codebase, a test suite, an architectural vision, and a step-by-step implementation plan simultaneously. Context windows are large now, but they're not infinite, and cramming everything into one makes the agent worse at all of it. The output gets diluted. The agent starts hedging. You lose the sharp, decisive moves that make AI-assisted development actually fast.

Separation of concerns — the same principle we've applied to software architecture for decades.

## The Pantheon

I built a system of specialized agents on top of [opencode](https://github.com/nichochar/opencode), each with a custom prompt, custom skills, and a specific role. opencode already uses Greek mythology naming internally, which turned out to be more than aesthetic. LLMs are trained extensively on Greek mythology. When you name an agent "Vulkanus" and tell it to forge robust code through test-driven development, you're tapping into a deep well of associative context the model already has. Call it "agent-1" and you get nothing. Call it Vulkanus, god of the forge, and the model leans into the role. Trust me bro.

The roster:

- **Zeus** — the orchestrator. Receives tasks, breaks them into subtasks, delegates to specialists, synthesizes results. Never writes code directly.
- **Vulkanus** — test-driven development. Writes the test first, then the implementation. Stubborn about coverage. That's the point.
- **Prometheus** — planning. Reads requirements, explores codebases, produces implementation plans. Thinks before anyone acts.
- **Oracle** — architecture. Evaluates trade-offs, reviews designs, catches structural problems before they become tech debt.

Each agent has custom prompts tuned to my workflow. Not the defaults — those are starting points. Months of iteration went into these prompts. Prometheus knows how I like implementation plans structured. Vulkanus knows our testing conventions. Zeus knows when to delegate and when to handle something directly.

The key architectural decision: separated context windows. Zeus doesn't carry Vulkanus's test output in its context. It gets a summary. This keeps each agent sharp and focused, operating in its zone of competence.

```jsonc
// .opencode/agents/vulkanus.md (simplified)
// Each agent gets a dedicated prompt with role, constraints, and skills
{
  "agents": {
    "vulkanus": {
      "model": "anthropic/claude-sonnet-4",
      "prompt": "agents/vulkanus.md",
      "skills": ["test-first", "coverage-check", "refactor"]
    },
    "zeus": {
      "model": "anthropic/claude-sonnet-4",
      "prompt": "agents/zeus.md",
      "skills": ["delegate", "synthesize", "plan"]
    }
  }
}
```

This is the Pantheon. It handles my daily coding work at Neno. But it's only half the story.

## Aris

Aris is my personal AI assistant. She runs on [OpenClaw](https://github.com/openclaw/openclaw) — an open-source agent framework — inside a Docker container on my home server, accessible via Tailscale. She has access to my health data from Apple Watch, my Google Calendar, my GitHub repos, my Telegram, and my location via OwnTracks. She handles scheduling, reminders, daily briefings, health tracking. Smart, capable, reliable.

But she couldn't code.

Not the way I need code written — with test coverage, architectural awareness, proper git workflow. She's a generalist assistant running on a conversational agent framework. Coding agents are a different discipline.

So I integrated opencode into Aris. Give her the ability to delegate coding tasks to the Pantheon.

The implementation is a bash script with `flock` serialization (one opencode session at a time — these things are resource-hungry), configurable timeouts, and structured logging:

```bash
#!/bin/bash
# opencode-delegate.sh — Aris delegates coding to the Pantheon
# Serialized via flock, 20-min default timeout, JSON output

LOCK_FILE="/tmp/opencode-delegate.lock"
TIMEOUT="${OPENCODE_TIMEOUT:-20}"
LOG_DIR="aris-ops/opencode-logs"

exec 200>"$LOCK_FILE"
flock -n 200 || { echo '{"status":"busy"}'; exit 1; }

AGENT="${AGENT:-zeus}"  # Default to Zeus orchestrator
opencode --agent "$AGENT" --non-interactive \
  --output json \
  "$PROMPT" 2>"$LOG_DIR/$(date +%s).log"
```

The critical design constraint: Aris never writes code herself. She always delegates. This keeps her context window small and efficient — she's managing tasks, not debugging TypeScript. She describes what needs to happen, Zeus figures out how, and the specialists execute.

## The PR

In my previous article, ["Living With the Lethal Trifecta,"](https://ihorkatkov.com) I wrote about Simon Willison's concept — the convergence of capabilities that makes AI systems both powerful and dangerous. One of those capabilities is autonomous action. What happened next is a case study in what that looks like in practice.

Aris's voice messages in Telegram broke. She was responding with garbage — garbled audio that made no sense. I noticed it, mentioned it in passing, and moved on. Aris didn't.

She identified two bugs. First, OpenClaw's auto-summary feature was mangling the text before it reached the TTS engine — the text going into speech synthesis wasn't what she intended to say. Second, the audio was being sent as MP3 when Telegram expects Opus-encoded voice messages. Two separate bugs, both contributing to the same symptom.

Then she did something I didn't expect. She used the Pantheon to research OpenClaw's source code. The platform she runs on.

Prometheus mapped the file system. Agents read source files across the codebase — the TTS pipeline, the Telegram channel adapter, the message processing chain. They wrote research markdown files. Over 50 of them. Structured notes on how each component worked, where the bugs originated, what the fix should look like.

Zeus synthesized the research, understood the codebase architecture, and wrote the fix.

Then the full cycle executed: build, test, fork the OpenClaw repository, create a branch, commit, push, open a pull request.

[PR #21193](https://github.com/openclaw/openclaw/pull/21193). Two files changed. Eight additions. Clean, minimal, correct.

My AI assistant, running on an open-source platform, autonomously found bugs in that platform, researched the source code, fixed the bugs, and contributed the fix back upstream. She improved the infrastructure she depends on.

No human wrote code. No human opened the PR. I reviewed it and clicked merge.

## The Oracle Loop

After the PR, I started thinking about what else Aris could improve about her own operation. Not her source code — that was the OpenClaw fix. Her behavior. Her patterns. Her inefficiencies.

Aris built her own logging system. Two files: `work-log.jsonl` tracks every task she completes — what was requested, what she did, how long it took, what tools she used. `tool-log.jsonl` tracks every tool invocation with inputs, outputs, and error states. This runs through OpenClaw's plugin hooks — which, in a fitting bit of recursion, Aris herself helped develop. (That was a three-day saga of debugging hook execution order. A separate agent, Mnemosyne, found the root cause in three minutes. Three days of my debugging versus three minutes of focused agent work. Humbling.)

On top of this logging sits the Oracle Loop. A separate Oracle agent — deliberately running on a different model (GPT, not Claude) to get a different perspective — executes on a cron schedule three times a day. It reads Aris's logs and writes two documents:

**`oracle-recommendations.md`** — specific, actionable infrastructure improvements. "Tool X is being called with malformed parameters 12% of the time. Add input validation." "Calendar queries are redundantly fetching the same data within 5-minute windows. Add caching."

**`improvement-journal.md`** — longer-term observations about behavioral patterns. Trends across days and weeks that individual log entries don't reveal.

The system analyzes its own behavior. Logs go in, structured recommendations come out, and those recommendations feed back into how Aris operates.

This is the closest thing I have to a recursive self-improvement loop. It's not true recursion — I'll get to that — but it creates a compounding effect that's hard to overstate.

## The Compounding Effect

Each capability Aris gains enables the next one. Research ability led to the OpenClaw fix. The OpenClaw fix improved her voice messages. Better voice messages meant more natural interaction. The logging system enabled the Oracle Loop. The Oracle Loop's recommendations improve her tool usage. Better tool usage means faster task completion. Faster task completion means more capacity for the next improvement.

Research → Fix → PR → Better autonomy → More fixes. It compounds.

This maps to something Greg McKeown writes about in *Essentialism* — it's not about doing more things. It's about building systems where each layer amplifies the next. Fewer projects, deeper investment, bigger returns. The Pantheon isn't six agents doing six separate jobs. It's six agents forming one system where Zeus's orchestration makes Vulkanus's testing more effective, which makes Prometheus's plans more reliable, which makes Zeus's orchestration better.

## What Comes Next

Here's where the ceiling is.

True self-improvement recursion doesn't exist yet. What I have is a manually-wired loop: Aris logs → Oracle analyzes → recommendations file → I review → changes get implemented (sometimes by Aris, sometimes by me). The human is still in the loop at critical junctures. The Oracle can recommend, but it can't autonomously restructure Aris's prompt or rewrite her delegation logic.

The gap between "system that analyzes itself" and "system that improves itself" is the gap the entire industry is staring at right now. Anthropic published their compiler rewrite paper showing agents that can tackle large-scale code migrations. Cursor shipped background agents that run while you're away. Everyone's moving toward long-running, autonomous agent orchestration.

But a straightforward solution doesn't exist. The problems are real: agent drift over long sessions, error accumulation without human checkpoints, the difficulty of evaluating whether an autonomous change actually made things better or just different. When your agent modifies its own prompts, how do you know the modification was an improvement? You need evaluation, and evaluation of open-ended behavior is an unsolved problem.

I think the industry solves long-running agent orchestration this year. The pieces are there — better models, better tooling, better understanding of agent architectures. What I'm doing with the Pantheon and the Oracle Loop is a workaround for the absence of true recursive improvement. A good workaround. A productive one. But a workaround.

The next version of this system won't need me to review Oracle's recommendations. It'll have a validation layer — probably another agent — that can test proposed changes against behavioral benchmarks before applying them. The Oracle suggests, the Validator tests, the system evolves. That's where this is going.

## The Insight

It's not about any individual agent. It's the architecture. The separation of concerns. The delegation patterns. The logging and analysis loop.

Every engineer I talk to about AI agents starts with the same question: "Which model should I use?" Wrong question. The model is a component. The architecture is the product. A well-orchestrated system of focused agents on a mid-tier model will outperform a monolithic agent on the best model available. I've watched it happen repeatedly.

Build the system. Name your agents after Greek gods if it helps (it does). Give each one a sharp, specific role. Wire them together with explicit delegation, not implicit context sharing. Log everything. Analyze the logs with a different model than the one generating them. And accept that true recursion isn't here yet — but the compounding effects of a well-built system are already worth the investment.

My assistant fixed her own platform and submitted the PR to prove it. That's not recursion. That's something better — it's compounding.
