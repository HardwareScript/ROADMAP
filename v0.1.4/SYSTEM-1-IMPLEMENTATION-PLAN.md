# System 1: Unified Parser & Two-Pass Compiler - Implementation Plan

**System**: Unified Parser & Two-Pass Compiler  
**Status**: ❌ NOT STARTED - Complete architectural rewrite from v0.1.3  
**Based On**: v0.1.4 Documentation  
**Prerequisites**: None (this is the foundation)  
**Last Updated**: March 20, 2026

---

## Overview

Transform `.hw` source code into Hardware IR using a unified parser that handles all `define` blocks, with two-pass compilation for symbol resolution.

**Architecture (v0.1.4 - Revolutionary Change)**:
- Lexer: `logos` crate with native SI unit tokens (e.g., `254µm`, `4.7kΩ`)
- Parser: Hand-written recursive descent for ALL `define` blocks
- Symbol Table: Two-pass compilation (register → resolve)
- AST: Unified structure for all definition types
- IR: Hardware IR with resolved symbols
- Standard Library: `hwc/stdlib/units.hw` for extensible unit definitions
- Dependencies: 4 total (logos, miette, thiserror, rayon) - NO `serde_yaml`

**Critical Changes from v0.1.3**:
- ❌ Remove: Separate parsers for `.hwmat`, `.hwp`, `.hwx`, `.hwf`, `.hwt`, `.hwa`, `.hwm`, `.hwsig`, `.hwtc`
- ❌ Remove: YAML parsing (`serde_yaml` dependency)
- ❌ Remove: Single-pass compilation
- ✅ Add: Unified parser for all `define` blocks
- ✅ Add: Symbol Table for two-pass compilation
- ✅ Add: Native SI unit parsing in lexer
- ✅ Add: Standard Library system (`hwc/stdlib/units.hw`)
- ✅ Add: `define material`, `define profile`, `define component`, `define mechanical`, `define interface`, `define test` support

**Current Status**:
- ✅ Workspace structure exists
- ✅ Lexer has SI unit tokens (generic measurement pattern)
- ✅ Parser handles all core `define` blocks (material, profile, component, space)
- ✅ Symbol Table implemented and tested
- ✅ Two-pass compilation implemented and tested
- ✅ Native SI unit parsing working
- ✅ Standard Library architecture implemented
- ⏭️ Optional parsers (mechanical, interface, test) - stubs exist
- ⏭️ Optional material/constraint population - empty placeholders work

---

## Phase 1: Project Setup & Dependency Cleanup ✅ COMPLETE

### 1.1 Workspace Structure ✅ COMPLETE
- [x] `hwc/` workspace root exists
- [x] `hwc/crates/hwc-parser/` crate exists
- [x] `hwc/crates/hwc-compiler/` crate exists
- [x] `hwc/crates/hwc-engine/` crate exists
- [x] `hwc/crates/hwc-physics/` crate exists
- [x] `hwc/crates/hwc-export/` crate exists
- [x] `hwc/crates/hwc-materials/` crate exists
- [x] `hwc/crates/hwc-stdlib/` crate exists
- [x] `hwc/crates/hwc-cli/` crate exists

### 1.2 Dependency Cleanup ✅ COMPLETE
- [x] Remove `serde_yaml` from `hwc-parser/Cargo.toml`
- [x] Remove `serde_yaml` from `hwc-compiler/Cargo.toml`
- [x] Remove `serde_yaml` from `hwc-materials/Cargo.toml`
- [x] Remove `serde` from all crates (no longer needed)
- [x] Verify only 4 dependencies remain: `logos`, `miette`, `thiserror`, `rayon`
- [x] Update workspace `Cargo.toml` to reflect v0.1.4 dependencies
- [x] Update Clippy rules to strict mode (unwrap_used, expect_used, panic = "deny")

