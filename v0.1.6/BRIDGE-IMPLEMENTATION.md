# Bridge Implementation: The Ohmic Contact System

## Executive Summary

This document defines the implementation of the **Bridge System** - the architectural solution for handling material transitions (especially Silicon-to-Metal contacts) in Hardware Script. This is the exact boundary where "Design" meets "Physics."

**Core Principle**: The compiler remains a "dumb enforcer" with no hardcoded chemistry knowledge. All material transition rules are profile-driven.

---

## 1. The "Sandwich" Contract Architecture

### The Three-Layer Separation

```
┌─────────────────────────────────────────────────────────────┐
│ Layer 1: MATERIALS (The Bricks)                             │
│ - Defined in @std/materials/bridges.hw                      │
│ - Pure physical properties (resistance, work function)      │
│ - No logic, just data                                       │
└─────────────────────────────────────────────────────────────┘
                            ↓
┌─────────────────────────────────────────────────────────────┐
│ Layer 2: PROFILES (The Shopping List)                       │
│ - Defined in @foundry/tsmc/180nm.hw                         │
│ - Maps material pairs to bridge materials                   │
│ - Factory-specific rules                                    │
└─────────────────────────────────────────────────────────────┘
                            ↓
┌─────────────────────────────────────────────────────────────┐
│ Layer 3: COMPILER (The Enforcer)                            │
│ - Looks up bridge rules from active profile                 │
│ - Validates material transitions                            │
│ - Inserts bridge materials automatically                    │
└─────────────────────────────────────────────────────────────┘
```

---

## 2. Terminology: "Bridge" (Not "Junction")

**Decision**: Use **"Bridge"** as the canonical term.

**Rationale**:
- "Junction" is a scientific noun (describes a state)
- "Bridge" is an engineering verb-noun (describes a solution)
- Aligns with Hardware Script's preposition-based syntax (`to`, `at`, `by`)
- A **via** is the "pipe"; the **bridge** is the "material" that makes the pipe safe

---

## 3. The Priority Stack: How Bridges Are Selected

The compiler follows a **three-tier priority system** (Last Import Wins):

### Priority 1: Explicit Declaration (Highest)
```hw
route M1.source to GND:
    bridge: Cobalt_Silicide  # User override
```

### Priority 2: Profile Contract (Standard)
```hw
profile TSMC_180nm:
    bridge Silicon_N to Copper: Tungsten_Silicide
    bridge Silicon_P to Copper: Cobalt_Silicide
```

### Priority 3: Standard Library Default (Fallback)
```hw
# @std/profiles/generic.hw
profile Generic:
    bridge Silicon to Metal: Generic_Silicide
    bridge Die to Die: Generic_Gold_Bump
    bridge PCB to Component: SAC305_Solder
```

---

## 4. Material Library Structure

### Standard Bridge Materials (`@std/materials/bridges.hw`)

```hw
# Silicon-to-Metal Bridges (IC Fabrication)
material Titanium_Silicide:
    resistivity: 13e-8  # Ω·m
    work_function: 4.3  # eV
    max_temp: 600       # °C
    category: ohmic_contact

material Cobalt_Silicide:
    resistivity: 18e-8
    work_function: 4.5
    max_temp: 700
    category: ohmic_contact

material Tungsten_Silicide:
    resistivity: 70e-8
    work_function: 4.6
    max_temp: 800
    category: ohmic_contact

# Die-to-Die Bridges (3D Integration)
material Gold_Bump:
    resistivity: 2.4e-8
    melting_point: 1064
    category: die_interconnect

material Copper_Pillar:
    resistivity: 1.7e-8
    melting_point: 1085
    category: die_interconnect

# PCB-to-Component Bridges
material SAC305_Solder:
    resistivity: 11e-8
    melting_point: 217
    category: pcb_solder

material Conductive_Epoxy:
    resistivity: 1e-4
    max_temp: 150
    category: pcb_adhesive
```

---

## 5. Profile Bridge Tables

### Example: TSMC 180nm Profile

