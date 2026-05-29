# System 3: Routing Engine - Implementation Plan

**System**: Routing Engine (3-Phase Pipeline)  
**Status**: ✅ ALGORITHMS COMPLETE (v0.1.3), ⚠️ NEEDS SYMBOL TABLE INTEGRATION  
**Migration Strategy**: **PRESERVE & ADAPT** (90% code preservation)  
**Based On**: v0.1.4 Documentation + v0.1.3 Implementation  
**Prerequisites**: System 1 (Parser) ✅ **COMPLETE**, System 2 (Voxel Engine) ⏭️ **PENDING**  
**Last Updated**: March 22, 2026

---

## Overview

The routing engine implements the 3-Phase Routing Pipeline (Constraint Manager → Geometry Router → Design Rule Check). The core algorithms (A* pathfinding, Manhattan routing, deterministic VecDeque) are **architecture-agnostic** and remain unchanged from v0.1.3.

**What Changes**: Data source integration (Symbol Table for materials/profiles instead of YAML)  
**What Stays**: All routing algorithms, pathfinding logic, DRC validation

**Architecture (Unchanged from v0.1.3)**:
- Phase 1: Constraint Manager (translates physics to geometry)
- Phase 2: Geometry Router (Manhattan routing with A* pathfinding)
- Phase 3: Design Rule Check (validates against constraints)
- Deterministic: VecDeque-based pathfinding for reproducibility
- Dependencies: None (pure algorithms)

**v0.1.4 Integration Points**:
- ✅ Material properties: Load from Symbol Table (System 1 provides `get_material()`)
- ✅ Profile constraints: Load from Symbol Table (System 1 provides `get_profile()`)
- ✅ Native SI unit values: No manual conversion needed (System 1 handles this)
- ✅ No changes to pathfinding, Manhattan routing, or DRC algorithms

**What System 1 Provides**:
- `SymbolTable::get_material(&str) -> Result<&MaterialDef, CompileError>`
- `SymbolTable::get_profile(&str) -> Result<&ProfileDef, CompileError>`
- Property extraction helpers for material properties
- Constraint conversion helpers (profile → ConstraintSet)

**Reference**: See `SYSTEM-1-IMPLEMENTATION-PLAN.md` Section: "Integration with Downstream Systems"

---

## Migration Strategy: Preserve & Adapt

### ✅ **Keep 100% (No Changes)**:
- `src/geometry_router.rs` - A* pathfinding, Manhattan routing
- `src/deterministic_routing.rs` - VecDeque-based routing
- `src/manhattan_rules.rs` - Layer direction enforcement
- All routing algorithms and pathfinding logic

### ⚠️ **Adapt (Minimal Changes)**:
- `src/constraint_manager.rs` - Load materials/profiles from Symbol Table
- `src/design_rule_check.rs` - Load constraints from Symbol Table

### ❌ **Remove**:
- YAML parsing for materials (if exists)
- YAML parsing for profiles (if exists)
- File loading code

---

## Phase 1: Constraint Manager Integration ✅ COMPLETE (1.1 ✅, 1.2 ✅, 1.3 ✅)

**Reference**: [ROUTING-AND-PHYSICS.md](../../Docs/v0.1.4/ROUTING-AND-PHYSICS.md) - Constraint generation

**Current Status**: ✅ Material and profile loading complete, all tests passing
**Previous Status**: Loaded from YAML (v0.1.3)

**Prerequisites**: ✅ System 1 complete - Symbol Table available

### 1.1 Update Material Property Loading ✅ COMPLETE

**Documentation Reference**: 
- [ROUTING-AND-PHYSICS.md](../../Docs/v0.1.4/ROUTING-AND-PHYSICS.md) Section: "Translation 1: Dielectric Breakdown to Clearance"
- [GENERIC-MEASUREMENT-ARCHITECTURE.md](../../Docs/v0.1.4/GENERIC-MEASUREMENT-ARCHITECTURE.md) Section: "Unit Classification at Parse Time"

