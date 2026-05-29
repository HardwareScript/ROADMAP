# Hardware Script Compiler - Advanced Architectural Gaps (GAP2)

**Version:** v0.1.4  
**Document Type:** Advanced Feature Gaps & Future Architecture  
**Status:** Planning Document for v0.2.0+  
**Last Updated:** March 2026

---

## Overview

This document describes advanced architectural features required for enterprise-grade hardware compilation. These features go beyond basic functionality and address:

1. **Procedural Routing** - Community-extensible routing patterns
2. **Stability & Determinism** - Anti-butterfly effect architecture
3. **Route Caching** - Persistent routing with lockfiles

These features are essential for:
- Large-scale boards (10,000+ nets)
- Collaborative hardware development
- Version control integration (Git)
- Reproducible builds across teams

---

## 1. Custom Routing Libraries: The "Procedural Router"

### Current State: Binary Routing Model

The system currently offers only two extremes:

**100% Manual Routing:**
```hw
route Memory.DataBus to CPU.DataBus:
    path:
        - [x:10, y:20, z:1]
        - [x:15, y:20, z:1]
        - [x:15, y:30, z:1]
        # ... user must specify every waypoint (exhausting)
```

**100% Automatic Routing:**
```hw
route Memory.DataBus to CPU.DataBus
# A* algorithm does whatever it wants (unpredictable)
```

### The Problem

Neither approach is suitable for enterprise hardware:

- **Manual routing** is tedious and error-prone for complex patterns (BGA fan-outs, serpentine buses)
- **Automatic routing** cannot guarantee specific patterns required for high-speed signals
- **No middle ground** for guided routing with mathematical patterns

### The Solution: Procedural Routing Engine

Hardware Script must allow users to define **mathematical path-generation patterns** that feed into the A* algorithm as guided constraints.

#### The Architectural Gap

The language currently lacks a `define strategy` or `define pattern` block. You need a way for community members to write algorithms in `.hw` that output an array of waypoints.

### The `strategy` Keyword

Extend the `route` syntax to accept a parameterized strategy.

#### Example: Future Syntax

**In `@std/routing/patterns.hw`:**
```hw
define strategy "ZigZag_LengthMatch":
    parameters:
        amplitude: Measurement
        pitch: Measurement
        target_length: Measurement
    
    # Internal logic outputs a list of mathematical waypoints
    # based on the start and end coordinates
    algorithm:
        # Calculate straight-line distance
        let direct_distance = distance(start, end)
        
        # Calculate how much extra length is needed
        let extra_length = target_length - direct_distance
        
        # Generate zigzag pattern
        let num_zigs = extra_length / (2 * amplitude)
        
        # Output waypoints
        for i in 0..num_zigs:
            yield [
                x: start.x + (i * pitch),
                y: start.y + (i % 2 == 0 ? amplitude : -amplitude),
                z: start.z
            ]
```

**In `main.hw`:**
```hw
import "ZigZag_LengthMatch" from "@std/routing"

route Memory.DataBus to CPU.DataBus:
    strategy: ZigZag_LengthMatch(
        amplitude: 2mm,
        pitch: 0.5mm,
        target_length: 45mm
    )
    orientation: left_aligned
```

### How It Works in the Compiler

When the compiler sees a `strategy`, it does NOT run the blind A* router. Instead:

1. **Execute Strategy Math:** The strategy's algorithm generates a sequence of "Soft Waypoints"
2. **Feed to A* Router:** These waypoints become constraints for the A* pathfinder
3. **Connect-the-Dots:** The A* router acts as a "connect-the-dots" engine, ensuring DRC rules (clearance, obstacles) are obeyed while strictly following the community-defined pattern

### Implementation Requirements

#### 1. Parser Extensions

Add `define strategy` as a new definition type:

```rust
pub enum DefinitionType {
    Material,
    Profile,
    Component,
    Module,
    Strategy,  // ← New
    // ...
}

pub struct StrategyDefinition {
    pub name: String,
    pub parameters: Vec<StrategyParameter>,
    pub algorithm: StrategyAlgorithm,
}

pub struct StrategyParameter {
    pub name: String,
    pub param_type: ParameterType,
    pub default: Option<Value>,
}
```

#### 2. Strategy Execution Engine

```rust
pub trait RoutingStrategy {
    fn generate_waypoints(
        &self,
        start: Point3D,
        end: Point3D,
        params: &HashMap<String, Value>,
        grid: &SparseVoxelGrid,
    ) -> Result<Vec<Point3D>, StrategyError>;
}
```

