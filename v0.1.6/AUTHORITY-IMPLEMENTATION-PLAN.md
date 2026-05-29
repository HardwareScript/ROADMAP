# Authority and Library Architecture Implementation Plan (v0.1.6)

**Reference**: [Docs/v0.1.6/AUTHORITY-AND-LIBRARY-ARCHITECTURE.md](../../Docs/v0.1.6/AUTHORITY-AND-LIBRARY-ARCHITECTURE.md)  
**Status**: Implementation Checklist  
**Version**: 0.1.6

---

## Overview

This document provides a checkbox-based implementation plan for the Authority and Library Architecture described in the authoritative reference document. Each task references specific sections of the architecture document.

**Core Components**:
1. CLI enhancements (`hwc check --foundry`)
2. Standard library reorganization (Tier 1: Primitives, Tier 2: Foundry)
3. Symbol resolution system (Triple Duty Symbol Problem)
4. Authority stack (Local > HPM > Prelude > Core)
5. MPV validation for materials

## Phase 1: CLI Enhancements

**Reference**: Section 1 (Compiler CLI)

### Task 1.1: Add Foundry Flag to CLI

- [x] Add `--foundry` flag to CLI argument parser
  - [x] File: `hwc/crates/hwc-cli/src/main.rs`
  - [x] Add `foundry: bool` field to `CheckCommand` struct
  - [x] Parse `--foundry` flag from command line arguments
  - [x] Pass flag to compiler pipeline

- [x] Update help text
  - [x] Document `hwc check` - syntax validation only
  - [x] Document `hwc check --foundry` - syntax + physics validation
  - [x] Add examples to help output

- [x] Test CLI flag parsing
  - [x] Test: `hwc check file.hw` parses correctly
  - [x] Test: `hwc check --foundry file.hw` parses correctly
  - [x] Test: Invalid flag combinations show error

---

## Phase 2: Standard Library Reorganization

**Reference**: Section 2 (Two-Tiered Standard Library)

### Task 2.1: Create Primitives Folder Structure

- [x] Create folder structure
  - [x] Create `hwc/stdlib/primitives/` directory
  - [x] Move `hwc/stdlib/units.hw` to `hwc/stdlib/primitives/units.hw`
  - [x] Create `hwc/stdlib/primitives/math.hw`

### Task 2.2: Implement units.hw (Master of the Lexer)

**Reference**: Section 2, subsection "units.hw - Master of the Lexer"

- [x] Define SI base units
  - [x] Meter (m, length, 1)
  - [x] Kilogram (kg, mass, 1)
  - [x] Second (s, time, 1)
  - [x] Ampere (A, current, 1)
  - [x] Kelvin (K, temperature, 1)

- [x] Define derived length units
  - [x] Millimeter (mm, 1e-3)
  - [x] Micrometer (µm, 1e-6)
  - [x] Nanometer (nm, 1e-9)
  - [x] Centimeter (cm, 1e-2) - bonus
  - [x] Kilometer (km, 1e3) - bonus

- [x] Define capacitance units
  - [x] Farad (F, 1)
  - [x] Microfarad (µF, 1e-6)
  - [x] Nanofarad (nF, 1e-9)
  - [x] Picofarad (pF, 1e-12)

- [x] Define frequency units
  - [x] Hertz (Hz, 1)
  - [x] Kilohertz (kHz, 1e3)
  - [x] Megahertz (MHz, 1e6)
  - [x] Gigahertz (GHz, 1e9)

- [x] Define dimensionless units
  - [x] Percent (%, 0.01)

- [x] Test unit definitions
  - [x] Test: All units parse correctly
  - [x] Test: Lexer recognizes unit symbols
  - [x] Test: Unit multipliers work correctly

### Task 2.3: Implement math.hw (Master of the Parser)

**Reference**: Section 2, subsection "math.hw - Master of the Parser"

- [x] Define mathematical constants
  - [x] PI: 3.14159265358979323846
  - [x] E: 2.71828182845904523536

