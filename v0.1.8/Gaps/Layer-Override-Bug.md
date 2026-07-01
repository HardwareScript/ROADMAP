# Native Systems Architecture for Scalable Physical Synthesis

**Version**: 0.1.8 Roadmap  
**Status**: Design Document  
**Priority**: CRITICAL - Blocks ASIC synthesis scaling

## Executive Summary

The current compiler uses sequential, voxel-based coordinate shifts that cause **layer override bugs** and prevent scaling to dense, multi-layer ASIC designs. This document specifies three native systems required to fix these architectural issues and enable correct physical synthesis.

---

## 1. The Layer Override Bug

### Current Behavior (BROKEN)

```hw
component NMOS_Transistor:
    layout:
        add pour(Aluminum) named Gate_Pad on layer: metal1:
            boundary: [x: 500nm, y: 375nm] to [x: 750nm, y: 625nm]

space CMOS_Inverter:
    # This placement causes ALL internal pours to shift to layer "active"
    add NMOS_Transistor named M1 at [x: 0.875um, y: 0.9um] on layer: active
```

**What Happens:**
1. Unroller processes component with Z-offset from `on layer: active` (100nm)
2. Internal pour declared as `on layer: metal1` (should be at 1355nm)
3. Unroller **adds** parent Z-offset to internal coordinates: `1355nm + 100nm = 1455nm`
4. This drifted coordinate no longer aligns with metal1's Z-interval
5. StackupManager resolves it to Layer 1 (active) instead of Layer 5 (metal1)
6. Result: "Aluminum (Layer 1)" forbidden junction error

### Debug Evidence

```
🔍 [VIA RESOLVER] bridge_layers called for net 'VOUT'
   Layer 1 to Layer 5
   Found 7 elements on layer 1, 3 elements on layer 5
   
🔍 [VIA RESOLVER] Attempting to bridge:
   From: Aluminum (Layer 1)  ← WRONG! Aluminum should not exist on active layer
   To: Aluminum (Layer 5)
```

**Severity**: CRITICAL - Makes complex component-based ASIC design impossible

---

## 2. Native System A: Zero-Stamping Scene Graph

### Current Implementation (Legacy)

```rust
// parametric_unroller/unroll.rs (BROKEN)
fn unroll_component(component: &ComponentPlacement) {
    let z_offset = resolve_placement_layer_z(component.elevation);
    
    for internal_pour in component.internal_pours {
        // ❌ WRONG: Additive Z transformation
        internal_pour.z += z_offset;
        place_geometry(internal_pour);
    }
}
```

### Required Implementation (Zero-Stamping)

```rust
/// System A: Template-Instance Model with Stackup-Relative Resolution
struct ComponentStamp {
    /// Analytical templates in local coordinate space
    internal_geometry: Vec<RelativeGeometry>,
}

struct RelativeGeometry {
    material: MaterialId,
    layer_name: LayerName,  // e.g., "metal1", not absolute Z
    local_bounds: BoundingBox2D,  // XY only, no Z
}

struct ComponentInstance {
    stamp_id: ComponentStampId,
    transform: TransformMatrix,  // XY translation + rotation
    placement_layer: LayerName,  // Reference layer for anchor
}

fn resolve_instance_geometry(
    instance: &ComponentInstance,
    stamp: &ComponentStamp,
    stackup: &StackupManager,
) -> Vec<PlacedGeometry> {
    stamp.internal_geometry.iter().map(|geom| {
        // ✅ CORRECT: Query stackup for absolute Z of declared layer
        let absolute_z = stackup.get_layer_z_interval(&geom.layer_name);
        
        PlacedGeometry {
            material: geom.material,
            bounds: BoundingBox3D {
                xy: instance.transform.apply(geom.local_bounds),
                z: absolute_z,  // Always correct, no drift
            }
        }
    }).collect()
}
```

### Benefits

1. **Zero Coordinate Drift**: Layers resolved by name, not arithmetic
2. **Template Reuse**: Component compiled once, instantiated N times
3. **Memory Efficiency**: Lightweight instances vs. duplicated geometry
4. **Correctness**: Impossible to place metal1 on wrong layer

---

## 3. Native System B: Conductive Horizon Via Resolver

### Current Implementation (Sequential)

