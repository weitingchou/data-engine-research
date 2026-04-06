# Phase 3: Operator Internals & Data Processing — Physical Plan Execution (Velox)

Velox executes queries through a Volcano-style pull-based operator pipeline where each `Driver` thread walks its operator chain from sink to source, calling `isBlocked()` / `needsInput()` / `getOutput()` / `addInput()` in a tight cooperative loop. What distinguishes Velox from traditional interpreters is that every operator processes entire columnar batches at once, gated by a `SelectivityVector` bitmask — no row-by-row iteration in the hot path. Expression evaluation is interpreted (no JIT) but compensates through compile-time template specialization, encoding peeling, and adaptive conjunct reordering. Stateful operators like hash join use split pipelines with barrier-based coordination and SIMD-accelerated probing. When memory pressure hits, operators spill to disk using the Presto wire-protocol serializer, triggered proactively through the memory reservation system rather than reactively on OOM.

## 1. The Operator Contract: A Volcano-Style State Machine

### Five Methods, Near-Identical to Trino

| Velox (C++) | Trino (Java) | DataFusion (Rust) |
|---|---|---|
| `needsInput() -> bool` | `needsInput() -> boolean` | N/A (stream-based) |
| `addInput(RowVectorPtr)` | `addInput(Page)` | N/A |
| `getOutput() -> RowVectorPtr` | `getOutput() -> Page` | `Stream::poll_next()` |
| `isBlocked(ContinueFuture*) -> BlockingReason` | `isBlocked() -> ListenableFuture<?>` | `Poll::Pending` + `Waker` |
| `isFinished() -> bool` | `isFinished() -> boolean` | `Poll::Ready(None)` |
| `noMoreInput()` | `finish()` | N/A (stream exhaustion) |

Key design facts:

- **Blocking uses `folly::SemiFuture<folly::Unit>`** (aliased as `ContinueFuture`) passed as an out-parameter. When an operator blocks, the Driver captures the future in a `BlockingState` and goes off-thread. The future's fulfillment re-enqueues the driver on the thread pool. 13 distinct `BlockingReason` variants provide fine-grained visibility.
- **`isBlocked()` is a control point, not just a query.** Complex operators perform side effects here — `HashProbe::isBlocked()` initiates `asyncWaitForHashTable()`, `Exchange::isBlocked()` fetches data, `HashBuild::isBlocked()` triggers post-build processing.
- **`SourceOperator`** stubs out the input methods (`needsInput()` returns false, `addInput()` / `noMoreInput()` throw). Sources generate data internally via `getOutput()`.
- **The `CALL_OPERATOR` macro** wraps every operator method invocation with `NonReclaimableSectionGuard` (prevents memory reclaim during execution), stats tracking, `OpCallStatus` for deadlock detection, and exception context.
- **No operator-level synchronization needed** — each Driver runs on exactly one thread at a time, so operators are always called single-threaded.

### Operator State: Implicit in Base, Explicit in Complex Operators

The base `Operator` class tracks state implicitly through `input_`, `noMoreInput_`, and `initialized_`. Complex operators define their own state enums:

- **HashBuild:** `kRunning → kWaitForBuild → kWaitForProbe → kFinish` (with spill cycles)
- **HashProbe:** `kWaitForBuild → kRunning → kWaitForPeers → kFinish` (with spill restore)

## 2. Vectorized Expression Evaluation: No JIT, No Row-at-a-Time

### Compilation Pipeline

`TypedExpr` trees from the planner are compiled into `Expr` trees by `ExprCompiler`:

1. **CSE deduplication** via `Scope::visited` map — repeated subexpressions share a single `Expr` node marked `isMultiplyReferenced_`.
2. **Function resolution** tries three registries: special forms (AND, OR, IF, TRY, CAST) → vector function registry → simple function registry (row-level UDFs wrapped via `SimpleFunctionAdapter`).
3. **Metadata computation** (`computeMetadata()`) determines `propagatesNulls_`, `deterministic_`, `distinctFields_`, and `hasConditionals_` — these drive the evaluation fast paths.

### Multi-Level Evaluation Dispatch

`Expr::eval()` follows a cascading fast-path chain:

```
eval()
  → [flat-no-nulls?] → evalFlatNoNulls() → applyFunction() → DONE
  → evalEncodings()
      → [can peel dictionary?] → peelEncodings() → evalWithNulls(newRows) → wrap() → DONE
      → evalWithNulls()
          → [propagatesNulls && mayHaveNulls?] → removeSureNulls() → evalAll(nonNullRows) → addNulls() → DONE
          → evalAll()
              → evalArgsDefaultNulls (progressively narrows MutableRemainingRows)
              → applyFunction(remainingRows) → DONE
```