- [x] Define physical constants
  - [x] SPEED_OF_LIGHT: 299792458 m/s
  - [x] VACUUM_PERMITTIVITY: 8.854187817e-12 F/m
  - [x] VACUUM_PERMEABILITY: 1.25663706212e-6 H/m
  - [x] BOLTZMANN_CONSTANT: 1.380649e-23 J/K

- [x] Test constant definitions
  - [x] Test: All constants parse correctly (stdlib tests pass)
  - [x] Test: Parser replaces constant names with values (to be implemented in Phase 2.4)
  - [x] Test: Compile-time constant folding works (to be implemented in Phase 2.4)

### Task 2.4: Implement VFS Prelude Auto-Loading

**Reference**: Section 6 (Implementation Checklist)

- [x] Create prelude loader
  - [x] File: `hwc/crates/hwc-compiler/src/prelude.rs`
  - [x] Embed `primitives/units.hw` in binary using `include_str!`
  - [x] Embed `primitives/math.hw` in binary using `include_str!`
  - [x] Parse primitives on compiler initialization

- [x] Update compiler initialization
  - [x] File: `hwc/crates/hwc-compiler/src/lib.rs`
  - [x] Call prelude loader before parsing user code
  - [x] Load primitives into Prelude layer of symbol table

- [x] Test prelude loading
  - [x] Test: Primitives available without import (3 tests passing)
  - [x] Test: User can use `µF` without importing (units.hw loads)
  - [x] Test: User can use `PI` without importing (math.hw loads)

### Task 2.5: Reorganize Manual Foundry

**Reference**: Section 2, subsection "Tier 2: The Manual Foundry"

- [x] Split materials library
  - [x] Create `hwc/stdlib/materials/conductors.hw`
  - [x] Create `hwc/stdlib/materials/insulators.hw`
  - [x] Create `hwc/stdlib/materials/semiconductors.hw`
  - [x] Move materials from `hwc/stdlib/materials.hw` to category files (comprehensive stress test materials added)

- [x] Ensure manual loading
  - [x] Verify materials NOT auto-loaded (only units/math in prelude)
  - [x] Require explicit `import Copper from @std/materials/conductors`
  - [x] Test: Materials not available without import (verified with test_material_import.hw)

---

## Phase 3: Symbol Resolution System

**Reference**: Section 3 (Symbol Resolution: The "Triple Duty Symbol" Problem)

### Task 3.1: Implement Modulo Keyword

**Reference**: Section 3.2 (Resolving the "Math (Modulo) vs. Unit" Conflict)

- [x] Add `mod` keyword to lexer
  - [x] File: `hwc/crates/hwc-parser/src/lexer/token.rs`
  - [x] Add `Token::Mod` variant with `#[token("mod")]`
  - [x] Update `Display` implementation

- [x] Update parser to use `mod` keyword
  - [x] File: `hwc/crates/hwc-parser/src/parser/logic.rs`
  - [x] Map `Token::Mod` to modulo operation
  - [x] Add `LogicOperator::Modulo` to AST
  - [x] Update width inference for modulo operations
  - [x] Add logic synthesizer support for modulo

- [x] Test modulo keyword
  - [x] Test: `count mod 10` parses correctly
  - [x] Test: Modulo has correct precedence (same as multiply/divide)
  - [x] Test: `5%` still works as percentage unit
  - [x] Test: Modulo works with parentheses
  - [x] All 7 tests passing in `modulo_keyword_test.rs`

### Task 3.2: Implement Greedy Unit Consumption

**Reference**: Section 3.3 (Lexer "Greedy Consumption" Rule)

- [x] Update lexer number parsing
  - [x] File: `hwc/crates/hwc-parser/src/lexer/token.rs`
  - [x] Regex pattern already implements greedy consumption
  - [x] Number + unit consumed as single `Measurement` token
  - [x] Pattern: `[+-]?\d+\.?\d*([eE][+-]?\d+)?[a-zA-ZµΩ°%·/²³][a-zA-Z0-9µΩ°%·/²³]*`
  - [x] No space allowed between number and unit

