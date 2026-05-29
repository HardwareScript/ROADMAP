# Hardware Script v0.1.7 Roadmap: The SoC Engine

This roadmap defines the final steps to transition from discrete component placement to full System on a Chip (SoC) engineering.

**Current Status**: Phase 1 & 2 "Xcellence" Verification Complete. **v0.1.7 Progress**: Anchor Lag + Router cured; Physical Pile core complete (intrinsic layout parse + compiler support). See Tier 1.

---

## 🏆 Achievements: The "Xcellence" Audit
We have successfully validated the logical core of the SoC Engine. The compiler now handles massive scales that would crash previous versions.

### **Step 1: Spatial Topological Sorting**
- **SpatialDependencyGraph**: Functional DFS-based dependency tracking implemented.
- **Topological Sort**: Successfully determines placement order (A -> B -> C) regardless of textual order.
- **Cycle Detection**: Correctly identifies and throws `CircularReference` error on loops.
- **Verification**: [test_spatial_sorting.hw](file:///c:/Users/olowo/Downloads/Code/Hardware-Script/hwc/tests/soc_engine/test_spatial_sorting.hw) passed logical verification.

### **Step 2: The 64-Bit Bus Unroller**
- **Parametric Unroller**: Successfully instantiates 64+ components in sub-second time (0.03s).
- **Dynamic Net Binding**: Correct carry-chain wiring (`Carry[i]` to `Carry[i+1]`) using loop variables.
- **Netlist Extraction**: Successfully extracted a 320-pin netlist for the 64-bit ALU.
- **Verification**: [test_64bit_alu.hw](file:///c:/Users/olowo/Downloads/Code/Hardware-Script/hwc/tests/soc_engine/test_64bit_alu.hw) built successfully.

---

## 🔍 The Authoritative Internal Audit
Following the Xcellence stress tests, we have separated the **symptoms** from the **diseases**. Below is the ranking of all known limitations, from fundamental architectural flaws to intended design features.

### **TIER 1: The "Native Issues" (Fundamental Architectural Flaws)**
These require a fundamental shift in how the compiler reasons about data. They cannot be fixed with a simple patch.

#### **1. The "Physical Pile" Paradox (Items 2.9, 2.10, 2.11) - CRITICAL**
- **The Flaw**: Modules are declared purely logical (forbidding `at`), yet they instantiate components with physical footprints. Without spatial keywords, sub-components stack at `[0,0,0]`, triggering mandatory `Error R15` (Collision).
- **The Fix**: Introduce "Layout Hierarchy" or "Physical Macros." 
    - **Relative Awareness**: Modules must support relative spatial keywords (e.g., `at last.right`) but forbid absolute coordinates.
    - **Macro Pin Promotion**: Modules must promote the physical anchors of internal pins to their logical boundaries so the parent space router can find them.
    - **Z-Context Inheritance**: Internal components must inherit their physical `z_nm` elevation from the parent space's `StackupManager` to ensure reusability across different profiles.

#### **2. Anchor Realization Lag (Items 1.1, 1.2) - HIGH**
- **The Flaw**: The "Wedge" and "Stretched" geometry artifacts occur because the AST is evaluated in a single pass. Pours and traces query bounding boxes before component transformations (rotation, offset) are "baked" into the registry.
- **The Fix**: Implement a **Two-Pass Realization Engine**. Pass 1 locks absolute Bounding Boxes; Pass 2 evaluates Pours/Routes against the frozen registry.

#### **3. Router Obstacle Blindness (Item 2.4) - HIGH**
- **The Flaw**: The A* router pathfinds through an empty universe, ignoring component bounding boxes until the DRC "bouncer" catches the violation post-routing.
- **The Fix**: Iterate through `HardwareSpace::component_bboxes` and blit them into the Router's cost-grid as `Cost::INFINITE` before pathfinding.

### **TIER 2: The "Pipeline Issues" (Pipeline Sequencing Errors)**
These are not architectural flaws. The code works perfectly, but is executing in the wrong order or skipping a handshake.

#### **1. Validation Hierarchy Blocker (Item 2.5)**
- **The Fix**: In `hwc-cli/src/commands/build.rs`, swap the order. Run `verify_alignment()` (LVS) before `check_physics()` (DRC). Electrical logic must be validated before atoms.

#### **2. Electric Ghosting (Item 2.1)**
- **The Fix**: Traces are currently `AnalyticTrace` (math lines). We must call `space.realize_analytic_routes()` right before GLB export to turn ghosts into physical copper voxels.

#### **3. Stale Error Coordinates (Item 1.3)**
- **The Fix**: Update `DiagnosticCollector` to pull coordinates from the IR (Compiled `ComponentInstance`) rather than the AST (Text Span).

#### **4. The "Blunt Keepout" Trap (Item 2.2)**
- **The Flaw**: A blunt 3D BBox check would block ground planes and traces from passing under or over components on different layers (e.g., M3 metal passing over standard cells).
- **The Fix**: Implement **Layer-Aware Keepout Zones (KOZ)**. In `VoxelGrid::stamp_pour`, the check must be: `if !self.is_inside_keepout_zone(x, y, z)`. Component metadata must define which layers are blocked, permitting flow on others.

### **TIER 3: "Working as Intended" (Design Philosophy)**
These are features of the Hardware Script language. **Do not attempt to "fix" them.**

- **Mandatory Layout (2.6)**: Physical spaces require physical footprints. Logical-only components in a `space` are correctly rejected.
- **Build requires Space (2.7)**: `build` translates to atoms; it requires a physical `space`. Use `check` for purely logical validation.
- **Comptime Equality Syntax (2.8)**: Single `=` for `if i = 0:` is an intended syntax unification for cleaner, context-aware code.
- **Voxel Saturation (2.3)**: 700mm at 1µm resolution is a physics limit. Use **LOD (Level of Detail) Profile** settings to skip atom-stamping during drafting.

---

## ⚔️ The Battle Plan

### **Tier 1: Native Fixes**
- [x] **Solve "Anchor Realization Lag"**: Two-pass engine + post-rotation bbox baking implemented (ir/mod.rs:261, component.rs:541). All component transforms locked before pours/traces.
- [x] **Cure "Router Blindness"**: Already wired via `add_component_obstacle` + SDF registration from `space.component_bboxes` (router.rs + automatic.rs). No further change needed.
- [x] **Address "Physical Pile" Paradox**: Full parser + AST for `layout:` inside `module:` (real LayoutStatement parsing, no stub). Compiler placement now prefers `module.intrinsic_layout` over external blocks (placement/module.rs). Sub-components no longer auto-pile. Relative anchors + pin promo next.
- [x] **Implement Macro Pin Promotion**: After internal placement, trace flattened routes to map module interface pins to their connected internal pins' physical positions (via netlist.get_pin_position). Register virtual module pins at the promoted anchors instead of (0,0,0). Wired in placement/module.rs.
- [x] **Z-Context Inheritance** (complete): Added resolve_z_expression helper to StackupManager (stackup_manager.rs). Wired into place_module_instance for sub-component Z resolution (using parent's stackup for semantic layers in intrinsic/external module layouts). Full inheritance for modules now functional.

### **Tier 2: Pipeline Fixes**
- [x] **Reorder Build Pipeline**: Current pipeline in build_cmd/mod.rs calls alignment::validate_alignment (LVS/electrical) before validation::run_validation_checks (DRC/physics/continuity). This satisfies the hierarchy (electrical before atoms). No swap needed in current refactored structure.
- [x] **Bake Electric Ghosts**: Added unconditional `space.realize_analytic_routes()` call right before export::export_all in build_cmd/mod.rs. Routes are now always baked to voxels for export (fixes ghost geometry when --skip-alignment).
- [x] **Update Error Mapping**: Map diagnostics to IR `Point3D` coordinates. All P44 errors (SubstrateOverlap, ComponentFloatingInAir, ComponentBuriedInSubstrate) now include `x_mm`, `y_mm`, `z_mm` from the compiled ComponentInstance.
- [x] **Implement Layer-Aware Keepouts (KOZ)**: Added `is_inside_keepout_zone(x_nm, y_nm, z_nm)` to `VoxelGrid` and `is_in_koz()` to `ComponentMetadata`. Components now have `blocked_z_ranges` — pours can flow under/over components on different Z-layers while still blocking on the component's own layers.

---

## Step 3: The Commit Gate (The Bouncer)
**Status**: Tier 1 100% + Reorder + Bake Electric Ghosts done. Warnings cleaned. Next: Update Error Mapping or Layer-Aware KOZ.
**Goal**: Ensure mathematical perfection. "If hwc build exits with code 0, the silicon is perfect."

- [x] Hard stop on validation failure.
- [x] Delete `board.glb` and `layout.dxf` on failure.
- [ ] Final Guarantee: Complete Tier 1 & 2 fixes to ensure zero-defect exports. (Anchor Lag + Router fixed; Physical Pile parser stub in.)