**Implementation (v0.1.4)**:
```rust
// hwc/crates/hwc-engine/src/constraint_manager/manager.rs

/// Trait for Symbol Table access (dependency inversion)
pub trait SymbolTableTrait {
    fn get_material(&self, name: &str) -> Result<&MaterialDefinition, String>;
}

/// Extract dielectric strength from material definition
fn extract_dielectric_strength(material: &MaterialDefinition) -> Result<f64, String> {
    for prop in &material.properties {
        if prop.key == "dielectric_strength" {
            match &prop.value {
                PropertyValue::Measurement(measurement) => {
                    return match &measurement.unit {
                        Unit::Custom(unit_str) if unit_str == "kV/mm" => Ok(measurement.value),
                        Unit::Custom(unit_str) if unit_str == "V/mm" => Ok(measurement.value / 1000.0),
                        Unit::Custom(unit_str) if unit_str == "MV/mm" => Ok(measurement.value * 1000.0),
                        _ => Err(format!("Unsupported unit: {:?}", measurement.unit)),
                    };
                }
                _ => return Err("dielectric_strength must be a measurement".to_string()),
            }
        }
    }
    Err(format!("Material '{}': missing dielectric_strength", material.name))
}

/// Generate constraints using Symbol Table
pub fn generate_net_constraints<S: SymbolTableTrait>(
    &self,
    net: &NetData,
    voltage_mv: i64,
    current_ma: i64,
    material_name: &str,
    symbol_table: &S,  // Generic parameter (dependency inversion)
    is_external: bool,
) -> Result<RouteConstraints, String> {
    let material_def = symbol_table.get_material(material_name)?;
    let dielectric_strength = extract_dielectric_strength(material_def)?;
    
    let min_clearance_nm = calculate_clearance_nm(voltage_mv, dielectric_strength, self.safety_factor);
    let min_trace_width_nm = calculate_trace_width_nm(current_ma, self.default_temp_rise_c, is_external);
    
    Ok(RouteConstraints {
        min_trace_width_nm: min_trace_width_nm.max(net.width_nm),
        min_clearance_nm,
        max_parallel_length_nm: self.default_max_parallel_nm,
        max_resistance_ohm: 100.0,
        max_current_ma: current_ma,
        impedance_ohm: None,
    })
}
```

**Changes Completed**:
- [x] Added `SymbolTableTrait` in hwc-engine (dependency inversion)
- [x] Implemented trait in hwc-compiler `SymbolTable`
- [x] Updated `generate_net_constraints()` to use generic trait parameter
- [x] Updated `generate_clearance_zone()` to use generic trait parameter
- [x] Added `extract_dielectric_strength()` helper function
- [x] Removed YAML dependencies (none existed in v0.1.3)
- [x] Handle `Unit::Custom(String)` for kV/mm, V/mm, MV/mm
- [x] Created `MockSymbolTable` for unit tests (46 tests passing)

**Estimated Impact**: 25 lines added, 0 lines removed (no YAML existed)

**Files Updated**:
- `hwc/crates/hwc-engine/src/constraint_manager/manager.rs` - Added trait and Symbol Table integration
- `hwc/crates/hwc-engine/src/constraint_manager/mod.rs` - Exported `SymbolTableTrait`
- `hwc/crates/hwc-engine/src/lib.rs` - Exported `SymbolTableTrait`
- `hwc/crates/hwc-compiler/src/symbol_table.rs` - Implemented trait

**System 1 Integration**:
- ✅ Use `SymbolTable::get_material()` provided by System 1
- ✅ Material properties already parsed with native SI units
- ✅ Property extraction handles kV/mm, V/mm, MV/mm units
- ✅ Dielectric strength values converted to kV/mm for calculations

**Testing Strategy**:
- ✅ Unit tests use `MockSymbolTable` (no circular dependency)
- ✅ Integration tests will use real `SymbolTable` from hwc-compiler
- ✅ Production code uses real `SymbolTable` passed from hwc-cli

### 1.2 Update Profile Constraint Loading ✅ COMPLETE

**Documentation Reference**:
- [ROUTING-AND-PHYSICS.md](../../Docs/v0.1.4/ROUTING-AND-PHYSICS.md) Section: "Translation 2: Current Capacity to Trace Width"
- [LANGUAGE-SPEC.md](../../Docs/v0.1.4/LANGUAGE-SPEC.md) Section: "2. Profile Definition"