### 1.3 Directory Restructuring ✅ COMPLETE
- [x] Remove serde imports from `hwc-materials/src/` files (blocking compilation)
- [x] Remove serde imports from `hwc-engine/src/` files (blocking compilation)
- [x] Stub out YAML loading functions with "use Symbol Table" errors
- [x] Add `MaterialDatabase::empty()` for testing without YAML
- [x] Create `hwc-compiler/src/symbol_table.rs` (Phase 4)
- [x] Update `hwc-compiler/src/ir_compiler.rs` to use empty MaterialDatabase
- [x] Remove old YAML parsing code from `hwc-materials/src/` (stubbed out, returns errors)
- [x] Remove separate file parsers (none exist - unified parser only)
- [x] Update `hwc-parser/tests/` for unified syntax (all tests use `define` blocks)

---

## Phase 2: Lexer Enhancement (Native SI Unit Parsing) ✅ COMPLETE

**Reference**: [LANGUAGE-SPEC.md](../../Docs/v0.1.4/LANGUAGE-SPEC.md) - Native SI unit syntax

**Status**: ✅ COMPLETE - Native SI unit parsing implemented and tested

### 2.1 Add SI Unit Tokens ✅ COMPLETE

**Critical Change**: Parser natively understands units like `254µm`, `4.7kΩ`, `100nF`

**Syntax Rule**: Units must be attached to numbers with NO space (following CSS/Rust convention)
- ✅ Correct: `254µm`, `4.7kΩ`, `100nF`, `8960kg/m³`
- ❌ Incorrect: `254 µm`, `4.7 kΩ`, `100 nF`, `8960 kg/m³`

**Why No Spaces?**
- Prevents ambiguity with keywords (`500 by 500` vs `500by`)
- Keeps lexer simple and deterministic (avoids stack overflow)
- Matches industry standards (CSS: `10px`, Rust: `10u32`)
- Enables clean separation between measurements and language syntax

- [x] Add distance unit tokens:
  - `mm`, `cm`, `µm` (micrometers)
  - Pattern: `NUMBER UNIT` → `Measurement(value, unit)`
  
- [x] Add electrical unit tokens:
  - Voltage: `V`, `mV`, `kV`
  - Current: `A`, `mA`, `µA`
  - Resistance: `Ω`, `kΩ`, `MΩ`, `Ohm`, `kOhm`, `MOhm` (keyboard aliases)
  - Capacitance: `F`, `pF`, `nF`, `µF`, `uF` (keyboard alias)
  - Frequency: `Hz`, `kHz`, `MHz`, `GHz`
  
- [x] Add material property unit tokens:
  - Density: `kg/m3`
  - Thermal conductivity: `W/mK`
  - Temperature: `C` (Celsius)
  - Resistivity: `Ω/m`
  - Current density: `A/mm2`
  
- [x] Add compound unit parsing:
  - `1.68e-8Ω/m` → `Measurement(1.68e-8, OhmPerMeter)`
  - `8960kg/m3` → `Measurement(8960, KgPerM3)`

### 2.2 Update Token Enum ✅ COMPLETE

- [x] Created `hwc-parser/src/lexer/units.rs` with comprehensive unit type system
- [x] Implemented unified `Measurement` struct wrapping value and unit
- [x] Added single `Token::Measurement(Measurement)` variant (Logos requires single-field variants)
- [x] Implemented parsing callbacks for all unit types
- [x] Added Display implementations for all unit types

### 2.3 Add `define` Block Keywords ✅ COMPLETE

- [x] Add keyword tokens:
  - `define` (already exists)
  - `material`, `profile`, `component`, `mechanical`, `interface`, `test`, `space`
  - `category`, `properties`, `metadata`, `pins`, `layout`, `electrical`, `render`
  - `trace`, `via`, `layer`, `clearance`
  - `bindings`, `protocols`, `setup`, `execute`, `assert`

### 2.4 Lexer Tests ✅ COMPLETE

