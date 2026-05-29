# SYSTEM FRAGMENTATION STRATEGY

**Documentation Base**: [v0.1.4 Documentation](../../Docs/v0.1.4/) - The authoritative reference for all implementation  
**Strategy**: Focus on one system at a time, complete it, then move to the next  
**Current Focus**: System 1 (Unified Parser & Two-Pass Compiler)

---

## CRITICAL ARCHITECTURAL CHANGE: v0.1.4

**v0.1.3 Architecture**: 10+ file extensions (`.hwmat`, `.hwp`, `.hwx`, `.hwf`, `.hwt`, `.hwa`, `.hwm`, `.hwsig`, `.hwtc`)  
**v0.1.4 Architecture**: 3-File System (`hw.toml`, `hw.lock`, `.hw`)

**Major Changes**:
1. **Unified `.hw` Extension**: Everything uses `define` blocks in `.hw` files
2. **Two-Pass Compilation**: Symbol Table registration → Space assembly
3. **Native SI Unit Parsing**: No more YAML dependencies (`serde_yaml` removed)
4. **Hyper-Lean Dependencies**: Down to 4 external crates (logos, miette, thiserror, rayon)

---

## CRITICAL RULE: NO HARDCODED DATA

**ALL systems MUST use data from these sources (in priority order):**

1. **Primary Data Sources** (ALWAYS check these first):
   - `materials.hw` - Material definitions using `define material` blocks
   - `profiles.hw` - Fabrication constraints using `define profile` blocks
   - `components.hw` - Component definitions using `define component` blocks
   - `constraints.hw` - Mechanical/signal constraints using `define mechanical` and `define signal_group` blocks

2. **If data doesn't exist**:
   - Add new `define` block to appropriate `.hw` file
   - Document the source and reasoning
   - Use native SI unit syntax (e.g., `254µm`, `4.7kΩ`)

3. **NEVER**:
   - Hardcode material properties (resistivity, thermal conductivity, etc.)
   - Hardcode constraint values (trace widths, clearances, etc.)
   - Hardcode component specifications
   - Use magic numbers without referencing source data
   - Use YAML files (native SI unit parsing eliminates YAML)

**Why this matters**: Hardcoded values create maintenance nightmares, make testing unrealistic, and break when standards change. Always pull from the data layer using `define` blocks.

---

## How This Works

1. **SYSTEM-FRAGMENTATION.md** (this file) - Master plan showing all 7 systems
2. **SYSTEM-X-IMPLEMENTATION-PLAN.md** - Detailed implementation for each system
3. Each system references specific v0.1.4 documentation sections
4. Each system MUST reference data sources (materials, profiles, components)
5. Complete one system before starting the next

---

## System Breakdown

Based on the complete v0.1.4 documentation, here's how the Hardware Script implementation is fragmented into distinct systems:

## SYSTEM 1: UNIFIED PARSER & TWO-PASS COMPILER

**What it is**: The foundation - lexer, unified parser for all `define` blocks, Symbol Table, and two-pass compilation  
**Documentation**: [LANGUAGE-SPEC.md](../../Docs/v0.1.4/LANGUAGE-SPEC.md), [COMPILER-INTERNALS.md](../../Docs/v0.1.4/COMPILER-INTERNALS.md)  
**Implementation Plan**: [SYSTEM-1-IMPLEMENTATION-PLAN.md](./SYSTEM-1-IMPLEMENTATION-PLAN.md)

**Data Sources**:
- None (pure syntax/grammar - no external data dependencies)

**Scope**:
- Logos-based lexer with native SI unit tokens
- Hand-written recursive descent parser for all `define` blocks
- Unified AST construction (material, profile, component, mechanical, interface, test, space)
- Symbol Table for two-pass compilation
- Pass 1: Register all definitions
- Pass 2: Resolve `define space` against Symbol Table
- Hardware IR construction
- Import resolution
- Loop unrolling
- Coordinate transformation (local → global)

**Key Changes from v0.1.3**:
- ✅ Unified parser handles all `define` blocks (not separate file parsers)
- ✅ Two-pass compilation with Symbol Table (not single-pass)
- ✅ Native SI unit parsing in lexer (no YAML, no `serde_yaml`)
- ✅ Single `.hw` file extension (not 10+ extensions)

**Why first**: Everything depends on this. Can't route, can't validate, can't export without parsing and compiling.

## SYSTEM 2: VOXEL ENGINE & SPATIAL OPERATIONS

**What it is**: The 3D tensor grid, Morton encoding, and spatial math  
**Documentation**: [COMPILER-INTERNALS.md](../../Docs/v0.1.4/COMPILER-INTERNALS.md) - Voxel grid architecture  
**Implementation Plan**: SYSTEM-2-IMPLEMENTATION-PLAN.md (to be created)