```hw
# @foundry/tsmc/180nm.hw
profile TSMC_180nm:
    # Bridge stacks (interface + fill)
    bridge Silicon_N to Aluminum:
        interface: Titanium_Silicide
        thickness: 50nm
        fill: Tungsten
    
    bridge Silicon_P to Aluminum:
        interface: Titanium_Silicide
        thickness: 50nm
        fill: Tungsten
    
    bridge Silicon_N to Copper:
        interface: Cobalt_Silicide
        thickness: 50nm
        fill: Tungsten
    
    bridge Silicon_P to Copper:
        interface: Cobalt_Silicide
        thickness: 50nm
        fill: Tungsten
    
    # Metal-to-Metal Bridges (if needed)
    bridge Aluminum to Copper:
        interface: Tungsten_Barrier
        thickness: 20nm
        fill: Tungsten
    
    # Substrate Bridges
    bridge Silicon to Substrate:
        interface: Backside_Metal
        thickness: 100nm
        fill: Copper
    
    # Default via fill material
    default_via_fill: Tungsten
```

### Example: Generic Profile (Default)

```hw
# @std/profiles/generic.hw
profile Generic:
    # Catch-all rules for prototyping
    bridge Silicon to Metal:
        interface: Generic_Silicide
        thickness: 50nm
        fill: Generic_Via_Fill
    
    bridge Metal to Metal:
        interface: Generic_Barrier
        thickness: 20nm
        fill: Generic_Via_Fill
    
    bridge Die to Die:
        interface: Generic_Gold_Bump
        thickness: 5um
        fill: Gold
    
    bridge PCB to Component:
        interface: SAC305_Solder
        thickness: 100um
        fill: SAC305_Solder
    
    default_via_fill: Generic_Via_Fill
```

---

## 6. Compiler Logic Flow

### The Bridge Lookup Algorithm

```rust
// Pseudocode for hwc-compiler/src/bridge_resolver.rs

struct BridgeStack {
    interface_material: MaterialId,  // The bridge (e.g., Silicide)
    fill_material: MaterialId,       // The via body (e.g., Tungsten)
    interface_thickness: f64,        // Typically 1 voxel (50nm)
}

fn resolve_bridge(
    from_material: MaterialId,
    to_material: MaterialId,
    profile: &Profile,
    explicit_override: Option<MaterialId>
) -> Result<BridgeStack, BridgeError> {
    
    // Priority 1: Explicit user override
    if let Some(bridge) = explicit_override {
        return Ok(BridgeStack {
            interface_material: bridge,
            fill_material: profile.default_via_fill,
            interface_thickness: profile.min_bridge_thickness,
        });
    }
    
    // Priority 2: Profile bridge table
    if let Some(stack) = profile.lookup_bridge_stack(from_material, to_material) {
        return Ok(stack);
    }
    
    // Priority 3: Standard library default
    if let Some(stack) = std_profile.lookup_bridge_stack(from_material, to_material) {
        return Ok(stack);
    }
    
    // Error: No bridge defined
    Err(BridgeError::ForbiddenJunction {
        from: from_material,
        to: to_material,
        suggestion: "Define a bridge in your profile or use an explicit bridge: declaration"
    })
}
```

### 6.5 Interface Stamping: The Sandwich Rule

**Critical Physical Reality**: In real chips, the bridge material (e.g., Silicide) is only a thin "crust" (typically 50nm) at the interface. The rest of the via is filled with a different material (e.g., Tungsten).

**The Problem**: If we fill the entire via with Silicide, resistance calculations will be incorrect (Silicide is ~10x more resistive than Tungsten).

**The Solution**: Interface stamping with compound stacks.

#### The Voxel Sandwich Algorithm

```
Layer 3 (Metal)     ┌─────────────┐
                    │   Tungsten  │  ← Via fill material
                    │   Tungsten  │
                    │   Tungsten  │
Layer 2 (ILD)       │   Tungsten  │
                    │   Tungsten  │
                    │  Silicide   │  ← Bridge interface (1 voxel)
Layer 1 (Silicon)   └─────────────┘
```

#### Implementation Steps