- **Flat-no-nulls fast path:** All inputs flat/constant with no nulls → skips encoding peeling, null checking, error handling entirely.
- **Encoding peeling:** Strips shared dictionary layers from inputs, evaluates on distinct base values only, re-wraps. A dictionary with 10,000 rows but 100 distinct values = 100x computation reduction. `evalWithMemo()` extends this across batches.
- **Two-level null pruning:** First level removes null rows before evaluating children (`removeSureNulls`). Second level prunes after each child evaluates (`evalArgsDefaultNulls`).
- **`MutableRemainingRows`** lazily copies the `SelectivityVector` only on first mutation — zero allocation on the common no-null path.

### The Vectorized Inner Loop

`SimpleFunctionAdapter::iterate()` generates compile-time-specialized loops:

```cpp
SelectivityVector::applyToSelected(func)
  → if isAllSelected(): tight for-loop (compiler can auto-vectorize)
  → else: bits::forEachSetBit() using __builtin_ctzll for sparse iteration
```

The `isAllSelected()` fast path is the innermost hot path for all expression evaluation — a simple sequential `for (row = begin; row < end; ++row)` loop that the compiler can vectorize and unroll.

### AND/OR Short-Circuit (ConjunctExpr)

Processes 64 rows at a time using bitwise operations (`updateAnd`/`updateOr` on raw `uint64_t` words). Progressively narrows `activeRows` — once all rows are decided, remaining conjuncts are skipped entirely. Includes **adaptive reordering** based on runtime selectivity statistics (`timeToDropValue()`), placing cheapest and most selective filters first.

## 3. FilterProject: Stateless Vectorized Operator

### Filter-Project Fusion

The local planner detects consecutive `FilterNode → ProjectNode` and fuses them into a single `FilterProject` operator, eliminating intermediate materialization. The filter and projections share a single `ExprSet`, enabling shared subexpression optimization across both.

### Processing Flow

1. **`addInput()`**: Trivially stores the input batch (`input_ = std::move(input)`).
2. **`getOutput()`**:
   - Creates `SelectivityVector` with all rows selected, `EvalCtx` binding expressions to input.
   - Evaluates filter expression → produces boolean vector.
   - `processFilterResults()` converts to bitmask + index array using 64-bit-at-a-time bitwise operations.
   - Narrows `SelectivityVector` to passing rows.
   - Evaluates projection expressions **only on surviving rows**.
   - `fillOutput()` assembles result: identity columns dictionary-wrapped (zero-copy), expression results placed directly.

### Key Optimizations

- **Identity projection detection:** `checkAddIdentityProjection()` recognizes simple field references — they become pointer copies, never re-evaluated.
- **Dictionary wrapping over physical compaction:** Filtered identity columns are wrapped in `DictionaryVector` with the surviving indices as the index buffer. No data bytes are copied. Contrast: DataFusion's `filter_record_batch()` physically compacts.
- **Zero-copy passthrough:** When all rows pass and projection is identity, `fillOutput()` returns `std::move(input_)` directly.

## 4. Hash Join: Split Pipeline with Barrier Coordination

### Build Side (`HashBuild`)

Each build-side Driver has its own independent `HashTable` with its own `RowContainer`. During `addInput()`:
1. Keys are decoded through `VectorHasher`s, null keys filtered.
2. Rows stored into `RowContainer` (pre-allocated 64KB page chunks, variable-length data via `HashStringAllocator`).
3. Key analysis determines hash table mode: `kArray` (small cardinality, direct indexing), `kNormalizedKey` (packed 64-bit key), or `kHash` (generic tag-based).

At `noMoreInput()`, a barrier (`Task::allPeersFinished`) elects the last driver to merge all independent tables into a unified hash index via `prepareJoinTable()`. The merged table is handed to the probe side through `HashJoinBridge::setHashTable()`.

### Probe Side (`HashProbe`)

Starts in `kWaitForBuild` state. Once the table arrives via `HashJoinBridge::tableOrFuture()`:
1. Decodes probe keys, calls `table_->joinProbe()`.
2. SIMD-accelerated tag probing (Swiss-table-inspired, 16 slots checked per SIMD comparison).
3. 4-way interleaved probing for instruction-level parallelism — hides cache-miss latency.
4. Streams results through `listJoinResults()` across multiple `getOutput()` calls via `JoinResultIterator`.
5. Duplicate build-side matches linked via embedded next-pointers in RowContainer rows.

