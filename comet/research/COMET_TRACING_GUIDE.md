# Apache DataFusion Comet Source Code Tracing Manifest (Hyper-Granular)

**Version:** 0.15.0 (commit f635cad8)

**Context:** Comet is a Spark plugin that replaces Spark's JVM-based physical execution with a native Rust engine powered by Apache DataFusion. It is a direct architectural analogue to the trino-rust-worker project: both take a JVM-based distributed query engine and offload physical plan execution to Rust/DataFusion. Studying Comet provides concrete, production-grade reference implementations for plan translation, JNI/FFI bridging, and fallback handling.

**Instructions for Claude:**
This is an atomic task list for analyzing the Apache DataFusion Comet source code. **DO NOT attempt to execute multiple tasks at once.** The user will specify a Task ID (e.g., "Claude, execute Task 1.1.A").
1. Read the specified source files using your CLI tools to understand the implementation.
2. **"Target Files" are a suggested starting point only** — they are not exhaustive. Automatically expand your research scope to any related dependencies, callers, implementations, or utilities that are relevant to the Focus. Follow the code wherever it leads.
3. Analyze the code deeply based on the specific "Focus" provided.
4. **Provide code snippets** for key concepts in your explanation. Quote relevant source code directly to support your analysis rather than describing it abstractly.
5. Generate the output exactly matching the `RESEARCH_TEMPLATE.md` structure.
6. Stop and wait for the user to verify the output and provide the next command.

---

## Phase 1: Plugin Foundation & Spark Integration
**Objective:** Understand how Comet plugs into Spark's extension points to intercept and replace physical plan execution. This is the entry point for everything else.

**Comparison Point (trino-rust-worker):** Trino has no plugin mechanism for replacing workers — the Rust worker must be a full drop-in process. Study how Comet leverages Spark's extensibility to understand what a "partial replacement" looks like vs. the "full replacement" approach required for Trino.

### Task 1.1: Spark Plugin Entry Point
* **Task 1.1.A: CometPlugin & Driver Initialization**
    * **Target Files:** `spark/src/main/scala/org/apache/spark/Plugins.scala`, `spark/src/main/scala/org/apache/comet/CometSparkSessionExtensions.scala`
    * **Focus:** Trace the full plugin initialization chain:
      1. How does `CometPlugin` register with Spark via `SparkPlugin`? What does `CometDriverPlugin.init()` do at `SparkContext` startup?
      2. How does it validate prerequisites (off-heap memory mode, compatible Spark version)?
      3. How does `CometSparkSessionExtensions` inject the three optimizer rules (`CometScanRule`, `CometExecRule`, `EliminateRedundantTransitions`) into the Spark query planning pipeline?
      4. At what point in Spark's planning lifecycle do these rules fire? (pre-columnar transitions, post-columnar transitions, query-stage prep)
      5. How is the native library (`libcomet.so`) loaded? Trace the `NativeBase` initialization.

### Task 1.2: Configuration Surface
* **Task 1.2.A: Comet Configuration & Feature Flags**
    * **Target Files:** `spark/src/main/scala/org/apache/comet/CometConf.scala`, `common/src/main/scala/org/apache/comet/CometConf.scala` (whichever exists)
    * **Focus:** Map the full configuration surface:
      1. What Spark config keys control Comet behavior? (`spark.comet.enabled`, `spark.comet.exec.enabled`, `spark.comet.exec.shuffle.enabled`, etc.)
      2. How does `spark.comet.exec.allowIncompatible` gate the `Incompatible` support level?
      3. What memory-related configs exist? (`spark.comet.memory.overhead.factor`, `spark.comet.memory.overhead.min`)
      4. What per-operator or per-expression feature toggles exist?

---

## Phase 2: The Fallback Mechanism (Operator Conversion Rules)
**Objective:** Trace how Comet decides which Spark physical operators can be replaced with native execution and which must remain on the JVM. This is the "boundary negotiation" between the two worlds.

**Comparison Point (trino-rust-worker):** The Rust worker must handle unsupported features. Does it fail the entire query? Does it fall back to a Java worker? Comet's three-level SupportLevel system (Compatible/Incompatible/Unsupported) and its contiguous-block optimization provide a mature reference for hybrid execution design.