**Data Sources**:
- None (pure spatial math - no external data dependencies)

**Scope**:
- VoxelGrid with Morton Z-curve encoding
- NetlistArena (ECS-style component/pin/net storage)
- Fixed-point geometry (Point3D, BoundingBox, TraceSegment)
- Collision detection
- Component placement
- Substrate spanning

**Key Changes from v0.1.3**:
- ✅ No changes to core voxel engine (already optimal)
- ✅ Integration with Symbol Table for component definitions

**Why second**: Once you can parse, you need to place things in space.

## SYSTEM 3: ROUTING ENGINE

**What it is**: The 3-phase routing pipeline (constraint → geometry → DRC)  
**Documentation**: [ROUTING-AND-PHYSICS.md](../../Docs/v0.1.4/ROUTING-AND-PHYSICS.md)  
**Implementation Plan**: SYSTEM-3-IMPLEMENTATION-PLAN.md (to be created)

**Data Sources**:
- `materials.hw` - Material properties for constraint generation (via Symbol Table)
- `profiles.hw` - Fabrication constraints (trace widths, clearances, via sizes) (via Symbol Table)

**Scope**:
- Phase 1: Constraint Manager (clearances, widths, materials)
- Phase 2: Geometry Router (Manhattan routing, A* pathfinding, via insertion)
- Phase 3: Design Rule Check (validation against profiles)
- Manual waypoint interpolation (Bresenham)
- Auto-routing algorithms

**Key Changes from v0.1.3**:
- ✅ Load constraints from `define profile` blocks (not `.hwp` files)
- ✅ Load materials from `define material` blocks (not `.hwmat` YAML)
- ✅ Symbol Table integration for constraint resolution

**Why third**: After placement, you need to connect things.

## SYSTEM 4: MATERIALS & PHYSICS VALIDATION

**What it is**: Material database and physics solvers  
**Documentation**: [ROUTING-AND-PHYSICS.md](../../Docs/v0.1.4/ROUTING-AND-PHYSICS.md) - Materials and physics  
**Implementation Plan**: SYSTEM-4-IMPLEMENTATION-PLAN.md (to be created)

**Data Sources**:
- `materials.hw` - ALL material properties (resistivity, thermal conductivity, dielectric strength, etc.) via `define material` blocks
- `profiles.hw` - Constraint profiles for validation thresholds via `define profile` blocks
- NEVER hardcode material properties - always load from Symbol Table

**Scope**:
- Material database (native SI unit parsing, no YAML)
- Electrical analysis (resistance, voltage drop, ampacity)
- Thermal analysis (temperature rise, heat dissipation)
- Electromagnetic analysis (impedance, signal integrity)
- Clearance validation (dielectric breakdown)

**Key Changes from v0.1.3**:
- ✅ Native SI unit parsing (no YAML, no `serde_yaml`)
- ✅ Load from `define material` blocks (not `.hwmat` files)
- ✅ Symbol Table integration for material lookup
- ✅ Parser handles units directly (e.g., `254µm`, `4.7kΩ`)

**Why fourth**: After routing, you need to validate physics.

## SYSTEM 5: EXPORT GENERATION & ASSETS

**What it is**: Custom emitters for manufacturing files and 3D visualization  
**Documentation**: [EXPORTS-AND-ASSETS.md](../../Docs/v0.1.4/EXPORTS-AND-ASSETS.md)  
**Implementation Plan**: SYSTEM-5-IMPLEMENTATION-PLAN.md (to be created)

**Data Sources**:
- `materials.hw` - Material colors and properties for 3D rendering (via Symbol Table)
- `profiles.hw` - Manufacturing constraints for export validation (via Symbol Table)
- Component render blocks for 3D asset paths

**Scope**:
- GerberEmitter (PCB manufacturing)
- GdsiiEmitter (silicon manufacturing)
- ObjExporter / GlbExporter (3D models)
- BlenderExporter (scene graph + Python scripts)
- QasmExporter (quantum computing)
- BOM generation
- Drill file generation

**Key Changes from v0.1.3**:
- ✅ Load render blocks from `define component` (not `.hwx` files)
- ✅ Load materials from Symbol Table (not YAML)
- ✅ No separate file parsers needed

**Why fifth**: After validation, you need to generate outputs.

## SYSTEM 6: ECOSYSTEM & TOOLING

**What it is**: Package manager, CLI, IDE integration, and unified file system  
**Documentation**: [ECOSYSTEM.md](../../Docs/v0.1.4/ECOSYSTEM.md)  
**Implementation Plan**: SYSTEM-6-IMPLEMENTATION-PLAN.md (to be created)

**Data Sources**:
- Package registry metadata
- Component library definitions (via `define component` blocks)
- Material database for package validation (via `define material` blocks)

