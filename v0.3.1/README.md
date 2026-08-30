# HardwareScript v0.3.1: Master Implementation Roadmap & Dependency Tree

**Target Version:** v0.3.1 (Production-Locked Standard)  
**Status:** Phase 4 Complete — In Progress (Phase 5 next)  
**Base Directory:** `ROADMAP/v0.3.1/`  
**Reference Specifications:** [Docs/v0.3.1/](file:///c:/Users/olowo/Downloads/Code/hw/Docs/v0.3.1/)

---

## 1. Critical Implementation Directives & Code Quality Mandate

> [!IMPORTANT]
> **PRE-RELEASE QUALITY STANDARD — ZERO TECHNICAL DEBT POLICY**  
> All phases must adhere strictly to these non-negotiable engineering directives during implementation:

1. **Zero Hardcoding & Zero Fallbacks:**
   - **No magic numbers or heuristic fallbacks.** Every physical dimension, dielectric constant ($\varepsilon_r$), track pitch, and standard-cell site dimension must be queried directly from canonical structs (e.g., `StackupManager`, PDK definitions, Liberty catalogs).
   - **No silent fallbacks.** If a query, lookup, or constraint fails, throw an explicit, strongly typed diagnostic error immediately (`miette`).

2. **Strong Typing & Single-Source Structs:**
   - Use the designated strongly typed domain primitives (`EntityId`, `MeasurementValue`, `PackedAigGraph`, `VolumetricTensor3D`, `StandardCellSiteRow`, `BaseSiliconLock`, `LegalizedCellInstance`).
   - Share pre-computed data structures across crate boundaries (e.g., NPN permutation automorphism groups computed in `hwc-synthesis` and stored in `EntityGraph` are consumed directly in $O(1)$ by `hwc-physics` and `hwc-router` without re-solving).

3. **Aggressive Technical Debt Deletion (No Dead / Commented-Out Code):**
   - **Delete obsolete code completely.** Do NOT comment out legacy functions, structs, or old branches.
   - Remove the dead code physically from the file.
   - Leave a concise one-line doc comment or commit note indicating what legacy mechanism was replaced (e.g., `// Replaces legacy MAX_STEP_LIMIT loop counter with DeterministicGuard fuel accounting`).

4. **Complete File Deletion for Obsolete Modules:**
   - If an entire file or subsystem is obsolete (e.g., legacy heuristic resolvers or superseded prototype passes), **delete the file entirely** and remove its module declaration from `mod.rs` and `Cargo.toml`.

5. **Zero Backward Compatibility Burden:**
   - HardwareScript is in active pre-release development. **Do NOT carry backward compatibility shims, legacy format adapters, deprecated aliases, or transitional wrappers.**
   - All code must build cleanly, strictly, and solely against the hardened v0.3.1 production specification.

---

## 2. Topological Implementation Dependency DAG

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                   TOPOLOGICAL IMPLEMENTATION DEPENDENCY DAG                 │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  [PHASE 0: Lexical & AST Primitives] ──► hwc-parser                         │
│       │                                                                     │
│       ▼                                                                     │
│  [PHASE 1: Identity & Memory Model]  ──► hwc-engine::entity_graph           │
│       │                                                                     │
│       ▼                                                                     │
│  [PHASE 2: Comptime VM & Buffering]  ──► hwc-compiler::eval                 │
│       │                                                                     │
│       ▼                                                                     │
│  [PHASE 3: Logic Synthesis & Legal]  ──► hwc-synthesis                      │
│       │                                                                     │
│       ▼                                                                     │
│  [PHASE 4: PDK & Gate-Array Library] ──► stdlib/pdk/sky130                  │
│       │                                                                     │
│       ▼                                                                     │
│  [PHASE 5: Signoff & 1-WL LVS Gate]  ──► hwc-physics                        │
│       │                                                                     │
│       ▼                                                                     │
│  [PHASE 6: Tri-Hybrid Router & ECO]  ──► hwc-router                         │
│       │                                                                     │
│       ▼                                                                     │
│  [PHASE 7: Exporters & Lockfile]     ──► hwc-export                         │
│       │                                                                     │
│       ▼                                                                     │
│  [PHASE 8: Salsa Ingestion & CLI]    ──► hwc-cli + hwc-compiler::query      │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 3. Summary Dependency Table

| Phase | Subsystem / Crate | Direct Dependencies | Blocks Downstream | Detailed Plan |
| :--- | :--- | :--- | :--- | :--- |
| **Phase 0** | `hwc-parser` | Standard library only | Phase 1, 2, 3 | [Phase 0 Plan](file:///c:/Users/olowo/Downloads/Code/hw/ROADMAP/v0.3.1/Phase-0-Lexical-AST-Primitives.md) |
| **Phase 1** | `hwc-engine` | Phase 0 | Phase 2, 3, 5, 6, 7 | [Phase 1 Plan](file:///c:/Users/olowo/Downloads/Code/hw/ROADMAP/v0.3.1/Phase-1-Core-Physical-Types-Identity.md) |
| **Phase 2** | `hwc-compiler::eval` | Phase 0, Phase 1 | Phase 4, Phase 8 | [Phase 2 Plan](file:///c:/Users/olowo/Downloads/Code/hw/ROADMAP/v0.3.1/Phase-2-Comptime-VM-Buffering.md) |
| **Phase 3** | `hwc-synthesis` | Phase 0, Phase 1 | Phase 4, Phase 5, Phase 6 | [Phase 3 Plan](file:///c:/Users/olowo/Downloads/Code/hw/ROADMAP/v0.3.1/Phase-3-Logic-Synthesis-Row-Legalization.md) |
| **Phase 4** | `stdlib/` | Phase 2, Phase 3 | Phase 5, Phase 6 | [Phase 4 Plan](file:///c:/Users/olowo/Downloads/Code/hw/ROADMAP/v0.3.1/Phase-4-PDK-Gate-Array-Library.md) |
| **Phase 5** | `hwc-physics` | Phase 1, Phase 2, Phase 3 | Phase 6, Phase 7 | [Phase 5 Plan](file:///c:/Users/olowo/Downloads/Code/hw/ROADMAP/v0.3.1/Phase-5-Signoff-1-WL-LVS-Gate.md) |
| **Phase 6** | `hwc-router` | Phase 1, Phase 3, Phase 5 | Phase 7, Phase 8 | [Phase 6 Plan](file:///c:/Users/olowo/Downloads/Code/hw/ROADMAP/v0.3.1/Phase-6-Tri-Hybrid-Router-ECO.md) |
| **Phase 7** | `hwc-export` | Phase 1, Phase 5, Phase 6 | Phase 8 | [Phase 7 Plan](file:///c:/Users/olowo/Downloads/Code/hw/ROADMAP/v0.3.1/Phase-7-Exporters-Lockfile.md) |
| **Phase 8** | `hwc-cli` + Salsa Query | Phases 0–7 | End-to-End Release | [Phase 8 Plan](file:///c:/Users/olowo/Downloads/Code/hw/ROADMAP/v0.3.1/Phase-8-Salsa-Query-Ingestion-CLI.md) |

---

## 4. Master Phase Index & Progress Checklist

- [x] **[Phase 0: Lexical, Grammar & AST Extensions (`hwc-parser`)](file:///c:/Users/olowo/Downloads/Code/hw/ROADMAP/v0.3.1/Phase-0-Lexical-AST-Primitives.md)**
  - [x] Brace & Natural Logic Tokens (`{ ... }`, `and`, `or`, `not`, `+=`, `-=`, `match`, `0..N`)
  - [x] Loop Semantic Keys (`key: "string_expr"`)
  - [x] Behavioral Logic Blocks (`logic { ... }`, `reg`, sequential & combinational AST nodes)
  - [x] Comptime Attributes (`#[comptime_fuel(...)]`)

- [x] **[Phase 1: Core Physical Types, Identity & Base Database (`hwc-engine`)](file:///c:/Users/olowo/Downloads/Code/hw/ROADMAP/v0.3.1/Phase-1-Core-Physical-Types-Identity.md)**
  - [x] Merkle Path Identity & Span-Independent `EntityId` (`identity.rs`)
  - [x] Bi-directional Identity Registry (`registry.rs`)
  - [x] Base Silicon Snapshot Provenance (`freeze_lock.rs` / `BaseSiliconLock`)
  - [x] Stackup & Dielectric Context Manager (`StackupManager`)

- [x] **[Phase 2: Comptime VM & Pure Geometry Buffering (`hwc-compiler::eval`)](file:///c:/Users/olowo/Downloads/Code/hw/ROADMAP/v0.3.1/Phase-2-Comptime-VM-Buffering.md)**
  - [x] Deterministic Fuel & Quota Guard (`sandbox.rs`)
  - [x] 128-bit Measurement & Value System (`value.rs`)
  - [x] Merkle-Bearing `GeometryRecord` with mandatory `id: EntityId` (`geometry_record.rs`)
  - [x] High-Density Flat Coordinate Arena (`FlatGeometryBuffer`)
  - [x] CallFrame Path-Stack Tracker & OpCode Emitters (`mod.rs`)

- [x] **[Phase 3: Digital Logic Synthesis & Row Legalization (`hwc-synthesis`)](file:///c:/Users/olowo/Downloads/Code/hw/ROADMAP/v0.3.1/Phase-3-Logic-Synthesis-Row-Legalization.md)**
  - [x] 64-bit Packed AIG Arena (`aig/arena.rs`)
  - [x] Word-Level E-Graphs & Term Rewriting (`datapath/egraph.rs`)
  - [x] SIMD Bit-Parallel Simulation & Cadical FRAIG Sweeping (`aig/fraig.rs`)
  - [x] Fast Liberty Parser & NPN Truth Table Canonicalizer (`mapper/npn.rs`)
  - [x] Priority $K$-Cut Technology Mapping (`mapper/priority_cuts.rs`)
  - [x] Shift-Left Analytical Placer & Abacus Row Legalizer (`mapper/row_legalizer.rs`)
  - [x] Single-Source Permittivity Delay Estimator (`ShiftLeftDelayEstimator`)
  - [x] Formal Combinational Equivalence Miter (`verify/cec.rs`)
  - [x] Universal `wasm64` Runner Bridge (`wasm/wasm64_runner.rs`)

- [x] **[Phase 4: Standard Library & Gate-Array Fillers (`stdlib/`)](file:///c:/Users/olowo/Downloads/Code/hw/ROADMAP/v0.3.1/Phase-4-PDK-Gate-Array-Library.md)**
  - [x] Gate-Array Filler PCell (`@std/pdk/sky130/ga_filler.hw`)
  - [x] Foundry Standard Cell Wrappers (NAND2, NOR2, AOI21, DFF, INV, MUX2, BUF, AND2, OR2, XOR2)
  - [x] Digital IP Macros in Pure `.hw` (`@std/digital/`)

- [ ] **[Phase 5: Physical Sign-off Gates & 1-WL LVS Engine (`hwc-physics`)](file:///c:/Users/olowo/Downloads/Code/hw/ROADMAP/v0.3.1/Phase-5-Signoff-1-WL-LVS-Gate.md)**
  - [ ] G-Cell Morton-ordered 8-wide AVX-512 SIMD DRC (`simd_drc.rs`)
  - [ ] PIVB Topological Solver & Automatic Rail Welder (`pivb.rs`)
  - [ ] BEM Parasitic Extraction (`parasitic_solver.rs`)
  - [ ] Mask Extraction & Float-to-Picometer Canonicalization (`lvs/extractor.rs`)
  - [ ] Uncommitted GA-Filler Device Pruning (`lvs/reduction.rs`)
  - [ ] 1-WL Bipartite Graph Isomorphism Matcher with $O(1)$ NPN Symmetries (`lvs/matcher.rs`)

- [x] **[Phase 6: Tri-Hybrid Physical Router & Freeze-Silicon ECO (`hwc-router`)](file:///c:/Users/olowo/Downloads/Code/hw/ROADMAP/v0.3.1/Phase-6-Tri-Hybrid-Router-ECO.md)**
  - [x] Pin Access Analysis with Row-Legalized Scoring (`paa/scoring.rs`)
  - [x] 14-Byte SoA `VolumetricTensor3D` & Congestion Feedback API (`global/tensor.rs`)
  - [x] FastGR GPU / L3 Pattern Global Router (`global/cuda_fastgr.rs`, `cpu_pathfinder.rs`)
  - [x] Panel Track Assignment with NPN Dynamic Pin Swapping (`track_assign/pin_swap.rs`)
  - [x] Dr. CU 2.0 Sparse-Grid Detailed Router (`detailed/sparse_grid.rs`, `timing_rrr.rs`)
  - [x] Thread-Local WASM Instance Pool for DOPHR Stage 3 (`ffi/wasm64_runner.rs`)
  - [x] Freeze-Silicon Metal-Only ECO Engine (`eco/`)

- [x] **[Phase 7: Manufacturing Stream Exporters & Persistence (`hwc-export`)](file:///c:/Users/olowo/Downloads/Code/hw/ROADMAP/v0.3.1/Phase-7-Exporters-Lockfile.md)**
  - [x] Clipper2 Non-Zero Winding 2D Copper Union (`welder.rs`)
  - [x] Earcut 3D Substrate Triangulation (`substrate.rs`)
  - [x] SEMI GDSII / OASIS Stream Writer with Cut-Mask Emission (`gdsii.rs`, `oasis.rs`)
  - [x] SPICE Netlist Exporter (`netlist.rs`)
  - [x] Zero-Copy `.hwx` Binary Stream Format (`hwx.rs`)

- [x] **[Phase 8: Salsa Query Ingestion, CLI & End-to-End Gauntlet (`hwc-cli`)](file:///c:/Users/olowo/Downloads/Code/hw/ROADMAP/v0.3.1/Phase-8-Salsa-Ingestion-CLI.md)**
  - [x] Demand-Driven Salsa Query Pipeline (`ir/query.rs`)
  - [x] Freeze-Silicon Snapshot Ingestion (`eco_query.rs`)
  - [x] Complete CLI Command Suite (`check`, `eval`, `build`, `test`)
  - [x] Full Tapeout Gauntlet Verification (`cmos_inverter.hw`, `accelerator.hw`, `divider_eco.hw`, `neural_crossbar_1024.hw`)

---

## 5. Primary Architecture Documentation References

For full mathematical proofs, Rust structs, and C-ABIs, refer to:
* [Comptime-Virtual-Machine.md](file:///c:/Users/olowo/Downloads/Code/hw/Docs/v0.3.1/Comptime-Virtual-Machine.md)
* [Digital-Logic-Synthesis.md](file:///c:/Users/olowo/Downloads/Code/hw/Docs/v0.3.1/Digital-Logic-Synthesis.md)
* [Pluggable-Routing-Engine-Architecture.md](file:///c:/Users/olowo/Downloads/Code/hw/Docs/v0.3.1/Pluggable-Routing-Engine-Architecture.md)
* [Stable-Structural-Identity.md](file:///c:/Users/olowo/Downloads/Code/hw/Docs/v0.3.1/Stable-Structural-Identity.md)
