# Technology Strategy Architecture Refactoring Plan

## Problem Statement

The current architecture has technology-specific behavior (PCB vs ASIC) scattered throughout the codebase using `if/else` checks on `min_annular_ring_nm`. This violates the Single Responsibility Principle and makes the code fragile and hard to maintain.

The PDK already declares `technology: "ASIC"` or `technology: "PCB"` at the beginning, but this information is not properly propagated through the system as a first-class architectural component.

## Root Cause

1. **Late Binding**: Technology strategy is determined ad-hoc in each subsystem
2. **Scattered Conditionals**: `if min_annular_ring_nm > 0` appears in multiple files
3. **No Central Authority**: Each module makes its own technology decisions
4. **Duplicated Logic**: Same conditional appears in routing, validation, export, DRC

## Proposed Solution

### Phase 1: Elevate TechnologyStrategy to Core Infrastructure

#### 1.1 Move TechnologyStrategy to hwc-types (Shared Core)

**Why**: `TechnologyStrategy` is a fundamental type that should be available to all crates without circular dependencies.

**File**: `hwc/crates/hwc-types/src/lib.rs`

```rust
//! Core type definitions shared across Hardware Script compiler crates.

/// Strongly-typed net ID (newtype wrapper around u32).
#[derive(Clone, Copy, Debug, PartialEq, Eq, PartialOrd, Ord, Hash, serde::Serialize, serde::Deserialize)]
pub struct NetId(pub u32);

impl NetId {
    pub const UNCONNECTED: NetId = NetId(0);
    
    #[inline]
    pub const fn new(id: u32) -> Self {
        Self(id)
    }
    
    #[inline]
    pub const fn raw(self) -> u32 {
        self.0
    }
    
    #[inline]
    pub const fn is_unconnected(self) -> bool {
        self.0 == 0
    }
}

/// Technology-specific fabrication strategy.
///
/// Determined once during compilation from PDK profile and used consistently
/// throughout routing, validation, export, and DRC subsystems.
#[derive(Debug, Clone, Copy, PartialEq, Eq, Hash, serde::Serialize, serde::Deserialize)]
pub enum TechnologyStrategy {
    /// PCB technology: Drilled/plated vias with annular rings.
    /// - Pads extend beyond drill holes
    /// - Requires trace_width/2 projection for escape routing
    /// - Annular ring expansion needed for connectivity validation
    Pcb,
    
    /// ASIC technology: Deposited contacts with flush boundaries.
    /// - Contacts are rectangular blocks with no overhang
    /// - Traces connect flush to contact edges
    /// - No annular ring expansion needed
    Asic,
}

impl TechnologyStrategy {
    /// Determine technology strategy from min_annular_ring constraint.
    ///
    /// # Decision Rule
    /// - If `min_annular_ring_nm > 0`: PCB (pads have overhang)
    /// - If `min_annular_ring_nm == 0`: ASIC (flush contacts)
    #[inline]
    pub const fn from_annular_ring(min_annular_ring_nm: i64) -> Self {
        if min_annular_ring_nm > 0 {
            Self::Pcb
        } else {
            Self::Asic
        }
    }

    /// Get annular ring expansion value for connectivity validation.
    ///
    /// Returns the amount to expand contact bounding boxes:
    /// - PCB: Returns the actual annular ring value
    /// - ASIC: Returns 0 (no expansion)
    #[inline]
    pub const fn contact_expansion(&self, min_annular_ring_nm: i64) -> i64 {
        match self {
            Self::Pcb => min_annular_ring_nm,
            Self::Asic => 0,
        }
    }

    /// Calculate port escape clearance for routing.
    ///
    /// # PCB Mode
    /// Escape clearance = (trace_width / 2) + min_clearance
    /// Accounts for trace width projection beyond pad edge.
    ///
    /// # ASIC Mode
    /// Escape clearance = min_clearance
    /// Traces connect flush to contact edges.
    #[inline]
    pub const fn port_escape_clearance(
        &self,
        trace_width_nm: i64,
        min_clearance_nm: i64,
    ) -> i64 {
        match self {
            Self::Pcb => (trace_width_nm / 2) + min_clearance_nm,
            Self::Asic => min_clearance_nm,
        }
    }

    /// Calculate obstacle inflation for navigable space extraction.
    #[inline]
    pub const fn obstacle_inflation(
        &self,
        trace_width_nm: i64,
        min_clearance_nm: i64,
    ) -> i64 {
        self.port_escape_clearance(trace_width_nm, min_clearance_nm)
    }

    /// Human-readable technology name for logging.
    #[inline]
    pub const fn name(&self) -> &'static str {
        match self {
            Self::Pcb => "PCB",
            Self::Asic => "ASIC",
        }
    }

    /// Check if this is PCB technology.
    #[inline]
    pub const fn is_pcb(&self) -> bool {
        matches!(self, Self::Pcb)
    }

    /// Check if this is ASIC technology.
    #[inline]
    pub const fn is_asic(&self) -> bool {
        matches!(self, Self::Asic)
    }
}

impl Default for TechnologyStrategy {
    /// Default to ASIC strategy (most conservative).
    fn default() -> Self {
        Self::Asic
    }
}
```

