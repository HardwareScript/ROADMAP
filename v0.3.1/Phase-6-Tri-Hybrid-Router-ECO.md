# Phase 6: Tri-Hybrid Physical Router & Freeze-Silicon ECO

**Target Crate:** `crates/hwc-router`  
**Pre-requisites:** Phase 1 (`hwc-engine`), Phase 3 (`hwc-synthesis` Legalized cells & NPN symmetries), Phase 5 (`hwc-physics` Signoff gates)  
**Blocks Downstream:** Phase 7 (`hwc-export`), Phase 8 (`hwc-cli`)  
**Specification References:**
* [Pluggable-Routing-Engine-Architecture.md  2](file:///c:/Users/olowo/Downloads/Code/hw/Docs/v0.3.1/Pluggable-Routing-Engine-Architecture.md#L49) (End-to-End Routing Pipeline)
* [Pluggable-Routing-Engine-Architecture.md  3](file:///c:/Users/olowo/Downloads/Code/hw/Docs/v0.3.1/Pluggable-Routing-Engine-Architecture.md#L96) (14-byte SoA `VolumetricTensor3D`)
* [Pluggable-Routing-Engine-Architecture.md  5.2](file:///c:/Users/olowo/Downloads/Code/hw/Docs/v0.3.1/Pluggable-Routing-Engine-Architecture.md#L544) (Thread-Local WASM Instance Pool for DOPHR Stage 3)
* [Pluggable-Routing-Engine-Architecture.md  5.3](file:///c:/Users/olowo/Downloads/Code/hw/Docs/v0.3.1/Pluggable-Routing-Engine-Architecture.md#L544) (`congestion_at_pm` feedback API)
* [Stable-Structural-Identity.md  4](file:///c:/Users/olowo/Downloads/Code/hw/Docs/v0.3.1/Stable-Structural-Identity.md#L364) (Freeze-Silicon Metal-Only ECO Architecture)

---

## 1. Core Responsibilities & Tasks

- [x] **1. Stage 1: Pin Access Analysis (`paa/scoring.rs`)**
  - [x] Evaluate pin geometries on row-legalized standard cells.
  - [x] Generate legal, on-grid `AccessPoint`s with enclosure scoring, eliminating off-grid starvation (`Error R01`).

- [x] **2. Stage 2: 3D Volumetric Capacity Tensor (`global/tensor.rs`)**
  - [x] Implement 14-byte/cell Structure-of-Arrays (SoA) `VolumetricTensor3D`.
  - [x] Expose `congestion_at_pm(x, y)` for synthesis congestion-aware placement feedback.
  - [x] Implement FastGR GPU pattern routing kernel (`cuda_fastgr.rs`) and L3 CPU fallback (`cpu_pathfinder.rs`). Emits `GCellVolume3D` guides.

- [x] **3. Stage 3: Panel Track Assignment & Pin Swapping (`track_assign/`)**
  - [x] Slices G-Cells into horizontal/vertical panels and solves Maximum Weight Bipartite Matching (`bipartite.rs`).
  - [x] Implement dynamic input pin swapping (`pin_swap.rs`) using NPN `input_automorphism_group` ($S_2, S_3$) to untangle crossing nets and eliminate vias.

- [x] **4. Stage 4: Detailed Routing (`detailed/`)**
  - [x] Implement Dr. CU 2.0 multi-level sparse-grid A* maze search (`sparse_grid.rs`).
  - [x] Implement in-search EOL, area, and spacing lookahead DRC heuristics (`lookahead_drc.rs`).
  - [x] Implement timing-slack-weighted (WNS/TNS) Negotiated Congestion Rip-Up & Repair (`timing_rrr.rs`).

- [x] **5. Universal `wasm64` Thread-Local Instance Pool (`ffi/wasm64_runner.rs`)**
  - [x] Implement `thread_local!` WASM instance storage (`Store` + `Instance` per Rayon worker thread) for safe lock-free Stage 3 spatial 4-color dispatch.
  - [x] Stage 2 global plugins execute single-threaded on master instance.

- [x] **6. Freeze-Silicon Metal-Only ECO Engine (`eco/`)**
  - [x] Implement base silicon immutability verification (assert Layers 1–20 have 0 mutations).
  - [x] Extract Craig Interpolant minimal Boolean patches with `cadical`.
  - [x] Map patch logic gates to nearest uncommitted GA-fillers via Min-Cost Max-Flow.
  - [x] Restrict detailed router strictly to Metal 1–4 jumper tracks.

---

## 2. Verification Gate

- [x] Run `top_soc.hw` routing: routes 42,000 nets across 16 threads in $<10\text{ s}$ with 0 DRC violations.
- [x] Run `divider_eco.hw` in metal-freeze ECO mode: reroutes M1–M3 jumpers in $<2\text{ ms}$, base silicon remains 100% untouched.
- [x] Verify thread-local WASM instance pool runs 16 concurrent plugin threads without linear memory corruption.
