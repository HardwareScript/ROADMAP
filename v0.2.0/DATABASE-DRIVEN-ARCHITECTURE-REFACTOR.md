# Database-Driven Architecture Refactor (v0.2.0)

## Executive Summary

**Problem**: The compiler silently succeeds while producing incorrect routing geometry. Root cause: hardcoded assumptions, scattered fallbacks, and missing lookup tables mean there's no single source of truth for critical routing decisions.

**Solution**: Eliminate all defaults, hardcoded values, and scattered conditionals. Replace with proper lookup tables and database queries. Make the system fail loudly when required data is missing.

**Impact**: This is a **breaking architectural change** that will touch most subsystems but will make the codebase maintainable, debuggable, and correct.

---

## Root Cause Analysis

### Problem 1: Via Connection Points Are Guessed, Not Looked Up

**Current Behavior**:
```rust
// contact/mod.rs line 394
let middle_z_nm = (contact_bbox.min.z + contact_bbox.max.z) / 2;
```

Via at Z=400-1250nm connects at Z=825nm (middle), but metal1 layer is at Z=1250-1650nm. **425nm gap!**

**Why This Happened**: No database tracks "where does this via connect to routing layers?"

---

### Problem 2: Layer Stack Exists But Isn't Queried

**Layer Stack Database Exists**:
```
Layer 0: active    Z=0-400nm
Layer 1: gate_oxide Z=400-450nm
Layer 2: poly      Z=450-850nm
Layer 3: d1        Z=850-1250nm
Layer 4: metal1    Z=1250-1650nm  <- Should route here!
```

**But Router Doesn't Use It**: Router calculates positions from bounding boxes instead of querying stackup.

---

### Problem 3: Bridge Rules Generate Vias But Don't Inform Routing

**Bridge Rules**:
```yaml
- from_material: Polysilicon
  to_material: Aluminum
  interface: Titanium_Silicide
```

**Via Library** builds vias from this, but **routing system** doesn't know:
- Which layers the via connects
- At what Z elevations
- Where to snap traces

---

### Problem 4: Silent Success Masks Failures

Build completes with:
```
✅ Physical continuity validation passed
✅ All requested formats exported
```

But traces don't connect! The validator should **FAIL LOUDLY** when:
- Via Z doesn't match routing layer Z
- Trace starts/ends mid-air
- Connection points don't align

---


## Proposed Architecture: Single Source of Truth Databases

### Database 1: Layer Connection Database

**Purpose**: Map every routable entity to its exact layer connections with Z elevations.

**Location**: `hwc-engine/src/routing_database.rs` (extend existing `HierarchicalRoutingDatabase`)

**Schema**:
```rust
/// Exact connection point for routing
#[derive(Debug, Clone, Copy)]
pub struct RoutingConnectionPoint {
    pub entity_id: EntityId,
    pub layer_id: LayerId,
    pub z_elevation: i64,        // Exact Z coordinate
    pub position_2d: (i64, i64), // XY position
    pub connection_type: ConnectionType,
}

#[derive(Debug, Clone, Copy)]
pub enum ConnectionType {
    /// Via top surface (connects to upper layer)
    ViaTop { via_bottom: i64, via_top: i64 },
    /// Via bottom surface (connects to lower layer)
    ViaBottom { via_bottom: i64, via_top: i64 },
    /// Pour surface (single Z plane)
    PourSurface { z: i64 },
    /// Pad surface
    PadSurface { z: i64 },
}

/// Database of all routing connections
pub struct LayerConnectionDatabase {
    /// All connection points indexed by entity
    connections: FxHashMap<EntityId, Vec<RoutingConnectionPoint>>,
    
    /// Layer name to ID mapping
    layer_registry: FxHashMap<CompactString, LayerId>,
    
    /// Layer Z bounds (from stackup)
    layer_elevations: FxHashMap<LayerId, (i64, i64)>, // (bottom, top)
}
```

