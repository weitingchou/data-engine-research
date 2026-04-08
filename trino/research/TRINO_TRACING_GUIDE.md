# Trino Source Code Tracing Manifest (Hyper-Granular)

**Instructions for Claude:**
This is an atomic task list for analyzing the Trino source code. **DO NOT attempt to execute multiple tasks at once.** The user will specify a Task ID (e.g., "Claude, execute Task 1.1.A"). 
1. Read the specified source files using your CLI tools to understand the implementation.
2. **"Target Files" are a suggested starting point only** — they are not exhaustive. Automatically expand your research scope to any related dependencies, callers, implementations, or utilities that are relevant to the Focus. Follow the code wherever it leads.
3. Analyze the code deeply based on the specific "Focus" provided.
4. **Provide code snippets** for key concepts in your explanation. Quote relevant source code directly to support your analysis rather than describing it abstractly.
5. Generate the output exactly matching the `RESEARCH_TEMPLATE.md` structure.
6. Stop and wait for the user to verify the output and provide the next command.

---

## Phase 1: Foundation Tracing Guide (Memory Layout)

### Task 1.1: Slice
* **Task 1.1.A: The `Slice` Memory Interface & Metadata**
    * **Target Files:** `io.airlift.slice.Slice`, `io.airlift.slice.Slices`
    * **Focus:** Analyze the internal metadata of a `Slice` (typically a base object reference, an address offset, and a size). Trace the APIs that utilize `Unsafe` memory access. Contrast `HeapSlice` with `DirectSlice`.

### Task 1.2: Block
* **Task 1.2.A: The `Block` Interface & Internal Metadata**
    * **Target Files:** `io.trino.spi.block.Block`, `io.trino.spi.block.VariableWidthBlock`, `io.trino.spi.block.LongArrayBlock`
    * **Focus:** Open `VariableWidthBlock` and trace its internal fields (`Slice`, `int[] offsets`, `boolean[] valueIsNull`). Analyze the read-only contract of the `Block` interface. Specifically trace the `getRegion()` method to understand how zero-copy columnar slicing is implemented without duplicating the underlying memory.

### Task 1.3: Page
* **Task 1.3.A: The `Page` Interface & Zero-Copy Mutations**
    * **Target Files:** `io.trino.spi.Page`
    * **Focus:** Inspect the internal fields of a `Page` (`Block[]`, `positionCount`). Analyze `Page` as a passive data container. Trace how `Page.getColumns()` and `Page.prependColumn()` create new `Page` instances via shallow copies of `Block` references. 

### Task 1.4: Physical Data Mapping
* **Task 1.4.A: Physical Data Mapping (The S3 Bridge)**
    * **Target Files:** `io.trino.spi.connector.ConnectorPageSource`, `io.trino.parquet.ParquetDataSource` (or ORC equivalent), `io.trino.plugin.iceberg.IcebergPageSource`
    * **Focus:** Trace the path of an S3 read. How does the data source pull a range of bytes from the object store into a `Slice`? How does the format reader extract values to populate the `BlockBuilder` and ultimately emit a `Page`?

---

## Phase 2: Worker Scheduling & The Driver (Execution Model)
**Objective:** Map the hierarchical relationship between Tasks and Drivers and trace their full lifecycles to design a compatible async/await model.

### Task 2.1: Task Architecture & Lifecycle
* **Task 2.1.A: Task Creation & Resource Wiring**
  * **Target Files:** `io.trino.execution.SqlTask`, `io.trino.execution.SqlTaskExecution`, `io.trino.execution.TaskManagerConfig`
  * **Focus:** How is a `SqlTask` initialized when a `TaskUpdateRequest` hits the worker? What triggers its creation? Trace how it wires up the `OutputBuffer`, `QueryContext`, and `TaskStateMachine` at birth.
* **Task 2.1.B: The Task State Machine & Terminal Transitions**
  * **Target Files:** `io.trino.execution.TaskStateMachine`
  * **Focus:** Trace the full state transitions: `PLANNED` -> `RUNNING` -> `FLUSHING` -> `FINISHED` (`FAILED`, `CANCELED`). Exactly what logic determines the transition from `FLUSHING` to `FINISHED`?
* **Task 2.1.C: The Task-Driver Relationship (Concurrency Scaling)**
  * **Target Files:** `io.trino.execution.SqlTaskExecution`, `io.trino.operator.PipelineFactory`, `io.trino.operator.PipelineContext`
  * **Focus:** Analyze how a single `SqlTask` creates multiple `PipelineContexts`. Trace the logic that decides how many `Driver` instances to spawn for a single pipeline based on available splits and concurrency settings.

### Task 2.2: The Scheduling Engine
* **Task 2.2.A: The Thread Pool Executor**
  * **Target Files:** `io.trino.execution.executor.TaskExecutor`
  * **Focus:** Analyze the core thread pool (Runner threads). How does it accept tasks? How does it manage the priority queue of `PrioritizedSplitRunner` objects?
* **Task 2.2.B: Split Prioritization & Time Quanta**
  * **Target Files:** `io.trino.execution.executor.PrioritizedSplitRunner`, `io.trino.execution.executor.TaskHandle`
  * **Focus:** How does the executor track CPU "quanta" (time slices)? Trace how it calculates the priority of a split to ensure fair scheduling across multiple queries.

