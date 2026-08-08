# AST Arena Refactor - Implementation Checklist

**Version:** v0.2.1  
**Date:** 2026-08-08  
**Reference:** See `AST-ARENA-REFACTOR.md` for architectural rationale  
**Goal:** Migrate from mixed `Box<T>` + `&'ast T` system to pure index-based arena

---

## Overview

This document provides a comprehensive, step-by-step checklist for implementing the AST Arena Refactor across the entire HardwareScript codebase. Each section contains actionable items with sub-tasks to prevent getting lost during implementation.

**Refactor Pattern:**
```rust
// BEFORE (Mixed Box + Lifetime):
pub enum Statement<'ast> {
    Component(Box<ComponentPlacement>),
    Contact(&'ast ContactPlacement),
}

// AFTER (Pure Index):
pub enum Statement {
    Component(ComponentId),
    Contact(ContactId),
}
```

**Key References:**
- Arena infrastructure: `hwc-parser/src/ast/arena.rs` (already implemented)
- ID types already defined: ComponentId, PourId, PlaneId, PolygonId, ContactId, RouteId, SpaceInstanceId, ForLoopId, StmtId, ExprId
- AstArena already has allocation methods: `alloc_component()`, `alloc_pour()`, etc.

---

## ⚠️ CRITICAL REFACTORING PRINCIPLES

### **NO BACKWARD COMPATIBILITY**
This is v0.2.1 (pre-1.0) - we are **NOT maintaining backward compatibility**. Delete old code aggressively:

- **DELETE** all `Box<T>` wrapper code when replaced with IDs
- **DELETE** all `&'ast T` reference handling when replaced with IDs  
- **DELETE** bumpalo arena code completely (parser owns AstArena now)
- **DELETE** any helper methods that clone arena-allocated types (use iterator + arena instead)
- **DELETE** deprecated/unused enum variants or fields
- **NO** `#[allow(dead_code)]` or `#[deprecated]` markers - just delete it

### **CLIPPY COMPLIANCE - ZERO WARNINGS**
Every change must satisfy Clippy without suppressions:

- **NO** `#[allow(clippy::large_enum_variant)]` - arena IDs eliminate this warning naturally
- **NO** `#[allow(clippy::too_many_arguments)]` - if functions need too many params, refactor into a struct
- **NO** `#[allow(clippy::type_complexity)]` - use type aliases for complex types
- **USE** proper struct patterns that Clippy approves of:
  ```rust
  // GOOD - Clippy approved
  pub struct SpaceDefinition {
      pub name: Identifier,
      pub statements: Vec<StmtId>,  // Small 4-byte IDs
  }
  
  // BAD - Would trigger large_enum_variant
  pub enum Statement {
      Component(ComponentPlacement),  // Don't inline large structs
  }
  
  // GOOD - Clippy approved  
  pub enum Statement {
      Component(ComponentId),  // 4 bytes, no warning
  }
  ```

### **CLEAN CODE RULES**
- All enum variants must be under 128 bytes (arena IDs make this automatic)
- Function parameters: max 7 parameters (use config structs for more)
- No unnecessary cloning - use references or cheap Copy types (like IDs)
- Type aliases for complex signatures:
  ```rust
  // Instead of: Result<Vec<(ComponentId, &ComponentPlacement)>, ParseError>
  type ComponentList<'a> = Vec<(ComponentId, &'a ComponentPlacement)>;
  type ParseResult<T> = Result<T, ParseError>;
  ```

---

## Phase 1: Core AST Type Definitions

### 1.1 Update `hwc-parser/src/ast/space/space_def.rs`

**Reference:** See AST-ARENA-REFACTOR.md "The Solution" section, lines 400-600

- [ ] **Remove lifetime parameter from `SpaceDefinition`**
  - [ ] Change `pub struct SpaceDefinition<'ast>` → `pub struct SpaceDefinition`
  - [ ] Remove `<'ast>` from all field types: `statements: Vec<SpaceTopLevelStatement<'ast>>` → `statements: Vec<SpaceTopLevelStatement>`
  - [ ] Update `layouts` field: `Vec<ModuleLayoutBlock<'ast>>` → `Vec<ModuleLayoutBlock>`
  
