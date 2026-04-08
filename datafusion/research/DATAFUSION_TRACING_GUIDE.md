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

## Phase 1: Foundation Tracing Guide (Memory Layout)
**Objective:** Understand Arrow's physical layout and DataFusion's async ingestion path to contrast with Trino's memory SPI.

### Task 1.1: Buffer
* **Task 1.1: The `Buffer` and Arrow Memory Management**
  * **Target Crates/Files:** `arrow-buffer` (`src/buffer/immutable.rs`, `src/bytes.rs`)
  * **Focus:** Trace how `Buffer` manages underlying `Bytes`. Analyze the implementation of `Buffer::slice()`. How does it use `Arc` for shared ownership? How is 64-byte alignment enforced?

### Task 1.2: Array
* **Task 1.2: Columnar Construction and Bitmaps (`Array`)**
  * **Target Crates/Files:** `arrow-array` (`src/array/primitive_array.rs`, `src/array/string_array.rs`)
  * **Focus:** Inspect the internal fields of a `StringArray`. How does it map to multiple `Buffer`s? Trace the `NullBuffer` implementation — how does Arrow pack 8 null values into a single byte, and how does this contrast with Trino's `boolean[]`?

### Task 1.3: RecordBatch
* **Task 1.3: The `RecordBatch` Envelope**
  * **Target Crates/Files:** `arrow-array` (`src/record_batch.rs`)
  * **Focus:** Analyze `RecordBatch::project()`. Confirm that it is a zero-copy operation manipulating `Arc<dyn Array>`. Compare its memory overhead to Trino's `Page`.

### Task 1.4: Physical Data Mapping
* **Task 1.4: Async Physical Data Mapping (The S3 Bridge)**
  * **Target Crates/Files:** `datafusion-physical-plan` (`src/parquet/mod.rs`), `parquet` (`src/arrow/async_reader/mod.rs`), `object_store`
  * **Focus:** Trace the execution of `ParquetExec::execute()`. How does it interact with `object_store` to fetch byte ranges asynchronously? Trace `ParquetRecordBatchStream` to see how `tokio::spawn` is used to parallelize I/O and decoding.

### Task 1.5: Memory Accounting
* **Task 1.5: RAII Memory Accounting**
  * **Target Crates/Files:** `datafusion-execution` (`src/memory_pool/mod.rs`)
  * **Focus:** Analyze the `MemoryReservation` struct. Trace its `try_grow()` method and its `Drop` implementation. How does this Rust-native approach prevent the under-counting issues possible in Trino's GC-based `getRetainedSizeInBytes()` model?

---

## Phase 2: Worker Scheduling & The Driver (Execution Model)
**Objective:** Trace the physical plan instantiation and the async stream execution pipeline. Understand how DataFusion leverages `tokio` for concurrency instead of building custom task schedulers.

### Task 2.1: Physical Plan Contract
* **Task 2.1: The Physical Plan Contract (`ExecutionPlan` & Partitioning)**
  * **Target Crates/Files:** `datafusion-physical-plan` (`src/lib.rs`, `src/execution_plan.rs`), `datafusion-physical-expr` (`src/partitioning.rs`)
  * **Focus:** Analyze the `ExecutionPlan` trait. Look at `properties()` (which defines partitioning and ordering via `PlanProperties`) and the `execute()` method. How does `execute()` take a partition index and a `TaskContext` to return a `SendableRecordBatchStream`? Trace how `output_partitioning()` declares the degree of concurrency. Analyze the `Partitioning` enum (`RoundRobinBatch`, `Hash`, `UnknownPartitioning`) and the `Distribution` enum. How does the optimizer bridge the gap between `required_input_distribution()` and `output_partitioning()` by inserting `RepartitionExec` or `CoalescePartitionsExec`?

