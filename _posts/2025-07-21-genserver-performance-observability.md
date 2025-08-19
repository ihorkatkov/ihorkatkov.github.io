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

Remember the last time you shipped a shiny new GenServer to production? It passed every unit test, handled your happy-path demo traffic, and looked rock-solid on paper. Then real users showed up. Latency spikes, CPU climbs, and suddenly the BEAM scheduler view in `:observer` looks like a Christmas tree. I've been there - and I've learned that **building a GenServer is the easy part**; making it *fast, observable, and bulletproof* is where the real work starts.

In my previous article I walked through a TDD approach to *testing* GenServers. This follow-up is the field manual I wish I had when I first pushed one of those servers to production. We'll build a mental model of how GenServers really cost CPU cycles, then apply a toolbox of performance and observability techniques you can drop into your code today.

By the end you'll know how to:

1.  Read the BEAM's “cost model” - mailbox size, scheduler reductions, message queue length - so you can spot trouble early.
2.  Refactor hot paths so callbacks never block schedulers.
3.  Push read-heavy state to ETS / `persistent_term` without losing consistency.
4.  Add cheap, composable Telemetry so dashboards light up before pagers do.
5.  Choose when to graduate from a single GenServer to GenStage, Broadway, or full-blown distributed sharding.

Let's dive in.

---

## 1. The Real GenServer Cost Model *(mental model)*

A GenServer is just a process with a mailbox, but the devil is in the scheduler details. The BEAM VM runs **N** schedulers - one per CPU core by default - and each scheduler works through a run queue of processes. Key things to watch:

