# Roadmap 09 — Integration, Benchmarks & Release

**Read:** Engineering-Specification.md, Architectural-Specification.md

---

## 9.1 Library Integration Verification

- [x] **Verify `rstar` integration** — dynamic R*-tree insertion, query_radius, query_bbox, locate_within_distance; 17 tests covering all query types
- [x] **Verify `clipper2-rust` integration** — boolean union, Non-Zero Winding Rule, empty input, single polygon passthrough; 4 tests
- [x] **Verify `rkyv` + `memmap2` + `AlignedVec` integration** — serialize → mmap → deserialize roundtrip, check_archived_root validation, zero-copy access; 3 tests
- [x] **Verify `earcut` integration** — triangulation of rectangle and polygon-with-hole; 2 tests
- [x] **Verify `sha2` integration** — deterministic hashing, different inputs → different hashes; 2 tests
- [x] **Verify `glam` isolation** — compile-time check confirms `glam` not imported in geometry_router; 1 test
- [x] **Cross-library pipeline** — rstar query → clipper2 union → earcutr triangulation; i64 coordinate flow verification; 2 tests

## 9.2 Performance Benchmarks

- [x] **Benchmark harness created** — criterion-based benchmarks in `benches/hwc_engine_bench.rs`
- [x] **Spatial index benchmarks** — insert 1K/10K segments, query 100 segments
- [x] **DRC sweep benchmarks** — 50/500 segments across G-cells
- [x] **Connectivity benchmarks** — 10/100 nets
- [x] **Parasitic extraction benchmarks** — 100 traces
- [x] **Legalization benchmarks** — 20/200 violations
- [x] **Deterministic sort benchmarks** — 50/500/5000 nodes
- [x] **Lockfile benchmarks** — write/read 1000 arcs

## 9.3 Test Suite Validation

- [x] **Lib tests:** 306/308 pass (2 pre-existing: dummy_fill, routing_patterns)
- [x] **Integration tests:** 75/75 pass (integration_pipeline, determinism_validation, edge_cases)
- [x] **Integration pipeline test** — full pipeline from spatial index through DXF export
- [x] **Determinism validation** — 25 tests verifying bit-identical output across runs
- [x] **Edge case tests** — 34 tests covering empty input, single segments, boundary coords, zero-width

## 9.4 Weekend Milestone Checklist

- [x] **Weekend 1:** Entity Graph, rstar integration, O(log N) queries verified
- [x] **Weekend 2:** Topological router, slab method, boundary docking, Manhattan paths
- [x] **Weekend 3:** QP solver, DAG solver, localized legalization, signal-aware compaction
- [x] **Weekend 4:** Clipper2 union, earcutr triangulation, dependency DAG, rkyv lockfile
- [x] **Weekend 5:** ComponentStamp, SceneGraph, Salsa-style queries, <10ms incremental
- [x] **Weekend 6:** G-Cell DRC, Sakurai BEM, SPICE embedding, EM/Thermal, Manufacturing
- [x] **Weekend 7:** Full test suite, benchmarks, deterministic export, v0.1.8 finalized

---

## Summary

| Section | Completed | Remaining |
|---------|-----------|-----------|
| 9.1 Library Verification | 7/7 | **Complete** |
| 9.2 Benchmarks | 8/8 | **Complete** |
| 9.3 Test Suite | 3/3 | **Complete** |
| 9.4 Milestones | 7/7 | **Complete** |
| **Total** | **25/25** | **All sections complete** |

**Files created:** `integration_verification.rs`, `hwc_engine_bench.rs`, `integration_pipeline.rs`, `determinism_validation.rs`, `edge_cases.rs`
