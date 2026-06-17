# Blog Brief — "Agents at the Center: Why We're Building Software Wrong"

**Intended title options:**
1. "Agents at the Center: Why We're Building Software Wrong"
2. "Stop Bolting AI onto Software. Put the Agent First."
3. "The Harness Pattern: How Kairos Changed How I Think About Architecture"

**Target audience:** Technical founders, senior engineers, indie hackers building with AI agents

**Target length:** 1,200–1,800 words

**Publication target:** Feb 26, 2026

---

## Core Thesis

We build software, then try to integrate AI agents as a feature.
It should be the opposite: **agent at the center, software as a harness.**

The shift isn't cosmetic. It changes where business logic lives, what code is for, and what "good architecture" even means.

---

## Key Evidence (already in repo)

**Kairos SPEC.md literally says:**
> "Claude Code acts as the orchestrator, invoking tools as needed. Each tool does one thing well. Human makes final decisions, system provides data and assists execution."

This is not a trading platform with AI features. It's an agent with a harness.

**What the harness looks like in Kairos:**
- `analyze-token`, `enter-position`, `exit-position` — dumb tools, one thing each
- Business logic (strategy, risk rules) lives in `STRATEGY.md` and `SPEC.md` — markdown, not code
- Agent reads the markdown, understands context, decides what to call and when
- Code only enforces mechanical correctness (TypeScript types, exit conditions, API rate limits)

**What would traditional fintech look like:**
- Build the trading platform (positions, orders, analytics)
- "Add AI": connect LLM to UI, wire up some APIs
- Agent is constrained by the data model and UI flows
- Business logic is in stored procedures / service layer — LLM can only ask, not decide

---

## Article Structure (proposed)

1. **Hook** — concrete moment from Kairos where the agent made the call, not the code
2. **The old way** — how we currently build software with AI bolted on
3. **The insight** — what changes when you flip the model
4. **What the harness pattern looks like** (with Kairos as example)
   - Tools that do one thing well
   - Business logic in human-readable context (markdown/docs), not code
   - Agent as orchestrator, not feature
5. **What this changes** — architecture, roles, where complexity lives
6. **Hard question** — when does this NOT work? (honest answer needed)
7. **Closer** — sharp line

---

## Interview Questions for Ihor

*Answer these tomorrow. Each answer = 2-4 sentences max. Concrete details, specific moments.*

**Q1: What was the moment building Kairos where you realized the architecture was different?**
Was there a specific decision you made — like "I'll put the strategy in markdown instead of code" — where you felt something click?

**Q2: What does "harness" mean practically? What code EXISTS and what doesn't exist?**
If someone asked "ok, where is the business logic in Kairos?", what do you point to? And what did you deliberately NOT put in code?

**Q3: What's the hardest thing for a traditional software engineer to let go of?**
What mental model breaks? Is it "the system should be deterministic"? "Business logic belongs in the service layer"? What's the friction?

**Q4: What is the agent actually doing that the code doesn't do?**
Give a concrete example from Kairos: the agent reads X, decides Y, calls Z. Walk through one real scenario.

**Q5: When does this pattern FAIL? Where would you NOT use it?**
Honest answer. High-frequency trading? Safety-critical systems? Regulated workflows? Where's the line?

**Q6: How does this change the role of the engineer?**
If the agent is the orchestrator and tools are the harness — what does the engineer build? What does "good engineering" mean in this model?

**Q7 (optional): What would you tell a team that's currently "adding AI" to existing software?**
One concrete piece of advice. Not "think different" — what do you actually do first?

---

## Raw Material Available

- Kairos repo: `github.com/ihorkatkov/kairos` (private, Aris has access)
- Key files: `SPEC.md`, `STRATEGY.md`, `AGENTS.md`, `src/commands/`, `src/lib/`
- Architecture diagram in SPEC.md Section 2
- Pool selection algorithm (Section 5.1) — shows agent-first design in code
- Ihor's conversation context from Feb 25: Elixir agent framework discussion, LaunchClaw, Star Trek manifesto

## Voice Notes

- Don't start with abstract thesis. Start with a concrete moment.
- Code review analogy from Whoop article worked well — use similar technical analogies here
- "Harness" is a good word — Ihor used it naturally. Keep it.
- The contrast with "bolted-on AI" is the tension that drives the piece
- Closer should be a sharp question or exec line — not a summary
