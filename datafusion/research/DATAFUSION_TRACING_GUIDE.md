# DataFusion Source Code Tracing Manifest (Hyper-Granular)

**Instructions for Claude:**
This is an atomic task list for analyzing the Apache DataFusion source code (and its underlying Apache Arrow components). **DO NOT attempt to execute multiple tasks at once.** The user will specify a Task ID (e.g., "Claude, execute Task 1.1.A"). 
1. Read the specified source files using your CLI tools to understand the implementation.
2. **"Target Files" are a suggested starting point only** — they are not exhaustive. Automatically expand your research scope to any related dependencies, callers, implementations, or utilities that are relevant to the Focus. Follow the code wherever it leads.
3. Analyze the code deeply based on the specific "Focus" provided.
4. **Provide code snippets** for key concepts in your explanation. Quote relevant source code directly to support your analysis rather than describing it abstractly.
5. Generate the output exactly matching the `RESEARCH_TEMPLATE.md` structure.
6. Stop and wait for the user to verify the output and provide the next command.

---

## Phase 1: Foundation Tracing Guide (DataFusion)
**Objective:** Understand Arrow's physical layout and DataFusion's async ingestion path to contrast with Trino's memory SPI.

### Buffer
* **Task 1.1: The `Buffer` and Arrow Memory Management**
  * **Target Crates/Files:** `arrow-buffer` (`src/buffer/immutable.rs`, `src/bytes.rs`)
  * **Focus:** Trace how `Buffer` manages underlying `Bytes`. Analyze the implementation of `Buffer::slice()`. How does it use `Arc` for shared ownership? How is 64-byte alignment enforced?

### Array
* **Task 1.2: Columnar Construction and Bitmaps (`Array`)**
  * **Target Crates/Files:** `arrow-array` (`src/array/primitive_array.rs`, `src/array/string_array.rs`)
  * **Focus:** Inspect the internal fields of a `StringArray`. How does it map to multiple `Buffer`s? Trace the `NullBuffer` implementation — how does Arrow pack 8 null values into a single byte, and how does this contrast with Trino's `boolean[]`?

### RecordBatch
* **Task 1.3: The `RecordBatch` Envelope**
  * **Target Crates/Files:** `arrow-array` (`src/record_batch.rs`)
  * **Focus:** Analyze `RecordBatch::project()`. Confirm that it is a zero-copy operation manipulating `Arc<dyn Array>`. Compare its memory overhead to Trino's `Page`.

### Physical Data Mapping
* **Task 1.4: Async Physical Data Mapping (The S3 Bridge)**
  * **Target Crates/Files:** `datafusion-physical-plan` (`src/parquet/mod.rs`), `parquet` (`src/arrow/async_reader/mod.rs`), `object_store`
  * **Focus:** Trace the execution of `ParquetExec::execute()`. How does it interact with `object_store` to fetch byte ranges asynchronously? Trace `ParquetRecordBatchStream` to see how `tokio::spawn` is used to parallelize I/O and decoding.

### Memory Accounting
* **Task 1.5: RAII Memory Accounting**
  * **Target Crates/Files:** `datafusion-execution` (`src/memory_pool/mod.rs`)
  * **Focus:** Analyze the `MemoryReservation` struct. Trace its `try_grow()` method and its `Drop` implementation. How does this Rust-native approach prevent the under-counting issues possible in Trino's GC-based `getRetainedSizeInBytes()` model?

---

## Phase 2: Execution Model Tracing Guide (DataFusion)
**Objective:** Trace the physical plan instantiation and the async stream execution pipeline. Understand how DataFusion leverages `tokio` for concurrency instead of building custom task schedulers.

### Physical Plan Contract
* **Task 2.1: The Physical Plan Contract (`ExecutionPlan` & Partitioning)**
  * **Target Crates/Files:** `datafusion-physical-plan` (`src/lib.rs`, `src/execution_plan.rs`), `datafusion-physical-expr` (`src/partitioning.rs`)
  * **Focus:** Analyze the `ExecutionPlan` trait. Look at `properties()` (which defines partitioning and ordering via `PlanProperties`) and the `execute()` method. How does `execute()` take a partition index and a `TaskContext` to return a `SendableRecordBatchStream`? Trace how `output_partitioning()` declares the degree of concurrency. Analyze the `Partitioning` enum (`RoundRobinBatch`, `Hash`, `UnknownPartitioning`) and the `Distribution` enum. How does the optimizer bridge the gap between `required_input_distribution()` and `output_partitioning()` by inserting `RepartitionExec` or `CoalescePartitionsExec`?