**Impact**: 
- Update `Cargo.toml` dependencies: All crates can now import from `hwc-types`
- Remove `technology_strategy.rs` from `hwc-engine/src/geometry_router/`

---

#### 1.2 Add TechnologyStrategy to HardwareSpace

**File**: `hwc/crates/hwc-engine/src/space/mod.rs`

```rust
use hwc_types::TechnologyStrategy;

pub struct HardwareSpace {
    pub name: CompactString,
    pub dimensions: Dimensions,
    pub substrate_material_id: MaterialId,
    pub view: SpaceView,
    pub material_registry: MaterialRegistry,
    pub entity_graph: EntityGraph,
    pub netlist: NetlistArena,
    pub vias: Vec<crate::geometry_router::Via>,
    pub pours: Vec<PourMetadata>,
    pub contacts: Vec<ContactMetadata>,
    pub net_classifications: FxHashMap<CompactString, NetClassification>,
    pub substrate_bbox: Option<crate::geometry::BoundingBox>,
    pub component_bboxes: FxHashMap<CompactString, crate::geometry::BoundingBox>,
    pub analytic_routes: Vec<AnalyticTrace>,
    pub fabrication_constraints: Option<hwc_materials::ConstraintSet>,
    pub keep_out_zones: Vec<KeepOutZone>,
    pub resolution_nm: i64,
    pub stackup_layers: Vec<StackupLayer>,
    
    /// **v0.2.0: Technology strategy determined from PDK profile**
    /// Set once during compilation and used consistently throughout all subsystems.
    /// No scattered conditionals - this is the single source of truth.
    pub technology_strategy: TechnologyStrategy,
}

impl HardwareSpace {
    pub fn new(
        name: CompactString,
        dimensions: Dimensions,
        substrate_material_id: MaterialId,
        material_registry: MaterialRegistry,
        view: SpaceView,
        resolution_nm: i64,
        technology_strategy: TechnologyStrategy, // NEW PARAMETER
    ) -> Self {
        let entity_graph = EntityGraph::new();
        let netlist = NetlistArena::new();

        Self {
            name,
            dimensions,
            substrate_material_id,
            material_registry,
            view,
            entity_graph,
            netlist,
            vias: Vec::new(),
            pours: Vec::new(),
            contacts: Vec::new(),
            net_classifications: FxHashMap::default(),
            substrate_bbox: None,
            component_bboxes: FxHashMap::default(),
            analytic_routes: Vec::new(),
            fabrication_constraints: None,
            keep_out_zones: Vec::new(),
            resolution_nm,
            stackup_layers: Vec::new(),
            technology_strategy, // NEW FIELD
        }
    }
    
    // ... rest of implementation
}
```

**Impact**:
- All `HardwareSpace::new()` call sites must pass `technology_strategy`
- This is set ONCE during space creation and never changes

---

### Phase 2: Update Space Creation to Set Technology Strategy

#### 2.1 Update Space Builder (Compilation Entry Point)

**File**: `hwc/crates/hwc-compiler/src/ir/space/builder.rs` (or wherever space is created)

Find where `HardwareSpace::new()` is called and add:

