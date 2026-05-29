# System 4 Implementation Plan: Materials & Physics Validation

**Hardware Script v0.1.2**  
**Focus**: Physics Solvers and Material-Based Validation  
**Priority**: HIGH - Validates routing against real-world physics  
**Created**: March 19, 2026

---

## ⚠️ CRITICAL: Data Sources - NO HARDCODING ALLOWED

**This system MUST use these data sources:**

### Primary Data Sources (ALWAYS use these):
1. **`hwc/data/standard-materials.yaml`** - ALL material properties
   - Conductor properties: resistivity, thermal conductivity, max current density
   - Insulator properties: dielectric strength, relative permittivity, max operating temp
   - Semiconductor properties: band gap, mobility, thermal properties
   - **NEVER hardcode these values in tests or implementation**

2. **`hwc/data/profiles/*.hwp`** - Fabrication constraint profiles
   - `standard-pcb.hwp` - IPC-2221 compliant PCB constraints
   - `standard-asic.hwp` - Typical foundry rules for silicon
   - `high-voltage.hwp` - Increased clearances for high voltage
   - `prototype.hwp` - Relaxed constraints for hobbyists

### How to Use Data Sources:
```rust
// ✅ CORRECT: Load from database
let db = MaterialDatabase::load_standard()?;
let copper = db.get_conductor("copper")?;
let resistivity = copper.resistivity_ohm_m;

// ❌ WRONG: Hardcoded values
let resistivity = 1.68e-8; // DON'T DO THIS!
```

### Testing Requirements:
- ALL tests MUST load material database: `MaterialDatabase::load_standard()`
- ALL tests MUST use real material properties from database
- NO magic numbers for material properties
- If a property doesn't exist, add it to `standard-materials.yaml`

---

## Executive Summary

System 4 implements comprehensive physics validation that ensures routed designs meet electrical, thermal, and electromagnetic requirements. This system builds on the material database (System 2) and routing engine (System 3) to provide real-world physics analysis.

**The 4 Physics Domains**:
```
1. Electrical Analysis (resistance, voltage drop, ampacity)
2. Thermal Analysis (temperature rise, heat dissipation)
3. Electromagnetic Analysis (impedance, signal integrity)
4. Clearance Validation (dielectric breakdown)
```

---

## Prerequisites (Already Complete)

✅ **System 1: Parser & AST** (Complete)
✅ **System 2: Materials & Constraints** (Complete)
✅ **System 3: Routing Engine** (Complete)
✅ **hwc-materials crate** (Material database with YAML loading)
✅ **hwc-physics crate** (Stub implementations for analyzers)

---

## Phase 1: Electrical Analysis Implementation ✅ COMPLETE

**Purpose**: Validate resistance, voltage drop, and current capacity

**Location**: `hwc/crates/hwc-physics/src/electrical.rs`

**Test Results**: ✅ 12/12 electrical tests passing (using real material database)

---

## Phase 2: Thermal Analysis Implementation ✅ COMPLETE

**Purpose**: Validate temperature rise and thermal safety

**Location**: `hwc/crates/hwc-physics/src/thermal.rs`

**Test Results**: ✅ 14/14 thermal tests passing (using real material database)

---

## Phase 3: Electromagnetic Analysis Implementation ✅ COMPLETE

**Purpose**: Validate impedance and signal integrity

**Location**: `hwc/crates/hwc-physics/src/electromagnetic.rs`

### Phase 3.1: Impedance Calculation ✅

- [x] Implement `calculate_microstrip_impedance()` function
- [x] Add unit tests

### Phase 3.2: Impedance Matching Validation ✅

- [x] Implement `validate_impedance_matching()` function
- [x] Add unit tests

### Phase 3.3: Crosstalk Analysis ✅

- [x] Implement `calculate_crosstalk_coefficient()` function
- [x] Implement `validate_crosstalk()` function
- [x] Add unit tests

**Test Results**: ✅ 14/14 electromagnetic tests passing (using real material database and stackup profiles)