**Implementation (v0.1.4)**:
```rust
// hwc/crates/hwc-engine/src/constraint_manager/manager.rs

/// Load fabrication constraints from profile definition.
pub fn load_fabrication_constraints<S: SymbolTableTrait>(
    &self,
    profile_name: &str,
    symbol_table: &S,
) -> Result<FabricationConstraints, String> {
    // Load profile from Symbol Table (v0.1.4 integration)
    let profile_def = symbol_table.get_profile(profile_name)?;
    
    // Extract trace constraints (required)
    let (min_trace_width_nm, min_trace_spacing_nm) = extract_trace_constraints(profile_def)?;
    
    // Extract via constraints (required)
    let (min_via_diameter_nm, min_annular_ring_nm) = extract_via_constraints(profile_def)?;
    
    // Extract clearance constraints (optional)
    let clearance = extract_clearance_constraints(profile_def)?;
    
    Ok(FabricationConstraints {
        min_trace_width_nm,
        min_trace_spacing_nm,
        min_via_diameter_nm,
        min_annular_ring_nm,
        high_voltage_clearance_nm: clearance.map(|(hv, _)| hv),
        safety_factor: clearance.map(|(_, sf)| sf).unwrap_or(2.0),
    })
}

/// Helper functions for profile extraction
fn measurement_to_nm(measurement: &Measurement) -> Result<i64, String> { /* ... */ }
fn extract_trace_constraints(profile: &ProfileDefinition) -> Result<(i64, i64), String> { /* ... */ }
fn extract_via_constraints(profile: &ProfileDefinition) -> Result<(i64, i64), String> { /* ... */ }
fn extract_clearance_constraints(profile: &ProfileDefinition) -> Result<Option<(i64, f64)>, String> { /* ... */ }
```

**Changes Completed**:
- [x] Added `get_profile()` to `SymbolTableTrait` in hwc-engine
- [x] Implemented trait method in hwc-compiler `SymbolTable`
- [x] Created `FabricationConstraints` type for profile data
- [x] Implemented `load_fabrication_constraints()` method
- [x] Added `measurement_to_nm()` helper function
- [x] Added `extract_trace_constraints()` helper function
- [x] Added `extract_via_constraints()` helper function
- [x] Added `extract_clearance_constraints()` helper function
- [x] Updated `MockSymbolTable` with test profiles
- [x] Added 8 new tests for profile loading (all passing)
- [x] Total: 54 tests passing (was 46)

**Estimated Impact**: 150 lines added, 0 lines removed (no YAML existed)

**Files Updated**:
- `hwc/crates/hwc-engine/src/constraint_manager/manager.rs` - Added profile loading and extraction helpers
- `hwc/crates/hwc-engine/src/constraint_manager/types.rs` - Added `FabricationConstraints` type
- `hwc/crates/hwc-engine/src/constraint_manager/mod.rs` - Exported `FabricationConstraints`
- `hwc/crates/hwc-compiler/src/symbol_table.rs` - Implemented `get_profile()` in trait

**System 1 Integration**:
- ✅ Use `SymbolTable::get_profile()` provided by System 1
- ✅ Profile constraints already parsed with native SI units
- ✅ Measurement conversion handles mm, cm, µm units
- ✅ All constraint values converted to nanometers for calculations

**Testing Strategy**:
- ✅ Unit tests use `MockSymbolTable` with PCB_Standard profile
- ✅ Tests verify trace, via, and clearance constraint extraction
- ✅ Tests verify measurement unit conversions (mm, cm, µm → nm)
- ✅ Integration tests will use real `SymbolTable` from hwc-compiler
- ✅ Production code uses real `SymbolTable` passed from hwc-cli

**Test Results**: ✅ 54/54 tests passing (8 new profile tests added)

### 1.3 Update Constraint Manager Tests ✅ COMPLETE

**Documentation Reference**:
- [LANGUAGE-SPEC.md](../../Docs/v0.1.4/LANGUAGE-SPEC.md) Section: "1. Material Definition" and "2. Profile Definition"

