# Module System Implementation - Missing Features Tracker

## Status: IN PROGRESS

This document tracks all missing features discovered while implementing Section 2.1 (Module System Failures) from GAp.md.

---

## ✅ COMPLETED

### 1. Module Flattening Core
- ✅ Module detection in `place_component()`
- ✅ Comptime evaluation (for loops, if conditionals)
- ✅ Module component placement with layout blocks
- ✅ Automatic placement fallback (with offsets)
- ✅ Array index evaluation in component names (e.g., `R[i]` → `R[0]`, `R[1]`, etc.)

### 2. Array Syntax in Layout Blocks ✅ FIXED
- ✅ Parser now supports `R[0]` syntax in layout block component names
- ✅ Location: `hwc/crates/hwc-parser/src/parser/definitions/space.rs:parse_module_layout_block()`
- ✅ Accepts `ComponentName[index]` syntax
- ✅ Tested with for-loop-generated components

### 3. Module Route Flattening ✅ FIXED
- ✅ Module routes are now converted to space routes
- ✅ Location: `hwc/crates/hwc-compiler/src/ir/placement.rs:place_module_instance()`
- ✅ Resolves module pin references (e.g., `R1.Pin2` → `Filter.R1.Pin2`)
- ✅ Detects and skips interface pin routes (routes connecting to module pins)
- ✅ Handles array pin indexing in routes
- ✅ Internal module connections are automatically routed

**Implementation Details**:
- Module routes are processed after component placement (lines 203-275)
- Interface pin detection checks both component and pin fields (lines 213-226)
- Array indices are converted from ArrayIndex enum to Option<usize> (lines 239-261)
- Routes are created with empty waypoints (logical connections) and passed to automatic router

**Test Files**:
- `hwc/examples/module_route_simple.hw` - Basic internal route flattening
- `hwc/examples/module_with_layout.hw` - Multiple internal routes with layout block

**Note**: Module interface pins (pins defined in `pins:` block) that connect to external components still need to be wired from the space level.

---

## 🚧 IN PROGRESS - NONE

---

## ❌ TODO - HIGH PRIORITY

### 4. Import System (@std/ paths)
**Issue**: `@std/materials` fails - lexer doesn't recognize `@` as valid import path start
**Location**: `hwc/crates/hwc-parser/src/lexer.rs`
**Error**: `Unexpected the text "@std/materials"` followed by `identifier required here`
**Current**: Lexer treats `@` as illegal character

**Fix Required**:
1. Update lexer to recognize `@` as valid start of import path
2. Wire up stdlib interceptor in compiler
3. Route `@std/*` to internal `hwc-stdlib` crate

### 5. Array Pin Expansion in Symbol Table ✅ FIXED
**Issue**: `MainDSP.Bus_Out[0]` fails - Symbol Table doesn't expand `Bus_Out[64]` into 64 distinct PinIds
**Location**: `hwc/crates/hwc-compiler/src/ir/routing/helpers.rs`
**Impact**: Cannot route individual bus lines
**Status**: COMPLETED

**Implementation**:
- Array pins are already expanded during module registration (e.g., `Bus_Out[0]`, `Bus_Out[1]`, etc.)
- Updated `get_pin_positions()` to construct full pin names when `pin_index` is present
- Pin lookup now correctly matches expanded pin names in the netlist
- All 543 tests pass, including array pin routing tests

---

## ❌ TODO - MEDIUM PRIORITY

### 6. For Loops in Layout Blocks
**Issue**: Parser doesn't support `for i in 0..3:` inside layout blocks
**Location**: `hwc/crates/hwc-parser/src/parser/definitions/space.rs:parse_module_layout_block()`
**Current**: Only parses simple component mappings
**Needed**: Parse and evaluate for loops in layout blocks

**Example**:
```hw
layout Chain1:
    for i in 0..3:
        R[i] at [100 + (i * 50), 100, 1]
```

### 7. Arithmetic Expressions in Layout Coordinates
**Issue**: Layout coordinates don't support expressions like `100 + (i * 50)`
**Location**: Layout coordinate parsing
**Current**: Only accepts literal values
**Needed**: Full expression evaluation in layout coordinates

### 8. Nested Module Instantiation
**Issue**: Modules can instantiate other modules, but this isn't tested
**Location**: `hwc/crates/hwc-compiler/src/ir/placement.rs:place_module_instance()`
**Current**: Should work recursively, but untested
**Needed**: Test and verify recursive module flattening

---

## ❌ TODO - LOW PRIORITY

### 9. Module Pin Exposure
**Issue**: Module pins need to be exposed to parent scope
**Location**: Module instantiation
**Current**: Module pins are defined but not accessible from space
**Needed**: Map module instance pins to internal component pins

**Example**:
```hw
# In space:
route ExternalComponent.Pin to Filter.Input  # Filter.Input should map to Filter.R1.A
```

### 10. Layout Block Validation
**Issue**: No validation that all module components have layout mappings
**Location**: `hwc/crates/hwc-compiler/src/ir/placement.rs:place_module_instance()`
**Current**: Falls back to automatic placement silently
**Needed**: Warn or error if layout block is incomplete

### 11. Module Parameter Passing
**Issue**: Modules can't accept parameters
**Location**: Module definition and instantiation
**Current**: Modules are static
**Needed**: Support parameterized modules

**Example**:
```hw
define module "ResistorArray" (count: Number, resistance: Measurement):
    for i in 0..count-1:
        add Resistor_0805 (resistance: resistance) named R[i]
```

---

## 📊 SUMMARY

- **Completed**: 3 major features (Module Flattening Core, Array Syntax in Layout Blocks, Module Route Flattening)
- **In Progress**: 0 features
- **High Priority**: 2 features (Imports, Array Pins)
- **Medium Priority**: 3 features (For loops in layouts, Expressions, Nested modules)
- **Low Priority**: 3 features (Pin exposure, Validation, Parameters)

**Total**: 11 features tracked (3 completed, 8 remaining)

---

## 🎯 NEXT STEPS

1. ✅ Complete array index evaluation in component names
2. ✅ Add array syntax support to layout block parser
3. ✅ Implement module route flattening
4. ⏭️ Fix import system (@std/ paths)
5. ⏭️ Implement array pin expansion in symbol table

---

## 📝 NOTES

- Module flattening is working for basic cases
- Layout blocks successfully map components to positions
- Comptime evaluation (for loops, if statements) is functional
- Need to systematically address each missing feature to achieve full GAp.md compliance