- [x] Test: `254µm` → `Measurement(254.0, Micrometers)` ✅
- [x] Test: `4.7kΩ` → `Measurement(4.7, Kiloohms)` ✅
- [x] Test: `100nF` → `Measurement(100.0, Nanofarads)` ✅
- [x] Test: `90°` → `Measurement(90.0, Degrees)` ✅
- [x] Test: `define space "TestBoard"` → correct token sequence ✅
- [x] Test: IEC 60062 notation (`4K7` tokenizes as identifier, not unit) ✅
- [x] Test: Compound units (`8960kg/m³`, `1400cm2/Vs`) ✅
- [x] Test: UTF-8 symbols (`Ω·m`, `W/(m·K)`) ✅
- [x] Created `test_lexer_v014.rs` demonstrating native SI unit parsing
- [x] Verified units must be attached to numbers (no space: `10mm` not `10 mm`)

**Key Achievement**: Lexer correctly handles Unicode SI notation while preventing keyword ambiguity

---

## Phase 3: Unified AST for All `define` Blocks ✅ COMPLETE

**Reference**: [LANGUAGE-SPEC.md](../../Docs/v0.1.4/LANGUAGE-SPEC.md) - All `define` block syntax

**Status**: ✅ COMPLETE - Unified AST structures implemented

### 3.1 Top-Level AST Structure ✅ COMPLETE

- [x] Updated `Program` struct to use `definitions: Vec<Definition>`
- [x] Created `Definition` enum with all definition types
- [x] Updated parser to use `parse_definition()` method

### 3.2 Material Definition AST ✅ COMPLETE

- [x] Created `MaterialDefinition` struct
- [x] Created `MaterialCategory` enum (Conductor, Insulator, Semiconductor)
- [x] Created `Property` and `PropertyValue` types
- [x] Added stub parser method `parse_material()`

### 3.3 Profile Definition AST ✅ COMPLETE

- [x] Created `ProfileDefinition` struct
- [x] Created `TraceConstraints` struct
- [x] Created `ViaConstraints` struct
- [x] Created `LayerConstraints` struct
- [x] Created `ClearanceConstraints` struct
- [x] Added stub parser method `parse_profile()`

### 3.4 Component Definition AST ✅ COMPLETE

- [x] Updated `ComponentDefinition` for v0.1.4 structure
- [x] Created `ComponentMetadata` struct
- [x] Created `LayoutBlock` struct (replaces old `Layout`)
- [x] Created `PinPosition` struct
- [x] Created `ElectricalBlock` struct
- [x] Created `RenderBlock` and `RenderType` enum
- [x] Added stub parser method `parse_component()`

### 3.5 Space Definition AST ✅ COMPLETE

- [x] Updated `SpaceDefinition` to include `profile: Option<String>`
- [x] Updated `SpaceDefinition` to include `mechanical: Option<String>`
- [x] Updated `parse_space()` to parse profile and mechanical references
- [x] Kept existing component placement and routing logic

### 3.6 Mechanical, Interface, Test AST ✅ COMPLETE

- [x] Created `MechanicalDefinition` struct
- [x] Created `MountingHole` and `Keepout` structs
- [x] Created `InterfaceDefinition` struct
- [x] Created `Binding`, `Protocol`, and `ProtocolPin` structs
- [x] Created `TestDefinition` struct
- [x] Created `TestAction`, `TestActionType`, `TestAssertion`, `TestCondition` types
- [x] Added stub parser methods for all definition types

---

## Phase 4: Symbol Table Implementation ✅ COMPLETE

**Reference**: [COMPILER-INTERNALS.md](../../Docs/v0.1.4/COMPILER-INTERNALS.md) - Two-pass compilation

**Status**: ✅ COMPLETE - Symbol Table implemented and tested

### 4.1 Symbol Table Data Structure ✅ COMPLETE