```rust
use hwc_types::TechnologyStrategy;

// During space creation, extract technology from profile:
let technology_strategy = profile_def
    .constraints
    .as_ref()
    .map(|c| {
        let min_annular = symbol_table.measurement_to_nm(&c.via.min_annular_ring)
            .unwrap_or(0);
        TechnologyStrategy::from_annular_ring(min_annular)
    })
    .unwrap_or_default(); // Default to ASIC

let space = HardwareSpace::new(
    name,
    dimensions,
    substrate_material_id,
    material_registry,
    view,
    resolution_nm,
    technology_strategy, // Pass it here
);
```

**Search Pattern**: Find all `HardwareSpace::new(` calls and update them.

---

### Phase 3: Eliminate Scattered Conditionals

#### 3.1 Update Connectivity Validation

**File**: `hwc/crates/hwc-cli/src/commands/build_cmd/validation/utils.rs`

**BEFORE** (scattered conditional):
```rust
let annular_ring_nm = space
    .fabrication_constraints
    .as_ref()
    .map(|c| c.min_annular_ring_nm)
    .unwrap_or(0);
```

**AFTER** (use strategy):
```rust
use hwc_types::TechnologyStrategy;

pub fn convert_metadata_to_physics(
    space: &HardwareSpace,
) -> (
    Vec<hwc_physics::connectivity::SubstrateLayerMetadata>,
    Vec<hwc_physics::RouteSegmentMetadata>,
) {
    let mut physics_substrate_layers = Vec::new();
    
    // v0.2.0: Use technology strategy from space (set during compilation)
    let strategy = space.technology_strategy;
    let annular_ring_nm = space
        .fabrication_constraints
        .as_ref()
        .map(|c| c.min_annular_ring_nm)
        .unwrap_or(0);
    
    let contact_expansion = strategy.contact_expansion(annular_ring_nm);
    
    eprintln!("[CONNECTIVITY] Technology: {}", strategy.name());
    eprintln!("[CONNECTIVITY] Contact expansion: {} nm", contact_expansion);
    
    // ... rest of function
    
    // When processing contacts, use contact_expansion instead of conditional:
    for layer in space.entity_graph.get_substrate_layers() {
        let expansion = if layer.layer_type == SubstrateLayerType::Contact {
            contact_expansion
        } else {
            0
        };
        
        // Apply expansion uniformly
        // ...
    }
}
```

---

#### 3.2 Update Router (Port Escape)

**File**: `hwc/crates/hwc-engine/src/geometry_router/port_escape.rs`

**BEFORE**:
```rust
let strategy = TechnologyStrategy::from_constraints(fabrication);
let clearance = strategy.calculate_port_escape_clearance(trace_width, min_clearance);
```

**AFTER**:
```rust
// Technology strategy is already available in space context
let clearance = space.technology_strategy.port_escape_clearance(
    trace_width_nm,
    min_clearance_nm
);
```

---

#### 3.3 Update Navigable Space

**File**: `hwc/crates/hwc-engine/src/geometry_router/navigable_space.rs`

**BEFORE** (creates temporary FabricationConstraints just to determine strategy):
```rust
let temp_fab = FabricationConstraints {
    min_annular_ring_nm,
    // ... other dummy fields
};
let strategy = TechnologyStrategy::from_constraints(&temp_fab);
```

**AFTER** (pass strategy directly):
```rust
pub fn new(
    raw_obstacles: Vec<BoundingBox>,
    trace_width_nm: i64,
    min_clearance_nm: i64,
    technology_strategy: TechnologyStrategy, // NEW PARAMETER
) -> Result<Self, SpatialDecompositionError> {
    let inflation = technology_strategy.obstacle_inflation(
        trace_width_nm,
        min_clearance_nm
    );
    
    eprintln!("[NAVIGABLE SPACE] Technology: {}", technology_strategy.name());
    eprintln!("[NAVIGABLE SPACE] Calculated inflation: {} nm", inflation);
    
    // ... rest
}
```

---

#### 3.4 Update Via Operations

**File**: `hwc/crates/hwc-engine/src/geometry_router/router/via_operations.rs`

**BEFORE** (scattered usage of `fabrication.min_annular_ring_nm`):
```rust
let annular_ring_nm = fabrication.min_annular_ring_nm;
let total_radius = (via_diameter + 2 * annular_ring) / 2;
```

