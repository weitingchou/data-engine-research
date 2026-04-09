# Phase 3: Operator Internals & Compute Overview (DataFusion)

## Table of Contents
- [1. The Volcano Contract: `Stream` vs. `Operator`](#1-the-volcano-contract-stream-vs-operator)
  - [Three Canonical `poll_next` Patterns](#three-canonical-poll_next-patterns)
  - [Observability: `ObservedStream` / `record_poll()`](#observability-observedstream--record_poll)
- [2. Compute: Physical Expressions and Arrow Kernels](#2-compute-physical-expressions-and-arrow-kernels)
- [3. Pipeline Anatomy: Simple vs. Complex Stream Chains](#3-pipeline-anatomy-simple-vs-complex-stream-chains)
  - [Simple Pipelines (Stateless Nesting)](#simple-pipelines-stateless-nesting)
  - [Complex Pipelines (Stateful Breakers)](#complex-pipelines-stateful-breakers)
- [4. Resilience: Proactive Disk Spilling](#4-resilience-proactive-disk-spilling)
- [5. SIMD & Auto-Vectorization Strategy](#5-simd--auto-vectorization-strategy)
  - [The Auto-Vectorization Contract](#the-auto-vectorization-contract)
  - [Why Auto-Vectorization Works](#why-auto-vectorization-works)
  - [DataFusion's Own Vectorized Loops](#datafusions-own-vectorized-loops)
  - [No Explicit SIMD — By Design](#no-explicit-simd--by-design)
  - [Implications for the Rust Worker](#implications-for-the-rust-worker)
- [Summary: Connecting the Dots](#summary-connecting-the-dots)

This phase focuses on the physical data processing engine. While Trino uses a custom push-pull Operator state machine and generates JVM bytecode for compute, DataFusion implements an **asynchronous, vectorized Volcano model** and relies on pre-compiled Apache Arrow compute kernels.

## 1. The Volcano Contract: `Stream` vs. `Operator`

To compare DataFusion with Trino, you must understand how their operator contracts differ in implementing the Volcano streaming model.

* **Trino's Push-Pull Contract:** An operator is a custom state machine. The Driver manually pushes data in (`addInput()`) and pulls data out (`getOutput()`), checking `isBlocked()` to yield the thread.
* **DataFusion's Async Pull Contract:** DataFusion relies entirely on Rust's native `futures::stream::Stream` trait. The equivalent of Trino's Operator interface is **`RecordBatchStream`** — a trait that extends `Stream<Item = Result<RecordBatch>>` with a `schema()` method. `SendableRecordBatchStream = Pin<Box<dyn RecordBatchStream + Send>>` is the universal type-erased stream type.
    * **`poll_next()`:** This single method replaces Trino's entire 5-method state machine (`needsInput`, `addInput`, `getOutput`, `isBlocked`, `isFinished`). A parent operator calls `poll_next()` on its child.
    * If the child has a `RecordBatch` ready, it returns `Poll::Ready(Some(batch))`.
    * If the child is waiting on I/O or a hash build, it returns `Poll::Pending`. The `tokio` runtime automatically parks the thread.
    * **`RecordBatchStreamAdapter`:** Uses `pin_project_lite` to wrap any `Stream<Item = Result<RecordBatch>>` + `SchemaRef` into a `RecordBatchStream`. This is the primary bridge for operators that construct ad-hoc streams.

### Three Canonical `poll_next` Patterns

| Pattern | Example | Behavior |
|---|---|---|
| **Pure functor** | `ProjectionStream` | Polls input once, transforms batch, returns. No buffering. |
| **Accumulation loop** | `FilterExecStream`, `CoalesceBatchesStream` | Loops pulling batches via `ready!()` until output threshold met (e.g., `batch_size`). |
| **State-machine dispatch** | `HashJoinStream` (6 states), `GroupedHashAggregateStream` (4 states) | `loop { match self.state { ... } }` with state transitions as enum assignments. Multiple transitions per poll call. |

### Observability: `ObservedStream` / `record_poll()`

Every stream is wrapped by `BaselineMetrics::record_poll()`, which records `output_rows`, `output_bytes`, `output_batches` only on `Poll::Ready` — zero cost on `Poll::Pending`. The `Drop` impl calls `try_done()` to record end time if the stream is dropped without explicit completion.

## 2. Compute: Physical Expressions and Arrow Kernels

Once data is pulled into an operator, it must be transformed.

* **`PhysicalExpr` Trait:** The core abstraction with `evaluate(&RecordBatch) -> Result<ColumnarValue>`. `PhysicalExprRef = Arc<dyn PhysicalExpr>` enables shared ownership across partitions. Expressions compose into trees via recursive `evaluate()` calls.
* **`ColumnarValue` Dual Representation:** Either `Array(ArrayRef)` for columnar data or `Scalar(ScalarValue)` for constants. Literals stay as scalars (no materialization). Column references are zero-copy `Arc::clone`. Arrow's `Datum` trait dispatches all four combinations (Array-Array, Scalar-Array, Array-Scalar, Scalar-Scalar) natively.
* **`BinaryExpr` — The Workhorse:** Handles arithmetic, comparison, boolean logic, bitwise, and regex. Uses a three-level short-circuit strategy for AND/OR: full short-circuit (all-true/all-false), **PreSelection** (when <=20% of rows are true, filter the batch before evaluating RHS), and normal vectorized evaluation.
* **Arrow Compute Kernels:** DataFusion never implements SIMD directly. All compute delegates to Arrow kernels:
    * Comparisons: `arrow::compute::kernels::cmp::{eq, lt, gt, ...}`
    * Arithmetic: `arrow::compute::kernels::numeric::{add, sub, mul, ...}`
    * Boolean: `arrow::compute::kernels::boolean::{and_kleene, or_kleene, not}`
    * Filtering: `arrow::compute::filter_record_batch` — applies a boolean mask to each column using SIMD-accelerated bitmap operations.
* **Expression Simplification (3 levels):** NOT pushdown (De Morgan's laws), cast unwrapping (removes unnecessary casts in comparisons), and constant folding (evaluates all-literal subtrees against a zero-row dummy batch at plan time).
* **`InListExpr` Optimization:** When all list elements are literals, pre-builds a `HashMap`-backed `StaticFilter` for O(1) per-row lookup instead of O(list_size) comparisons.
* **`evaluate_selection()`:** Safety mechanism for CASE expressions — returns an empty array when no rows match the selection, preventing division-by-zero in fallible expressions.

## 3. Pipeline Anatomy: Simple vs. Complex Stream Chains

In DataFusion, a "Pipeline" is a chain of nested `Stream` objects executing in a single `tokio` task. The pipeline breaks when it hits a stateful operator or an explicit repartitioning boundary.

### Simple Pipelines (Stateless Nesting)

* **`FilterExec`:** Creates a `FilterExecStream` with a child stream, predicate, optional embedded projection, and a `LimitedBatchCoalescer` (target `batch_size=8192`). The `poll_next()` loop: (1) checks coalescer for completed batches, (2) pulls from child via `ready!()`, (3) projects columns first (zero-copy `RecordBatch::project`), (4) evaluates predicate to `BooleanArray`, (5) applies `filter_record_batch()` (Arrow vectorized kernel — no row iteration), (6) pushes into coalescer. **Project-first-then-filter** is intentional: projection is zero-copy, and filtering fewer columns is cheaper.
* **`ProjectionExec`:** A pure functor map. `poll_next()` pulls one batch, evaluates each expression, builds output `RecordBatch`. No buffering, no coalescing. `Column::evaluate()` returns `Arc::clone(batch.column(idx))` — a reference count bump, not a data copy. Adds zero latency.
* **`CoalesceBatchesExec` is deprecated (since v52).** Batch coalescing is now operator-local. `FilterExec`'s `LimitedBatchCoalescer` accumulates small filtered batches until `batch_size` rows, solving the tiny-batch problem from selective filters. Input batches > `target_batch_size / 2` are emitted directly without copying.
* **No Fused ScanFilterProject Operator.** Unlike Trino's monolithic `ScanFilterAndProjectOperator`, DataFusion achieves equivalent results through composable optimizer rules: `ProjectionPushdown` pushes column selection into the scan's `ProjectionMask`, `FilterPushdown` pushes predicates into `ParquetSource` for row-group/page/row-level pruning, and `FilterExec::EmbeddedProjection` embeds projections into the filter node.

### Complex Pipelines (Stateful Breakers)

* **`HashJoinExec`:** The build side is collected cooperatively via `OnceFut<JoinLeftData>` (wrapping `Shared<BoxFuture>`) — **no `tokio::spawn`**. Multiple output partitions share the same build computation via cloning. The `HashJoinStream` state machine: `WaitBuildSide -> FetchProbeBatch -> ProcessProbeBatch -> ExhaustedProbeSide -> Completed`. `ProcessProbeBatch` is synchronous with batch-size-limited hash lookups via `MapOffset` for resumability.
* **`JoinHashMap`:** Uses `hashbrown::HashTable<(u64, u32)>` (not `HashMap`) with a LIFO chained structure (`Vec<u32>`, 1-indexed, 0 = end sentinel). Batches inserted in **reverse order** to preserve input ordering during probe traversal. Join type affects three code paths: bitmap tracking, index adjustment (`adjust_indices_by_join_type`), and final unmatched-row emission.
* **`AggregateExec` / `GroupedHashAggregateStream`:** Separates group-key interning (`GroupValues` with vectorized `intern()`) from aggregate state (`GroupsAccumulator`). `GroupValues` only assigns group indices; accumulators only store per-group state indexed by those indices. The `SkipAggregationProbe` monitors group/row ratio — when partial aggregation is ineffective (high cardinality), it bypasses the hash table and converts rows directly to intermediate states via `convert_to_state()`. Also uses `hashbrown::HashTable` for low-level control with precomputed hashes.

## 4. Resilience: Proactive Disk Spilling

Trino's spilling mechanism is *push-based*: a global `MemoryRevokingScheduler` forces operators to spill. DataFusion's spilling mechanism is **pull-based and proactive**.

* **The Trigger:** `ExternalSorter::insert_batch()` calls `reservation.try_grow(2 * batch_size)`. On `Err(ResourcesExhausted)`, it calls `sort_and_spill_in_mem_batches()` — sorts all in-memory data, incrementally appends to an `InProgressSpillFile`, frees the reservation, then retries `try_grow`. The `FairSpillPool` divides memory equally among spillable consumers.
* **`SpillManager` Stack:** `SpillManager` creates temp files via `DiskManager`, tracks metrics, and reads back spill files. `InProgressSpillFile` wraps a lazily-initialized `IPCStreamWriter` (only created on first `append_batch()` — empty partitions produce no file). Writer uses Arrow IPC **Stream** format with optional LZ4/Zstd compression.
* **Read-Back:** `SpillReaderStream` reads one batch per `spawn_blocking` task (not one task per file, to prevent Tokio deadlocks). Validation is skipped since DataFusion wrote the data.
* **`DiskManager` & `RefCountedTempFile`:** Lazily creates temp directories, tracks global disk usage via `AtomicU64`. `RefCountedTempFile` uses `Arc`-based reference counting to subtract usage only when the last clone drops. Every `append_batch()` call triggers `update_disk_usage()` with a check against `max_temp_directory_size` (100GB default).
* **Multi-Level Merge:** `MultiLevelMergeBuilder` handles cases with too many spill files for a single merge pass — runs iterative merge-and-re-spill loops until all data can be combined. Each spill file tracks `max_record_batch_memory` for merge memory budgeting.
* **Cross-Operator Spilling:** The `SpillManager` infrastructure is shared across operators:

| Operator | Spill Behavior |
|---|---|
| **ExternalSorter** (SortExec) | Sorts in-memory batches, writes sorted run, merges during output |
| **GroupedHashAggregateStream** | Emits all groups as intermediate states, sorts by group keys, writes to spill file. Re-merges and re-aggregates on read-back. |
| **RepartitionExec** | Uses SpillManager for partition buffering when downstream is slow |
| **SortMergeJoin** | Uses SpillManager + SpillPool for materializing join sides |

* **Key Config Options:** `sort_spill_reservation_bytes` (10MB pre-reserved merge headroom), `sort_in_place_threshold_bytes` (1MB), `max_spill_file_size_bytes` (128MB for SpillPoolWriter rotation), `max_temp_directory_size` (100GB global disk limit).

## 5. SIMD & Auto-Vectorization Strategy

DataFusion achieves SIMD performance through **pure LLVM auto-vectorization** — zero explicit SIMD intrinsics exist anywhere in DataFusion or arrow-rs (v58.1.0). This is not an omission but a deliberate design strategy.

### The Auto-Vectorization Contract

The entire stack is engineered so LLVM's auto-vectorizer can emit SIMD instructions:

* **Arrow's 64-byte memory alignment** guarantees all `Buffer` allocations are cache-line-aligned, exceeding AVX-512 (64B), AVX2 (32B), and NEON (16B) requirements. LLVM can emit aligned loads (`vmovdqa`) without special alignment logic.
* **Compile-time configuration:** `RUSTFLAGS='-C target-cpu=native'` enables the full instruction set of the build machine. Without it, only SSE2 (128-bit) is used on x86_64. With it on AVX2 hardware, 256-bit registers process 8× i32 per instruction; on AVX-512, 512-bit registers process 16× i32.
* **LTO is critical:** The release profile uses `lto = true`, `codegen-units = 1`. Without LTO, Arrow kernel inner loops cannot be inlined across the crate boundary into DataFusion's expression evaluation — preventing loop fusion, constant propagation, and cross-kernel optimization.

### Why Auto-Vectorization Works

Arrow compute kernels are ideal auto-vectorization targets because:
1. Inner loops iterate over contiguous `&[T]` slices — no scatter/gather patterns.
2. Loop bodies are simple arithmetic or comparison operations.
3. Null handling uses bitmask operations (`u64` AND/OR), not per-element branching.
4. The `Datum` trait dispatches scalar-vs-array at the call site, not inside the loop.

### DataFusion's Own Vectorized Loops

Where DataFusion writes its own inner loops (aggregation, group-by), it uses explicit auto-vectorization-friendly patterns:

* **`unsafe { get_unchecked_mut }` in `PrimitiveGroupsAccumulator`:** Eliminates bounds checks in the accumulation inner loop, allowing LLVM to vectorize the `value_fn(group_index, new_value)` closure.
* **64-bit chunk null processing:** `NullState::accumulate()` processes null bitmaps in `u64` chunks via `bit_chunks()`, checking 64 validity bits per word with `mask & index_mask`.
* **No-null fast paths everywhere:** `null_count() == 0` checks gate dedicated zero-overhead paths that skip bitmap processing entirely. The `const NULLABLE: bool` generic parameter enables compile-time elimination of null handling code.
* **`BooleanBuffer::collect_bool`:** The workhorse for boolean construction — used in comparisons, IN-list, join hash maps. LLVM can vectorize the inner closure to compare 32 elements at once (AVX-512) and pack results into bits.

### No Explicit SIMD — By Design

A thorough search confirms zero uses of `std::arch`, `std::simd`, `packed_simd`, `#[target_feature]`, or any platform-specific intrinsic. Arrow-rs historically experimented with `packed_simd2` (the `simd` feature) but removed it in favor of auto-vectorization. The rationale: auto-vectorization is portable across architectures (x86, ARM, RISC-V) and benefits automatically from LLVM improvements, while explicit SIMD creates a maintenance burden and architecture-specific code paths.

### Implications for the Rust Worker

DataFusion's strategy is directly adoptable: use Arrow arrays as the internal data model, compile with `-C target-cpu=native` and LTO, and let LLVM handle vectorization for straightforward compute. For the critical paths where auto-vectorization is unreliable (hash table probing, null-aware compress/expand, Parquet bit-unpacking), supplement with explicit SIMD from `std::arch` — following Velox's approach for those specific hot paths.

## Summary: Connecting the Dots

1. The **`RecordBatchStream`** trait (extending `futures::Stream`) is the universal operator contract. `poll_next()` replaces Trino's 5-method `Operator` interface.
2. **`PhysicalExpr::evaluate()`** delegates all computation to Arrow kernels — DataFusion never implements SIMD directly. `BinaryExpr` uses three-level short-circuiting for boolean expressions.
3. **Simple pipelines** (`ProjectionStream(FilterStream(ScanStream))`) compose as nested structs with zero abstraction overhead. Projection is zero-copy (`Arc::clone`). Filter coalesces small batches inline.
4. **Complex pipelines** break the stream at stateful operators. Hash joins use cooperative `OnceFut` (no `tokio::spawn`), LIFO-chained `JoinHashMap` with `hashbrown::HashTable`, and a 5-state machine. Aggregation separates group interning from accumulation with a skip-aggregation probe for high cardinality.
5. **Disk spilling** is pull-based and proactive: operators detect memory pressure via `try_grow()` failure and voluntarily spill via `SpillManager` using Arrow IPC format. A shared infrastructure (`SpillManager` + `InProgressSpillFile` + `DiskManager`) serves Sort, Aggregate, Repartition, and SortMergeJoin.
6. **SIMD is achieved through pure auto-vectorization** — no explicit intrinsics. Arrow's 64-byte alignment, LTO, and `-C target-cpu=native` combine to let LLVM emit AVX2/AVX-512 instructions for compute kernels. DataFusion's own aggregation loops use `unsafe` unchecked access and 64-bit chunk null processing to stay on the auto-vectorization fast path.
