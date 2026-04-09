# Phase 1: Foundation — Data Layout & The Ingestion Bridge (DataFusion)

## Table of Contents
- [1. The Data Hierarchy: Buffer -> Array -> RecordBatch](#1-the-data-hierarchy-buffer---array---recordbatch)
  - [`Buffer` — The Byte Substrate (Trino's `Slice`)](#buffer--the-byte-substrate-trinos-slice)
  - [`Array` — The Columnar Accessor (Trino's `Block`)](#array--the-columnar-accessor-trinos-block)
  - [`RecordBatch` — The Passive Envelope (Trino's `Page`)](#recordbatch--the-passive-envelope-trinos-page)
- [2. Separation of Data and Compute](#2-separation-of-data-and-compute)
- [3. From S3 to Memory: The Async Ingestion Pipeline](#3-from-s3-to-memory-the-async-ingestion-pipeline)
- [4. Memory Tracking (RAII and Reservations)](#4-memory-tracking-raii-and-reservations)

This phase maps DataFusion's in-memory data representation from the lowest byte-level abstraction up to the `RecordBatch` that flows through the execution engine. Because DataFusion is built natively on Apache Arrow, its memory model is governed by the Arrow specification — a fundamentally different foundation from Trino's JVM-based Slice/Block/Page hierarchy.

## 1. The Data Hierarchy: Buffer -> Array -> RecordBatch

### `Buffer` — The Byte Substrate (Trino's `Slice`)

`Buffer` (from `arrow-buffer`) is a bounded, immutable, reference-counted view over a contiguous memory region. It has three fields: `Arc<Bytes> data`, `*const u8 ptr`, and `usize length`.

Key facts from the source:

- **Two-tier ownership model.** `Bytes` owns the raw memory (`NonNull<u8>` pointer + length + `Deallocation` strategy). `Arc<Bytes>` enables shared ownership via atomic reference counting. `Buffer` adds a sliding window (ptr + length) over the shared `Bytes`.
- **Dual deallocation strategy.** The `Deallocation` enum has two variants: `Standard(Layout)` for memory allocated by Rust's global allocator, and `Custom(Arc<dyn Allocation>, usize)` for externally-owned memory (FFI buffers, `bytes::Bytes` from network I/O). The `Custom` variant uses a type-erased, reference-counted handle — the external allocation is freed when the last Arrow buffer referencing it is dropped.
- **Raw pointer for LLVM vectorization.** `Buffer` stores a `*const u8` pointer rather than a `usize` offset. The source comments explain: storing an offset would require pointer arithmetic (`base + offset`) at every access, which causes LLVM's auto-vectorizer to fail on certain patterns.
- **Zero-copy slicing:** `Buffer::slice(offset)` clones the `Arc<Bytes>` (refcount increment), then advances the raw pointer and shrinks the length. No data is copied. Multiple Buffers can view different sub-regions of the same allocation.
- **Architecture-specific alignment:** `MutableBuffer::with_capacity()` allocates with cache-aware alignment — 64 bytes on ARM/x86, 128 bytes on x86_64 (matching Intel's L2D streamer block size). Sizes are additionally rounded to 64-byte multiples. Alignment is NOT preserved after slicing.
- **Zero-copy network adoption:** `From<bytes::Bytes>` wraps network I/O bytes directly into a `Buffer` via `Deallocation::Custom`, enabling zero-copy from S3 reads through to columnar compute.

### `Array` — The Columnar Accessor (Trino's `Block`)

`Array` is an `unsafe trait` (implementations must guarantee Arrow spec compliance) representing a logical columnar array. Unlike Trino's sealed `Block` with 11 `ValueBlock` implementations, Arrow composes arrays from multiple `Buffer`s.

**`PrimitiveArray<T>`** (e.g., `Int32Array`, `Float64Array`) has three fields: `DataType`, `ScalarBuffer<T::Native>` (values), `Option<NullBuffer>` (nulls). Values at null positions are arbitrary (not zeroed), enabling SIMD-friendly processing without null-masking during writes.

**`StringArray`** (alias for `GenericByteArray<GenericStringType<i32>>`) has four fields:
1. `OffsetBuffer<i32>` — N+1 monotonically increasing offsets. Element `i` spans `offsets[i]..offsets[i+1]`.
2. `Buffer` — concatenated UTF-8 string bytes, no separators.
3. `Option<NullBuffer>` — bit-packed validity bitmap.
4. `DataType` — `DataType::Utf8`.

For `["hello", "", "world"]`: offsets = `[0, 5, 5, 10]`, values = `b"helloworld"`. Value access is two pointer arithmetic ops into the shared buffer — no allocation.

**Null Representation: Bit-Packed Bitmap (vs. Trino's `boolean[]`)**

Arrow uses `NullBuffer` wrapping `BooleanBuffer` wrapping `Buffer`. Values are stored as little-endian packed bits: 1 bit per value, 8 values per byte. The `is_null(i)` call chain bottoms out at: `(*ptr.add(i / 8) & (1 << (i % 8))) != 0` — two integer ops.

| Rows | Arrow NullBuffer | Trino boolean[] | Savings |
|---|---|---|---|
| 1,000,000 | ~122 KB | ~977 KB | 8x |
| 100 columns × 1M rows | ~12 MB | ~93 MB | ~81 MB saved |

When a column has no nulls, the null buffer is `None` — no bitmap allocated. Trino uses `@Nullable boolean[] valueIsNull` with `null` indicating no nulls — same optimization, different mechanism.

**Zero-copy slicing:** `Array::slice(offset, length)` adjusts internal offsets and clones `Arc`-backed buffers. `StringArray::slice()` shares the full values buffer — only the offsets window shrinks. Like Trino's `Block.getRegion()`, sliced views retain the original backing allocation.

### `RecordBatch` — The Passive Envelope (Trino's `Page`)

`RecordBatch` is a structurally minimal container: `SchemaRef` (= `Arc<Schema>`) + `Vec<Arc<dyn Array>>` + `row_count: usize`. Its ~40-byte stack overhead plus 16 bytes per column (Arc fat pointer) is negligible.

Key facts:
- **Self-describing.** Unlike Trino's `Page` (which has no schema — type correctness is enforced by pipeline construction), `RecordBatch` carries `Arc<Schema>`, enabling runtime type validation at batch boundaries via `try_new()`.
- **`project(indices)` is confirmed zero-copy.** Clones K `Arc<dyn Array>` column pointers (atomic refcount bump each) and K `Arc<Field>` references from the schema. Uses `unsafe { new_unchecked() }` to skip redundant validation. O(K), independent of row count.
- **`slice(offset, length)`** delegates to `Array::slice()` on each column. Each column adjusts buffer pointers without copying data.
- **`Fields = Arc<[FieldRef]>` is doubly reference-counted.** Cloning `Fields` increments the outer Arc (one atomic op for all fields). Projecting individual fields clones inner `FieldRef = Arc<Field>` entries.
- **`row_count` stored separately** for the zero-column edge case (e.g., `SELECT count(*) FROM ...` intermediate results).

## 2. Separation of Data and Compute

Like Trino, DataFusion strictly separates data from compute. `RecordBatch` has no `.filter()` or `.join()` methods — compute is applied externally by physical execution plans that consume and produce `RecordBatch` streams through the async pull-based protocol (Phase 2). This means the entire data layer can be understood independently of the execution engine.

## 3. From S3 to Memory: The Async Ingestion Pipeline

Unlike Trino's synchronous, single-threaded-per-split I/O, DataFusion is built on `tokio` for asynchronous I/O. The ingestion path transforms remote bytes into `RecordBatch`es through a multi-stage state machine:

```
ObjectStore (S3) → Parquet Footer → Pruning Cascade → Push Decoder → Arrow Arrays → RecordBatch
```

| Stage | Component | I/O? | Description |
|---|---|---|---|
| 1. File assignment | `DataSourceExec::execute(partition)` | No | Resolves ObjectStore, creates FileStream + ParquetOpener |
| 2. Metadata loading | `ParquetOpenFuture::LoadMetadata` | Yes | Reads Parquet footer via `ObjectStore::get_range()` |
| 3. Pruning cascade | `ParquetOpenFuture` states | Mixed | File → row group → page → row (see below) |
| 4. Byte range fetch | `ObjectStore::get_ranges()` | Yes | Fetches column chunks for surviving row groups |
| 5. Decode | `ParquetPushDecoder` | No | Decodes Parquet encoding → Arrow Buffers → RecordBatch |

Key design choices:
- **No `tokio::spawn` in the read path.** All work within a single partition is driven by a single Tokio task. There are zero occurrences of `tokio::spawn` or `SpawnedTask` in the `datasource-parquet` read path. Parallelism comes from the scheduler running multiple partitions concurrently.
- **Hand-written `Future::poll` state machine.** `ParquetOpenFuture` uses an explicit 11-state enum rather than `async fn`. CPU-only states (pruning, filter preparation) execute in a tight loop without yielding; I/O states yield `Poll::Pending` to Tokio. This avoids unnecessary context switches between CPU-only steps.
- **Push-based decoder separates I/O from compute.** `ParquetPushDecoder::try_decode()` returns `NeedsData(byte_ranges)`, `Data(batch)`, or `Finished`. The caller fetches bytes from any source and pushes them in. The decoder never does I/O itself — a clean separation allowing the same decoder to work with any backend.
- **Four-level predicate pruning cascade:**
  1. **File-level:** `FilePruner::should_prune()` — skips entire files using file statistics + partition values.
  2. **Row group-level:** `PruningPredicate` with min/max statistics + bloom filters (loaded lazily).
  3. **Page-level:** `PagePruningAccessPlanFilter` using page index min/max → generates `RowSelection`.
  4. **Row-level:** `RowFilter` with `ArrowPredicate`s applied during decoding. Late materialization: filter columns decoded first, predicate evaluated, remaining columns decoded only for matching rows.
- **Zero-copy from network to compute.** S3 bytes arrive as `bytes::Bytes` via `object_store`. The Parquet decoder maps them directly into Arrow `Buffer`s via `From<bytes::Bytes>` using `Deallocation::Custom` — no intermediate copy.
- **`CooperativeStream` prevents CPU starvation.** After each decoded batch, `tokio::task::consume_budget().await` yields control if the Tokio task has consumed its cooperative scheduling budget. This is DataFusion's analog to Trino's 1-second time quantum.

## 4. Memory Tracking (RAII and Reservations)

DataFusion handles memory tracking via Rust's RAII pattern, which is fundamentally different from Trino's GC-based `getRetainedSizeInBytes()` accounting.

**The Core Abstraction: `MemoryReservation`**

Two fields: `Arc<SharedRegistration>` (link to pool + consumer identity) and `AtomicUsize` (bytes currently reserved). The operator calls `try_grow(bytes)` before allocating. If the pool approves, the local counter increments. If the pool rejects, the operator must spill or fail — `Err(ResourcesExhausted)` is returned immediately (no blocking future like Trino).

The `Drop` implementation is the keystone: when a `MemoryReservation` goes out of scope — normal completion, early return, panic unwind, or cancellation — the compiler guarantees `drop()` runs, which atomically returns all reserved bytes to the pool. Zero gap between logical release and pool update.

**Pool Implementations:**

| Pool | Locking | Enforcement | Use Case |
|---|---|---|---|
| `UnboundedMemoryPool` | `AtomicUsize` | None | Development/testing |
| `GreedyMemoryPool` | `AtomicUsize` (lock-free CAS) | Hard cap, first-come-first-served | Simple production |
| `FairSpillPool` | `Mutex` | Fair share: `(pool - unspillable) / num_spill` per consumer | Production with spilling |
| `TrackConsumersPool<I>` | Wraps any pool | Delegates to inner | Debugging — rich OOM error messages |

**Flat Tracking Model (vs. Trino's Hierarchy):**

Trino builds a deep tracking tree mirroring execution (Pool → Query → Task → Pipeline → Driver → Operator). DataFusion's model is flat: each operator's `MemoryReservation` talks directly to the global pool. There is no per-query memory limit, no per-task aggregation. The tradeoff: simpler implementation but no query-level isolation.

**Key Differences from Trino:**

| Dimension | Trino | DataFusion |
|---|---|---|
| Accounting model | Pull: `getRetainedSizeInBytes()` queries structures | Push: `try_grow()` declares intent before allocation |
| Under-counting risk | Operator can allocate without telling pool | Convention-based, but visible in code review |
| On pool exhaustion | Returns `ListenableFuture` — driver yields | Returns `Err` immediately — operator must spill or fail |
| Deallocation timing | GC-dependent: non-deterministic delay | `Drop`: synchronous, deterministic, zero delay |
| Sub-view accounting | `retainedSize` always includes full backing array | `MemoryReservation` tracks declared bytes, not backing array size |