### Dynamic Filter Pushdown

After receiving the hash table, `HashProbe` derives range/bloom filters from `VectorHasher` statistics and pushes them to upstream scans via `driver->pushdownFilters()`. For single unique-key joins with no dependents, the join can be replaced entirely by the dynamic filter.

## 5. Disk Spilling: Proactive, Reservation-Based

### Trigger Mechanism

Spilling is triggered **proactively via memory reservation**, not reactively on OOM:

1. **Operator-initiated:** `HashBuild::ensureInputFits()` calls `pool()->maybeReserve()`. The reservation flows to the `SharedArbitrator`, which may reclaim memory from the requesting operator itself (via `ReclaimableSectionGuard` temporarily marking the operator reclaimable).
2. **Arbitrator-initiated:** The `SharedArbitrator` detects system-wide pressure and calls `Operator::reclaim()` on target operator pools.

### Spill Execution Pipeline

```
SpillerBase::spill()
  → fillSpillRuns() (iterate RowContainer, hash, assign to per-partition SpillRun)
  → runSpill() (parallel write tasks per partition)
    → writeSpill() (extract 64-row / 256KB micro-batches)
      → SpillState::appendToPartition()
        → SpillWriter::write() (PrestoVectorSerde serialization)
          → SerializedPageFile::write() (disk I/O)
```

### Key Design Choices

- **Presto wire-protocol serializer** reused for spilling — automatic type support, optional compression (LZ4/ZSTD), battle-tested correctness.
- **Dedicated spill memory pool** (`memory::spillMemoryPool()`) isolated from operator pools — breaks the circular dependency of needing memory to free memory.
- **Peer operators spilled atomically** — all parallel `HashBuild` operators checked for reclaimable state before any are spilled, preventing inconsistent hash table state.
- **Recursive spilling** up to 4 levels via hierarchical `SpillPartitionId` encoding (3 bits per level, 4 levels max).
- **Async spill tasks** propagate `MemoryArbitrationContext` via `createAsyncMemoryReclaimTask()` to prevent recursive arbitration on background threads.

## 6. Comparison with Trino and DataFusion

### Expression Evaluation

| Dimension | Velox | Trino | DataFusion |
|-----------|-------|-------|------------|
| Compilation | `TypedExpr` → `Expr` tree (interpreter) | `TypedExpr` → bytecode (Airlift JIT) | `PhysicalExpr` trait objects |
| Null handling | `SelectivityVector` bitmask pruning before function call | Per-row null check in generated code | Arrow validity bitmaps |
| Vectorization | `applyToSelected()` auto-vectorizable loop | JIT-compiled loops | Arrow compute kernels (explicit SIMD) |
| Short-circuit | 64-row bitwise AND/OR with adaptive reorder | Row-level short-circuit | Arrow `BooleanArray` compute |
| Dictionary optimization | Encoding peeling + cross-batch memoization | None (always decode) | None |

### Hash Join

| Dimension | Velox | Trino | DataFusion |
|-----------|-------|-------|------------|
| Build parallelism | Per-driver independent tables, barrier merge | Shared page accumulation, single-threaded lookup build | Single-threaded per partition |
| Hash table | 3-mode (array/normalized/hash), SIMD tag probing | Open-addressing linear probe (`PagesHash`) | `hashbrown::raw` (Rust Swiss table) |
| Probe optimization | 4-way interleaved SIMD probing | Single-row probe | Single-row probe |
| Dynamic filters | Range + bloom filter pushdown post-build | `DynamicFilter` framework | Limited support |
| Bridge mechanism | `HashJoinBridge` with promise/future | `LookupSourceProvider` with `ListenableFuture` | Tokio channel + `SharedHashJoinState` |

### Spilling

| Dimension | Velox | Trino | DataFusion |
|-----------|-------|-------|------------|
| Trigger | Proactive reservation failure (`maybeReserve`) | Revocable memory + `MemoryRevokingScheduler` | Self-spill when exceeding reservation |
| Arbitration | Centralized `SharedArbitrator`, cross-query reclaim | Per-query revocation | No centralized arbitrator |
| Serialization | Presto wire-protocol (`PrestoVectorSerde`) | Custom page serialization | Arrow IPC format |
| Coordination | Atomic peer-operator spilling (all or none) | Per-operator independent | Per-operator independent |
| Recursive spill | Up to 4 levels via `SpillPartitionId` hierarchy | Recursive re-partition | Not supported |
| Spill memory | Dedicated `spillMemoryPool()` (not arbitrated) | Revocable memory pool | Standard pool |
