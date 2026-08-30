# Phase 4: Standard Library & Gate-Array (GA) Fillers

**Target Package:** `stdlib/`  
**Status:** ✅ Complete — All PDK & Digital IP modules implemented, full workspace builds clean & passes test suite  
**Completed:** 2026-08-30  
**Pre-requisites:** Phase 2 (`hwc-compiler::eval`), Phase 3 (`hwc-synthesis`)  
**Blocks Downstream:** Phase 5 (`hwc-physics`), Phase 6 (`hwc-router`)  
**Specification References:**
* [Stable-Structural-Identity.md § 4.2](file:///c:/Users/olowo/Downloads/Code/hw/Docs/v0.3.1/Stable-Structural-Identity.md#L451) (Gate-Array Filler Cell Architecture)
* [Digital-Logic-Synthesis.md § 1](file:///c:/Users/olowo/Downloads/Code/hw/Docs/v0.3.1/Digital-Logic-Synthesis.md#L12) (Tier 1 Procedural Controllers `@std/digital`)

---

## 1. Core Responsibilities & Tasks

- [x] **1. Gate-Array (GA) Filler PCell (`stdlib/pdk/sky130/ga_filler.hw`)**
  - [x] Implement `sky130_fd_sc_hd__ga_fill` containing uncommitted PMOS and NMOS transistor diffusions and poly gates in Base Layers 1–20.
  - [x] Terminals remain untied beneath Metal-1 until configured by an ECO patch.

- [x] **2. Foundry Standard Cell Wrappers (`stdlib/pdk/sky130/`)**
  - [x] Implement pure `.hw` parametric cell wrappers for SkyWater SKY130 standard cells: `nand2`, `nor2`, `aoi21`, `dff`, `inv`, `mux2`, `buf`, `and2`, `or2`, `xor2`.
  - [x] Pin offsets and bounding boxes match discrete PDK site pitch ($2.72\,\mu\text{m} \times 0.46\,\mu\text{m}$).
  - [x] Add `resistor.hw` (`sky130_res_high_po`) and `profile.hw` (`SKY130_1V8_CMOS`).

- [x] **3. Digital IP Macro Library (`stdlib/digital/`)**
  - [x] Implement Tier 1 procedural hardware controller macros: SPI master/slave (`spi.hw`), UART TX/RX (`uart.hw`), I2C master (`i2c.hw`), synchronous FIFOs (`fifo.hw`), and CLA adders (`adder.hw`).

---

## 2. Verification Gate

- [x] Instantiate 8 GA-filler cells in `divider_eco.hw`: verifies base silicon diffusions are generated with untied terminals.
- [x] Elaboration of `@std/digital/spi` macro instantiates cleanly into pure `GeometryBuffer` / `Value` in $<1\text{ ms}$.
- [x] Full integration test suites pass in `crates/hwc-stdlib/tests/phase4_stdlib_tests.rs` and `crates/hwc-compiler/tests/phase4_compiler_tests.rs`.
