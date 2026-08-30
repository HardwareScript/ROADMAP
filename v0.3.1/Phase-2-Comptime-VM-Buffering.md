# Phase 2: Comptime VM & Pure Geometry Buffering

**Target Crate:** `crates/hwc-compiler::eval`  
**Pre-requisites:** Phase 0 (`hwc-parser`), Phase 1 (`hwc-engine::identity`)  
**Blocks Downstream:** Phase 4 (`stdlib/`), Phase 8 (`hwc-cli` / query pipeline)  
**Specification References:**
* [Comptime-Virtual-Machine.md  3](file:///c:/Users/olowo/Downloads/Code/hw/Docs/v0.3.1/Comptime-Virtual-Machine.md#L102) (Deterministic Dynamic Fuel formulation)
* [Comptime-Virtual-Machine.md  4.1](file:///c:/Users/olowo/Downloads/Code/hw/Docs/v0.3.1/Comptime-Virtual-Machine.md#L153) (Pure `GeometryRecord` with mandatory `id: EntityId`)
* [Comptime-Virtual-Machine.md  5.1](file:///c:/Users/olowo/Downloads/Code/hw/Docs/v0.3.1/Comptime-Virtual-Machine.md#L231) (`DeterministicGuard` with RAM quota)
* [Comptime-Virtual-Machine.md  5.5](file:///c:/Users/olowo/Downloads/Code/hw/Docs/v0.3.1/Comptime-Virtual-Machine.md#L726) (`FlatGeometryBuffer` high-density coordinate arena)

---

## 1. Core Responsibilities & Tasks

- [x] **1. Fuel & Host Memory Quota Guard (`sandbox.rs`)**
  - [x] Delete legacy hardcoded `MAX_STEP_LIMIT`.
  - [x] Implement `DeterministicGuard` with `DEFAULT_BASE_FUEL = 100_000_000` + area scaling ($10\text{M} / \text{mm}^2$) + explicit `#[comptime_fuel]` attribute overrides.
  - [x] Implement host RAM quota tracking (`track_allocation` / `track_deallocation`) with a default 2 GB limit (`Error C03`).

- [x] **2. 128-bit Measurement & Strong Value Model (`value.rs`)**
  - [x] Implement `MeasurementValue { raw: i128, dimension: UnitDimension }` for overflow-free picometer physical calculations.
  - [x] Implement strong `Value` enum (`Void`, `Bool`, `Int`, `Float`, `String`, `Measurement`, `Point2D`, `Point3D`, `Array`, `StructInstance`, `EnumVariant`, `NetHandle`, `SpaceHandle`).

- [x] **3. Merkle-Bearing Pure Geometry Buffering (`geometry_record.rs`)**
  - [x] Implement `GeometryRecord` enum (`Polygon`, `Contact`, `Device`, `RouteIntent`), each variant carrying a mandatory `id: EntityId`.
  - [x] Implement `GeometryBuffer` container for Salsa query purity.
  - [x] Implement `FlatGeometryBuffer` (contiguous `coordinate_pool: Vec<i64>` + `CompactGeometryRecordHeader`) for high-density arrays (>100k records).

- [x] **4. VM Execution Loop & Path-Stack Tracking (`mod.rs`, `context.rs`)**
  - [x] Update `CallFrame` to hold active `path: HierarchicalPath`.
  - [x] Push/pop path segments on `OpCode::Call` and `OpCode::Return`.
  - [x] Update `OpCode::EmitPolygon`, `OpCode::EmitContact`, and `OpCode::EmitDevice` to compute `EntityId::compute(&frame.path, ...)` at emission time and push pure records to `output_buffer`.

---

## 2. Verification Gate

- [ ] Run `cmos_inverter.hw` elaboration: pure `GeometryBuffer` emitted with deterministic `EntityId`s.
- [ ] Run `neural_crossbar_1024.hw` (1,048,576 cells unrolled): execution completes in $<300\text{ ms}$, peak RAM $<150\text{ MB}$, zero step limit crashes.
- [ ] Verify Salsa query memoization produces 100% cache hits when upstream whitespace is modified.
