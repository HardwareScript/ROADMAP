# v0.1.8 Implementation Roadmap — Master Index

**Status:** Feature Freeze (v0.1.8-alpha)

**Source Documents:**
- Architectural-Specification.md
- Core-System-Architecture.md
- Engineering-Specification.md
- PHYSICAL-SYNTHESIS-MIDDLE-END-SPEC.md
- Syntax-&-Definition.md

---

## Roadmap Files

| # | File | Feature Cluster |
|---|------|-----------------|
| 01 | `01-DATABASE-SPATIAL-FOUNDATION.md` | Entity Graph, Hybrid Spatial Index, Zero-Stamping Scene Graph, FixedTransform2D |
| 02 | `02-ROUTING-PIPELINE.md` | Partition, Soft Corridor, Negotiated Congestion, Route Decomposition, Topological Router, Multi-Net |
| 03 | `03-LEGALIZATION-COMPACTION.md` | QP Window Solver, clarabel, Active-Set/DAG, Constraint-Aware Compaction |
| 04 | `04-VERIFICATION-DRC.md` | Unified Sweep DRC, Incremental DRC, BEM Parasitic, EM/Thermal, Manufacturing, Connectivity |
| 05 | `05-EXPORT-GEOMETRY.md` | Stackup Slicing, Clipper2 Union, Canonicalization, Geometry Refinement, Export Isolation, earcut |
| 06 | `06-CACHING-INCREMENTAL.md` | Semantic Lockfile (rkyv), Dependency DAG, Salsa Query Engine, Vector Persistence |
| 07 | `07-LANGUAGE-SYNTAX-AST.md` | Syntax transitions, Grammar changes, AST simplification |
| 08 | `08-DETERMINISM-BUILD.md` | Deterministic Pipeline, i128 Transforms, Stable Hashing, Tie-Breaking |
| 09 | `09-INTEGRATION-RELEASE.md` | Library APIs, Benchmarks, Transition Guide, Weekend Milestones |

---

## Dependency Graph

```
01 Database Foundation
 +---> 02 Routing Pipeline
 |      +---> 03 Legalization & Compaction
 |             +---> 04 Verification & DRC
 |                    +---> 05 Export & Geometry
 |
 +---> 06 Caching & Incremental (also depends on 02)
 +---> 07 Language Syntax & AST (also depends on 01)
 +---> 08 Determinism & Build (depends on 01, 02)
 +---> 09 Integration & Release (depends on all above)
```

## Recommended Implementation Order

1. **01** — Database & Spatial Foundation (Weekend 1)
2. **07** — Language Syntax & AST (can run parallel with 01)
3. **02** — Routing Pipeline (Weekend 2)
4. **03** — Legalization & Compaction (Weekend 3)
5. **05** — Export & Geometry (Weekend 4, partially parallel with 06)
6. **06** — Caching & Incremental (Weekend 5)
7. **04** — Verification & DRC (Weekend 6)
8. **08** — Determinism & Build (throughout, validated in Weekend 7)
9. **09** — Integration & Release (Weekend 7)
