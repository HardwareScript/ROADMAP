# SYSTEM FRAGMENTATION STRATEGY

**Documentation Base**: [v0.1.3 Documentation](../Docs/v0.1.3/) - The authoritative reference for all implementation  
**Strategy**: Focus on one system at a time, complete it, then move to the next  
**Current Focus**: System 1 (Core Language & Parser)

---

## CRITICAL RULE: NO HARDCODED DATA

**ALL systems MUST use data from these sources (in priority order):**

1. **Primary Data Sources** (ALWAYS check these first):
   - `hwc/data/standard-materials.yaml` - Material properties (conductors, insulators, semiconductors)
   - `hwc/data/profiles/*.hwp` - Fabrication constraints (standard-pcb, standard-asic, high-voltage, prototype)
   - `hwc/data/components/*.hwx` - Component definitions
   - `hwc/data/footprints/*.hwf` - Footprint definitions

2. **If data doesn't exist**:
   - Check if it should be added to standard data files
   - Create custom data file in appropriate format (.hwmat, .hwp, .hwx, .hwf)
   - Document the source and reasoning

3. **NEVER**:
   - Hardcode material properties (resistivity, thermal conductivity, etc.)
   - Hardcode constraint values (trace widths, clearances, etc.)
   - Hardcode component specifications
   - Use magic numbers without referencing source data

**Why this matters**: Hardcoded values create maintenance nightmares, make testing unrealistic, and break when standards change. Always pull from the data layer.

---

## How This Works

1. **SYSTEM-FRAGMENTATION.md** (this file) - Master plan showing all 7 systems
2. **SYSTEM-X-IMPLEMENTATION-PLAN.md** - Detailed implementation for each system
3. Each system references specific v0.1.3 documentation sections
4. Each system MUST reference data sources (materials, profiles, components)
5. Complete one system before starting the next

---

## System Breakdown

Based on the complete v0.1.3 documentation, here's how the Hardware Script implementation is fragmented into distinct systems:

## SYSTEM 1: CORE LANGUAGE & PARSER

**What it is**: The foundation - lexer, parser, AST, and basic IR  
**Documentation**: [LANGUAGE-SPEC.md](../Docs/v0.1.3/LANGUAGE-SPEC.md), [COMPILER-INTERNALS.md](../Docs/v0.1.3/COMPILER-INTERNALS.md)  
**Implementation Plan**: [SYSTEM-1-IMPLEMENTATION-PLAN.md](./SYSTEM-1-IMPLEMENTATION-PLAN.md)

**Data Sources**:
- None (pure syntax/grammar - no external data dependencies)

**Scope**:
- Hand-written recursive descent parser (replacing Pest)
- Logos-based lexer
- AST construction for .hw syntax
- Basic IR (HardwareIR struct)
- Import resolution
- Loop unrolling
- Coordinate transformation (local → global)

**Why first**: Everything depends on this. Can't route, can't validate, can't export without parsing.

## SYSTEM 2: VOXEL ENGINE & SPATIAL OPERATIONS

**What it is**: The 3D tensor grid, Morton encoding, and spatial math  
**Documentation**: [COMPILER-INTERNALS.md](../Docs/v0.1.3/COMPILER-INTERNALS.md) - Voxel grid architecture  
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

**Why second**: Once you can parse, you need to place things in space.

## SYSTEM 3: ROUTING ENGINE

**What it is**: The 3-phase routing pipeline (constraint → geometry → DRC)  
**Documentation**: [ROUTING-AND-PHYSICS.md](../Docs/v0.1.3/ROUTING-AND-PHYSICS.md)  
**Implementation Plan**: SYSTEM-3-IMPLEMENTATION-PLAN.md (to be created)

**Data Sources**:
- `hwc/data/standard-materials.yaml` - Material properties for constraint generation
- `hwc/data/profiles/*.hwp` - Fabrication constraints (trace widths, clearances, via sizes)

**Scope**:
- Phase 1: Constraint Manager (clearances, widths, materials)
- Phase 2: Geometry Router (Manhattan routing, A* pathfinding, via insertion)
- Phase 3: Design Rule Check (validation against .hwp profiles)
- Manual waypoint interpolation (Bresenham)
- Auto-routing algorithms

**Why third**: After placement, you need to connect things.

## SYSTEM 4: MATERIALS & PHYSICS VALIDATION

**What it is**: Material database and physics solvers  
**Documentation**: [ROUTING-AND-PHYSICS.md](../Docs/v0.1.3/ROUTING-AND-PHYSICS.md) - Materials and physics  
**Implementation Plan**: SYSTEM-4-IMPLEMENTATION-PLAN.md (to be created)

