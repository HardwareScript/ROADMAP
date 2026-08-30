# Phase 0: Lexical, Grammar & AST Extensions

**Target Crate:** `crates/hwc-parser`  
**Pre-requisites:** None (Foundation Layer)  
**Blocks Downstream:** Phase 1 (`hwc-engine`), Phase 2 (`hwc-compiler::eval`), Phase 3 (`hwc-synthesis`)  
**Specification References:**
* [Digital-Logic-Synthesis.md  10.1](file:///c:/Users/olowo/Downloads/Code/hw/Docs/v0.3.1/Digital-Logic-Synthesis.md) (`logic { ... }` behavioral syntax)
* [Stable-Structural-Identity.md  2.1](file:///c:/Users/olowo/Downloads/Code/hw/Docs/v0.3.1/Stable-Structural-Identity.md) (Loop semantic keys `key: "..."`)
* [Comptime-Virtual-Machine.md  3.2](file:///c:/Users/olowo/Downloads/Code/hw/Docs/v0.3.1/Comptime-Virtual-Machine.md) (`#[comptime_fuel(...)]` attributes)

---

## 1. Core Responsibilities & Tasks

- [x] **1. Lexer & Grammar Tokens**
  - [x] Support explicit brace blocks `{ ... }` and natural boolean logic keywords (`and`, `or`, `not`).
  - [x] Support assignment and arithmetic tokens: `+=`, `-=`, `match`, and loop ranges (`0..N`, `0..=N`).

- [x] **2. Loop Semantic Keys in AST**
  - [x] Extend `ForStmt` AST node to parse optional semantic loop keys: `for ch in 0..4 key: "chan_{ch}" { ... }`.
  - [x] Preserve semantic key expressions in AST for lowering into `PathSegment::SemanticKey`.

- [x] **3. Behavioral Logic Blocks (`logic { ... }`)**
  - [x] Implement AST nodes for `logic { ... }` blocks inside modules and spaces.
  - [x] Implement sequential register declarations: `reg state: Int = 0 on: clk.posedge reset_to: 0 when: not rst_n`.
  - [x] Implement combinational assignments and next-state assignments: `state.next = 1`, `data_out = ...`.

- [x] **4. Comptime Attributes**
  - [x] Implement parser support for space/module attributes such as `#[comptime_fuel(500_000_000)]`.
  - [x] Attach parsed attributes to `SpaceDecl` and `ModuleDecl` AST structures.

---

## 2. Verification Gate

- [x] `parser::test_parse_logic_block()` cleanly parses sequential registers, conditions, and bitwise expressions.
- [x] `parser::test_loop_semantic_key()` cleanly parses loop keys with exact `SourceSpan` tracking.
- [x] Attribute parsing tests verify `#[comptime_fuel]` is correctly extracted.
