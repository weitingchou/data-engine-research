# Phase 3: Operator Internals & Compute Overview (DataFusion)

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

## Summary: Connecting the Dots

1. The **`RecordBatchStream`** trait (extending `futures::Stream`) is the universal operator contract. `poll_next()` replaces Trino's 5-method `Operator` interface.
2. **`PhysicalExpr::evaluate()`** delegates all computation to Arrow kernels — DataFusion never implements SIMD directly. `BinaryExpr` uses three-level short-circuiting for boolean expressions.
3. **Simple pipelines** (`ProjectionStream(FilterStream(ScanStream))`) compose as nested structs with zero abstraction overhead. Projection is zero-copy (`Arc::clone`). Filter coalesces small batches inline.
4. **Complex pipelines** break the stream at stateful operators. Hash joins use cooperative `OnceFut` (no `tokio::spawn`), LIFO-chained `JoinHashMap` with `hashbrown::HashTable`, and a 5-state machine. Aggregation separates group interning from accumulation with a skip-aggregation probe for high cardinality.
5. **Disk spilling** is pull-based and proactive: operators detect memory pressure via `try_grow()` failure and voluntarily spill via `SpillManager` using Arrow IPC format. A shared infrastructure (`SpillManager` + `InProgressSpillFile` + `DiskManager`) serves Sort, Aggregate, Repartition, and SortMergeJoin.
