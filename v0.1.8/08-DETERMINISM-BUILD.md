# Roadmap 08 — Determinism & Build Pipeline

**Read:** Core-System-Architecture.md, Engineering-Specification.md

---

## 8.1 Deterministic Topological Sorting

- [x] **Implement topological sort on dependency graph** — `deterministic_toposort()` Kahn's algorithm with `BTreeSet<u64>` for deterministic tie-breaking (smallest ID first); `CycleError` with cycle path diagnostics; `sort_definitions()` high-level API

## 8.2 Fixed-Point Coordinate Transforms (i128)

- [x] **Ensure all coordinate transforms use pure i128 intermediate arithmetic** — `transform_point_i128()`, `transform_bbox_i128()` with i128 intermediates, SCALE_FACTOR = 10^9
- [x] **Verify overflow safety** — `verify_no_overflow()` checked_mul verification; `MAX_SAFE_COORD_NM` constant; documented: 200mm PCB coordinates reach 2e11 pm, intermediate products reach ~1.414e20 which fits i128
- [x] **Verify bit-identical results** — deterministic trig lookup table for 8 standard angles; first-order approximation for arbitrary angles

## 8.3 Stable Hash Map Iteration

- [x] **Use deterministic hash maps** — `StableHashMap<K, V>` wrapper around `FxHashMap` with `iter_deterministic()` returning entries sorted by key; `StableHashSet<K>` with same guarantee; `DETERMINISTIC_SEED` constant

## 8.4 Tie-Breaking Pathfinder Heuristics

- [x] **Implement deterministic tie-breaking** — `DeterministicCost` with full `Ord` implementation: f-cost → g-cost → direction_penalty (horizontal first) → x-coordinate → y-coordinate; `DeterministicPriorityQueue` with deterministic pop; `DirectionPriority` enum

## 8.5 Bit-Identical Serialization

- [x] **Sort export files by coordinate and Net ID before writing** — `sort_segments_deterministic()`, `sort_contours_deterministic()`, `sort_traces_deterministic()`, `sort_mesh_vertices_deterministic()`, `sort_triangles_deterministic()`; deterministic DXF/SPICE/CSV BOM export
- [x] **Verify byte-identical output** — `verify_bit_identical()`, `verify_export_deterministic()`, `content_hash()` SHA-256

---

## Summary

| Section | Completed | Remaining |
|---------|-----------|-----------|
| 8.1 Deterministic Sort | 1/1 | **Complete** |
| 8.2 i128 Transforms | 3/3 | **Complete** |
| 8.3 Stable Hash Maps | 1/1 | **Complete** |
| 8.4 Tie-Breaking | 1/1 | **Complete** |
| 8.5 Bit-Identical Export | 2/2 | **Complete** |
| **Total** | **8/8** | **All sections complete** |

**Files created:** `deterministic_sort.rs`, `stable_hash_map.rs`, `i128_transforms.rs`, `deterministic_pathfinder.rs`, `deterministic_export.rs`
