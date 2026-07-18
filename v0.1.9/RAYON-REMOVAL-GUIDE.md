# Rayon Removal Guide - Zero-Dependency Parallel Routing

**Status:** Implementation Complete  
**Goal:** Remove Rayon entirely, replace with `std::thread::scope`  
**Benefits:** Zero dependency overhead, deterministic threading, perfect cache locality  
**Priority:** HIGH - Eliminates global thread pool contention

---

## Implementation Status

All 5 Rayon usage locations have been migrated to `std::thread::scope`. Rayon has
been fully removed from the `hwc` workspace (zero references to `rayon`,
`par_iter`, or `into_par_iter`). The only remaining `rayon` dependency is
unrelated, in `hsm/src-tauri/Cargo.toml`.

### Locations Migrated
- [x] **Location 1** `hwc-engine/src/geometry_router/gcell_sweep.rs` — G-cell DRC sweeps via `std::thread::scope` with `available_parallelism` core-based chunking
- [x] **Location 2** `hwc-engine/src/geometry_router/parallel_router.rs` — Domain-based routing, one thread per domain
- [x] **Location 3** `hwc-engine/src/geometry_router/router/core.rs` — Legacy A* migrated to `std::thread::scope` (rather than deleted)
- [x] **Location 4** `hwc-engine/src/geometry_router/thermal.rs` — Current density validation with `Result<_, String>` error propagation
- [x] **Location 5** `hwc-physics/src/lib.rs` — Physics analysis, one thread per analyzer

### Dependency Removal
- [x] Remove Rayon from crate Cargo.toml files (`hwc-engine`, `hwc-physics`, `hwc-export`)
- [x] Remove Rayon from workspace Cargo.toml
- [x] Remove all Rayon import statements
- [x] `cargo tree | grep rayon` returns empty (for `hwc` workspace)

### Success Criteria
- [x] Zero Rayon references in `hwc` codebase
- [x] `cargo tree | grep rayon` returns empty (hwc workspace)
- [ ] All tests passing
- [ ] Compilation times ≤ previous baseline
- [ ] Binary size reduced by 5-10%
- [x] Deterministic parallel execution
- [x] No unsafe code introduced

---



---

**Document Version:** 1.1  
**Last Updated:** 2026-07-16  
**Owner:** Architecture Team  
**Status:** Implementation Complete
