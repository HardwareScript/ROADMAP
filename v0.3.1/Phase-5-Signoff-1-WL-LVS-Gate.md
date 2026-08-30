# Phase 5: Physical Sign-off Gates & 1-WL LVS Engine

**Target Crate:** `crates/hwc-physics`  
**Pre-requisites:** Phase 1 (`hwc-engine`), Phase 2 (`hwc-compiler::eval`), Phase 3 (`hwc-synthesis` NPN types)  
**Blocks Downstream:** Phase 6 (`hwc-router`), Phase 7 (`hwc-export`)  
**Specification References:**
* [Stable-Structural-Identity.md  3](file:///c:/Users/olowo/Downloads/Code/hw/Docs/v0.3.1/Stable-Structural-Identity.md#L184) (1-WL Graph Isomorphism LVS Gate)
* [Stable-Structural-Identity.md  3.4](file:///c:/Users/olowo/Downloads/Code/hw/Docs/v0.3.1/Stable-Structural-Identity.md#L345) (GA-Filler Device Reduction & Pruning)
* [Stable-Structural-Identity.md  3.5](file:///c:/Users/olowo/Downloads/Code/hw/Docs/v0.3.1/Stable-Structural-Identity.md#L420) (Parameter Unit Canonicalization)
* [Pluggable-Routing-Engine-Architecture.md  7](file:///c:/Users/olowo/Downloads/Code/hw/Docs/v0.3.1/Pluggable-Routing-Engine-Architecture.md#L608) (Zero-Trust Signoff Gate: G-Cell SIMD DRC & PIVB)

---

## 1. Core Responsibilities & Tasks

- [ ] **1. G-Cell Morton SIMD DRC (`simd_drc.rs`)**
  - [ ] Implement 8-wide AVX-512 SIMD geometric sweep across Morton-ordered spatial buckets.
  - [ ] Verify minimum spacing, minimum area, End-of-Line (EOL) clearance, and cut-mask enclosure rules.

- [ ] **2. PIVB Topological Solver & Rail Welder (`pivb.rs`)**
  - [ ] Implement Tarjan's Strongly Connected Components (SCC) on pre-welded Clipper2 contours.
  - [ ] Verify 100% net continuity (0 opens/shorts) and automatic power rail continuity across abutted standard-cell rows.

- [ ] **3. BEM Parasitic Extraction (`parasitic_solver.rs`)**
  - [ ] Implement Wheeler $\varepsilon_{\text{eff}}$ + Sakurai microstrip parasitic model ($C_{\text{gnd}}$, $C_{\text{coupling}}$, $R_{\text{wire}}$).
  - [ ] Verify electromigration current density (P21) and thermal rise (P22).

- [ ] **4. Layout Extraction & Parameter Canonicalization (`lvs/extractor.rs`)**
  - [ ] Extract layout bipartite graph $G_L(V_{\text{dev}}, V_{\text{net}}, E_L)$ from continuous GDSII polygon masks.
  - [ ] Flatten golden schematic bipartite graph $G_S(V_{\text{dev}}, V_{\text{net}}, E_S)$.
  - [ ] Canonicalize all float device parameters to integer picometers (`(w_float * 1e12).round() as i64`).

- [ ] **5. Uncommitted GA-Filler Spare Transistor Pruning (`lvs/reduction.rs`)**
  - [ ] Implement `LvsDeviceReduction::prune_uncommitted_ga_fillers(devices, signal_net_ids)` to tag and prune untied spare transistors from $G_L$ before 1-WL coloring (eliminates false `LVS_01`).
  - [ ] Preserve configured GA-fillers with active signal connections.

- [ ] **6. 1-WL Weisfeiler-Lehman Graph Isomorphism Matcher (`lvs/matcher.rs`)**
  - [ ] Implement iterative 32-pass 1-WL multiset color refinement over bipartite circuit graphs.
  - [ ] Read `input_automorphism_group` from `EntityGraph` for $O(1)$ symmetric pin-swapping verification without solving graph automorphisms from scratch.

---

## 2. Verification Gate

- [ ] Run 1-WL LVS on `cmos_inverter.hw`: verifies 100% isomorphic graph match ($G_L \cong G_S$).
- [ ] Run 1-WL LVS on `divider_eco.hw`: verifies 0 false `LVS_01` alarms from 8 uncommitted GA-fillers, while configured ECO jumper cells pass.
- [ ] Run PIVB solver on row-abutted cells: verifies continuous power rails with 1 connected component.