**Scope**:
- hpm package manager (install, publish, search)
- hwc CLI (build, verify, test, generate)
- Unified `.hw` file handling (no separate parsers for 10+ extensions)
- VS Code extension (syntax highlighting, LSP)
- Documentation system (hwsd)
- CI/CD integration

**Key Changes from v0.1.3**:
- ✅ Unified `.hw` extension (not 10+ file types)
- ✅ Single parser for all `define` blocks
- ✅ Simplified IDE integration (one grammar, not 10)
- ✅ LSP handles all `define` blocks uniformly

**Why sixth**: After core functionality works, you need developer experience.

## SYSTEM 7: ADVANCED FEATURES

**What it is**: Abstraction blocks, multi-target compilation, and paradigm support  
**Documentation**: [LANGUAGE-SPEC.md](../../Docs/v0.1.4/LANGUAGE-SPEC.md), [COMPILER-INTERNALS.md](../../Docs/v0.1.4/COMPILER-INTERNALS.md)  
**Implementation Plan**: SYSTEM-7-IMPLEMENTATION-PLAN.md (to be created)

**Data Sources**:
- Extended material database for advanced paradigms (photonics, quantum)
- Component library for standard abstractions
- Target-specific constraint profiles

**Scope**:
- Abstraction blocks (pins, behavior, layout, electrical, render)
- Multi-target compilation (PCB, FPGA, ASIC, SPICE, sim, viz)
- Component library (standard library)
- Testing framework (via `define test` blocks)
- Advanced paradigm support (photonics, ternary, quantum)

**Key Changes from v0.1.3**:
- ✅ Test benches use `define test` blocks (not `.hwt` files)
- ✅ Interfaces use `define interface` blocks (not `.hwf` files)
- ✅ All definitions in unified `.hw` ecosystem

**Why seventh**: After ecosystem is stable, you add advanced capabilities.

---

## Implementation Priority

**Priority 1**: ✅ SYSTEM 1 (Unified Parser & Two-Pass Compiler) - **CORE COMPLETE**  
**Priority 2**: ⏭️ SYSTEM 2 (Voxel Engine) - Symbol Table Integration  
**Priority 3**: ⏭️ SYSTEM 3 (Routing Engine) - Symbol Table Integration  
**Priority 4**: ⏭️ SYSTEM 4 (Materials & Physics) - Symbol Table Integration + YAML Elimination  
**Priority 5**: ⏭️ SYSTEM 5 (Export Generation) - Symbol Table Integration  
**Priority 6**: ⏭️ SYSTEM 6 (Ecosystem & Tooling) - Build on foundation  
**Priority 7**: ⏭️ SYSTEM 7 (Advanced Features) - Build on foundation

---

## Current Status

**Completed Systems**:
- ✅ System 1 (v0.1.4) - **CORE COMPLETE (78%)** - Parser, Symbol Table, Two-Pass Compiler working
  - ✅ Phases 1-7 complete (core functionality)
  - ⏭️ Phases 8-9 optional (migration guide, rustdoc)
  - **Status**: Production-ready for core features

**Systems Needing Integration**:
- ⚠️ System 2 (Voxel Engine) - NEEDS Symbol Table integration
- ⚠️ System 3 (Routing Engine) - NEEDS Symbol Table integration  
- ⚠️ System 4 (Physics Validation) - NEEDS Symbol Table integration + YAML elimination
- ⚠️ System 5 (Export Generation) - NEEDS Symbol Table integration
- ❌ System 6 (Ecosystem & Tooling) - NOT STARTED
- ❌ System 7 (Advanced Features) - NOT STARTED

**Active System**: System 2 (Voxel Engine) - Symbol Table Integration  
**Next Recommended**: Complete Systems 2-5 integration (Symbol Table migration)

**Migration Strategy**:
1. ✅ System 1: Unified parser, Symbol Table, Two-pass compilation (DONE)
2. ⏭️ System 2: Update placement/routing to use Symbol Table
3. ⏭️ System 3: Update constraint manager to use Symbol Table
4. ⏭️ System 4: Remove YAML, use Symbol Table for materials
5. ⏭️ System 5: Update exporters to use Symbol Table
6. ⏭️ Systems 6-7: Build on completed foundation

---

## Key Architectural Differences: v0.1.3 → v0.1.4

| Aspect | v0.1.3 | v0.1.4 |
|--------|--------|--------|
| **File Extensions** | 10+ types (`.hwmat`, `.hwp`, `.hwx`, etc.) | 3 types (`hw.toml`, `hw.lock`, `.hw`) |
| **Compilation** | Single-pass | Two-pass with Symbol Table |
| **Material Data** | YAML files (`.hwmat`) | `define material` blocks |
| **Profiles** | YAML files (`.hwp`) | `define profile` blocks |
| **Components** | Separate `.hwx` files | `define component` blocks |
| **Unit Parsing** | YAML values (e.g., `min_width_nm: 254000`) | Native SI (e.g., `min_width: 254µm`) |
| **Dependencies** | 5 crates (includes `serde_yaml`) | 4 crates (no YAML) |
| **Parser** | Separate parsers per file type | Unified parser for all `define` blocks |
| **Symbol Resolution** | Immediate during parsing | Deferred to Pass 2 |