```rust
pub struct SymbolTable {
    materials: HashMap<String, MaterialDef>,
    profiles: HashMap<String, ProfileDef>,
    components: HashMap<String, ComponentDef>,
    mechanicals: HashMap<String, MechanicalDef>,
    interfaces: HashMap<String, InterfaceDef>,
    tests: HashMap<String, TestDef>,
}

impl SymbolTable {
    pub fn new() -> Self {
        Self {
            materials: HashMap::new(),
            profiles: HashMap::new(),
            components: HashMap::new(),
            mechanicals: HashMap::new(),
            interfaces: HashMap::new(),
            tests: HashMap::new(),
        }
    }
    
    pub fn register_material(&mut self, def: MaterialDef) -> Result<(), CompileError> {
        if self.materials.contains_key(&def.name) {
            return Err(CompileError::DuplicateDefinition {
                name: def.name.clone(),
                kind: "material",
                span: def.span,
            });
        }
        self.materials.insert(def.name.clone(), def);
        Ok(())
    }
    
    pub fn get_material(&self, name: &str) -> Result<&MaterialDef, CompileError> {
        self.materials.get(name).ok_or_else(|| CompileError::UndefinedSymbol {
            name: name.to_string(),
            kind: "material",
        })
    }
    
    // Similar methods for profile, component, etc.
}
```

### 4.2 Symbol Registration (Pass 1) ✅ COMPLETE

- [x] Implement `register_material()`
- [x] Implement `register_profile()`
- [x] Implement `register_component()`
- [x] Implement `register_mechanical()`
- [x] Implement `register_interface()`
- [x] Implement `register_test()`
- [x] Add duplicate definition detection
- [x] Add span tracking for error messages

### 4.3 Symbol Resolution (Pass 2) ✅ COMPLETE

- [x] Implement `get_material()`
- [x] Implement `get_profile()`
- [x] Implement `get_component()`
- [x] Add undefined symbol error handling
- [ ] Add circular dependency detection (deferred to Phase 6)

### 4.4 Symbol Table Tests ✅ COMPLETE

- [x] Test: Register material successfully
- [x] Test: Duplicate material definition error
- [x] Test: Resolve material by name
- [x] Test: Undefined material error
- [x] Test: Register all definition types
- [ ] Test: Cross-references between definitions (Phase 6)

**Test Results**: All 5 symbol table tests pass ✅

---

## Phase 5: Unified Parser for `define` Blocks ✅ COMPLETE

**Reference**: [LANGUAGE-SPEC.md](../../Docs/v0.1.4/LANGUAGE-SPEC.md) - Complete grammar

**Status**: ✅ COMPLETE - All parsers implemented and tested (100%)

**Achievement**: Full unified parser for all `define` blocks with comprehensive test coverage.

### 5.1 Parser Core ✅ COMPLETE

- [x] Verify `Parser` struct exists
- [x] Verify `current()`, `advance()`, `check()`, `expect()` methods exist
- [x] Update error generation for new definition types
- [x] Add `expect_number()` helper method
- [x] Add `skip_until_dedent()` helper method

### 5.2 Top-Level Parsing ✅ COMPLETE

```rust
impl Parser {
    pub fn parse(&mut self) -> Result<Program, ParseError> {
        let mut imports = Vec::new();
        let mut definitions = Vec::new();
        
        while !self.is_at_end() {
            if self.check(Token::Import) {
                imports.push(self.parse_import()?);
            } else if self.check(Token::Define) {
                definitions.push(self.parse_definition()?);
            } else {
                return Err(self.error("Expected 'import' or 'define'"));
            }
        }
        
        Ok(Program { imports, definitions })
    }
    
    fn parse_definition(&mut self) -> Result<Definition, ParseError> {
        self.expect(Token::Define)?;
        
        match self.current() {
            Token::Material => Ok(Definition::Material(self.parse_material()?)),
            Token::Profile => Ok(Definition::Profile(self.parse_profile()?)),
            Token::Component => Ok(Definition::Component(self.parse_component()?)),
            Token::Mechanical => Ok(Definition::Mechanical(self.parse_mechanical()?)),
            Token::Interface => Ok(Definition::Interface(self.parse_interface()?)),
            Token::Test => Ok(Definition::Test(self.parse_test()?)),
            Token::Space => Ok(Definition::Space(self.parse_space()?)),
            _ => Err(self.error("Expected definition type after 'define'")),
        }
    }
}
```

### 5.2 Top-Level Parsing ✅ COMPLETE