**Changes Completed**:
- [x] Updated all constraint generation tests to use `MockSymbolTable`
- [x] Created `MockSymbolTable` with FR4 and Air materials
- [x] Created test helpers for material definitions
- [x] Verified all 46 tests pass
- [x] Used `MaterialDefinition` AST structure from hwc-parser

**Test Results**: ✅ 46/46 tests passing
- Clearance tests: 6/6 passing
- Trace width tests: 9/9 passing
- Crosstalk tests: 8/8 passing
- Manager tests: 10/10 passing
- Types tests: 8/8 passing
- Determinism tests: 5/5 passing

**Testing Strategy**:
- ✅ Unit tests use `MockSymbolTable` (isolated, fast, no circular dependency)
- ✅ Mock implements `SymbolTableTrait` with test materials (FR4, Air)
- ✅ Integration tests will use real `SymbolTable` from hwc-compiler
- ✅ Production uses real `SymbolTable` passed from hwc-cli

**Why Mock for Unit Tests?**

The circular dependency ONLY appears during **test compilation**:
```
hwc-engine tests → need SymbolTable → from hwc-compiler → depends on hwc-engine → CIRCULAR!
```

When running `cargo test` on `hwc-engine`, the tests compile BEFORE `hwc-compiler` is fully compiled, so the trait implementation isn't visible yet.