### Task 2.2: Execution Context
* **Task 2.2: The Execution Context (`TaskContext` & Resource Wiring)**
  * **Target Crates/Files:** `datafusion-execution` (`src/task.rs`, `src/runtime_env.rs`)
  * **Focus:** Trace the `TaskContext`. How is it wired up before execution begins? How does it link the `MemoryPool`, `DiskManager`, `ObjectStoreRegistry`, and session properties to the execution of a specific partition? Compare it to Trino's `DriverContext` — notice how lightweight it is, relying on Rust's call stack rather than a complex tracking tree.

### Task 2.3: The Stream Lifecycle
* **Task 2.3.A: Stream Initialization**
  * **Target Crates/Files:** `datafusion-physical-plan` (`src/stream.rs`, `src/filter.rs`, `src/projection.rs`)
  * **Focus:** How is an operator instantiated into an active stream? Trace `FilterExec::execute()` — it returns a `FilterExecStream`. How does `RecordBatchStreamAdapter` wrap arbitrary futures into streams? Trace how the nested stream chain is constructed when `execute()` calls its child's `execute()`.
* **Task 2.3.B: The Pull-Based Execution Loop (`poll_next`)**
  * **Target Crates/Files:** `datafusion-physical-plan` (`src/stream.rs`, `src/filter.rs`), `datafusion-physical-expr-common` (`src/metrics/baseline.rs`)
  * **Focus:** This is the core engine loop. Trace how `poll_next()` cascades down the stream chain. Note how the `ready!` macro bubbles up `Poll::Pending` asynchronously. Document where the stream hits `.await` points that yield control back to Tokio. Contrast with Trino's `Driver` loop manually checking `operator.needsInput()` and `operator.isBlocked()`. How does `BaselineMetrics::record_poll()` wrap every stream for observability?
* **Task 2.3.C: Stream Termination & Cancellation**
  * **Target Crates/Files:** `datafusion-physical-plan` (operator stream implementations), `datafusion-common-runtime` (`src/common.rs`, `src/join_set.rs`)
  * **Focus:** Two shutdown paths: normal completion and cancellation. For normal completion, trace the path when `poll_next()` returns `Poll::Ready(None)` — how do `MemoryReservation` drops automatically release memory back to the pool? For cancellation, trace the drop-based protocol: how does `SpawnedTask`'s abort-on-drop guarantee cleanup? How do channel closures propagate shutdown to background tasks? Contrast with Trino's explicit `CANCELING → CANCELED` state machine and `DriverAndTaskTerminationTracker`.

### Task 2.4: Intra-Node Concurrency
* **Task 2.4.A: Tokio Task Spawning**
  * **Target Crates/Files:** `datafusion-common-runtime` (`src/common.rs`, `src/join_set.rs`), `datafusion-physical-plan` (`src/common.rs`, `src/execution_plan.rs`)
  * **Focus:** How are multiple partitions mapped to actual CPU threads? Trace `SpawnedTask` (abort-on-drop wrapper around `tokio::task::JoinHandle`), `JoinSet` (managed task set), `spawn_buffered()` (channel-based stream decoupling), and `collect_partitioned()` (one Tokio task per partition). Why is raw `tokio::spawn` banned in operator code?
* **Task 2.4.B: Local Repartitioning (The Exchange)**
  * **Target Crates/Files:** `datafusion-physical-plan` (`src/repartition/mod.rs`)
  * **Focus:** Trace `RepartitionExec`. How does it take `N` input streams and map them to `M` output streams? Analyze how it uses `tokio::sync::mpsc` channels to move `RecordBatch`es across thread boundaries. How does it handle backpressure (channel capacity)? How does hash vs. round-robin routing work?

### Task 2.5: Distributed Orchestration
* **Task 2.5: Distributed Orchestration Context (Optional)**
  * **Target Repository:** `apache/datafusion-ballista` or `apache/datafusion-ray`
  * **Focus:** Briefly look at how an external scheduler wraps DataFusion. How does Ballista take an `ExecutionPlan`, break it at `ShuffleWriterExec` boundaries, and distribute those as tasks to remote executors?

