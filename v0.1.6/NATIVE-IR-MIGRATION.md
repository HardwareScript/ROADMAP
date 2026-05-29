# Native IR Migration - Full Elimination of hardware_ir.rs

## Goal
Eliminate ALL intermediate representations. HardwareSpace is the ONLY IR.
Logic synthesizer directly mutates HardwareSpace. No bridges, no conversions.

## Summary

**Completed Phases:**
- ✅ Phase 1: Delete Legacy Files (all old IR files deleted/stubbed)
- ✅ Phase 2: Update lib.rs Exports (only program_to_space exported)
- ✅ Phase 3: Rewrite Logic Synthesizer (Native) - COMPLETE
  - Core structure with `&mut HardwareSpace`
  - All expression synthesis (literals, binary ops, etc.)
  - Control flow synthesis (if/match with MUX placement)
  - Register synthesis
  - Statement synthesis with nested blocks
  - Direct mutation, zero intermediate storage
- ✅ Phase 4: Update IR Integration (native synthesizer integration)
- ✅ Phase 5: Fix Module Flattener (deprecated instantiate_module)
- ✅ Phase 6: Stub Optimizer (placeholder types)
- ✅ Phase 7: Update CLI Commands (use program_to_space only)

**Remaining Phases:**
- Phase 8: Update Tests
- Phase 9: Verify Compilation
- Phase 10: Run Tests
- Phase 11: Documentation Updates

## Phase 1: Delete Legacy Files

- [x] Delete `hwc/crates/hwc-compiler/src/hardware_ir.rs`
- [x] Delete `hwc/crates/hwc-compiler/src/two_pass_compiler.rs`
- [x] Delete `hwc/crates/hwc-compiler/src/ir_compiler.rs`
- [x] Delete `hwc/crates/hwc-compiler/src/synthesis_types.rs`
- [x] Delete `hwc/crates/hwc-compiler/src/logic_synthesizer/statement.rs`
- [x] Delete `hwc/crates/hwc-compiler/src/logic_synthesizer/expression.rs`
- [x] Delete `hwc/crates/hwc-compiler/src/logic_synthesizer/register.rs`
- [x] Delete `hwc/crates/hwc-compiler/src/logic_synthesizer/control_flow.rs`
- [x] Stub out `hwc/crates/hwc-compiler/src/optimizer.rs` (kept for exports)

## Phase 2: Update lib.rs Exports

- [x] Remove `hardware_ir` module declaration
- [x] Remove `two_pass_compiler` module declaration
- [x] Remove `ir_compiler` module declaration
- [x] Remove `synthesis_types` module declaration
- [x] Remove all `hardware_ir` type exports (HardwareIR, PlacedComponent, NetRoute, etc.)
- [x] Remove `compile_two_pass` export
- [x] Remove `compile_to_ir` export
- [x] Keep only `program_to_space` as the single compilation entry point

## Phase 3: Rewrite Logic Synthesizer (Native)

### Core Structure
- [x] Create `logic_synthesizer/mod.rs` with native architecture
- [x] `LogicSynthesizer` takes `&mut HardwareSpace` in constructor
- [x] `LogicSynthesizer` takes `&SymbolTable` in constructor
- [x] Remove `synthesized_components: Vec<PlacedComponent>` field
- [x] Remove `synthesized_nets: Vec<NetRoute>` field
- [x] `synthesize_logic_block()` returns `Result<Vec<WidthWarning>, SynthesisError>`
- [x] NO return of components/nets - direct mutation only

### Expression Synthesis (Native)
- [x] `synthesize_literal()` - directly calls `self.place_component()`
- [x] `synthesize_boolean()` - directly calls `self.place_component()`
- [x] `synthesize_binary_add()` - directly calls `self.place_component()` for adder
- [x] `synthesize_binary_sub()` - directly calls `self.place_component()` for subtractor
- [x] `synthesize_binary_and()` - directly calls `self.place_component()` for AND gate
- [x] `synthesize_binary_or()` - directly calls `self.place_component()` for OR gate
- [x] `synthesize_binary_xor()` - directly calls `self.place_component()` for XOR gate
- [x] `synthesize_binary_shl()` - directly calls `self.place_component()` for left shifter
- [x] `synthesize_binary_shr()` - directly calls `self.place_component()` for right shifter
- [x] `synthesize_binary_eq()` - directly calls `self.place_component()` for comparator
- [x] `synthesize_binary_neq()` - directly calls `self.place_component()` for comparator
- [x] `synthesize_binary_lt()` - directly calls `self.place_component()` for comparator
- [x] `synthesize_binary_gt()` - directly calls `self.place_component()` for comparator
- [x] `synthesize_binary_lte()` - directly calls `self.place_component()` for comparator
- [x] `synthesize_binary_gte()` - directly calls `self.place_component()` for comparator
- [x] `synthesize_binary_mul()` - directly calls `self.place_component()` for multiplier
- [x] `synthesize_binary_div()` - directly calls `self.place_component()` for divider
- [x] `synthesize_binary_mod()` - directly calls `self.place_component()` for modulo
- [x] After each component placement, directly call `self.add_net()` for connections

