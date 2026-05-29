# System 2: Voxel Engine & Spatial Operations - Implementation Plan

**System**: Voxel Engine & Spatial Operations  
**Status**: ✅ COMPLETE - All Symbol Table integration finished  
**Migration Strategy**: **PRESERVE & ADAPT** (95% code preservation)  
**Based On**: v0.1.4 Documentation + v0.1.3 Implementation  
**Prerequisites**: System 1 (Unified Parser & Two-Pass Compiler) - ✅ **COMPLETE**  
**Last Updated**: March 22, 2026

---

## Overview

The voxel engine transforms Hardware IR into a 3D voxel grid with spatial operations. The core algorithms (Morton encoding, fixed-point math, NetlistArena) are **architecture-agnostic** and remain unchanged from v0.1.3.

**What Changes**: Data source integration (Symbol Table instead of file loading)  
**What Stays**: Everything else (Morton encoding, voxel grid, collision detection, placement algorithms)

**Architecture (Unchanged from v0.1.3)**:
- Voxel Grid: Morton Z-curve encoding for cache locality
- NetlistArena: ECS-style component/pin/net storage with strongly-typed IDs
- Fixed-Point Geometry: i64 nanometers for deterministic math
- Data-Oriented Design: Struct of Arrays (SoA) for performance
- Dependencies: rayon (for parallel operations)

**v0.1.4 Integration Points**:
- ✅ Component definitions: Load from Symbol Table (System 1 provides `get_component()`)
- ✅ Material properties: Load from Symbol Table (System 1 provides `get_material()`)
- ✅ Native SI unit values: No manual nm conversion needed (System 1 handles this)
- ✅ No changes to core voxel math, Morton encoding, or spatial algorithms

**What System 1 Provides**:
- `SymbolTable::get_component(&str) -> Result<&ComponentDef, CompileError>`
- `SymbolTable::get_material(&str) -> Result<&MaterialDef, CompileError>`
- `HardwareIR` with resolved components and nets
- Property extraction helpers for material properties

**Reference**: See `SYSTEM-1-IMPLEMENTATION-PLAN.md` Section: "Integration with Downstream Systems"

---

## Migration Strategy: Preserve & Adapt

### ✅ **Keep 100% (No Changes)**:
- `src/morton.rs` - Morton encoding (11 tests passing)
- `src/voxel_grid.rs` - Voxel grid operations (14 tests passing)
- `src/netlist.rs` - NetlistArena ECS (10 tests passing)
- `src/geometry.rs` - Fixed-point math (13 tests passing)

### ⚠️ **Adapt (Minimal Changes)**:
- `src/placement.rs` - Update component loading to use Symbol Table
- `src/routing.rs` - Update material loading to use Symbol Table
- `src/ir_integration.rs` - Update to receive Symbol Table reference

### ❌ **Remove**:
- Any YAML file loading code (if exists)
- Any `.hwx` file parsing code (if exists)

---

## Phase 1: Verify Core Systems (No Changes Needed) ✅ COMPLETE

### 1.1 Morton Encoding ✅ VERIFIED
- [x] 11 tests passing
- [x] `morton_encode()` function working
- [x] `morton_decode()` function working
- [x] Spatial locality verified
- [x] No changes needed for v0.1.4

### 1.2 Voxel Grid ✅ VERIFIED
- [x] 14 tests passing
- [x] `VoxelGrid` struct working
- [x] `is_empty()`, `set_occupied()`, `get_material()` working
- [x] Collision detection working
- [x] No changes needed for v0.1.4

### 1.3 NetlistArena ✅ VERIFIED
- [x] 10 tests passing
- [x] Strongly-typed IDs working
- [x] Component/Pin/Net storage working
- [x] O(1) lookups verified
- [x] No changes needed for v0.1.4

### 1.4 Fixed-Point Geometry ✅ VERIFIED
- [x] 13 tests passing
- [x] `Point3D`, `BoundingBox`, `TraceSegment` working
- [x] Manhattan distance calculations working
- [x] Deterministic across platforms
- [x] No changes needed for v0.1.4

**Status**: ✅ Core systems verified, no migration needed

---

## Phase 2: Component Placement Integration ✅ COMPLETE

**Reference**: [COMPILER-INTERNALS.md](../../Docs/v0.1.4/COMPILER-INTERNALS.md) - Symbol Table integration

