# Gap 5: Logic Synthesis Compiler Integration

**Status**: Parser Complete ✓ | Compiler Integration Pending

The parser successfully handles logic synthesis syntax. Now we need the compiler to translate logic AST into physical hardware.

---

## Gap 5.1: Semantic Lowering (Pass 2 Compilation)

**Problem**: Parser creates AST for `let result = a + b`, but compiler doesn't translate it to physical hardware.

**Tasks**:
- [x] Implement logic block visitor in `hwc-compiler/src/two_pass_compiler.rs`
- [x] Add `LogicSynthesizer` struct to handle logic-to-hardware translation
- [x] Map binary operators to stdlib components:
  - [x] `+` → `RippleCarryAdder` from stdlib
  - [x] `-` → `Subtractor` from stdlib
  - [x] `&`, `|`, `^` → Logic gates from stdlib
  - [x] `<<`, `>>` → Barrel shifters from stdlib
- [x] Map control flow to MUX components:
  - [x] `if/else` → 2-to-1 MUX (inline and multi-line)
  - [x] `match` → N-to-1 MUX (2, 4, 8, 16 arms)
- [x] Support multi-line if expressions in assignments:
  - [x] Modify `LogicExpression::If` to use `BlockOrExpr` instead of `Box<LogicExpression>`
  - [x] Update `parse_if_inline_expression` to call `parse_block_or_expr()`
  - [x] Update synthesizer to handle block bodies in if expressions
- [x] Map `Reg()` to D flip-flop instantiation
- [x] Generate physical routing between synthesized components
- [x] Add logic synthesis to HardwareIR generation

---

## Gap 5.2: Pin-to-Variable Binding ✅

**Problem**: No validation that logic variables match declared module pins.

**Status**: COMPLETE - Variables validated against pins, wires, and imported definitions

**Tasks**:
- [x] Create `ElectricalSymbolTable` in compiler
- [x] Load module pins into symbol table before parsing logic block
- [x] Validate all logic variables are either:
  - [x] Declared pins
  - [x] Internal wires (`let` statements)
  - [x] Imported definitions (components, modules, enums, structs from symbol table)
- [x] Add semantic error: `Error [L01]: Unbound wire '{name}' in logic block`
- [x] Add hint showing available pins and wires
- [x] Check global symbol table for imported definitions when variable not found locally

---

## Gap 5.3: Bit-Width Inference & Validation

**Problem**: No validation that wire widths match on assignments.

**Tasks**:
- [x] Implement `WidthInferencePass` in compiler
- [x] Track bit width for all expressions:
  - [x] Literals: `0xFF` = 8 bits
  - [x] Variables: Use declared width `let x[16]`
  - [x] Operations: `a[8] + b[8]` = 9 bits (with carry)
  - [x] Slicing: `x[7..0]` = 8 bits
- [x] Validate assignments: `let out[4] = expr` requires expr to be 4 bits
- [x] Add semantic error: `Error [L02]: Width mismatch: Cannot assign {src_width}-bit value to {dst_width}-bit wire`
- [x] Add hint: `Use slicing: result[{dst_width-1}..0]` (via error message)
- [x] Handle width coercion rules (truncation, sign extension)

---

## Gap 5.4: Import & Standard Library Resolution ✅

**Problem**: `import ALU from @std/logic/alu` is parsed but not resolved.

**Status**: COMPLETE - Verified working with test files

**Tasks**:
- [x] Implement `ModuleResolver` in compiler
- [x] Add stdlib path resolution:
  - [x] `@std/*` → `hwc/stdlib/`
  - [x] `@org/pkg` → external packages (error for now)
- [x] Update lexer to tokenize `@std/path/to/module` as single `ImportPath` token
- [x] Update parser to handle `ImportPath` token
- [x] Load and parse imported files during Pass 0
- [x] Merge imported definitions into global symbol table
- [x] Handle circular import detection
- [x] Add semantic error: `Error [C24]: Cannot resolve import '{path}'`
- [x] Cache parsed imports to avoid re-parsing
- [x] Verified: `examples/test-import.hw` compiles successfully

---

## Gap 5.5: Logic-Specific Error Codes & Diagnostics

**Problem**: Generic syntax errors don't guide hardware engineers to fix issues.