### Control Flow Synthesis (Native)
- [x] `synthesize_if_expression()` - directly places MUX component
- [x] `synthesize_match_expression()` - directly places MUX component (2-to-1, 4-to-1, etc.)
- [x] Handle match arm synthesis with direct placement
- [x] Create ground/zero constants with direct placement when needed
- [x] `synthesize_block_or_expr()` - handles blocks and expressions

### Register Synthesis (Native)
- [x] `synthesize_register()` - directly places Register component
- [x] Handle clock domain tracking
- [x] Validate clock domains after synthesis

### Statement Synthesis (Native)
- [x] `synthesize_statement()` - dispatcher for different statement types
- [x] `synthesize_let()` - handle wire declarations
- [x] `synthesize_assignment()` - handle assignments to wires/registers
- [x] Handle nested blocks with direct placement (via `synthesize_block_or_expr()`)

### Helper Methods (Native)
- [x] `place_component()` - calls `ComponentPlacer::place_component()` directly
- [x] `add_net()` - looks up component/pin IDs, calls `space.netlist.add_net()` directly
- [x] `generate_component_name()` - generates unique names
- [x] NO methods that return PlacedComponent or NetRoute
- [x] NO methods that build intermediate lists

## Phase 4: Update IR Integration ✅

- [x] Update `ir/logic.rs` to use native synthesizer
- [x] Remove all component/net list handling
- [x] `synthesize_and_place_logic()` just calls synthesizer, no conversion
- [x] Synthesizer mutates space directly, function returns warnings only
- [x] Remove all references to old IR types in export crate
- [x] Update `CompiledOutput` to remove `ir` field
- [x] Update all export functions to work with HardwareSpace only
- [x] Fix CLI build command to not use old IR

**Export Crate Updates:**
- Removed `HardwareIR`, `PlacedComponent`, `NetRoute`, `ParameterUnit` from all code
- Updated signatures: `gerber`, `gdsii`, `obj`, `glb`, `blender`, `bom`, `netlist`, `excellon`
- All export functions now take only `(&HardwareSpace, &SymbolTable, &Path)`
- Stubbed implementations marked with TODO for future HardwareSpace integration
- Zero technical debt from old IR system

## Phase 5: Fix Module Flattener ✅

- [x] Remove `use crate::hardware_ir` from `module_flattener.rs`
- [x] Verify `FlattenedModule` only uses parser types
- [x] Ensure no references to PlacedComponent or NetRoute
- [x] Deprecate `instantiate_module()` function (now in ir/placement.rs)

## Phase 6: Fix or Stub Optimizer ✅

- [x] Option A: Stub out optimizer completely (placeholder types)
- [x] Remove all `hardware_ir` imports
- [x] Provide placeholder types for compilation

## Phase 7: Update CLI Commands ✅

- [x] Update `build.rs` - remove `compile_two_pass()` call
- [x] Update `build.rs` - use only `program_to_space()`
- [x] Update `check.rs` - remove `compile_two_pass()` call
- [x] Update `check.rs` - use only `program_to_space()`

## Phase 8: Update Tests

- [ ] Find all tests using `compile_two_pass()`
- [ ] Update to use `program_to_space()` instead
- [ ] Find all tests using `HardwareIR` type
- [ ] Update to use `HardwareSpace` instead
- [ ] `hwc/crates/hwc-compiler/tests/two_pass_integration_test.rs`
- [ ] `hwc/crates/hwc-compiler/tests/recursive_mux_tree_test.rs`
- [ ] `hwc/crates/hwc-compiler/tests/array_index_arithmetic_test.rs`
- [ ] Any other tests in `hwc/crates/hwc-compiler/tests/`

## Phase 9: Verify Compilation ✅

- [x] Run `cargo check --workspace`
- [x] **BUILD SUCCESSFUL** with only warnings (no errors)
- [ ] Fix all compilation errors
- [ ] Ensure no references to deleted types remain
- [ ] Ensure no references to deleted functions remain

## Phase 10: Run Tests

- [ ] Run `cargo test --workspace`
- [ ] Fix failing tests
- [ ] Ensure all tests pass

## Phase 11: Documentation Updates

- [ ] Update CHANGELOG.md with breaking changes
- [ ] Note that `compile_two_pass()` is removed
- [ ] Note that `HardwareIR` is removed
- [ ] Note that `program_to_space()` is the only entry point
- [ ] Update any architecture docs that reference old IR

## Key Principles

1. **NO intermediate representations** - HardwareSpace is the only IR
2. **Direct mutation** - LogicSynthesizer mutates space, doesn't return data
3. **No conversion layers** - No bridges, no adapters, no converters
4. **Single source of truth** - HardwareSpace.netlist is the netlist
5. **Pre-release freedom** - Break everything, fix everything, no technical debt

## Success Criteria

- [ ] Zero references to `hardware_ir` module in codebase
- [ ] Zero references to `HardwareIR` type in codebase
- [ ] Zero references to `PlacedComponent` type in codebase (except in deleted files)
- [ ] Zero references to `NetRoute` type in codebase (except in deleted files)
- [ ] Zero references to `compile_two_pass()` in codebase
- [ ] `cargo check --workspace` passes
- [ ] `cargo test --workspace` passes
- [ ] Logic synthesis works end-to-end
- [ ] Build command works end-to-end
- [ ] Check command works end-to-end
