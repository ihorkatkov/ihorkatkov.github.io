Title: Running GenServers in a Distributed Cluster — Article Plan

Goal
- Teach engineers when a distributed GenServer architecture is warranted and how to implement it safely and consistently, building on the testing (2024‑11‑01) and performance/observability (2025‑07‑21) articles.

Scope
- Focus strictly on Elixir/GenServer specifics for distributed operation. Assume familiarity with general distributed systems concepts (consistency, partitions, retries).

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

Decision Tips (Elixir‑centric)
- If hot keys overload a single mailbox despite callback hygiene → shard by key using `:erlang.phash2/2` or a ring.
- If you truly need “exactly one” coordinator across nodes → guarded global registration (`:global`, `syn`, or `Horde`) with health checks.
- Prefer per‑node worker pools for locality; keep work stateless or reconstructible.

Core Mental Model (How it changes in a cluster)
- Nodes communicate over the network; per‑sender message order is preserved, but latency and failures are normal.
- Netsplits lead to concurrent leaders and duplicate work unless guarded.
- Global state synchronization is an application concern unless you adopt a coordinating library with its own trade‑offs.

Architecture Patterns (How)
1) Sharded GenServers (recommended default for keyed workloads)
   - Route messages by a stable shard key using consistent hashing.
   - Use `Registry` + `DynamicSupervisor` per node; name shards via `{:via, Registry, {RegistryName, shard_key}}`.
   - Prefer sticky routing to minimize state movement; rebuild state on crash from a source of truth.
   - Libraries: `libcluster` for node discovery; `Registry` for local names; optional `Horde` for cross‑node.

2) Global Singleton (use sparingly)
   - One process in the whole cluster for coordination/locks.
   - Options: `:global`, `syn`, `Horde.Registry` with constraints.
   - Clarify failure semantics and avoid it for high‑throughput hot paths.

3) Per‑Node Worker Pools
   - Each node manages its own workers for locality (e.g., CPU‑bound tasks, I/O near caches).
   - A router distributes work across nodes; keep workers stateless or reconstructible.

Migration Strategy (simple)
- Add a router in front of the existing GenServer; start with 1 shard, then increase shard count.
- For stateful servers, rebuild state on target shard at start (`handle_continue`) and switch traffic gradually.
- Rollback by routing all keys back to the original shard via a feature flag.

State Strategies
- Ephemeral state with rebuild: Rehydrate from DB/event log on start; simplest and safest.
- Checkpointed state: Periodic snapshots to durable store; resume on failover.
- CRDT/gossip (advanced): For approximate aggregates or conflict‑free merges.
- Avoid in‑memory cross‑node shared mutable state assumptions.

Consistency, Failure, and Back‑Pressure
- Embrace at‑least‑once with idempotent handlers; exactly‑once is unrealistic without external coordination.
- Define netsplit behavior (reject writes, buffer, or accept with reconciliation).
- Bound mailboxes and enforce timeouts; prefer async patterns and retries with jitter.

Data Consistency (practical BEAM notes)
- Prefer idempotent commands and per‑key sequencing; add idempotency keys where retries may duplicate work.
- If bridging DB and messaging, a transactional outbox can keep producers simple; avoid heavy protocols in GenServers.

Observability Across the Cluster
- Emit Telemetry with `node()` and shard identifiers; aggregate per node and per shard.
- Trace cross‑node calls (OpenTelemetry) to attribute latency.
- Dashboards: message_queue_len per shard, restarts/rebalances, node churn, SLOs (p95 callback times).

Monitoring Tips (Elixir‑centric)
- Track `:telemetry` events with `node()` and `shard` tags; alert on mailbox growth and callback duration.

Operations & Security (OTP specifics)
- Cluster formation with `libcluster` (DNS/gossip); consistent node names per environment.
- Manage cookies securely; restrict ports; TLS distribution only if required by environment.
- Draining: pause routing, let shards finish, then stop; rebuild state on start.

Security Deep‑Dive
- Node authentication beyond cookies (TLS distribution, mTLS between nodes, cookie rotation).
- Network segmentation and least‑privilege security groups; restrict EPMD/ports.
- Audit logging: correlate admin operations and cross‑node state transitions with trace IDs.

Testing Strategies (build on prior article)
- Local multi‑node tests using `:peer`/`Node.spawn_link/2` and ExUnit helpers.
- Fault injection: kill nodes, simulate netsplits, validate idempotency and recovery.
- Contract tests for routers and shard selection; property tests for routing stability.

Minimal Distributed Testing
- Spin up peers with `:peer` or `Node.spawn_link/2`; assert routing invariants and state rebuild behavior.
- Inject simple failures: kill a shard process, restart, and verify idempotency.

Step‑By‑Step Recipes (to include with concise code in the article)
- Recipe 1: Form a cluster with `libcluster` in a release (Kubernetes example) and verify node membership.
- Recipe 2: Implement a sharded GenServer
  - Consistent hash on key → `via` name → start/find shard under `DynamicSupervisor`.
  - Bounded mailbox, Telemetry, and non‑blocking callbacks (reuse patterns from perf article).
- Recipe 3: Global singleton using `Horde.Registry` (with cautions and fallbacks).
- Recipe 4: Durable checkpointing and recovery flow (snapshot + replay) with idempotent commands.
- Recipe 5: Cluster‑wide observability with PromEx/Grafana and tracing correlation.

Keep It Simple (what we’ll show)
- `libcluster` config, `Registry` naming via `{:via, Registry, …}`.
- Simple shard router using `:erlang.phash2/2`.
- Optional global registration example with `:global` or `Horde` and trade‑offs.

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
1. Goals and scope (Elixir/GenServer specifics only)
2. Quick start decision tips
3. OTP distribution mental model (what changes for GenServers)
4. Patterns: sharded, global singleton (when unavoidable), per‑node workers
5. Simple migration path
6. State and consistency (idempotency, rebuild on start)
7. Failure handling and back‑pressure
8. Observability essentials (Telemetry with node/shard)
9. Ops basics: libcluster, cookies, draining
10. Recipes
11. Troubleshooting
12. Checklist and takeaways

Cross‑References to Prior Articles
- Reuse thin‑server design and callback isolation for distributed paths.
- Apply the Telemetry patterns and budgets to shard callbacks and routers.
- Adapt the testing strategies to multi‑node and failure testing.

Deliverable
- A practical, opinionated guide with copy‑pasteable recipes and a final checklist engineers can apply to migrate from single‑node GenServers to a resilient clustered deployment.

Real‑World Examples (brief)
- Sharded user session managers (route by `user_id`).
- Global scheduler for low‑throughput maintenance tasks.
- Per‑node ingestion workers for locality‑sensitive pipelines.

Alternative Patterns (brief pointers)
- When to choose GenStage/Broadway for pipeline/back‑pressure needs.
- External coordination (Redis, Postgres advisory locks) when global ordering/locks are simpler externally.