- [x] Implement `parse_definition()` dispatcher
- [x] Route to correct parser based on definition type (material, profile, component, etc.)
- [x] Update `parse()` method to use `parse_definition()`

### 5.3 Material Definition Parsing ✅ COMPLETE

- [x] Implement `parse_material()`
- [x] Implement `parse_material_category()`
- [x] Implement `parse_properties()`
- [x] Implement `parse_property_value()`
- [x] Handle keyword tokens (category, properties)
- [x] Support measurements, strings, and numbers in properties
- [x] Test: Parse material with all fields (3/3 tests passing)
- [x] Test: Parse minimal material (category only)
- [x] Test: Parse material with measurements

### 5.4 Profile Definition Parsing 🚧 IN PROGRESS

### 5.4 Profile Definition Parsing ✅ COMPLETE

- [x] Implement `parse_profile()` (full implementation)
- [x] Implement `parse_trace_constraints()`
- [x] Implement `parse_via_constraints()`
- [x] Implement `parse_layer_constraints()`
- [x] Implement `parse_clearance_constraints()`
- [x] Test: Parse profile with all constraint types (3/3 tests passing)
- [x] Test: Parse minimal profile
- [x] Test: Parse profile with trace only

### 5.5 Component Definition Parsing ✅ COMPLETE

**Status**: ✅ COMPLETE - Component parsing fully implemented and tested

**Completed Work:**
- [x] Implement `parse_component_def()` (full implementation)
- [x] Implement `parse_component_metadata()`
- [x] Implement `parse_pins_block()`
- [x] Implement `parse_layout_block()`
- [x] Implement `parse_electrical_block()`
- [x] Implement `parse_render_block()`
- [x] Test: Parse component with all blocks (3/3 tests passing)
- [x] Test: Parse minimal component
- [x] Test: Parse actual resistor component
- [x] Test: Parse actual capacitor component

**Test Results**: All 3 component parsing tests pass ✅
- `test_parse_minimal_component` ✅
- `test_parse_actual_resistor_component` ✅
- `test_parse_actual_capacitor_component` ✅

**Reference**: See `hwc/data/standard-components.hw` for real-world examples

### 5.6 Space Definition Parsing ✅ COMPLETE (from v0.1.3)

- [x] Update `parse_space()` to handle `profile:` field (will integrate in Phase 6)
- [x] Update `parse_space()` to handle `mechanical:` field (will integrate in Phase 6)
- [x] Keep existing component placement and routing logic

### 5.7 Mechanical, Interface, Test Parsing ✅ COMPLETE

**Status**: ✅ COMPLETE - All three parsers fully implemented and tested

**Completed Work:**
- [x] Mechanical Parser
  - [x] Parse `dimensions:` field (measurement by measurement by measurement)
  - [x] Parse `mounting_holes:` list with position and diameter
  - [x] Parse `keepout:` list with regions and optional height
  - [x] Test: Parse mechanical with all fields ✅
  - [x] Test: Parse minimal mechanical ✅
  - [x] Test: Parse mechanical with mounting holes ✅
  - [x] Test: Parse mechanical with keepouts ✅

- [x] Interface Parser
  - [x] Parse `target:` field (string)
  - [x] Parse `bindings:` block with signal = pin_ref pairs
  - [x] Parse `protocols:` block with nested pin mappings and properties
  - [x] Test: Parse interface with all fields ✅
  - [x] Test: Parse minimal interface ✅
  - [x] Test: Parse interface with bindings ✅
  - [x] Test: Parse interface with protocols ✅

- [x] Test Parser
  - [x] Parse `setup:` block with test actions (apply voltage)
  - [x] Parse `execute:` block with test actions (short, wait)
  - [x] Parse `assert:` block with assertions (comparisons)
  - [x] Test: Parse test with all blocks ✅
  - [x] Test: Parse minimal test ✅

**Test Results**: All 8 mechanical/interface/test parsing tests pass ✅