**Tasks**:
- [x] Add logic-specific error codes to `error_codes.rs`:
  - [x] `L01`: Unbound wire/variable
  - [x] `L02`: Width mismatch
  - [x] `L03`: Combinational loop detected
  - [x] `L04`: Clock domain crossing without synchronizer
  - [x] `L05`: Multiple drivers on single wire
  - [x] `L06`: Uninitialized register
  - [x] `L07`: Invalid enum variant
  - [x] `L08`: Struct field mismatch
- [x] Update `ElectricalSymbolError` with error codes and hints
- [x] Update `WidthError` with error codes and hints
- [x] Add domain-specific help messages
- [x] Add code examples in error hints
- [x] Create v0.1.4 error handling philosophy document

---

## Gap 5.6: Register & Clock Domain Handling ✅

**Problem**: `Reg(clock: Clk, reset: Rst, init: 0)` needs proper clock domain tracking.

**Status**: COMPLETE - Full register synthesis with clock/reset routing

**Tasks**:
- [x] Track clock domains in symbol table
- [x] Validate all registers in same module use same clock
- [x] Detect clock domain crossings
- [x] Require explicit synchronizers for CDC
- [x] Add warning: `Warning [L04]: Clock domain crossing detected`
- [x] Extract clock signal from Reg() expressions
- [x] Store clock domain per register in ElectricalSymbolTable
- [x] Validate no multiple clock domains in single module
- [x] Check for CDC when assigning to register.next
- [x] Route clock signal to CLK pin of flip-flop
- [x] Route reset signal to RST pin of flip-flop
- [x] Route init value to D input of flip-flop
- [x] Select appropriately sized register component (8/16/32/64 bit)
- [x] Created test files: test-single-clock.hw, test-clock-domain.hw, test-register-routing.hw

---

## Gap 5.7: Enum & Struct Type Checking ✅

**Problem**: No validation that enum variants and struct fields are used correctly.

**Status**: COMPLETE - Enum and struct validation implemented

**Tasks**:
- [x] Register enum definitions in type system (already done in Pass 1)
- [x] Register struct definitions in type system (already done in Pass 1)
- [x] Validate enum variant access: `State.Idle` in match patterns
- [x] Validate struct field access: `instr.opcode`
- [x] Check `as` casts are valid: `RawInstr as Instruction`
- [x] Add semantic error: `Error [L07]: Unknown enum variant '{variant}' for enum '{enum_name}'`
- [x] Add semantic error: `Error [L08]: Unknown field '{field}' for struct '{struct_name}'`
- [x] Implement FieldAccess expression synthesis with struct validation
- [x] Implement Cast expression synthesis with type validation
- [x] Implement ArrayAccess (bus slicing) synthesis
- [x] Implement Bundle (concatenation) synthesis
- [x] Created test files: test-enum-validation.hw, test-enum-error.hw, test-struct-validation.hw, test-struct-error.hw

---

## Gap 5.8: Combinational Loop Detection ✅

**Problem**: No detection of invalid feedback loops without registers.

**Status**: COMPLETE - Combinational loops detected during Pass 1 validation

**Tasks**:
- [x] Build dependency graph for all logic expressions
- [x] Detect cycles in combinational logic
- [x] Allow cycles only through `Reg()` boundaries
- [x] Add semantic error: `Error [L03]: Combinational loop detected: {wire_chain}`
- [x] Add hint: `Insert a register to break the loop: let x = Reg(...)`
- [x] Implement proper semantic pass ordering (Gap 5.13)

**Implementation Notes**:
- Dependency graph built in Pass 1 before any other validation
- Loop detection runs before width inference to avoid error masking
- Registers properly marked to allow feedback through flip-flops
- Test files: `test-combinational-loop.hw`, `test-combinational-loop-ok.hw`, `test-complex-loop.hw`

---

## Gap 5.9: Standard Library Components ✅

**Problem**: Need actual implementations of synthesized components.

**Status**: COMPLETE - Parser handles complex nested conditionals with bundles

**Tasks**:
- [x] Implement `stdlib/logic/adders.hw`:
  - [x] RippleCarryAdder (8, 16, 32, 64 bit)
  - [x] CarryLookaheadAdder
  - [x] Full test coverage with complex nested operations