### Task 2.3: Driver Lifecycle & The Execution Loop
* **Task 2.3.A: Driver Initialization & Pipeline Plumbing**
  * **Target Files:** `io.trino.operator.Driver`, `io.trino.operator.DriverContext`
  * **Focus:** How is a `Driver` instantiated as a sequence of `Operator` instances? Trace how the `DriverContext` is used to track memory and CPU at the driver level.
* **Task 2.3.B: The Cooperative Yield Loop (The Engine Heart)**
  * **Target Files:** `io.trino.operator.Driver` (focus on `processFor()` and `processInternal()`)
  * **Focus:** This is the core cooperative loop. Document exactly what causes a `Driver` to suspend (yield) and return a `ListenableFuture` to the executor (e.g., blocked on an operator, or quantum exhausted).
* **Task 2.3.C: Driver Termination & Cleanup**
  * **Target Files:** `io.trino.operator.Driver` (focus on `close()` and `isFinished()`)
  * **Focus:** When is a `Driver` considered done? Trace the cleanup path: closing operators, releasing memory back to the `PipelineContext`, and signaling the `TaskStateMachine`.

---

## Phase 3: Operator Internals and Data Processing (Physical Plan Execution)
**Objective:** Trace the physical data processing nodes to understand Trino's columnar, streaming, Volcano-style execution engine. This phase ignores scheduling and focuses purely on how data is transformed in memory.

### Task 3.1: The Operator Engine Fundamentals
Before looking at specific operations, we need to understand the contract every Operator must follow to allow cooperative multitasking.

* **Task 3.1.A: The Operator State Machine**
    * **Target Files:** `io.trino.operator.Operator`, `io.trino.operator.OperatorFactory`
    * **Focus:** Analyze the non-blocking Volcano model. Trace the exact sequence of `needsInput()`, `addInput()`, `getOutput()`, `isFinished()`, and `isBlocked()`. How does an Operator tell the Driver "I need more data" vs. "I need to wait for memory"?
* **Task 3.1.B: Operator Resource Context**
    * **Target Files:** `io.trino.operator.OperatorContext`
    * **Focus:** How does an individual Operator track its CPU time, wall time, and memory allocations? How do these localized metrics bubble up to the `DriverContext`?
* **Task 3.1.C: The Complete Operator Catalog**
    * **Target Files:** `io.trino.operator.*Operator.java`, `io.trino.operator.*OperatorFactory` (all ~43 Operator implementations in `io.trino.operator/` and sub-packages)
    * **Focus:** Enumerate every `Operator` implementation in Trino 480. For each operator, classify it along these dimensions: (1) **Category** — source, sink, transform, join, aggregation, window, exchange, metadata, or DML; (2) **Pipeline role** — does it start a pipeline (source), end a pipeline (sink), or sit in the middle (transform)? (3) **State model** — stateless (streaming, one page in / one page out), stateful-blocking (accumulates all input before producing output), or stateful-streaming (maintains state but can produce output incrementally)? (4) **Spill support** — does this operator implement `startMemoryRevoke()`/`finishMemoryRevoke()`? (5) **Brief function** — one-sentence description of what the operator does. The output should be a comprehensive reference table covering every operator, grouped by category.

### Task 3.2: The Data Payload (Pages & Blocks)
Operators don't process rows; they process columnar batches. Understanding these data structures is mandatory for tracing Operator logic.

* **Task 3.2.A: The Columnar Memory Model**
    * **Target Files:** `io.trino.spi.Page`, `io.trino.spi.block.Block`
    * **Focus:** How is a `Page` structured? Trace how a `Block` represents a single column of data in memory (e.g., `DictionaryBlock`, `RunLengthEncodedBlock`, `VariableWidthBlock`). How do Operators read from these structures without copying data?

### Task 3.3: Stateless & Linear Pipelines
Tracing the simplest data flow where one input page directly results in one or more output pages.

* **Task 3.3.A: Simple Projection & Filtering**
    * **Target Files:** `io.trino.operator.ScanFilterAndProjectOperator`, `io.trino.operator.project.PageProcessor`
    * **Focus:** Trace a raw block of data coming from a connector, passing through a filter, and yielding a transformed `Page`. How does Trino compile these expressions into bytecode for faster execution?
* **Task 3.3.B: Expression Serialization Format** `[KG-7]`
    * **Target Files:** `io.trino.sql.planner.plan.Assignments` (projection expressions), `io.trino.sql.ir.Expression` and all subclasses (`io.trino.sql.ir.Call`, `io.trino.sql.ir.Comparison`, `io.trino.sql.ir.Reference`, `io.trino.sql.ir.Constant`, etc.), Jackson serialization annotations on `Expression`
    * **Focus:** The coordinator compiles SQL into optimized bytecode for the Java worker (`ExpressionCompiler` → `PageProcessor`). But what is actually *sent over the wire* to the worker?
      1. Is the wire format an expression AST (tree of `Expression` nodes), pre-compiled JVM bytecode, or something else entirely?
      2. If AST: trace the `Expression` class hierarchy. Document every `Expression` subclass and its JSON fields (e.g., `Call` has `functionName` + `arguments`, `Comparison` has `operator` + `left` + `right`, `Reference` has `name` referring to a column, etc.).
      3. How are function references serialized? (By name? By a `FunctionId`? With type signature?)
      4. How are literal constants serialized? (Inline value? Type-tagged? How are complex types like `ARRAY` or `ROW` literals represented?)
      5. How does the worker reconstruct executable expressions from this wire format? Trace from JSON deserialization to `PageProcessor` compilation.
      6. Capture concrete JSON for: a comparison (`col1 > 10`), an arithmetic expression (`col1 + col2 * 3`), a function call (`UPPER(name)`), and a CAST expression.