---

## Phase 3: Operator Internals and Data Processing (Physical Plan Execution)
**Objective:** Trace the Async Volcano execution model. Map how `poll_next()` orchestrates simple nested pipelines, how complex pipelines handle asynchronous state boundaries, and how Arrow kernels execute the math.

### Task 3.1: The Async Volcano Contract
* **Task 3.1: The Async Volcano Contract (`Stream`)**
  * **Target Crates/Files:** `datafusion-physical-plan` (`src/stream.rs`), `futures` library (conceptually)
  * **Focus:** Analyze the `RecordBatchStream` trait. How does it extend Rust's `Stream`? Trace how the `poll_next()` macro `ready!` is used to bubble up `Poll::Pending` states asynchronously. Document the exact mechanical differences between Trino's explicit `addInput/getOutput` and DataFusion's implicit `poll_next()`.

### Task 3.2: Physical Expressions & Compute
* **Task 3.2: Physical Expressions & Compute Kernels**
  * **Target Crates/Files:** `datafusion-physical-expr` (`src/expressions/mod.rs`, `src/physical_expr.rs`), `arrow-ord` / `arrow-arith`
  * **Focus:** Analyze the `PhysicalExpr` trait and its `evaluate()` method. Trace a simple filter predicate. How does it evaluate to a `BooleanArray`? Trace the subsequent call to `arrow::compute::filter_record_batch` to see the SIMD application.

### Task 3.6: SIMD & Vectorized Compute `[KG-14]`
How does DataFusion/Arrow leverage SIMD hardware for columnar compute? This is the primary reference for the Rust worker's compute strategy.

* **Task 3.6.A: Arrow Compute Kernel SIMD Architecture**
    * **Target Crates/Files:** `arrow-arith/src/` (arithmetic kernels), `arrow-ord/src/` (comparison kernels), `arrow-select/src/filter.rs` (filter kernel), `arrow-string/src/` (string operations), `arrow-buffer/src/` (alignment), `datafusion-physical-expr/src/expressions/` (expression evaluation)
    * **Focus:** Trace how Arrow compute achieves SIMD:
      1. How does Arrow's **64-byte memory alignment** (cache-line aligned) enable SIMD? Trace the allocation path from `MutableBuffer` through `alloc::ALIGNMENT`.
      2. Does Arrow-rs use explicit SIMD intrinsics (`std::arch`, `packed_simd`, `portable-simd`), or does it rely on LLVM auto-vectorization of tight loops? Check for `#[target_feature]` attributes.
      3. Trace the inner loop of key kernels: `add_scalar`, `cmp_op`, `filter_array`, `cast_numeric`. Are these structured for auto-vectorization (no branches, contiguous arrays, `@llvm.assume.align`)?
      4. How does Arrow's **null bitmap** (LSB-first, bit-packed) interact with SIMD? Is the null check vectorized? Are there specialized "no-null" fast paths?
      5. How does `BooleanBufferBuilder` / bit-packing interact with SIMD for filter mask construction?
      6. What role does Rust's `#[inline]` and LTO play in cross-crate kernel inlining?

### Task 3.3: Simple Pipelines
* **Task 3.3: Simple Pipelines (Stateless Stream Nesting)**
  * **Target Crates/Files:** `datafusion-physical-plan` (`src/filter.rs`, `src/projection.rs`)
  * **Focus:** Trace the `poll_next()` loop in `FilterExecStream`. How does it pull a batch, apply the mask, and yield the result without row-by-row iteration? Confirm the zero-copy nature of `ProjectionExecStream`. Contrast this nested struct model with Trino's fused `ScanFilterAndProjectOperator`.