- [x] Implement `stdlib/logic/gates.hw`:
  - [x] AND, OR, XOR, NOT, NAND, NOR gates
  - [x] Multi-input gates (2, 4, 8 inputs)
  - [x] Priority encoder (8-bit with nested if/else)
  - [x] Barrel shifter (8-bit with match expressions)
  - [x] Gate arrays (4x8 parallel operations)
  - [x] Complex nested if/else with bundle expressions
- [x] Implement `stdlib/logic/mux.hw`:
  - [x] Mux2to1, Mux4to1, Mux8to1, Mux16to1
- [x] **Parser Enhancements** (Foundational Fixes):
  - [x] **Bracket Balancing**: Lexer skips newlines inside `[]` and `()` for clean multi-line arrays
  - [x] **Comma Continuation**: Lexer supports multi-line pin declarations without brackets
  - [x] **Linear AST Transformation**: Parser uses forward-only LL(1) parsing, then transforms single-expression blocks into expressions (rustc-style)
  - [x] **Multi-line if/else**: Full support for indented if/else expressions with bundles
  - [x] **Nested Conditionals**: Unlimited nesting depth for if/else and match expressions
  - [x] **Comment Handling**: Comments now work inside if/else blocks, match expressions, and bundle arrays
- [x] Implement `stdlib/logic/shifters.hw`:
  - [x] BarrelShifter, LogicalShift, ArithmeticShift
- [x] Implement `stdlib/logic/registers.hw`:
  - [x] DFlipFlop, Register, RegisterFile

**Metadata Decision**: Component metadata (area, delay, power) does NOT belong in logic modules. Logic modules are pure behavioral descriptions - Platonic ideals free from physical reality. Physical metadata belongs in the target-specific component library (PDK for ASIC, 74HC chips for PCB) that the synthesizer pulls from during compilation. This preserves the logical/physical duality.