#### 3. Standard Library Strategies

Create `hwc/stdlib/routing/patterns.hw` with common patterns:

- `StraightLine` - Direct path (default)
- `ZigZag_LengthMatch` - Serpentine for length matching
- `BGA_Fanout` - Radial escape routing for BGA packages
- `DifferentialPair` - Parallel routing with controlled spacing
- `Trombone` - U-shaped length matching
- `ArcRoute` - Curved paths for RF applications

#### 4. Strategy Validation

Before routing, validate that:
- Strategy parameters are within valid ranges
- Generated waypoints don't violate DRC rules
- Waypoints are reachable from start/end points

### Benefits

1. **Community Extensibility:** Users can publish routing strategies to HPM
2. **Reusability:** Complex patterns become one-line declarations
3. **Predictability:** Designers control the routing style while maintaining DRC compliance
4. **Small Core:** The compiler stays lean; complexity lives in the ecosystem

### Test Cases

**Test 1: Length Matching**
```hw
# Route 8-bit bus with all traces exactly 45mm long
for i in 0..7:
    route DDR.Data[i] to CPU.Data[i]:
        strategy: ZigZag_LengthMatch(
            amplitude: 1.5mm,
            pitch: 0.4mm,
            target_length: 45mm
        )

# Verify: All traces are 45mm ± 0.05mm
```

**Test 2: BGA Fanout**
```hw
# Escape 256 balls from BGA package
route CPU.Ball[0..255] to Breakout.Pin[0..255]:
    strategy: BGA_Fanout(
        escape_angle: 45deg,
        via_distance: 0.5mm,
        layer_transition: 2
    )

# Verify: No trace collisions, all balls routed
```

---

## 2. The Anti-Butterfly Effect: Stability vs. Determinism

### Critical Computer Science Distinction

**Determinism:** Running the exact same code 100 times produces the exact same Gerber file 100 times.
- ✅ **Already Achieved:** By using `VecDeque` instead of `HashSet`

**Stability (Anti-Butterfly Effect):** Changing one line of code results in the minimum possible changes to the output Gerber file.
- ❌ **Not Achieved:** Moving one component can cascade and rewire the entire board

### The Problem

Right now, the router is **deterministic** but NOT **stable**. Because nets are routed sequentially, if moving Component A forces Net 1 to take a slightly different path, it becomes a new obstacle for Net 2, which changes the path for Net 3, creating a cascading avalanche that rewires the entire board.

**Example Scenario:**
```
Initial State:
- Net 1 routes at Y=10
- Net 2 routes at Y=11 (avoids Net 1)
- Net 3 routes at Y=12 (avoids Net 2)

After moving Component A by 1mm:
- Net 1 now routes at Y=11 (new obstacle)
- Net 2 must reroute to Y=13 (avoids new Net 1)
- Net 3 must reroute to Y=15 (avoids new Net 2)
- Net 4-100 all cascade and change

Result: Git diff shows 100 nets changed, but only 1 component moved!
```

### The Solution: Two Architectural Pillars

To achieve Stability, you must implement two new architectural pillars:

---

### Pillar A: Strict Net Ordering (The Routing Hierarchy)

#### The Problem

The order in which nets are fed to the A* router determines who gets the best paths and who gets blocked. If this order changes randomly when a component moves, you get the Butterfly Effect.

#### The Fix: Physics-Based Routing Hierarchy

Before routing begins, the `ConstraintManager` must sort the `NetlistArena` into a strict, immutable hierarchy based on **physics**, not code placement:

```rust
pub enum NetPriority {
    Critical = 0,      // Clocks, resets, high-speed differential pairs
    Power = 1,         // Power rails (if not using planes)
    HighSpeed = 2,     // DDR, PCIe, USB 3.0
    DataBus = 3,       // Parallel data buses
    LowSpeed = 4,      // I2C, UART, SPI
    GPIO = 5,          // General purpose I/O
}

pub struct NetMetadata {
    pub priority: NetPriority,
    pub signal_group: Option<String>,
    pub max_length: Option<i64>,
    pub impedance: Option<f64>,
}
```

#### Routing Order

1. **Critical/High-Speed Nets** (e.g., PCIe, Clocks) — Routed first, shortest path
2. **Power/Ground Nets** (if not using planes) — Routed second, thickest paths
3. **Data Buses** — Routed third, pattern-matched
4. **General Purpose I/O (GPIO)** — Routed last, fills the remaining space

