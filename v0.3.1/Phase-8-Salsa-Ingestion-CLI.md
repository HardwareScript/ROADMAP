# Phase 8: Salsa Query Ingestion, CLI & End-to-End Gauntlet

**Target Crates:** `crates/hwc-compiler::ir::query`, `crates/hwc-cli`  
**Pre-requisites:** Phases 0 through 7 (All Subsystems Completed & Verified)  
**Blocks Downstream:** End-to-End v0.3.1 Production Release  
**Specification References:**
* [Comptime-Virtual-Machine.md  4.2](file:///c:/Users/olowo/Downloads/Code/hw/Docs/v0.3.1/Comptime-Virtual-Machine.md#L222) (Salsa Query Compliance & Incremental Memoization)
* [Stable-Structural-Identity.md  4.5](file:///c:/Users/olowo/Downloads/Code/hw/Docs/v0.3.1/Stable-Structural-Identity.md#L570) (Salsa `BaseSiliconLock` ECO Query Tracking)
* [Digital-Logic-Synthesis.md  10.2](file:///c:/Users/olowo/Downloads/Code/hw/Docs/v0.3.1/Digital-Logic-Synthesis.md#L450) (`hwc build` CLI execution trace)

---

## 1. Core Responsibilities & Tasks

- [x] **1. Salsa Demand-Driven Query Pipeline (`ir/query.rs`)**
  - [x] Implement query flow:
    ```
    parse_ast_query ──► eval_space_query (Arc<GeometryBuffer>) ──► Single-Pass Ingestion
                                                                          │
    export_gdsii_query ◄── run_physics_signoff_query ◄── route_space_query ◄┘
    ```
  - [x] Implement single-pass ingestion worker pouring `GeometryBuffer` into `hwc-engine::EntityGraph` using embedded `id: EntityId`s.
  - [x] Implement `base_silicon_snapshot` and `verify_freeze_silicon_immutability` queries in `eco_query.rs`.

- [x] **2. High-Throughput CLI Interface (`hwc-cli`)**
  - [x] `hwc check <file.hw>`: Type check and syntax validation in $<5\text{ ms}$.
  - [x] `hwc eval <file.hw>`: Pure comptime elaboration and geometry footprint audit in $<10\text{ ms}$.
  - [x] `hwc build <file.hw>`: Full end-to-end compilation to GDSII and SPICE ($<40\text{ ms}$ cold build).
  - [x] `hwc build --eco-mode=metal-freeze <file.hw>`: Post-tapeout Freeze-Silicon ECO mode.
  - [x] `hwc test <file.hw>`: Automated SPICE testbench execution with embedded `ngspice.wasm`.

- [x] **3. End-to-End Tapeout Verification Gauntlet**
  - [x] Run `cmos_inverter.hw` (CMOS baseline).
  - [x] Run `neural_crossbar_1024.hw` ($1024 \times 1024$ crossbar, 1,048,576 cells, $<300\text{ ms}$).
  - [x] Run `accelerator.hw` (Tier 2 behavioral logic + SPI + 30 standard cells + routing + SAT miter, $<40\text{ ms}$).
  - [x] Run `divider_eco.hw` (Freeze-silicon ECO mode, 0 base mutations, 1-WL LVS passing).

---

## 2. Verification Gate

- [x] All 4 gauntlet test files build with 0 DRC violations, 0 LVS false alarms, and 0 memory leaks.
- [x] Changing whitespace or comments in any source file yields $<3\text{ ms}$ incremental rebuild times via 100% Salsa cache hits.
- [x] Bit-identical GDSII checksums verified across repeated runs on Windows, macOS, and Linux.