---

## Migration Checklist

### System 1 Migration (Unified Parser)
- [ ] Remove Pest grammar for separate file types
- [ ] Add native SI unit tokens to Logos lexer
- [ ] Implement unified parser for all `define` blocks
- [ ] Implement Symbol Table data structure
- [ ] Implement Pass 1: Symbol registration
- [ ] Implement Pass 2: Space assembly
- [ ] Remove `serde_yaml` dependency
- [ ] Update all tests to v0.1.4 syntax

### System 2 Migration (Voxel Engine)
- [ ] Update component placement to use Symbol Table
- [ ] Update material lookup to use Symbol Table
- [ ] No changes to core voxel math (already optimal)

### System 3 Migration (Routing)
- [ ] Update constraint loading to use Symbol Table
- [ ] Update material lookup to use Symbol Table
- [ ] No changes to core routing algorithms

### System 4 Migration (Physics)
- [ ] Update material database to use Symbol Table
- [ ] Remove YAML parsing code
- [ ] Use native SI unit values from `define material` blocks

### System 5 Migration (Export)
- [ ] Update render block access to use Symbol Table
- [ ] Update material color lookup to use Symbol Table
- [ ] No changes to core emitters

### System 6 Migration (Ecosystem)
- [ ] Update CLI to handle unified `.hw` files
- [ ] Update VS Code extension for unified grammar
- [ ] Update LSP for all `define` blocks
- [ ] Simplify file type handling (3 types instead of 10+)

### System 7 Migration (Advanced)
- [ ] Update test bench parsing to use `define test` blocks
- [ ] Update interface parsing to use `define interface` blocks
- [ ] No changes to abstraction block logic

---

## Documentation References

**v0.1.4 Core Documentation**:
- [LANGUAGE-SPEC.md](../../Docs/v0.1.4/LANGUAGE-SPEC.md) - Unified `.hw` syntax with `define` blocks
- [COMPILER-INTERNALS.md](../../Docs/v0.1.4/COMPILER-INTERNALS.md) - Two-pass compilation architecture
- [ECOSYSTEM.md](../../Docs/v0.1.4/ECOSYSTEM.md) - 3-File system and project structure
- [ROUTING-AND-PHYSICS.md](../../Docs/v0.1.4/ROUTING-AND-PHYSICS.md) - Routing algorithms (unchanged)
- [EXPORTS-AND-ASSETS.md](../../Docs/v0.1.4/EXPORTS-AND-ASSETS.md) - Export generation (unchanged)

**v0.1.3 Reference** (for migration):
- [v0.1.3 Documentation](../../Docs/v0.1.3/) - Previous architecture for comparison

---

## Success Criteria

**System 1 Complete When**: ✅ **ACHIEVED**
- ✅ Unified parser handles all `define` blocks
- ✅ Symbol Table correctly registers all definitions
- ✅ Two-pass compilation produces correct Hardware IR
- ✅ Native SI unit parsing works (no YAML)
- ✅ All v0.1.3 tests migrated and passing
- ✅ `serde_yaml` dependency removed
- ✅ Compile time < 10 seconds
- ✅ Zero unsafe code

**All Systems Complete When**:
- ✅ System 1: Full v0.1.4 parser architecture implemented
- ⏭️ System 2: Voxel engine uses Symbol Table
- ⏭️ System 3: Routing engine uses Symbol Table
- ⏭️ System 4: Physics validation uses Symbol Table (no YAML)
- ⏭️ System 5: Export generation uses Symbol Table
- ⏭️ System 6: CLI handles 3-File system correctly
- ⏭️ System 7: Advanced features implemented
- ⏭️ All tests passing with unified `.hw` syntax
- ⏭️ Documentation matches implementation
- ⏭️ IDE integration works with unified grammar
- ⏭️ Package manager handles `.hw` files correctly

---

## Notes

**Breaking Changes**: v0.1.4 is a breaking change from v0.1.3. All existing `.hwmat`, `.hwp`, `.hwx` files must be migrated to unified `.hw` syntax with `define` blocks.

**Migration Tools**: Consider building automated migration tools to convert v0.1.3 projects to v0.1.4 syntax.

**Backward Compatibility**: Not maintained. v0.1.4 is a clean break for architectural simplification.

**Timeline**: System 1 is the critical path. All other systems depend on it. Estimate 2-4 weeks for System 1 migration.