#### Benefits

If you move a GPIO resistor, the Clock and Power nets will **not change at all** because they were routed first. The butterfly effect is **contained**.

#### Implementation

```rust
impl ConstraintManager {
    pub fn sort_nets_by_priority(&mut self) -> Vec<NetId> {
        let mut nets: Vec<(NetId, NetMetadata)> = self.nets
            .iter()
            .map(|(id, net)| (*id, self.analyze_net_priority(net)))
            .collect();
        
        // Sort by priority (ascending), then by net name (for determinism)
        nets.sort_by(|a, b| {
            a.1.priority.cmp(&b.1.priority)
                .then_with(|| a.0.cmp(&b.0))
        });
        
        nets.into_iter().map(|(id, _)| id).collect()
    }
    
    fn analyze_net_priority(&self, net: &Net) -> NetMetadata {
        // Analyze net characteristics to determine priority
        // - Check signal_group for high-speed indicators
        // - Check connected components for power/clock
        // - Default to GPIO for unknown nets
    }
}
```

---

### Pillar B: Persistent Route Caching (The Lockfile)

#### The Concept

This is how modern software handles dependencies (`Cargo.lock`, `package-lock.json`). Hardware Script must do the same for physical traces.

#### The `.hw.routes.lock` File

When the user runs `hwc build`, the compiler should generate a **visible** (not hidden) `.hw.routes.lock` file. This file stores the exact X/Y/Z waypoints of every successful route from the last compile.

**File Format (JSON):**
```json
{
  "version": "0.1.4",
  "board": "Main_Flight_Board",
  "grid": {
    "dimensions": [200, 100, 2],
    "resolution": [1.0, 1.0, 1.0]
  },
  "routes": [
    {
      "net_id": "Net_1",
      "source": "R1.Pin2",
      "destination": "Amp_Tx.RF_IN",
      "waypoints": [
        [20, 50, 1],
        [25, 50, 1],
        [130, 50, 1],
        [130, 35, 1]
      ],
      "length_mm": 115.5,
      "layer_transitions": 0,
      "hash": "a3f5b2c1"
    }
  ]
}
```

#### The New Routing Pipeline

1. **Read `.hw.routes.lock`**
   - Load all previously successful routes

2. **Plop Frozen Routes**
   - Place all routes onto the Voxel Grid as "Frozen" obstacles
   - Mark these voxels as immutable

3. **Check DRC**
   - Did the user's new code (e.g., moving a component) cause a collision with a Frozen route?

4. **Selective Rerouting**
   - **If NO collision:** Keep the old route exactly as it was (Zero Butterfly Effect, Git diff is clean)
   - **If YES collision:** "Rip up" ONLY the colliding net, unfreeze it, and send it to the A* router to find a new path

5. **Write New Lockfile**
   - Update `.hw.routes.lock` with any changed routes
   - Preserve unchanged routes exactly

#### Implementation

```rust
pub struct RouteLockfile {
    pub version: String,
    pub board: String,
    pub grid: GridMetadata,
    pub routes: Vec<LockedRoute>,
}

pub struct LockedRoute {
    pub net_id: String,
    pub source: String,
    pub destination: String,
    pub waypoints: Vec<Point3D>,
    pub length_mm: f64,
    pub layer_transitions: u32,
    pub hash: String,  // Hash of source/dest positions for validation
}

impl Router {
    pub fn route_with_lockfile(
        &mut self,
        lockfile: Option<RouteLockfile>,
    ) -> Result<Vec<Route>, RoutingError> {
        let mut frozen_routes = Vec::new();
        let mut nets_to_route = Vec::new();
        
        if let Some(lock) = lockfile {
            for locked_route in lock.routes {
                // Validate that source/dest haven't moved
                if self.validate_route_endpoints(&locked_route) {
                    // Check for collisions with new components
                    if !self.has_collision(&locked_route) {
                        // Keep the old route
                        frozen_routes.push(locked_route);
                        self.mark_voxels_frozen(&locked_route);
                    } else {
                        // Must reroute this net
                        nets_to_route.push(locked_route.net_id);
                    }
                } else {
                    // Endpoints moved, must reroute
                    nets_to_route.push(locked_route.net_id);
                }
            }
        }
        
        // Route only the nets that need rerouting
        let new_routes = self.route_nets(nets_to_route)?;
        
        // Combine frozen and new routes
        Ok(frozen_routes.into_iter()
            .chain(new_routes)
            .collect())
    }
}
```