### Task 3.4: Complex Pipelines
* **Task 3.4: Complex Pipelines (Stateful Breakers & Joins)**
  * **Target Crates/Files:** `datafusion-physical-plan` (`src/joins/hash_join.rs`, `src/aggregates/mod.rs`)
  * **Focus:** Trace `HashJoinExec`. How does the async state machine transition between the Build phase and the Probe phase inside its `poll_next()` implementation? Trace the construction of the `JoinHashMap` and see if `tokio::spawn` is used to collect the build side concurrently.

### Task 3.5: Disk Spilling
* **Task 3.5: Proactive Disk Spilling**
  * **Target Crates/Files:** `datafusion-physical-plan` (`src/sorts/sort.rs`, `src/spill.rs`), `datafusion-execution` (`src/disk_manager.rs`)
  * **Focus:** Trace `ExternalSorter` (used by `SortExec`). Trace the exact failure path of `try_grow()`. How does the operator initiate `spill_to_disk()`? Look at how the `DiskManager` writes and later re-reads the `arrow-ipc` format.

---

## Phase 4: Communication Interfaces (Data Exchange)
**Objective:** Map out the Storage API (`TableProvider`), the internal memory exchange (`RepartitionExec`), and the distributed network protocols (`Arrow Flight` and `Protobuf`) to contrast with Trino's networking architecture.

### Task 4.1: The Storage Plane
* **Task 4.1.A: The Storage Plane — TableProvider Contract**
  * **Target Crates/Files:** `datafusion-expr` (`src/table_source.rs`), `datafusion-catalog` (`src/table_provider.rs`), `datafusion-core` (`src/datasource/`)
  * **Focus:** Trace the `TableProvider` trait. How does `scan()` receive projections, filters, and limit hints? How does `supports_filters_pushdown()` negotiate predicate handling (Exact, Inexact, Unsupported)? Trace `DataSink` for write paths (`INSERT`, `CREATE TABLE AS`). Compare `TableProvider` to Trino's `ConnectorPageSource`.
* **Task 4.1.B: The Storage Plane — Parquet Scan Pipeline**
  * **Target Crates/Files:** `datafusion-core` (`src/datasource/physical_plan/parquet/`), `parquet` crate (`src/arrow/async_reader/`), `object_store`
  * **Focus:** Trace the full Parquet scan pipeline inside `ParquetExec`. How does row-group pruning use min/max statistics? How does page index pruning work? How does the row-level `RowFilter` apply pushed-down predicates during decode? Trace `object_store` byte-range fetching and how it integrates with `tokio` for async I/O.

### Task 4.1.C: Iceberg Integration & Delete File Handling
* **Task 4.1.C: DataFusion Iceberg Integration & Delete File Handling** `[KG-ICE-4]`
  * **Target Crates/Files:** `iceberg-rust` repo — `crates/iceberg/src/scan.rs`, `crates/iceberg/src/delete_file_manager.rs` (or equivalent), `crates/datafusion/src/` (DataFusion integration module), `crates/iceberg/src/arrow/` (Arrow bridge)
  * **Focus:** Trace how the `iceberg-rust` project integrates with DataFusion:
    1. How does the Iceberg `TableProvider` implementation translate `scan()` (with projections, filters, limit) into Iceberg table scans? How does it map DataFusion's `Expr` filters to Iceberg's predicate format?
    2. **Delete file handling:** How are positional delete files applied? How are equality delete files applied? At what stage — during Parquet decode, as a post-filter on RecordBatches, or via a separate stream combinator? What is the merge-on-read (MOR) pipeline architecture?
    3. **Schema evolution:** How does the reader handle data files written under older schemas? Is column ID-based mapping used (like Trino) or name-based?
    4. **Predicate pushdown:** How do Iceberg partition predicates flow down to row-group pruning in the Parquet reader? Is there a multi-level pushdown (partition → file → row-group → page)?
    5. **Snapshot isolation:** How is a consistent snapshot selected for a scan? How does time-travel work?
    6. Compare the maturity and completeness of this implementation vs. Trino's Java Iceberg connector (Task 4.3.D).

