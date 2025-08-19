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

State Strategies
- Ephemeral state with rebuild: Rehydrate from DB/event log on start; simplest and safest.
- Checkpointed state: Periodic snapshots to durable store; resume on failover.
- CRDT/gossip (advanced): For approximate aggregates or conflict‑free merges.
- Avoid in‑memory cross‑node shared mutable state assumptions.

Consistency, Failure, and Back‑Pressure
- Embrace at‑least‑once with idempotent handlers; exactly‑once is unrealistic without external coordination.
- Define netsplit behavior (reject writes, buffer, or accept with reconciliation).
- Bound mailboxes and enforce timeouts; prefer async patterns and retries with jitter.

Observability Across the Cluster
- Emit Telemetry with `node()` and shard identifiers; aggregate per node and per shard.
- Trace cross‑node calls (OpenTelemetry) to attribute latency.
- Dashboards: message_queue_len per shard, restarts/rebalances, node churn, SLOs (p95 callback times).

Operations & Security
- Cluster formation: `libcluster` strategies (Kubernetes DNS, gossip). Node naming conventions.
- Distribution security: cookies management, network policies/firewalls; consider TLS distribution if required.
- Rolling deploys and state safety: drain/hand‑off strategies, readiness gates.

Testing Strategies (build on prior article)
- Local multi‑node tests using `:peer`/`Node.spawn_link/2` and ExUnit helpers.
- Fault injection: kill nodes, simulate netsplits, validate idempotency and recovery.
- Contract tests for routers and shard selection; property tests for routing stability.

Step‑By‑Step Recipes (to include with concise code in the article)
- Recipe 1: Form a cluster with `libcluster` in a release (Kubernetes example) and verify node membership.
- Recipe 2: Implement a sharded GenServer
  - Consistent hash on key → `via` name → start/find shard under `DynamicSupervisor`.
  - Bounded mailbox, Telemetry, and non‑blocking callbacks (reuse patterns from perf article).
- Recipe 3: Global singleton using `Horde.Registry` (with cautions and fallbacks).
- Recipe 4: Durable checkpointing and recovery flow (snapshot + replay) with idempotent commands.
- Recipe 5: Cluster‑wide observability with PromEx/Grafana and tracing correlation.

Anti‑Patterns to Call Out
- Synchronous cross‑node `GenServer.call/3` in hot paths.
- Assuming automatic state handoff between GenServers without explicit design.
- Unbounded mailboxes and no back‑pressure in distributed scenarios.
- Overusing global singletons instead of sharding.

Article Outline (ToC)
1. Motivation and mental model of distribution
2. Decision framework: When you need a cluster
3. Architecture patterns (shards, singleton, per‑node workers)
4. State strategies and correctness (idempotency, recovery)
5. Failure modes, netsplits, and back‑pressure
6. Observability and SLOs in a cluster
7. Ops: cluster formation, security, and deploys
8. Recipes with code
9. Checklist and takeaways

Cross‑References to Prior Articles
- Reuse thin‑server design and callback isolation for distributed paths.
- Apply the Telemetry patterns and budgets to shard callbacks and routers.
- Adapt the testing strategies to multi‑node and failure testing.

Deliverable
- A practical, opinionated guide with copy‑pasteable recipes and a final checklist engineers can apply to migrate from single‑node GenServers to a resilient clustered deployment.