### Task 2.1: CometExecRule — The Core Conversion Logic
* **Task 2.1.A: Bottom-Up Operator Conversion**
    * **Target Files:** `spark/src/main/scala/org/apache/comet/rules/CometExecRule.scala`
    * **Focus:** Trace the central plan transformation logic:
      1. How does the rule walk the Spark physical plan bottom-up? What is the entry point method (`apply` / `transform`)?
      2. How does `convertToComet` decide whether a specific `SparkPlan` node can be replaced? What checks must pass?
      3. What is the `nativeExecs` map? Enumerate every Spark operator that has a registered Comet replacement (e.g., `ProjectExec → CometProjectExec`, `FilterExec → CometFilterExec`, etc.).
      4. What is the critical constraint that **all children must already be `CometNativeExec`** before a parent can be converted? How does this create "contiguous native blocks"?
      5. How does `withInfo(op, ...)` annotate unsupported operators for `EXPLAIN` output?

* **Task 2.1.B: The SupportLevel System**
    * **Target Files:** `spark/src/main/scala/org/apache/comet/serde/SupportLevel.scala`, `spark/src/main/scala/org/apache/comet/rules/CometExecRule.scala`
    * **Focus:** Analyze the three-tier support classification:
      1. What are the semantics of `Compatible`, `Incompatible`, and `Unsupported`?
      2. How does `Incompatible` differ from `Unsupported`? What concrete examples exist of each?
      3. How does the `spark.comet.exec.allowIncompatible` config gate `Incompatible` operators?
      4. Where are support levels assigned — in the rule, in the serde, or both?
      5. How are support level decisions logged or surfaced to users?

* **Task 2.1.C: Contiguous Block Serialization**
    * **Target Files:** `spark/src/main/scala/org/apache/comet/rules/CometExecRule.scala` (focus on `convertBlock` / block detection)
    * **Focus:** Trace the second pass after bottom-up conversion:
      1. How does `convertBlock()` identify the topmost `CometNativeExec` in each contiguous native block?
      2. How does it serialize the entire native subtree into a single protobuf byte array?
      3. What happens at the boundary between a native block and a JVM operator? How is data transferred (Arrow → UnsafeRow → Arrow)?
      4. How does `EliminateRedundantTransitions` optimize away unnecessary columnar↔row transitions at block boundaries?

### Task 2.2: CometScanRule — Scan Conversion
* **Task 2.2.A: Scan Operator Replacement**
    * **Target Files:** `spark/src/main/scala/org/apache/comet/rules/CometScanRule.scala`
    * **Focus:** How are Spark scan operators (e.g., `FileSourceScanExec`, `BatchScanExec`) converted to Comet scan equivalents?
      1. Which scan types are supported? Which are not?
      2. How does the scan rule interact with `CometExecRule`? Does the scan rule run first?
      3. How are Parquet scans handled differently from other formats (CSV, ORC)?
      4. How does column projection and predicate pushdown flow from Spark through to the native scan?

---

## Phase 3: Plan Translation (Spark Catalyst → Protobuf → DataFusion)
**Objective:** Trace the complete serialization pipeline that translates Spark's Catalyst expression and operator trees into protobuf messages consumable by the Rust native engine. This is the "protocol" between the two worlds.

**Comparison Point (trino-rust-worker):** The Rust worker must parse Trino's `PlanFragment` JSON and reconstruct a DataFusion execution plan. Comet faces the same challenge with Spark's Catalyst plans, but uses protobuf instead of JSON. Study how Comet maps ~150+ Spark expressions to protobuf — this is the same enumeration problem the Rust worker faces with Trino's expression AST (Task 3.3.B in `TRINO_TRACING_GUIDE`).

### Task 3.1: Expression Serialization
* **Task 3.1.A: The Expression Serde Registry**
    * **Target Files:** `spark/src/main/scala/org/apache/comet/serde/QueryPlanSerde.scala`, `spark/src/main/scala/org/apache/comet/serde/CometExpressionSerde.scala`
    * **Focus:** Trace how Spark Catalyst expressions are translated to protobuf:
      1. How does `exprSerdeMap` map `Class[_ <: Expression]` to `CometExpressionSerde[_]` handlers?
      2. How does `exprToProto(expr, inputs, binding)` dispatch to the correct handler?
      3. What is the `CometExpressionSerde` trait? What methods does it define?
      4. How are expression bindings (column references) resolved against input schemas?
      5. Enumerate the expression categories covered: math, string, predicate, temporal, hash, conditional, bitwise, array, map, struct, cast, misc.

