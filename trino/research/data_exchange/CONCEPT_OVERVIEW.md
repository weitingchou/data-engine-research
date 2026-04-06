# Phase 4: Communication & Interface Overview

To build or rewrite a Trino worker, one must view it as a hub connecting three distinct planes of communication. Each plane uses different protocols and has different performance characteristics.

## 1. The Control Plane (Coordinator <-> Worker)
This is the **Management Interface**. It is characterized by low-frequency, JSON-based REST communication.

* **Instruction Receipt:** The worker acts as a REST server. The coordinator POSTs a `TaskUpdateRequest` to the worker. The plan and splits arrive **separately**: an initial POST delivers the plan fragment, and subsequent updates add splits as they become available. The worker's `SqlTaskManager` uses `needsPlan` optimization to skip re-sending the plan on follow-up updates.
* **The Status Loop:** Trino does not use a persistent "push" connection for status. Instead, the coordinator uses **Long Polling** with version-based change detection. It asks the worker, "What is the status of Task X since version N?" and the worker holds the request open until either the status version advances or a timeout occurs. The wait includes a randomized component to prevent thundering-herd on restart.
* **Dynamic Filter Coordination:** This is the most sophisticated part of the control plane. The lifecycle has **two tiers**:
  - **Task-local (intra-stage):** When build and probe sides of a join run in the same task, `LocalDynamicFilterConsumer` delivers the domain directly through in-memory futures -- no coordinator round-trip needed.
  - **Coordinator-mediated (inter-stage):** When build and probe run on different workers, the domain travels through `DynamicFiltersCollector` on the build-side worker, is fetched by `DynamicFiltersFetcher` via REST polling (triggered by version bumps in `TaskStatus`), aggregated in `DynamicFilterService` on the coordinator, and pushed to scan-side workers via `TaskUpdateRequest`.

  The aggregation uses **three-tier degradation**: first it tries to collect all distinct values as a `Domain`. If the domain exceeds the size limit, it falls back to a **min/max range**. If even that is too large or the row count exceeds the threshold, it collapses to `Domain.all()` (no filtering). This prevents a single high-cardinality join key from consuming unbounded memory on the coordinator.

## 2. The Data Plane (Worker <-> Worker / Shuffle)
This is the **High-Performance Interface**. It is the "Internal Network" of the cluster where the majority of data movement happens.

* **The Pull Model:** Trino shuffles are **pull-based**. An "Upstream" worker (producing data) stores its results in an `OutputBuffer`. For hash-partitioned shuffles, a `PartitionedOutputBuffer` knows exactly how many downstream partitions exist and maintains a dedicated `ClientBuffer` per partition. The downstream workers are responsible for reaching out via HTTP GET to pull results from their assigned partition.
* **Wire Format:** Data is serialized through `CompressingEncryptingPageSerializer` into a binary format: each serialized page has a **12-byte header** (`[positionCount:i32][uncompressedSize:i32][compressedSize:i32]`), followed by block data. Compression (LZ4 or ZSTD) and optional AES-CBC encryption are applied **per-block**, not per-page, preserving the columnar structure from Phase 3.
* **Token-Based Exchange Protocol:** Each downstream consumer tracks a **sequence token** per upstream source. An HTTP GET to `/v1/task/{taskId}/results/{bufferId}/{token}` returns a `BufferResult` containing the pages starting at that token plus a `nextToken` for the follow-up request. This makes the protocol **idempotent**: replaying a request with the same token returns the same pages. An **eager acknowledgment** request is sent asynchronously after receiving pages, allowing the upstream `ClientBuffer` to free memory immediately rather than waiting for the next poll.
* **Flow Control:** Back-pressure is **client-side capacity gating**, not HTTP header signaling. `DirectExchangeClient` tracks the local `StreamingDirectExchangeBuffer` capacity and only dispatches new HTTP requests when `remainingCapacity > 0`, multiplied by a configurable concurrency factor (default 3x). On the producer side, `OutputBufferMemoryManager` blocks drivers via a `SettableFuture` when the buffer exceeds its memory limit.
* **The Coordinator as Final Consumer:** The root stage's `OutputBuffer` is identical to any intermediate shuffle buffer -- there is no special "final result buffer." The difference is entirely on the consumer side: the coordinator's `Query` object uses the same `DirectExchangeClient` / `HttpPageBufferClient` machinery to pull serialized pages. It then deserializes them and converts Block values to JSON via per-type `TypeEncoder` implementations (e.g., `BigintEncoder`, `VarcharEncoder`, `ArrayEncoder`). The client-facing protocol has two phases: `QueuedStatementResource` (`/v1/statement`) handles submission/queuing, and `ExecutingStatementResource` (`/v1/statement/executing`) streams paginated `QueryResults` JSON with `nextUri`-based pagination.

## 3. The Storage Plane (Worker <-> Connector)
This is the **External Interface**. It defines how Trino interacts with the outside world (Iceberg, Delta, Postgres, S3).

* **The SPI Boundary:** The worker does not talk to S3 directly; it talks to a **Connector**. The interface is defined by the **Service Provider Interface (SPI)**.
* **Splits to Pages (Read Path):** The Connector is given a `ConnectorSplit` (an opaque handle representing a file or a range of rows). The Connector's job is to provide a `ConnectorPageSource` that turns that split into a stream of `SourcePage` objects.
* **Three Predicate Pushdown Channels:** The worker pushes predicates into the storage plane through three distinct mechanisms:
  1. **ConnectorTableHandle:** Static predicates baked into the table handle during planning. The connector can use these to skip partitions, files, or row groups at the physical level (e.g., Parquet metadata pruning).
  2. **PageProcessor:** The engine's compiled filter/projection expressions applied by `ScanFilterAndProjectOperator` after pages are read. This catches predicates that the connector cannot evaluate.
  3. **DynamicFilter:** Runtime predicates that narrow over query lifetime. Passed as a live object to `ConnectorPageSourceProvider.createPageSource()`, allowing the connector to re-check `getCurrentPredicate()` before each page read and skip newly-prunable data.
* **Pages to Storage (Write Path):** For `INSERT` and `CREATE TABLE AS` operations, a `TableWriterOperator` pushes pages into a `ConnectorPageSink` via `appendPage()`. When the upstream pipeline is exhausted, `finish()` returns opaque **fragment** descriptors (serialized as `Slice`). These fragments flow upward through an Exchange to the coordinator, where a `TableFinishOperator` collects all fragments and calls `ConnectorMetadata.finishCreateTable()` / `finishInsert()` to **atomically commit** the entire write. This two-phase commit design ensures no metadata changes are visible until the coordinator's final commit succeeds -- providing all-or-nothing semantics. The `abort()` path deletes partially written data on failure.

## Summary: The Hub Architecture

1.  **Coordinator** sends a **TaskUpdateRequest** (Plan, then Splits incrementally) via **REST/JSON**.
2.  **Worker** starts **Drivers** to process **Splits** via the **Connector SPI**.
3.  **Connector** streams raw data as **SourcePages** into the Worker, with predicates pushed through three channels.
4.  **Worker** transforms data, serializes into **12-byte-header binary pages**, and places them in an **Output Buffer**.
5.  **Downstream Workers** (or Coordinator) fetch data via **token-based HTTP pull**, with client-side capacity gating.
6.  **Worker** reports **Status** with version-based long-polling; **Dynamic Filters** flow through a two-tier coordination path.
7.  For writes, **fragment descriptors** flow upward to the Coordinator for **atomic two-phase commit** via the Connector SPI.