- [ ] **Refactor `SpaceTopLevelStatement` enum**
  - [ ] Remove lifetime: `pub enum SpaceTopLevelStatement<'ast>` → `pub enum SpaceTopLevelStatement`
  - [ ] Change `Component(Box<ComponentPlacement>)` → `Component(ComponentId)`
  - [ ] Change `Pour(Box<PourPlacement>)` → `Pour(PourId)`
  - [ ] Change `Plane(Box<PlanePlacement>)` → `Plane(PlaneId)`
  - [ ] Change `Contact(&'ast ContactPlacement)` → `Contact(ContactId)`
  - [ ] Change `Route(&'ast Route)` → `Route(RouteId)`
  - [ ] Change `SpaceInstance(Box<SpaceInstancePlacement>)` → `SpaceInstance(SpaceInstanceId)`
  - [ ] Change `ForLoop(SpaceForLoop<'ast>)` → `ForLoop(ForLoopId)`
  - [ ] Keep unchanged: `Polygon(PolygonPlacement)`, `Expose(Expose)`, `RouteNetPolicy(RouteNetPolicy)`, `Region(RegionDefinition)`, `Let(LetBinding)`, `Const(ConstBinding)`

- [ ] **Refactor `SpaceForLoop` struct**
  - [ ] Remove lifetime: `pub struct SpaceForLoop<'ast>` → `pub struct SpaceForLoop`
  - [ ] Change `body: Vec<SpaceStatement<'ast>>` → `body: Vec<SpaceStatement>`

- [ ] **Refactor `SpaceIfConditional` struct**
  - [ ] Remove lifetime: `pub struct SpaceIfConditional<'ast>` → `pub struct SpaceIfConditional`
  - [ ] Change `then_body: Vec<SpaceStatement<'ast>>` → `then_body: Vec<SpaceStatement>`
  - [ ] Change `else_body: Vec<SpaceStatement<'ast>>` → `else_body: Vec<SpaceStatement>`

- [ ] **Refactor `SpaceStatement` enum**
  - [ ] Remove lifetime: `pub enum SpaceStatement<'ast>` → `pub enum SpaceStatement`
  - [ ] Change `Component(Box<ComponentPlacement>)` → `Component(ComponentId)`
  - [ ] Change `Pour(Box<PourPlacement>)` → `Pour(PourId)`
  - [ ] Change `Plane(Box<PlanePlacement>)` → `Plane(PlaneId)`
  - [ ] Change `Contact(&'ast ContactPlacement)` → `Contact(ContactId)`
  - [ ] Change `Route(&'ast Route)` → `Route(RouteId)`
  - [ ] Change `SpaceInstance(Box<SpaceInstancePlacement>)` → `SpaceInstance(SpaceInstanceId)`
  - [ ] Change `ForLoop(Box<SpaceForLoop<'ast>>)` → `ForLoop(ForLoopId)`
  - [ ] Change `If(SpaceIfConditional<'ast>)` → `If(SpaceIfConditional)`
  - [ ] Keep unchanged: `Let(LetBinding)`

- [ ] **DELETE old `impl SpaceDefinition` helper methods**
  - [ ] **DELETE** `components()` method that returns `Vec<ComponentPlacement>` (clones data unnecessarily)
  - [ ] **DELETE** `pours()`, `planes()`, `contacts()`, `for_loops()` methods (same reason)
  - [ ] **REPLACE** with zero-copy iterator methods:
    ```rust
    // NEW - Zero-copy, iterator-based access
    impl SpaceDefinition {
        pub fn component_ids(&self) -> impl Iterator<Item = ComponentId> + '_ {
            self.statements.iter().filter_map(|s| match s {
                SpaceTopLevelStatement::Component(id) => Some(*id),
                _ => None,
            })
        }
        
        // If caller needs actual data, they provide arena:
        pub fn components<'a>(&self, arena: &'a AstArena) -> impl Iterator<Item = &'a ComponentPlacement> + 'a {
            self.component_ids().map(move |id| &arena.components[id])
        }
    }
    ```
  - [ ] Keep `polygons()` as-is IF it returns direct values (not cloning)
  - [ ] Keep `route_net_policies()`, `regions_from_statements()` only if they don't clone

---

### 1.2 Update `hwc-parser/src/ast/mod.rs`

**Reference:** Current file uses `Definition<'ast>` with mixed allocation strategies