**API**:
```rust
impl LayerConnectionDatabase {
    /// Register a via's connection points (called during placement)
    pub fn register_via(
        &mut self,
        entity_id: EntityId,
        bottom_layer: &str,
        bottom_z: i64,
        top_layer: &str,
        top_z: i64,
        position_2d: (i64, i64),
    ) -> Result<(), DatabaseError>;
    
    /// Get connection point for routing FROM an entity ON a specific layer
    pub fn get_connection_point(
        &self,
        entity_id: EntityId,
        layer: &str,
    ) -> Result<RoutingConnectionPoint, DatabaseError>;
    
    /// Validate all connections (call before routing)
    pub fn validate(&self) -> Result<(), Vec<ValidationError>>;
}
```

---


### Database 2: Routing Layer Database

**Purpose**: Define exact Z elevations for routing on each layer (derived from stackup).

**Schema**:
```rust
/// Routing surface definition
#[derive(Debug, Clone)]
pub struct RoutingLayer {
    pub id: LayerId,
    pub name: CompactString,
    pub material: MaterialId,
    
    /// Z elevation for centerline routing
    pub routing_z: i64,
    
    /// Full layer bounds (from stackup)
    pub z_bottom: i64,
    pub z_top: i64,
    
    /// Whether this layer supports routing
    pub is_routable: bool,
}

pub struct RoutingLayerDatabase {
    layers: FxHashMap<LayerId, RoutingLayer>,
    layer_names: FxHashMap<CompactString, LayerId>,
}
```

**Derivation Rules**:
```rust
impl RoutingLayerDatabase {
    /// Build from stackup (called once during compilation)
    pub fn from_stackup(
        stackup: &[StackupLayer],
        material_registry: &MaterialRegistry,
    ) -> Self {
        let mut db = Self::new();
        
        for layer in stackup {
            // Only conductive layers are routable
            let is_routable = material_registry
                .get_category(layer.material_id)
                .map(|c| c == MaterialCategory::Conductor)
                .unwrap_or(false);
            
            if is_routable {
                // Routing centerline: prefer bottom of layer for metal
                // (where vias connect)
                let routing_z = layer.z_bottom;
                
                db.register_layer(RoutingLayer {
                    id: LayerId::new(layer.id),
                    name: layer.name.clone(),
                    material: layer.material_id,
                    routing_z,
                    z_bottom: layer.z_bottom,
                    z_top: layer.z_top,
                    is_routable: true,
                });
            }
        }
        
        db
    }
    
    /// Get routing Z for a named layer (REQUIRED - no fallback)
    pub fn get_routing_z(&self, layer_name: &str) -> Result<i64, DatabaseError> {
        self.layer_names
            .get(layer_name)
            .and_then(|id| self.layers.get(id))
            .map(|layer| layer.routing_z)
            .ok_or_else(|| DatabaseError::LayerNotFound {
                layer: layer_name.into(),
            })
    }
}
```

---


### Database 3: Via-to-Layer Mapping Database

**Purpose**: Track which layers each via connects and at what Z.

**Schema**:
```rust
/// Via connection specification (generated from bridge rules + stackup)
#[derive(Debug, Clone)]
pub struct ViaConnection {
    pub via_material: MaterialId,
    pub bottom_layer: LayerId,
    pub bottom_layer_name: CompactString,
    pub bottom_connection_z: i64,
    pub top_layer: LayerId,
    pub top_layer_name: CompactString,
    pub top_connection_z: i64,
}

pub struct ViaLayerMappingDatabase {
    /// Maps (from_material, to_material) -> ViaConnection
    via_specs: FxHashMap<(MaterialId, MaterialId), ViaConnection>,
}
```