**Solution**: Use `MockSymbolTable` in unit tests:
- ✅ Isolated unit tests (no compiler dependency)
- ✅ Fast compilation (don't compile hwc-compiler for every test)
- ✅ Clear test intent (mock shows exact material properties being tested)
- ✅ Real `SymbolTable` used in integration tests and production

**Production Architecture** (Clean, No Circular Dependency):
```
hwc-parser (AST) → hwc-compiler (SymbolTable) → hwc-engine (uses trait) → hwc-cli (orchestrates)
```

**System 1 Integration**:
- ✅ Test materials match System 1 AST structure (`MaterialDefinition`)
- ✅ Properties use `PropertyValue::Measurement` with `Unit::Custom`
- ✅ Can reuse definitions from `hwc/data/standard-materials.hw` in integration tests

---

## Phase 2: Geometry Router (No Changes) ✅ VERIFIED

**Reference**: [ROUTING-AND-PHYSICS.md](../../Docs/v0.1.4/ROUTING-AND-PHYSICS.md) - Manhattan routing

**Status**: ✅ All routing algorithms are architecture-agnostic

### 2.1 Manhattan Routing ✅ NO CHANGES NEEDED
- [x] Layer direction enforcement working
- [x] `is_valid_move()` function working
- [x] `assign_layer_directions()` working
- [x] No changes needed for v0.1.4

### 2.2 A* Pathfinding ✅ NO CHANGES NEEDED
- [x] Deterministic VecDeque-based pathfinding working
- [x] `route_net_deterministic()` function working
- [x] `get_neighbors_stable()` function working
- [x] Heuristic calculations working
- [x] No changes needed for v0.1.4

### 2.3 Collision Detection ✅ NO CHANGES NEEDED
- [x] `is_voxel_available()` working
- [x] `check_clearance_violation()` working
- [x] `mark_route_occupied()` working
- [x] No changes needed for v0.1.4

**Status**: ✅ All routing algorithms verified, no migration needed

---

## Phase 3: Design Rule Check Integration ✅ COMPLETE

**Reference**: [ROUTING-AND-PHYSICS.md](../../Docs/v0.1.4/ROUTING-AND-PHYSICS.md) - DRC validation

**Current Status**: ✅ DRC validators now use fabrication constraints from Symbol Table  
**Target Status**: ✅ ACHIEVED - DRC loads constraints from Symbol Table

**Prerequisites**: ✅ System 1 complete - Symbol Table available, ✅ Phase 1 complete - Fabrication constraints available

### 3.1 Update Clearance Validation ✅ COMPLETE

**Documentation Reference**:
- [ROUTING-AND-PHYSICS.md](../../Docs/v0.1.4/ROUTING-AND-PHYSICS.md) Section: "Translation 1: Dielectric Breakdown to Clearance"

**Implementation (v0.1.4)**:
```rust
// hwc/crates/hwc-engine/src/design_rule_check/clearance.rs

pub fn validate_clearances(
    nets: &[NetVoxels],
    constraints: &ConstraintRulebook,
) -> Vec<DrcViolation> {
    // Get required clearance from fabrication constraints (v0.1.4)
    let required_clearance_nm = constraints
        .fabrication
        .as_ref()
        .and_then(|fab| fab.high_voltage_clearance_nm)
        .or_else(|| {
            constraints
                .fabrication
                .as_ref()
                .map(|fab| fab.min_trace_spacing_nm)
        })
        .unwrap_or(200_000); // Default 0.2mm
    
    // Check each pair of nets for clearance violations
    // ...
}
```

**Changes Completed**:
- [x] Added `fabrication` field to `ConstraintRulebook`
- [x] Added `set_fabrication_constraints()` method to `ConstraintRulebook`
- [x] Updated `validate_clearances()` to use fabrication constraints
- [x] Clearance uses `high_voltage_clearance_nm` if available, falls back to `min_trace_spacing_nm`
- [x] Added 2 new tests for fabrication constraint usage
- [x] All tests passing (41 DRC tests)

**Estimated Impact**: 15 lines changed

**Files Updated**:
- `hwc/crates/hwc-engine/src/constraint_manager/types.rs` - Added fabrication field to ConstraintRulebook
- `hwc/crates/hwc-engine/src/design_rule_check/clearance.rs` - Updated to use fabrication constraints

**System 1 Integration**:
- ✅ Uses `FabricationConstraints` from Phase 1.2
- ✅ Constraints already in nanometers (no conversion needed)
- ✅ Fallback to defaults if constraints not provided

### 3.2 Update Trace Width Validation ✅ COMPLETE

**Documentation Reference**:
- [ROUTING-AND-PHYSICS.md](../../Docs/v0.1.4/ROUTING-AND-PHYSICS.md) Section: "Translation 2: Current Capacity to Trace Width"

**Implementation (v0.1.4)**:
```rust
// hwc/crates/hwc-engine/src/design_rule_check/trace_width.rs

pub fn validate_trace_widths(
    nets: &[NetVoxels],
    constraints: &ConstraintRulebook,
    voxel_size_nm: i64,
) -> Vec<DrcViolation> {
    // Get required trace width from fabrication constraints (v0.1.4)
    let required_width_nm = constraints
        .fabrication
        .as_ref()
        .map(|fab| fab.min_trace_width_nm)
        .unwrap_or(100_000); // Default 0.1mm
    
    // Check each net for trace width violations
    // ...
}
```

**Changes Completed**:
- [x] Updated `validate_trace_widths()` to use fabrication constraints
- [x] Trace width uses `min_trace_width_nm` from profile
- [x] Added 2 new tests for fabrication constraint usage
- [x] All tests passing (41 DRC tests)

**Estimated Impact**: 10 lines changed

**Files Updated**:
- `hwc/crates/hwc-engine/src/design_rule_check/trace_width.rs` - Updated to use fabrication constraints

**System 1 Integration**:
- ✅ Uses `FabricationConstraints` from Phase 1.2
- ✅ IPC-2221 formula remains unchanged (universal constant)
- ✅ Only data loading changes (constraints from Symbol Table)

### 3.3 Update Thermal Validation ✅ NO CHANGES NEEDED

**Documentation Reference**:
- [ROUTING-AND-PHYSICS.md](../../Docs/v0.1.4/ROUTING-AND-PHYSICS.md) Section: "The 3-Phase Routing Pipeline"

**Status**: ✅ Thermal validation already uses constraints from ConstraintRulebook

**Current Implementation**:
```rust
// hwc/crates/hwc-engine/src/design_rule_check/thermal.rs

pub fn validate_thermal(
    nets: &[NetVoxels],
    constraints: &ConstraintRulebook,
    material: &MaterialProperties,
    voxel_size_nm: i64,
) -> Vec<DrcViolation> {
    // Already uses constraints.default_current_ma
    let current_ma = constraints.default_current_ma.unwrap_or(20);
    
    // Already uses constraints.max_temp_rise_c
    let max_temp_rise = constraints.max_temp_rise_c.unwrap_or(10.0);
    
    // Material properties passed separately (correct - from material definitions, not profiles)
    // ...
}
```

**Why No Changes Needed**:
- Thermal validation already reads from `ConstraintRulebook`
- Material properties come from material definitions (not profiles)
- Current and temperature limits are already configurable via constraints

**Estimated Impact**: 0 lines changed

**System 1 Integration**:
- ✅ Material properties from `MaterialDefinition` (Phase 1.1)
- ✅ Thermal limits from `ConstraintRulebook`
- ✅ No profile-specific thermal constraints needed

### 3.4 Update DRC Tests ✅ COMPLETE

**Documentation Reference**:
- [LANGUAGE-SPEC.md](../../Docs/v0.1.4/LANGUAGE-SPEC.md) Section: "2. Profile Definition"

**Changes Completed**:
- [x] Added 4 new tests for fabrication constraint usage
- [x] Test clearance validation with fabrication constraints
- [x] Test clearance validation with high voltage clearance
- [x] Test trace width validation with fabrication constraints
- [x] Test trace width validation passes with sufficient width
- [x] All 41 DRC tests passing (was 37)

**Test Results**: ✅ 41/41 tests passing (4 new tests)

**Estimated Impact**: 80 lines added (test code)

**System 1 Integration**:
- ✅ Tests use `FabricationConstraints` directly
- ✅ Tests verify constraints are read from rulebook
- ✅ Tests verify fallback to defaults when constraints not provided

---

## Phase 4: Parallel Validation (No Changes) ✅ VERIFIED

**Reference**: [ROUTING-AND-PHYSICS.md](../../Docs/v0.1.4/ROUTING-AND-PHYSICS.md) - Parallel validation

**Status**: ✅ Parallel validation is architecture-agnostic

### 4.1 Rayon Integration ✅ NO CHANGES NEEDED
- [x] `validate_physics_parallel()` working
- [x] Read-only validation working
- [x] Deterministic results verified
- [x] 5× performance improvement verified
- [x] No changes needed for v0.1.4

**Status**: ✅ Parallel validation verified, no migration needed

---

## Phase 5: Testing & Verification ✅ COMPLETE

### 5.1 Unit Test Migration ✅ COMPLETE

**Test Categories**:
- [x] Manhattan routing (tests) - ✅ NO CHANGES NEEDED
- [x] A* pathfinding (tests) - ✅ NO CHANGES NEEDED
- [x] Deterministic routing (tests) - ✅ NO CHANGES NEEDED
- [x] Constraint generation (tests) - ✅ USING MOCK SYMBOL TABLE
- [x] DRC validation (tests) - ✅ USING MOCK SYMBOL TABLE

**Test Results**: ✅ 217/217 unit tests passing

### 5.2 Integration Test Migration ✅ COMPLETE

- [x] Complete routing pipeline test (automatic_routing_test.rs)
- [x] Clearance violation test (integration_test.rs)
- [x] Trace width violation test (integration_test.rs)
- [x] Origin matrix validation (origin_matrix_validation_test.rs)
- [x] Origin point validation (origin_point_test.rs)
- [x] All integration tests verified passing

**Test Results**: ✅ 39/39 integration tests passing
- automatic_routing_test: 10/10 ✅
- integration_test: 7/7 ✅
- origin_matrix_validation_test: 8/8 ✅
- origin_point_test: 14/14 ✅

### 5.3 Performance Verification ✅ COMPLETE

- [x] Deterministic routing (same input → same output) - ✅ UNCHANGED
- [x] Parallel validation (5× speedup) - ✅ UNCHANGED
- [x] A* pathfinding performance - ✅ UNCHANGED

**Total Test Results**: ✅ 256/256 tests passing (217 unit + 39 integration)

## Phase 6: Documentation Update ❌ NOT STARTED

### 6.1 Code Documentation ❌ NOT STARTED

- [ ] Update rustdoc for functions with new signatures
- [ ] Document Symbol Table integration
- [ ] Add examples using Symbol Table

### 6.2 Migration Notes ❌ NOT STARTED

- [ ] Document changes from v0.1.3 to v0.1.4
- [ ] List function signature changes
- [ ] Provide migration examples