### Task 3.6: Vectorization & SIMD Utilization `[KG-14]`
How does Trino leverage CPU vectorization for columnar compute? Critical for understanding what performance the Java worker achieves versus what the Rust worker can target with native SIMD.

* **Task 3.6.A: JVM Auto-Vectorization & Explicit SIMD**
    * **Target Files:** `io.trino.operator.project.PageProcessor`, `io.trino.sql.gen.ColumnarFilterCompiler`, `io.trino.sql.gen.PageFunctionCompiler`, `io.trino.type.TypeOperators`, hash function implementations (`XxHash64`), `io.trino.spi.block.*Block` (bulk read/write loops)
    * **Focus:** Analyze Trino's relationship with SIMD hardware:
      1. Does Trino use explicit SIMD intrinsics (e.g., Java Vector API / `jdk.incubator.vector`)? Or does it rely solely on JVM auto-vectorization of tight loops?
      2. How do the bytecode-compiled filter/projection expressions interact with JIT vectorization? Does the `ColumnarFilterCompiler` (columnar path) produce loops that HotSpot can auto-vectorize?
      3. Trace the inner loops of critical compute paths: hash computation (`XxHash64`), comparison operators, arithmetic, string operations. Are these structured for auto-vectorization (no branches, contiguous memory access, fixed-width types)?
      4. What is the impact of Trino's MSB-first null bitmap on vectorization? Does the null-check loop prevent auto-vectorization?
      5. How does the `SWAR` (SIMD Within A Register) byte-level probing in `FlatGroupByHash` work? Is this Trino's closest equivalent to explicit SIMD?

### Task 3.4: Stateful & Complex Pipelines
Tracing operations that must hold state across multiple `addInput()` calls, and operations that bridge multiple Pipelines.

* **Task 3.4.A: Hash Join - The Build Pipeline**
    * **Target Files:** `io.trino.operator.HashBuilderOperator`, `io.trino.operator.join.JoinHash`
    * **Focus:** How does Trino ingest the entire "build" side of a join into memory? Trace the internal structure of the hash table. How does it handle memory limits before the probe phase begins?
* **Task 3.4.B: Hash Join - The Probe Pipeline**
    * **Target Files:** `io.trino.operator.LookupJoinOperator`
    * **Focus:** How does the "probe" side stream through and match against the built hash table without blocking? How are the output pages constructed from the matched indices?
* **Task 3.4.C: Aggregation & Accumulators**
    * **Target Files:** `io.trino.operator.aggregation.AggregationOperator`, `io.trino.operator.aggregation.Accumulator`
    * **Focus:** Trace a Group-By operation. How does the operator maintain state across multiple `addInput()` calls? Differentiate between partial (local) aggregations and final (global) aggregations.

### Task 3.5: Resilience and Memory Management
What happens when a single Operator demands more memory than the worker can provide?

* **Task 3.5.A: The Disk Spilling Mechanism**
    * **Target Files:** `io.trino.spill.Spiller`, `io.trino.operator.SpillContext`
    * **Focus:** Trace the spilling trigger mechanism. When memory limits are reached, how does a stateful Operator (like an Aggregation or Hash Join) pause, serialize its state to disk, free up RAM, and later read it back?

---

## Phase 4: Communication Interfaces (Data Exchange)
**Objective:** Understand the three critical interface boundaries of a Trino Worker: Control (Coordinator), Data (Shuffle), and Storage (Connector). This defines the full API surface required for a compatible worker implementation.

### Task 4.1: The Control Plane (Coordinator ↔ Worker)
How the "Brain" manages the "Muscle." This is primarily REST/JSON based.

* **Task 4.1.A: Task Lifecycle Management**
    * **Target Files:** `io.trino.server.TaskResource`, `io.trino.execution.SqlTask`
    * **Focus:** Analyze the REST endpoints for creating, updating, and deleting tasks. Trace the `TaskUpdateRequest` JSON structure. How does the worker receive the "Physical Plan" from the coordinator?
* **Task 4.1.B: Status & Heartbeat Reporting**
    * **Target Files:** `io.trino.execution.TaskStateMachine`, `io.trino.server.TaskStatus`
    * **Focus:** How does the worker report its health, memory usage, and execution progress back to the coordinator? Trace the asynchronous long-polling mechanism used to update task status.
* **Task 4.1.C: Dynamic Filter Coordination**
    * **Target Files:** `io.trino.server.DynamicFilterService`
    * **Focus:** Trace how a worker "collects" a filter (from a join build side) and sends it back to the coordinator, and how the coordinator then "broadcasts" it to other workers to prune scans.