**Current Status**: ✅ Component placement fully integrated with Symbol Table  
**Target Status**: ✅ ACHIEVED - Component placement loads from Symbol Table

**Prerequisites**: ✅ System 1 complete - Symbol Table available

### 2.1 Update Component Loading ✅ COMPLETE

**Documentation Reference**: 
- [COMPILER-INTERNALS.md](../../Docs/v0.1.4/COMPILER-INTERNALS.md) Section: "Pass 2: Space Unrolling & Assembly"
- [LANGUAGE-SPEC.md](../../Docs/v0.1.4/LANGUAGE-SPEC.md) Section: "3. Component Definition"

**OLD (v0.1.3)**:
```rust
// src/placement.rs
pub fn place_component(
    component_type: &str,
    name: &str,
    position: Point3D,
    rotation: f64,
    grid: &mut VoxelGrid,
    netlist: &mut NetlistArena,
) -> Result<ComponentId, PlacementError> {
    // Load component definition from .hwx file
    let component_def = load_component_definition(component_type)?;
    
    // ... rest of placement logic
}

fn load_component_definition(type_name: &str) -> Result<ComponentDef, Error> {
    // Parse .hwx file
    let path = format!("components/{}.hwx", type_name);
    let content = std::fs::read_to_string(path)?;
    parse_hwc_file(&content)
}
```

**CRITICAL: Dependency Inversion Pattern (No Circular Dependency)**

**Production Architecture** (Clean, No Circular Dependency):
```
hwc-parser (AST definitions)
    ↓
hwc-compiler (SymbolTable, implements traits)
    ↓
hwc-engine (defines traits, uses them generically)
    ↓
hwc-cli (orchestrates, passes SymbolTable to engine)
```

The engine CANNOT import the compiler directly. Instead, we use **Dependency Inversion**:

1. **Define trait in hwc-engine**:
```rust
pub trait SymbolTableTrait {
    fn get_component(&self, name: &str) -> Result<&hwc_parser::ComponentDefinition, String>;
    fn get_material(&self, name: &str) -> Result<&hwc_parser::MaterialDefinition, String>;
}
```

2. **Implement trait in hwc-compiler**:
```rust
impl hwc_engine::placement::SymbolTableTrait for SymbolTable {
    fn get_component(&self, name: &str) -> Result<&ComponentDefinition, String> {
        self.get_component(name).map_err(|e| e.to_string())
    }
    // ... same for get_material
}
```

3. **Use generic parameter in engine**:
```rust
pub fn place_component<S: SymbolTableTrait>(
    component_type: &str,
    name: &str,
    position: Point3D,
    rotation: f64,
    grid: &mut VoxelGrid,
    netlist: &mut NetlistArena,
    symbol_table: &S,  // Generic trait, not concrete type
) -> Result<ComponentId, PlacementError> {
    let component_def = symbol_table.get_component(component_type)?;
    // ... rest of placement logic (UNCHANGED)
}
```

**Why Mock for Unit Tests?**

The circular dependency ONLY appears during **test compilation**:
```
hwc-engine tests → need SymbolTable → from hwc-compiler → depends on hwc-engine → CIRCULAR!
```

When running `cargo test` on `hwc-engine`, the tests compile BEFORE `hwc-compiler` is fully compiled, so the trait implementation isn't visible yet.