#### Git Integration

The `.hw.routes.lock` file should be:
- ✅ **Committed to version control** (like `Cargo.lock`)
- ✅ **Human-readable** (JSON format for easy diffs)
- ✅ **Deterministic** (sorted by net name for consistent diffs)

**Example Git Diff:**
```diff
diff --git a/.hw.routes.lock b/.hw.routes.lock
index a3f5b2c..d4e8f1a 100644
--- a/.hw.routes.lock
+++ b/.hw.routes.lock
@@ -15,7 +15,7 @@
       "waypoints": [
         [20, 50, 1],
         [25, 50, 1],
-        [130, 50, 1],
+        [132, 50, 1],  # Only this net changed!
         [130, 35, 1]
       ],
```

### Benefits of Stability Architecture

1. **Minimal Git Diffs:** Only changed routes appear in version control
2. **Faster Compilation:** Frozen routes skip the A* algorithm entirely
3. **Predictable Behavior:** Moving one component doesn't cascade across the board
4. **Team Collaboration:** Merge conflicts are rare and localized
5. **Incremental Builds:** Only reroute what's necessary

### Test Cases

**Test 1: Component Movement**
```
1. Compile board with 100 nets
2. Move one GPIO resistor by 2mm
3. Recompile
4. Verify: Only 1-3 nets rerouted (not all 100)
5. Check Git diff: Only changed nets in lockfile
```

**Test 2: New Component Addition**
```
1. Compile board with 50 nets
2. Add new component with 5 nets
3. Recompile
4. Verify: Original 50 nets unchanged, only 5 new nets routed
5. Check lockfile: 50 old routes preserved, 5 new routes added
```

**Test 3: Collision Detection**
```
1. Compile board
2. Move component that blocks existing trace
3. Recompile
4. Verify: Blocked trace rerouted, others unchanged
5. Check lockfile: Only colliding net has new waypoints
```

---

## Implementation Roadmap

### Phase 1: Procedural Routing (v0.2.0)

1. Add `define strategy` parser support
2. Implement strategy execution engine
3. Create standard library routing patterns
4. Integrate with A* router as guided constraints
5. Add strategy validation and error handling

**Estimated Effort:** 3-4 weeks

### Phase 2: Routing Hierarchy (v0.2.1)

1. Implement net priority analysis
2. Add physics-based sorting to ConstraintManager
3. Update router to respect priority order
4. Add priority override syntax for manual control
5. Test with complex boards (1000+ nets)

**Estimated Effort:** 2-3 weeks

### Phase 3: Route Lockfile (v0.2.2)

1. Design lockfile format (JSON schema)
2. Implement lockfile serialization/deserialization
3. Add frozen route validation logic
4. Implement selective rerouting
5. Add CLI flags (`--no-lockfile`, `--force-reroute`)
6. Test Git integration and diff quality

**Estimated Effort:** 3-4 weeks

### Phase 4: Integration & Testing (v0.2.3)

1. Combine all three features
2. Test on enterprise boards (Mac motherboard scale)
3. Benchmark compilation speed improvements
4. Measure Git diff quality
5. Document best practices

**Estimated Effort:** 2 weeks

**Total Estimated Effort:** 10-13 weeks

---

## Success Criteria

### Procedural Routing
- ✅ Users can define custom routing strategies in `.hw` files
- ✅ Standard library includes 5+ common patterns
- ✅ Strategies can be published to HPM
- ✅ A* router respects strategy waypoints while maintaining DRC

### Routing Hierarchy
- ✅ Nets are sorted by physics-based priority
- ✅ Moving low-priority components doesn't affect high-priority nets
- ✅ Compilation is deterministic with consistent net ordering

### Route Lockfile
- ✅ `.hw.routes.lock` generated on every successful build
- ✅ Unchanged routes are preserved across compilations
- ✅ Git diffs show only actually changed routes
- ✅ Compilation speed improves by 50-80% on incremental builds
- ✅ Lockfile is human-readable and merge-friendly

---

## Conclusion

These three features transform Hardware Script from a basic compiler into an enterprise-grade EDA tool:

1. **Procedural Routing** enables community-driven innovation
2. **Routing Hierarchy** prevents cascading changes
3. **Route Lockfile** enables Git-based hardware development

Together, they provide the foundation for collaborative, version-controlled hardware design at scale.