**AFTER** (use strategy):
```rust
let annular_ring_nm = space.technology_strategy.contact_expansion(
    fabrication.min_annular_ring_nm
);
let total_radius = (via_diameter + 2 * annular_ring_nm) / 2;
```

---

#### 3.5 Update DRC Checks

**File**: `hwc/crates/hwc-engine/src/design_rule_check/via_checks.rs`

**BEFORE**:
```rust
let min_annular_ring_nm = fabrication.min_annular_ring_nm;

if actual_annular_ring_nm < min_annular_ring_nm {
    // Report violation
}
```

**AFTER**:
```rust
// Only check annular ring for PCB technology
if space.technology_strategy.is_pcb() {
    let min_annular_ring_nm = fabrication.min_annular_ring_nm;
    
    if actual_annular_ring_nm < min_annular_ring_nm {
        // Report violation
    }
}
// For ASIC: Skip annular ring checks entirely (they don't apply)
```

---

#### 3.6 Update Query Engine

**File**: `hwc/crates/hwc-compiler/src/ir/query_engine.rs`

**BEFORE**:
```rust
pub struct NavigableSpaceContext {
    pub trace_width_nm: i64,
    pub min_clearance_nm: i64,
    pub min_annular_ring_nm: i64, // REMOVE THIS
    pub board_bounds: BoundingBox,
}
```

**AFTER**:
```rust
pub struct NavigableSpaceContext {
    pub trace_width_nm: i64,
    pub min_clearance_nm: i64,
    pub technology_strategy: TechnologyStrategy, // ADD THIS
    pub board_bounds: BoundingBox,
}

// When calling SpatialDecomposer::new:
SpatialDecomposer::new(
    obstacles,
    context.trace_width_nm,
    context.min_clearance_nm,
    context.technology_strategy, // Pass strategy, not raw value
)
```

---

### Phase 4: Update All Call Sites

#### Files to Update (Search and Replace)

1. **Space Creation**:
   - `hwc-compiler/src/ir/space/builder.rs`
   - Any test files creating `HardwareSpace`

2. **Router Entry Points**:
   - `hwc-engine/src/geometry_router/mod.rs`
   - `hwc-compiler/src/ir/routing/global/engine.rs`

3. **Validation Entry Points**:
   - `hwc-cli/src/commands/build_cmd/validation/mod.rs`
   - `hwc-cli/src/commands/build_cmd/alignment.rs`

4. **Export Systems**:
   - Any GDSII/Gerber exporters that use annular ring values

5. **Component Unrolling**:
   - `hwc-compiler/src/ir/placement/component/unrolling.rs`
   - Replace direct `min_annular_ring_nm` access with `space.technology_strategy`

---

### Phase 5: Testing Strategy

#### Unit Tests

Add to `hwc-types/src/lib.rs`:

```rust
#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn test_pcb_strategy() {
        let strategy = TechnologyStrategy::from_annular_ring(150_000);
        assert_eq!(strategy, TechnologyStrategy::Pcb);
        assert!(strategy.is_pcb());
        assert!(!strategy.is_asic());
        
        let expansion = strategy.contact_expansion(150_000);
        assert_eq!(expansion, 150_000);
        
        let escape = strategy.port_escape_clearance(200_000, 150_000);
        assert_eq!(escape, 250_000); // (200k/2) + 150k
    }

    #[test]
    fn test_asic_strategy() {
        let strategy = TechnologyStrategy::from_annular_ring(0);
        assert_eq!(strategy, TechnologyStrategy::Asic);
        assert!(strategy.is_asic());
        assert!(!strategy.is_pcb());
        
        let expansion = strategy.contact_expansion(0);
        assert_eq!(expansion, 0);
        
        let escape = strategy.port_escape_clearance(200, 0);
        assert_eq!(escape, 0); // ASIC: no offset
    }

    #[test]
    fn test_default_is_asic() {
        let strategy = TechnologyStrategy::default();
        assert_eq!(strategy, TechnologyStrategy::Asic);
    }
}
```

#### Integration Tests

Create `hwc/tests/technology_strategy_integration.rs`:

