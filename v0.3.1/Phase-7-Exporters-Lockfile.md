# Phase 7: Manufacturing Stream Exporters & Persistence

**Target Crate:** `crates/hwc-export`  
**Pre-requisites:** Phase 1 (`hwc-engine`), Phase 5 (`hwc-physics`), Phase 6 (`hwc-router`)  
**Blocks Downstream:** Phase 8 (`hwc-cli`)  
**Specification References:**
* [Pluggable-Routing-Engine-Architecture.md  7.1](file:///c:/Users/olowo/Downloads/Code/hw/Docs/v0.3.1/Pluggable-Routing-Engine-Architecture.md#L638) (Cut-Mask Geometry Flow for Sub-2nm Lithography)
* [Stable-Structural-Identity.md  4.5](file:///c:/Users/olowo/Downloads/Code/hw/Docs/v0.3.1/Stable-Structural-Identity.md#L570) (Base Silicon Checksum & Tapeout Emission)

---

## 1. Core Responsibilities & Tasks

- [x] **1. 2D Boundary Copper Welding (`welder.rs`)**
  - [x] Execute Clipper2 Non-Zero Winding 2D Boolean union strictly at the export boundary (keeping internal database representations fast and granular).

- [x] **2. 3D Substrate Triangulation (`substrate.rs`)**
  - [x] Implement Earcut polygon triangulation for 3D GLB export and web viewport rendering.

- [x] **3. SEMI GDSII & OASIS Binary Stream Writers (`gdsii.rs`, `oasis.rs`)**
  - [x] Stream tapeout-ready SEMI GDSII / OASIS files with configurable layer/datatype mappings.
  - [x] Stream `CutMaskPolygon` records directly to foundry cut-mask layers (e.g. Layer 42, Datatype 0) for sub-2nm SAQP/EUV lithography.

- [x] **4. SPICE Netlist Exporter (`netlist.rs`)**
  - [x] Emit `.sp` subcircuits populated with extracted BEM parasitic resistors ($R_s$) and ground/coupling capacitors ($C_{12}, C_{1g}$).

- [x] **5. Unified Zero-Copy Binary Format (`hwx.rs`)**
  - [x] Implement `.hwx` binary format with `rkyv` + `memmap2` for zero-copy memory-mapped loading in $<0.5\text{ ms}$.

---

## 2. Verification Gate

- [x] Export `Chip_Layout/board.gds` from `accelerator.hw`: verified bit-exact in KLayout with valid layer mappings.
- [x] Export `circuit.sp`: runs ngspice transient simulation without floating node errors.
- [x] Round-trip `.hwx` serialization and mmap loading in $<0.5\text{ ms}$.
