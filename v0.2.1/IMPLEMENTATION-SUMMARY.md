# HardwareScript v0.2.1 Implementation Summary

**Version**: v0.2.1  
**Status**: Ready for Implementation  
**Estimated Time**: 48-63 hours (6-8 working days)  

---

## What We're Building

HardwareScript v0.2.1 eliminates ALL hardcoded assumptions, defaults, and fallbacks by implementing a **database-driven architecture** where:

1. **Everything queries the database** - No more guessing via Z coordinates
2. **Everything fails loudly** - Missing data = compiler error with hints
3. **Single source of truth** - PDK/profile data flows through databases
4. **Context-aware parsing** - Lexer tokens pruned, prepositions parsed contextually
5. **Comptime anchor arithmetic** - Mathematical expressions over entity properties

---

## The Three Core Problems We're Solving

### Problem 1: Via Connections Are Guessed, Not Looked Up

**Current (BAD)**:
```rust
let middle_z_nm = (contact_bbox.min.z + contact_bbox.max.z) / 2;  // WRONG!
```

**New (GOOD)**:
```rust
let connection = space.layer_connection_db.get_connection_point(entity_id, "metal1")?;
let z = connection.z_elevation;  // FROM DATABASE
```

### Problem 2: Lexer Token Pollution Prevents Name Freedom

**Current (BAD)**:
```rust
#[token("above")] Above,  // Can't use "above" as a net name!
```

**New (GOOD)**:
```rust
Token::Identifier("above")  // Parsed contextually in align: blocks
```

### Problem 3: Spatial Math Requires Hardcoded Keywords

**Current (BAD)**:
```rust
// Need new keyword for every geometric idea
center_between: [Pad_A, Pad_B]  // Doesn't exist!
```

**New (GOOD)**:
```rust
// Use comptime anchor arithmetic
at: [x: (Pad_A.center_x + Pad_B.center_x) / 2, y: Pad_A.center_y]
```

---


## Architecture Changes

### New Core Components

```
hwc-types/
  └── TechnologyStrategy enum (moved from geometry_router)
  
hwc-engine/
  ├── layer_connection_database.rs (NEW)
  ├── routing_layer_database.rs (NEW)
  └── via_layer_mapping_database.rs (NEW)

hwc-parser/
  ├── token.rs (22+ tokens REMOVED)
  └── parser/context.rs (NEW: contextual identifier matching)

hwc-compiler/
  ├── anchor_arithmetic/ (NEW)
  │   ├── expression_evaluator.rs
  │   ├── dependency_dag.rs
  │   └── anchor_properties.rs
  └── ir/compilation/
      └── database_integration.rs (NEW)
```

### Database Schema

#### LayerConnectionDatabase
```rust
EntityId → Vec<RoutingConnectionPoint>
  where RoutingConnectionPoint {
    layer_id, z_elevation, position_2d, connection_type
  }
```

#### RoutingLayerDatabase
```rust
LayerName → RoutingLayer {
  routing_z,  // FROM STACKUP, NO DEFAULT
  z_bottom, z_top, is_routable
}
```

#### ViaLayerMappingDatabase
```rust
(FromMaterial, ToMaterial) → ViaConnection {
  bottom_layer, bottom_connection_z,
  top_layer, top_connection_z
}
```

---

## Implementation Phases (Detailed Breakdown)

### Phase 0: Preparation (2-3 hours)
**Goal**: Understand current codebase and plan changes

**Tasks**:
- [ ] Read authoritative spec completely
- [ ] Audit codebase for hardcoded patterns:
  - `(min.z + max.z) / 2`
  - `.unwrap_or(DEFAULT)`
  - `.or_else(|| fallback)`
  - `if min_annular_ring_nm > 0`
- [ ] Document all files needing changes
- [ ] Create test files for validation

**Search Commands**:
```bash
# Find hardcoded Z calculations
rg "\(.*\.z.*\+.*\.z.*\)\s*/\s*2" --type rust

# Find fallback chains
rg "unwrap_or\(|or_else\(" --type rust

# Find scattered technology checks
rg "min_annular_ring_nm.*[><=]" --type rust
```

---


### Phase 1: Database Infrastructure (6-8 hours)
**Goal**: Create the three core databases

#### Step 1.1: TechnologyStrategy to hwc-types (1.5 hours)
- [ ] Move enum to `hwc-types/src/lib.rs`
- [ ] Add methods: `from_annular_ring()`, `contact_expansion()`, `port_escape_clearance()`
- [ ] Write unit tests
- [ ] Update all imports
- [ ] Remove old `geometry_router/technology_strategy.rs`

#### Step 1.2: LayerConnectionDatabase (2 hours)
- [ ] Create `hwc-engine/src/layer_connection_database.rs`
- [ ] Define `RoutingConnectionPoint` struct
- [ ] Define `ConnectionType` enum
- [ ] Implement `register_via()` method
- [ ] Implement `get_connection_point()` method
- [ ] Implement `validate()` method
- [ ] Write unit tests

#### Step 1.3: RoutingLayerDatabase (2 hours)
- [ ] Create `hwc-engine/src/routing_layer_database.rs`
- [ ] Define `RoutingLayer` struct
- [ ] Implement `from_stackup()` builder
- [ ] Implement `get_routing_z()` query (NO FALLBACK!)
- [ ] Add validation for non-conductive layers
- [ ] Write unit tests

#### Step 1.4: ViaLayerMappingDatabase (1.5 hours)
- [ ] Create `hwc-engine/src/via_layer_mapping_database.rs`
- [ ] Define `ViaConnection` struct
- [ ] Implement `from_bridge_rules_and_stackup()` builder
- [ ] Implement query methods
- [ ] Add validation for missing materials
- [ ] Write unit tests

#### Step 1.5: Integrate into HardwareSpace (1 hour)
- [ ] Add three database fields to `HardwareSpace`
- [ ] Add `technology_strategy: TechnologyStrategy` field
- [ ] Update `HardwareSpace::new()` signature
- [ ] Build databases during space creation
- [ ] Update all call sites

**Validation**: Run `cargo test hwc-engine` - all unit tests pass

---

### Phase 2: Lexer Token Pruning (4-6 hours)
**Goal**: Remove 22+ hardcoded tokens from Logos lexer

#### Tokens to Remove:
```rust
// Prepositions → Identifier
RightOf, LeftOf, Above, Below

// Origin codes → Identifier  
TopLeft, BottomLeft, TopRight, BottomRight

// Loop state → Deleted (use i * pitch instead)
Last
```

#### Step 2.1: Update Token Enum (2 hours)
- [ ] Remove tokens from `hwc-parser/src/lexer/token.rs`
- [ ] Update `Token::Display` implementation
- [ ] Remove `#[token("...")]` annotations
- [ ] Verify `Identifier` regex still works

#### Step 2.2: Verify Tokenizer (1 hour)
- [ ] Run tokenizer tests
- [ ] Manually test: `above`, `below`, `right_of`, `tl`, `last`
- [ ] Verify they now tokenize as `Identifier`

#### Step 2.3: Update Documentation (1 hour)
- [ ] Update token documentation
- [ ] Update grammar files
- [ ] Note breaking changes

**Validation**: Run `cargo test hwc-parser` - tokenizer tests pass

---