**Data Sources Used**:
- ✅ `hwc/data/standard-materials.hwmat` - Material properties (FR4, copper, aluminum, air)
- ✅ `hwc/data/profiles/standard-2layer-stackup.hwp` - PCB stackup for impedance calculations
- ✅ `hwc/data/profiles/standard-4layer-stackup.hwp` - 4-layer stackup with impedance control

---

## Phase 4: Clearance Validation Enhancement ✅ COMPLETE

**Purpose**: Enhance existing clearance validation with physics

**Location**: `hwc/crates/hwc-physics/src/clearance.rs`

### Phase 4.1: Voltage-Based Clearance ✅

- [x] Implement `calculate_required_clearance()` function
  - Uses material dielectric strength from database
  - Calculates required clearance based on voltage and material
  - Applies 2× safety factor (industry standard)
  - Formula: clearance = (voltage / dielectric_strength) × safety_factor

- [x] Implement `validate_clearance()` function
  - Compares actual clearance with required clearance
  - Returns violation if insufficient
  - Includes detailed error information

- [x] Add unit tests
  - Test: 120V through air (3kV/mm) → 0.08mm required ✅
  - Test: 120V through FR4 (20kV/mm) → 0.012mm required ✅
  - Test: High voltage (1000V) scenarios ✅
  - Test: Low voltage (3.3V) scenarios ✅
  - Test: FR4 vs air comparison ✅

### Phase 4.2: Altitude and Environmental Factors ✅

- [x] Implement `adjust_clearance_for_altitude()` function
  - Input: base clearance, altitude_m
  - Air thins at altitude → lower dielectric strength
  - Formula: clearance_adjusted = clearance × (1 + altitude/10000)
  - Output: adjusted clearance

- [x] Implement `validate_clearance_with_altitude()` function
  - Validates clearance with altitude adjustment
  - Returns altitude adjustment violation if needed

- [x] Add unit tests
  - Test: Sea level → no adjustment ✅
  - Test: 3000m altitude → 30% increase in clearance ✅
  - Test: 5000m altitude → 50% increase ✅
  - Test: 10000m altitude → 100% increase (double) ✅

**Test Results**: ✅ 19/19 clearance tests passing (using real material database)

**Data Sources Used**:
- ✅ Material dielectric strength values (air: 3 kV/mm, FR4: 20 kV/mm)
- ✅ Voltage-based clearance calculations
- ✅ Environmental factors (altitude adjustment)

---

## Phase 5: Physics Engine Integration ✅ COMPLETE

**Purpose**: Integrate all physics analyzers into unified engine

**Location**: `hwc/crates/hwc-physics/src/lib.rs`

### Phase 5.1: Unified Physics Validation ✅

- [x] Implement `PhysicsEngine::validate_design()` method
  - Input: routed board, material database, constraints
  - Run all 4 physics analyzers
  - Collect all violations
  - Output: comprehensive physics report

- [x] Add integration tests
  - Test: Valid design passes all checks ✅
  - Test: High current violation detected ✅
  - Test: Thermal violation detected ✅
  - Test: Impedance mismatch detected ✅
  - Test: Clearance violation detected ✅

### Phase 5.2: Physics Report Generation ✅

- [x] Create `PhysicsReport` struct
  - electrical_violations: Vec<ElectricalViolation> ✅
  - thermal_violations: Vec<ThermalViolation> ✅
  - em_violations: Vec<EMViolation> ✅
  - clearance_violations: Vec<ClearanceViolation> ✅

- [x] Implement `format_report()` method
  - Human-readable output ✅
  - Grouped by violation type ✅
  - Include suggestions for fixes ✅

**Test Results**: ✅ 17/17 integration tests passing + 59 analyzer tests = 76 total tests passing

---

## Phase 6: Error Code Integration ✅ COMPLETE

**Purpose**: Map physics violations to error codes

**Location**: `hwc/crates/hwc-physics/src/error_mapping.rs`

### Phase 6.1: Electrical Error Codes ✅

- [x] Implement P20: VOLTAGE_DROP_TOO_HIGH ✅
- [x] Implement P21: TRACE_TOO_THIN (enhanced) ✅
- [x] Implement P23: RESISTANCE_TOO_HIGH ✅

### Phase 6.2: Thermal Error Codes ✅

