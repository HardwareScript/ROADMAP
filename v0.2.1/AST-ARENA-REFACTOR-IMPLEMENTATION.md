# AST Arena Refactor - Implementation Status

**Version:** v0.2.1  
**Last Updated:** 2026-08-08  
**Status:** 65% Complete - Parser allocation blocking

---

## Critical Issue Discovered

**Problem:** Initial migration created HYBRID SYSTEM where only `Component` used arena IDs while `Material`, `Module`, `Profile`, `Space`, `Bridge` remained as direct values.

**Solution:** Migrate to 100% UNIFORM system - ALL definitions as arena IDs.

---

## Completed ✓

### Arena Infrastructure
- [x] Added ID types for all definitions (MaterialDefId, ModuleDefId, etc.)
- [x] Added IndexVec storage in AstArena for all types
- [x] Added allocation methods (alloc_material_def, alloc_module_def, etc.)
- [x] Updated with_capacity() and clear() methods
- [x] Exported AstArena from ast module

### AST Type Definitions  
- [x] Updated Definition enum to use IDs for all variants
- [x] Removed Box<T> from Profile and Space variants
- [x] Removed all lifetime parameters from space_def.rs, module.rs, layout.rs
- [x] Changed SpaceTopLevelStatement to use IDs (ComponentId, PourId, etc.)
- [x] Changed ModuleStatement to use ModuleComponentId

### Compiler/CLI Integration
- [x] Updated compilation.rs to lookup all definitions from arena
- [x] Updated check.rs to lookup all definitions from arena  
- [x] Updated DeviceExtractor to accept arena parameter
- [x] Fixed ExtractedDevices::from_module() arena lookup

---

## Remaining Work (Blocks Compilation)

### 1. Fix Parser Definition Allocation ⚠️ CRITICAL
**File:** `hwc-parser/src/parser/definitions/mod.rs`  
**Errors:** 15 type mismatches

Parser returns struct values but Definition expects IDs. Need to allocate to arena:

- [ ] parse_bridge: `let id = self.arena.alloc_bridge_def(bridge); Definition::Bridge(id)`
- [ ] parse_material: `let id = self.arena.alloc_material_def(mat); Definition::Material(id)`
- [ ] parse_profile: `let id = self.arena.alloc_profile_def(profile); Definition::Profile(id)`
- [ ] parse_module: `let id = self.arena.alloc_module_def(module); Definition::Module(id)`
- [ ] parse_mechanical: `let id = self.arena.alloc_mechanical_def(mech); Definition::Mechanical(id)`
- [ ] parse_interface: `let id = self.arena.alloc_interface_def(iface); Definition::Interface(id)`
- [ ] parse_test: `let id = self.arena.alloc_test_def(test); Definition::Test(id)`
- [ ] parse_space: `let id = self.arena.alloc_space_def(space); Definition::Space(id)`
- [ ] parse_unit: `let id = self.arena.alloc_unit_def(unit); Definition::Unit(id)`
- [ ] parse_device: `let id = self.arena.alloc_device_def(device); Definition::Device(id)`
- [ ] parse_const: `let id = self.arena.alloc_const_def(const_def); Definition::Const(id)`

### 2. Fix Serde Deserialization ⚠️ CRITICAL
**File:** `hwc-parser/src/ast/arena.rs` (line ~624)  
**Error:** Missing 11 fields in AstArena deserializer

- [ ] Add Field enum variants: MaterialDefs, ModuleDefs, ProfileDefs, SpaceDefs, BridgeDefs, MechanicalDefs, InterfaceDefs, TestDefs, DeviceDefs, UnitDefs, ConstDefs
- [ ] Add match arms in visit_map: `Field::MaterialDefs => material_defs = Some(map.next_value()?)`
- [ ] Add fields to AstArena construction: `material_defs: material_defs.unwrap_or_default()`
- [ ] Update FIELDS constant with new field names
- [ ] Update Serialize impl if needed

### 3. Handle Special Definition Types
- [ ] **Pattern/Strategy**: Keep as small direct values OR create dedicated IDs?
- [ ] **MaterialAlias**: Should point to MaterialDefId or store separate?
- [ ] **SignalGroup**: Keep reusing InterfaceDefId or create dedicated?
- [ ] Verify small types (Enum, Struct, Logic, Shape, SpiceModel, Subcircuit) stay as direct values

### 4. Cleanup
- [ ] Remove unused import: `use arena::ComponentDefId;` from ast/mod.rs line 61
- [ ] Remove any remaining Box<T> from Definition variants
- [ ] Verify no &'ast lifetimes remain in public APIs

---

## Verification Checklist

- [ ] `cargo build --workspace` succeeds
- [ ] `cargo clippy --workspace -- -D warnings` passes
- [ ] `cargo test --workspace` passes
- [ ] `cargo fmt --all` applied
- [ ] No `Box<T>` in Definition enum
- [ ] All Definition variants ~8 bytes (4-byte ID + 4-byte discriminant)
- [ ] Memory usage reduced by 11% (per benchmarks)

---

## Build Error Summary

**Current state:** 16 compilation errors
- 1 Serde error: Missing fields in AstArena deserializer
- 15 Parser errors: Type mismatches (expected ID, found struct)

**Fix priority:**
1. Serde deserializer (enables compilation)
2. Parser allocation (fixes all 15 type errors)
3. Special types + cleanup