```rust
//! Integration test: Verify technology strategy flows through entire pipeline

#[test]
fn test_asic_cmos_inverter_with_strategy() {
    // Compile ASIC test case
    let result = compile_test_file("tests/ASIC-Minimal/cmos_inverter.hw");
    assert!(result.is_ok());
    
    let space = result.unwrap();
    
    // Verify technology strategy was set correctly
    assert_eq!(space.technology_strategy, TechnologyStrategy::Asic);
    
    // Verify no annular ring expansion was applied
    // (Check substrate layer bounding boxes match exactly with no padding)
}

#[test]
fn test_pcb_design_with_strategy() {
    // Compile PCB test case
    let result = compile_test_file("tests/PCB/simple_board.hw");
    assert!(result.is_ok());
    
    let space = result.unwrap();
    
    // Verify technology strategy was set correctly
    assert_eq!(space.technology_strategy, TechnologyStrategy::Pcb);
    
    // Verify annular ring expansion was applied
}
```

---

## Migration Checklist

### Step 1: Core Type Migration
- [ ] Move `TechnologyStrategy` to `hwc-types/src/lib.rs`
- [ ] Add full documentation and inline methods
- [ ] Add unit tests
- [ ] Update `Cargo.toml` dependencies

### Step 2: HardwareSpace Integration
- [ ] Add `technology_strategy: TechnologyStrategy` field to `HardwareSpace`
- [ ] Update `HardwareSpace::new()` signature
- [ ] Add getter method if needed
- [ ] Update serialization/deserialization

### Step 3: Compilation Pipeline
- [ ] Update space builder to extract and set technology strategy
- [ ] Verify strategy is set during profile parsing
- [ ] Add debug logging for strategy selection

### Step 4: Router Subsystem
- [ ] Update `port_escape.rs` to use `space.technology_strategy`
- [ ] Update `interface_escape.rs`
- [ ] Update `navigable_space.rs` to accept strategy parameter
- [ ] Update `via_operations.rs`
- [ ] Remove temporary `FabricationConstraints` creation

### Step 5: Validation Subsystem
- [ ] Update `validation/utils.rs` to use `space.technology_strategy`
- [ ] Remove scattered `min_annular_ring_nm > 0` checks
- [ ] Update connectivity validation
- [ ] Update continuity validation

### Step 6: DRC Subsystem
- [ ] Update `via_checks.rs` to skip annular ring checks for ASIC
- [ ] Update other DRC rules that depend on technology

### Step 7: Export Subsystem
- [ ] Update GDSII exporter
- [ ] Update Gerber exporter
- [ ] Update drill file generator

### Step 8: Testing
- [ ] Run existing test suite (should pass with no behavior changes)
- [ ] Add new unit tests for `TechnologyStrategy` methods
- [ ] Add integration tests for ASIC and PCB workflows
- [ ] Verify `cmos_inverter.hw` still compiles and validates

### Step 9: Documentation
- [ ] Update architecture docs
- [ ] Update API docs
- [ ] Add migration notes for custom tooling

### Step 10: Cleanup
- [ ] Remove old `technology_strategy.rs` from `geometry_router/`
- [ ] Remove scattered conditional logic
- [ ] Remove `min_annular_ring_nm` from contexts where not needed
- [ ] Run `cargo clippy` and fix warnings

---

## Files Affected (Complete List)

### Core Types
1. `hwc/crates/hwc-types/src/lib.rs` - Add TechnologyStrategy enum

### Engine
2. `hwc/crates/hwc-engine/src/space/mod.rs` - Add field to HardwareSpace
3. `hwc/crates/hwc-engine/src/geometry_router/mod.rs` - Remove old technology_strategy.rs, update imports
4. `hwc/crates/hwc-engine/src/geometry_router/port_escape.rs` - Use space.technology_strategy
5. `hwc/crates/hwc-engine/src/geometry_router/interface_escape.rs` - Use space.technology_strategy
6. `hwc/crates/hwc-engine/src/geometry_router/navigable_space.rs` - Accept strategy parameter
7. `hwc/crates/hwc-engine/src/geometry_router/router/via_operations.rs` - Use strategy methods
8. `hwc/crates/hwc-engine/src/design_rule_check/via_checks.rs` - Conditional checks based on strategy