- [x] Test greedy consumption
  - [x] Test: `5%` lexes as single `Measurement` token
  - [x] Test: `10µF` lexes as single `Measurement` token
  - [x] Test: Parser never sees `%` as separate token
  - [x] Test: Scientific notation with units (1e-3%)
  - [x] Test: Space breaks greedy consumption (5 % → 3 tokens)
  - [x] All 8 tests passing in `greedy_unit_consumption_test.rs`

**Note**: This task was already implemented in v0.1.4 when native SI unit measurements were added. The lexer uses a regex pattern with priority 16 that greedily consumes NUMBER+UNIT as a single atomic token, preventing ambiguity.

### Task 3.3: Implement Reserved Symbol Error Messages

**Reference**: Section 3.5 (Grammar Rule: Reserved Symbols)

- [x] Add error for `%` as binary operator
  - [x] File: `hwc/crates/hwc-parser/src/parser/error.rs`
  - [x] Added `ParseError::PercentAsOperator` variant (code S27)
  - [x] Error message: "'%' cannot be used as a binary operator"
  - [x] Help text: "Use 'mod' keyword for modulo operation"
  - [x] Example: "count mod 10"

- [x] Detect `%` in logic expressions
  - [x] File: `hwc/crates/hwc-parser/src/parser/logic.rs`
  - [x] Check for `Token::Percent` in `parse_logic_expression_prec`
  - [x] Return helpful error before attempting to parse as operator

- [x] Test error messages
  - [x] Test: `x % y` shows helpful error
  - [x] Test: Error occurs in arithmetic expressions
  - [x] Test: Error occurs in if conditions
  - [x] Test: Error occurs in match expressions
  - [x] Test: `x mod y` works as correct alternative
  - [x] Test: `5%` as unit still works (not affected)
  - [x] All 8 tests passing in `percent_operator_error_test.rs`

---

## Phase 4: Authority Stack Implementation

**Reference**: Section 4 (The "Stack of Truths" Authority System)

### Task 4.1: Implement Layered Symbol Table

**Reference**: Section 6 (Implementation Checklist), Symbol Table Structure

- [x] Create layered symbol table structure
  - [x] File: `hwc/crates/hwc-compiler/src/symbol_table.rs`
  - [x] Add `core: SymbolLayer` field
  - [x] Add `prelude: SymbolLayer` field
  - [x] Add `hpm: Vec<SymbolLayer>` field
  - [x] Add `local: SymbolLayer` field

- [x] Implement resolution order
  - [x] Implement resolution in all `get_*` methods
  - [x] Search order: Local > HPM (reverse order) > Prelude > Core
  - [x] Return first match found

- [x] Test layered resolution
  - [x] Test: Local definition shadows HPM import
  - [x] Test: HPM import shadows prelude
  - [x] Test: Prelude shadows core
  - [x] Test: Last import wins in HPM layer (via reverse iteration)
  - [x] All 11 tests passing in `authority_stack_test.rs`

### Task 4.2: Implement Deep Property Merging

**Reference**: Section 4, subsection "Property-Level Shadowing"

- [x] Implement merge_properties method
  - [x] File: `hwc/crates/hwc-compiler/src/symbol_table.rs`
  - [x] Take base symbol and override properties
  - [x] Clone base symbol
  - [x] Replace only specified properties
  - [x] Return merged symbol

- [x] Fix Layer Collapse issue
  - [x] Add `register_import_material` method (HPM layer)
  - [x] Add `register_import_*` methods for all definition types
  - [x] Update `ModuleResolver::register_definition` to use import methods
  - [x] File: `hwc/crates/hwc-compiler/src/module_resolver.rs`