1. **Identify the interface voxel**: The exact Z-layer where Material A transitions to Material B
2. **Stamp the bridge disc**: Place exactly 1 voxel layer of bridge material at the interface
3. **Fill the pipe**: Remaining voxels use the standard via fill material from the profile
4. **Result**: Correct chemistry at the interface, correct conductivity in the pipe

```rust
// In hwc-compiler/src/auto_via_inserter.rs

fn insert_via_with_bridge(
    &mut self,
    from_layer: LayerId,
    to_layer: LayerId,
    position: Point3D
) -> Result<ViaId> {
    
    let from_material = self.get_layer_material(from_layer);
    let to_material = self.get_layer_material(to_layer);
    
    // Resolve the bridge stack (interface + fill)
    let bridge_stack = self.resolve_bridge(
        from_material,
        to_material,
        self.active_profile,
        None  // No explicit override in auto mode
    )?;
    
    // Calculate via geometry
    let via_height = self.calculate_via_height(from_layer, to_layer);
    let interface_voxels = (bridge_stack.interface_thickness / self.voxel_size).ceil() as usize;
    
    // Create the compound via structure
    let via = CompoundVia {
        from_layer,
        to_layer,
        position,
        // Bottom interface (touching source material)
        interface_material: bridge_stack.interface_material,
        interface_voxels,
        // Main body (the conductive pipe)
        fill_material: bridge_stack.fill_material,
        diameter: self.profile.min_via_diameter,
    };
    
    // Stamp voxels with compound structure
    self.stamp_compound_via(&via)?;
    
    self.vias.push(via);
    Ok(via.id)
}

fn stamp_compound_via(&mut self, via: &CompoundVia) -> Result<()> {
    let z_start = self.get_layer_z(via.from_layer);
    let z_end = self.get_layer_z(via.to_layer);
    
    for z in z_start..=z_end {
        let material = if z < z_start + via.interface_voxels {
            via.interface_material  // Bridge disc at bottom
        } else {
            via.fill_material       // Tungsten pipe for the rest
        };
        
        self.voxel_grid.stamp_cylinder(
            via.position,
            z,
            via.diameter,
            material
        );
    }
    
    Ok(())
}
```

---

## 7. Design Rule Checking (DRC): The "Bouncer" Philosophy

### Core Principle: Explicit Assembly, Automatic Routing

Hardware Script follows the **"Bouncer at Assembly, Butler at Router"** philosophy:

- **Assembly Level** (Manual `add pour`, `add contact`): The compiler is a **Bouncer** - it enforces physics but does NOT auto-inject bridges
- **Router Level** (High-level `route` command): The compiler is a **Butler** - it automatically inserts bridges per profile rules

**Rationale**: This matches industry standards (Cadence, Synopsys, Magic VLSI) where manual layout requires explicit contact layers, but high-level routing tools handle connectivity automatically.

### 7.1 Forbidden Junction Detection (Assembly Level)

When a user manually places materials that require a bridge, the DRC **fails the build** with a clear error:

```hw
# User code (WRONG - missing bridge)
space chip:
    add pour(Silicon_N) on z:1
    add pour(Copper) on z:3  # Direct contact - FORBIDDEN!
```

```
Error P45: Forbidden Junction
  ┌─ design.hw:42:5
  │
42│     add pour(Copper) on z:3
  │     ^^^^^^^^^^^^^^^^^^^^^^^ Direct contact between Copper and Silicon_N
  │
  = note: Profile 'TSMC_180nm' forbids direct Silicon-to-Metal contact
  = note: Required bridge: Cobalt_Silicide (per profile rules)
  = help: Insert bridge material explicitly:
          
          add pour(Silicon_N) on z:1
          add contact(Cobalt_Silicide) at [x, y] spanning z:2 to z:2
          add pour(Copper) on z:3
```

**Why No Auto-Injection at Assembly Level?**

1. **Physical Integrity**: User must know they're adding resistance and Z-thickness
2. **Educational Value**: Students learn WHY the chip works (the silicide bridge)
3. **Foundry Control**: Different regions may need different bridges (speed vs. power optimization)
4. **Atomic Traceability**: Every atom is explicit - no "magic"