**Build Process** (called once during compilation):
```rust
impl ViaLayerMappingDatabase {
    pub fn from_bridge_rules_and_stackup(
        bridge_rules: &[BridgeRule],
        stackup: &[StackupLayer],
        material_registry: &MaterialRegistry,
    ) -> Result<Self, DatabaseError> {
        let mut db = Self::new();
        
        for bridge in bridge_rules {
            let from_mat_id = material_registry
                .get_id(&bridge.from_material)
                .ok_or_else(|| DatabaseError::MaterialNotFound {
                    material: bridge.from_material.clone(),
                })?;
            
            let to_mat_id = material_registry
                .get_id(&bridge.to_material)
                .ok_or_else(|| DatabaseError::MaterialNotFound {
                    material: bridge.to_material.clone(),
                })?;
            
            // Find layers in stackup
            let from_layer = stackup
                .iter()
                .find(|l| l.material_id == from_mat_id)
                .ok_or_else(|| DatabaseError::LayerNotFoundForMaterial {
                    material: bridge.from_material.clone(),
                })?;
            
            let to_layer = stackup
                .iter()
                .find(|l| l.material_id == to_mat_id)
                .ok_or_else(|| DatabaseError::LayerNotFoundForMaterial {
                    material: bridge.to_material.clone(),
                })?;
            
            // Connection points: top of from_layer, bottom of to_layer
            let connection = ViaConnection {
                via_material: material_registry
                    .get_id(&bridge.interface)
                    .ok_or_else(|| DatabaseError::MaterialNotFound {
                        material: bridge.interface.clone(),
                    })?,
                bottom_layer: LayerId::new(from_layer.id),
                bottom_layer_name: from_layer.name.clone(),
                bottom_connection_z: from_layer.z_top,
                top_layer: LayerId::new(to_layer.id),
                top_layer_name: to_layer.name.clone(),
                top_connection_z: to_layer.z_bottom,
            };
            
            db.register_via_spec((from_mat_id, to_mat_id), connection)?;
        }
        
        Ok(db)
    }
}
```

---


## Implementation Plan

### Phase 1: Build the Databases (Infra Layer)

**Timeline**: 3-4 hours

#### Step 1.1: Create `LayerConnectionDatabase`

**File**: `hwc-engine/src/layer_connection_database.rs` (new file)

**Tasks**:
- [ ] Define `RoutingConnectionPoint` struct
- [ ] Define `ConnectionType` enum
- [ ] Implement `LayerConnectionDatabase` with insert/query methods
- [ ] Add validation logic
- [ ] Add unit tests

#### Step 1.2: Create `RoutingLayerDatabase`

**File**: `hwc-engine/src/routing_layer_database.rs` (new file)

**Tasks**:
- [ ] Define `RoutingLayer` struct
- [ ] Implement `from_stackup()` builder
- [ ] Implement `get_routing_z()` query (MUST NOT HAVE FALLBACK)
- [ ] Add validation: fail if non-conductive layer is marked routable
- [ ] Add unit tests

#### Step 1.3: Create `ViaLayerMappingDatabase`

**File**: `hwc-engine/src/via_layer_mapping_database.rs` (new file)

**Tasks**:
- [ ] Define `ViaConnection` struct
- [ ] Implement `from_bridge_rules_and_stackup()` builder
- [ ] Add validation: fail if bridge rule references missing layers
- [ ] Add unit tests

#### Step 1.4: Integrate into `HardwareSpace`

**File**: `hwc-engine/src/space/mod.rs`

**Add fields**:
```rust
pub struct HardwareSpace {
    // ... existing fields ...
    
    /// v0.2.0: Database of all routing connection points
    pub layer_connection_db: LayerConnectionDatabase,
    
    /// v0.2.0: Database of routing layer Z elevations
    pub routing_layer_db: RoutingLayerDatabase,
    
    /// v0.2.0: Database of via-to-layer mappings
    pub via_layer_mapping_db: ViaLayerMappingDatabase,
}
```