- [ ] **Refactor `Program` struct**
  - [ ] Remove lifetime: `pub struct Program<'ast>` → `pub struct Program`
  - [ ] Change `definitions: Vec<Definition<'ast>>` → `definitions: Vec<Definition>`

- [ ] **Refactor `Definition` enum**
  - [ ] Remove lifetime: `pub enum Definition<'ast>` → `pub enum Definition`
  - [ ] Change `Component(&'ast ComponentDefinition)` → `Component(ComponentDefinition)` or keep as direct value (not currently arena-allocated)
  - [ ] Change `Module(ModuleDefinition<'ast>)` → `Module(ModuleDefinition)`
  - [ ] Change `Space(Box<SpaceDefinition<'ast>>)` → `Space(SpaceDefinition)` (remove Box AND lifetime)
  - [ ] Keep other variants unchanged if they don't have lifetimes

---

### 1.3 Update `hwc-parser/src/ast/module.rs`

**Reference:** Module types currently use `<'ast>` for nested statements

- [ ] **Refactor `ModuleDefinition` struct**
  - [ ] Remove lifetime: `pub struct ModuleDefinition<'ast>` → `pub struct ModuleDefinition`
  - [ ] Change `statements: Vec<ModuleStatement<'ast>>` → `statements: Vec<ModuleStatement>`
  - [ ] Change `intrinsic_layout: Option<Vec<LayoutStatement<'ast>>>` → `Option<Vec<LayoutStatement>>`

- [ ] **Refactor `ModuleStatement` enum**
  - [ ] Remove lifetime: `pub enum ModuleStatement<'ast>` → `pub enum ModuleStatement`
  - [ ] Change `AddComponent(&'ast ModuleComponentPlacement)` → `AddComponent(ModuleComponentPlacement)` (direct value, likely not arena-allocated)
  - [ ] Change `For(ForLoop<'ast>)` → `For(ForLoop)`
  - [ ] Change `If(IfConditional<'ast>)` → `If(IfConditional)`
  - [ ] Keep `Route(ModuleRoute)` unchanged

- [ ] **Refactor `ForLoop` struct**
  - [ ] Remove lifetime: `pub struct ForLoop<'ast>` → `pub struct ForLoop`
  - [ ] Change `body: Vec<ModuleStatement<'ast>>` → `body: Vec<ModuleStatement>`

- [ ] **Refactor `IfConditional` struct**
  - [ ] Remove lifetime: `pub struct IfConditional<'ast>` → `pub struct IfConditional`
  - [ ] Change `then_body: Vec<ModuleStatement<'ast>>` → `then_body: Vec<ModuleStatement>`
  - [ ] Change `else_body: Option<Vec<ModuleStatement<'ast>>>` → `Option<Vec<ModuleStatement>>`

---

### 1.4 Update `hwc-parser/src/ast/space/layout.rs`

- [ ] **Refactor `ModuleLayoutBlock` struct**
  - [ ] Remove lifetime: `pub struct ModuleLayoutBlock<'ast>` → `pub struct ModuleLayoutBlock`
  - [ ] Change `statements: Vec<LayoutStatement<'ast>>` → `statements: Vec<LayoutStatement>`

- [ ] **Refactor `LayoutStatement` enum**
  - [ ] Remove lifetime: `pub enum LayoutStatement<'ast>` → `pub enum LayoutStatement`
  - [ ] Change any `&'ast T` references to direct values or IDs as appropriate
  - [ ] Update nested for loop bodies if present

---

### 1.5 Verify Other AST Files

- [ ] **Check `hwc-parser/src/ast/component.rs`**
  - [ ] Verify `ComponentPlacement` has no `<'ast>` lifetime
  - [ ] Verify `ComponentDefinition` has no `<'ast>` lifetime
  - [ ] If any lifetimes exist, remove them following the pattern above

- [ ] **Check `hwc-parser/src/ast/space/placements.rs`**
  - [ ] Verify `ContactPlacement`, `PourPlacement`, `PlanePlacement`, `PolygonPlacement`, `SpaceInstancePlacement` have no lifetimes
  - [ ] These should be plain structs with no `<'ast>` parameters

- [ ] **Check `hwc-parser/src/ast/space/routes.rs`**
  - [ ] Verify `Route` struct has no `<'ast>` lifetime
  - [ ] Verify `Expose` struct has no `<'ast>` lifetime