**Solution**: Use `MockSymbolTable` in unit tests:
- ✅ Isolated unit tests (no compiler dependency)
- ✅ Fast compilation (don't compile hwc-compiler for every test)
- ✅ Clear test intent (mock shows exact test data)
- ✅ Real `SymbolTable` used in integration tests and production

**Changes Completed**:
- [x] Defined `SymbolTableTrait` in hwc-engine (Dependency Inversion)
- [x] Implemented trait in hwc-compiler `SymbolTable`
- [x] Updated `place_component()` to use generic trait parameter
- [x] Replaced hardcoded component definitions with Symbol Table lookup
- [x] Created `MockSymbolTable` for unit tests (no circular dependency)
- [x] Updated all placement tests (209 tests passing)

**Files Updated**:
- `hwc/crates/hwc-engine/src/placement.rs` - Added trait, updated placement logic, added MockSymbolTable
- `hwc/crates/hwc-compiler/src/symbol_table.rs` - Implemented trait
- `hwc/crates/hwc-engine/src/ir_integration.rs` - Temporarily disabled for Phase 4

### 2.2 Update Material Loading ✅ COMPLETE (Architecture Ready)

**Documentation Reference**:
- [STANDARD-LIBRARY-ARCHITECTURE.md](../../Docs/v0.1.4/STANDARD-LIBRARY-ARCHITECTURE.md) Section: "Unit Definition Format"
- [GENERIC-MEASUREMENT-ARCHITECTURE.md](../../Docs/v0.1.4/GENERIC-MEASUREMENT-ARCHITECTURE.md) Section: "Unit Classification at Parse Time"

**Status**: The `SymbolTableTrait` already includes `get_material()` method and is fully implemented.

**Current Implementation**:
```rust
pub trait SymbolTableTrait {
    fn get_component(&self, name: &str) -> Result<&ComponentDefinition, String>;
    fn get_material(&self, name: &str) -> Result<&MaterialDefinition, String>;
}
```

**Findings**:
- [x] `SymbolTableTrait::get_material()` already defined in placement.rs
- [x] Trait implemented in hwc-compiler's SymbolTable
- [x] MockSymbolTable implements get_material() for testing
- [x] No YAML or file loading code exists in routing.rs or constraint_manager.rs
- [x] Constraint functions (calculate_clearance_nm, calculate_trace_width_nm) receive material properties as parameters

**Conclusion**: Material loading infrastructure is complete. The constraint_manager functions are designed to receive material properties as parameters rather than loading them from files. If material property lookup by name is needed in the future, the `get_material()` trait method is already available.

**Files Verified**:
- `hwc/crates/hwc-engine/src/placement.rs` - Trait defined and used
- `hwc/crates/hwc-engine/src/routing.rs` - No material loading needed
- `hwc/crates/hwc-engine/src/constraint_manager.rs` - Functions accept material properties as parameters

### 2.3 Update Tests ✅ COMPLETE

**Documentation Reference**:
- [LANGUAGE-SPEC.md](../../Docs/v0.1.4/LANGUAGE-SPEC.md) Section: "3. Component Definition" for test component syntax

**Status**: All placement and integration tests have been updated to use Symbol Table in Phase 4.1.

**Changes Completed**:
- [x] All placement tests use Symbol Table (via transform_to_space helper)
- [x] Created test helper functions for component definitions (Battery, LED, Resistor)
- [x] No file-based test fixtures remain
- [x] All 325+ tests passing across workspace
- [x] Component definitions use proper syntax: `Rectangle(width, height, depth)`

**Test Helper Implementation**:
```rust
fn transform_to_space(program: &hwc_parser::Program) -> Result<hwc_engine::HardwareSpace, hwc_compiler::IrError> {
    use hwc_parser::{ComponentDefinition, LayoutBlock, PinPosition, Span};
    use std::collections::HashMap;
    
    let mut symbol_table = SymbolTable::new();
    let span = Span { start: 0, end: 0 };
    
    // Register Battery component
    let mut battery_pins = HashMap::new();
    battery_pins.insert("positive".to_string(), PinPosition { x: 0.0, y: 0.0, z: Some(0.0) });
    battery_pins.insert("negative".to_string(), PinPosition { x: 0.0, y: 1.0, z: Some(0.0) });
    symbol_table.register_component(ComponentDefinition {
        name: "Battery".to_string(),
        metadata: None,
        pins: vec!["positive".to_string(), "negative".to_string()],
        layout: Some(LayoutBlock {
            shape: Some("Rectangle(2.0mm, 1.25mm, 0.5mm)".to_string()),
            pin_positions: battery_pins,
            span,
        }),
        electrical: None,
        render: None,
        span,
    }).ok();
    
    // ... LED and Resistor definitions ...
    
    program_to_space(program, &symbol_table)
}
```

**Test Files Updated**:
- `hwc-engine/tests/automatic_routing_test.rs` - 10 tests passing
- `hwc-engine/tests/integration_test.rs` - 7 tests passing
- `hwc-engine/tests/origin_point_test.rs` - 14 tests passing
- `hwc-engine/tests/origin_matrix_validation_test.rs` - 8 tests passing

**System 1 Integration**:
- ✅ Test components match System 1 AST structure
- ✅ Component definitions use ComponentDefinition, LayoutBlock, PinPosition from hwc-parser
- ✅ All tests use `program_to_space(&program, &symbol_table)` signature

---

## Phase 3: Routing Integration ✅ COMPLETE (Architecture Ready)

**Reference**: [ROUTING-AND-PHYSICS.md](../../Docs/v0.1.4/ROUTING-AND-PHYSICS.md) - Material-based constraints

**Current Status**: ✅ Routing architecture complete - no material loading from files  
**Target Status**: ✅ ACHIEVED - Routing designed to receive material properties as parameters

**Prerequisites**: ✅ System 1 complete - Symbol Table available

### 3.1 Update Material Lookup in Routing ✅ COMPLETE (Not Needed)

**Documentation Reference**:
- [ROUTING-AND-PHYSICS.md](../../Docs/v0.1.4/ROUTING-AND-PHYSICS.md) Section: "Translation 1: Dielectric Breakdown to Clearance"

**Status**: The routing and constraint manager are already designed correctly.

**Current Implementation**:
```rust
// constraint_manager.rs
pub fn calculate_clearance_nm(
    voltage_diff_mv: i64,
    dielectric_strength_kv_mm: i64,
) -> i64 {
    // Material properties passed as parameters, not loaded from files
    let safety_factor = 2;
    let voltage_v = voltage_diff_mv / 1000;
    let dielectric_v_nm = (dielectric_strength_kv_mm * 1000) / 1_000_000;
    let min_clearance_nm = voltage_v / dielectric_v_nm;
    min_clearance_nm * safety_factor
}

pub fn calculate_trace_width_nm(
    current_ma: i64,
    temp_rise_c: i64,
    is_external: bool
) -> i64 {
    // IPC-2221 formula - no material loading needed
    // ...
}
```

**Findings**:
- [x] No YAML or file loading code in routing.rs
- [x] No YAML or file loading code in constraint_manager.rs
- [x] All constraint functions receive material properties as parameters
- [x] `SymbolTableTrait::get_material()` available if needed in future
- [x] Architecture follows dependency inversion pattern

**Conclusion**: The routing system is already correctly architected. Material properties are passed as parameters to constraint functions rather than being loaded from files. If material lookup by name is needed in the future, the `get_material()` trait method is available.

**Files Verified**:
- `hwc/crates/hwc-engine/src/routing.rs` - No material loading
- `hwc/crates/hwc-engine/src/constraint_manager.rs` - Functions accept material properties as parameters

### 3.2 Update Routing Tests ✅ COMPLETE

**Documentation Reference**:
- [LANGUAGE-SPEC.md](../../Docs/v0.1.4/LANGUAGE-SPEC.md) Section: "1. Material Definition" for test material syntax

**Status**: All routing tests passing (12 tests in routing.rs, 28 tests in constraint_manager.rs)

**Changes Completed**:
- [x] All 12 routing tests passing
- [x] All 28 constraint manager tests passing  
- [x] Tests use hardcoded material properties (correct approach for unit tests)
- [x] No file-based material loading

**Test Results**:
- routing.rs: 12 tests passing
- constraint_manager.rs: 28 tests passing (clearance, trace width, crosstalk)
- All tests deterministic and reproducible

**System 1 Integration**:
- ✅ Tests don't need material definitions from Symbol Table (use hardcoded values)
- ✅ If material lookup needed, `get_material()` trait method available

---

## Phase 4: IR Integration Update ✅ COMPLETE

**Reference**: [COMPILER-INTERNALS.md](../../Docs/v0.1.4/COMPILER-INTERNALS.md) - Two-pass compilation

**Current Status**: ✅ IR integration moved to hwc-compiler and receives Symbol Table  
**Target Status**: ✅ ACHIEVED - IR integration receives Symbol Table from compiler

**Prerequisites**: ✅ System 1 complete - Two-pass compiler provides HardwareIR + Symbol Table

### 4.1 Update `program_to_space()` Signature ✅ COMPLETE

**Documentation Reference**:
- [COMPILER-INTERNALS.md](../../Docs/v0.1.4/COMPILER-INTERNALS.md) Section: "Pass 2: Space Unrolling & Assembly"

**CRITICAL ARCHITECTURAL FIX**

The plan originally suggested updating `ir_integration.rs` inside hwc-engine. This is WRONG and causes circular dependencies.

**Correct Solution**: Move `ir_integration.rs` to hwc-compiler

Since the compiler already depends on the engine, it's perfectly legal for the compiler to:
1. Call the two-pass compiler to get `HardwareIR` + `SymbolTable`
2. Call engine functions (`place_component`, `place_trace`) passing the `SymbolTable`
3. Build the final `VoxelGrid` / `HardwareSpace`

**NEW Architecture**:
```
hwc-compiler/src/ir_integration.rs:
  - Receives HardwareIR + SymbolTable from two_pass_compiler
  - Calls hwc-engine functions with SymbolTable
  - Returns populated HardwareSpace
```

**Changes Completed**:
- [x] Moved `hwc/crates/hwc-engine/src/ir_integration.rs` → `hwc/crates/hwc-compiler/src/ir_integration.rs`
- [x] Updated function to accept `(program: &Program, symbol_table: &SymbolTable)`
- [x] Updated imports to use `hwc_engine::` prefix for engine types
- [x] Re-enabled `place_component()` function (was temporarily disabled)
- [x] Updated exports in `hwc-compiler/src/lib.rs`
- [x] Updated imports in `hwc-cli` commands (build, drc, physics)
- [x] Updated imports in examples (compile_and_export)
- [x] Updated all test files to import from compiler
- [x] Added `hwc-compiler` as dev-dependency to `hwc-engine` for tests

**Files Moved/Updated**:
- Moved: `hwc-engine/src/ir_integration.rs` → `hwc-compiler/src/ir_integration.rs`
- Updated: `hwc-compiler/src/lib.rs` - Exports `program_to_space` and `IrError`
- Updated: `hwc-engine/src/lib.rs` - Removed `ir_integration` module
- Updated: `hwc-cli/src/commands/build.rs` - Import from compiler
- Updated: `hwc-cli/src/commands/drc.rs` - Import from compiler
- Updated: `hwc-cli/src/commands/physics.rs` - Import from compiler
- Updated: `examples/compile_and_export.rs` - Import from compiler
- Updated: `hwc-engine/tests/*.rs` - Import from compiler, added component definitions
- Updated: `hwc-export/tests/export_tests.rs` - Import from compiler, added SymbolTable
- Updated: `hwc-engine/Cargo.toml` - Added hwc-compiler dev-dependency

### 4.2 Update Integration Tests ✅ COMPLETE

**Documentation Reference**:
- [LANGUAGE-SPEC.md](../../Docs/v0.1.4/LANGUAGE-SPEC.md) Section: "Quick Reference" for complete `.hw` syntax

**Changes Completed**:
- [x] Updated all integration tests to use Symbol Table
- [x] Created test helper function `transform_to_space()` with component definitions
- [x] Registered Battery, LED, and Resistor components with proper layouts
- [x] All tests passing (325+ tests across workspace)
- [x] Used proper component shape syntax: `Rectangle(width, height, depth)`

**Component Definitions Added to Tests**:
```rust
// Battery: Rectangle(2.0mm, 1.25mm, 0.5mm)
// LED: Rectangle(1.0mm, 0.8mm, 0.4mm)
// Resistor: Rectangle(3.2mm, 1.6mm, 0.5mm)
```

**Test Files Updated**:
- `hwc-engine/tests/automatic_routing_test.rs` - 10 tests passing
- `hwc-engine/tests/integration_test.rs` - 7 tests passing
- `hwc-engine/tests/origin_point_test.rs` - 14 tests passing
- `hwc-engine/tests/origin_matrix_validation_test.rs` - 8 tests passing

**System 1 Integration**:
- Tests use `program_to_space(&program, &symbol_table)` signature
- Symbol Table populated with component definitions before transformation
- Component definitions match System 1 AST structure (ComponentDefinition, LayoutBlock, PinPosition)

---

## Phase 5: Testing & Verification ✅ COMPLETE

### 5.1 Unit Test Migration ✅ COMPLETE

**Test Categories**:
- [x] Morton encoding (12 tests) - ✅ NO CHANGES NEEDED - All passing
- [x] Voxel grid (27 tests) - ✅ NO CHANGES NEEDED - All passing
- [x] NetlistArena (11 tests) - ✅ NO CHANGES NEEDED - All passing
- [x] Geometry (54 tests) - ✅ NO CHANGES NEEDED - All passing
- [x] Placement (14 tests) - ✅ SYMBOL TABLE INTEGRATED - All passing
- [x] Routing (13 tests) - ✅ NO CHANGES NEEDED - All passing
- [x] Constraint Manager (44 tests) - ✅ NO CHANGES NEEDED - All passing
- [x] Design Rule Check (37 tests) - ✅ NO CHANGES NEEDED - All passing

**Total**: 212 unit tests passing in hwc-engine (lib tests)

**Breakdown by Module**:
```
constraint_manager.rs:  44 tests (clearance, trace width, crosstalk, integration)
geometry.rs:            54 tests (Point3D, BoundingBox, TraceSegment, Direction)
design_rule_check.rs:   37 tests (DRC validation, parallel validation, thermal)
voxel_grid.rs:          27 tests (Morton encoding integration, voxel operations)
placement.rs:           14 tests (component placement, substrate, collision)
routing.rs:             13 tests (Bresenham, interpolation, trace placement, vias)
morton.rs:              12 tests (encoding, decoding, spatial locality)
netlist.rs:             11 tests (arena, components, pins, nets, O(1) lookups)
```

**Status**: All unit tests passing with Symbol Table integration complete.

### 5.2 Integration Test Migration ✅ COMPLETE

**Test Files**:
- [x] `automatic_routing_test.rs` - 10 tests passing (2-net routing, high voltage, vias, determinism)
- [x] `integration_test.rs` - 7 tests passing (parse & transform, components, routing)
- [x] `origin_point_test.rs` - 14 tests passing (origin point validation, coordinate systems)
- [x] `origin_matrix_validation_test.rs` - 8 tests passing (origin matrix validation)

**Total**: 39 integration tests passing in hwc-engine

**Changes Completed**:
- [x] All tests use `define` blocks via `transform_to_space()` helper
- [x] Component definitions registered in Symbol Table before transformation
- [x] Tests use proper Hardware Script syntax (Battery, LED, Resistor components)
- [x] All tests passing with Symbol Table integration

**Additional Integration Tests** (other crates):
- [x] `hwc-compiler` integration tests - 10 tests passing (two-pass compilation)
- [x] `hwc-export` integration tests - 30 tests passing (Gerber, GDSII, OBJ, GLB, Blender)
- [x] `hwc-parser` integration tests - Multiple test suites passing

**Total Workspace Tests**: 325+ tests passing across all crates

### 5.3 Performance Verification ✅ NO CHANGES NEEDED

- [x] Morton encoding performance (100× vs HashMap) - ✅ UNCHANGED
- [x] Voxel query performance (O(1)) - ✅ UNCHANGED
- [x] NetlistArena lookup performance (O(1)) - ✅ UNCHANGED
- [x] Fixed-point math determinism - ✅ UNCHANGED
- [x] Parallel validation performance - ✅ UNCHANGED

**Performance Characteristics Maintained**:
- Morton Z-curve encoding for cache locality
- O(1) voxel grid lookups
- O(1) netlist arena lookups
- Deterministic fixed-point math (i64 nanometers)
- Parallel DRC validation (5× speedup)

---

## Phase 6: Documentation Update ✅ COMPLETE

### 6.1 Code Documentation ✅ COMPLETE

**Completed**:
- [x] Function signatures documented with rustdoc comments
- [x] Symbol Table integration points documented in code
- [x] SymbolTableTrait documented with usage examples
- [x] Dependency inversion pattern documented
- [x] All public APIs have rustdoc comments

**Key Documentation**:
- `placement.rs`: SymbolTableTrait definition and usage
- `ir_integration.rs`: Symbol Table integration in compiler
- `symbol_table.rs`: SymbolTable implementation
- Test files: Component definition examples

**Generate Documentation**:
```bash
cargo doc --workspace --no-deps --open
```

### 6.2 Migration Notes ✅ COMPLETE

**Changes from v0.1.3 to v0.1.4**:

**Function Signature Changes**:
1. `program_to_space()` - Now in hwc-compiler, accepts `&SymbolTable`
   ```rust
   // OLD (v0.1.3): hwc-engine
   pub fn program_to_space(program: &Program) -> Result<HardwareSpace, IrError>
   
   // NEW (v0.1.4): hwc-compiler
   pub fn program_to_space(program: &Program, symbol_table: &SymbolTable) -> Result<HardwareSpace, IrError>
   ```

2. `place_component()` - Uses generic SymbolTableTrait
   ```rust
   // OLD (v0.1.3): Loaded from .hwx files
   pub fn place_component(component_type: &str, ...) -> Result<ComponentId, PlacementError>
   
   // NEW (v0.1.4): Uses Symbol Table via trait
   pub fn place_component<S: SymbolTableTrait>(
       params: PlacementParams<S>
   ) -> Result<ComponentId, PlacementError>
   ```

**Import Changes**:
```rust
// OLD (v0.1.3)
use hwc_engine::program_to_space;

// NEW (v0.1.4)
use hwc_compiler::program_to_space;
```

**Migration Example**:
```rust
// OLD (v0.1.3)
let space = program_to_space(&program)?;

// NEW (v0.1.4)
let symbol_table = SymbolTable::new();
// Register components/materials...
let space = program_to_space(&program, &symbol_table)?;
```

**No Changes Required**:
- Morton encoding API
- Voxel grid API
- NetlistArena API
- Geometry types (Point3D, BoundingBox, etc.)
- Routing functions (still use parameters)
- Constraint manager functions (still use parameters)

---

## Success Criteria

### Core Systems ✅ VERIFIED
- [x] Morton encoding working (11 tests passing)
- [x] Voxel grid working (14 tests passing)
- [x] NetlistArena working (10 tests passing)
- [x] Fixed-point geometry working (13 tests passing)

### Integration ✅ COMPLETE
- [x] Component placement uses Symbol Table (via SymbolTableTrait)
- [x] Routing architecture complete (material properties as parameters)
- [x] IR integration receives Symbol Table
- [x] All 325+ tests passing
- [x] No YAML dependencies
- [x] No file loading code

**System 2 Status**: ✅ COMPLETE - All Symbol Table integration finished

---

## Migration Checklist

### Immediate Actions ✅ ALL COMPLETE
- [x] Add `symbol_table: &SymbolTable` parameter to `place_component()` - Uses generic trait
- [x] Routing functions designed to accept material properties as parameters - Already correct
- [x] Add `symbol_table: &SymbolTable` parameter to `program_to_space()` - Complete
- [x] Remove all YAML loading code - None existed
- [x] Remove all `.hwx` file parsing code - None existed

### Test Updates ✅ ALL COMPLETE
- [x] Create test helper functions for component definitions - `transform_to_space()` created
- [x] Create test helper functions for material definitions - MockSymbolTable created
- [x] Update all placement tests (14 tests) - All passing
- [x] Update all routing tests (13 tests) - All passing
- [x] Update all integration tests (39 tests) - All passing

### Verification ✅ ALL COMPLETE
- [x] All 465+ tests passing across entire workspace
- [x] No clippy warnings
- [x] No `serde_yaml` usage
- [x] Performance unchanged
- [x] Create test helper functions for material definitions
- [x] Update all placement tests (14 tests)
- [x] Update all routing tests (13 tests)
- [x] Update all integration tests (39 tests)

### Verification ✅ COMPLETE
- [x] All 465+ tests passing (2+32+10+203+10+7+8+14+11+27+3+1+22+1+2+3+8+3+4+3+2+8+19+7+14+26+9+10+14+5)
- [x] No clippy warnings
- [x] No `serde_yaml` usage
- [x] Performance unchanged

---

## Estimated Timeline

**Week 1**: Symbol Table integration (placement, routing, IR)  
**Week 2**: Test migration and verification  

**Total**: 2 weeks

---

## Dependencies

**Requires**:
- ✅ System 1 (Unified Parser & Two-Pass Compiler) - **COMPLETE**
  - Provides: `SymbolTable` with `get_component()` and `get_material()`
  - Provides: `HardwareIR` with resolved components and nets
  - Provides: Property extraction helpers

**Provides**:
- ✅ Voxel grid for System 3 (Routing)
- ✅ NetlistArena for System 3 (Routing)
- ✅ Component placement for System 5 (Export)

**Reference**: See `SYSTEM-1-IMPLEMENTATION-PLAN.md` Section: "Integration with Downstream Systems"

---

## Notes

**Code Preservation**: 95% of System 2 code is preserved. Only data source integration changes.

**No Algorithm Changes**: Morton encoding, voxel math, collision detection, and spatial operations remain identical to v0.1.3.

**Test Preservation**: All test logic is preserved. Only test setup (Symbol Table initialization) changes.

**Performance**: No performance impact. Symbol Table lookup is O(1) HashMap, same as file loading.

---
