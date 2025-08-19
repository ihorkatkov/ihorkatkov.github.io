Title: Running GenServers in a Distributed Cluster — Article Plan

Goal
- Teach engineers when a distributed GenServer architecture is warranted and how to implement it safely and consistently, building on the testing (2024‑11‑01) and performance/observability (2025‑07‑21) articles.

Audience & Prerequisites
- Elixir engineers who have built and tested single‑node GenServers and understand performance basics.
- Familiarity with concepts from the previous two articles: thin GenServers, non‑blocking callbacks, Telemetry, and testing strategies.

Non‑Goals
- Not a deep dive into distributed databases or consensus protocols.
- Not an exhaustive guide to all clustering libraries; we present practical defaults and trade‑offs.

Key Questions This Article Answers (Why)
- When do you actually need distributed GenServers instead of scaling a single node vertically?
- What characteristics of your workload suggest sharding, a global singleton, or local per‑node workers?
- How do failures, netsplits, and latency change correctness assumptions?

Decision Framework (When)
- Workload partitionability: Can requests be keyed (user_id, account_id) to route to a shard?
- State strategy: Is state ephemeral and reconstructible, or must it be durable across node failure?
- Consistency tolerance: Is eventual consistency acceptable? Do you have idempotent operations?
- Availability/SLA: Do you need failover across nodes and rolling deploys without downtime?
- Team/ops constraints: Running on Kubernetes? Need automatic node discovery? Security hardening for distribution?

Quick Start — Start Here If...
- You see mailbox growth or p95 callback time > 10ms on a hot GenServer despite non‑blocking callbacks → consider sharding.
- You need coordinated leadership (exactly one scheduler of X) across nodes → consider a guarded global singleton (with fences and health checks).
- You need node‑local CPU/I/O locality with simple stateless work → per‑node worker pools.

Decision Tree & Thresholds (to include as flowchart in article)
- If single‑node memory > 60–70% of limit or CPU > 70% sustained and hot keys exist → shard by key.
- If writes require cross‑entity coordination and strict ordering → avoid naïve distribution; consider central coordinator or transactional outbox/sagas.
- If per‑key concurrency > cores on a single node or > 1k req/s per key group → shard by key range/consistent hash.

Core Mental Model (How it changes in a cluster)
- Nodes communicate over the network; per‑sender message order is preserved, but latency and failures are normal.
- Netsplits lead to concurrent leaders and duplicate work unless guarded.
- Global state synchronization is an application concern unless you adopt a coordinating library with its own trade‑offs.

Architecture Patterns (How)
1) Sharded GenServers (recommended default for keyed workloads)
   - Route messages by a stable shard key using consistent hashing.
   - Use `Registry` + `DynamicSupervisor` per node; optionally add a global registry.
   - Prefer sticky routing to minimize state movement; rebuild state on crash from a source of truth.
   - Libraries: `libcluster` for node discovery; optionally `Horde` for cross‑cluster registries/supervision.

2) Global Singleton (use sparingly)
   - One process in the whole cluster for coordination/locks.
   - Options: `:global`, `syn`, `Horde.Registry` with constraints.
   - Clarify failure semantics and avoid it for high‑throughput hot paths.

3) Per‑Node Worker Pools
   - Each node manages its own workers for locality (e.g., CPU‑bound tasks, I/O near caches).
   - A router distributes work across nodes; keep workers stateless or reconstructible.

Migration Strategy (from single node to cluster)
- Gradual enablement: introduce a router and keep existing GenServer as the only shard; then add more shards.
- Dual‑write/dual‑read windows for stateful servers: write to both old and new shards, read‑prefer new; verify metrics before cutover.
- Zero‑downtime: drain via readiness gates; stop routing new keys to old node; hand off remaining keys by rebuild on target.
- Rollback: retain the old single‑node path behind a feature flag; ensure idempotent ops and replay capability from durable log.
- Data migration: snapshot + change‑data‑capture replay; or event‑sourced rebuild per key.

State Strategies
- Ephemeral state with rebuild: Rehydrate from DB/event log on start; simplest and safest.
- Checkpointed state: Periodic snapshots to durable store; resume on failover.
- CRDT/gossip (advanced): For approximate aggregates or conflict‑free merges.
- Avoid in‑memory cross‑node shared mutable state assumptions.

Consistency, Failure, and Back‑Pressure
- Embrace at‑least‑once with idempotent handlers; exactly‑once is unrealistic without external coordination.
- Define netsplit behavior (reject writes, buffer, or accept with reconciliation).
- Bound mailboxes and enforce timeouts; prefer async patterns and retries with jitter.

Data Consistency Patterns (cross‑shard)
- Sagas with compensating actions for multi‑entity workflows.
- Transactional outbox/inbox to bridge DB and messaging reliably.
- Two‑phase commit (rare; only where absolutely necessary and bounded), with clear timeouts and failure paths.
- Cross‑shard idempotency keys to de‑duplicate retries.

Observability Across the Cluster
- Emit Telemetry with `node()` and shard identifiers; aggregate per node and per shard.
- Trace cross‑node calls (OpenTelemetry) to attribute latency.
- Dashboards: message_queue_len per shard, restarts/rebalances, node churn, SLOs (p95 callback times).