### Task 4.2: The Local Data Plane
* **Task 4.2: The Local Data Plane (Intra-Node Exchange) — Cross-Reference**
  * **Reference:** See Phase 2, Task 2.4.B (`23_datafusion_48_2.4.B_local_repartitioning.md`)
  * **Focus:** This topic is fully covered in the Phase 2 research. That file traces `RepartitionExec::execute()`, custom Gate-based channels (`DistributionSender`/`DistributionReceiver`), hash/round-robin routing, backpressure, spill-to-disk, and preserve-order mode. No additional research needed.

### Task 4.3: The Network Data Plane
* **Task 4.3: The Network Data Plane (Arrow Flight)**
  * **Target Crates/Files:** `arrow-flight` (`src/encode.rs`, `src/decode.rs`), `datafusion-ballista` (optional, for context on `FlightClient`)
  * **Focus:** Analyze how a `RecordBatch` stream is converted into a stream of `FlightData` protobuf messages via `FlightDataEncoder`. Trace the receiving side (`FlightDataDecoder`) to confirm how the byte payloads are wrapped into `Buffer`s without copying. Contrast the Arrow Flight gRPC streaming model with Trino's custom token-based HTTP chunk pulling.

### Task 4.4: The Distributed Control Plane
* **Task 4.4: The Distributed Control Plane (Plan Serialization)**
  * **Target Crates/Files:** `datafusion-proto` (`src/physical_plan/mod.rs`, `src/physical_plan/to_proto.rs`)
  * **Focus:** How does DataFusion serialize a complex Physical Plan (e.g., a HashJoin tree) into a byte array so it can be sent to remote workers? Trace the `AsExecutionPlan` trait. Contrast this Protobuf-based physical plan transmission with Trino's REST/JSON `TaskUpdateRequest`.

---

## Phase 5: Memory Tracking & Arbitration (Memory Management)
**Objective:** Map out the RAII-based memory accounting system. Understand how memory pools arbitrate resources, how operators interact with reservations, and how spilling is triggered synchronously.

### Task 5.1: The MemoryPool
* **Task 5.1: The `MemoryPool` Trait and Implementations**
  * **Target Crates/Files:** `datafusion-execution` (`src/memory_pool/mod.rs`, `src/memory_pool/pool.rs`)
  * **Focus:** Analyze the `MemoryPool` trait. Trace the implementations of `GreedyMemoryPool` and `FairSpillPool`. How does `FairSpillPool` track per-consumer usage to ensure even distribution? Compare this to Trino's unified global pool.

### Task 5.2: The RAII Reservation
* **Task 5.2: The RAII Memory Reservation Lifecycle**
  * **Target Crates/Files:** `datafusion-execution` (`src/memory_pool/mod.rs` — focus on the `MemoryReservation` struct)
  * **Focus:** Trace the creation of a `MemoryReservation`. Look closely at its `Drop` implementation. How does it guarantee that `pool.shrink()` is called when the reservation is destroyed? Contrast this compiler-enforced safety with Trino's `free()` calls.

### Task 5.3: Memory Consumers & Spilling
* **Task 5.3: Memory Consumers & Proactive Spilling**
  * **Target Crates/Files:** `datafusion-execution` (`src/memory_pool/mod.rs` — focus on `MemoryConsumer`), `datafusion-physical-plan` (`src/sorts/sort.rs` or `src/joins/hash_join.rs`)
  * **Focus:** Trace how a specific operator (like `ExternalSorter`) registers as a `MemoryConsumer`. Trace the execution path when `reservation.try_grow()` fails. How does this directly trigger the spilling logic (mapped in Phase 3) without requiring a background revoking thread?