**Lexer Enhancements**:
- [x] Added `Token::Equals` for `=` operator
- [x] Added `Token::LessThan` for `<` operator
- [x] Added `Token::GreaterThan` for `>` operator

**Total Parser Tests**: 67 tests passing (was 32)

---

## Phase 6: Two-Pass Compilation ✅ COMPLETE

**Reference**: [COMPILER-INTERNALS.md](../../Docs/v0.1.4/COMPILER-INTERNALS.md) - Two-pass architecture

**Status**: ✅ COMPLETE - Core architecture implemented, material/constraint population complete

**New Requirements from Documentation Review**:
- ⚠️ **BREAKING CHANGE NEEDED**: Material/constraint population must use Standard Library system
- ⚠️ **BREAKING CHANGE NEEDED**: Profile constraints need proper conversion to ConstraintSet
- 📖 **Reference**: [STANDARD-LIBRARY-ARCHITECTURE.md](../../Docs/v0.1.4/STANDARD-LIBRARY-ARCHITECTURE.md) for unit loading
- 📖 **Reference**: [GENERIC-MEASUREMENT-ARCHITECTURE.md](../../Docs/v0.1.4/GENERIC-MEASUREMENT-ARCHITECTURE.md) for Unit::Custom handling

### 6.1 Pass 1: Symbol Registration ✅ COMPLETE

- [x] Implement `compile_two_pass()` entry point
- [x] Register all definitions into Symbol Table
- [x] Detect duplicate definitions
- [x] Detect missing space definition
- [x] Detect multiple space definitions
- [x] Test: Basic two-pass compilation (3/3 tests passing)
- [x] Test: No space definition error
- [x] Test: Duplicate material error

### 6.2 Pass 2: Space Assembly ✅ COMPLETE

- [x] Extract space metadata (name, dimensions, grid, origin)
- [x] Calculate voxel sizes
- [x] Compile substrate placement
- [x] Compile component placements (basic)
- [x] Compile routes (basic)
- [x] Create empty MaterialDatabase (placeholder)
- [x] Create empty ConstraintSet (placeholder)

### 6.3 Measurement Conversion ✅ COMPLETE

- [x] Reuse existing `measurement_to_nm()` function
- [x] Reuse existing `coordinate_to_nm()` function
- [x] Implement `convert_unit()` for parameter units

### 6.4 Profile to Constraints Conversion ✅ COMPLETE

**Status**: ✅ Implementation complete and validated against documentation

**Completed Work**:
- [x] Implement `convert_profile_to_constraints()` with IPC-2221 references
- [x] Convert trace constraints to ConstraintSet with proper sources
- [x] Convert via constraints to ConstraintSet with proper sources
- [x] Convert layer constraints to ConstraintSet with proper sources
- [x] Convert clearance constraints to ConstraintSet with proper sources
- [x] Add physics-based calculation functions:
  - [x] `calculate_clearance_nm()` - Translation 1: Dielectric breakdown
  - [x] `calculate_trace_width_nm()` - Translation 2: IPC-2221 current capacity
  - [x] `calculate_crosstalk_penalty()` - Translation 3: EMI and crosstalk
- [x] Test: Profile resolution populates constraints correctly
- [x] Verify constraint conversion matches ROUTING-AND-PHYSICS.md formulas
- [x] Fix IPC-2221 formula to use mils² (not mm²) per standard
- [x] Update documentation with correct formula and examples

**Key Achievement**: Physics-based constraint calculations now match IPC-2221A Section 6.2 exactly

**Files Updated**:
- `hwc/crates/hwc-compiler/src/conversions.rs` - Physics calculations
- `Docs/v0.1.4/ROUTING-AND-PHYSICS.md` - Corrected formula and examples

### 6.5 Material Database Population ✅ COMPLETE

**Status**: ✅ Implementation complete and validated against documentation