### 7.2 Automatic Bridge Insertion (Router Level)

When using high-level routing commands, the compiler **automatically** handles bridges:

```hw
# User code (HIGH-LEVEL)
space chip:
    route M1.source to GND  # Compiler handles bridge automatically
```

**Compiler behavior**:
1. Detects Silicon → Metal transition
2. Looks up bridge in profile: `Silicon_N to Copper: Cobalt_Silicide`
3. Automatically stamps interface voxel with Cobalt_Silicide
4. Fills remaining via with Tungsten (via fill material)

### 7.3 Bridge Validation Rules

The DRC system validates:
1. **Material Compatibility**: Bridge material is compatible with both source and target
2. **Thermal Limits**: Bridge can handle the operating temperature
3. **Geometric Constraints**: Bridge thickness meets profile requirements
4. **Electrical Properties**: Bridge resistance is within acceptable range
5. **Interface Physics**: Work function and Schottky barrier are acceptable

---

## 8. Implementation Roadmap

### Phase 1: Core Infrastructure
- [ ] **hwc-parser**: Add `bridge` keyword and syntax
  - Parse `bridge A to B: Material` in profile blocks
  - Parse `bridge: Material` in route/contact blocks
- [ ] **hwc-compiler/ir**: Add `BridgeTable` to IR
  - Store profile bridge mappings
  - Add bridge resolution logic
- [ ] **hwc-engine**: Support bridge materials in VoxelGrid
  - Allow multi-layer material stacks
  - Handle bridge material IDs

### Phase 2: AutoVia Integration
- [ ] **hwc-compiler/auto_via_inserter**: Integrate bridge lookup
  - Query bridge table during via insertion
  - Insert bridge materials automatically
- [ ] **hwc-compiler/bridge_resolver**: Implement priority stack
  - Handle explicit overrides
  - Profile lookup
  - Default fallback

### Phase 3: Standard Library
- [ ] **stdlib/materials/bridges.hw**: Define standard bridge materials
  - Silicides (Ti, Co, W, Ni)
  - Die interconnects (Au bump, Cu pillar)
  - PCB solders (SAC305, SnPb)
- [ ] **stdlib/profiles/generic.hw**: Create default profile
  - Generic bridge rules for prototyping

### Phase 4: DRC Integration
- [ ] **hwc-diagnostics**: Add forbidden junction errors
  - Detect direct material contacts
  - Suggest appropriate bridges
- [ ] **hwc-compiler/drc**: Validate bridge properties
  - Thermal compatibility
  - Electrical resistance
  - Geometric constraints

---

## 9. User Experience Examples

### Beginner: "Just Works" (Router Level)
```hw
import @std/profiles/generic

space chip:
    route M1.source to GND  # Compiler auto-inserts Generic_Silicide interface + Generic_Via_Fill
```

### Intermediate: Profile-Driven (Router Level)
```hw
import @foundry/tsmc/180nm

space chip:
    route M1.source to GND  # Compiler uses TSMC's Cobalt_Silicide interface + Tungsten fill
```

### Advanced: Explicit Control (Router Level)
```hw
import @foundry/tsmc/180nm

space chip:
    route M1.source to GND:
        bridge: Tungsten_Silicide  # Override TSMC default interface material
```

### Expert: Manual Assembly (Assembly Level)
```hw
import @foundry/tsmc/180nm

space chip:
    # Manual control - user must be explicit about every layer
    add pour(Silicon_N) on z:1:
        boundary: [x: 0, y: 0] to [x: 100um, y: 100um]
    
    # User MUST explicitly add the bridge contact
    add contact(Cobalt_Silicide) at [50um, 50um] spanning z:2 to z:2:
        diameter: 500nm
    
    # Then add the via fill
    add contact(Tungsten) at [50um, 50um] spanning z:3 to z:5:
        diameter: 500nm
    
    add pour(Copper) on z:6:
        boundary: [x: 0, y: 0] to [x: 100um, y: 100um]
```

**Note**: At Assembly level, if you forget the `Cobalt_Silicide` contact, the DRC throws **Error P45: Forbidden Junction** and the build fails. No auto-injection.