### Execution Context
* **Task 2.2: The Execution Context (`TaskContext` & Resource Wiring)**
  * **Target Crates/Files:** `datafusion-execution` (`src/task.rs`, `src/runtime_env.rs`)
  * **Focus:** Trace the `TaskContext`. How is it wired up before execution begins? How does it link the `MemoryPool`, `DiskManager`, `ObjectStoreRegistry`, and session properties to the execution of a specific partition? Compare it to Trino's `DriverContext` — notice how lightweight it is, relying on Rust's call stack rather than a complex tracking tree.

### The Stream Lifecycle
* **Task 2.3.A: Stream Initialization**
  * **Target Crates/Files:** `datafusion-physical-plan` (`src/stream.rs`, `src/filter.rs`, `src/projection.rs`)
  * **Focus:** How is an operator instantiated into an active stream? Trace `FilterExec::execute()` — it returns a `FilterExecStream`. How does `RecordBatchStreamAdapter` wrap arbitrary futures into streams? Trace how the nested stream chain is constructed when `execute()` calls its child's `execute()`.
* **Task 2.3.B: The Pull-Based Execution Loop (`poll_next`)**
  * **Target Crates/Files:** `datafusion-physical-plan` (`src/stream.rs`, `src/filter.rs`), `datafusion-physical-expr-common` (`src/metrics/baseline.rs`)
  * **Focus:** This is the core engine loop. Trace how `poll_next()` cascades down the stream chain. Note how the `ready!` macro bubbles up `Poll::Pending` asynchronously. Document where the stream hits `.await` points that yield control back to Tokio. Contrast with Trino's `Driver` loop manually checking `operator.needsInput()` and `operator.isBlocked()`. How does `BaselineMetrics::record_poll()` wrap every stream for observability?
* **Task 2.3.C: Stream Termination & Cancellation**
  * **Target Crates/Files:** `datafusion-physical-plan` (operator stream implementations), `datafusion-common-runtime` (`src/common.rs`, `src/join_set.rs`)
  * **Focus:** Two shutdown paths: normal completion and cancellation. For normal completion, trace the path when `poll_next()` returns `Poll::Ready(None)` — how do `MemoryReservation` drops automatically release memory back to the pool? For cancellation, trace the drop-based protocol: how does `SpawnedTask`'s abort-on-drop guarantee cleanup? How do channel closures propagate shutdown to background tasks? Contrast with Trino's explicit `CANCELING → CANCELED` state machine and `DriverAndTaskTerminationTracker`.

### Intra-Node Concurrency
* **Task 2.4.A: Tokio Task Spawning**
  * **Target Crates/Files:** `datafusion-common-runtime` (`src/common.rs`, `src/join_set.rs`), `datafusion-physical-plan` (`src/common.rs`, `src/execution_plan.rs`)
  * **Focus:** How are multiple partitions mapped to actual CPU threads? Trace `SpawnedTask` (abort-on-drop wrapper around `tokio::task::JoinHandle`), `JoinSet` (managed task set), `spawn_buffered()` (channel-based stream decoupling), and `collect_partitioned()` (one Tokio task per partition). Why is raw `tokio::spawn` banned in operator code?
* **Task 2.4.B: Local Repartitioning (The Exchange)**
  * **Target Crates/Files:** `datafusion-physical-plan` (`src/repartition/mod.rs`)
  * **Focus:** Trace `RepartitionExec`. How does it take `N` input streams and map them to `M` output streams? Analyze how it uses `tokio::sync::mpsc` channels to move `RecordBatch`es across thread boundaries. How does it handle backpressure (channel capacity)? How does hash vs. round-robin routing work?

### Distributed Orchestration
* **Task 2.5: Distributed Orchestration Context (Optional)**
  * **Target Repository:** `apache/datafusion-ballista` or `apache/datafusion-ray`
  * **Focus:** Briefly look at how an external scheduler wraps DataFusion. How does Ballista take an `ExecutionPlan`, break it at `ShuffleWriterExec` boundaries, and distribute those as tasks to remote executors?

---

## Phase 3: The Operator Pipeline (Physical Plan Execution)
**Objective:** Trace the physical data processing nodes to understand streaming query execution.

* **Task 3.1.A: Physical Expressions (`PhysicalExpr`)**
  * **Target Crates/Files:** `datafusion-physical-expr/src/physical_expr.rs`
  * **Focus:** Analyze how column references and scalar math are evaluated against a `RecordBatch`. Trace the `evaluate()` method.
