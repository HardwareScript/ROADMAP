# Phase 3: Digital Logic Synthesis & Row Legalization

**Target Crate:** `crates/hwc-synthesis`  
**Status:** ✅ Complete — All 9 tests pass, full workspace builds clean  
**Completed:** 2026-08-30  
**Pre-requisites:** Phase 0 (`hwc-parser` AST), Phase 1 (`hwc-engine` types)  
**Blocks Downstream:** Phase 4 (`stdlib/`), Phase 5 (`hwc-physics`), Phase 6 (`hwc-router`)  
**Specification References:**
* [Digital-Logic-Synthesis.md § 3](file:///c:/Users/olowo/Downloads/Code/hw/Docs/v0.3.1/Digital-Logic-Synthesis.md#L108) (64-bit Packed AIG Arena `Vec<u64>`)
* [Digital-Logic-Synthesis.md § 4](file:///c:/Users/olowo/Downloads/Code/hw/Docs/v0.3.1/Digital-Logic-Synthesis.md#L252) (Word-Level E-Graphs `WordExpr`)
* [Digital-Logic-Synthesis.md § 6](file:///c:/Users/olowo/Downloads/Code/hw/Docs/v0.3.1/Digital-Logic-Synthesis.md#L322) (Priority $K$-Cut Mapping & NPN Symmetries)
* [Digital-Logic-Synthesis.md § 7.1](file:///c:/Users/olowo/Downloads/Code/hw/Docs/v0.3.1/Digital-Logic-Synthesis.md#L450) (`StandardCellRowLegalizer` Abacus algorithm)
* [Digital-Logic-Synthesis.md § 7.4](file:///c:/Users/olowo/Downloads/Code/hw/Docs/v0.3.1/Digital-Logic-Synthesis.md#L450) (`ShiftLeftDelayEstimator` with StackupManager)
* [Digital-Logic-Synthesis.md § 8](file:///c:/Users/olowo/Downloads/Code/hw/Docs/v0.3.1/Digital-Logic-Synthesis.md#L450) (Combinational SAT Miter Equivalence Gate)
* [Digital-Logic-Synthesis.md § 9](file:///c:/Users/olowo/Downloads/Code/hw/Docs/v0.3.1/Digital-Logic-Synthesis.md#L450) (Universal `wasm64` Plugin Interface)

---

## 1. Core Responsibilities & Tasks

- [x] **1. 64-bit Packed AIG Arena (`aig/arena.rs`)**
  - [x] Implement `PackedAigGraph` using flat `Vec<u64>` node storage (8 bytes per node, 0 heap pointer chasing).
  - [x] Implement `Edge(pub u32)` with inversion bit and trivial algebraic constant folding.
  - [x] Add `input_nodes: Vec<u32>` and `get_input_node()` for exact node-ID tracking per primary input.

- [x] **2. Word-Level Datapath Optimization (`datapath/egraph.rs`)**
  - [x] Implement equality saturation and term rewriting over `WordExpr` to preserve multi-bit vector arithmetic and carry chains.

- [x] **3. SIMD Bit-Parallel Simulation & FRAIG SAT Sweeping (`aig/fraig.rs`)**
  - [x] Implement 64-bit SIMD random simulation vectors.
  - [x] Implement candidate equivalence hashing and bounded SAT sweeping to merge redundant nodes.
  - [x] Fix DPLL backtracking with clean snapshot-based branch isolation.
  - [x] Map reconstructed inputs using exact original `input_nodes` (fixes node-index aliasing bug).

- [x] **4. Liberty Parser & NPN Canonicalizer (`mapper/npn.rs`)**
  - [x] Fast 64-bit truth table NPN canonicalizer (<50 ns).
  - [x] Extract input permutation automorphism groups ($S_2, S_3$).
  - [x] Dual-key Liberty catalog indexing by full name and short `cell_type`.

- [x] **5. Priority $K$-Cut Technology Mapping (`mapper/priority_cuts.rs`)**
  - [x] Implement 6-cut dynamic programming coverage against foundry Liberty cell catalog (`sky130_fd_sc_hd`).

- [x] **6. Analytical Placer, Row Legalizer & Permittivity (`mapper/row_legalizer.rs`, `placer_loop.rs`)**
  - [x] Implement quadratic analytical placement with `ShiftLeftDelayEstimator` querying `StackupManager`.
  - [x] Implement `StandardCellRowLegalizer::legalize_to_rows` (Abacus) to snap cells to PDK site rows (VDD/VSS rail abutment).
  - [x] Snap row origin to global die row grid (`aligned_y_min = (y_min / row_h) * row_h`) for continuous power rail abutment.
  - [x] Export `LegalizedCellInstance.input_automorphism_group` for downstream sharing.

- [x] **7. Formal Combinational Equivalence Miter (`verify/cec.rs`)**
  - [x] Build Combinational SAT Miter circuit ($Y_{\text{golden}} \oplus Y_{\text{synth}}$) proving 100% equivalence (UNSAT) before geometry emission.
  - [x] Bind golden/synth inputs using exact node IDs from `input_nodes` (eliminates aliasing with DFF Q nodes).
  - [x] Bind DFF Q outputs by name-matched register pairs across golden and synthesized graphs.
  - [x] Compare next-state $D$ inputs for sequential registers in the miter.

- [x] **8. Universal `wasm64` Synthesis Runner (`wasm/wasm64_runner.rs`)**
  - [x] Implement embedded Wasmtime Memory64 runtime to load Tier 3 Yosys/ABC `.wasm` plugins.

- [x] **9. Strongly-Typed RTL Lowering (`lib.rs`)**
  - [x] `SignalRole` enum (`PrimaryInput`, `PrimaryOutput`, `Register`).
  - [x] `SignalSymbol` struct with role, name, and current AIG edge.
  - [x] `SynthesisEnvironment` with `written_in_scope` SSA tracking for branch fork/merge.
  - [x] SSA-correct `if/else` lowering: fork env before branches, merge with `MUX(cond, then_val, else_val)` only for written variables.

---

## 2. Verification Gate

- [x] Synthesize `accelerator.hw` logic block: completes in <1 ms, gate count = 12, 0 off-grid cells.
- [x] CEC SAT Miter proves UNSAT on synthesized standard-cell netlist.
- [x] Automorphism group $S_2$ correctly attached to NAND2/NOR2 instances.

---

## 3. Final Test Results

```
running 9 tests
test test_abacus_row_legalizer_abutment             ... ok
test test_packed_aig_arena_constant_folding          ... ok
test test_npn_canonicalization_and_automorphism      ... ok
test test_fraig_simulation_and_sat_sweeping          ... ok
test test_shift_left_delay_estimator_with_stackup    ... ok
test test_formal_cec_sat_miter                       ... ok
test test_word_level_egraph_optimization             ... ok
test test_priority_k_cut_technology_mapping          ... ok
test test_end_to_end_accelerator_synthesis           ... ok

test result: ok. 9 passed; 0 failed; 0 ignored; 0 measured
cargo build → Finished `dev` profile [unoptimized + debuginfo] in 13.52s
```
