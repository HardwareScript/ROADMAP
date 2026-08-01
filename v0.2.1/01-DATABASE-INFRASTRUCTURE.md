# Phase 1: Database Infrastructure (6-8 hours)

**Status**: Not Started  
**Priority**: CRITICAL (Foundation for entire v0.2.1)  
**Dependencies**: None  
**Blocks**: All other phases  

---

## Overview

This phase creates the three core databases that replace all hardcoded assumptions:

1. **LayerConnectionDatabase** - Exact connection points for routing
2. **RoutingLayerDatabase** - Exact Z elevations per layer
3. **ViaLayerMappingDatabase** - Via-to-layer connection specs
4. **TechnologyStrategy** refactoring - Centralize PCB/ASIC behavior

**Critical Rule**: These databases are built ONCE during compilation from PDK data and queried everywhere else. No defaults, no fallbacks.

---

## Step 1.1: Move TechnologyStrategy to hwc-types (2 hours)

### File: `hwc/crates/hwc-types/src/lib.rs`

**Action**: Add TechnologyStrategy enum to shared types

```rust
/// Technology-specific fabrication strategy.
#[derive(Debug, Clone, Copy, PartialEq, Eq, Hash, serde::Serialize, serde::Deserialize)]
pub enum TechnologyStrategy {
    Pcb,
    Asic,
}

impl TechnologyStrategy {
    #[inline]
    pub const fn from_annular_ring(min_annular_ring_nm: i64) -> Self {
        if min_annular_ring_nm > 0 { Self::Pcb } else { Self::Asic }
    }

    #[inline]
    pub const fn contact_expansion(&self, min_annular_ring_nm: i64) -> i64 {
        match self { Self::Pcb => min_annular_ring_nm, Self::Asic => 0 }
    }

    #[inline]
    pub const fn port_escape_clearance(&self, trace_width_nm: i64, min_clearance_nm: i64) -> i64 {
        match self { Self::Pcb => (trace_width_nm / 2) + min_clearance_nm, Self::Asic => min_clearance_nm }
    }

    #[inline]
    pub const fn name(&self) -> &'static str {
        match self { Self::Pcb => "PCB", Self::Asic => "ASIC" }
    }

    #[inline]
    pub const fn is_pcb(&self) -> bool { matches!(self, Self::Pcb) }

    #[inline]
    pub const fn is_asic(&self) -> bool { matches!(self, Self::Asic) }
}
```

### Tasks:
- [ ] Add TechnologyStrategy to hwc-types/src/lib.rs
- [ ] Add unit tests (see test template below)
- [ ] Update Cargo.toml dependencies
- [ ] Remove old geometry_router/technology_strategy.rs
- [ ] Update all imports to use hwc_types::TechnologyStrategy


### Unit Test Template:

```rust
#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn test_pcb_strategy() {
        let strategy = TechnologyStrategy::from_annular_ring(150_000);
        assert_eq!(strategy, TechnologyStrategy::Pcb);
        assert_eq!(strategy.contact_expansion(150_000), 150_000);
        assert_eq!(strategy.port_escape_clearance(200_000, 150_000), 250_000);
    }

    #[test]
    fn test_asic_strategy() {
        let strategy = TechnologyStrategy::from_annular_ring(0);
        assert_eq!(strategy, TechnologyStrategy::Asic);
        assert_eq!(strategy.contact_expansion(0), 0);
        assert_eq!(strategy.port_escape_clearance(200, 0), 0);
    }
}
```

---

## Step 1.2: Create LayerConnectionDatabase (2 hours)

### File: `hwc/crates/hwc-engine/src/layer_connection_database.rs`

**Purpose**: Track exact connection points for every routable entity

```rust
use rustc_hash::FxHashMap;
use compact_str::CompactString;

#[derive(Debug, Clone, Copy)]
pub struct RoutingConnectionPoint {
    pub entity_id: EntityId,
    pub layer_id: LayerId,
    pub z_elevation: i64,
    pub position_2d: (i64, i64),
    pub connection_type: ConnectionType,
}

#[derive(Debug, Clone, Copy)]
pub enum ConnectionType {
    ViaTop { via_bottom: i64, via_top: i64 },
    ViaBottom { via_bottom: i64, via_top: i64 },
    PourSurface { z: i64 },
    PadSurface { z: i64 },
}

pub struct LayerConnectionDatabase {
    connections: FxHashMap<EntityId, Vec<RoutingConnectionPoint>>,
    layer_registry: FxHashMap<CompactString, LayerId>,
    layer_elevations: FxHashMap<LayerId, (i64, i64)>,
}

impl LayerConnectionDatabase {
    pub fn new() -> Self {
        Self {
            connections: FxHashMap::default(),
            layer_registry: FxHashMap::default(),
            layer_elevations: FxHashMap::default(),
        }
    }