**Implementation Notes**:
- `gates.hw` includes Priority_Encoder8 with 8 levels of nested if/else using bundle expressions
- `mux.hw` includes Mux2to1, Mux4to1, Mux8to1, Mux16to1 with extreme complexity stress tests
- `shifters.hw` includes 8 shifter variants with barrel shifter decomposition and mode selection
- `registers.hw` includes 10 register modules with state machines, register files, and pipeline stages
- Parser successfully handles complex expressions like `[7[3], 1[1], (0 * 4)]` in indented blocks
- Lexer uses industry-standard bracket balancing (Python/F#/Ruby style)
- Parser uses Linear AST Transformation (rustc style) - parse forward, transform post-parsing
- Comment bug fixed: `self.skip_whitespace()` now properly clears comments in all contexts
- All stdlib files compile successfully (1500+ lines total)

**Metadata Strategy**: Logic modules are pure behavioral descriptions - they represent the Platonic ideal of a gate/adder/mux, free from physical constraints. Physical metadata (area, delay, power) belongs in the target-specific component library (PDK for ASIC, 74HC for PCB) that the synthesizer references during compilation. This preserves the logical/physical duality and allows the same logic to compile for different targets (5nm ASIC vs discrete PCB chips).

**Next Steps**: Gap 5.10 - Integration testing, then Gap 5.11 - Ghost Synthesis (translate logic AST to physical HardwareIR)

---
## Gap 5.10: Integration Testing ✅

**Status**: COMPLETE - All integration tests passing

**Tasks**:
- [x] Test simple ALU compilation end-to-end
  - Result: 5 components (adder, subtractor, AND, OR, MUX), 14 routes
- [x] Test state machine compilation
  - Result: 5 components (register, state transition logic), 19 routes
- [x] Test counter with register
  - Result: 6 components (register, adder, MUX), 10 routes
- [x] Test nested match expressions
  - Result: 27 components (12 operations + 5 muxes + constants), 50 routes
- [x] Test imported logic modules
  - Result: 4 components, 9 routes
- [x] Verify all tests compile without errors
- [x] Verify synthesized component counts are non-zero

**Test Files Created**:
- `hwc/tests/logic_integration/test_simple_alu.hw` - 4-operation ALU
- `hwc/tests/logic_integration/test_state_machine.hw` - Enum-based FSM
- `hwc/tests/logic_integration/test_counter.hw` - Register with feedback
- `hwc/tests/logic_integration/test_nested_match.hw` - Complex nested multiplexers
- `hwc/tests/logic_integration/test_import_logic.hw` - Standard library imports

**Key Findings**:
- Logic synthesis successfully generates hardware from behavioral descriptions
- Complex nested expressions automatically create multiplexer trees
- Enum variants correctly infer bit widths
- Register feedback loops work correctly with clock domains
- Import system works with logic modules

**Next Steps**: Gap 5.11 already complete, move to Gap 5.12 (miette diagnostics)

---

## Gap 5.11: Ghost Synthesis (Zero Components Bug) ✅

**Problem**: Compiler successfully validates logic block semantics but doesn't generate any physical components or routes. The flattening pass reports "Flattened 0 components" and "Flattened 0 routes", causing the exporter to panic with "Component definition not found".

**Root Cause**: The semantic checker validates the logic AST but the synthesis lowering pass wasn't connecting assignment statement results to their target wires.

**Status**: COMPLETE - Logic synthesis now generates physical components and routes

**Tasks**:
- [x] Ensure `LogicSynthesizer` is called after semantic validation passes
- [x] Verify synthesized components are added to `HardwareIR.components`
- [x] Verify synthesized routes are added to `HardwareIR.routes`
- [x] Fix `synthesize_assignment` to create routes from expression outputs to target wires
- [x] Add integration test that verifies flattened component count > 0

**Implementation Notes**:
- Fixed `synthesize_assignment()` in `statement.rs` to create routes from synthesized expression outputs to assignment targets
- The synthesizer was generating components correctly but not connecting them to the output pins
- Test case: Simple ALU with 4 operations generates 5 components (adder, subtractor, AND, OR, MUX) and 14 routes
- Verified with `hwc check` showing correct component and route counts

**Example**:
```hw
logic {
    let result = A + B  // Now generates: add Adder_8Bit named _add_1
                        // And generates: route A to _add_1.InA
                        // And generates: route B to _add_1.InB
                        // And generates: route _add_1.Out to result
}
```

---

## Gap 5.12: Lost miette Visuals (UX Gap)

**Problem**: Semantic errors print generic text messages without beautiful miette code snippets showing exactly where the error occurred. The AST contains `Span` data (line/column), but errors lose this information before reaching the CLI.

**Root Cause**: `LogicError`, `ElectricalSymbolError`, and `WidthError` enums return generic `String` or error types without `#[label(...)]` macros, stripping span data.

**Status**: PARTIAL - Basic miette integration complete, but not Rust-level quality yet

**Completed Tasks**:
- [x] Update `ElectricalSymbolError` to implement `miette::Diagnostic`
- [x] Update `WidthError` to implement `miette::Diagnostic`
- [x] Update `SynthesisError` to implement `miette::Diagnostic`
- [x] Add `SourceSpan` fields to error variants
- [x] Add `#[label("...")]` macros to highlight exact error location
- [x] Add `#[help("...")]` macros for actionable hints
- [x] Propagate `Span` from AST nodes through semantic passes
- [x] Create `span_utils.rs` helper module for span conversion
- [x] Update error construction to capture spans from AST nodes
- [x] Make `SymbolError` implement `Diagnostic` with transparent propagation
- [x] Make `TwoPassError` implement `Diagnostic` with transparent propagation
- [x] Update CLI to use `miette::Report::new(e).with_source_code(source)`
- [x] Configure `GraphicalReportHandler` with Unicode theme in main.rs
- [x] Test that CLI shows code snippets with labeled spans

**Current Behavior**:
```
Error: L05 (link)
  ├─ Wire 'x' already declared
   ╭─[5:1]
 5 │         Out[8]
 6 │
 7 │     logic:
 8 │         let x = 5
   ·         ─────────
   ·              ╰── first declared here
 9 │         let x = 10
   ·         ──────────
   ·              ╰── duplicate wire declared here
10 │
11 │         Out = x
   ╰────
  help: Each wire name must be unique within a logic block
```

**Remaining Tasks for Rust-Level Quality**:
- [ ] **Multi-error reporting**: Rust shows ALL errors in a file, not just the first one
  - [ ] Collect all errors during compilation instead of early-exit
  - [ ] Display all errors with proper grouping and priority
  - [ ] Add `--error-limit` flag (default 10, like rustc)
- [ ] **Suggestion system**: Rust suggests fixes with "did you mean?" hints
  - [ ] Implement fuzzy string matching for undefined wires (Levenshtein distance)
  - [ ] Suggest similar wire names: `Error: Unbound wire 'resut'` → `help: did you mean 'result'?`
  - [ ] Suggest available pins when wire not found
  - [ ] Suggest correct enum variants when variant is wrong
- [ ] **Fix-it hints**: Rust shows exact code to fix the error
  - [ ] Add `note: try this:` with corrected code snippet
  - [ ] Show before/after diffs for complex fixes
  - [ ] Support `--fix` flag to auto-apply suggestions (like `cargo fix`)
- [ ] **Related information**: Rust shows context from other locations
  - [ ] Add `note: first declared here` with span to original declaration
  - [ ] Show related type definitions when type mismatch occurs
  - [ ] Link to documentation URLs for error codes
- [ ] **Warning system**: Rust has warnings separate from errors
  - [ ] Implement `Warning` diagnostic level (currently only errors)
  - [ ] Add warnings for implicit truncation, unused wires, etc.
  - [ ] Add `--deny-warnings` flag to treat warnings as errors
- [ ] **Error recovery**: Rust continues parsing after errors to find more issues
  - [ ] Implement error recovery in parser (currently panics on first error)
  - [ ] Continue semantic analysis with placeholder values after errors
  - [ ] Show multiple errors in single compilation run
- [ ] **Color and formatting**: Rust has sophisticated terminal output
  - [ ] Verify color output works on Windows/Linux/Mac terminals
  - [ ] Add `--color=auto|always|never` flag
  - [ ] Test with different terminal widths and Unicode support
- [ ] **Error code documentation**: Rust has `rustc --explain E0308`
  - [ ] Create detailed documentation for each error code (L01-L10, C01-C30, etc.)
  - [ ] Add `hwc explain L05` command to show detailed error explanation
  - [ ] Include examples of correct and incorrect code for each error
- [ ] **Span precision**: Rust highlights exact tokens, not whole lines
  - [ ] Improve span precision to highlight specific identifiers/operators
  - [ ] Currently spans are sometimes too broad (whole statement vs specific token)
  - [ ] Add multi-span labels for complex errors (e.g., "this needs to match this")
- [ ] **Parser error quality**: Rust parser gives excellent error messages
  - [ ] Current parser errors are generic (e.g., "expected token")
  - [ ] Need context-aware error messages (e.g., "expected `:` after module name")
  - [ ] Add recovery suggestions (e.g., "did you forget a comma?")

**Gap Assessment**:
- **Current state**: Basic miette integration works, errors show source code with spans
- **Rust-level quality**: We're at ~30% of Rust's error handling sophistication
- **Biggest gaps**: Multi-error reporting, suggestion system, error recovery, parser error quality
- **Priority**: Focus on multi-error reporting and suggestion system first (highest user impact)

**Example of Rust-Level Error (Target)**:
```
error[L01]: unbound wire 'resut' in logic block
  --> examples/test.hw:8:18
   |
 8 |     let output = resut + 5
   |                  ^^^^^ not found in this scope
   |
help: a wire with a similar name exists
   |
 7 |     let result = A + B
   |         ------ similarly named wire defined here
   |
help: did you mean `result`?
   |
 8 |     let output = result + 5
   |                  ~~~~~~

error[L02]: width mismatch in assignment
  --> examples/test.hw:9:9
   |
 9 |     let out[4] = output
   |         ^^^^^^   ------ this expression has type `wire[9]`
   |         |
   |         expected 4-bit wire, found 9-bit wire
   |
help: use slicing to truncate
   |
 9 |     let out[4] = output[3..0]
   |                        +++++++

error: aborting due to 2 previous errors

For more information about these errors, try `hwc explain L01`.
```

---

## Gap 5.13: Error Precedence Masking (Semantic Pass Ordering) ✅

**Problem**: When a wire is undefined (L01 error), the compiler throws a width inference error (L02) instead because it tries to calculate the width before checking if the variable exists.

**Root Cause**: Semantic passes run in wrong order. Width inference runs before name resolution.

**Status**: COMPLETE - Proper pass ordering implemented

**Tasks**:
- [x] Enforce semantic pass ordering in `two_pass_compiler.rs`:
  1. **Pass 1: Dependency Analysis & Loop Detection**: Detect combinational loops (L03 errors)
  2. **Pass 2: Name Resolution**: Check all variables exist (L01 errors)
  3. **Pass 3: Width Inference**: Calculate and validate widths (L02 errors)
  4. **Pass 4: Hardware Generation**: Generate components and routing
- [x] Add early-exit on first error category (don't cascade errors)
- [x] Pre-register all let bindings before evaluating expressions
- [x] Integration test: undefined wire should throw L01, not L02
- [x] Document pass ordering in implementation

**Implementation Notes**:
- Logic synthesizer now runs 4 distinct passes in strict order
- Each pass completes fully before the next begins
- Errors are caught at the earliest possible pass
- Forward references work correctly (concurrent hardware semantics)
- Test verified: `test-combinational-loop.hw` correctly throws L03, not L02

**Example**:
```hw
logic {
    let result = unknown_wire + 5  // Now throws L01 (unbound), not L02 (width)
}
```

**Current Behavior**: `Error [L03]: Combinational loop detected: x → y → x`  
**Previous Behavior**: `Error [L02]: Cannot infer width for variable 'y'` (masked the real issue)

---

## Gap 5.14: Bottom-Up Width Inference (Automatic Type Deduction) ✅

**Problem**: Users must explicitly annotate widths everywhere (`let A[8] = ...`), making the language tedious. Modern HDLs (Chisel, Amaranth) infer widths from right-hand side expressions.

**Root Cause**: Width inference is top-down (left-to-right) instead of bottom-up (right-to-left).

**Status**: COMPLETE - Automatic width inference with explicit truncation warnings

**Tasks**:
- [x] Implement bottom-up width inference in `width_inference.rs`:
  - [x] Infer width from RHS expression: `let A = B + C` where B=8, C=8 → A=9
  - [x] Infer width from literals: `let x = 0xFF` → x=8
  - [x] Infer width from operations: `let y = x << 2` → y=width(x)+2
  - [x] Allow explicit override: `let A[4] = B + C` → truncate to 4 bits with warning
- [x] Update `ElectricalSymbolTable` to store inferred widths
- [x] Add width propagation for chained assignments
- [x] Add warning for implicit truncation:
  - [x] Created `WidthWarning` enum with `ImplicitTruncation` variant
  - [x] Created `WidthValidationResult` enum to return Ok/Warning/Error
  - [x] Modified `validate_assignment` to accept `explicit_width` parameter
  - [x] When explicit width is smaller than expression, return warning instead of error
  - [x] Warnings are collected and displayed during compilation
- [x] Update documentation with width inference rules
- [x] Add integration tests for automatic width deduction

**Implementation Notes**:
- Width inference was already partially implemented in Gap 5.3
- The key addition is allowing explicit truncation with warnings
- When a user writes `let D[4] = expr` where expr is 9 bits, the compiler now:
  1. Allows the truncation (it's explicit)
  2. Issues warning L10: "Implicit truncation: 9-bit expression assigned to 4-bit wire"
  3. Suggests using explicit slicing: `D = expr[3..0]`
- Warnings are displayed during compilation but don't prevent success
- Test file: `test_width_warning.hw` demonstrates the feature

**Example**:
```hw
// Before (tedious):
let A[8] = input
let B[8] = 0xFF
let C[9] = A + B
let result[4] = C[3..0]

// After (automatic):
let A = input      // Infer from input pin width
let B = 0xFF       // Infer 8 bits from literal
let C = A + B      // Infer 9 bits (8 + 8 with carry)
let result[4] = C  // Explicit truncation (warning issued: ⚠️  Implicit truncation: 9-bit expression assigned to 4-bit wire 'result')
```

**Output Example**:
```
🔍 Checking: test_width_warning.hw
✅ Syntax valid
⚠️  1 warning(s) in module 'WWT1'
   Implicit truncation: 9-bit expression assigned to 4-bit wire 'D'
✅ Semantic validation passed
```