**Build during space creation**:
```rust
impl HardwareSpace {
    pub fn new(...) -> Self {
        // Build routing layer database from stackup
        let routing_layer_db = RoutingLayerDatabase::from_stackup(
            &stackup_layers,
            &material_registry,
        ).expect("Failed to build routing layer database");
        
        // Build via-layer mapping from bridge rules + stackup
        let via_layer_mapping_db = ViaLayerMappingDatabase::from_bridge_rules_and_stackup(
            &bridge_rules,
            &stackup_layers,
            &material_registry,
        ).expect("Failed to build via-layer mapping database");
        
        // Layer connection database starts empty, populated during placement
        let layer_connection_db = LayerConnectionDatabase::new();
        
        Self {
            // ... existing fields ...
            layer_connection_db,
            routing_layer_db,
            via_layer_mapping_db,
        }
    }
}
```

---


### Phase 2: Fix Contact Placement to Register Connections

**Timeline**: 2-3 hours

#### Step 2.1: Update `place_contact()` to Register Connections

**File**: `hwc-compiler/src/ir/placement/contact/mod.rs`

**Current (WRONG)**:
```rust
let middle_z_nm = (contact_bbox.min.z + contact_bbox.max.z) / 2;
```

**New (CORRECT)**:
```rust
// Query the via-layer mapping to determine connection layers
let from_layer_id = stackup_manager
    .get_layer_id(&contact.from_elevation)
    .ok_or_else(|| IrError::InvalidLayer {
        layer: format!("{:?}", contact.from_elevation),
    })?;

let to_layer_id = stackup_manager
    .get_layer_id(&contact.to_elevation)
    .ok_or_else(|| IrError::InvalidLayer {
        layer: format!("{:?}", contact.to_elevation),
    })?;

// Get the via connection spec from database
let via_spec = space.via_layer_mapping_db
    .get_via_connection(from_material_id, to_material_id)
    .map_err(|e| IrError::ViaConnectionNotFound {
        from_material: contact.material.clone(),
        to_material: /* determined from bridge rule */,
        hint: "Add a bridge rule in your PDK connecting these materials".into(),
    })?;

// Register BOTH connection points (bottom and top)
space.layer_connection_db.register_via(
    entity_id,
    &via_spec.bottom_layer_name,
    via_spec.bottom_connection_z,  // Top of bottom layer
    &via_spec.top_layer_name,
    via_spec.top_connection_z,     // Bottom of top layer
    (xy_point.x, xy_point.y),
)?;

// Create connection interface at BOTH Z levels
// (not the middle!)
let bottom_interface = create_interface_at_z(
    contact_bbox,
    via_spec.bottom_connection_z,
    origin,
);

let top_interface = create_interface_at_z(
    contact_bbox,
    via_spec.top_connection_z,
    origin,
);

space.entity_graph.register_space_entity_interface(
    format!("{}:bottom", contact.name.base),
    bottom_interface,
);

space.entity_graph.register_space_entity_interface(
    format!("{}:top", contact.name.base),
    top_interface,
);
```

**Validation**: After registration, verify connection points match stackup:
```rust
// Validate via connection points against stackup
space.layer_connection_db.validate_via(entity_id, &space.routing_layer_db)?;
```

---


### Phase 3: Fix Router to Query Databases

**Timeline**: 4-5 hours

#### Step 3.1: Update Route Specification Resolution

**File**: `hwc-compiler/src/ir/routing/automatic/constraints.rs`

**Current**: Route says `layer: metal1` but router doesn't validate or use it properly