- [ ] **Check `hwc-parser/src/ast/expression.rs`**
  - [ ] Verify `Expression` enum has no `<'ast>` lifetime
  - [ ] If ExprId is used, ensure expressions can be arena-allocated

---

## Phase 2: Parser Infrastructure Changes

### 2.1 Update `hwc-parser/src/parser/mod.rs`

**Reference:** AST-ARENA-REFACTOR.md "Implementation Guide" section

- [ ] **Refactor `Parser` struct**
  - [ ] Remove lifetime: `pub struct Parser<'ast>` → `pub struct Parser`
  - [ ] Change `arena: &'ast Bump` → `arena: AstArena` (owned arena)
  - [ ] Update struct to own the arena instead of borrowing it
    ```rust
    pub struct Parser {
        tokens: Vec<SpannedToken>,
        current: usize,
        arena: AstArena,  // Owned, not borrowed
    }
    ```

- [ ] **Update `Parser::new()` constructor**
  - [ ] Change signature: `pub fn new(tokens: Vec<SpannedToken>, arena: &'ast Bump)` → `pub fn new(tokens: Vec<SpannedToken>) -> Self`
  - [ ] Initialize arena inside: `arena: AstArena::new()`
  - [ ] Remove arena parameter from function signature

- [ ] **Add method to extract arena after parsing**
  - [ ] Add `pub fn into_parts(self) -> (Program, AstArena)` method
  - [ ] This allows caller to get both the parsed AST and the arena

- [ ] **Update all `impl<'ast> Parser<'ast>` blocks**
  - [ ] Change `impl<'ast> Parser<'ast>` → `impl Parser` throughout the file
  - [ ] Update method signatures that return arena-allocated types

---

### 2.2 Update Parser Definition Files

**Pattern for ALL parser files:**
```rust
// OLD:
impl<'ast> Parser<'ast> {
    fn parse_foo(&mut self) -> Result<&'ast Foo, ParseError> {
        let foo = Foo { ... };
        Ok(self.arena.alloc(foo))
    }
}

// NEW:
impl Parser {
    fn parse_foo(&mut self) -> Result<FooId, ParseError> {
        let foo = Foo { ... };
        let id = self.arena.alloc_foo(foo);
        Ok(id)
    }
}
```

- [ ] **Update `hwc-parser/src/parser/definitions/space/core.rs`**
  - [ ] Change `impl<'ast> Parser<'ast>` → `impl Parser`
  - [ ] Change `parse_space()` return type: `Result<SpaceDefinition<'ast>, _>` → `Result<SpaceDefinition, _>`
  - [ ] Update all internal calls to match new signatures

- [ ] **Update `hwc-parser/src/parser/definitions/space/placements/contact.rs`**
  - [ ] Change `impl<'ast> Parser<'ast>` → `impl Parser`
  - [ ] Change `parse_contact()` return type: `Result<&'ast ContactPlacement, _>` → `Result<ContactId, _>`
  - [ ] Replace `Ok(self.arena.alloc(contact))` with `Ok(self.arena.alloc_contact(contact))`

- [ ] **Update `hwc-parser/src/parser/definitions/space/placements/pour.rs`**
  - [ ] Change `impl<'ast> Parser<'ast>` → `impl Parser`
  - [ ] Change return type to `Result<PourId, _>`
  - [ ] Use `self.arena.alloc_pour()`

- [ ] **Update `hwc-parser/src/parser/definitions/space/placements/plane.rs`**
  - [ ] Change `impl<'ast> Parser<'ast>` → `impl Parser`
  - [ ] Change return type to `Result<PlaneId, _>`
  - [ ] Use `self.arena.alloc_plane()`

- [ ] **Update `hwc-parser/src/parser/definitions/space/placements/polygon.rs`**
  - [ ] Change `impl<'ast> Parser<'ast>` → `impl Parser`
  - [ ] Keep return type as `Result<PolygonPlacement, _>` (direct value, not arena-allocated per current design)

- [ ] **Update `hwc-parser/src/parser/definitions/space/placements/space_instance.rs`**
  - [ ] Change `impl<'ast> Parser<'ast>` → `impl Parser`
  - [ ] Change return type to `Result<SpaceInstanceId, _>`
  - [ ] Use `self.arena.alloc_space_instance()`

