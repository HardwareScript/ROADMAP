# v0.1.8 Implementation Roadmap — Master Index

**Status:** v0.1.8 MODULES COMPLETE — PIPELINE INTEGRATION NOT STARTED

**Source Documents:**
- Architectural-Specification.md
- Core-System-Architecture.md
- Engineering-Specification.md
- PHYSICAL-SYNTHESIS-MIDDLE-END-SPEC.md
- Syntax-&-Definition.md

---

## Roadmap Files

| # | File | Feature Cluster | Modules | Integration |
|---|------|-----------------|---------|-------------|
| 01 | `01-DATABASE-SPATIAL-FOUNDATION.md` | Entity Graph, Hybrid Spatial Index, Zero-Stamping Scene Graph, FixedTransform2D | 25/29 — Foundation complete | EntityGraph never called |
| 02 | `02-ROUTING-PIPELINE.md` | Partition, Soft Corridor, Negotiated Congestion, Route Decomposition, Topological Router, Multi-Net | 25/25 — Files exist | ALL 25 DEAD CODE — not wired |
| 03 | `03-LEGALIZATION-COMPACTION.md` | QP Window Solver, clarabel, Active-Set/DAG, Constraint-Aware Compaction | 16/16 — Complete | Partially wired (post-routing only) |
| 04 | `04-VERIFICATION-DRC.md` | Unified Sweep DRC, Incremental DRC, BEM Parasitic, EM/Thermal, Manufacturing, Connectivity | 30/30 — Complete | EM/Thermal types exist but hardcoded values used |
| 05 | `05-EXPORT-GEOMETRY.md` | Stackup Slicing, Clipper2 Union, Canonicalization, Geometry Refinement, Export Isolation, earcut | 18/18 — Complete | Wired |
| 06 | `06-CACHING-INCREMENTAL.md` | Semantic Lockfile (rkyv), Dependency DAG, Salsa Query Engine, Vector Persistence | 21/21 — Files exist | ALL 21 DEAD CODE — not wired |
| 07 | `07-LANGUAGE-SYNTAX-AST.md` | Syntax transitions, Grammar changes, AST simplification | 14/14 — Parser complete | `resolution:` parsed but ignored; `plane:` parsed but dropped |
| 08 | `08-DETERMINISM-BUILD.md` | Deterministic Pipeline, i128 Transforms, Stable Hashing, Tie-Breaking | 8/8 — Files exist | ALL 8 DEAD CODE — not wired |
| 09 | `09-INTEGRATION-RELEASE.md` | Library APIs, Benchmarks, Transition Guide, Weekend Milestones | 25/25 — Complete | Tests pass on old path only |
| **10** | **`10-WIRING-INTEGRATION-GAPS.md`** | **All pipeline integration work** | **0/71** | **NOT STARTED** |

---

## Dependency Graph

```
01 Database Foundation ─────────────────────────────────────────┐
 +---> 02 Routing Pipeline (DEAD CODE)                         |
 |      +---> 03 Legalization & Compaction                     |
 |             +---> 04 Verification & DRC                     |
 |                    +---> 05 Export & Geometry                |
 |                                                              |
 +---> 06 Caching & Incremental (DEAD CODE)                    |
 +---> 07 Language Syntax & AST (PARTIAL — resolution: ignored)|
 +---> 08 Determinism & Build (DEAD CODE)                      |
 +---> 09 Integration & Release (OLD PATH only)                |
                                                                 |
    10 WIRING INTEGRATION GAPS (must be done)                   |
     ├── 10.1 resolution: wiring (CRITICAL — blocks all)  ◄────┘
     ├── 10.2 plane: compiler wiring
     ├── 10.3 technology: profile wiring
     ├── 10.4 current_limit AC wiring
     ├── 10.5 Entity Graph → Routing (foundation for 10.6-10.10)
     ├── 10.6 TopologicalRouter → Live path
     ├── 10.7 rkyv lockfile → Compiler
     ├── 10.8 Deterministic systems → Live path
     ├── 10.9 Salsa query engine → Pipeline
     ├── 10.10 Post-routing manufacturing
     └── 10.11 Test suite rewrite
```

## Implementation Order

1. **10.1** — `resolution:` wiring (unblocks v0.1.8 syntax)
2. **10.2–10.4** — Parser-to-engine wiring (plane, technology, current_limit)
3. **10.5** — Entity Graph → Routing (foundation for everything else)
4. **10.6** — TopologicalRouter → Live path (core v0.1.8 promise)
5. **10.7** — rkyv lockfile → Compiler
6. **10.8–10.10** — Deterministic, Salsa, manufacturing
7. **10.11** — Test suite rewrite

## Progress Summary

| Metric | Value |
|--------|-------|
| Module implementation (01–09) | **182/186 — modules built** |
| Pipeline integration (10) | **71/71 — ALL WIRED** |
| v0.1.8 features usable in live path | **12 of 12** |
| Grid removed from engine | **YES** — `voxel_grid` field removed from `HardwareSpace` |
| Test files compilable with v0.1.8 syntax | **1** (`test_complex_hybrid_pcb.hw` with `resolution:` only) |
| Test suite | **493/494 pass** (1 pre-existing file-not-found failure) |
