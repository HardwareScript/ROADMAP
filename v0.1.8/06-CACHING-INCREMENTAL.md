# Roadmap 06 — Caching & Incremental Compilation

**Read:** Core-System-Architecture.md, Architectural-Specification.md, Engineering-Specification.md

---

## 6.1 Semantic Lockfile System (rkyv Binary)

- [ ] **Define `CompactLockfileBinary` struct** — `version`, `board`, `placement_hash`, `arcs`, `instances`
- [ ] **Implement semantic fingerprint calculation** — `hash(Component Bounds + Rules + Stackup Layers + Router Version)`
- [ ] **Implement lockfile write** — serialize to `project.routes.lock` using `rkyv` with `AlignedVec` (16-byte alignment)
- [ ] **Implement zero-copy memory-mapped load** — `memmap2` + `rkyv` deserialization in microseconds; zero parsing overhead
- [ ] **Implement lockfile invalidation** — detect hash mismatch; discard lock and re-run pathfinder
- [ ] **Implement `hwc lock inspect` CLI tool** — decode binary to human-readable JSON on demand; no secondary JSON file generated during builds
- [ ] **Verify sub-millisecond lockfile loads** on SoC-scale designs (100M transistors)

## 6.2 Incremental Dependency DAG

- [ ] **Implement dependency mapping** — every route segment registers dependencies on connected components, pins, and spatial corridors
- [ ] **Implement granular invalidation** — on component move: traverse DAG upward, identify directly connected nets and intersecting G-cells, mark dirty
- [ ] **Implement incremental re-route** — only dirty, unlocked nets undergo pathfinding; rest of board remains locked

## 6.3 Vector Route Persistence (Base-36 RLC)

- [ ] **Implement Case-Boundary Base-36 RLC format** — direction commands uppercase (R, L, U, D), magnitude lowercase alphanumeric (0-9, a-z)
- [ ] **Implement symmetrical winding** — convert traces to direction-magnitude vectors
- [ ] **Implement zero-delimiter compression** — no spaces or commas; parser splits on-the-fly
- [ ] **Verify compression ratio** — 50-line coordinate list to ~10-character string (e.g., "RkU46Dk")

## 6.4 Salsa-Style Memoized Query Engine

- [ ] **Integrate `salsa` crate** — demand-driven incremental computation framework
- [ ] **Lower AST into fine-grained entity inputs** — `component_placement_input(file_id, component_id)`, `route_statement_input(file_id, route_id)`, `stackup_input(file_id)`
- [ ] **Wrap compiler phases in `#[salsa::query]` functions:**
  - `parse_ast(file_id)` returns `Arc<AstSpace>`
  - `resolve_symbols(file_id)` returns `Arc<SymbolTable>`
  - `partition_gcells(file_id)` returns `Arc<GCellLayout>`
  - `route_gcell(file_id, gcell_id)` returns `Arc<LocalRoute>`
  - `verify_gcell(file_id, gcell_id)` returns `Arc<DrcReport>`
- [ ] **Wire incremental invalidation** — only affected query nodes re-evaluated on entity modification
- [ ] **Verify boundary port relocation invalidation** — only 2 adjacent G-cells invalidated; all others cached
- [ ] **Verify cyclic dependencies impossible** — queries are pure functions with no side effects
- [ ] **Verify sub-10 ms incremental compile** for minor edits