- [ ] **Update `hwc-parser/src/parser/definitions/component/main.rs`**
  - [ ] Change `impl<'ast> Parser<'ast>` → `impl Parser`
  - [ ] Update `parse_component_def()` to remove lifetimes
  - [ ] Update return types and internal logic

- [ ] **Update `hwc-parser/src/parser/routing.rs`**
  - [ ] Change `impl<'ast> Parser<'ast>` → `impl Parser`
  - [ ] Change route parsing to return `RouteId`
  - [ ] Use `self.arena.alloc_route()`

- [ ] **Update `hwc-parser/src/parser/definitions/module/main.rs`**
  - [ ] Change `impl<'ast> Parser<'ast>` → `impl Parser`
  - [ ] Change return type: `Result<ModuleDefinition<'ast>, _>` → `Result<ModuleDefinition, _>`

- [ ] **Update `hwc-parser/src/parser/definitions/module/statements.rs`**
  - [ ] Change `impl<'ast> Parser<'ast>` → `impl Parser`
  - [ ] Update statement parsing to remove lifetimes
  - [ ] Update `parse_add_component()` return type as needed

- [ ] **Update `hwc-parser/src/parser/definitions/module/control_flow.rs`**
  - [ ] Change `impl<'ast> Parser<'ast>` → `impl Parser`
  - [ ] Change `parse_for_loop()` return: `Result<ForLoop<'ast>, _>` → `Result<ForLoop, _>`
  - [ ] Change `parse_if()` return: `Result<IfConditional<'ast>, _>` → `Result<IfConditional, _>`

- [ ] **Update all other parser definition files** (apply same pattern):
  - [ ] `definitions/const_def.rs`
  - [ ] `definitions/shape/geometry.rs`
  - [ ] `definitions/space/dimensions.rs`
  - [ ] `definitions/material.rs`
  - [ ] `definitions/interface.rs`
  - [ ] `definitions/mechanical.rs`
  - [ ] `definitions/signal_group.rs`
  - [ ] `definitions/module/pins.rs`
  - [ ] `definitions/test.rs`
  - [ ] `definitions/profile/*.rs` (all profile submodules)
  - [ ] `definitions/unit.rs`
  - [ ] `definitions/subcircuit.rs`
  - [ ] `definitions/spice_model.rs`
  - [ ] `definitions/pattern.rs`
  - [ ] `definitions/device.rs`
  - [ ] `definitions/mod.rs`
  - [ ] `expression.rs`
  - [ ] `logic/mod.rs`
  - [ ] `helpers/*.rs` (tokens, whitespace, utils)

---

### 2.3 Update Parser Entry Point

- [ ] **Update `hwc-parser/src/lib.rs`**
  - [ ] Update `parse()` function signature to remove arena parameter if present
  - [ ] Create arena inside parse function
  - [ ] Return both Program and AstArena: `Result<(Program, AstArena), ParseError>`
    ```rust
    pub fn parse(source: &str, file_path: Option<&str>) -> Result<(Program, AstArena), ParseError> {
        let tokens = lex(source, file_path)?;
        let mut parser = Parser::new(tokens);
        let program = parser.parse_program()?;
        let arena = parser.arena;  // Extract arena
        Ok((program, arena))
    }
    ```

---

## Phase 3: Compiler and Symbol Table Updates

### 3.1 Update `hwc-compiler/src/symbol_table/definition.rs`

- [ ] **Remove lifetime from `Definition` enum**
  - [ ] Change `pub enum Definition<'ast>` → `pub enum Definition`
  - [ ] Update all variants to remove `<'ast>` parameters
  - [ ] Verify all contained types no longer have lifetimes

---

### 3.2 Update `hwc-compiler/src/symbol_table/layer.rs`

- [ ] **Remove lifetime from `SymbolLayer`**
  - [ ] Change `pub struct SymbolLayer<'ast>` → `pub struct SymbolLayer`
  - [ ] Update field types: `FxHashMap<CompactString, Definition<'ast>>` → `FxHashMap<CompactString, Definition>`

- [ ] **Remove lifetime from `SymbolTable`**
  - [ ] Change `pub struct SymbolTable<'ast>` → `pub struct SymbolTable`
  - [ ] Update all `SymbolLayer<'ast>` fields → `SymbolLayer`
  - [ ] Update all method signatures


---

### 3.3 Update Compiler IR and Lowering