---

## 10. The Voxel Reality: 3D Grid Occupancy

### How Bridges Occupy Space in the Voxel Grid

Hardware Script uses a **3D voxel grid** where materials occupy discrete cubic volumes. There are no "holes" - only **material occupancy**.

### The Layer Stack Reality

```
┌─────────────────────────────────────────────────────────┐
│ Layer 3: Metal (Copper)                                 │  z:6
│ ─────────────────────────────────────────────────────── │
│ Layer 2: Inter-Layer Dielectric (SiO2 "Glass")         │  z:2-5
│          ↑                                              │
│          └─ This is where the CONTACT lives             │
│ ─────────────────────────────────────────────────────── │
│ Layer 1: Silicon (Transistor)                           │  z:1
└─────────────────────────────────────────────────────────┘
```

### The Contact Pillar: A Vertical Sandwich

A contact is not a flat square - it's a **3D pillar** through the dielectric layer.

**Example**: If your voxel resolution is 10nm and Layer 2 (ILD) is 100nm thick, your contact is **10 voxels tall**.

```
Voxel View (Side Profile):

z:6  ┌───────────┐  ← Metal layer (Copper)
     │           │
z:5  │ Tungsten  │  ← Via fill (body of pipe)
z:4  │ Tungsten  │
z:3  │ Tungsten  │
z:2  │ Silicide  │  ← Bridge interface (1 voxel touching Silicon)
     └───────────┘
z:1  ═══════════════  ← Silicon layer
```

### Where Does the Bridge Sit?

**Answer**: The bridge occupies the **interface voxels** within the contact pillar.

- **NOT in the Silicon layer** (z:1) - that would change transistor physics
- **NOT in the Metal layer** (z:6) - that should be pure copper for routing
- **IN the first voxel of the contact** (z:2) - the exact interface where the pillar leaves Silicon

### The "Overwrite Occupancy" Rule

We don't "dig holes" in Hardware Script. We **overwrite material IDs** in the voxel grid.

**Manual Assembly Example**:
```hw
# Step 1: Draw Silicon base
add pour(Silicon_N) on z:1

# Step 2: Draw bridge interface (bottom of pillar)
add contact(Ti_Silicide) at [100nm, 100nm] spanning z:2 to z:2:
    diameter: 500nm

# Step 3: Draw via fill (rest of pillar)
add contact(Tungsten) at [100nm, 100nm] spanning z:3 to z:5:
    diameter: 500nm

# Step 4: Draw metal layer
add pour(Copper) on z:6
```

**Automatic Router Example**:
```hw
# High-level command
route M1.source to GND

# Compiler automatically:
# 1. Detects Silicon (z:1) → Metal (z:6) transition
# 2. Looks up bridge: Silicon_N to Copper → Cobalt_Silicide + Tungsten
# 3. Stamps z:2 with Cobalt_Silicide (interface)
# 4. Stamps z:3-5 with Tungsten (fill)
```

### Why This Is God-Tier: Atomic Traceability

When exported to GDSII:
- The foundry sees a **Silicide layer square** at coordinate [100nm, 100nm] on z:2
- The foundry sees a **Contact layer square** at the same coordinate on z:3-5
- The foundry knows: "Etch here, deposit silicide, fill with tungsten"

**Result**: The Silicon stays Silicon, the Metal stays Metal, and the Contact handles the transition physics.

---

## 11. HPM Integration: The Business Model

### Foundry Package Example

```hw
# @tsmc/180nm package structure
@tsmc/180nm/
├── profile.hw          # Bridge table (proprietary rules)
├── materials.hw        # Material definitions
├── drc_rules.hw        # Design rule constraints
└── README.md           # Usage documentation
```

### Package Installation
```bash
hpm install @tsmc/180nm
```

### Package Usage
```hw
import @tsmc/180nm

space my_chip:
    # All bridges automatically use TSMC-certified materials
    route power_rail to substrate
```

---

## 11. HPM Integration: The Business Model

### Foundry Package Example