**New**: Query the database for exact Z elevation:
```rust
pub fn resolve_route_constraints(
    route: &hwc_parser::Route,
    space: &HardwareSpace,
    // ... other params
) -> Result<ResolvedRouteConstraints, IrError> {
    // Get layer name from route declaration (REQUIRED)
    let layer_name = route.layer
        .as_ref()
        .ok_or_else(|| IrError::MissingRouteParameter {
            parameter: "layer",
            route: format!("{} to {}", route.from, route.to),
            hint: "Every route must explicitly declare which layer to use.\n\
                   Example: route A to B:\n    layer: metal1".into(),
        })?;
    
    // Query routing layer database for exact Z (NO FALLBACK!)
    let routing_z = space.routing_layer_db
        .get_routing_z(layer_name)
        .map_err(|e| IrError::InvalidRoutingLayer {
            layer: layer_name.clone(),
            available_layers: space.routing_layer_db.list_routable_layers(),
            hint: format!("Available routing layers: {}", 
                space.routing_layer_db.list_routable_layers().join(", ")),
        })?;
    
    eprintln!("[ROUTE CONSTRAINTS] Layer '{}' resolves to Z={}nm (from database)", 
        layer_name, routing_z);
    
    Ok(ResolvedRouteConstraints {
        trace_width_nm,
        layer_name: layer_name.clone(),
        routing_z,  // FROM DATABASE, NOT CALCULATED
        escape_stub_nm,
    })
}
```

#### Step 3.2: Update Boundary Resolution to Query Connections

**File**: `hwc-compiler/src/ir/routing/helpers/boundary_resolution.rs`

**Current**: Calculates boundary points from bounding boxes

**New**: Query layer connection database:
```rust
pub fn resolve_route_boundary_points(
    space: &HardwareSpace,
    route: &ResolvedRoute,
    routing_layer: &str,
    trace_width_nm: i64,
) -> Result<(Point3D, Point3D, Normal2D, Normal2D), IrError> {
    
    // Query connection points from database
    let from_connection = space.layer_connection_db
        .get_connection_point(route.from, routing_layer)
        .map_err(|e| IrError::NoConnectionPoint {
            entity: format!("{:?}", route.from),
            layer: routing_layer.into(),
            hint: "This entity does not have a connection point on this layer.\n\
                   Check that your via spans the correct layers.".into(),
        })?;
    
    let to_connection = space.layer_connection_db
        .get_connection_point(route.to, routing_layer)
        .map_err(|e| IrError::NoConnectionPoint {
            entity: format!("{:?}", route.to),
            layer: routing_layer.into(),
            hint: "This entity does not have a connection point on this layer.".into(),
        })?;
    
    // Validate Z matches routing layer
    if from_connection.z_elevation != space.routing_layer_db.get_routing_z(routing_layer)? {
        return Err(IrError::ConnectionZMismatch {
            entity: format!("{:?}", route.from),
            connection_z: from_connection.z_elevation,
            routing_layer_z: space.routing_layer_db.get_routing_z(routing_layer)?,
            hint: "Via connection Z does not match routing layer Z. \
                   This is a compiler bug.".into(),
        });
    }
    
    // Build boundary points from database-provided positions
    let start_point = Point3D::new(
        from_connection.position_2d.0,
        from_connection.position_2d.1,
        from_connection.z_elevation,  // FROM DATABASE
    );
    
    let goal_point = Point3D::new(
        to_connection.position_2d.0,
        to_connection.position_2d.1,
        to_connection.z_elevation,  // FROM DATABASE
    );
    
    // ... calculate normals using obstacle-aware logic ...
    
    Ok((start_point, goal_point, start_normal, goal_normal))
}
```

---


### Phase 4: Add Fail-Fast Validation

**Timeline**: 2-3 hours

#### Step 4.1: Pre-Routing Validation

**File**: `hwc-compiler/src/ir/compilation/routing_phase.rs`