### Task 5.4: Task Context & Global Limits
* **Task 5.4: Task Context and Global Limits**
  * **Target Crates/Files:** `datafusion-execution` (`src/task.rs`), `datafusion-common` (`src/config.rs`)
  * **Focus:** Trace how the `MemoryPool` is instantiated and attached to the `TaskContext`. How are global memory limits defined in the `SessionConfig` and passed down to the physical execution plan?

---

## Phase 6: Test Infrastructure & Validation Patterns `[KG-13]`
**Objective:** Map DataFusion's test hierarchy and idiomatic Rust testing patterns. This serves as a reference for designing the Rust worker's test suite — particularly for unit-testing operators in isolation, property-based testing of serialization, and integration-testing via sqllogictest.

### Task 6.1: Test Hierarchy, Frameworks & CI
* **Task 6.1.A: Test Organization & Infrastructure**
    * **Target Crates/Files:** Workspace `Cargo.toml` (test configuration), `.github/` (CI workflows), `datafusion/core/tests/`, `datafusion/sqllogictest/`, per-crate `#[cfg(test)]` modules
    * **Focus:** Map the complete test taxonomy:
      1. What test categories exist? (unit tests via `#[cfg(test)]`, integration tests via `tests/` directories, sqllogictest, benchmarks)
      2. How are tests organized? (per-crate `#[cfg(test)]` modules vs. `tests/` directories vs. dedicated test crates)
      3. How is sqllogictest used? What `.slt` files exist and what do they cover? How are expected results maintained?
      4. What CI infrastructure runs these tests? (GitHub Actions workflows, test matrix, feature flags)
      5. How are test fixtures and shared utilities organized? (`datafusion/core/tests/data/`, shared test helpers)

### Task 6.2: Data Model & Serialization Tests (aligns with Impl Phase 1)
* **Task 6.2.A: Arrow Serialization & Round-Trip Tests**
    * **Target Crates/Files:** `arrow-rs` test modules (buffer, array round-trips), `datafusion-proto/src/` (protobuf serialization tests), `datafusion/core/tests/` (IPC/Parquet round-trips)
    * **Focus:** How data model correctness is validated:
      1. How are Arrow array round-trips tested? (construct → serialize → deserialize → assert equality)
      2. How does `datafusion-proto` test plan serialization/deserialization fidelity?
      3. Are there property-based or fuzz tests for serialization?
      4. What test utilities exist for constructing test `RecordBatch` instances?

### Task 6.3: Operator & Expression Tests (aligns with Impl Phases 4-5, 8)
* **Task 6.3.A: Operator & Expression Testing Patterns**
    * **Target Crates/Files:** `datafusion/physical-plan/src/` (operator `#[cfg(test)]` modules), `datafusion/physical-expr/src/` (expression tests), `datafusion/core/tests/`
    * **Focus:** How individual components are tested:
      1. How are operators tested in isolation? (synthetic `RecordBatch` input, `collect()` output, assertion patterns)
      2. How are physical expressions tested? (direct evaluation tests, edge cases, null handling)
      3. What test utility functions exist? (`assert_batches_eq`, `assert_batches_sorted_eq`, `partitions_to_sorted_vec`)
      4. How are stateful operators tested (joins, aggregations, sorts) — do tests cover spill paths?
      5. Are there parameterized tests that run operators across multiple data types?

### Task 6.4: Integration & Correctness Tests (aligns with Impl Phase 9)
* **Task 6.4.A: SQL-Level Correctness & Reference Validation**
    * **Target Crates/Files:** `datafusion/sqllogictest/`, `datafusion/core/tests/sql/`, benchmarks
    * **Focus:** How end-to-end SQL correctness is validated:
      1. How does DataFusion use sqllogictest for correctness testing against Postgres as reference?
      2. What query coverage do the `.slt` files provide? (DML, DDL, window functions, subqueries, etc.)
      3. How are TPC-H queries used for validation and benchmarking?
      4. What patterns could the Rust worker adopt for validating query-level correctness against the Java Trino coordinator?
      5. How are regression tests structured when a bug is found and fixed?