```hw
# @tsmc/180nm package structure
@tsmc/180nm/
├── profile.hw          # Bridge table with compound stacks (proprietary rules)
├── materials.hw        # Material definitions
├── drc_rules.hw        # Design rule constraints
└── README.md           # Usage documentation
```

### Package Installation
```bash
hpm install @tsmc/180nm
```

### Package Usage
```hw
import @tsmc/180nm

space my_chip:
    # All bridges automatically use TSMC-certified materials
    # Interface: Cobalt_Silicide (50nm)
    # Fill: Tungsten (remaining via height)
    route power_rail to substrate
```

**Business Value**: Foundries can monetize their process knowledge by publishing certified HPM packages with optimized bridge stacks (interface + fill materials).

---

## 12. Why This Is "Zen"

### Separation of Concerns
- **Compiler**: Geometry + Material IDs (no chemistry knowledge)
- **Physics Engine**: Material properties (no design rules)
- **Profile**: Factory rules (no hardcoded logic)

### "Bouncer at Assembly, Butler at Router"
- **Assembly Level**: User is master of every atom - compiler enforces physics but doesn't hide complexity
- **Router Level**: Compiler handles connectivity automatically using profile rules
- **Industry Alignment**: Matches Cadence/Synopsys (abstract layers) and Magic VLSI (explicit contacts)

### Democratic and Professional
- **Students**: Use generic profile with router commands - things "just work"
- **Engineers**: Import foundry package - get certified compound stacks
- **Researchers**: Define custom materials and bridges at assembly level

### Atomic Truth
- Bridge is a compound stack (interface + fill) with real physical properties
- DRC validates based on material constants (resistivity, work function, thermal limits)
- No "magic" - everything is explicit and traceable in the voxel grid
- Correct resistance modeling (thin Silicide interface + thick Tungsten fill)

---

## 13. Next Steps

This document completes the **Bridge** portion of Assembly Completeness with the following critical refinements:

1. ✅ **Gap A Fixed**: Interface stamping with compound stacks (thin Silicide interface + thick Tungsten fill)
2. ✅ **Gap B Clarified**: "Bouncer" philosophy at Assembly level (no auto-injection, explicit contacts required)
3. ✅ **Voxel Reality Defined**: Bridges occupy interface voxels in the 3D grid, preserving atomic traceability

The next connection types to address are:

1. **TSV (Through-Silicon Via)**: The "Final Boss" - requires liner materials (insulator sleeves) to prevent substrate short-circuits. Critical for 3D-stacked AI chips.
2. **Wire Bonds**: PCB-to-die connections with different physics (mechanical + electrical)
3. **Solder Bumps**: Flip-chip and BGA connections for package-level interconnects

**Question**: Ready to tackle TSV implementation? This is the most complex connection type because it requires:
- **Liner materials** (insulator sleeve around the via)
- **Multi-die stack coordination** (via must align across multiple silicon layers)
- **Keep-out zones** (TSVs create stress fields that affect nearby transistors)

---

## Appendix: Key Terminology

| Term | Definition |
|------|------------|
| **Bridge** | A material that enables safe electrical connection between two incompatible materials |
| **Compound Stack** | A multi-layer via structure: interface material (bridge) + fill material (conductor) |
| **Interface Stamping** | Placing a thin bridge layer (1 voxel) at the material transition point |
| **Ohmic Contact** | A low-resistance bridge between semiconductor and metal |
| **Silicide** | A compound of silicon and metal (e.g., TiSi₂, CoSi₂) used as a bridge |
| **Profile** | A foundry-specific configuration defining bridge rules and constraints |
| **Bouncer** | Compiler behavior at Assembly level - enforces physics, no auto-injection |
| **Butler** | Compiler behavior at Router level - automatically inserts bridges per profile |
| **Voxel Grid** | 3D discrete grid where materials occupy cubic volumes |
| **ILD** | Inter-Layer Dielectric - the insulator layer between conductors (usually SiO2) |

---

**Status**: Ready for implementation  
**Priority**: High (Core Assembly Completeness)  
**Dependencies**: Profile parsing, Material library, AutoVia system, VoxelGrid compound stacks