**Add before routing**:
```rust
pub fn execute_routing_phase(
    space: &mut HardwareSpace,
    routes: Vec<ResolvedRoute>,
) -> Result<(), IrError> {
    
    // VALIDATION CHECKPOINT 1: Verify all routing layers are valid
    eprintln!("[VALIDATION] Checking routing layer database...");
    space.routing_layer_db.validate()?;
    
    // VALIDATION CHECKPOINT 2: Verify all vias have connection points
    eprintln!("[VALIDATION] Checking via connection points...");
    space.layer_connection_db.validate()?;
    
    // VALIDATION CHECKPOINT 3: Verify each route can find connection points
    eprintln!("[VALIDATION] Checking route endpoint connections...");
    for route in &routes {
        let layer = route.layer_name.as_str();
        
        // Check FROM endpoint
        space.layer_connection_db
            .get_connection_point(route.from, layer)
            .map_err(|e| IrError::PreRoutingValidationFailed {
                route: format!("{:?} to {:?}", route.from, route.to),
                layer: layer.into(),
                problem: format!("FROM endpoint has no connection on layer '{}'", layer),
                hint: "Check that your via spans this layer, or that your pour is on this layer".into(),
            })?;
        
        // Check TO endpoint
        space.layer_connection_db
            .get_connection_point(route.to, layer)
            .map_err(|e| IrError::PreRoutingValidationFailed {
                route: format!("{:?} to {:?}", route.from, route.to),
                layer: layer.into(),
                problem: format!("TO endpoint has no connection on layer '{}'", layer),
                hint: "Check that your via spans this layer, or that your pour is on this layer".into(),
            })?;
    }
    
    eprintln!("[VALIDATION] ✓ Pre-routing checks passed");
    
    // Now proceed with routing...
    // ...
}
```

#### Step 4.2: Post-Routing Validation

**Add after routing**:
```rust
    // ... routing completed ...
    
    // VALIDATION CHECKPOINT 4: Verify traces connect at correct Z
    eprintln!("[VALIDATION] Checking trace Z elevations...");
    for trace in &space.analytic_routes {
        if let Some((z_min, z_max)) = trace.layer_z_range {
            // Verify all horizontal segments are within layer bounds
            for seg in &trace.segments {
                if seg.start.z == seg.end.z {  // Horizontal
                    let seg_z = seg.start.z;
                    if seg_z < z_min || seg_z > z_max {
                        return Err(IrError::PostRoutingValidationFailed {
                            net: trace.net_name.clone(),
                            problem: format!(
                                "Trace segment at Z={}nm is outside layer bounds {}->{}nm",
                                seg_z, z_min, z_max
                            ),
                            hint: "This indicates a via connection Z mismatch.".into(),
                        });
                    }
                }
            }
        }
    }
    
    eprintln!("[VALIDATION] ✓ Post-routing checks passed");
    
    Ok(())
}
```

---


### Phase 5: Remove Dead Code and Defaults

**Timeline**: 3-4 hours

#### Step 5.1: Remove Hardcoded Z Calculations

**Files to update**:
- `hwc-compiler/src/ir/placement/contact/mod.rs` - Remove `middle_z_nm`
- `hwc-compiler/src/ir/routing/helpers/boundary_resolution.rs` - Remove bbox-based Z calculation
- `hwc-engine/src/geometry_router/router/routing_methods/global_routing.rs` - Remove Z guessing

**Search pattern**: `(min.z + max.z) / 2`

#### Step 5.2: Remove Fallback Chains

**Pattern to eliminate**:
```rust
// BAD - has fallback
let value = calculate_from_bbox()
    .or_else(|| guess_from_material())
    .unwrap_or(DEFAULT_VALUE);

// GOOD - query database or fail
let value = database.get_value(key)
    .map_err(|e| Error::RequiredValueMissing { key, hint })?;
```

**Files to audit**:
- All files with `.unwrap_or(` or `.unwrap_or_else(`
- All files with `.or_else(`
- All files with `match ... { Some(x) => x, None => DEFAULT }`

#### Step 5.3: Remove Unused Connection Interface Code

**Audit**: The `PhysicalInterface` system with `AccessRegions` might be overengineered

**Question**: Do we need obstacle-aware port selection if we have exact connection points from database?

**Possible simplification**:
- Connection database gives exact XYZ
- Router uses that directly
- Remove `AccessRegion` calculation
- Remove boundary port scoring

**Decision point**: Defer to after Phase 4 to see if AccessRegions add value

#### Step 5.4: Remove Technology Strategy Workarounds

**File**: `hwc-engine/src/geometry_router/technology_strategy.rs`