- [x] Implement P22: COMPONENT_OVERHEATING (enhanced) ✅
- [x] Implement P24: TEMPERATURE_RISE_EXCEEDS_LIMIT (enhanced) ✅
- [x] Implement P25: THERMAL_CLUSTERING ✅

### Phase 6.3: Electromagnetic Error Codes ✅

- [x] Implement P31: IMPEDANCE_MISMATCH (enhanced) ✅
- [x] Implement P32: CROSSTALK_RISK (enhanced) ✅
- [x] Implement P34: SIGNAL_INTEGRITY_VIOLATION (enhanced) ✅

**Additional Features**:
- [x] Created `PhysicsError` struct with code, message, and suggestion
- [x] Implemented conversion functions for all violation types
- [x] Added `PhysicsReport::to_errors()` method for batch conversion
- [x] All error messages include actionable suggestions

**Test Results**: ✅ 8/8 error mapping tests + 4 integration tests = 12 tests passing

---

## Phase 7: Material Database Integration

**Purpose**: Connect physics solvers to material database

**Location**: `hwc/crates/hwc-physics/src/lib.rs`

### Phase 7.1: Material Property Lookup

- [ ] Implement `PhysicsEngine::load_materials()` method
  - Load material database from YAML
  - Cache material properties for fast lookup
  - Support custom material databases

- [ ] Add unit tests
  - Test: Load standard materials
  - Test: Load custom materials
  - Test: Material not found error

### Phase 7.2: Dynamic Property Calculation ✅

- [x] Implement temperature-dependent properties
  - Resistivity changes with temperature
  - Thermal conductivity changes with temperature
  - Use material database temperature coefficients

- [x] Add unit tests (10 tests passing)
  - Test: Copper resistivity at 25°C vs 100°C
  - Test: FR4 properties at different temperatures
  - Test: Aluminum and gold temperature coefficients
  - Test: Resistance calculations at different temperatures
  - Test: Materials without temperature coefficients

---

## Phase 8: Parallel Physics Validation ✅

**Purpose**: Optimize physics validation with parallelism

**Location**: `hwc/crates/hwc-physics/src/lib.rs`

### Phase 8.1: Parallel Analyzer Execution ✅

- [x] Implement `validate_physics_parallel()` function
  - Run all 4 analyzers in parallel using Rayon
  - Read-only access to board (no mutations)
  - Collect results deterministically

- [x] Add performance tests (9 tests passing)
  - Test: Single-threaded baseline
  - Test: Parallel execution determinism
  - Test: Sequential vs parallel same results
  - Test: Thread safety
  - Test: Read-only access verification

---

## Phase 9: CLI Integration ✅

**Purpose**: Add physics validation commands to CLI

**Location**: `hwc/crates/hwc-cli/src/commands/`

### Phase 9.1: Physics Validation Command ✅

- [x] Add `hwc physics` command
  - Run physics validation on existing build
  - Output detailed physics report
  - Support --verbose flag for detailed analysis
  - Support --parallel flag for multi-threaded validation

- [x] Add `--skip-physics` flag to `hwc build`
  - Skip physics validation for faster iteration
  - Default: enabled (always validate)

---

## Phase 10: Documentation and Examples

**Purpose**: Document physics validation system

### Phase 10.1: Update Documentation

- [ ] Update `Docs/v0.1.3/ROUTING-AND-PHYSICS.md`
  - Add physics validation examples
  - Document error codes
  - Provide troubleshooting guide

### Phase 10.2: Create Examples

- [ ] Create `hwc/examples/physics_validation.hw`
  - Demonstrate electrical analysis
  - Demonstrate thermal analysis
  - Demonstrate impedance control

---

## Testing Strategy

### Unit Tests
- Each physics analyzer: 90%+ coverage
- Material property calculations
- Error code generation

### Integration Tests
- Complete physics validation pipeline
- Multi-domain violations
- Material database integration

### Performance Tests
- Large board validation (1000+ nets)
- Parallel execution speedup
- Memory usage profiling


## Document Version

**Version**: 1.0  
**Created**: March 19, 2026  
**Status**: Implementation Plan  
**Part of**: Hardware Script v0.1.2 Roadmap