* **Task 3.2.A: Simple Projection & Filtering**
  * **Target Crates/Files:** `datafusion-physical-plan/src/filter.rs`, `datafusion-physical-plan/src/projection.rs`
  * **Focus:** Trace the simplest data flow: taking an input stream, applying a boolean mask (Filter), and returning a transformed `RecordBatch` (Projection).
* **Task 3.3.A: Hash Join - Building the Table**
  * **Target Crates/Files:** `datafusion-physical-plan/src/joins/hash_join.rs` (focus on build-side logic)
  * **Focus:** How does DataFusion ingest the "build" side of a join? Analyze the `JoinHashMap` structure and how it stores indices.
* **Task 3.3.B: Hash Join - Probing**
  * **Target Crates/Files:** `datafusion-physical-plan/src/joins/hash_join.rs` (focus on probe-side logic)
  * **Focus:** How does the "probe" side stream through and match against the built hash table? Trace the batch-level joining logic.
* **Task 3.4.A: Aggregation & Accumulators**
  * **Target Crates/Files:** `datafusion-physical-plan/src/aggregates/mod.rs`, `datafusion-physical-expr/src/aggregate/accumulator.rs`
  * **Focus:** How does an `AggregateExec` maintain group-by state? Analyze the `Accumulator` trait (`update_batch`, `evaluate`).
* **Task 3.5.A: Sorting and Spilling Mechanism**
  * **Target Crates/Files:** `datafusion-physical-plan/src/sorts/sort.rs`, `datafusion-physical-plan/src/sorts/sort_preserving_merge.rs`
  * **Focus:** When sorting data that exceeds memory, how does `SortExec` serialize its state to disk? Trace the spilling trigger and temp file creation.

---

## Phase 4: Network & Data Exchange (Shuffle)
**Objective:** Understand how DataFusion shuffles data between partitions and threads locally, as well as the mechanisms for over-the-wire exchange.

* **Task 4.1.A: Local Repartitioning (The Exchange)**
  * **Target Crates/Files:** `datafusion-physical-plan/src/repartition/mod.rs`
  * **Focus:** How does `RepartitionExec` take `N` input streams and shuffle them into `M` output streams?
* **Task 4.1.B: MPSC Channel Shuffling**
  * **Target Crates/Files:** `datafusion-physical-plan/src/repartition/mod.rs` (focus on channel routing)
  * **Focus:** Trace the use of Tokio MPSC (Multi-Producer, Single-Consumer) channels to move `RecordBatch`es across async task boundaries.
* **Task 4.1.C: Hash vs Round-Robin Routing**
  * **Target Crates/Files:** `datafusion-physical-plan/src/common/ipc.rs` or routing logic in `repartition`
  * **Focus:** Trace the hash calculation for partitioning. How does it ensure specific rows are routed to the correct downstream partition?
* **Task 4.2.A: Over-The-Wire Encoding (Arrow Flight)**
  * **Target Crates/Files:** `arrow-flight/src/encode.rs`, `arrow-flight/src/decode.rs`
  * **Focus:** How are `RecordBatch` streams converted into gRPC FlightData messages for network transfer?

---

## Phase 5: Memory Tracking & Arbitration (Memory Management)
**Objective:** Map out the strict memory accounting system used to prevent out-of-memory errors.

* **Task 5.1.A: The `MemoryPool` Trait**
  * **Target Crates/Files:** `datafusion-execution/src/memory_pool/mod.rs`
  * **Focus:** Analyze the core `MemoryPool` trait. How are `grow()` and `shrink()` methods defined to track global JVM/process memory?
* **Task 5.1.B: Pool Implementations**
  * **Target Crates/Files:** `datafusion-execution/src/memory_pool/pool.rs`
  * **Focus:** Compare `GreedyMemoryPool` (first-come, first-served) and `FairSpillPool` (ensuring even distribution among tasks).
* **Task 5.2.A: The `MemoryReservation` Lifecycle**
  * **Target Crates/Files:** `datafusion-execution/src/memory_pool/mod.rs` (focus on `MemoryReservation` struct)
  * **Focus:** Trace the RAII pattern here. How does a `MemoryReservation` guarantee that memory is accurately reported and freed when an operator is dropped?
* **Task 5.2.B: Memory Consumers**
  * **Target Crates/Files:** `datafusion-execution/src/memory_pool/mod.rs` (focus on `MemoryConsumer`)
  * **Focus:** How does a specific operator register itself as a consumer? Trace the mechanism an operator uses to request an allocation size.