- [ ] **Check `hwc-compiler/src/ir/*.rs` files for lifetime dependencies**
  - [ ] Search for `<'ast>` in all IR type definitions
  - [ ] Remove lifetimes if they reference AST types
  - [ ] Update IR builders/lowering functions to accept arena references

- [ ] **Update compilation pipeline**
  - [ ] Modify compilation entry point to accept `AstArena`
  - [ ] Thread arena through lowering phases
  - [ ] Ensure arena lives long enough for entire compilation

---

## Phase 4: Export and External Crate Updates

### 4.1 Update `hwc-export/src/exporter.rs`

- [ ] **Remove lifetime from `CompiledOutput`**
  - [ ] Change `pub struct CompiledOutput<'ast>` → `pub struct CompiledOutput`
  - [ ] Update `symbol_table: SymbolTable<'ast>` → `symbol_table: SymbolTable`
  - [ ] Add `arena: Arc<AstArena>` field if needed for deferred access

- [ ] **Update exporter functions**
  - [ ] Remove `<'ast>` from function signatures
  - [ ] Accept arena as parameter where AST access is needed
  - [ ] Update GDSII/SVG/STL export logic to lookup via arena

---

### 4.2 Update `hwc-cli/src/commands/build_cmd/compilation.rs`

**Reference:** This is the top-level compilation orchestrator

- [ ] **Remove lifetime from `CompilationResult`**
  - [ ] Change `pub struct CompilationResult<'ast>` → `pub struct CompilationResult`
  - [ ] Remove `arena: Bump` field
  - [ ] Add `arena: AstArena` field

- [ ] **Update compilation flow**
  - [ ] Change parser call to return `(Program, AstArena)` tuple
  - [ ] Store arena in CompilationResult
  - [ ] Pass arena reference to compiler stages
  - [ ] Ensure arena is dropped last (keep as first field in struct)

---

### 4.3 Update `hwc-engine` crate (if AST dependencies exist)

- [ ] **Search for AST type usage in engine**
  - [ ] `grep -r "ComponentPlacement\|PourPlacement" hwc-engine/src/`
  - [ ] If engine holds AST references, update to use IDs + arena
  - [ ] Update routing algorithms to work with arena-based access

- [ ] **Update geometry router if needed**
  - [ ] Check `hwc-engine/src/geometry_router/**/*.rs`
  - [ ] If routes/contacts are accessed, use arena lookup pattern
  - [ ] Ensure thread safety (arena is Send + Sync)

---

## Phase 5: Testing and Validation

### 5.1 Update Parser Tests

- [ ] **Update `hwc-parser/tests/*.rs`**
  - [ ] Remove `<'ast>` from test helper functions
  - [ ] Update assertions to use arena lookups
  - [ ] Pattern:
    ```rust
    let (program, arena) = parse(source)?;
    let space = &program.definitions[0];
    if let Definition::Space(space_def) = space {
        let comp_id = space_def.statements[0];
        if let SpaceTopLevelStatement::Component(id) = comp_id {
            let component = &arena.components[id];
            assert_eq!(component.name, "R1");
        }
    }
    ```

- [ ] **Update `hwc-parser/examples/error_demo.rs`**
  - [ ] Remove lifetimes from example code
  - [ ] Update to use new parse API returning arena

---

### 5.2 Update Integration Tests

- [ ] **Update `hwc/tests/**/*.rs` (if they exist)**
  - [ ] Update test compilation flows
  - [ ] Ensure arena is properly threaded through tests
  - [ ] Verify no lifetime errors in test code

---

### 5.3 Verify Benchmarks

- [ ] **Check `hwc-parser/benches/*.rs`**
  - [ ] Update benchmarks to use new API
  - [ ] Verify allocation_strategies.rs benchmark still works
  - [ ] Run benchmarks to confirm performance improvements

---

## Phase 6: Documentation and Cleanup

### 6.1 Update Code Comments

- [ ] **Search for outdated comments mentioning lifetimes**
  - [ ] `grep -r "'ast" --include="*.rs"` in hwc workspace
  - [ ] Update comments that reference arena lifetimes
  - [ ] Add comments explaining arena-based access pattern

- [ ] **Update module-level documentation**
  - [ ] `hwc-parser/src/ast/mod.rs` - update module docs
  - [ ] `hwc-parser/src/parser/mod.rs` - update parser docs
  - [ ] Reference AST-ARENA-REFACTOR.md in comments

---