```rust
// via_resolver/mod.rs (INEFFICIENT)
fn insert_via_stack(from_layer: usize, to_layer: usize) {
    let mut current = from_layer;
    while current < to_layer {
        // ❌ Tries to bridge every layer, including insulators
        let via = find_via(current, current + 1)?;
        current += 1;
    }
}
```

**Problems:**
- Attempts vias through SiO2 (insulator) layers
- No early validation against PDK bridge rules
- Sequential search, doesn't leverage stackup structure

### Required Implementation (Horizon-Based)

```rust
/// System B: Conductive Horizon Mapping with PDK Verification
struct ConductiveHorizon {
    layer_index: usize,
    layer_name: LayerName,
    material: MaterialId,
    z_interval: (i64, i64),
}

impl ViaResolver {
    fn extract_conductive_horizons(
        &self,
        stackup: &StackupManager,
    ) -> Vec<ConductiveHorizon> {
        stackup.ordered_layers()
            .enumerate()
            .filter(|(_, layer)| stackup.is_layer_conductive(layer))
            .map(|(idx, layer)| ConductiveHorizon {
                layer_index: idx,
                layer_name: layer.clone(),
                material: stackup.get_material_for_layer(layer),
                z_interval: stackup.get_layer_z_interval(layer),
            })
            .collect()
    }
    
    fn bridge_conductive_horizons(
        &self,
        from: &ConductiveHorizon,
        to: &ConductiveHorizon,
        bridge_table: &BridgeTable,
    ) -> Result<ViaStack, BridgeError> {
        // ✅ CORRECT: Validate BEFORE attempting via insertion
        let bridge = bridge_table.lookup(
            &from.material.name,
            &to.material.name,
        ).ok_or(BridgeError::ForbiddenJunction {
            from: from.material.name.clone(),
            to: to.material.name.clone(),
        })?;
        
        // Only bridge between conductive layers with approved bridges
        Ok(ViaStack {
            from_z: from.z_interval.0,
            to_z: to.z_interval.1,
            bridge_material: bridge.interface_material.clone(),
            fill_material: bridge.fill_material.clone(),
        })
    }
}
```

### Benefits

1. **Skips Insulators**: Only considers electrically active layers
2. **Early PDK Validation**: Fails at compile-time, not runtime
3. **Direct Horizon Mapping**: No layer-by-layer iteration
4. **Correct Material Matching**: Uses actual element materials, not stackup defaults

---

## 4. Native System C: Continuous Spatial Indexing

### Current Implementation (Voxel Grid)

```rust
// LEGACY: Voxel-based collision detection
struct VoxelGrid {
    cells: HashMap<(i64, i64, i64), Vec<EntityId>>,  // ❌ Memory explosive
    cell_size: i64,
}
```

**Problems:**
- O(N³) memory for 3D grid
- Query performance degrades with density
- Doesn't scale to sub-micron ASIC features

### Required Implementation (R*-Tree + G-Cell Sweep)

```rust
/// System C: Logarithmic Spatial Index with Vectorized DRC
use rstar::RTree;

struct EntityGraph {
    /// ✅ R*-Tree for O(log N) queries
    spatial_index: RTree<IndexedEntity>,
}

struct IndexedEntity {
    bbox: BoundingBox3D,
    net_id: NetId,
    material_id: MaterialId,
    entity_type: EntityType,
}

impl EntityGraph {
    fn query_region(&self, bbox: &BoundingBox3D) -> impl Iterator<Item = &IndexedEntity> {
        self.spatial_index.locate_in_envelope_intersecting(&bbox.to_envelope())
    }
}

/// G-Cell Sweep DRC (replaces voxel-based checks)
struct GCellSweepDRC {
    g_cell_size: i64,  // e.g., 1000nm = 1μm
}

impl GCellSweepDRC {
    fn run_drc(&self, space: &HardwareSpace) -> Vec<DRCViolation> {
        let g_cells = self.partition_space(space.dimensions);
        
        g_cells.par_iter().flat_map(|g_cell| {
            // Query all entities in this G-cell
            let entities: Vec<_> = space.entity_graph
                .query_region(&g_cell.bbox)
                .collect();
            
            // ✅ Single-pass line-sweep within G-cell
            self.sweep_line_drc(&entities, g_cell)
        }).collect()
    }
    
    fn sweep_line_drc(
        &self,
        entities: &[&IndexedEntity],
        g_cell: &GCell,
    ) -> Vec<DRCViolation> {
        // Event-driven sweep: O(N log N) per G-cell
        let mut events = self.create_sweep_events(entities);
        events.sort_by_key(|e| e.x_coordinate);
        
        let mut active_set = BTreeSet::new();
        let mut violations = Vec::new();
        
        for event in events {
            match event.type {
                EventType::Enter => {
                    // Check against all active entities
                    for active in &active_set {
                        if let Some(violation) = self.check_drc_rule(event.entity, active) {
                            violations.push(violation);
                        }
                    }
                    active_set.insert(event.entity);
                }
                EventType::Exit => {
                    active_set.remove(&event.entity);
                }
            }
        }
        
        violations
    }
}
```