* **Task 4.1.D: Worker Discovery & Registration Protocol** `[KG-3]`
    * **Target Files:** `io.trino.server.ServerMainModule` (startup wiring), `io.airlift.discovery.client.DiscoveryModule`, `io.airlift.discovery.client.ServiceAnnouncement`, `io.trino.server.InternalCommunicationModule`
    * **Focus:** Trace the worker startup sequence:
      1. How does a worker register itself with the coordinator? Is it via Airlift's `DiscoveryClient` posting to a discovery service?
      2. What `ServiceAnnouncement` properties does the worker advertise? (node ID, HTTP URI, connector IDs, etc.)
      3. Is there a periodic heartbeat from the worker to the coordinator (separate from task-level status)? What happens if the coordinator doesn't see a worker heartbeat?
      4. How does the coordinator maintain its list of active workers? Trace the `NodeManager` / `InternalNodeManager` to see how the worker fleet is tracked.
      5. What configuration properties must the worker set to join a cluster? (`coordinator-url`? `discovery.uri`? `node.id`?)
* **Task 4.1.E: TaskUpdateRequest Full JSON Schema** `[KG-4]`
    * **Target Files:** `io.trino.server.TaskUpdateRequest`, `io.trino.execution.TaskInfo`, Jackson annotations, any JSON schema tests
    * **Focus:** Document every field in the `TaskUpdateRequest` JSON payload:
      1. Top-level fields: `session`, `extraCredentials`, `fragment` (the plan), `sources` (split assignments), `outputIds`, `dynamicFilterDomains`, etc.
      2. The `session` object: what fields does it carry? (`queryId`, `transactionId`, `user`, `source`, `catalog`, `schema`, `timeZone`, `language`, `systemProperties`, `catalogProperties`, etc.)
      3. The `sources` field: how are `SplitAssignment` objects structured? How do they map splits to plan node IDs?
      4. Which fields are optional vs. required? Which are sent only on the first update vs. every update?
      5. Capture a complete concrete JSON example from a real `TaskUpdateRequest`.
* **Task 4.1.F: Plan Fragment JSON Structure & Node Tagging** `[KG-6]`
    * **Target Files:** `io.trino.sql.planner.PlanFragment`, `io.trino.sql.planner.plan.PlanNode` (and all subclasses), `io.trino.server.TaskResource` (JSON serialization path), Jackson `@JsonTypeInfo` / `@JsonSubTypes` annotations on PlanNode
    * **Focus:** Trace the exact JSON structure of a serialized `PlanFragment`. Specifically:
      1. How are plan nodes tagged for polymorphic deserialization? (Is there an `@type` field? A `type` discriminator? What are the exact string values for each node type?)
      2. What is the top-level structure of `PlanFragment` JSON? (Fields: `id`, `root` plan node, `partitioning`, `partitionedSources`, `outputLayout`, etc.)
      3. Trace the recursive structure: how does a `JoinNode` reference its left/right children? How does a `TableScanNode` reference its table handle?
      4. Capture concrete JSON examples by tracing serialization of at least three plan types: (a) a simple `TableScan -> Filter -> Project -> Output`, (b) a `HashJoin` with build and probe sides, (c) an `Aggregation` with GROUP BY.
      5. Document the `PartitioningScheme` and `PartitioningHandle` JSON — these determine how output is distributed.
* **Task 4.1.G: Type System JSON Serialization** `[KG-8]`
    * **Target Files:** `io.trino.spi.type.Type` (and all implementations), `io.trino.spi.type.TypeSignature`, `io.trino.type.TypeDeserializer`, Jackson annotations on type classes
    * **Focus:** How are Trino SQL types serialized in plan fragment JSON?
      1. Document the JSON representation of each base type: `BOOLEAN`, `TINYINT`, `SMALLINT`, `INTEGER`, `BIGINT`, `REAL`, `DOUBLE`, `VARCHAR`, `VARCHAR(n)`, `CHAR(n)`, `VARBINARY`, `DATE`, `TIME`, `TIMESTAMP`, `TIMESTAMP WITH TIME ZONE`, `INTERVAL`, `DECIMAL(p,s)`, `UUID`, `JSON`.
      2. Document parameterized types: `ARRAY(element)`, `MAP(key, value)`, `ROW(name1 type1, name2 type2, ...)`.
      3. Is the serialization based on `TypeSignature` (string-based like `"varchar(256)"`) or a structured JSON object?
      4. How are type parameters encoded for `DECIMAL(p,s)` — inline in the signature string or as separate fields?
* **Task 4.1.H: TaskStatus & TaskInfo Full JSON Schema** `[KG-5]`
    * **Target Files:** `io.trino.execution.TaskStatus`, `io.trino.execution.TaskInfo`, `io.trino.operator.TaskStats`, `io.trino.operator.PipelineStats`, `io.trino.operator.OperatorStats`, Jackson annotations
    * **Focus:** Document the complete response schemas:
      1. `TaskStatus`: all fields (`taskId`, `taskInstanceId`, `version`, `state`, `self` URI, `failures`, `queuedPartitionedDrivers`, `runningPartitionedDrivers`, `outputBufferStatus`, `dynamicFiltersVersion`, memory usage fields, etc.)
      2. `TaskInfo`: all fields (superset of `TaskStatus` plus `taskStats`, `needsPlan`, etc.)
      3. `TaskStats`: all aggregate metrics fields
      4. `PipelineStats`: per-pipeline metrics
      5. `OperatorStats`: per-operator metrics (this is what the coordinator uses for query progress UI)
      6. The `OutputBufferInfo` nested object: buffer state, partition statuses, memory usage