**Data Sources**:
- `hwc/data/standard-materials.yaml` - ALL material properties (resistivity, thermal conductivity, dielectric strength, etc.)
- `hwc/data/profiles/*.hwp` - Constraint profiles for validation thresholds
- NEVER hardcode material properties - always load from database

**Scope**:
- .hwmat material database (YAML parsing)
- Electrical analysis (resistance, voltage drop, ampacity)
- Thermal analysis (temperature rise, heat dissipation)
- Electromagnetic analysis (impedance, signal integrity)
- Clearance validation (dielectric breakdown)

**Why fourth**: After routing, you need to validate physics.

## SYSTEM 5: EXPORT GENERATION & ASSETS

**What it is**: Custom emitters for manufacturing files and 3D visualization  
**Documentation**: [EXPORTS-AND-ASSETS.md](../Docs/v0.1.3/EXPORTS-AND-ASSETS.md)  
**Implementation Plan**: SYSTEM-5-IMPLEMENTATION-PLAN.md (to be created)

**Data Sources**:
- `hwc/data/standard-materials.yaml` - Material colors and properties for 3D rendering
- `hwc/data/profiles/*.hwp` - Manufacturing constraints for export validation
- Component footprints for BOM generation

**Scope**:
- GerberEmitter (PCB manufacturing)
- GdsiiEmitter (silicon manufacturing)
- ObjExporter / GlbExporter (3D models)
- BlenderExporter (scene graph + Python scripts)
- QasmExporter (quantum computing)
- BOM generation
- Drill file generation

**Why fifth**: After validation, you need to generate outputs.

## SYSTEM 6: ECOSYSTEM & TOOLING

**What it is**: Package manager, CLI, IDE integration, and file extensions  
**Documentation**: [ECOSYSTEM.md](../Docs/v0.1.3/ECOSYSTEM.md)  
**Implementation Plan**: SYSTEM-6-IMPLEMENTATION-PLAN.md (to be created)

**Data Sources**:
- Package registry metadata
- Component library definitions
- Material database for package validation

**Scope**:
- hpm package manager (install, publish, search)
- hws CLI (build, verify, test, generate)
- File extension handlers (.hwx, .hwp, .hwf, .hwmat, .hwsig, .hwtc, .hwt, .hwa)
- VS Code extension (syntax highlighting, LSP)
- Documentation system (hwsd)
- CI/CD integration

**Why sixth**: After core functionality works, you need developer experience.

## SYSTEM 7: ADVANCED FEATURES

**What it is**: Abstraction blocks, multi-target compilation, and paradigm support  
**Documentation**: [VISION.md](../Docs/v0.1.3/VISION.md), [LANGUAGE-SPEC.md](../Docs/v0.1.3/LANGUAGE-SPEC.md)  
**Implementation Plan**: SYSTEM-7-IMPLEMENTATION-PLAN.md (to be created)

**Data Sources**:
- Extended material database for advanced paradigms (photonics, quantum)
- Component library for standard abstractions
- Target-specific constraint profiles

**Scope**:
- Abstraction blocks (pins, behavior, layout, electrical, render)
- Multi-target compilation (PCB, FPGA, ASIC, SPICE, sim, viz)
- Component library (standard library)
- Testing framework (.hwt test benches)
- Advanced paradigm support (photonics, ternary, quantum)

**Why seventh**: After ecosystem is stable, you add advanced capabilities.

---

## Implementation Priority

**Priority 1**: SYSTEM 1 (Core Language & Parser) - **IN PROGRESS**  
**Priority 2**: SYSTEM 2 (Voxel Engine)  
**Priority 3**: SYSTEM 3 (Routing Engine)  
**Priority 4**: SYSTEM 4 (Materials & Physics)  
**Priority 5**: SYSTEM 5 (Export Generation)  
**Priority 6**: SYSTEM 6 (Ecosystem & Tooling)  
**Priority 7**: SYSTEM 7 (Advanced Features)

---

## Current Status

**Completed Systems**:
- ✅ System 1 (Core Language & Parser) - COMPLETE
- ✅ System 2 (Voxel Engine) - COMPLETE

**Active System**: None - Ready to start System 3 or System 5  
**Next Recommended**: System 5 (Export Generation) - Re-enable hwc-export crate  
**Alternative**: System 3 (Routing Engine) - 3-phase routing pipeline

**Summary**: [SYSTEM-1-AND-2-COMPLETION-SUMMARY.md](./SYSTEM-1-AND-2-COMPLETION-SUMMARY.md)