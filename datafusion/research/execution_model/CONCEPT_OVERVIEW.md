# Phase 2: Execution Model Overview (Physical Plans & Async Streams)

## Table of Contents
- [1. Single-Node Core vs. Distributed Extensions](#1-single-node-core-vs-distributed-extensions)
- [2. The Execution Hierarchy: From Plan to Streams](#2-the-execution-hierarchy-from-plan-to-streams)
- [3. The Async Pull Model (Volcano over Tokio)](#3-the-async-pull-model-volcano-over-tokio)
  - [Operator Patterns](#operator-patterns)
  - [The `ready!()` Macro](#the-ready-macro)
  - [Metrics and Observability](#metrics-and-observability)
- [4. The Scheduling Engine: Tokio (Not a Custom Scheduler)](#4-the-scheduling-engine-tokio-not-a-custom-scheduler)
  - [Controlled Spawning](#controlled-spawning)
- [5. Repartitioning: The Internal Exchange](#5-repartitioning-the-internal-exchange)
- [6. Drop-Based Cancellation (No State Machine)](#6-drop-based-cancellation-no-state-machine)
- [7. The `TaskContext`: Resource Wiring](#7-the-taskcontext-resource-wiring)
- [Summary: Connecting the Dots](#summary-connecting-the-dots)

If Trino's execution model is a custom-built, highly choreographed factory floor with its own shift managers (Drivers), DataFusion's model is an elegant water pipe system leveraging gravity (Rust's async/await and the `tokio` runtime) to flow data through.

## 1. Single-Node Core vs. Distributed Extensions

**Crucial Context:** Trino is inherently a distributed cluster. DataFusion Core is an **embeddable, single-node query engine**.
* In Trino, Phase 2 involved shattering a query into Stages and moving them across a network.
* In DataFusion, the core engine only shatters a query across **CPU cores on a single machine**. To get distributed execution (like Trino), you must wrap DataFusion with a distributed scheduler like **Ballista** or **DataFusion-Ray**. Because you are researching for a worker rewrite, we will focus on DataFusion's single-node physical execution, which represents what runs *inside* a single distributed worker.

## 2. The Execution Hierarchy: From Plan to Streams

DataFusion compiles a SQL string into a logical plan, and then optimizes it into a **Physical Plan**. The hierarchy is dramatically flatter than Trino's (Query → Stage → Task → Pipeline → Driver → Operator).

* **`ExecutionPlan` (The Blueprint):** A trait representing a physical operation (e.g., `DataSourceExec` for scanning, `FilterExec`, `HashJoinExec`). The plan forms a DAG (Directed Acyclic Graph). It doesn't process data; it knows *how* to process data. Every `ExecutionPlan` caches its `PlanProperties` at construction time — a struct holding `Partitioning`, `EquivalenceProperties`, `EmissionType` (Incremental/Final/Both), `Boundedness` (Bounded/Unbounded), `EvaluationType` (Lazy/Eager), and `SchedulingType` (Cooperative/NonCooperative). These six properties fully describe an operator's execution characteristics.
* **Partitions (Concurrency):** Every `ExecutionPlan` exposes a partition count via `output_partitioning().partition_count()`. If `DataSourceExec` has 16 partitions, DataFusion can scan the files using 16 concurrent streams. This is the exact equivalent of Trino spawning multiple Drivers for different Splits.
* **`Distribution` vs `Partitioning`:** `Distribution` expresses a *requirement* (what an operator needs from its input — `SinglePartition`, `HashPartitioned(exprs)`, or `UnspecifiedDistribution`). `Partitioning` describes a *property* (what an operator actually produces). The **`EnforceDistribution`** physical optimizer rule bridges the gap in two phases: (1) a top-down pass reorders hash partition keys to maximize partition reuse across joins, then (2) a bottom-up pass inserts `RepartitionExec`, `CoalescePartitionsExec`, or `SortPreservingMergeExec` nodes where `required_input_distribution()` isn't satisfied. `EvaluationType::Eager` marks operators that are data exchange points — critical for distributed engines to identify network shuffle boundaries.
* **`SendableRecordBatchStream` (The Worker):** When you call `execute(partition, context)` on an `ExecutionPlan`, it returns a `SendableRecordBatchStream` (a `Pin<Box<dyn RecordBatchStream + Send>>`). This is the instantiation of the work. The `execute()` method itself is deliberately *not* async — it creates and returns a stream immediately. Computation only begins when the stream is polled.

## 3. The Async Pull Model (Volcano over Tokio)

DataFusion uses a pull-based Volcano model, but it is entirely asynchronous.

* **Nested Streams:** Operators are nested. The top-level stream (e.g., a Limit) holds a reference to its child (e.g., a Sort), which holds a reference to its child (e.g., a Scan). Each `poll_next()` call cascades down synchronously within a single `Future::poll()`. When any level returns `Poll::Pending`, the entire cascade unwinds and the Tokio task is parked until a waker fires.
* **No Custom Drivers:** Unlike Trino, which has a `Driver` loop manually shuttling `Page`s between `Operator`s and checking a 1-second time quantum, DataFusion operators simply implement the Rust `Stream` trait.
* **Yielding via `.await`:** When a DataFusion operator is blocked (e.g., waiting for S3 I/O, or waiting for a hash table to build), it hits an `.await` point. The `tokio` async runtime automatically suspends that task and schedules another one on the CPU thread. There is no custom `DriverYieldSignal` required.

### Operator Patterns

Unlike Trino's uniform `needsInput()`/`addInput()`/`getOutput()` state machine, DataFusion operators implement `poll_next()` directly with pattern-specific logic:

| Pattern | Example | Behavior |
|---|---|---|
| **Map (stateless)** | `ProjectionStream` | Polls input once, transforms the batch, returns immediately |
| **Filter (stateless + coalescing)** | `FilterExecStream` | Polls input in a loop, applies predicate, coalesces small batches to meet `batch_size` |
| **Accumulate-then-emit** | `SortExec` | Uses `once(async { consume_all; sort }).try_flatten()` to defer accumulation. Dual MemoryReservation (buffer + merge). Spills to disk on OOM. |
| **Dual-input** | `HashJoinStream` | Build side collected via `OnceFut` (shared across partitions), probe side streamed. 5-state machine: WaitBuild -> FetchProbe -> ProcessProbe -> ExhaustedProbe -> Completed |
| **OOM-aware aggregate** | `GroupedHashAggregateStream` | Three OOM modes: EmitEarly (partial), Spill (full), ReportError (ordered). Skip aggregation probe monitors group/row ratio. |
| **K-way merge** | `SortPreservingMergeStream` | Loser tree tournament: O(log N) comparisons per element. Round-robin tie-breaker prevents upstream buffer imbalance. |
| **Channel-based** | `RepartitionExec` | Custom Gate-based channels (not tokio::mpsc). Hash/round-robin routing. Spill-to-disk on memory pressure. |

### The `ready!()` Macro

The `ready!()` macro is DataFusion's equivalent of Trino's "check if blocked" pattern:
```rust
let batch = ready!(self.input.poll_next(cx));
```
If the input returns `Poll::Pending`, `ready!()` immediately returns `Poll::Pending` from the current function — yielding the thread. If it returns `Poll::Ready`, execution continues. This single macro replaces Trino's `isBlocked()` check + `ListenableFuture` return.

### Metrics and Observability

Every stream is wrapped by `ObservedStream` (via `BaselineMetrics::record_poll()`), which records output rows, bytes, batches, and elapsed compute time — analogous to Trino's `OperatorContext` timing stats.

## 4. The Scheduling Engine: Tokio (Not a Custom Scheduler)

This is the most significant architectural difference from Trino. DataFusion has **no custom thread pool, no priority queue, no multilevel feedback queue, and no time quanta**.

* **Tokio Work-Stealing:** All partition streams are multiplexed onto Tokio's fixed-size thread pool (default: 1 thread per CPU core). Tokio's work-stealing scheduler distributes work across threads automatically.
* **Implicit Yielding:** Where Trino Drivers check a `DriverYieldSignal` to cooperatively yield after 1 second, DataFusion streams yield implicitly at every `.await` point. When `poll_next()` returns `Poll::Pending`, the stream yields the thread back to Tokio.
* **Cooperative Scheduling via `CooperativeStream`:** The `EnsureCooperative` optimizer automatically wraps non-cooperative leaf and exchange nodes with `CooperativeExec`, which injects `CooperativeStream`. This wrapper consumes Tokio's task budget per batch produced (budget of 128), ensuring long-running operators yield to the scheduler. Three compile-time modes: direct `poll_proceed()`, `consume_budget()` fallback, or manual per-stream counter. This is DataFusion's answer to Trino's `DriverYieldSignal` — but enforced automatically at the optimizer level rather than manually in each operator.
* **No Priority System:** All streams are equal. There is no equivalent of Trino's 5-level priority queue that gives short queries 16x the CPU share. This is a tradeoff: simpler implementation, but no fairness guarantees across concurrent queries.

### Controlled Spawning

DataFusion restricts when and how Tokio tasks are spawned:

* **`SpawnedTask`:** A wrapper around `tokio::task::JoinHandle` that **aborts the task on `Drop`**. Raw `tokio::spawn` is banned in operator code — the `ExecutionPlan::execute()` docs explicitly state this. Orphaned tasks would survive query cancellation. All spawned futures are wrapped with `trace_future()` for global tracer injection (OpenTelemetry, Jaeger) before being handed to Tokio.
* **`spawn_buffered()`:** Decouples a producer stream from its consumer via a bounded channel and a background `SpawnedTask`. This enables pipeline parallelism — a sort can produce batches in the background while the consumer processes them concurrently.
* **`collect_partitioned()`:** The top-level API that spawns one Tokio task per partition via `JoinSet`, driving all partition streams concurrently.

## 5. Repartitioning: The Internal Exchange

Because DataFusion core runs on a single node, its "Exchange" mechanism is just passing data between threads in memory.

* **`RepartitionExec`:** When `EnforceDistribution` determines that data must be redistributed across cores (e.g., before a Hash Join), it inserts a `RepartitionExec`. Supports two routing modes: `Hash(exprs, N)` for key-based distribution and `RoundRobinBatch(N)` for load balancing.
* **Custom Gate-based Channels (not tokio::mpsc):** `RepartitionExec` uses custom `DistributionSender`/`DistributionReceiver` pairs backed by a shared `Gate` for backpressure — not Tokio's standard MPSC channels. Each input partition spawns a background task that pulls batches and routes them to the appropriate output channel. The `Gate` mechanism allows receivers to signal backpressure, pausing senders when downstream is slow. Spill-to-disk support handles memory pressure via `IPCWriter`/`IPCReader`. A preserve-order mode maintains input ordering by creating N×M dedicated channels (one per input-output pair) and merging with `StreamingMerge`.
* **`CoalescePartitionsExec`:** Merges N partition streams into a single output stream. Used when an operator requires `SinglePartition` distribution (e.g., a global sort or final aggregation). The top-level `execute_stream()` API automatically wraps multi-partition plans in this.

## 6. Drop-Based Cancellation (No State Machine)

DataFusion has no explicit cancellation protocol like Trino's `CANCELING → CANCELED` state machine. Instead, cancellation is implicit through Rust's ownership model:

1. The consumer drops the stream (or the `JoinSet` holding the tasks).
2. `SpawnedTask::drop()` calls `handle.abort()` on the Tokio task.
3. The aborted task's captured state (streams, reservations, channels) is dropped.
4. `MemoryReservation::drop()` returns memory to the pool.
5. `RefCountedTempFile::drop()` deletes spill files from disk and decrements the `DiskManager` usage counter.
6. Channel senders/receivers close, causing any background tasks to observe errors and exit.

Three cleanup paths exist — **natural completion** (stream returns `None`, resources freed on scope exit), **cancellation/drop** (abort cascade as above), and **error** (error propagated up, resources freed by Drop on unwind). All three guarantee: memory returned to pool, spill files deleted, metrics finalized, background tasks aborted. The key difference: natural completion allows final metrics flushing; cancellation may leave partial metrics.

This is Rust's ownership model doing the work that Trino's `DriverAndTaskTerminationTracker` does explicitly. There is no two-phase termination, no resource cleanup chain traversal — the compiler guarantees destructors run.

## 7. The `TaskContext`: Resource Wiring

`TaskContext` is DataFusion's equivalent of Trino's combined `TaskContext` + `DriverContext` + `OperatorContext`. It is a single struct shared (via `Arc`) across all streams in a query partition:

* **`SessionConfig`:** Query-level settings (`batch_size=8192`, `target_partitions=num_cpus`, `sort_spill_reservation_bytes=10MB`, `max_spill_file_size_bytes=128MB`, etc.)
* **`RuntimeEnv`:** Process-level resources built via `RuntimeEnvBuilder` — the `MemoryPool` (default: `UnboundedMemoryPool` — **no memory limit unless explicitly configured**), `DiskManager`, `CacheManager` (metadata + file list caches with TTL), and `ObjectStoreRegistry`. Production deployments should configure `FairSpillPool` or `GreedyMemoryPool` to prevent OOM.
* **`FunctionRegistry`:** UDF/UDAF/UDWF registrations — `TaskContext` itself implements the `FunctionRegistry` trait, delegating to its inner registries.

Unlike Trino's deep context hierarchy (Query → Task → Pipeline → Driver → Operator), DataFusion uses a single flat context. There is no per-pipeline or per-operator context — memory tracking is done at the operator level via `MemoryReservation`, which talks directly to the global `MemoryPool`.

## Summary: Connecting the Dots

1. A SQL query is compiled into a DAG of **`ExecutionPlan`** nodes, each caching `PlanProperties` (6 fields: `Partitioning`, `EquivalenceProperties`, `EmissionType`, `Boundedness`, `EvaluationType`, `SchedulingType`).
2. The **`EnforceDistribution`** optimizer compares `required_input_distribution()` vs `output_partitioning()` in a two-phase pass (top-down key reorder, bottom-up enforcement) and inserts **`RepartitionExec`/`CoalescePartitionsExec`/`SortPreservingMergeExec`** nodes to match requirements.
3. The **`EnsureCooperative`** optimizer wraps non-cooperative leaf and exchange nodes with `CooperativeExec`, injecting Tokio budget consumption to guarantee yielding.
4. For each partition, the engine calls `execute()`, yielding a **`SendableRecordBatchStream`**. The `execute()` method is synchronous — computation starts only when the stream is polled.
5. The streams are deeply nested. **Tokio's work-stealing thread pool** drives all streams concurrently — no custom scheduler, no priority queue, no time quanta.
6. Each stream's **`poll_next()` cascades** down the operator chain synchronously. The `ready!()` macro yields the thread when any upstream is not ready.
7. Data moves between CPU cores via **`RepartitionExec`** and custom **Gate-based channels** with backpressure, spill-to-disk, and optional order preservation.
8. **`SpawnedTask` + `spawn_buffered()`** enable pipeline parallelism while guaranteeing cleanup via abort-on-drop. All spawned tasks are instrumented via `trace_future()` for distributed tracing.
9. **Cancellation is implicit**: dropping a stream triggers a cascade of `Drop` implementations that abort tasks, close channels, delete spill files, and return memory to the pool.