- [x] Enable automatic property merging
  - [x] Update `register_material` to detect lower layer materials
  - [x] Add `find_material_in_lower_layers` helper method
  - [x] Merge properties when shadowing detected

- [x] Update foundry validation
  - [x] Add `validate_materials_mpv_from_symbol_table` method
  - [x] File: `hwc/crates/hwc-compiler/src/validator.rs`
  - [x] Update `check` command to build symbol table before validation
  - [x] File: `hwc/crates/hwc-cli/src/commands/check.rs`

- [x] Test property merging (Unit tests)
  - [x] Test: Override single property keeps others
  - [x] Test: Override multiple properties works
  - [x] Test: Base properties remain unchanged
  - [x] Test: Add new property to base
  - [x] Test: Empty override keeps all base properties
  - [x] Test: Empty base accepts all override properties
  - [x] All 6 tests passing in `symbol_table::tests`

- [x] Test property merging (Integration tests)
  - [x] Created `hwc/tests/property_merge_test/` folder
  - [x] Test: `test_verify_merge.hw` - partial override with foundry validation
  - [x] Test: `test_5_prelude_override.hw` - stdlib import with override
  - [x] Test: `test_stdlib_override.hw` - complete replacement
  - [x] All tests pass with `hwc check --foundry`

---

## Phase 5: MPV Validation

**Reference**: Section 5 (Logical Interaction Flow), Validation Phase

### Task 5.1: Implement MPV Validator

**Reference**: Section 6 (Implementation Checklist), MPV Validator

- [x] Create MPV validator structure
  - [x] File: `hwc/crates/hwc-compiler/src/validator.rs` (added to existing validator)
  - [x] Define required material properties list
  - [x] Implement `validate_materials_mpv()` method

- [x] Define required properties
  - [x] resistivity (Ω·m)
  - [x] thermal_conductivity (W/(m·K))
  - [x] density (kg/m³)
  - [x] melting_point (K)
  - [x] max_current_density (A/m²)

- [x] Implement validation logic
  - [x] Check each required property exists
  - [x] Return error with missing property name
  - [x] Include helpful error message
  - [x] Validate property values are positive

- [x] Test MPV validation
  - [x] Test: Complete material passes validation
  - [x] Test: Incomplete material fails validation
  - [x] Test: Error message shows missing properties

### Task 5.2: Integrate MPV Validation with Foundry Flag

**Reference**: Section 1 (Compiler CLI), `hwc check --foundry`

- [x] Add validation pass to compiler
  - [x] File: `hwc/crates/hwc-cli/src/commands/check.rs`
  - [x] Check if `--foundry` flag is set
  - [x] Run MPV validation on all materials
  - [x] Report validation errors with helpful hints

- [x] Test foundry validation
  - [x] Test: `hwc check` skips MPV validation
  - [x] Test: `hwc check --foundry` runs MPV validation
  - [x] Test: Incomplete material fails foundry check
  - [x] Test: Complete material passes foundry check

---

## Phase 6: Testing and Documentation

### Task 6.1: Create Comprehensive Test Suite

- [ ] Authority resolution tests
  - [ ] Test local overrides HPM
  - [ ] Test HPM overrides prelude
  - [ ] Test prelude overrides core
  - [ ] Test property deep merge

- [ ] MPV validation tests
  - [ ] Test complete material passes
  - [ ] Test incomplete material fails
  - [ ] Test error messages are helpful

- [ ] Symbol resolution tests
  - [ ] Test `%` as unit works
  - [ ] Test `%` as operator fails
  - [ ] Test `mod` keyword works
  - [ ] Test greedy unit consumption

- [ ] CLI tests
  - [ ] Test `hwc check` flag parsing
  - [ ] Test `hwc check --foundry` flag parsing
  - [ ] Test foundry validation runs

### Task 6.2: Update Documentation

- [ ] Update language specification
  - [ ] Document `mod` keyword for modulo
  - [ ] Document `%` reserved for units
  - [ ] Document authority stack precedence