### Compiler
9. `hwc/crates/hwc-compiler/src/ir/space/builder.rs` - Set technology_strategy during creation
10. `hwc/crates/hwc-compiler/src/ir/query_engine.rs` - Replace min_annular_ring_nm with strategy
11. `hwc/crates/hwc-compiler/src/ir/routing/global/engine.rs` - Pass strategy to router
12. `hwc/crates/hwc-compiler/src/ir/placement/component/unrolling.rs` - Use strategy for pad calculations
13. `hwc/crates/hwc-compiler/src/conversions/profile_conversion.rs` - Extract and set strategy

### CLI
14. `hwc/crates/hwc-cli/src/commands/build_cmd/validation/utils.rs` - Use space.technology_strategy
15. `hwc/crates/hwc-cli/src/commands/build_cmd/validation/mod.rs` - Update call sites
16. `hwc/crates/hwc-cli/src/commands/build_cmd/alignment.rs` - Update call sites
17. `hwc/crates/hwc-cli/src/commands/build_cmd/validation/drc.rs` - Use strategy for checks
18. `hwc/crates/hwc-cli/src/commands/drc.rs` - Use strategy for standalone DRC

### Tests
19. All test files that create `HardwareSpace` instances
20. New integration tests for strategy propagation

---

## Benefits After Refactoring

1. **Single Source of Truth**: Technology is determined once from PDK
2. **No Scattered Conditionals**: All behavior encapsulated in strategy enum
3. **Type Safety**: Can't forget to check technology type
4. **Easier to Extend**: Adding new technology (e.g., "MEMS") is straightforward
5. **Better Testing**: Can mock/override strategy for tests
6. **Clearer Intent**: Code explicitly shows "this is PCB behavior" vs "this is ASIC behavior"
7. **Performance**: No repeated checks of `min_annular_ring_nm > 0`

---

## Timeline Estimate

- **Phase 1-2** (Core types + HardwareSpace): 2-3 hours
- **Phase 3** (Eliminate conditionals): 4-6 hours
- **Phase 4** (Update call sites): 3-4 hours
- **Phase 5** (Testing): 2-3 hours
- **Total**: 11-16 hours of focused refactoring

---

## Risk Assessment

**Low Risk Areas**:
- Core type definition (new code, no breaking changes to behavior)
- Adding field to HardwareSpace (compile-time checked)

**Medium Risk Areas**:
- Router subsystem (complex, but well-tested)
- Validation subsystem (touches connectivity logic)

**High Risk Areas**:
- Any code that manually calculates pad sizes or expansions
- Export systems (Gerber/GDSII) that may have implicit assumptions

**Mitigation**:
- Run full test suite after each phase
- Keep existing integration tests passing
- Add logging to verify strategy is used correctly

---

## Post-Refactoring Validation

Run this checklist after refactoring:

```bash
# 1. Build everything
cargo build --all

# 2. Run all tests
cargo test --all

# 3. Test ASIC design (should use ASIC strategy)
cargo run --bin hwc -- build tests/ASIC-Minimal/cmos_inverter.hw

# 4. Test PCB design (should use PCB strategy)
cargo run --bin hwc -- build tests/PCB/simple_board.hw

# 5. Check no hardcoded technology assumptions
rg "min_annular_ring_nm\s*[><=]" --type rust

# 6. Verify strategy is used everywhere
rg "technology_strategy" --type rust

# 7. Run clippy
cargo clippy --all

# 8. Check for TODO/FIXME comments
rg "TODO|FIXME" --type rust
```

---

## Success Criteria

- [ ] No more `if min_annular_ring_nm > 0` conditionals in codebase
- [ ] `HardwareSpace` has `technology_strategy` field set during compilation
- [ ] All router calls use `space.technology_strategy` methods
- [ ] All validation calls use `space.technology_strategy` methods
- [ ] All DRC calls conditionally apply rules based on strategy
- [ ] All existing tests pass
- [ ] New unit tests for `TechnologyStrategy` pass
- [ ] Integration test for ASIC passes (cmos_inverter.hw)
- [ ] Integration test for PCB passes (if available)
- [ ] `cargo clippy` reports no warnings
- [ ] Code is cleaner and more maintainable