### 6.2 Update Cargo.toml Dependencies

- [ ] **Remove bumpalo dependency from `hwc-parser/Cargo.toml`**
  - [ ] Delete `bumpalo = "..."` line
  - [ ] Verify no other crates depend on bumpalo for AST

- [ ] **Verify no new dependencies needed**
  - [ ] Arena implementation is pure stdlib (no new deps)

---

### 6.3 Run Comprehensive Tests

- [ ] **Build entire workspace**
  - [ ] `cargo build --workspace`
  - [ ] Fix any compilation errors systematically
  - [ ] Start with parser crate, then compiler, then others

- [ ] **Run all tests**
  - [ ] `cargo test --workspace`
  - [ ] Fix test failures one by one
  - [ ] Verify all existing tests pass

- [ ] **Run Clippy**
  - [ ] `cargo clippy --workspace -- -D warnings`
  - [ ] Fix any new warnings (especially large_enum_variant should be gone)

- [ ] **Run formatter**
  - [ ] `cargo fmt --all`

---

## Phase 7: Advanced Patterns and Optimizations

### 7.1 Implement Arena Pre-allocation (Optional Optimization)

- [ ] **Add capacity estimation to parser**
  - [ ] Count tokens before parsing to estimate AST size
  - [ ] Call `AstArena::with_capacity(estimated_size)` in Parser::new()
  - [ ] Reference: AST-ARENA-REFACTOR.md performance section

---

### 7.2 Add Arena Statistics (Optional for Debugging)

- [ ] **Enhance AstArena with introspection**
  - [ ] Already has `memory_usage()` method
  - [ ] Add `print_statistics()` method for debugging
  - [ ] Track allocation counts per type

---

### 7.3 Consider Future Salsa Integration

- [ ] **Plan for incremental compilation**
  - [ ] Arena types are already `'static` compatible
  - [ ] Document how to use with Salsa queries
  - [ ] Reference: AST-ARENA-REFACTOR.md Salsa section

---

## Common Patterns Reference

### Pattern 1: Allocation in Parser
```rust
// OLD:
let component = ComponentPlacement { ... };
let boxed = Box::new(component);
statements.push(SpaceTopLevelStatement::Component(boxed));

// NEW:
let component = ComponentPlacement { ... };
let comp_id = self.arena.alloc_component(component);
statements.push(SpaceTopLevelStatement::Component(comp_id));
```

### Pattern 2: Access in Compiler
```rust
// OLD:
match stmt {
    SpaceTopLevelStatement::Component(boxed) => {
        process_component(&**boxed)
    }
}

// NEW:
match stmt {
    SpaceTopLevelStatement::Component(comp_id) => {
        let component = &arena.components[comp_id];
        process_component(component)
    }
}
```

### Pattern 3: Iteration Over Arena Types
```rust
// OLD:
for stmt in &space.statements {
    if let SpaceTopLevelStatement::Component(boxed) = stmt {
        process(&**boxed);
    }
}

// NEW:
for stmt in &space.statements {
    if let SpaceTopLevelStatement::Component(id) = stmt {
        let component = &arena.components[id];
        process(component);
    }
}
```

### Pattern 4: Parser Method Signatures
```rust
// OLD:
impl<'ast> Parser<'ast> {
    fn parse_contact(&mut self) -> Result<&'ast ContactPlacement, ParseError> {
        let contact = ContactPlacement { ... };
        Ok(self.arena.alloc(contact))
    }
}

// NEW:
impl Parser {
    fn parse_contact(&mut self) -> Result<ContactId, ParseError> {
        let contact = ContactPlacement { ... };
        let id = self.arena.alloc_contact(contact);
        Ok(id)
    }
}
```

### Pattern 5: Helper Methods Taking Arena
```rust
// OLD:
impl<'ast> SpaceDefinition<'ast> {
    pub fn components(&self) -> Vec<ComponentPlacement> {
        self.statements.iter()
            .filter_map(|s| match s {
                SpaceTopLevelStatement::Component(boxed) => Some((**boxed).clone()),
                _ => None,
            })
            .collect()
    }
}

// NEW:
impl SpaceDefinition {
    pub fn component_ids(&self) -> impl Iterator<Item = ComponentId> + '_ {
        self.statements.iter()
            .filter_map(|s| match s {
                SpaceTopLevelStatement::Component(id) => Some(*id),
                _ => None,
            })
    }
    
    // Or with arena access:
    pub fn components<'a>(&self, arena: &'a AstArena) -> impl Iterator<Item = &'a ComponentPlacement> + 'a {
        self.component_ids()
            .map(move |id| &arena.components[id])
    }
}
```

