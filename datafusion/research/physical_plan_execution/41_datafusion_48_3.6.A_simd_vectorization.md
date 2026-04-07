# Module Teardown: Arrow Compute Kernel SIMD Architecture (Task 3.6.A) [KG-14]

## Table of Contents

- [0. Research Focus](#0-research-focus)
- [1. High-Level Overview](#1-high-level-overview)
- [2. Detailed Analysis](#2-detailed-analysis)
  - [2.1 Arrow Memory Alignment and SIMD Enablement](#21-arrow-memory-alignment-and-simd-enablement)
  - [2.2 Explicit SIMD vs LLVM Auto-Vectorization](#22-explicit-simd-vs-llvm-auto-vectorization)
  - [2.3 Expression Evaluation to Arrow Kernel Call Path](#23-expression-evaluation-to-arrow-kernel-call-path)
  - [2.4 Null Bitmap Interaction with SIMD](#24-null-bitmap-interaction-with-simd)
  - [2.5 BooleanBuffer and Filter Mask Construction](#25-booleanbuffer-and-filter-mask-construction)
  - [2.6 LTO, Inlining, and Cross-Crate Optimization](#26-lto-inlining-and-cross-crate-optimization)
  - [2.7 GroupsAccumulator: DataFusion's Own Vectorized Aggregation](#27-groupsaccumulator-datafusions-own-vectorized-aggregation)
  - [2.8 Vectorized Group-By with Auto-Vectorization Hints](#28-vectorized-group-by-with-auto-vectorization-hints)
- [3. Key Design Insights](#3-key-design-insights)
- [4. Porting Considerations (DataFusion Patterns -> Trino Rust Worker)](#4-porting-considerations-datafusion-patterns---trino-rust-worker)

## 0. Research Focus

* **Task ID:** 3.6.A
* **Focus:** Analyze how DataFusion and Arrow-rs achieve SIMD acceleration for compute kernels. Determine whether explicit SIMD intrinsics or LLVM auto-vectorization is the primary strategy. Trace the path from `PhysicalExpr::evaluate()` through to Arrow compute kernel inner loops. Understand how Arrow's 64-byte memory alignment, null bitmap design, and `BooleanBuffer` bit-packing interact with SIMD. Assess the role of LTO and `#[inline]` in cross-crate kernel performance.

**Key Source Files Examined:**
- `datafusion/physical-expr/src/expressions/binary.rs` -- `BinaryExpr::evaluate()`, the central dispatch for arithmetic and comparison
- `datafusion/physical-expr-common/src/datum.rs` -- `apply()` and `apply_cmp()`, the bridge from DataFusion to Arrow `Datum` kernels
- `datafusion/physical-plan/src/filter.rs` -- `FilterExec` stream, calls `filter_record_batch`
- `datafusion/functions-aggregate-common/src/aggregate/groups_accumulator/accumulate.rs` -- `NullState`, `accumulate()`, 64-bit chunk null processing
- `datafusion/functions-aggregate-common/src/aggregate/groups_accumulator/prim_op.rs` -- `PrimitiveGroupsAccumulator` with `get_unchecked_mut`
- `datafusion/physical-plan/src/aggregates/group_values/multi_group_by/primitive.rs` -- vectorized equality with explicit auto-vectorization design
- `datafusion/physical-plan/src/aggregates/group_values/multi_group_by/mod.rs` -- `GroupColumn` trait, `vectorized_intern`
- `datafusion/physical-expr/src/expressions/in_list.rs` -- `BooleanBuffer::collect_bool` usage for membership tests
- `datafusion/physical-plan/src/spill/mod.rs` -- alignment-aware IPC writing
- `Cargo.toml` (workspace root) -- LTO and codegen-units profiles
- `docs/source/user-guide/crate-configuration.md` -- SIMD and LTO documentation
- `rust-toolchain.toml` -- Rust 1.94.0

## 1. High-Level Overview

DataFusion's SIMD strategy is best described as **"auto-vectorization by design"** -- neither DataFusion nor arrow-rs (as of v58.1.0) use explicit SIMD intrinsics (`std::arch`, `packed_simd`, `std::simd`, or `#[target_feature]`). Instead, the entire stack is engineered so that LLVM's auto-vectorizer can emit SIMD instructions for the hot loops. This works through a layered approach:

1. **Arrow's columnar memory layout** provides dense, aligned, contiguous arrays of primitive types that are natural SIMD operands.
2. **Arrow compute kernels** (in the `arrow-arith`, `arrow-ord`, `arrow-select` crates) are written as tight loops over `&[T::Native]` slices, which LLVM auto-vectorizes when compiled with `-C target-cpu=native`.
3. **DataFusion acts as a dispatcher** -- it evaluates expression trees, determines operand types, and delegates to the appropriate Arrow kernel. DataFusion itself writes very few inner loops over data; the hot paths are in arrow-rs.
4. **Where DataFusion does write inner loops** (aggregation accumulation, group-by equality), it uses `unsafe { get_unchecked }` and branchless patterns explicitly designed to help LLVM auto-vectorize.
5. **LTO (`lto = true`, `codegen-units = 1`)** in the release profile enables cross-crate inlining, which is essential because the DataFusion -> arrow-rs boundary would otherwise prevent inlining of kernel inner loops.

The result: with `RUSTFLAGS='-C target-cpu=native'` and LTO, DataFusion achieves SIMD-level performance for arithmetic, comparison, filtering, and aggregation without a single line of platform-specific SIMD code.

## 2. Detailed Analysis

### 2.1 Arrow Memory Alignment and SIMD Enablement

**Arrow's alignment guarantee.** Arrow-rs allocates all buffers through its own allocator which guarantees 64-byte alignment. This is a spec-level requirement of the Apache Arrow format -- all buffers must be aligned to at least 8 bytes, but the Rust implementation uses 64-byte alignment. This means:

- A `PrimitiveArray<Int32Type>` backed by a `Buffer` has its `&[i32]` data aligned to a 64-byte boundary.
- This alignment matches or exceeds the requirements of AVX-512 (64 bytes), AVX2 (32 bytes), SSE (16 bytes), and NEON (16 bytes).
- LLVM can emit aligned load/store instructions (`vmovdqa` on x86) rather than unaligned ones, which is faster on some microarchitectures.

**How DataFusion relies on it.** DataFusion never allocates its own data buffers for columnar data -- it always constructs Arrow arrays through Arrow's builder APIs or kernel return values, which inherit the 64-byte alignment. The only place DataFusion is explicitly alignment-aware is in IPC spill writing:

```
// From datafusion/physical-plan/src/spill/mod.rs:281-288
// Depending on the schema, some array types such as StringViewArray require
// larger (16 byte in this case) alignment. If the actual buffer layout after
// IPC read does not satisfy the alignment requirement, Arrow ArrayBuilder will
// copy the buffer into a newly allocated, properly aligned buffer.
let alignment = get_max_alignment_for_schema(schema);
let mut write_options = IpcWriteOptions::try_new(alignment, false, metadata_version)?;
```

This ensures that when data is spilled to disk and read back, the IPC alignment is sufficient to avoid buffer copies. The `get_max_alignment_for_schema` function walks the schema fields and returns the maximum required alignment (minimum 8 bytes), computed from `BufferSpec::FixedWidth { alignment, .. }`.

**Practical significance for SIMD.** The 64-byte alignment means that every Arrow buffer naturally starts at a cache-line boundary (typical cache lines are 64 bytes). This gives two benefits:
1. SIMD loads never cross cache-line boundaries at the start of the buffer.
2. No special alignment logic is needed in compute kernels -- they can simply iterate over the slice and LLVM handles the rest.

### 2.2 Explicit SIMD vs LLVM Auto-Vectorization

**No explicit SIMD intrinsics anywhere in the DataFusion codebase.** A thorough search found:
- Zero uses of `std::arch`, `std::simd`, `packed_simd`, or `portable_simd`
- Zero uses of `#[target_feature(enable = "...")]`
- Zero uses of `_mm256_*`, `_mm512_*`, or any platform-specific intrinsic

Arrow-rs v58.1.0 (the version used by DataFusion 53.0.0) follows the same approach. The arrow-rs project historically experimented with explicit SIMD (the `simd` feature in older versions used `packed_simd2`), but this was removed. The current arrow-rs relies entirely on LLVM auto-vectorization.

**The auto-vectorization contract.** DataFusion's documentation explicitly describes this contract:

```
# From docs/source/user-guide/crate-configuration.md:61-79

By default, the Rust compiler produces code that runs on a wide range of CPUs,
but may not take advantage of all the features of your specific CPU (such as
certain SIMD instructions). This is especially true for x86_64 CPUs, where the
default target is `x86_64-unknown-linux-gnu`, which only guarantees support for
the `SSE2` instruction set. DataFusion can benefit from the more advanced
instructions in the `AVX2` and `AVX512` to speed up operations like filtering,
aggregation, and joins.

RUSTFLAGS='-C target-cpu=native' cargo run --release
```

This means:
- **Without `-C target-cpu=native`:** Kernels compile with SSE2 only (x86_64 baseline). LLVM still auto-vectorizes, but with 128-bit registers only.
- **With `-C target-cpu=native` on AVX2 hardware:** LLVM uses 256-bit YMM registers, processing 4x f64 or 8x i32 per instruction.
- **With `-C target-cpu=native` on AVX-512 hardware:** LLVM uses 512-bit ZMM registers, processing 8x f64 or 16x i32 per instruction.

**Why auto-vectorization works here.** Arrow compute kernels are ideal for auto-vectorization because:
1. Inner loops iterate over contiguous `&[T]` slices (not scatter/gather patterns).
2. Loop bodies are simple arithmetic or comparison operations.
3. There are no data-dependent branches in the hot path (null handling is done via bitmask operations, not per-element branching).
4. The `Datum` trait enables scalar-vs-array dispatch at the call site, not inside the loop.

### 2.3 Expression Evaluation to Arrow Kernel Call Path

The complete call path from SQL expression to SIMD-enabled computation:

**Step 1: `BinaryExpr::evaluate()` dispatches by operator.**

```
// From binary.rs:273-391
fn evaluate(&self, batch: &RecordBatch) -> Result<ColumnarValue> {
    let lhs = self.left.evaluate(batch)?;   // recursive evaluate
    // ... short-circuit checks ...
    let rhs = self.right.evaluate(batch)?;
    match self.op {
        Operator::Plus => return apply(&lhs, &rhs, add_wrapping),
        Operator::Minus => return apply(&lhs, &rhs, sub_wrapping),
        Operator::Multiply => return apply(&lhs, &rhs, mul_wrapping),
        Operator::Divide => return apply(&lhs, &rhs, div),
        Operator::Eq | Operator::Lt | ... => return apply_cmp(self.op, &lhs, &rhs),
        _ => {}
    }
}
```

**Step 2: `apply()` adapts `ColumnarValue` to Arrow's `Datum` trait.**

```
// From datum.rs:35-56
pub fn apply(
    lhs: &ColumnarValue, rhs: &ColumnarValue,
    f: impl Fn(&dyn Datum, &dyn Datum) -> Result<ArrayRef, ArrowError>,
) -> Result<ColumnarValue> {
    match (&lhs, &rhs) {
        (Array(left), Array(right)) => Ok(Array(f(&left.as_ref(), &right.as_ref())?)),
        (Scalar(left), Array(right)) => Ok(Array(f(&left.to_scalar()?, &right.as_ref())?)),
        (Array(left), Scalar(right)) => Ok(Array(f(&left.as_ref(), &right.to_scalar()?)?)),
        (Scalar(left), Scalar(right)) => { /* scalar path */ }
    }
}
```

The `Datum` trait is the key innovation in arrow-rs -- it allows kernels to accept either arrays or scalars without broadcasting the scalar to a full array. When `to_scalar()` is called, it creates a single-element array marked as scalar, and the kernel's inner loop handles the `array[i] op scalar[0]` pattern by loading the scalar once and broadcasting it within the SIMD operation.

**Step 3: Arrow kernel inner loop (in arrow-rs).**

The actual `add_wrapping`, `sub_wrapping`, `eq`, `lt`, etc. live in the `arrow-arith` and `arrow-ord` crates. While we cannot read their source from this repository, their architecture is well-known:

For arithmetic (e.g., `add_wrapping`):
```
// Conceptual inner loop in arrow-arith:
fn add_wrapping(lhs: &dyn Datum, rhs: &dyn Datum) -> ArrayRef {
    // 1. Downcast to PrimitiveArray<T>
    // 2. Get &[T::Native] slices from both sides
    // 3. Compute null union: lhs.nulls() | rhs.nulls()
    // 4. Inner loop: result[i] = lhs_values[i].wrapping_add(rhs_values[i])
    //    ^ This loop auto-vectorizes with LLVM
    // 5. Return PrimitiveArray with combined nulls
}
```

For comparisons (e.g., `eq`):
```
// Conceptual inner loop in arrow-ord:
fn eq(lhs: &dyn Datum, rhs: &dyn Datum) -> BooleanArray {
    // 1. Downcast to typed arrays
    // 2. Inner loop produces BooleanBuffer via collect_bool or direct bit writing
    // 3. Union null buffers from both inputs
    // 4. Return BooleanArray
}
```

**Step 4: `FilterExec` applies the boolean result.**

```
// From filter.rs:937-941
match as_boolean_array(&array) {
    Ok(filter_array) => {
        let batch = filter_record_batch(&batch, filter_array)?;
    }
}
```

`arrow::compute::filter_record_batch` is an Arrow kernel that uses the `BooleanArray` as a selection mask to produce a new `RecordBatch` containing only the selected rows. Internally, it uses `SlicesIterator` to find contiguous runs of `true` bits (which can be copied as memcpy-like bulk operations), falling back to element-by-element gather for sparse selections.

### 2.4 Null Bitmap Interaction with SIMD

Arrow's null bitmap is stored as a `BooleanBuffer` -- a dense bit-packed buffer using LSB-first byte order, where bit `i` indicates whether element `i` is valid (1) or null (0).

**64-bit chunk processing.** The critical pattern throughout DataFusion and arrow-rs is processing null bitmaps in 64-bit chunks rather than bit-by-bit. This is visible in DataFusion's accumulation code:

```
// From accumulate.rs:393-431 (nulls present, no filter)
let group_indices_chunks = group_indices.chunks_exact(64);
let data_chunks = data.chunks_exact(64);
let bit_chunks = nulls.inner().bit_chunks();  // yields u64 chunks

group_indices_chunks
    .zip(data_chunks)
    .zip(bit_chunks.iter())
    .for_each(|((group_index_chunk, data_chunk), mask)| {
        let mut index_mask = 1;
        group_index_chunk.iter().zip(data_chunk.iter()).for_each(
            |(&group_index, &new_value)| {
                let is_valid = (mask & index_mask) != 0;
                if is_valid {
                    value_fn(group_index, new_value);
                }
                index_mask <<= 1;
            },
        )
    });
```

This `bit_chunks()` method returns an iterator of `u64` values, each encoding 64 validity bits. The `mask & index_mask` check is a single AND instruction. This is significantly more efficient than calling `is_null(i)` per element, which would require recomputing the byte and bit offset each time.

**No-null fast paths.** DataFusion extensively uses null-count checks to skip bitmap processing entirely:

1. **In `NullState::accumulate()`** (accumulate.rs:174-181):
   ```
   if let SeenValues::All { num_values } = &mut self.seen_values
       && opt_filter.is_none()
       && values.null_count() == 0
   {
       accumulate(group_indices, values, None, value_fn);
       // ^ this path skips all null handling
   }
   ```

2. **In `PrimitiveGroupValueBuilder::vectorized_equal_to()`** (primitive.rs:181):
   ```
   if !NULLABLE || (array.null_count() == 0 && !self.nulls.might_have_nulls()) {
       self.vectorized_equal_to_non_nullable(...)  // no null checks
   } else {
       self.vectorized_equal_nullable(...)  // per-element null checks
   }
   ```

3. **In `greatest()`** (greatest.rs:97-107):
   ```
   // Fast path: no nulls, use vectorized arrow kernel directly
   if !lhs.data_type().is_nested()
       && lhs.logical_null_count() == 0
       && rhs.logical_null_count() == 0
   {
       return cmp::gt_eq(&lhs, &rhs).map_err(|e| e.into());
   }
   ```

4. **In `accumulate()`** (accumulate.rs:382-388):
   ```
   match (values.null_count() > 0, opt_filter) {
       (false, None) => {
           // Tightest loop -- no null checks, no filter checks
           for (&group_index, &new_value) in group_indices.iter().zip(data.iter()) {
               value_fn(group_index, new_value);
           }
       }
   ```

5. **Const-generic NULLABLE parameter** in `PrimitiveGroupValueBuilder<T, const NULLABLE: bool>` (primitive.rs:41):
   The `NULLABLE` const generic is resolved at compile time, meaning the non-nullable variant compiles to code with zero null-check overhead. The compiler completely eliminates dead branches.

**Null bitmap union for kernel outputs.** Arrow kernels that combine two arrays compute the output null buffer as the intersection of both input null buffers: `NullBuffer::union(l.nulls(), r.nulls())`. This is a bitwise AND of two `u64` slices -- itself SIMD-vectorizable.

### 2.5 BooleanBuffer and Filter Mask Construction

`BooleanBuffer` is Arrow's bit-packed boolean representation -- 1 bit per element, packed 8 per byte, LSB first. It is used both for null bitmaps and for filter/comparison results.

**`BooleanBuffer::collect_bool` -- the workhorse for SIMD-friendly boolean construction.**

This method is used pervasively throughout DataFusion for constructing boolean results from element-wise predicates:

```
// Pattern found in datum.rs, in_list.rs, greatest.rs, least.rs, join_hash_map.rs, etc.
BooleanBuffer::collect_bool(len, |i| some_predicate(i))
```

Examples from the codebase:
- **Comparison in datum.rs:174**: `BooleanBuffer::collect_bool(len, |i| cmp_with_op(i, i))`
- **InList membership**: `BooleanBuffer::collect_bool(needle_values.len(), |i| self.values.contains(&needle_values[i]))`
- **Hash containment in join_hash_map.rs:468**: `BooleanBuffer::collect_bool(hash_values.len(), |i| { map.find(hash, ...).is_some() })`
- **Greatest function**: `BooleanBuffer::collect_bool(lhs.len(), |i| cmp(i, i).is_ge())`

`collect_bool` internally writes bits into a `MutableBuffer` in chunks, which LLVM can auto-vectorize when the closure body is simple enough. For a comparison closure like `|i| lhs[i] < rhs[i]`, LLVM can vectorize this to compare 32 elements at once (with AVX-512) and pack the results into bits.

**BooleanArray/BooleanBuffer bitwise operations.** Arrow provides bitwise AND (`&`), OR (`|`), NOT (`!`) on `BooleanBuffer`. These operate on the underlying `u64` word arrays:

```
// Used in in_list.rs for combining needle nulls with haystack-induced nulls:
let combined_validity = &needle_validity & &haystack_validity;
```

These word-level operations naturally vectorize -- LLVM turns `u64` AND/OR/NOT chains into SIMD bitwise instructions.

**SlicesIterator for sparse filter application.** When applying a filter to a RecordBatch, Arrow's `SlicesIterator` converts a `BooleanArray` into an iterator of `(start, end)` ranges representing contiguous runs of `true` bits. This is used in:

```
// From binary.rs:895 (pre_selection_scatter)
SlicesIterator::new(left_result).for_each(|(start, end)| {
    // Copy contiguous slices from right array
    let len = end - start;
    right_result.slice(right_array_pos, len).iter().for_each(|v| ...);
});
```

The `SlicesIterator` itself operates on `u64` chunks, using bit tricks (`trailing_zeros`, `count_ones`) to find runs efficiently. For dense selections (most rows pass), this reduces filtering to a few large memcpy operations.

### 2.6 LTO, Inlining, and Cross-Crate Optimization

**Release profile configuration** (from workspace Cargo.toml):

```toml
[profile.release]
codegen-units = 1    # Single codegen unit for maximum optimization
lto = true           # Full link-time optimization
strip = true         # Eliminate debug info

[profile.release-nonlto]
inherits = "release"
codegen-units = 16   # Faster compile, slightly less optimization
lto = false

[profile.ci-optimized]
inherits = "release"
codegen-units = 16
lto = "thin"         # Thin LTO for faster CI builds with good optimization
```

**Why LTO is critical for SIMD.** DataFusion and arrow-rs are separate crates. Without LTO, the Rust compiler cannot inline arrow-rs kernel functions into DataFusion call sites. This means:

1. **Without LTO:** Each call to `arrow::compute::kernels::numeric::add_wrapping` is a function call across the crate boundary. The inner loop of the kernel cannot be specialized for the calling context. The overhead of the function call itself is negligible, but LLVM cannot fuse operations (e.g., a filter followed by an add cannot be fused into a single pass).

2. **With LTO (`lto = true`, `codegen-units = 1`):** LLVM sees the entire program as a single module. It can inline arrow-rs kernel inner loops into DataFusion's `BinaryExpr::evaluate()` call chain. This enables:
   - Loop fusion (combining multiple operations into one pass)
   - Constant propagation through kernel boundaries (e.g., a scalar `ColumnarValue` can be constant-folded into the kernel's inner loop)
   - Dead code elimination of unused kernel branches
   - Better register allocation across the call boundary

3. **With thin LTO (`lto = "thin"`):** A compromise used in CI. Performs some cross-crate optimization but not as aggressively as full LTO.

**`#[inline]` usage.** DataFusion uses `#[inline]` sparingly. In `binary.rs`, only the `boolean_op` helper is marked `#[inline]`:

```rust
#[inline]
fn boolean_op(left: &dyn Array, right: &dyn Array,
    op: impl FnOnce(&BooleanArray, &BooleanArray) -> Result<BooleanArray, ArrowError>,
) -> Result<Arc<dyn Array + 'static>, ArrowError> { ... }
```

This is sufficient because with LTO enabled, LLVM makes its own inlining decisions across all crates. The `#[inline]` on `boolean_op` is a hint for the non-LTO case (development builds).

In the `GroupIndexView` helper (multi_group_by/mod.rs), `#[inline]` is used on the small accessor methods:
```rust
#[inline]
pub fn is_non_inlined(&self) -> bool { (self.0 & NON_INLINED_FLAG) > 0 }

#[inline]
pub fn value(&self) -> u64 { self.0 & VALUE_MASK }
```

Arrow-rs itself uses `#[inline]` more aggressively on its compute kernel functions, which is important for the non-LTO case.

**Rust toolchain.** DataFusion pins to Rust 1.94.0 (from `rust-toolchain.toml`). This is significant because newer Rust/LLVM versions continuously improve auto-vectorization. Rust 1.94 includes LLVM 19, which has good auto-vectorization for the patterns used by arrow-rs.

### 2.7 GroupsAccumulator: DataFusion's Own Vectorized Aggregation

While most compute-intensive work delegates to Arrow kernels, DataFusion's `GroupsAccumulator` system implements its own inner loops for grouped aggregation. This is where DataFusion does its most performance-sensitive loop writing.

**`PrimitiveGroupsAccumulator` update pattern** (prim_op.rs:89-116):

```rust
fn update_batch(&mut self, values: &[ArrayRef], group_indices: &[usize],
    opt_filter: Option<&BooleanArray>, total_num_groups: usize) -> Result<()>
{
    let values = values[0].as_primitive::<T>();
    self.values.resize(total_num_groups, self.starting_value);
    self.null_state.accumulate(
        group_indices, values, opt_filter, total_num_groups,
        |group_index, new_value| {
            // SAFETY: group_index is guaranteed to be in bounds
            let value = unsafe { self.values.get_unchecked_mut(group_index) };
            (self.prim_fn)(value, new_value);
        },
    );
}
```

Key observations:
- `unsafe { get_unchecked_mut }` eliminates bounds checking in the inner loop. This is critical for auto-vectorization -- bounds checks introduce conditional branches that prevent LLVM from vectorizing.
- The `prim_fn` closure (e.g., `|acc, val| *acc += val` for SUM) is generic and inlined by the compiler.
- The `NullState::accumulate()` function has four specialized code paths for the (nulls, filter) combinations, avoiding branches in the hot loop.

**The four-way dispatch in `accumulate()`** (accumulate.rs:382-467):

| Nulls | Filter | Code Path |
|-------|--------|-----------|
| No    | No     | Tight `zip` loop, no checks -- best for auto-vectorization |
| Yes   | No     | 64-bit chunk processing of null bitmap |
| No    | Yes    | Iterator with filter check per element |
| Yes   | Yes    | Iterator with both null and filter checks |

The (false, None) path is the most SIMD-friendly:
```rust
(false, None) => {
    let iter = group_indices.iter().zip(data.iter());
    for (&group_index, &new_value) in iter {
        value_fn(group_index, new_value);
    }
}
```

However, note that even this "vectorizable" loop has a data-dependent store pattern (`values[group_index]`), which limits SIMD effectiveness because the group indices are not contiguous. This is inherent to grouped aggregation -- the values must be scattered into group buckets.

### 2.8 Vectorized Group-By with Auto-Vectorization Hints

DataFusion's `multi_group_by` module contains the most explicit auto-vectorization engineering in the codebase. The `PrimitiveGroupValueBuilder::vectorized_equal_to_non_nullable` method (primitive.rs:60-101) is designed specifically to help LLVM vectorize:

```rust
fn vectorized_equal_to_non_nullable(
    &self, lhs_rows: &[usize], array: &ArrayRef,
    rhs_rows: &[usize], equal_to_results: &mut [bool],
) {
    let array_values = array.as_primitive::<T>().values();
    for (&lhs_row, &rhs_row, equal_to_result) in izip!(...) {
        let result = {
            // Getting unchecked not only for bound checks but because
            // the bound checks are what prevents auto-vectorization
            let left = unsafe { *self.group_values.get_unchecked(lhs_row) };
            let right = unsafe { *array_values.get_unchecked(rhs_row) };
            // Always evaluate, to allow for auto-vectorization
            left.is_eq(right)
        };
        *equal_to_result = result && *equal_to_result;
    }
}
```

Three deliberate design choices for auto-vectorization:

1. **`unsafe { get_unchecked }` everywhere:** The comments explicitly state this is "not only for bound checks but because the bound checks are what prevents auto-vectorization." Bounds checks generate conditional branches that break SIMD vectorization.

2. **Unconditional evaluation ("Always evaluate, to allow for auto-vectorization"):** Instead of short-circuiting on `!*equal_to_result`, the comparison is always performed and combined with `result && *equal_to_result`. This eliminates a data-dependent branch that would prevent vectorization. The result is that unnecessary comparisons are performed for already-false results, but the SIMD throughput more than compensates.

3. **Separate nullable vs non-nullable paths:** The non-nullable path (above) is maximally clean for LLVM. The nullable path (lines 104-137) uses early `continue` on `!*equal_to_result` because the null checks already introduce branching that prevents vectorization.

**Vectorized intern and the batch processing pattern** (mod.rs:422-465):

```rust
fn vectorized_intern(&mut self, cols: &[ArrayRef], groups: &mut Vec<usize>) -> Result<()> {
    // Phase 1: Classify rows into "append" vs "equal_to" buckets
    self.vectorized_operation_preprocess(cols, groups)?;
    // Phase 2: Perform vectorized_append for new rows
    self.vectorized_append(cols)?;
    // Phase 3: Perform vectorized_equal_to for existing rows
    self.vectorized_equal_to(cols, groups);
}
```

This three-phase design separates the hash-table lookup (scatter, not vectorizable) from the equality comparison (vectorizable) and the append (vectorizable). By batching the vectorizable operations, LLVM gets longer contiguous loops to work with.

## 3. Key Design Insights

### Insight 1: Auto-Vectorization is a Deliberate Architecture, Not an Accident

DataFusion and arrow-rs do not merely "hope" LLVM auto-vectorizes -- the entire architecture is designed around it. This manifests as:
- Arrow's columnar layout provides the dense, aligned memory that SIMD requires.
- Compute kernels are written as simple loops over `&[T]` slices with no interior branching.
- Null handling is separated from value processing via 64-bit chunk bitmap iteration.
- The `Datum` trait pushes scalar-vs-array dispatch outside the inner loop.
- DataFusion's own inner loops use `get_unchecked` and unconditional evaluation specifically to enable auto-vectorization.
- Build configuration (`-C target-cpu=native`, LTO) unlocks the SIMD instruction sets.

### Insight 2: The Null Bitmap Design is the SIMD Gatekeeper

The single biggest factor determining SIMD effectiveness is how nulls are handled. Arrow's bit-packed null bitmap is both a blessing and a challenge:

- **Blessing:** 64 null checks fit in a single `u64`. A `null_count() == 0` check enables entire fast paths that skip null handling.
- **Challenge:** Per-element null checking (`if is_valid { process(value) }`) introduces data-dependent branches that prevent vectorization.
- **Solution:** The four-way dispatch pattern (nulls x filter) ensures the tightest loop is used when possible. The no-null path is always the fastest.
- **Implication for data modeling:** Columns declared NOT NULL in the schema directly translate to faster execution, because the non-nullable code paths are compiled with `const NULLABLE: bool = false`.

### Insight 3: Cross-Crate LTO is a Non-Negotiable Performance Requirement

DataFusion's architecture splits compute between two crate ecosystems:
- `arrow-arith`, `arrow-ord`, `arrow-select`: low-level kernels
- `datafusion-physical-expr`, `datafusion-physical-plan`: expression evaluation, operator logic

Without LTO, every call from DataFusion to an Arrow kernel is an opaque function call. The inner loop of `add_wrapping` cannot be inlined into `BinaryExpr::evaluate()`. The release profile's `lto = true` and `codegen-units = 1` are not merely build optimizations -- they are architectural requirements for achieving SIMD performance.

The `codegen-units = 1` setting is equally important: with multiple codegen units, LLVM partitions the crate into independent compilation units, limiting cross-function optimization even within a single crate.

### Insight 4: The Datum Trait Eliminates Scalar Broadcast Without Losing SIMD

A naive implementation of `column + 5` would broadcast `5` into a full array of 8192 elements, then call the array-array add kernel. Arrow's `Datum` trait avoids this:

```
(ColumnarValue::Array(left), ColumnarValue::Scalar(right)) =>
    Ok(ColumnarValue::Array(f(&left.as_ref(), &right.to_scalar()?)?))
```

The kernel receives `(&Array, &Scalar)` and its inner loop loads the scalar once, then uses it for every iteration. LLVM can place the scalar in a register and broadcast it to all SIMD lanes (`vpbroadcastd` on AVX2), achieving the same throughput as array-array operations without the allocation or memory bandwidth of materializing the broadcast array.

### Insight 5: DataFusion Rarely Writes Inner Loops -- Arrow Kernels Do the Work

DataFusion is primarily a **dispatcher**, not a **compute engine**:
- `BinaryExpr::evaluate()` matches on the operator and calls `apply(&lhs, &rhs, add_wrapping)`.
- `FilterExec` calls `filter_record_batch(&batch, filter_array)`.
- `CastExpr` calls `arrow::compute::cast`.
- `GreatestFunc` calls `cmp::gt_eq` for the no-null fast path.

The exceptions where DataFusion writes its own inner loops are:
1. **Grouped aggregation** (`accumulate.rs`, `prim_op.rs`): Because aggregation requires scatter operations into group-indexed accumulators.
2. **Group-by equality** (`primitive.rs`, `bytes_view.rs`): Because the equality check pattern (indexed scatter comparison) does not map to a standard Arrow kernel.
3. **Short-circuit scatter** (`binary.rs` `pre_selection_scatter`): Because this combines two boolean arrays based on a selection mask, which is not a standard Arrow operation.

### Insight 6: Batch Size is the Implicit SIMD Tuning Knob

DataFusion's default batch size of 8192 rows is not arbitrary:
- It is large enough to amortize function call overhead and fill SIMD pipelines.
- It is small enough to keep working sets in L1/L2 cache (8192 x 8 bytes = 64KB for a single i64 column, which fits in most L1 caches).
- `CoalesceBatchesExec` ensures that after filtering (which can produce small batches), batches are re-coalesced to the target size before further processing.
- `FilterExec` has its own `FILTER_EXEC_DEFAULT_BATCH_SIZE` of 8192.

### Insight 7: `BooleanBuffer::collect_bool` is the Unifying Pattern

Across the codebase, `BooleanBuffer::collect_bool(len, |i| predicate(i))` is the standard way to construct boolean masks. Found in:
- Comparison evaluation (`datum.rs`)
- InList membership testing (`in_list.rs`)
- Greatest/least functions (`greatest.rs`, `least.rs`)
- Hash join containment checks (`join_hash_map.rs`)

This is more than a convenience -- it provides a single, well-optimized path for boolean mask construction that Arrow-rs can optimize once and all users benefit. The closure-based API ensures that the bit-packing logic (shifting bits into bytes, handling the final partial byte) is centralized.

## 4. Porting Considerations (DataFusion Patterns -> Trino Rust Worker)

### 4.1 Adopt Auto-Vectorization-First, Explicit SIMD Only Where Needed

DataFusion proves that auto-vectorization is sufficient for high performance in a columnar query engine. For the Trino Rust worker:

- **Use arrow-rs as the compute kernel library.** Do not rewrite arithmetic, comparison, or filter kernels. Arrow-rs provides battle-tested, auto-vectorization-friendly implementations.
- **Compile with `-C target-cpu=native` for production.** This is the single most impactful SIMD setting. Consider making this the default in the worker's build system.
- **Consider explicit SIMD only for:** hash computation (where the AES-NI-based approach can outperform auto-vectorized scalar hashing), string operations (where SIMD string search like `memchr` provides significant speedups), and bloom filters (where explicit SIMD can exploit specific bit manipulation patterns).

### 4.2 LTO Configuration is Mandatory

The worker must use `lto = true` and `codegen-units = 1` for release builds. This is non-negotiable for performance:

```toml
[profile.release]
lto = true
codegen-units = 1
```

The build time cost is significant (potentially 2-5x slower compilation), but the runtime benefit is large. Consider a `release-dev` profile with `lto = false` for development iteration:

```toml
[profile.release-dev]
inherits = "release"
lto = false
codegen-units = 16
```

### 4.3 Null Handling Strategy

Follow DataFusion's multi-path null handling:

1. **Schema-level NOT NULL propagation:** If a column is known non-nullable from the schema, thread this information through to kernel selection at compile time (using const generics like `const NULLABLE: bool`).
2. **Runtime `null_count() == 0` fast paths:** Even for nullable columns, check at batch entry whether the batch actually contains nulls. If not, take the non-nullable path.
3. **64-bit chunk processing for nulls:** When nulls are present, process validity bitmaps in `u64` chunks, not bit-by-bit.
4. **Never branch on individual nulls in the inner loop** if the loop should auto-vectorize. Use bitmask operations to combine results with null information outside the loop.

### 4.4 Replicate the Datum Pattern for Scalar Operations

Trino operators frequently evaluate `column op constant` expressions. Ensure the Trino Rust worker:

1. Detects constant operands at plan time and represents them as scalars (not broadcast arrays).
2. Passes scalars through the `Datum` trait to Arrow kernels, avoiding materialization.
3. For custom kernels not in arrow-rs, implement the scalar path explicitly -- load the scalar into a register and broadcast across SIMD lanes.

### 4.5 Inner Loop Guidelines for Custom Kernels

When writing code that must auto-vectorize (e.g., custom aggregation logic):

1. **Use `unsafe { get_unchecked }` for indexed access** when bounds are guaranteed by invariants. Document the safety contract.
2. **Avoid data-dependent branches in the loop body.** Use conditional moves (`if cond { a } else { b }` compiles to `cmov`) or unconditional evaluation with masking.
3. **Keep the loop body simple.** Complex closures with multiple memory accesses per iteration are harder for LLVM to vectorize.
4. **Separate nullable and non-nullable code paths** at the function level, not inside the loop.
5. **Use `izip!` (from itertools) for multi-array iteration** to help LLVM reason about loop bounds.

### 4.6 Batch Size Tuning

Start with DataFusion's default of 8192 rows. This provides:
- Good SIMD pipeline utilization (8192 / 16 = 512 iterations for AVX-512 on i32)
- L1 cache friendliness for single-column operations (64KB for i64)
- Reasonable overhead amortization for function calls and batch setup

Consider smaller batches (2048-4096) for wide rows (many columns) to avoid L2 cache pressure, and larger batches (16384-32768) for narrow, scan-heavy workloads.

### 4.7 Build System Integration

The Trino Rust worker's build system should:

1. **Set RUSTFLAGS automatically** for production builds: `RUSTFLAGS='-C target-cpu=native'`
2. **Provide a CI profile** with `lto = "thin"` for faster CI builds with reasonable optimization.
3. **Document the SIMD target hierarchy:** SSE2 (baseline) < AVX2 (recommended minimum) < AVX-512 (best performance where available).
4. **Consider PGO (Profile-Guided Optimization)** for production deployments. DataFusion's documentation states PGO can improve performance by up to 25%.
5. **Pin the Rust toolchain** (as DataFusion does with `rust-toolchain.toml`) to ensure reproducible SIMD codegen across builds.

### 4.8 Monitoring and Validation

Since auto-vectorization is opaque, validate it:
- Use `cargo-show-asm` or `RUSTFLAGS='--emit=asm'` to inspect critical inner loops for SIMD instructions (look for `ymm`/`zmm` registers on x86, `v` registers on ARM).
- Add benchmarks for key kernels with and without `-C target-cpu=native` to quantify the SIMD benefit.
- Profile with `perf stat` and check for `instructions_retired_128b_packed_double`, etc., to confirm SIMD utilization at runtime.
- Be aware that DataFusion's aggregation scatter pattern (`values[group_index] += new_value`) does NOT vectorize because of the gather/scatter addressing. This is inherent to grouped aggregation and cannot be improved with SIMD alone.