* **Task 4.1.I: Pipeline & Operator Stats JSON Fields** `[KG-9]`
    * **Target Files:** `io.trino.operator.OperatorStats`, `io.trino.operator.DriverStats`, `io.trino.operator.PipelineStats`, `io.trino.operator.TaskStats`
    * **Focus:** Enumerate every metric field reported by the worker:
      1. For `OperatorStats`: `operatorId`, `planNodeId`, `operatorType`, `totalDrivers`, `addInputCalls`, `addInputWall`, `addInputCpu`, `inputDataSize`, `inputPositions`, `getOutputCalls`, `getOutputWall`, `getOutputCpu`, `outputDataSize`, `outputPositions`, `physicalWrittenDataSize`, `blockedWall`, `finishCalls`, `finishWall`, `finishCpu`, `userMemoryReservation`, `revocableMemoryReservation`, `peakUserMemoryReservation`, `peakRevocableMemoryReservation`, `spilledDataSize`, `connectorMetrics`, `connectorOperatorTimer`, `dynamicFilterSplitsProcessed`, etc.
      2. Which of these fields does the coordinator *require* (vs. best-effort)? Are there fields that, if missing or zero, cause the coordinator to make incorrect scheduling decisions?
      3. How are timing values represented? (ISO duration strings? Milliseconds as longs?)
      4. How are data sizes represented? (`DataSize` objects with value+unit? Raw bytes as longs?)

### Task 4.2: The Data Plane (Worker ↔ Worker Shuffle)
How data moves between workers during a query. This is high-volume HTTP streaming.

* **Task 4.2.A: Result Buffering (The Server Side)**
    * **Target Files:** `io.trino.execution.buffer.OutputBuffer`, `io.trino.execution.buffer.PagesSerde`
    * **Focus:** How are `Page` objects serialized into the wire format? Trace the logic in `PartitionedOutputBuffer` that hashes rows to decide which downstream worker gets which data.
* **Task 4.2.B: The Exchange Client (The Client Side)**
    * **Target Files:** `io.trino.operator.exchange.ExchangeClient`, `io.trino.operator.HttpPageBufferClient`
    * **Focus:** How does a worker pull data from multiple upstream workers? Trace the HTTP request cycle: headers used for flow control (`X-Trino-Buffer-Remaining-Bytes`), retries, and backoff.
* **Task 4.2.C: The Coordinator as the Final Consumer**
    * **Target Files:** `io.trino.server.StatementResource`, `io.trino.execution.DataPuller` (or how `Query` manages the final exchange)
    * **Focus:** Trace how the coordinator fetches the final result set from the root task. How does the root worker's `OutputBuffer` differ (if at all) from an intermediate shuffle buffer? How are internal `Pages` finally converted into the client-facing format?
* **Task 4.2.D: Block-Level Binary Encoding** `[KG-1]`
    * **Target Files:** `io.trino.spi.block.BlockEncodingSerde`, `io.trino.spi.block.BlockEncoding` (and all implementations: `LongArrayBlockEncoding`, `VariableWidthBlockEncoding`, `IntArrayBlockEncoding`, `ByteArrayBlockEncoding`, `Int128ArrayBlockEncoding`, `DictionaryBlockEncoding`, `RunLengthBlockEncoding`, `RowBlockEncoding`, `ArrayBlockEncoding`, `MapBlockEncoding`, etc.)
    * **Focus:** Trace the exact binary layout written by each `BlockEncoding.writeBlock()` method:
      1. For `LongArrayBlockEncoding`: how are the `positionCount`, null bitmap, and `long[]` values encoded? Byte order? Is the null bitmap packed (1 bit per position) or expanded (`boolean[]` → byte-per-position)?
      2. For `VariableWidthBlockEncoding`: how are the offsets array, the byte data slice, and the null bitmap laid out?
      3. For `DictionaryBlockEncoding`: how is the dictionary block encoded (recursive?), followed by the `int[]` ids?
      4. For `RunLengthBlockEncoding`: how is the single value block + position count encoded?
      5. For complex types (`RowBlockEncoding`, `ArrayBlockEncoding`, `MapBlockEncoding`): how is nesting handled?
      6. Document the exact byte sequence for at least two concrete examples: (a) a `LongArrayBlock` with 1024 positions and 3 nulls, (b) a `VariableWidthBlock` with mixed-length strings.
* **Task 4.2.E: Block Type Tags & Page Envelope** `[KG-2]`
    * **Target Files:** `io.trino.execution.buffer.PagesSerdeUtil`, `io.trino.spi.block.BlockEncodingSerde` (the `readBlock`/`writeBlock` envelope), `io.trino.execution.buffer.PageSerializer`, `io.trino.execution.buffer.PageDeserializer`
    * **Focus:** Trace the complete page serialization pipeline:
      1. How does `PageSerializer` wrap the 12-byte page header (`positionCount`, `uncompressedSize`, `compressedSize`) around the block data?
      2. Before each block, how is the block *type* identified? Is there a string tag (e.g., `"LONG_ARRAY"`)? A numeric enum? How is it length-prefixed?
      3. When compression is applied (LZ4/ZSTD), is it applied to the entire page payload or per-block? What metadata identifies the compression codec?
      4. When encryption is applied, what is the IV/nonce layout?
      5. Trace `PagesSerdeUtil.readPages()` / `writePages()` to understand how multiple pages are framed in a single HTTP response body (for the exchange protocol).
      6. Document the complete byte layout of a serialized HTTP response containing 2 pages with 3 columns each.