* **Task 3.1.B: Expression Category Deep Dives**
    * **Target Files:** `spark/src/main/scala/org/apache/comet/serde/arithmetic.scala`, `spark/src/main/scala/org/apache/comet/serde/strings.scala`, `spark/src/main/scala/org/apache/comet/serde/datetime.scala`, `spark/src/main/scala/org/apache/comet/serde/arrays.scala`, `spark/src/main/scala/org/apache/comet/serde/cast.scala` (or equivalent)
    * **Focus:** Trace expression serialization for each category:
      1. How are arithmetic expressions (Add, Subtract, Multiply, Divide, Remainder) serialized? How is overflow handling communicated?
      2. How are Cast expressions serialized? How is the source and target type pair validated for native support?
      3. How are string expressions (Substring, Upper, Lower, Concat, Like, RegExp) serialized?
      4. How are temporal expressions (DateAdd, DateDiff, UnixTimestamp) serialized? How is timezone handling communicated?
      5. For each category, what expressions are NOT supported and why?

* **Task 3.1.C: Aggregate Expression Serialization**
    * **Target Files:** `spark/src/main/scala/org/apache/comet/serde/QueryPlanSerde.scala` (aggrSerdeMap), `spark/src/main/scala/org/apache/comet/serde/aggregates.scala`
    * **Focus:** Trace how Spark aggregate expressions are translated:
      1. What does `aggrSerdeMap` cover? (Sum, Count, Avg, Min, Max, Corr, Stddev, etc.)
      2. How does `aggExprToProto` handle aggregate-specific fields (filter, query context, expression ID)?
      3. How does partial vs. final aggregation mode affect serialization?
      4. What aggregate functions are NOT supported?

### Task 3.2: Operator Serialization
* **Task 3.2.A: The Operator Serde Registry**
    * **Target Files:** `spark/src/main/scala/org/apache/comet/serde/CometOperatorSerde.scala`, `spark/src/main/scala/org/apache/comet/rules/CometExecRule.scala` (nativeExecs map)
    * **Focus:** Trace how Spark physical operators are translated to protobuf:
      1. How does `CometOperatorSerde[T]` define the `convert(op, builder, childOp*) → Option[Operator]` method?
      2. How does `createExec(nativeOp, op)` construct a `CometNativeExec` from the protobuf?
      3. Trace the serialization of key operators: Filter, Project, HashAggregate, SortMergeJoin, BroadcastHashJoin, Sort, Window, Limit.
      4. How are operator-specific parameters serialized? (e.g., join type, join condition, sort order, partition spec)

### Task 3.3: Data Type Serialization
* **Task 3.3.A: Spark DataType → Protobuf Type Mapping**
    * **Target Files:** `spark/src/main/scala/org/apache/comet/serde/QueryPlanSerde.scala` (serializeDataType, supportedDataType), `native/proto/src/proto/types.proto`
    * **Focus:** Trace the type serialization layer:
      1. How does `serializeDataType(dt)` map Spark `DataType` to protobuf `Types.DataType`?
      2. What type IDs are assigned (0–16)?
      3. How are parameterized types handled? (Decimal precision/scale, Array element type, Map key/value types, Struct field types)
      4. What types are NOT supported by `supportedDataType(dt)`? Why?

### Task 3.4: The Protobuf Schema
* **Task 3.4.A: Operator & Expression Proto Definitions**
    * **Target Files:** `native/proto/src/proto/operator.proto`, `native/proto/src/proto/expr.proto`, `native/proto/src/proto/types.proto`, `native/proto/src/proto/partitioning.proto`
    * **Focus:** Document the complete protobuf schema:
      1. What does the `Operator` message look like? What `oneof op_struct` variants exist? (Scan, Projection, Filter, Sort, HashAggregate, Limit, ShuffleWriter, Expand, SortMergeJoin, HashJoin, Window, NativeScan, IcebergScan, etc.)
      2. What does the `Expr` message look like? What `oneof expr_struct` variants exist?
      3. How is `Partitioning` defined for shuffle writers?
      4. How does `ConfigMap` pass Spark configuration to the native engine?
      5. How does the `Metric` proto tree surface native metrics to Spark UI?