### Benefits

1. **Logarithmic Queries**: O(log N) lookups vs. O(N³) voxel access
2. **Scalable Memory**: O(N) entities vs. O(N³) grid cells
3. **Parallel G-Cell DRC**: Independent cells process concurrently
4. **Vector-First**: Native 2D/3D geometry, no discretization

---

## 5. Implementation Roadmap

### Phase 1: System A - Zero-Stamping (v0.2.0)

**Goal**: Fix layer override bug, enable correct component placement

**Tasks:**
1. ✅ Create `ComponentStamp` and `ComponentInstance` types
2. ✅ Modify `parametric_unroller` to build stamps, not place geometry
3. ✅ Implement `resolve_instance_geometry` with stackup queries
4. ✅ Update `space_builder` to instantiate components lazily
5. ✅ Test: CMOS inverter compiles without "Aluminum (Layer 1)" error

**Success Criteria:**
- [ ] Internal pour layers always resolve to correct stackup Z-intervals
- [ ] Component can be placed `on layer: X` without affecting internal geometry layers
- [ ] Memory usage decreases for repeated component instances

### Phase 2: System B - Conductive Horizon Resolver (v0.2.1)

**Goal**: Optimize via insertion, validate PDK rules early

**Tasks:**
1. ✅ Implement `extract_conductive_horizons` from stackup
2. ✅ Refactor `ViaResolver` to bridge horizons, not sequential layers
3. ✅ Add early PDK bridge validation before via insertion
4. ✅ Remove legacy sequential via search

**Success Criteria:**
- [ ] Via resolver skips insulator layers (SiO2, gate_oxide)
- [ ] Forbidden junctions caught at compile-time with clear error
- [ ] Via insertion time reduces by 50%+ on multi-layer designs

### Phase 3: System C - Spatial Indexing (v0.2.2)

**Goal**: Scale DRC to dense ASIC layouts

**Tasks:**
1. ✅ Integrate `rstar` R*-Tree into `EntityGraph`
2. ✅ Implement G-cell partitioning (1μm cells for ASIC, 1mm for PCB)
3. ✅ Build sweep-line DRC within G-cells
4. ✅ Parallelize G-cell DRC processing
5. ✅ Remove legacy voxel grid

**Success Criteria:**
- [ ] DRC runs in <5s for 10K entity ASIC design
- [ ] Memory usage <1GB for 100K entity design
- [ ] Parallel DRC scales linearly with CPU cores

---

## 6. Immediate Next Steps

### Short-term Fix (Today)

Remove verbose debug logging and document the bug:

```rust
// via_resolver/library/mod.rs
- println!("🔍 [VIA LIBRARY] Building via library from bridge rules...");
+ // System A required: Layer override bug prevents correct resolution
```

Update COMPILER-GAP-REPORT.md with findings.

### Medium-term (v0.2.0 Sprint)

Begin System A implementation:
1. Design `ComponentStamp` data structure
2. Refactor `ComponentDefinition` AST → `ComponentStamp` IR
3. Prototype stackup-relative geometry resolution
4. Test with CMOS inverter

---

## 7. References

- **Bug Report**: "Aluminum (Layer 1)" forbidden junction
- **Root Cause**: Z-coordinate additive transformation in `parametric_unroller`
- **Via Library Fix**: Category matching disabled for same-material bridges (✅ completed)
- **Architecture**: Template-instance model inspired by OpenROAD and Magic VLSI

---

**Document Status**: Draft  
**Last Updated**: 2026-06-30  
**Next Review**: Start of v0.2.0 sprint