**Completed Work**:
- [x] Implement `populate_material_database()` with authoritative sources
- [x] Convert material properties to ConductorProperties/InsulatorProperties
- [x] Handle all material categories (conductor, insulator, semiconductor)
- [x] Test: Material resolution populates database correctly
- [x] All default values cite authoritative sources (Materials Project, IPC, NIST)
- [x] Created `hwc/data/standard-materials.hw` with verified data
- [x] Created `hwc/data/standard-profiles.hw` with IPC-2221 standards
- [x] Created `hwc/data/standard-components.hw` with real datasheets
- [x] Verify material property extraction matches GENERIC-MEASUREMENT-ARCHITECTURE.md
- [x] Ensure Unit::Custom(String) handling for unknown units
- [x] Validate all default values cite authoritative sources

**Key Achievement**: Material database uses real-world data from authoritative sources

**Files Validated**:
- `hwc/crates/hwc-compiler/src/conversions.rs` - Material population logic
- `hwc/crates/hwc-materials/src/database.rs` - MaterialDatabase structure
- `hwc/data/standard-materials.hw` - Standard material definitions

### 6.6 SI Notation Standards Compliance ✅ COMPLETE

**Status**: ✅ Implementation complete and validated against documentation

**Completed Work**:
- [x] Verify lexer accepts both ASCII and Unicode per GENERIC-MEASUREMENT-ARCHITECTURE.md
- [x] Verify display always outputs proper Unicode symbols
- [x] Validate all compound units follow NIST/BIPM standards
- [x] Test keyboard aliases work correctly (`Ohm` → `Ω`, `uF` → `µF`)
- [x] Fixed resistivity notation: `Ω·m` (ohm-meter, NOT Ω/m)
- [x] Updated all units to NIST/BIPM standards
- [x] Lexer accepts both ASCII and Unicode input
- [x] Display always outputs proper Unicode symbols
- [x] Updated parsers to handle material property units
- [x] All conversions reference authoritative sources

**Key Achievement**: Full compliance with NIST/BIPM SI notation standards

**Files Validated**:
- `hwc/crates/hwc-parser/src/lexer/units.rs` - Unit type definitions
- `hwc/crates/hwc-parser/src/lexer/token.rs` - Generic measurement regex
- `hwc/crates/hwc-parser/src/lexer/parsers.rs` - Unit parsing callbacks

**Current Status**: Phase 6 is fully complete with all validation tasks done.

### 6.7 Module Support & Comptime Evaluation ✅ COMPLETE

**Status**: ✅ COMPLETE - Full implementation with comprehensive testing

**Parser Implementation** ✅:
- [x] Add `define module` to AST (ModuleDefinition, PinDeclaration, ModuleStatement)
- [x] Add `Module` variant to `Definition` enum
- [x] Implement `parse_module()` method with full syntax support
- [x] Add module registration to Pass 1 (Symbol Table)
- [x] Add control flow keywords to lexer (`for`, `in`, `if`, `else`)
- [x] Add operators to lexer (`..`, `==`, `!=`, `+`)
- [x] Implement `parse_for_loop()` with Ruby-style inclusive ranges
- [x] Implement `parse_if_conditional()` with optional else blocks
- [x] Implement array index parsing with arithmetic (`i`, `i+1`, literals)
- [x] Implement module pin reference parsing (supports `Bus[i]` and `Component[i].Pin`)

**Compiler Integration** ✅:
- [x] Add `register_module()` to SymbolTable
- [x] Implement module registration in Pass 1
- [x] Create `module_flattener.rs` with comptime evaluation
- [x] Implement comptime `for` loop unrolling (Ruby-style inclusive ranges)
- [x] Implement comptime `if` conditional evaluation
- [x] Implement `ArrayIndex` evaluation (literals, variables, arithmetic)
- [x] Implement `Condition` evaluation (==, !=, <, >)
- [x] Export `flatten_module()` from compiler crate
- [x] Integration with CLI build command

**Testing** ✅:
- [x] 15 comprehensive module flattening tests (all passing)
- [x] 64-bit ALU example test (257 routes generated correctly)
- [x] Nested loops, conditionals, arithmetic expressions
- [x] End-to-end compilation test with real .hw file

