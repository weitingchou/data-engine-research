# Phase 6: Test Infrastructure & Validation Strategy Overview

## Table of Contents
- [1. Test Hierarchy](#1-test-hierarchy)
- [2. Wire Format Tests](#2-wire-format-tests)
- [3. Control Plane Tests](#3-control-plane-tests)
- [4. Operator Tests](#4-operator-tests)
- [5. Exchange Protocol Tests](#5-exchange-protocol-tests)
- [6. Integration & End-to-End Tests](#6-integration--end-to-end-tests)
- [Summary: Implications for the Rust Worker](#summary-implications-for-the-rust-worker)

To build a drop-in replacement worker, we must not only reimplement Trino's runtime behavior — we must prove correctness at every layer. This phase maps Trino's existing test infrastructure to understand what tests exist, what gaps remain, and how the Rust worker's testing strategy should be structured.

## 1. Test Hierarchy

Trino uses **four test tiers**, none categorized by annotations — all selective execution is via Maven profiles with surefire include/exclude patterns keyed on file paths:

* **Unit Tests (JUnit 5 + AssertJ):** The bulk of the test suite. Operators, block encodings, expressions, and serialization are all tested here. JUnit 5 parallel execution is enabled globally at both method and class levels. Tests use `@TestInstance(PER_CLASS)` with `@Execution(CONCURRENT)` to share expensive fixtures (like `DistributedQueryRunner`) while running methods concurrently.

* **Integration Tests (JUnit 5 + Testcontainers + `DistributedQueryRunner`):** Connector-specific query tests. `BaseConnectorTest` uses a `TestingConnectorBehavior` enum to dynamically skip tests based on connector capabilities — connectors declare what they support, and ~50 test methods skip themselves for unsupported operations.

* **Product Tests (Tempto/TestNG + Testcontainers):** Docker-based end-to-end tests via `trino-product-tests-launcher`. All containers are created programmatically (not via docker-compose). **Actively being deprecated** in favor of JUnit-based integration tests, but still covers ~38 suites including TPC-H Q1-Q22, TPC-DS, and connector-specific SQL conventions.

* **Benchmarks (JMH + Benchto):** Micro-benchmarks for operators and block operations (JMH), macro-benchmarks for cluster-level performance (Benchto client).

CI runs a single workflow (`ci.yml`, 1134 lines) with 12 jobs and a dynamically built test matrix (~50 entries). Git-flow Incremental Builder (GIB) detects changed modules to run only impacted tests on PRs. The CI even has a self-test (`TestCiWorkflow.java`) that validates the YAML structure.

## 2. Wire Format Tests

All 13 block encoding types have round-trip coverage. 10 of 13 have dedicated SPI-level tests via `BaseBlockEncodingTest`, which tests sizes from 2 to 1,000,000 with three null-fill patterns (none, alternating, all-null). `ARRAY`, `MAP`, and `RUN_LENGTH` are tested indirectly through `AbstractTestBlock.copyBlockViaBlockSerde()`.

* **Null bitmap testing** is thorough: `TestEncoderUtil` validates MSB-first bit-packing for lengths 0 to 8192, offsets of 0/2/256, and four null patterns. Each fixed-width encoding additionally tests null-compaction (writing only non-null values without gaps).

* **HTTP envelope** (16-byte: magic `0xfea4f001` + xxHash64 checksum + pageCount) is tested through `MockExchangeRequestProcessor` and `TestingExchangeHttpClientHandler`. Data corruption testing flips byte 42 and verifies both ABORT and RETRY behavior.

* **Critical gap: No byte-level golden fixtures.** All tests verify semantic equality (deserialize, compare values), not exact bytes. No cross-version compatibility tests exist — no golden files, no version-tagged test data.

* **Test data generators:** `BlockAssertions` (seed 633969769) is the primary factory for all types including nested types and dictionary/RLE wrappers. `BenchmarkDataGenerator` provides simpler generators for benchmarks.

## 3. Control Plane Tests

Task lifecycle tests (`TestSqlTask`, `BaseTestSqlTaskManager`, `TestSqlTaskExecution`) create **real tasks with actual plan fragments** — a `TableScanNode` with `SOURCE_DISTRIBUTION` wired through a real `LocalExecutionPlanner`. These are not mocked. `BaseTestSqlTaskManager` is tested with both `TimeSharingTaskExecutor` and `ThreadPerDriverTaskExecutor` via subclasses.

* **JSON serialization** follows roundtrip patterns (`fromJson(toJson(obj)) == obj`). Stats types, symbols (including the `"<len>|<type>|<name>"` key format), plan nodes, and row patterns have dedicated roundtrip tests. However, there are no JSON snapshot/golden-file tests — the schema is implicitly defined by Jackson annotations.

* **REST endpoint testing validates the client, not the server.** `TestHttpRemoteTask` stands up a fake `TestingTaskResource` and validates that `HttpRemoteTask` correctly sends `TaskUpdateRequest`, polls status, handles failures (`TASK_MISMATCH`, `REJECTED_EXECUTION`), fetches dynamic filters lazily, and adapts split batch sizes. The real `TaskResource` at `/v1/task` is never tested in isolation.

* **Expression evaluation** spans three levels: IR optimizer rule tests (pure transforms on expression trees), `ExpressionInterpreter` (constant folding), and `TestExpressionCompiler` (end-to-end SQL evaluation through bytecode compilation via `QueryAssertions`).

## 4. Operator Tests

The operator test infrastructure centers on a few key utilities:

* **`OperatorAssertion.toPages()`** — Replaces the real `Driver` loop for unit tests. Implements the `needsInput() → addInput() → getOutput()` protocol with automatic memory revocation and a 1,000-iteration safety limit. The `revokeMemory` parameter makes spill testing automatic.

* **`RowPagesBuilder`** — Fluent API for building synthetic input pages with `.row()`, `.addSequencePage()`, `.pageBreak()`. Tests also construct raw `Page` objects directly for edge cases (RLE blocks, dirty null bytes, oversized values).

* **`TestingTaskContext` / `TestingOperatorContext`** — Lightweight context factories that construct the full `TaskContext → PipelineContext → DriverContext → OperatorContext` stack with in-memory memory pools and configurable memory limits.

* **`DummySpillerFactory`** — In-memory spiller that stores pages in lists and counts spill operations. Every stateful operator test systematically tests a matrix of `(spillEnabled, revokeMemoryWhenAddingPages, memoryLimit)` combinations.

* **`AbstractTestAggregationFunction`** — Abstract base providing 7 template tests (empty, single, multi, all-null, mixed-null, negative values, sliding window) that every aggregation function inherits. There are ~133 aggregation test files.

## 5. Exchange Protocol Tests

* **Token-based pull protocol** is thoroughly tested in `TestHttpPageBufferClient` (full lifecycle: fetch, ACK via sequenceId advance, buffer-complete signal, DELETE destroy) and `TestDirectExchangeClient` (multi-location, streaming, deduplication). `MockExchangeRequestProcessor` serves as the authoritative reference for the binary wire protocol.

* **HTTP headers:** All 7 `X-Trino-*` headers are defined in `InternalHeaders.java`. Six are set/validated in mock processors, but tested implicitly through end-to-end flows rather than in isolation.

* **Critical gap: No golden hash values.** All hash tests (`TestInterpretedHashGenerator`, `TestFlatHashStrategy`) compare two Java implementations against each other. No hardcoded expected values exist anywhere. A Java harness must be written to generate golden test vectors for the Rust port.

* **Back-pressure** is extensively tested in `TestPartitionedOutputBuffer`: `enqueuePage()` returns a `ListenableFuture` that blocks when the buffer is full, unblocking on consumer ACK, `setNoMorePages()`, or `destroy()`.

## 6. Integration & End-to-End Tests

**`DistributedQueryRunner`** creates all nodes (coordinator + workers) in-process as `TestingTrinoServer` instances within a single JVM. The test hierarchy is `AbstractTestQueryFramework` → `AbstractTestQueries` → `BaseConnectorTest` → connector-specific tests. `QueryAssertions` compares Trino results against an **H2 in-memory database** pre-loaded with TPC-H data as the correctness oracle.

* **The worker registration protocol is language-agnostic:** workers `POST /v1/announce` with their URI, serve `GET /v1/info` with matching version and environment. The Rust worker can register with a real Java coordinator.

* **`DistributedQueryRunner` cannot host external workers** (no `addExternalWorker(URI)` method), but a standalone coordinator with `workerCount=0` + `node-scheduler.include-coordinator=false` forces all execution to the externally registered Rust worker.

* **Product tests** use Testcontainers (not docker-compose) with the Tempto framework. The `EnvSinglenodeCompatibility` environment already runs two different Trino versions side-by-side — a direct template for mixed Java coordinator + Rust worker testing. All tests go through JDBC to the coordinator; no test ever contacts a worker directly.

## Summary: Implications for the Rust Worker

1. **Wire format correctness requires golden file suites.** Trino tests verify semantic equality, not byte-level identity. The Rust worker must generate golden fixtures from the Java implementation: serialized blocks (all 13 types), hash values for known inputs, JSON wire types, and serialized pages.
2. **Operator-level testing is well-patterned.** The `OperatorAssertion` → `toPages()` driver loop, `RowPagesBuilder`, `DummySpillerFactory`, and systematic spill matrix patterns should be ported directly to Rust.
3. **End-to-end testing uses the HTTP announce protocol.** A real Java coordinator can be started (via Docker or `TpchQueryRunner.main()`), and the Rust worker registers via `POST /v1/announce`. TPC-H Q1-Q22 provides the first integration test suite.
4. **Product tests can be reused.** A new `EnvMultinodeRustWorker` Docker environment can inject the Rust worker while reusing all existing Tempto SQL tests, expected result files, and coordinator configuration unchanged.
5. **The biggest gap is cross-language conformance.** No tests in Trino validate that a non-Java implementation produces correct wire output. This is entirely new testing infrastructure the Rust project must build.