---

## Troubleshooting Guide

### Error: "cannot infer lifetime for `&ComponentPlacement`"
**Solution:** Add explicit lifetime to function signature or use arena lookup pattern

### Error: "mismatched types: expected `ComponentId`, found `RouteId`"
**Success!** This is exactly the type safety we want. Fix by using correct ID type.

### Error: "cannot borrow `arena` as mutable more than once"
**Solution:** Batch allocations or use separate scopes:
```rust
let id1 = arena.alloc_component(comp1);
let id2 = arena.alloc_component(comp2);
// NOT: let id1 = arena.alloc_component(comp_with_reference_to_arena);
```

### Error: "arena does not live long enough"
**Solution:** Ensure arena outlives all ID usage. Move arena ownership higher in call stack.

### Pattern: Passing arena to functions
```rust
// Prefer immutable reference for lookups:
fn process_space(space: &SpaceDefinition, arena: &AstArena) { ... }

// Use mutable reference only for allocation:
fn parse_space(&mut self) -> Result<SpaceDefinition, ParseError> {
    let id = self.arena.alloc_component(...);  // self.arena is &mut
}
```

---

## Migration Checklist Summary

**Phase 1: Core AST** (Estimated: 2-4 hours)
- [ ] 1.1 space_def.rs - SpaceDefinition, enums
- [ ] 1.2 mod.rs - Program, Definition
- [ ] 1.3 module.rs - ModuleDefinition
- [ ] 1.4 layout.rs - ModuleLayoutBlock
- [ ] 1.5 Verify other AST files

**Phase 2: Parser** (Estimated: 4-8 hours)
- [ ] 2.1 parser/mod.rs - Parser struct
- [ ] 2.2 All parser definition files (~30 files)
- [ ] 2.3 lib.rs entry point

**Phase 3: Compiler** (Estimated: 2-4 hours)
- [ ] 3.1 symbol_table/definition.rs
- [ ] 3.2 symbol_table/layer.rs
- [ ] 3.3 IR and lowering

**Phase 4: External Crates** (Estimated: 2-3 hours)
- [ ] 4.1 hwc-export
- [ ] 4.2 hwc-cli
- [ ] 4.3 hwc-engine (if needed)

**Phase 5: Testing** (Estimated: 2-4 hours)
- [ ] 5.1 Parser tests
- [ ] 5.2 Integration tests
- [ ] 5.3 Benchmarks

**Phase 6: Cleanup** (Estimated: 1-2 hours)
- [ ] 6.1 Update comments
- [ ] 6.2 Update Cargo.toml
- [ ] 6.3 Run tests and clippy

**Phase 7: Optimizations** (Optional, Estimated: 1-2 hours)
- [ ] 7.1 Pre-allocation
- [ ] 7.2 Statistics
- [ ] 7.3 Salsa planning

**Total Estimated Time: 14-27 hours** (depends on codebase familiarity)

---

## Success Criteria

- [ ] **Zero lifetime parameters** in public AST types
- [ ] **Zero `Box<T>`** in AST enum variants for arena-allocated types
- [ ] **Zero bumpalo dependency** in Cargo.toml
- [ ] **All tests pass** (`cargo test --workspace`)
- [ ] **No clippy warnings** (`cargo clippy --workspace`)
- [ ] **Benchmark shows improvement** (allocation 1.26× faster per AST-ARENA-REFACTOR.md)
- [ ] **Memory usage reduced** (11% reduction per benchmarks)
- [ ] **Thread safety enabled** (can use rayon for parallel validation)

---

## References

1. **AST-ARENA-REFACTOR.md** - Architectural rationale and benchmarks
2. **hwc-parser/src/ast/arena.rs** - Arena implementation (already complete)
3. **Rust RFC: rustc arena allocation** - https://rust-lang.github.io/rfcs/2256-compiler-arena.html
4. **"Generativity" blog post by withoutboats** - Explains lifetime vs index tradeoffs

---

*This checklist should be updated as sub-agent progresses through implementation. Mark items complete with [x] as they are verified working.*
