# Phase 6: Test Infrastructure & Validation Patterns Overview

DataFusion's test infrastructure serves as the primary reference for how the Rust-based Trino worker should structure its own test suite. This phase maps DataFusion's idiomatic Rust testing patterns — from inline unit tests to cross-engine SQL correctness validation.

## 1. Test Hierarchy

DataFusion uses a **six-tier test pyramid** across a 68-crate workspace:

* **Unit Tests (`#[cfg(test)]`):** Pervasive inline test modules in virtually every source file — 66+ files in `physical-plan` alone, 33 in `optimizer`. The primary mechanism for testing individual functions, expressions, and operator edge cases.

* **Integration Tests (`tests/` directories):** Eight crates have `tests/` directories. The `core` crate uses a **single-binary hub pattern** (`core_integration.rs`) that includes 15+ test module directories via `mod` declarations — this avoids the N-binary link-time explosion that would otherwise occur with one binary per test file.

* **sqllogictest (474 `.slt` files, ~145K lines):** The primary SQL regression mechanism. Covers SQL semantics, optimizer behavior, datasource formats, Spark compatibility (239 files), Postgres compatibility, and TPC-H. Each file runs in an isolated `SessionContext` in parallel. Supports three modes: Normal (validate), Postgres runner (cross-engine), and **Complete** (auto-generate expected results by running queries and writing output back into `.slt` files).

* **Fuzz Tests:** Randomized correctness for joins, sorts, aggregates, and spilling. Gated behind the `extended_tests` feature flag — run post-merge, not on every PR. Includes `force_hash_collisions` mode for worst-case hash table behavior.

* **Benchmarks:** Criterion micro-benchmarks (28 files in `core/benches/`) plus a dedicated `benchmarks/` crate with TPC-H, TPC-DS, ClickBench, H2O, and IMDB workloads. CI verifies benchmark query correctness.

* **Doc Tests:** `#[doc]` examples verified on every PR.

CI uses three workflows: `rust.yml` (every PR — unit + integration + sqllogictest + clippy + fmt), `extended.yml` (post-merge — fuzz, extended sqllogictest, SQLite compatibility), and `dev.yml` (lint). Four custom Cargo profiles (`ci`, `ci-optimized`, `release-nonlto`, `profiling`) balance compile time vs. test speed.

## 2. Key Test Utilities

DataFusion provides a rich set of test utilities that should be adopted for the Rust worker:

* **Assertion macros:** `assert_batches_eq!` (exact row order) and `assert_batches_sorted_eq!` (sorted comparison) are the core assertion primitives. Both operate on pretty-printed `RecordBatch` table output, making test failures readable.

* **Data construction:** `record_batch!` macro for quick typed construction. `TestMemoryExec` as the universal mock data source with configurable partitions, projections, and sort info. `build_table_i32` and `make_partition` for common patterns.

* **Specialized mocks:** `MockExec` (error injection), `BlockingExec` (cancellation testing), `BarrierExec` (synchronized concurrent execution start). `assert_is_pending` + `assert_strong_count_converges_to_zero` for verifying async cancellation and cleanup.

* **Snapshot testing:** `insta` used in `sql/`, `substrait/`, `physical-plan/`, and `pruning/` for plan representation testing. Auto-updatable with `cargo insta review`.

* **Parametrized testing:** `rstest` with `#[template]` / `#[apply]` patterns to test operators across combinations of batch size, join variant, and hash strategy (e.g., hash join tests run across 10 combinations per test case).

## 3. Serialization Tests

Serialization testing in `datafusion-proto` follows exhaustive roundtrip testing at five layers: primitive types, ScalarValues, expressions, plans, and full SQL queries. Three test modules under `datafusion/proto/tests/cases/` contain ~150 test functions.

* **Plan roundtrips use Debug-format comparison** (not `PartialEq`) — pragmatic but lossy. Explicitly acknowledged in code comments as a known limitation.

* **Arrow IPC is embedded within protobuf** for nested ScalarValue types (List, Struct, Map, Dictionary), meaning every nested value roundtrip implicitly tests Arrow IPC serialization.

* **Self-validating serialization:** `Expr::to_bytes()` immediately attempts deserialization to catch prost recursion issues.

* **All 22 TPC-H queries** are run through physical plan serialization as integration tests.

* **Gaps:** No property-based or fuzz testing for serialization. No corpus of serialized plans from older versions for backward compatibility. No concurrent serialization stress tests.

## 4. Operator & Expression Tests

Operators are tested by feeding synthetic `RecordBatch` data through `TestMemoryExec`, executing via `execute(partition, task_ctx)` + `common::collect(stream)`, and asserting with `assert_batches_eq!` or `insta::assert_snapshot!`.

* **Spill testing is integrated directly into each operator's test module** using `RuntimeEnvBuilder::with_memory_limit()` and `DiskManagerBuilder`. Tests verify both correct results AND spill metrics (count, bytes, rows). Sort-merge join tests alone span 4,000+ lines.

* **Cross-type coverage** uses macros (`test_coercion!`, `generic_test_cast!`) and data-driven test case structures that run identical logic across 10+ types (all integer, float, string, binary, date, timestamp, interval, decimal types).

* **Expression tests** construct a `RecordBatch`, call `evaluate(&batch)`, and check result array elements. The `InListPrimitiveTestCase` pattern and similar structures provide declarative test case tables.

## 5. SQL-Level Correctness

sqllogictest is the backbone of DataFusion's SQL regression testing. Key patterns:

* **Postgres as reference oracle:** Six `pg_compat_*` files use `onlyif`/`skipif` directives to share identical expected results across DataFusion and Postgres (via Testcontainers), validating behavioral equivalence.

* **Auto-completion:** `--complete` mode runs queries, captures output, and writes expected results back into `.slt` files. This eliminates manual expected-result maintenance.

* **Output normalization** is the core challenge: both engines must produce identical string representations for NULL, NaN (platform-agnostic), floats (via BigDecimal), empty strings, and timestamps. The `conversion.rs` module handles this.

* **TPC-H dual purpose:** The `tpch/` directory splits into `plans/` (EXPLAIN output regression via insta) and `answers/` (result correctness), with queries run twice under both hash join and sort-merge join strategies.

## Summary: Patterns for the Rust Worker

1. **Adopt sqllogictest early** with a two-engine adapter: the Java Trino coordinator as the reference oracle (replacing Postgres), and the Rust worker as the target. Use `--complete` against Java Trino to seed expected results.
2. **Create a `trw-test-utils` crate from day one** with `RecordBatch`/`Page` builders (port of `RowPagesBuilder`), random data generators, TPC-H schemas, and assertion macros (`assert_batches_eq!`).
3. **Use `insta` for snapshot testing** of plan display, serialized representations, and error messages.
4. **Use `rstest` for parameterized operator tests** across batch sizes, join types, spill on/off, and data types.
5. **Integrate spill testing directly into each operator** using configurable memory limits and in-memory spillers, following DataFusion's pattern rather than separating spill tests into a separate suite.
6. **Add property-based testing (proptest)** for serialization round-trips, improving on DataFusion's gap. Generate random types, expressions, and plans; verify round-trip fidelity.
7. **Use feature-flag-gated test tiers:** fast unit tests block PRs; `integration_tests` (requires Java coordinator) and `extended_tests` (fuzz, large-scale) run post-merge.
8. **Use the single integration test binary pattern** to avoid link-time explosion as the crate count grows.