**Files**:
- `hwc/crates/hwc-parser/src/ast/module.rs` ✅
- `hwc/crates/hwc-parser/src/parser/definitions/module.rs` ✅
- `hwc/crates/hwc-compiler/src/module_flattener.rs` ✅
- `hwc/crates/hwc-compiler/tests/module_flattening_test.rs` ✅
- `project/test_module_simple.hw` ✅

---

## Phase 7: Testing ✅ COMPLETE

**Reference**: [LANGUAGE-SPEC.md](../../Docs/v0.1.4/LANGUAGE-SPEC.md) - Complete syntax examples

**Status**: ✅ Integration tests validate the architecture

### 7.1 Lexer Tests ✅ COMPLETE (from Phase 2)

- [x] Test: Native SI unit parsing (`254µm`, `4.7kΩ`, `100nF`)
- [x] Test: Compound units (`1.68e-8 Ω/m`, `8960 kg/m3`)
- [x] Test: All `define` keywords tokenize correctly
- [x] Test: Reject IEC 60062 notation (`4K7` → identifier, not unit)
- [x] Test: Unicode unit symbols (`Ω`, `µ`)
- [x] Test: Keyboard aliases (`Ohm`, `uF`)

### 7.2 Parser Tests ✅ COMPLETE (from Phase 5)

- [x] Test: Parse `define material` block (3/3 tests passing)
- [x] Test: Parse `define profile` block (3/3 tests passing)
- [x] Test: Parse `define space` block with profile reference
- [x] Test: Error on duplicate definitions
- [x] Test: Error on undefined references

### 7.3 Symbol Table Tests ✅ COMPLETE (from Phase 4)

- [x] Test: Register material successfully
- [x] Test: Duplicate material error
- [x] Test: Resolve material by name
- [x] Test: Undefined material error
- [x] Test: Register all definition types
- [x] Test: Cross-references work correctly

### 7.4 Two-Pass Compilation Tests ✅ COMPLETE (from Phase 6)

- [x] Test: Pass 1 registers all definitions (3/3 tests passing)
- [x] Test: Pass 2 resolves space correctly
- [x] Test: Error on missing space definition
- [x] Test: Error on multiple space definitions

### 7.5 Integration Tests ✅ COMPLETE

- [x] Test: Complete LED circuit with materials, profile, space (10/10 tests passing)
- [x] Test: Profile reference in space works
- [x] Test: Forward references work (space before profile)
- [x] Test: Order independence (definitions in any order)
- [x] Test: Duplicate material detection
- [x] Test: Multiple spaces detection
- [x] Test: No space detection
- [x] Test: Full pipeline validation

**Test Results**: All 10 integration tests pass ✅

**Architecture Validated**:
- ✅ Lexer → Parser → Symbol Table → Two-Pass Compiler → IR pipeline works
- ✅ Forward references work (define space before materials/profiles)
- ✅ Order independence works (definitions in any order)
- ✅ Error detection works (duplicates, missing definitions)
- ✅ Profile references in space are parsed correctly

**Next Steps**: Return to Phase 6.4 and 6.5 to implement material/constraint population

------

## Phase 8: Documentation (Rustdoc) ⏭️ OPTIONAL

**Status**: ⏭️ Deferred - Not blocking for System 2-5 integration

**When to implement**: Before public v0.1.4 release

### 8.1 Code Documentation ⏭️ DEFERRED

- [ ] Add rustdoc comments to Symbol Table
- [ ] Add rustdoc comments to all `parse_*` methods
- [ ] Add rustdoc comments to two-pass compilation
- [ ] Add rustdoc comments to module flattening
- [ ] Add rustdoc comments to comptime evaluation
- [ ] Document measurement conversion functions
- [ ] Generate rustdoc with `cargo doc`

### 8.2 User Documentation ⏭️ DEFERRED

- [ ] Update grammar file for unified syntax
- [ ] Document Symbol Table architecture
- [ ] Document two-pass compilation flow
- [ ] Document module flattening and comptime evaluation
- [ ] Document native SI unit parsing

**Note**: This phase is documentation only and does not block System 2-5 integration.