**Current**: Used to calculate contact expansions and escape clearances

**After refactor**: All expansions come from layer connection database

**Action**: Evaluate if `TechnologyStrategy` is still needed or if all behavior is now database-driven

---


## Testing Strategy

### Unit Tests

#### Test 1: Layer Connection Database
```rust
#[test]
fn test_via_registration_and_lookup() {
    let mut db = LayerConnectionDatabase::new();
    
    db.register_via(
        entity_id,
        "active",
        400,  // top of active layer
        "metal1",
        1250, // bottom of metal1 layer
        (650, 1000),
    ).unwrap();
    
    // Should find connection on both layers
    let bottom_conn = db.get_connection_point(entity_id, "active").unwrap();
    assert_eq!(bottom_conn.z_elevation, 400);
    
    let top_conn = db.get_connection_point(entity_id, "metal1").unwrap();
    assert_eq!(top_conn.z_elevation, 1250);
}

#[test]
fn test_missing_connection_fails() {
    let db = LayerConnectionDatabase::new();
    
    let result = db.get_connection_point(entity_id, "poly");
    assert!(result.is_err());
    assert!(matches!(result.unwrap_err(), DatabaseError::NoConnectionPoint { .. }));
}
```

#### Test 2: Routing Layer Database
```rust
#[test]
fn test_routing_z_from_stackup() {
    let stackup = vec![
        StackupLayer {
            name: "active".into(),
            z_bottom: 0,
            z_top: 400,
            material_id: silicon_id,
        },
        StackupLayer {
            name: "metal1".into(),
            z_bottom: 1250,
            z_top: 1650,
            material_id: aluminum_id,
        },
    ];
    
    let db = RoutingLayerDatabase::from_stackup(&stackup, &material_registry);
    
    // Should use bottom of layer as routing Z
    let metal1_z = db.get_routing_z("metal1").unwrap();
    assert_eq!(metal1_z, 1250);
}

#[test]
fn test_routing_z_for_nonexistent_layer_fails() {
    let db = RoutingLayerDatabase::from_stackup(&stackup, &material_registry);
    
    let result = db.get_routing_z("nonexistent");
    assert!(result.is_err());
}
```

### Integration Tests

#### Test 3: PMOS Transistor (Your Failing Case)
```rust
#[test]
fn test_pmos_transistor_routing() {
    let result = compile_and_build("tests/ASIC-Minimal/pmos_transistor.hw");
    
    // Should succeed now with proper via connections
    assert!(result.is_ok());
    
    let space = result.unwrap();
    
    // Verify Via_Source connects at correct Z
    let via_conn = space.layer_connection_db
        .get_connection_point(via_source_id, "metal1")
        .unwrap();
    
    assert_eq!(via_conn.z_elevation, 1250, 
        "Via should connect at bottom of metal1 layer");
    
    // Verify trace routes at correct Z
    let vdd_trace = space.analytic_routes
        .iter()
        .find(|t| t.net_name == "VDD")
        .unwrap();
    
    for seg in &vdd_trace.segments {
        if seg.start.z == seg.end.z {  // Horizontal segment
            assert_eq!(seg.start.z, 1250,
                "Horizontal trace should be at metal1 routing Z");
        }
    }
}
```

#### Test 4: Via Z Mismatch Detection
```rust
#[test]
fn test_via_z_mismatch_fails_validation() {
    // Manually create a space with mismatched via
    let mut space = create_test_space();
    
    // Register via with WRONG Z (simulate old buggy behavior)
    space.layer_connection_db.register_via(
        entity_id,
        "active",
        825,  // WRONG! This is middle of via, not top
        "metal1",
        1250,
        (650, 1000),
    ).unwrap();
    
    // Validation should fail
    let result = space.layer_connection_db.validate(&space.routing_layer_db);
    assert!(result.is_err());
    
    let err = result.unwrap_err();
    assert!(err.to_string().contains("does not match layer elevation"));
}
```

---