Monitoring & Alerting Cookbook
- Alert when `message_queue_len` p95 per shard > 50 for 5m; p99 callback duration > 20ms; restarts > 3/hour per shard.
- SLI/SLO examples: 99% of `handle_call` duration < 10ms; 99.9% routing latency < 5ms intra‑node, < 20ms cross‑node.
- Sample dashboards: per‑node CPU/mem, shard heatmap, error budgets, netsplit occurrences, handoff duration.

Operations & Security
- Cluster formation: `libcluster` strategies (Kubernetes DNS, gossip). Node naming conventions.
- Distribution security: cookies management, network policies/firewalls; consider TLS distribution if required.
- Rolling deploys and state safety: drain/hand‑off strategies, readiness gates.

Security Deep‑Dive
- Node authentication beyond cookies (TLS distribution, mTLS between nodes, cookie rotation).
- Network segmentation and least‑privilege security groups; restrict EPMD/ports.
- Audit logging: correlate admin operations and cross‑node state transitions with trace IDs.

Testing Strategies (build on prior article)
- Local multi‑node tests using `:peer`/`Node.spawn_link/2` and ExUnit helpers.
- Fault injection: kill nodes, simulate netsplits, validate idempotency and recovery.
- Contract tests for routers and shard selection; property tests for routing stability.

Testing Enhancements
- Chaos engineering drills: periodic process kills, node churn, artificial latency and packet loss.
- Load testing distributed paths with per‑key traffic models and skewed key distributions.
- Comprehensive netsplit scenarios: partition A|B, asymmetric loss, and prolonged split with reconciliation checks.

Step‑By‑Step Recipes (to include with concise code in the article)
- Recipe 1: Form a cluster with `libcluster` in a release (Kubernetes example) and verify node membership.
- Recipe 2: Implement a sharded GenServer
  - Consistent hash on key → `via` name → start/find shard under `DynamicSupervisor`.
  - Bounded mailbox, Telemetry, and non‑blocking callbacks (reuse patterns from perf article).
- Recipe 3: Global singleton using `Horde.Registry` (with cautions and fallbacks).
- Recipe 4: Durable checkpointing and recovery flow (snapshot + replay) with idempotent commands.
- Recipe 5: Cluster‑wide observability with PromEx/Grafana and tracing correlation.

Enhanced Recipes
- Recipe 0: “Should I distribute?” decision worksheet and scorecard.
- Gradual migration with dual‑write/verify/cutover/rollback.
- Handling cluster membership changes: rebalancing shards and draining safely.
- Cross‑region deployments: latency budgets, quorum choices, and read locality.

Anti‑Patterns to Call Out
- Synchronous cross‑node `GenServer.call/3` in hot paths.
- Assuming automatic state handoff between GenServers without explicit design.
- Unbounded mailboxes and no back‑pressure in distributed scenarios.
- Overusing global singletons instead of sharding.

Troubleshooting & Debugging
- Symptoms: slow cross‑node calls, growing mailboxes, GC spikes, inconsistent state after netsplits.
- Techniques: distributed tracing across nodes, `:observer` per node, `recon` sampling, netsplit detection hooks.
- Regression patterns: increased tail latency from cross‑node fan‑out, shard hotspotting after hash change.

Article Outline (ToC)
1. Motivation and mental model of distribution
2. Quick Start: Start here if… (decision cheatsheet + flowchart)
3. Common pitfalls to know before distributing
4. Decision framework: When you need a cluster (with thresholds)
5. Architecture patterns (shards, singleton, per‑node workers)
6. Migration strategy from single node
7. State strategies and data consistency (idempotency, sagas, 2PC caveats)
8. Failure modes, netsplits, and back‑pressure
9. Observability and SLOs in a cluster (cookbook)
10. Operations & security (deep‑dive)
11. Capacity planning and sizing
12. Recipes with code
13. Troubleshooting & debugging
14. Case studies (do’s and don’ts)
15. Cost analysis and when not to distribute
16. Checklist and takeaways

Cross‑References to Prior Articles
- Reuse thin‑server design and callback isolation for distributed paths.
- Apply the Telemetry patterns and budgets to shard callbacks and routers.
- Adapt the testing strategies to multi‑node and failure testing.

Deliverable
- A practical, opinionated guide with copy‑pasteable recipes and a final checklist engineers can apply to migrate from single‑node GenServers to a resilient clustered deployment.

Real‑World Case Studies (to prepare)
- Sharded user session managers for consumer apps with hotspot keys; routing by user_id to reduce cross‑node chatter.
- Global scheduler singleton for low‑throughput maintenance jobs with strict sequencing.
- Per‑node ingestion workers for log processing where locality and throughput trump global ordering.

Capacity Planning Guidance
- Shard count heuristic: 8–16× number of cores cluster‑wide; allow rebalancing headroom.
- Node sizing: prefer more smaller nodes for failure blast‑radius; ensure memory for peak queues + ETS buffers.
- Scale up vs. out: add nodes when per‑node CPU > 70% sustained and p95 callback time degrades despite local optimizations.

Alternative Patterns
- When to choose GenStage/Broadway for back‑pressure pipelines.
- Event sourcing with distributed GenServers for rebuildable state and clearer recovery.
- External coordination (Redis, etcd, Postgres advisory locks) vs. pure Erlang distribution—trade‑offs and when to mix.