*   **Mailbox size** - [`Process.info(pid, :message_queue_len)`](https://hexdocs.pm/elixir/Process.html#info/2) tells you how many messages are waiting. A consistently growing queue is a red flag, as an overloaded mailbox can delay its own replies, inflating end‑to‑end latency of other processes on the same scheduler.
*   **Reductions** - every BEAM operation costs reductions; long-running callbacks burn the budget, delaying other work.
*   **Scheduler migrations** - when a process hogs a scheduler for too long it triggers load balancing and the VM may migrate it to a different scheduler core. This context switch can lead to CPU cache misses as the process's data is no longer in the local L1/L2 cache, introducing latency.
*   **Sync vs. async** - `GenServer.call/3` blocks the caller; `cast/2` doesn't. Calls are convenient but couple lifecycles and back-pressure.

Tools I keep on my belt:

```elixir
:observer.start()
:recon.proc(:info)
# A library like telemetry_metrics_statsd to consume telemetry events
```

Spend five minutes watching these metrics during load and your optimisation story usually writes itself.

---

## 2. Performance & Throughput Techniques

### 2.1 Keep Callbacks Non-Blocking

If a callback waits on disk, network, or heavy CPU, **your entire GenServer stalls**. The key is to move blocking work out of the GenServer's main loop. The [`Task`](https://hexdocs.pm/elixir/Task.html) module provides several patterns for this.

For "fire-and-forget" work where the caller doesn't need a result, `Task.start/1` offloads the work into a new, linked process. The GenServer can immediately process the next message.

```elixir
def handle_cast({:track_event, event}, state) do
  # This task is linked to the GenServer. If it crashes, the GenServer crashes.
  Task.start(fn -> Analytics.track(event) end)
  {:noreply, state}
end
```

When a result is needed but you can't block the GenServer, a common pattern is to have the GenServer start a task and return it to the caller. The caller then `Task.await/1`s the result. This frees the GenServer while the client waits.

```elixir
# In the GenServer
def handle_call({:compute, input}, _from, state) do
  task = Task.async(fn -> heavy_math(input) end)
  {:reply, {:ok, task}, state}
end

# In a client module
def compute(server, input) do
  {:ok, task} = GenServer.call(server, {:compute, input})
  Task.await(task, 30_000) # Always use a timeout!
end
```

For background jobs that shouldn't be linked to your GenServer, use a [`Task.Supervisor`](https://hexdocs.pm/elixir/Task.Supervisor.html) to run them as supervised, independent processes.

**The Goal:** Keep your `handle_call` and `handle_cast` callbacks consistently fast (a good budget is <1ms). When profiling reveals a slow callback, delegate the work using one of these patterns.

### 2.2 Post-Init Heavy Work with `handle_continue`

Boot time matters when your GenServer sits inside a supervision tree - a slow `init/1` delays the whole app. Load large datasets *after* the process is up:

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

### 2.3 Externalize Read-Heavy State (ETS / `persistent_term`)

A GenServer's state is its bottleneck; every read is a serialized request. For highly contended data, moving state to [`:ets`](https://www.erlang.org/doc/man/ets.html) or [`:persistent_term`](https://www.erlang.org/doc/man/persistent_term.html) can unlock massive read concurrency. But this power comes with sharp trade-offs.

ETS tables, especially with `read_concurrency: true`, offer fast, parallel reads. However, this comes at a cost:
- **Write Serialization:** By default, all writes are still serialized through the single process that owns the table. Consider using `true` or `auto` (OTP 25+) for `write_concurrency`. Different objects of the same table can be mutated (and read) by concurrent processes. This is achieved to some degree at the expense of memory consumption and the performance of sequential access and concurrent reading.
- **Consistency:** `read_concurrency` can lead to dirty reads. A reader might see a partially updated record if a write is happening concurrently.
- **Ownership:** The table's lifecycle is tied to the owner process. If it dies, the table vanishes.

For truly static data that is read often and written almost never, `:persistent_term` is a powerful alternative. Reads are virtually free—no message passing, no memory copies, no GC impact. The catch is that `persistent_term.put/2` is a globally blocking operation that can cause a multi-millisecond pause across the entire BEAM (on a modern OTP (25+) it's typically sub‑millisecond for small updates). It should only be used for data that is set once at application boot or updated very rarely during a maintenance window.

**Verdict:** Use these tools surgically. Profile your application, understand the read/write ratio, and always measure the performance impact of both reads *and* writes before committing to this pattern.

### 2.4 Batching & Coalescing Patterns

Sometimes the cheapest optimisation is *do less*. Accumulate writes and flush every X milliseconds:

```elixir
def init(_opts) do
  schedule_flush()
  {:ok, %{buffer: []}}
end

def handle_cast({:track, metric}, state) do
  {:noreply, %{state | buffer: [metric | state.buffer]}}
end

def handle_info(:flush, state) do
  schedule_flush()
  flush(state.buffer)
  {:noreply, %{state | buffer: []}}
end

defp schedule_flush do
  Process.send_after(self(), :flush, 1000)
end
```

Used sparingly, batching smooths traffic spikes without complex back-pressure logic.

### 2.5 Back-Pressure & Demand Control

If producers outpace your GenServer, queues explode. Options:

1.  **Bounded mailbox** - set a max queue length and reject or drop messages after a threshold.
2.  **Timeouts on `call/3`** - force callers to handle slowness.

```elixir
  @impl true
  def handle_call({:process, _item}, _from, state) do
    # Check the mailbox size first.
    case Process.info(self(), :message_queue_len) do
      {:message_queue_len, len} when len > @max_queue_len ->
        # "Reject" the call because the server is overloaded.
        {:reply, {:error, :overloaded}, state}

      _ ->
        # Mailbox is not full, process the request.
        # ... do actual work ...
        {:reply, :ok, %{state | processed: state.processed + 1}}
    end
  end

```

Consider moving to [`GenStage`](https://hexdocs.pm/gen_stage/GenStage.html) or [`Broadway`](https://hexdocs.pm/broadway/Broadway.html) when:

*   You need standardized, pull-based back-pressure across a multi-stage data processing pipeline.
*   Your workload naturally fits a consumer-producer model (e.g., consuming from SQS).
*   You need concurrent processing of events while preserving order within a partition.

Migration can be incremental. You can embed a `GenStage` producer inside an existing `GenServer` and fan out from there.

### 2.6 Sharding Hot Keys

One GenServer → one mailbox. Hot keys will hit the limit. Partition with a [`Registry`](https://hexdocs.pm/elixir/Registry.html):

```elixir
key = :erlang.phash2(customer_id, 16)
# Note: if `customer_id` is user-controllable, this could be
# vulnerable to hash-collision attacks creating a hot shard.
{:ok, pid} = MyShardSupervisor.start_child(key)
```

Or reach for libraries like [`hash_ring`](https://github.com/g-andrade/hash_ring). 
---

## 3. Observability & Instrumentation

You can't fix what you can't see. The BEAM emits rich [`:telemetry`](https://github.com/beam-telemetry/telemetry) events - use them.

```elixir
:telemetry.execute([
  :my_app, :genserver, :callback, :stop
],
%{duration: duration},
%{module: __MODULE__, callback: :handle_call})
```

Pipe these events into a library like `PromEx` to expose them to Grafana or Datadog. Add tracing (`OpenTelemetry`) around external calls to stitch latency graphs end-to-end. Set *budgets* (SLOs) and alert on 95th percentile, not averages.

---

## 4. Conclusion

A GenServer is a beautiful abstraction, but it hides sharp edges. With a clear mental model and a small set of techniques - non-blocking callbacks, state externalisation, batching, back-pressure, and solid instrumentation - you can take that weekend prototype and run it under serious production load.

Every optimization is a trade-off. Always profile your application to identify true bottlenecks before adding complexity. Instrument first, optimise second.

Next up in this series: **distributed GenServers and cluster-wide coordination** - we'll tackle hand-off, global registries, and truly elastic scaling. Stay tuned.
---

## Key Takeaways

*   Non-blocking callbacks keep schedulers healthy.
*   Move highly contended reads to ETS or `:persistent_term`, but understand the trade-offs.
*   Instrument first, optimise second.
*   Use supervision and back-pressure patterns to stay resilient.

## Resources & Further Reading

*   Official OTP docs - [`gen_server`](https://www.erlang.org/doc/man/gen_server.html), [`:erlang.process_info/2`](https://www.erlang.org/doc/man/erlang.html#process_info-2).
*   Fred Hébert - *“Adopting Erlang/OTP”* chapters on monitoring.
*   Saša Jurić - *“Elixir in Action”* sections on performance.
*   Erlang Solutions - *“Designing for Scalability with Erlang/OTP”*. 