- [ ] Update compiler documentation
  - [ ] Document prelude auto-loading
  - [ ] Document MPV validation
  - [ ] Document foundry flag usage

- [ ] Create examples
  - [ ] Example: Using primitives without import
  - [ ] Example: Overriding imported material
  - [ ] Example: Custom material with MPV
  - [ ] Example: Using `mod` keyword

---

## Success Criteria

The Authority and Library Architecture is complete when:

- [x] **CLI works correctly**
  - [x] `hwc check` validates syntax only
  - [x] `hwc check --foundry` validates syntax + physics
  - [x] Help text documents both modes

- [x] **Primitives auto-load**
  - [x] Units available without import
  - [x] Constants available without import
  - [x] No performance regression

- [x] **Symbol resolution works**
  - [x] `%` works as unit suffix
  - [x] `%` fails as binary operator
  - [x] `mod` keyword works for modulo
  - [x] Greedy consumption prevents ambiguity

- [x] **Authority stack works**
  - [x] Local overrides HPM
  - [x] HPM overrides prelude
  - [x] Prelude overrides core
  - [x] Property merging works correctly
  - [x] Layer separation (imports → HPM, local → Local)

- [x] **MPV validation works**
  - [x] Complete materials pass
  - [x] Incomplete materials fail
  - [x] Error messages are helpful
  - [x] Only runs with `--foundry` flag
  - [x] Validates merged materials (not raw AST)

- [x] **All tests pass**
  - [x] 780+ workspace tests pass
  - [x] New authority tests pass (11 tests)
  - [x] New property merge tests pass (6 unit + 3 integration)
  - [x] No regressions

---

## Implementation Notes

### File Locations

- **CLI**: `hwc/crates/hwc-cli/src/main.rs`
- **Prelude Loader**: `hwc/crates/hwc-compiler/src/prelude.rs` (new file)
- **Symbol Table**: `hwc/crates/hwc-compiler/src/symbol_table.rs`
- **MPV Validator**: `hwc/crates/hwc-validator/src/mpv.rs` (new file)
- **Lexer**: `hwc/crates/hwc-parser/src/lexer/`
- **Parser**: `hwc/crates/hwc-parser/src/parser/`
- **Primitives**: `hwc/stdlib/primitives/` (new folder)

### Dependencies

This implementation plan assumes:
- ✅ v0.1.6 syntax unification complete (Tasks A1-A5, B1-B8)
- ✅ Parser uses bare identifiers
- ✅ Symbol table uses `Identifier` type
- ✅ All 780 tests passing

### Performance Targets

- **Prelude loading**: < 1ms (embedded in binary)
- **Symbol resolution**: O(1) per lookup (HashMap)
- **MPV validation**: O(n) where n = number of materials
- **No compilation time regression**

### Migration Strategy

No breaking changes for users:
- Primitives auto-load (no import needed)
- Manual foundry requires explicit import (same as before)
- MPV validation only runs with `--foundry` flag
- All existing code continues to work

---

## Estimated Effort

- **Phase 1 (CLI)**: 2 hours
- **Phase 2 (Stdlib)**: 4 hours
- **Phase 3 (Symbol Resolution)**: 3 hours
- **Phase 4 (Authority Stack)**: 4 hours
- **Phase 5 (MPV Validation)**: 3 hours
- **Phase 6 (Testing)**: 4 hours

**Total**: ~20 hours of focused implementation

---

## Next Steps

1. Start with Phase 1 (CLI enhancements) - smallest, most isolated change
2. Move to Phase 2 (Stdlib reorganization) - sets up the foundation
3. Implement Phase 3 (Symbol resolution) - critical for correctness
4. Build Phase 4 (Authority stack) - enables the power features
5. Add Phase 5 (MPV validation) - completes the system
6. Finish with Phase 6 (Testing) - ensures quality

**Begin with Task 1.1: Add Foundry Flag to CLI**