* **Task 4.2.F: Partition Hash Function for Shuffle** `[KG-11]`
    * **Target Files:** `io.trino.operator.PartitionFunction`, `io.trino.operator.InterpretedHashGenerator`, `io.trino.spi.type.TypeOperators` (hashCodeOperator), `io.trino.type.BlockTypeOperators`
    * **Focus:** Trace the exact hash function used to assign rows to output partitions:
      1. Which hash algorithm is used? (Murmur3? XxHash? Java's `hashCode()`?)
      2. How is the hash computed for each type? (e.g., BIGINT → direct long hash? VARCHAR → hash of bytes? DECIMAL → ?)
      3. How is the hash mapped to a partition number? (`hash % partitionCount`? Consistent hashing?)
      4. Is the hash function stable across Trino versions? (It must be, for mixed-version clusters.)
      5. Provide the exact hash computation for BIGINT, VARCHAR, DOUBLE, BOOLEAN, and TIMESTAMP so the Rust implementation can produce identical partition assignments.
* **Task 4.2.G: Exchange Protocol HTTP Headers & Framing** `[KG-12]`
    * **Target Files:** `io.trino.server.TaskResource.getResults()`, `io.trino.operator.HttpPageBufferClient`, `io.trino.execution.buffer.OutputBufferInfo`, HTTP header constants in Trino's codebase
    * **Focus:** Document every HTTP header used in the exchange protocol:
      1. Request headers: `X-Trino-Max-Size`, `X-Trino-Buffer-Remaining-Bytes`, `X-Trino-Task-Instance-Id`, any authentication headers.
      2. Response headers: `X-Trino-Task-Instance-Id`, `X-Trino-Page-Token`, `X-Trino-Page-Next-Token`, `X-Trino-Buffer-Complete`, `Content-Type`, any custom headers.
      3. The acknowledge/destroy endpoint: `DELETE /v1/task/{taskId}/results/{bufferId}/{token}` — what does the token mean here?
      4. How is the HTTP response body framed when it contains multiple serialized pages? (Concatenated? Length-prefixed?)
      5. What happens on error? (HTTP status codes, error response format)
      6. Trace the exact request/response sequence for a complete page fetch cycle: initial request → page data → acknowledge → next request.

### Task 4.3: The Storage Plane (Worker ↔ Connector)
How the worker interacts with the SPI (Service Provider Interface) to get raw data.

* **Task 4.3.A: The Page Source (Reading)**
    * **Target Files:** `io.trino.spi.connector.ConnectorPageSource`, `io.trino.connector.hive.HivePageSource` (as an example)
    * **Focus:** Trace the boundary where a "Split" is converted into a stream of `Page` objects. How does the worker pass column projections and predicate pushdowns into the connector?
* **Task 4.3.B: The Page Sink (Writing)**
    * **Target Files:** `io.trino.spi.connector.ConnectorPageSink`
    * **Focus:** Trace how data is sent to a connector for `INSERT` or `CREATE TABLE AS` operations. How does the worker handle transaction commits/rollbacks via the connector?
* **Task 4.3.C: Split JSON Formats (Hive & Iceberg)** `[KG-10]`
    * **Target Files:** `io.trino.plugin.hive.HiveSplit`, `io.trino.plugin.iceberg.IcebergSplit`, `io.trino.spi.connector.ConnectorSplit`, Jackson annotations, `io.trino.split.SplitJSON` or equivalent serialization wrappers
    * **Focus:** Document the exact JSON structure of split objects as delivered in `TaskUpdateRequest.sources`:
      1. `HiveSplit`: file path, start offset, length, file size, file modified time, partition keys (schema?), bucket number, table/partition format info, S3 region, etc.
      2. `IcebergSplit`: file path, start offset, length, file format (PARQUET/ORC), partition spec, residual filter expression, etc.
      3. How is the `ConnectorSplit` polymorphism handled in JSON? (Connector-specific `@type` tag? A wrapper with `connectorId` + opaque JSON blob?)
      4. How are split assignments bundled? (`SplitAssignment` → list of `ScheduledSplit` → `Split` → `ConnectorSplit`)
* **Task 4.3.D: Iceberg PageSource & Delete File Handling** `[KG-ICE-1]`
    * **Target Files:** `io.trino.plugin.iceberg.IcebergPageSource`, `io.trino.plugin.iceberg.IcebergPageSourceProvider`, `io.trino.plugin.iceberg.delete.PositionDeleteFilter`, `io.trino.plugin.iceberg.delete.EqualityDeleteFilter`, `io.trino.plugin.iceberg.IcebergParquetPageSource`
    * **Focus:** Trace the Iceberg connector's read pipeline from split to pages:
      1. How does `IcebergPageSourceProvider.createPageSource()` construct the page source from an `IcebergSplit`? How does it select the format reader (Parquet vs ORC)?
      2. How does `IcebergPageSource` wrap the underlying format-specific page source (e.g., Parquet reader)?
      3. **Delete file handling (MOR):** How are positional delete files applied? How are equality delete files applied? At what stage in the pipeline do deletes filter out rows — before or after the format reader produces pages?
      4. **Schema evolution:** When the data file was written under an older schema, how does the connector project/coerce columns to match the current query schema? Trace the `SchemaParser` and column ID-based mapping.
      5. How does `IcebergSplit.deleteFiles` flow into the page source construction?
* **Task 4.3.E: Iceberg PageSink & Commit Protocol** `[KG-ICE-2]`
    * **Target Files:** `io.trino.plugin.iceberg.IcebergPageSink`, `io.trino.plugin.iceberg.IcebergPageSinkProvider`, `io.trino.plugin.iceberg.IcebergMetadata` (focus on `finishInsert`, `finishCreateTable`), `io.trino.plugin.iceberg.IcebergWritableTableHandle`
    * **Focus:** Trace the Iceberg connector's write pipeline:
      1. How does `IcebergPageSink` write pages to Parquet files on object storage? How is the output file path determined (partition layout, file naming)?
      2. How does the partition spec control file organization? How are rows routed to partition-specific writers?
      3. What do the fragment `Slice`s returned by `finish()` contain? (File path, metrics, partition data?)
      4. How does `IcebergMetadata.finishInsert()` collect fragments from all workers and perform the atomic Iceberg commit (append snapshot)?
      5. How does the write transaction handle rollback on failure (`abort()`)?

---

## Phase 5: Memory Tracking & Arbitration (Memory Management)
**Objective:** Map out the strict hierarchical memory accounting system to understand flow control, spilling triggers, and query arbitration. This is vital for replicating safe resource management in a Rust worker.

### Task 5.1: The Global Pool and the Tracking Tree
* **Task 5.1.A: The Global Pool and the Tracking Tree**
    * **Target Files:** `io.trino.memory.MemoryPool`, `io.trino.memory.QueryContext`, `io.trino.memory.context.MemoryTrackingContext`
    * **Focus:** Analyze how the single global `MemoryPool` is constructed. Trace the creation of the context tree (Query -> Task -> Pipeline -> Driver -> Operator). How do deltas (additions/subtractions of memory) propagate up this tree without causing severe lock contention?

### Task 5.2: Operator Allocation and Blocking
* **Task 5.2.A: Operator Allocation and Blocking**
    * **Target Files:** `io.trino.memory.context.LocalMemoryContext`, `io.trino.memory.context.UserMemoryContext`
    * **Focus:** Trace the exact mechanism an Operator uses to report an allocation: `LocalMemoryContext.setBytes()`. What is the exact execution path when this call results in a limit being exceeded? Trace how the resulting `ListenableFuture` is passed back to the Driver yield loop.

### Task 5.3: Revocable Memory and Spilling
* **Task 5.3.A: Revocable Memory and Spilling**
    * **Target Files:** `io.trino.memory.context.RevocableMemoryContext`, `io.trino.execution.MemoryRevokingScheduler`
    * **Focus:** Look at `HashBuilderOperator` or `HashAggregationOperator`. Trace how they allocate memory via the `RevocableMemoryContext`. How does the `MemoryRevokingScheduler` monitor the global pool, select a victim task, and invoke the asynchronous revoking process?

### Task 5.4: Cluster Arbitration and the OOM Killer
* **Task 5.4.A: Cluster Arbitration and the OOM Killer**
    * **Target Files:** `io.trino.memory.ClusterMemoryManager`, `io.trino.memory.LowMemoryKiller`
    * **Focus:** This requires jumping to the Coordinator. How does the coordinator aggregate the `MemoryInfo` from all workers? Trace the logic in `ClusterMemoryManager` when a worker reports a "blocked" state. How does the `LowMemoryKiller` select a victim, and how is the "kill" signal propagated back down the Control Plane (Phase 4) to the workers?

---

## Phase 6: Test Infrastructure & Validation Strategy `[KG-13]`
**Objective:** Map Trino's complete test hierarchy to understand how correctness is validated at every level — from individual block encodings to full distributed queries. This is critical for determining which existing tests can validate the Rust worker as a drop-in replacement, which need adaptation, and where new cross-language test harnesses are required.

### Task 6.1: Test Hierarchy, Frameworks & CI
* **Task 6.1.A: Test Hierarchy & Organization**
    * **Target Files:** Top-level `pom.xml` (test profiles, surefire/failsafe config), `testing/trino-testing/`, `testing/trino-testing-services/`, `testing/trino-tests/`, `testing/trino-product-tests/`, `.github/` (CI workflows)
    * **Focus:** Map the complete test taxonomy:
      1. What test categories exist? (unit tests, integration tests, product tests, benchmark tests, etc.)
      2. What frameworks are used? (JUnit 5, TestNG, AssertJ, etc.)
      3. How are tests categorized and selectively run? (Maven profiles, surefire groups, `@Tag` annotations?)
      4. What is the directory structure convention? (`src/test/java/` placement, test module organization)
      5. What CI infrastructure runs these tests? (GitHub Actions workflows, test matrix)

### Task 6.2: Data Model & Wire Format Tests (aligns with Impl Phase 1)
* **Task 6.2.A: Block & Page Serialization Tests**
    * **Target Files:** `core/trino-main/src/test/java/io/trino/block/` (block encoding tests), `core/trino-main/src/test/java/io/trino/execution/buffer/` (page serialization tests), `lib/trino-orc/src/test/` (if wire format tested via ORC path)
    * **Focus:** Identify tests that validate wire-level data correctness:
      1. Are there round-trip tests for each of the 13 block encoding types (`LONG_ARRAY`, `VARIABLE_WIDTH`, `DICTIONARY`, etc.)?
      2. How are null bitmaps tested (especially the MSB-first convention)?
      3. Are there tests for the 16-byte HTTP envelope, xxHash64 checksum, 64KB chunk compression?
      4. Are there cross-version compatibility tests (serialized bytes from one version deserialized by another)?
      5. What test data generators exist for `Block` and `Page` objects? (random generators, edge-case builders)

### Task 6.3: Control Plane & Task Lifecycle Tests (aligns with Impl Phases 2-3)
* **Task 6.3.A: Task & REST Endpoint Tests**
    * **Target Files:** `core/trino-main/src/test/java/io/trino/execution/` (task tests), `core/trino-main/src/test/java/io/trino/server/` (REST endpoint tests)
    * **Focus:** Identify tests for the control plane:
      1. Task lifecycle tests (`TestSqlTask`, `TestSqlTaskManager`) — do they create real tasks with plan fragments?
      2. JSON serialization tests — are there tests that serialize/deserialize `TaskUpdateRequest`, `TaskStatus`, `PlanFragment`, and assert structure?
      3. REST endpoint tests — are `TaskResource` endpoints tested in isolation? With what test harness?
      4. Plan fragment parsing tests — are there tests that deserialize specific plan node types from JSON?
      5. Expression evaluation tests — how are expression compilation and evaluation tested?

### Task 6.4: Operator & Execution Tests (aligns with Impl Phases 4-5, 8)
* **Task 6.4.A: Operator Correctness Tests**
    * **Target Files:** `core/trino-main/src/test/java/io/trino/operator/` (operator tests), `core/trino-main/src/test/java/io/trino/sql/planner/` (planner integration tests)
    * **Focus:** Understand how individual operators are validated:
      1. How are operators tested in isolation? (synthetic input pages? mock drivers? `OperatorAssertion` utilities?)
      2. What is the `OperatorAssertion` / `TestingOperatorContext` test infrastructure?
      3. How are stateful operators tested (joins, aggregations) — do tests cover spill paths?
      4. How are edge cases tested (null handling, empty pages, single-row pages, large pages)?
      5. Are there parameterized tests that run the same operator across multiple data types?

### Task 6.5: Data Plane & Shuffle Tests (aligns with Impl Phase 6)
* **Task 6.5.A: Exchange Protocol & Partition Hash Tests**
    * **Target Files:** `core/trino-main/src/test/java/io/trino/exchange/` (exchange tests), `core/trino-main/src/test/java/io/trino/operator/output/` (output buffer tests), `core/trino-main/src/test/java/io/trino/type/` (hash function tests)
    * **Focus:** Identify tests for the data plane:
      1. Are there tests for the token-based pull protocol (sequence tokens, ACK, buffer destroy)?
      2. Are there tests for the 7 custom HTTP headers?
      3. Are there tests for the partition hash function that assert specific hash values for known inputs? (Critical for cross-language validation)
      4. Are there tests for `OutputBuffer` back-pressure behavior?
      5. What test artifacts (recorded page binaries, expected hash values) could be extracted for Rust-side property tests?

### Task 6.6: Integration & End-to-End Tests (aligns with Impl Phase 9)
* **Task 6.6.A: `DistributedQueryRunner` & Query Assertions**
    * **Target Files:** `testing/trino-testing/src/main/java/io/trino/testing/DistributedQueryRunner.java`, `testing/trino-testing/src/main/java/io/trino/testing/AbstractTestQueries.java`, `testing/trino-testing/src/main/java/io/trino/testing/QueryAssertions.java`, `testing/trino-tests/src/test/java/io/trino/tests/`
    * **Focus:** Understand how full end-to-end query tests work:
      1. How does `DistributedQueryRunner` spin up a coordinator + workers in-process? Can it use out-of-process workers?
      2. What is `AbstractTestQueries` and how do connector-specific test classes extend it?
      3. How does `QueryAssertions` validate query results? (`MaterializedResult` comparison, row ordering, type matching)
      4. How are TPC-H/TPC-DS queries used for validation? (`AbstractTestTpchQueries`, `TestDistributedQueriesTpch`)
      5. Can the Rust worker be tested by replacing the Java worker in `DistributedQueryRunner`, or must we build an external test harness?

* **Task 6.6.B: Product Tests & Cross-Compatibility**
    * **Target Files:** `testing/trino-product-tests/`, `testing/trino-product-tests-launcher/`, connector-specific test modules
    * **Focus:** Understand the outer-loop test infrastructure:
      1. What are product tests? How are they structured? (Docker-based? Tempto framework?)
      2. What connectors are covered by product tests (Hive, Iceberg, Delta, TPCH)?
      3. How could product tests be adapted to use an external Rust worker process?
      4. What test artifacts (SQL scripts, expected results, Docker compose files) could be reused?
      5. Are there any mixed-version or mixed-implementation tests already in the codebase?