---

## Phase 4: The JNI & Arrow FFI Bridge
**Objective:** Trace the exact mechanism by which the JVM invokes Rust code and passes data across the language boundary with zero-copy via the Arrow C Data Interface. This is the most performance-critical boundary in the system.

**Comparison Point (trino-rust-worker):** The Rust worker uses HTTP/JSON to receive plans and a custom binary wire format for shuffle data — no JNI. However, if the architecture ever evolves toward an in-process native library (like Velox's Prestissimo), Comet's JNI/FFI bridge is the exact pattern to follow. Even for the current HTTP-based design, studying how Comet manages memory across the JVM/Rust boundary is directly relevant to managing memory across HTTP boundaries.

### Task 4.1: JVM-Side JNI Declarations
* **Task 4.1.A: Native Method Declarations & Arrow FFI Pointers**
    * **Target Files:** `spark/src/main/scala/org/apache/comet/Native.scala`, `spark/src/main/scala/org/apache/comet/NativeBase.scala`
    * **Focus:** Analyze the JNI interface contract:
      1. Enumerate every `@native` method declared. For each, document: name, parameters, return type, and purpose.
      2. How are `arrayAddrs` / `schemaAddrs` (Arrow C Data Interface `FFI_ArrowArray` / `FFI_ArrowSchema` pointers) allocated on the JVM side?
      3. How does the JVM allocate pointer arrays and the Rust side write into them?
      4. What lifecycle management exists? (createPlan → executePlan → releasePlan)
      5. What tracing/profiling hooks exist? (traceBegin/traceEnd, logMemoryUsage, getRustThreadId)

* **Task 4.1.B: CometExecIterator — The Execution Driver**
    * **Target Files:** `spark/src/main/scala/org/apache/comet/CometExecIterator.scala`
    * **Focus:** Trace the Scala iterator that drives native execution:
      1. How does construction call `nativeLib.createPlan(...)` with protobuf bytes and JVM iterator references?
      2. How does `hasNext` → `executePlan()` → `ColumnarBatch` work? What is the per-batch execution protocol?
      3. How does the iterator handle end-of-stream (return value -1)?
      4. How does `close()` call `releasePlan()` for cleanup?
      5. How are Arrow `ColumnarBatch` objects constructed from the FFI pointer arrays?

### Task 4.2: Rust-Side JNI Implementations
* **Task 4.2.A: JNI API Entry Points**
    * **Target Files:** `native/core/src/execution/jni_api.rs`, `native/core/src/lib.rs`
    * **Focus:** Trace the Rust implementations of JNI methods:
      1. How does `Java_org_apache_comet_Native_createPlan` deserialize protobuf, initialize the Tokio runtime, create a DataFusion `SessionContext`, and return an opaque `ExecutionContext` pointer?
      2. How does `Java_org_apache_comet_Native_executePlan` handle the two execution modes: (a) plans with JVM scan sources (busy-poll with `pull_input_batches`), and (b) plans without JVM scan sources (Tokio task + mpsc channel)?
      3. How does the `ExecutionContext` struct manage lifetime? (heap-allocated, stored as raw pointer `jlong`)
      4. How is the global Tokio runtime initialized? (once, using `spark.executor.cores` as thread count)

* **Task 4.2.B: Arrow C Data Interface — Zero-Copy Transfer**
    * **Target Files:** `native/core/src/execution/jni_api.rs` (focus on `move_to_spark`), Arrow FFI modules
    * **Focus:** Trace the zero-copy data transfer mechanism:
      1. How does `move_to_spark(arrayAddr, schemaAddr)` export a `RecordBatch` column to Arrow C Data Interface structs?
      2. Who owns the memory after the transfer? How is it freed? (JVM-side `ArrowBuf` release → Rust deallocator callback?)
      3. What happens when the JVM side closes the `ColumnarBatch`? How does memory flow back?
      4. Are there any copies in this path? What makes it "zero-copy"?

* **Task 4.2.C: Columnar ↔ Row Conversion**
    * **Target Files:** `native/core/src/execution/jni_api.rs` (columnarToRowInit/Convert/Close), relevant Spark-side callers
    * **Focus:** Trace the columnar-to-row conversion path:
      1. When is this path used? (native block output → JVM operator that expects `UnsafeRow`)
      2. How does the Rust side convert Arrow columnar data to Spark's `UnsafeRow` binary format?
      3. What is the performance cost of this conversion vs. staying fully columnar?

### Task 4.3: Shuffle JNI Interface
* **Task 4.3.A: Native Shuffle Operations**
    * **Target Files:** `native/core/src/execution/jni_api.rs` (writeSortedFileNative, sortRowPartitionsNative, decodeShuffleBlock), `native/shuffle/`
    * **Focus:** Trace the native shuffle path:
      1. How does `writeSortedFileNative` write UnsafeRows to Arrow IPC files?
      2. How does `sortRowPartitionsNative` perform in-memory radix sort of partition IDs?
      3. How does `decodeShuffleBlock` decompress and decode IPC blocks into Arrow arrays?
      4. How does this native shuffle path compare to Spark's built-in shuffle? What performance advantage does it provide?

---

## Phase 5: Rust Native Engine (DataFusion Planner & Execution)
**Objective:** Trace how the deserialized protobuf plan is converted into a DataFusion physical execution plan and executed. This is where the Rust engine meets DataFusion.

**Comparison Point (trino-rust-worker):** The Rust worker must perform the same translation: Trino `PlanFragment` JSON → DataFusion `ExecutionPlan` tree. Comet's `PhysicalPlanner` is the closest existing reference for this translation layer. Study how it maps protobuf operator/expression variants to DataFusion nodes — the Rust worker will need an analogous mapper from Trino's JSON plan nodes.

### Task 5.1: The Physical Planner
* **Task 5.1.A: Protobuf → DataFusion Plan Translation**
    * **Target Files:** `native/core/src/execution/planner.rs`
    * **Focus:** Trace the `PhysicalPlanner::create_plan()` method:
      1. How does it take a protobuf `Operator` tree and produce a `(Vec<ScanExec>, Vec<ShuffleScanExec>, Arc<SparkPlan>)`?
      2. How does the `match` on `OpStruct` variants dispatch to operator-specific construction? Enumerate the supported variants.
      3. How does it recursively build child plans?
      4. How does `OperatorRegistry::global()` provide an extensible override mechanism?
      5. How are scan inputs (`ScanExec`) wired to JVM-side iterators for data feeding?

* **Task 5.1.B: Expression Reconstruction**
    * **Target Files:** `native/core/src/execution/planner.rs` (create_expr, create_scalar_function_expr)
    * **Focus:** Trace expression reconstruction from protobuf:
      1. How does `create_expr(spark_expr, input_schema)` produce `Arc<dyn PhysicalExpr>`?
      2. How does `ExpressionRegistry::global()` provide extensible overrides?
      3. How are scalar functions resolved from the session context's `FunctionRegistry`?
      4. How are Bound references mapped to DataFusion `Column` expressions?
      5. How are Cast, CaseWhen, If, In, and NormalizeNaNAndZero handled?

### Task 5.2: SparkPlan Wrapper
* **Task 5.2.A: The SparkPlan Execution Wrapper**
    * **Target Files:** `native/core/src/execution/spark_plan.rs`
    * **Focus:** Trace the `SparkPlan` wrapper:
      1. How does `SparkPlan` wrap `Arc<dyn ExecutionPlan>` while carrying Spark plan ID and child references?
      2. How does the `native_plan` field relate to the underlying DataFusion plan?
      3. How are metrics collected and propagated back through SparkPlan to the JVM?

### Task 5.3: Spark-Compatible Expression Implementations
* **Task 5.3.A: Rust Expression Implementations**
    * **Target Files:** `native/spark-expr/src/`, `native/aggregates/src/`
    * **Focus:** Trace how Spark-specific expression semantics are implemented in Rust:
      1. What expressions require custom Rust implementations beyond standard DataFusion? Why?
      2. How do Spark-compatible Cast semantics differ from DataFusion/Arrow defaults? (e.g., overflow handling, null behavior, string-to-number parsing)
      3. How are Spark-compatible aggregate functions implemented? (Spark's Sum with overflow behavior, Avg with decimal precision, etc.)
      4. How are UDFs registered in the DataFusion session context via `prepare_datafusion_session_context`?

### Task 5.4: ScanExec — JVM Data Ingestion
* **Task 5.4.A: JVM Scan Data Flow**
    * **Target Files:** `native/core/src/execution/` (ScanExec implementation), `native/core/src/execution/jni_api.rs` (pull_input_batches)
    * **Focus:** Trace how scan data flows from JVM to Rust:
      1. How does `ScanExec` implement `ExecutionPlan` to produce a `SendableRecordBatchStream`?
      2. How does `pull_input_batches` call back into the JVM to fetch the next batch of Arrow data?
      3. How does the busy-poll loop in `executePlan` interact with `ScanExec`'s `Poll::Pending` signals?
      4. How does the JVM-side `CometBatchIterator` provide data to the Rust scan?

---

## Phase 6: Memory Management & Metrics
**Objective:** Trace how Comet manages native memory alongside the JVM's memory accounting, and how execution metrics flow from Rust back to Spark's UI.

**Comparison Point (trino-rust-worker):** The Rust worker must report memory usage back to the Trino Coordinator (via `TaskStatus` JSON). Comet faces a similar challenge: reporting native memory to Spark's memory manager. Study how Comet bridges these memory accounting worlds.

### Task 6.1: Native Memory Management
* **Task 6.1.A: Allocator Selection & Memory Pool**
    * **Target Files:** `native/core/src/lib.rs` (global allocator), `native/core/Cargo.toml` (features: jemalloc, mimalloc), DataFusion memory pool configuration in `jni_api.rs`
    * **Focus:** Trace memory management strategy:
      1. How does Comet select between jemalloc and mimalloc? What compile-time features control this?
      2. How is the DataFusion `MemoryPool` configured in the `SessionContext`? What limits are set?
      3. How does `spark.comet.memory.overhead.factor` interact with Spark's executor memory model?
      4. How does native memory spilling work when the DataFusion pool is exhausted?

* **Task 6.1.B: Cross-Boundary Memory Accounting**
    * **Target Files:** `spark/src/main/scala/org/apache/comet/CometExecIterator.scala`, `native/core/src/execution/jni_api.rs` (logMemoryUsage)
    * **Focus:** Trace how memory is tracked across the JVM/Rust boundary:
      1. How does the JVM side know how much native memory is in use?
      2. How does `logMemoryUsage` report native allocations?
      3. How does Spark's `TaskMemoryManager` interact with Comet's native memory?
      4. What happens when Spark requests memory spill from an executor running Comet?

### Task 6.2: Metrics Propagation
* **Task 6.2.A: Native Metrics → Spark UI**
    * **Target Files:** `native/proto/src/proto/metric.proto`, `spark/src/main/scala/org/apache/comet/` (metrics-related classes), `native/core/src/execution/` (metrics collection)
    * **Focus:** Trace how execution metrics flow from Rust to Spark:
      1. How does the `Metric` proto tree represent native execution metrics?
      2. How are DataFusion metrics (elapsed time, rows processed, spill count) captured in the Rust engine?
      3. How are these metrics transferred across JNI and surfaced in Spark's SQL UI?
      4. How does `CometMetricsListener` aggregate and display native metrics?

---

## Phase 7: Test Infrastructure & Validation Strategy
**Objective:** Map Comet's complete test hierarchy to understand how correctness is validated — from individual expression compatibility to full Spark query equivalence. Comet's testing challenge is unique: every native result must be **bit-for-bit identical** to Spark's JVM output, or the fallback/incompatibility classification is wrong.

**Comparison Point (trino-rust-worker):** The Rust worker faces the same correctness bar: query results from the Rust worker must be indistinguishable from the Java worker. Comet's approach to testing Spark compatibility (especially edge cases around null handling, overflow, Cast semantics, and timezone behavior) is directly applicable. See `TRINO_TRACING_GUIDE` Phase 6 for the Trino test hierarchy and `DATAFUSION_TRACING_GUIDE` Phase 6 for DataFusion's sqllogictest patterns.

### Task 7.1: Test Hierarchy, Frameworks & CI
* **Task 7.1.A: Test Organization & Infrastructure**
    * **Target Files:** Top-level `pom.xml` or `build.sbt` (test configuration), `spark/src/test/`, `native/core/src/` (`#[cfg(test)]` modules), `.github/` (CI workflows), `Makefile` or build scripts
    * **Focus:** Map the complete test taxonomy:
      1. What test categories exist? (Scala unit tests, Rust unit tests, integration tests, end-to-end Spark tests, benchmarks)
      2. What frameworks are used on each side? (ScalaTest/JUnit on JVM, `#[cfg(test)]`/`#[test]` in Rust)
      3. How are tests organized? (per-operator test suites, per-expression test suites, full SQL query suites)
      4. What CI infrastructure runs these tests? (GitHub Actions workflows, Spark version matrix, test profiles)
      5. How are tests run across multiple Spark versions (3.4, 3.5, 4.0)? How is version-specific behavior gated?

### Task 7.2: Expression Compatibility Tests
* **Task 7.2.A: Spark vs. Native Expression Equivalence**
    * **Target Files:** `spark/src/test/scala/org/apache/comet/` (expression test suites), `native/spark-expr/src/` (Rust-side `#[cfg(test)]` modules)
    * **Focus:** How is expression-level compatibility validated?
      1. How are Spark expressions tested for bit-exact equivalence between JVM and native execution?
      2. How are edge cases tested? (null inputs, overflow, NaN, infinity, empty strings, boundary values)
      3. How are Cast expressions tested across all supported type pairs?
      4. How are timezone-sensitive expressions tested? (different timezones, DST transitions)
      5. Are there property-based or fuzz tests that generate random inputs and compare JVM vs. native results?
      6. How are Rust-side expression unit tests structured? Do they test against known Spark output values?

### Task 7.3: Operator & Query-Level Tests
* **Task 7.3.A: Operator Correctness & SQL Query Tests**
    * **Target Files:** `spark/src/test/scala/org/apache/comet/` (operator test suites, SQL test suites), `spark/src/test/scala/org/apache/comet/exec/` (if exists)
    * **Focus:** How are operators and full queries validated?
      1. How are individual operators tested? (synthetic DataFrames, Spark SQL queries, result comparison)
      2. How are joins tested across all join types (inner, outer, semi, anti, cross) and join strategies (hash, sort-merge, broadcast)?
      3. How are aggregations tested? (partial vs. final, with/without GROUP BY, DISTINCT aggregates)
      4. How are window functions tested? (frame specs, ranking functions, ROWS vs. RANGE)
      5. Are there TPC-H or TPC-DS query suites used for end-to-end validation?
      6. How do tests verify that fallback produces correct results when a native operator is unsupported?

### Task 7.4: Shuffle & Data Exchange Tests
* **Task 7.4.A: Native Shuffle Correctness**
    * **Target Files:** `spark/src/test/scala/org/apache/comet/` (shuffle test suites), `native/shuffle/src/` (Rust-side tests)
    * **Focus:** How is shuffle correctness validated?
      1. How are native shuffle read/write paths tested for data integrity?
      2. How is partition assignment tested for hash consistency with Spark's partitioner?
      3. How are edge cases tested? (empty partitions, skewed data, large payloads, compression)
      4. Are there tests that mix native shuffle with JVM-based shuffle stages?

### Task 7.5: Fallback & Boundary Tests
* **Task 7.5.A: Hybrid Execution Correctness**
    * **Target Files:** `spark/src/test/scala/org/apache/comet/` (fallback test suites, transition tests)
    * **Focus:** How is the JVM↔native boundary validated?
      1. How are columnar↔row transitions at native block boundaries tested for data integrity?
      2. How are queries with mixed native and JVM operators tested?
      3. How are SupportLevel classifications tested? (is there a test that asserts specific operators are Compatible/Incompatible/Unsupported?)
      4. How are regression tests structured when a new operator or expression gains native support?
      5. How is `EliminateRedundantTransitions` tested to ensure it doesn't drop necessary conversions?
