---
layout: post
title: "You Built a GenServer. Now Make It Fast, Observable, and Bulletproof."
date: 2025-07-20
categories: [Elixir, Performance]
tags: [elixir, genserver, performance, observability, resilience, software-engineering]
author: "Ihor Katkov"
draft: true
---

## Introduction

Remember the last time you shipped a shiny new GenServer to production?  It passed every unit test, handled your happy-path demo traffic, and looked rock-solid on paper.  Then real users showed up.  Latency spikes, CPU climbs, and suddenly the BEAM scheduler view in `:observer` looks like a Christmas tree.  I’ve been there – and I’ve learned that **building a GenServer is the easy part**; making it *fast, observable, and bulletproof* is where the real work starts.

In my previous article I walked through a TDD approach to *testing* GenServers.  This follow-up is the field manual I wish I had when I first pushed one of those servers to production.  We’ll build a mental model of how GenServers really cost CPU cycles, then apply a toolbox of performance and observability techniques you can drop into your code today.

By the end you’ll know how to:

1. Read the BEAM’s “cost model” – mailbox size, scheduler reductions, message queue length – so you can spot trouble early.
2. Refactor hot paths so callbacks never block schedulers.
3. Push read-heavy state to ETS / `persistent_term` without losing consistency.
4. Add cheap, composable Telemetry so dashboards light up before pagers do.
5. Choose when to graduate from a single GenServer to GenStage, Broadway, or full-blown distributed sharding.

Let’s dive in.

---

## 1. The Real GenServer Cost Model *(mental model)*

A GenServer is just a process with a mailbox, but the devil is in the scheduler details.  The BEAM VM runs **N** schedulers – one per CPU core by default – and each scheduler works through a run queue of processes.  Key things to watch:

* **Mailbox size** – `Process.info(pid, :message_queue_len)` tells you how many messages are waiting.  A single overloaded mailbox can starve other processes.
* **Reductions** – every BEAM operation costs reductions; long-running callbacks burn the budget, delaying other work.
* **Scheduler migrations** – when a process hogs a scheduler, the VM may migrate it, thrashing caches and hurting latency.
* **Sync vs. async** – `GenServer.call/3` blocks the caller; `cast/2` doesn’t.  Calls are convenient but couple lifecycles and back-pressure.

Tools I keep on my belt:

```elixir
:observer.start()
:recon.proc(:info)
telemetry_metrics_statsd
```

Spend five minutes watching these metrics during load and your optimisation story usually writes itself.

---

## 2. Performance & Throughput Techniques

### 2.1 Keep Callbacks Non-Blocking

If a callback waits on disk, network, or heavy CPU **your entire GenServer stalls**.  The fix is delightfully boring: move work elsewhere.

```elixir
def handle_call({:compute, input}, _from, state) do
  Task.start(fn -> heavy_math(input) end)
  {:reply, :ok, state}
end
```

The caller gets an immediate `:ok`, the task is linked so crashes bubble, and the scheduler stays healthy.  Measure with `:timer.tc(fn -> … end)` or Telemetry and set a budget (I shoot for <1 ms).

### 2.2 Post-Init Heavy Work with `handle_continue`

Boot time matters when your GenServer sits inside a supervision tree – a slow `init/1` delays the whole app.  Load large datasets *after* the process is up:

```elixir
def init(opts) do
  {:ok, %{}, {:continue, :warm_cache}}
end

def handle_continue(:warm_cache, state) do
  cache = load_big_table()
  {:noreply, %{state | cache: cache}}
end
```

Your supervision tree comes online instantly, and the heavy work happens without blocking.

### 2.3 Externalise Read-Heavy State (ETS / `persistent_term`)

GenServers are single-writer, single-reader; every read fights for the mailbox.  ETS provides concurrent reads:

```elixir
:ets.new(:kv, [:named_table, :public, read_concurrency: true])
```

For truly static data, `persistent_term` is zero-copy and zero-GC:

```elixir
persistent_term.put({__MODULE__, :config}, huge_config_map)
```

Trade-offs?  Writes are expensive and global.  Measure before moving.

### 2.4 Batching & Coalescing Patterns

Sometimes the cheapest optimisation is *do less*.  Accumulate writes and flush every X milliseconds:

```elixir
def handle_cast({:track, metric}, state) do
  {:noreply, %{state | buffer: [metric | state.buffer]}}
end

# Flush on timer or queue length
```

Used sparingly, batching smooths traffic spikes without complex back-pressure logic.

### 2.5 Back-Pressure & Demand Control

If producers outpace your GenServer, queues explode.  Options:

1. **Bounded mailbox** – reject or drop messages after a threshold.
2. **Timeouts on `call/3`** – force callers to handle slowness.
3. **Demand signalling** – borrow `gen_stage` semantics in plain messages (`{:demand, n}`).

Back-pressure is architecture, not syntax; pick a contract and document it.

### 2.6 Sharding Hot Keys

One GenServer → one mailbox.  Hot keys will hit the limit.  Partition with a `Registry`:

```elixir
key = :erlang.phash2(customer_id, 16)
{:ok, pid} = MyShardSupervisor.start_child(key)
```

Or reach for libraries like `hash_ring`.  Benchmark both median and tail latencies before & after – surprises abound.

### 2.7 When to Graduate to GenStage / Broadway

Rules of thumb:

* >10 k msgs/sec sustained – start thinking GenStage.
* Complex pull-based back-pressure – Broadway gives it out-of-the-box.
* Multiple consumers that must stay in order – GenStage’s demand model shines.

Migration is incremental – you can embed a GenStage producer inside an existing GenServer and fan out gradually.

---

## 3. Observability & Instrumentation

You can’t fix what you can’t see.  The BEAM emits rich Telemetry – use it.

```elixir
:telemetry.execute([
  :my_app, :genserver, :callback, :stop
],
%{duration: duration},
%{module: __MODULE__, callback: :handle_call})
```

Pipe these events into PromEx → Grafana or Datadog.  Add tracing (`OpenTelemetry`) around external calls to stitch latency graphs end-to-end.  Set *budgets* (SLOs) and alert on 95th percentile, not averages.

---

## 4. Conclusion

A GenServer is a beautiful abstraction, but it hides sharp edges.  With a clear mental model and a small set of techniques – non-blocking callbacks, state externalisation, batching, back-pressure, and solid instrumentation – you can take that weekend prototype and run it under serious production load.

Next up in this series: **distributed GenServers and cluster-wide coordination** – we’ll tackle hand-off, global registries, and truly elastic scaling.  Stay tuned.

---

## Key Takeaways

* Non-blocking callbacks keep schedulers healthy.
* Move highly contended reads to ETS or `persistent_term`.
* Instrument first, optimise second.
* Use supervision and back-pressure patterns to stay resilient.

## Resources & Further Reading

* Official OTP docs – `gen_server`, `:erlang.process_info/2`.
* Fred Hébert – *“Adopting Erlang/OTP”* chapters on monitoring.
* Saša Jurić – *“Elixir in Action”* sections on performance.
* Erlang Solutions – *“Designing for Scalability with Erlang/OTP”*. 