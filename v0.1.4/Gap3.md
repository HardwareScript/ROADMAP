# Hardware Script Compiler - Advanced Architecture Evolution (GAP3)

**Version:** v0.1.4+  
**Document Type:** Advanced Compiler Architecture & Universal Design Capabilities  
**Status:** Planning Document for v0.1.4.2+ (Parallel Routing) and v0.3.0+ (Behavioral Synthesis)  
**Last Updated:** March 2026

---

## Overview

This document describes the advanced architectural evolution required to transform Hardware Script from a basic compiler into a universal hardware design system capable of:

1. **Parallel Compilation** - Multi-threaded routing with domain isolation (v0.1.4.2)
2. **Behavioral Synthesis** - Logic synthesis through AST macro expansion (v0.3.0)
3. **Physics-Aware Compilation** - Voxel-based physics calculations (v0.3.0)
4. **Universal Design** - From PCBs to SoCs without external tools (v0.3.0)

These features represent the evolution from "basic compiler" to "universal hardware design platform."

---

## PART 1: Hierarchical Parallel Deterministic Routing

**Target Version:** v0.1.4.2  
**Prerequisites:** FxHashMap ✅, Morton encoding, i64 coordinates, module flattening  
**Performance Goal:** 10-100x speedup on multi-core systems

### The Challenge

To route a billion voxels fast without breaking determinism, we use **Domain Isolation**. If two threads cannot mathematically touch the same voxel, there are no race conditions, no locks, and zero butterfly effects.

Because Hardware Script enforces **Logical/Physical Duality** (`define module` vs `define space` with `layout`), you already have the perfect natural boundaries for this.

### The First Principle: The "Glass Box" Rule

When a module is mapped to physical space via a `layout` block, the compiler calculates its 3D Bounding Box.

**Rule 1:** A child thread assigned to route Module A is trapped inside its Glass Box. It cannot read or write voxels outside its boundaries.

**Rule 2:** The child thread must route internal components to the module's "Interface Pins" (which are placed flush on the glass of the bounding box).

**Rule 3:** The Main Thread waits for all Glass Boxes to finish, then routes the global connections between the Glass Boxes.


### Rust Implementation Blueprint

#### 1. The Domain Structure

```rust
pub struct RoutingDomain {
    pub domain_id: String,           // e.g., "MainDSP.ALU_Core"
    pub bounding_box: BoundingBox,   // The physical 3D "Glass Box"
    pub internal_nets: Vec<NetId>,   // Routes entirely inside this module
    pub interface_pins: Vec<PinId>,  // Pins on the edge connecting to the outside
    pub local_grid: FxHashMap<u64, u32>, // Chunked voxel map relative to the box origin
}

pub struct BoundingBox {
    pub min: Point3D,  // Bottom-left-front corner (i64 nanometers)
    pub max: Point3D,  // Top-right-back corner (i64 nanometers)
}

pub struct RoutedDomain {
    pub id: String,
    pub box_offset: Point3D,              // For coordinate translation during assembly
    pub routes: Vec<Route>,               // All internal routes
    pub grid_chunk: FxHashMap<u64, u32>,  // Occupied voxels (Morton-encoded)
}
```

#### 2. The 3-Phase Execution Pipeline

**Phase 1: Partitioning (Single Threaded, Fast)**

The `ConstraintManager` scans the AST and carves the board into `RoutingDomain` chunks based on module instantiations.

```rust
impl ConstraintManager {
    pub fn partition_into_domains(&self, space: &SpaceDefinition) -> Vec<RoutingDomain> {
        let mut domains = Vec::new();
        
        // For each module instantiation in the space
        for placement in &space.placements {
            if let PlacementType::Module(module_ref) = &placement.placement_type {
                // Calculate bounding box from layout block
                let bbox = self.calculate_module_bounding_box(module_ref, &placement.position);
                
                // Identify internal nets (both pins inside this module)
                let internal_nets = self.find_internal_nets(module_ref);
                
                // Identify interface pins (connect to outside world)
                let interface_pins = self.find_interface_pins(module_ref);
                
                // Create isolated voxel grid for this domain
                let local_grid = FxHashMap::default();
                
                domains.push(RoutingDomain {
                    domain_id: format!("{}.{}", space.name, placement.name),
                    bounding_box: bbox,
                    internal_nets,
                    interface_pins,
                    local_grid,
                });
            }
        }
        
        domains
    }
}
```

**Phase 2: Local Parallel Routing (Multi-Threaded via Rayon)**

We spawn threads using Rayon. Because each `RoutingDomain` has its own isolated `local_grid` (FxHashMap), they require **zero Mutexes or RwLocks**.

```rust
use rayon::prelude::*;

impl ParallelRouter {
    pub fn route_domains(domains: Vec<RoutingDomain>) -> Vec<RoutedDomain> {
        domains.into_par_iter().map(|mut domain| {
            // Thread isolates here. It runs A* strictly within local_grid.
            // Output is deterministic because the input grid is isolated.
            let local_routes = Self::route_internal_nets(&mut domain);
            
            RoutedDomain {
                id: domain.domain_id,
                box_offset: domain.bounding_box.min, // For later assembly
                routes: local_routes,
                grid_chunk: domain.local_grid,
            }
        }).collect()
    }
    
    fn route_internal_nets(domain: &mut RoutingDomain) -> Vec<Route> {
        let mut routes = Vec::new();
        
        // Create local A* router that only sees this domain's grid
        let mut router = AStarRouter::new(&domain.local_grid, &domain.bounding_box);
        
        // Route all internal nets
        for net_id in &domain.internal_nets {
            let net = domain.get_net(net_id);
            
            // Convert global coordinates to local (relative to bounding box)
            let local_start = net.start - domain.bounding_box.min;
            let local_end = net.end - domain.bounding_box.min;
            
            // Route within local coordinate space
            if let Ok(path) = router.route(local_start, local_end) {
                routes.push(Route {
                    net_id: *net_id,
                    waypoints: path,
                });
                
                // Mark voxels as occupied in local grid
                router.mark_path_occupied(&path, net_id);
            }
        }
        
        routes
    }
}
```

**Phase 3: Assembly & Global Routing (Single Threaded, Deterministic)**

The main thread merges all the `grid_chunk` FxHashMaps into the global grid by adding their X/Y/Z offsets. Then, it routes the global buses between the domains (e.g., from Domain A's interface pin to Domain B's interface pin).

```rust
impl ParallelRouter {
    pub fn assemble_and_route_global(
        routed_domains: Vec<RoutedDomain>,
        global_nets: Vec<NetId>,
    ) -> GlobalRoutingResult {
        let mut global_grid = FxHashMap::default();
        
        // Step 1: Merge all domain grids into global grid
        for domain in &routed_domains {
            for (morton_code, material_id) in &domain.grid_chunk {
                // Decode Morton code to local coordinates
                let local_pos = morton_decode(*morton_code);
                
                // Translate to global coordinates
                let global_pos = Point3D {
                    x: local_pos.x + domain.box_offset.x,
                    y: local_pos.y + domain.box_offset.y,
                    z: local_pos.z + domain.box_offset.z,
                };
                
                // Encode to global Morton code
                let global_morton = morton_encode(global_pos);
                
                // Insert into global grid
                global_grid.insert(global_morton, *material_id);
            }
        }
        
        // Step 2: Route global nets between domains
        let mut global_router = AStarRouter::new(&global_grid, &global_bounds);
        let mut global_routes = Vec::new();
        
        for net_id in global_nets {
            let net = self.get_net(&net_id);
            
            // These nets cross domain boundaries
            if let Ok(path) = global_router.route(net.start, net.end) {
                global_routes.push(Route {
                    net_id,
                    waypoints: path,
                });
                
                global_router.mark_path_occupied(&path, &net_id);
            }
        }
        
        GlobalRoutingResult {
            grid: global_grid,
            domain_routes: routed_domains,
            global_routes,
        }
    }
}
```

### Why This Achieves Perfect Parallelism

1. **Zero Locks:** Each thread owns its `FxHashMap` exclusively. No `Mutex`, no `RwLock`, no atomic operations.

2. **Deterministic Output:** Because Phase 2 threads cannot interfere with each other, each domain produces the exact same routes every time.

3. **Cache Locality:** Morton encoding ensures that spatially close voxels have numerically close keys, maximizing CPU cache hits.

4. **Scalability:** On an 8-core CPU, you can route 8 modules simultaneously. On a 64-core server, 64 modules in parallel.

### Performance Expectations

**Test Board:** 1000-net board with 10 modules, 10,000×10,000×10 voxel grid

| Implementation | Compilation Time | Speedup |
|----------------|------------------|---------|
| Single-threaded A* | 30 minutes | 1x |
| Parallel (4 cores) | 8 minutes | 3.75x |
| Parallel (8 cores) | 4 minutes | 7.5x |
| Parallel (16 cores) | 2 minutes | 15x |

**Why not linear scaling?** Phase 1 (partitioning) and Phase 3 (assembly + global routing) are single-threaded. Amdahl's Law applies.

### Implementation Requirements

1. **Add Rayon Dependency:**
   ```toml
   [dependencies]
   rayon = "1.8"
   ```

2. **Implement Bounding Box Calculation:**
   - Parse `layout` blocks to determine module physical boundaries
   - Calculate min/max coordinates for each module instantiation

3. **Implement Net Classification:**
   - Classify nets as "internal" (both pins in same module) or "global" (cross module boundaries)
   - Build interface pin list for each module

4. **Implement Coordinate Translation:**
   - Convert between global and local coordinate spaces
   - Handle Morton encoding/decoding with offsets

5. **Update A* Router:**
   - Accept bounding box constraints
   - Prevent routing outside domain boundaries
   - Support both local and global coordinate spaces

### Test Cases

**Test 1: Domain Isolation**
```hw
define space "TestBoard":
    dimensions: 100mm by 100mm by 2mm
    grid: 1000 by 1000 by 2
    
    define module "ModuleA":
        pins: In, Out
        add Resistor named R1
        route In to R1.Pin1
        route R1.Pin2 to Out
    
    define module "ModuleB":
        pins: In, Out
        add Capacitor named C1
        route In to C1.Pin1
        route C1.Pin2 to Out
    
    layout ModuleA at [x:10mm, y:10mm, z:0]
    layout ModuleB at [x:50mm, y:10mm, z:0]
    
    # Global route between modules
    route ModuleA.Out to ModuleB.In

# Expected: ModuleA and ModuleB route in parallel, then global route connects them
```

**Test 2: Nested Modules**
```hw
define module "SubModule":
    pins: A, B
    add Resistor named R1
    route A to R1.Pin1
    route R1.Pin2 to B

define module "ParentModule":
    pins: In, Out
    add SubModule named Sub1
    add SubModule named Sub2
    route In to Sub1.A
    route Sub1.B to Sub2.A
    route Sub2.B to Out

# Expected: Sub1 and Sub2 route in parallel within ParentModule's domain
```

**Test 3: Performance Benchmark**
```hw
# Create 16 identical modules
for i in 0..15:
    define module "Module[i]":
        pins: In, Out
        # 50 internal components and routes
        for j in 0..49:
            add Resistor named R[j]
            route R[j].Pin1 to R[j].Pin2
    
    layout Module[i] at [x: (i%4)*25mm, y: (i/4)*25mm, z:0]

# Expected: All 16 modules route in parallel (16-core CPU = 16x speedup)
```

---

## PART 2: Behavioral Unrolling & Voxel Physics

**Target Version:** v0.3.0  
**Prerequisites:** Parallel routing, complete module system, physics engine foundation  
**Goal:** Replace external EDA tools (Verilog synthesizers, SPICE simulators)

### The Ultimate First-Principles Breakthrough

By keeping the compiler "dumb," we achieve universal power. The compiler doesn't need to know what a CPU or an SoC is. It only needs to know two things: **AST Macro Expansion (Unrolling)** and **Voxel Algebra (Physics)**.

Because Hardware Script draws inspiration from Ruby, we can make this incredibly elegant. We use Ruby's philosophy of "developer happiness" and expressive blocks to handle the complexity natively.

### Behavioral Unrolling (Replacing Verilog/Logic Synthesis)

**The First Principle:** Logic synthesis is just pattern matching. When the user types `+`, the compiler shouldn't magically know how to build an adder. Instead, `+` is just syntactic sugar for a macro that unrolls into atomic logic gates (NAND, NOR, XOR).

We introduce the `behavior:` block, which uses Ruby-like expressive syntax.

#### The User's Code (Intent)

```hw
import "tsmc_5nm_pdk.hw" as Silicon

define module "ALU_Core":
    pins: 
        A[64], B[64]
        Sum[64]
        Clock
        
    behavior:
        # Ruby-like event block
        on Clock.rising_edge do
            Sum = A + B
        end
```

#### How the "Dumb" Compiler Unrolls This

The compiler parses `Sum = A + B`. It doesn't call an external synthesizer. It simply triggers a built-in AST expansion pass:

1. **Operator Overload Lookup:** The compiler looks at the imported standard library (`tsmc_5nm_pdk.hw`) for the definition of the `+` operator.

2. **AST Substitution:** It finds that `+` maps to a `define module "RippleCarryAdder"`. 

3. **Comptime Unrolling:** The compiler silently replaces `Sum = A + B` with the contents of that module.

Behind the scenes, the AST expands into pure Hardware Script geometry:

```hw
# What the compiler generates internally (Unrolled AST)
for i in 0..63:
    add Silicon.XOR_Gate named HalfAdd[i]
    add Silicon.AND_Gate named Carry[i]
    
    route A[i] to HalfAdd[i].In1
    route B[i] to HalfAdd[i].In2
    route HalfAdd[i].Out to Sum[i]
    # ... routing continues for carry chain
```

**Why this is brilliant:** The compiler remains 100% dumb. It didn't "synthesize" an SoC. It just read a `behavior:` block, found the matching standard library macro, and unrolled it into a `for` loop of geometric components. You just bypassed millions of dollars of proprietary EDA software using pure text substitution.

### Implementation Requirements

1. **Add `behavior:` Block Parser:**
   ```rust
   pub struct BehaviorBlock {
       pub events: Vec<BehaviorEvent>,
   }
   
   pub enum BehaviorEvent {
       OnRisingEdge { signal: String, statements: Vec<BehaviorStatement> },
       OnFallingEdge { signal: String, statements: Vec<BehaviorStatement> },
       Always { statements: Vec<BehaviorStatement> },
   }
   
   pub enum BehaviorStatement {
       Assignment { target: String, expression: Expression },
       Conditional { condition: Expression, then_block: Vec<BehaviorStatement>, else_block: Option<Vec<BehaviorStatement>> },
   }
   ```

2. **Implement Operator Overloading:**
   ```rust
   pub struct OperatorDefinition {
       pub operator: String,  // "+", "-", "*", "/", "&", "|", "^"
       pub module_name: String,  // "RippleCarryAdder", "Multiplier", etc.
       pub bit_width: Option<usize>,  // For parameterized operators
   }
   ```

3. **AST Macro Expansion Pass:**
   ```rust
   impl BehaviorSynthesizer {
       pub fn expand_behavior_block(
           &self,
           behavior: &BehaviorBlock,
           module: &ModuleDefinition,
       ) -> Result<Vec<Placement>, SynthesisError> {
           let mut placements = Vec::new();
           
           for event in &behavior.events {
               match event {
                   BehaviorEvent::OnRisingEdge { signal, statements } => {
                       // Expand each statement into component placements
                       for stmt in statements {
                           placements.extend(self.expand_statement(stmt)?);
                       }
                   }
                   // ... handle other event types
               }
           }
           
           Ok(placements)
       }
       
       fn expand_statement(&self, stmt: &BehaviorStatement) -> Result<Vec<Placement>, SynthesisError> {
           match stmt {
               BehaviorStatement::Assignment { target, expression } => {
                   // Parse expression tree
                   // Look up operator definitions
                   // Instantiate corresponding modules
                   // Generate routing
                   self.expand_expression(target, expression)
               }
               // ... handle conditionals, loops, etc.
           }
       }
   }
   ```

4. **Standard Library Operator Definitions:**

Create `hwc/stdlib/logic/operators.hw`:

```hw
# 64-bit Ripple Carry Adder
define module "Add64":
    pins: A[64], B[64], Sum[64], CarryOut
    
    # Internal carry chain
    for i in 0..63:
        add XOR_Gate named XorAB[i]
        add XOR_Gate named XorSum[i]
        add AND_Gate named AndAB[i]
        add AND_Gate named AndCarry[i]
        add OR_Gate named OrCarry[i]
        
        route A[i] to XorAB[i].In1
        route B[i] to XorAB[i].In2
        
        if i == 0:
            route XorAB[i].Out to Sum[i]
        else:
            route XorAB[i].Out to XorSum[i].In1
            route Carry[i-1] to XorSum[i].In2
            route XorSum[i].Out to Sum[i]
        
        # Carry logic
        route A[i] to AndAB[i].In1
        route B[i] to AndAB[i].In2
        # ... (full carry chain logic)

# Register operator overload
operator "+" for 64-bit = Add64
```

### Voxel Physics (Replacing Static Timing Analysis & Thermal Sims)

**The First Principle:** Physics isn't magic; it's just algebra applied to 3D space.

We don't need external simulation tools. Because your routing engine writes every trace into a 3D sparse `FxHashMap`, the compiler possesses the mathematically perfect dimensions of every wire on the chip.

We expand Layer 4 (`hwc-physics`) to iterate over the voxels and apply raw physics formulas.

#### The User's Code (Constraints)

```hw
define profile "M_Series_Performance":
    physics_limits:
        max_delay: 250ps        # Target: 4.0 GHz clock speed
        max_voltage_drop: 5%    # Power delivery limit
        max_temp_rise: 40C      # Thermal limit
```

#### How the "Dumb" Physics Engine Validates It

After the A* router finishes drawing the voxels, Layer 4 wakes up. It doesn't know it's analyzing an Apple SoC. It just reads the `FxHashMap` and applies basic high-school physics:

**1. Calculate RC Delay (Time)**

The compiler picks a routed net (e.g., from an XOR gate to a Memory cell).

- **Length (L):** It counts the number of voxels in the net. `Length = voxel_count * voxel_size_nm`.
- **Area (A):** It checks the profile trace width and thickness.
- **Resistance (R):** It reads `materials.hw` to find the resistivity (ρ) of Copper. It calculates `R = ρ × (L / A)`.
- **Capacitance (C):** It calculates the distance (d) from the trace voxels to the nearest Ground Plane voxels. It reads the dielectric permittivity (ε) from `materials.hw` and calculates `C = ε × (A / d)`.
- **Delay:** `Time = R × C`.

If the calculated Time is `260ps`, and the profile says `max_delay: 250ps`, the compiler throws an error:

```
Error [P10]: Trace too long. RC Delay exceeds 250ps.
  --> Net: XOR_Gate_42.Out to Memory_Cell_15.In
  --> Calculated: 260ps
  --> Maximum: 250ps
  --> Suggestion: Reduce trace length or insert buffer
```

**2. Calculate Electromigration / Thermal (Energy)**

The compiler checks power nets.

- **Current Density:** It divides the total current by the voxel Area (A).
- If the current density exceeds the `max_current_density` defined in `materials.hw` for that specific metal, it throws an error:

```
Error [P12]: Electromigration risk. Voxel area too small for current.
  --> Net: VDD_Core
  --> Current: 50A
  --> Trace width: 10µm
  --> Current density: 5000 A/mm²
  --> Maximum: 3000 A/mm²
  --> Suggestion: Increase trace width to 17µm
```

#### The "Auto-Fix" (Buffer Insertion)

If you want to take it one step further into "Hero" territory, the compiler can auto-fix physics errors just like it auto-routes.

If an RC Delay is too high (trace is too long), the compiler simply cuts the net in half in the `FxHashMap`, inserts a `Repeater_Buffer` component from the standard library, and connects them. The signal is boosted, the delay resets, and the 4GHz constraint is met.

### Implementation Requirements

1. **Physics Engine Structure:**

```rust
pub struct PhysicsEngine {
    pub material_db: MaterialDatabase,
    pub profile: ProfileDefinition,
}

impl PhysicsEngine {
    pub fn validate_timing(&self, grid: &FxHashMap<u64, u32>, nets: &[Net]) -> Vec<PhysicsViolation> {
        let mut violations = Vec::new();
        
        for net in nets {
            // Extract trace geometry from voxel grid
            let trace_voxels = self.extract_trace_voxels(grid, net);
            
            // Calculate physical properties
            let length = self.calculate_trace_length(&trace_voxels);
            let resistance = self.calculate_resistance(&trace_voxels, length);
            let capacitance = self.calculate_capacitance(&trace_voxels, grid);
            
            // Calculate RC delay
            let delay = resistance * capacitance;
            
            // Check against profile limits
            if let Some(max_delay) = self.profile.physics_limits.max_delay {
                if delay > max_delay {
                    violations.push(PhysicsViolation::TimingViolation {
                        net_id: net.id,
                        calculated_delay: delay,
                        max_delay,
                        suggestion: self.suggest_timing_fix(delay, max_delay),
                    });
                }
            }
        }
        
        violations
    }
    
    fn calculate_resistance(&self, voxels: &[Point3D], length: i64) -> f64 {
        // Get material properties
        let material = self.material_db.get_material(voxels[0].material_id);
        let resistivity = material.resistivity; // Ω·m
        
        // Get trace cross-sectional area from profile
        let width = self.profile.trace.min_width;
        let thickness = self.profile.trace.thickness;
        let area = width * thickness; // m²
        
        // R = ρ × (L / A)
        resistivity * (length as f64 / area as f64)
    }
    
    fn calculate_capacitance(&self, voxels: &[Point3D], grid: &FxHashMap<u64, u32>) -> f64 {
        // Find nearest ground plane
        let distance_to_plane = self.find_nearest_plane_distance(voxels, grid);
        
        // Get dielectric properties
        let dielectric = self.material_db.get_dielectric();
        let permittivity = dielectric.relative_permittivity * EPSILON_0;
        
        // Get trace area
        let width = self.profile.trace.min_width;
        let length = voxels.len() as f64 * self.profile.voxel_size;
        let area = width * length;
        
        // C = ε × (A / d)
        permittivity * (area / distance_to_plane as f64)
    }
}
```

2. **Material Database Extensions:**

Update `materials.hw` to include electrical properties:

```hw
define material "Copper":
    category: Conductor
    color: #B87333
    resistivity: 1.68e-8  # Ω·m at 20°C
    thermal_conductivity: 401  # W/(m·K)
    max_current_density: 3000  # A/mm²
    temperature_coefficient: 0.00393  # per °C

define material "FR4":
    category: Insulator
    color: #2E7D32
    relative_permittivity: 4.5
    loss_tangent: 0.02
    thermal_conductivity: 0.3  # W/(m·K)
```

3. **Auto-Fix Buffer Insertion:**

```rust
impl PhysicsEngine {
    pub fn auto_fix_timing_violation(
        &self,
        grid: &mut FxHashMap<u64, u32>,
        violation: &TimingViolation,
    ) -> Result<Fix, FixError> {
        // Calculate how many buffers needed
        let num_buffers = (violation.calculated_delay / violation.max_delay).ceil() as usize;
        
        // Find optimal buffer insertion points
        let insertion_points = self.find_buffer_insertion_points(
            grid,
            &violation.net_id,
            num_buffers,
        );
        
        // Insert buffers from standard library
        for point in insertion_points {
            self.insert_buffer(grid, point, &violation.net_id)?;
        }
        
        Ok(Fix::BuffersInserted { count: num_buffers })
    }
}
```

---

## PART 3: Universal Compiler Philosophy (SoC Design from First Principles)

**Target Version:** v0.3.0  
**Prerequisites:** Behavioral synthesis, physics engine, parallel routing  
**Goal:** Design custom silicon chips without external EDA tools

### The Architectural Trap (And How to Avoid It)

You have just caught the exact architectural trap that destroys most ambitious software projects.

If we hardcode SoC-specific logic into the compiler, we destroy its universality. The compiler must remain **"dumb."** It should not know what a "PCB" or a "System on a Chip" or a "Quantum Qubit" is.

According to your original vision: **Hardware Script declares reality.** Reality is just 3D space, populated by materials, which obey physical limits. That is all the compiler should understand.

### The Real Problem: The Pipeline is Missing "Transformation Passes"

Your data structures (Sparse Voxel Grid, `materials.hw`, `profiles.hw`) are already perfect. They can handle the scale and the physics constraints.

The only reason people use external EDA tools (like Verilog synthesizers or Static Timing Analyzers) is to **transform** human intent into geometry, and then **validate** that geometry against physics.

To eliminate external EDA tools, Hardware Script doesn't need to become "smart." It just needs a pipeline of **Generic Transformation Passes**—exactly like how the LLVM compiler works for software.

### The Scale Problem (Solved via "AST Unrolling")

**The Problem:** You cannot type 100 billion `add Transistor` commands by hand. You need to write `behavior: Sum = A + B` and have the system build the geometry.

**The First-Principles Solution:**

The compiler doesn't need to know what an "ALU" or a "CPU" is. We just upgrade **Pass 2 (Comptime Unrolling)**.

Right now, your compiler can unroll a `for` loop:
```hw
for i in 0..64: add Transistor
```

We simply extend this to unroll **behavioral logic**. When the compiler sees `Sum = A + B`, it treats it exactly like a macro:

1. It looks up a standard library file (e.g., `@std/logic_gates.hw`).
2. It mathematically translates `A + B` into a graph of XOR and AND gates.
3. It unrolls those gates into the AST as if the user had typed `add XOR_Gate` 1,000 times.

**Why this keeps the compiler dumb:** The compiler is just doing text/AST substitution. It doesn't know it's designing an M4 chip. It's just expanding a behavioral macro into atomic components, and then passing those components to your existing voxel engine.

### The Time & Energy Problem (Solved via "Generic Physics Solvers")

**The Problem:** Electrons take time to travel (RC delay) and generate heat/degrade materials (Electromigration).

**The First-Principles Solution:**

You already solved this in theory: *"Every trace, every copper has the limits you can declare for the energy."*

We do not hardcode "Static Timing Analysis" or "SoC Thermal Rules" into the core routing engine. Instead, we expand **Layer 4 (`hwc-physics`)** into a set of generic, modular physics solvers.

Reality works on simple equations:
- **Time (RC Delay):** Resistance × Capacitance
- **Energy (Heat):** I²R (Current squared × Resistance)

**How the "Dumb" Physics Engine works:**

1. The routing engine finishes drawing 3D voxels. It is done.
2. The `hwc-physics` engine steps in. It looks at the 3D grid.
3. It sees a continuous line of voxels. It checks `materials.hw` and sees "Material = Copper, Resistivity = X".
4. It calculates the exact Resistance and Capacitance based *purely* on the 3D geometry and the material text file.
5. It checks the user's `profile.hw` (e.g., `max_delay: 250ps`, `max_current_density: 30A/mm²`).
6. If the math violates the profile, it throws an error: `Error [P16]: Trace from A to B takes 260ps. Maximum allowed is 250ps.`

**Why this keeps the compiler dumb:** The physics engine doesn't know it's analyzing a 4GHz Apple M4 processor or a 10kHz audio amplifier. It is literally just applying high-school physics equations (Geometry × Material Properties) and checking if the result exceeds the number the user typed in the `.hw` file.

### The Definitive Architecture: How Hardware Script Replaces All EDA Tools

To ensure a user can go from "Zero to Hero" (designing an SoC from scratch without Verilog, Synopsys, or Cadence), the architecture remains exactly the 5-Layer pipeline you already designed, we just ensure the layers are feature-complete:

**Layer 1: Intent (`.hw` files)**
User declares reality. They declare materials, profiles, and logic (`Sum = A + B`).

**Layer 2: Logic Synthesis (AST Expansion)**
The compiler acts as a native synthesizer. It translates behavioral math (`+`, `-`, `*`) into atomic component placements (`add NAND`, `add NOR`) using standard library definitions. *This replaces external Verilog synthesizers.*

**Layer 3: The Voxel Engine (Spatial Layout)**
The Hierarchical Parallel Router places components into the 3D sparse `FxHashMap` and routes them. *This replaces external Place & Route tools.*

**Layer 4: The Physics Engine (Validation)**
The generic solvers run over the 3D voxels. They calculate Resistance, Capacitance, Thermal Density, and Signal Delay using the properties in `materials.hw`. *This replaces external Static Timing Analyzers (STA) and Thermal Simulators.*

**Layer 5: Export**
The voxel grid is sliced and exported to GDSII (for chips) or Gerber (for PCBs).

### The Complete "Zero to Hero" Pipeline in Action

Here is what it looks like for a user building a custom Silicon chip from scratch, using zero external tools:

```hw
import "tsmc_5nm.hw" as TSMC  # Contains material properties and standard cells

define space "MyCustomSoC":
    dimensions: 10mm by 10mm by 0.1mm
    grid: 100000 by 100000 by 15  # 5nm voxel resolution
    profile: TSMC.HighPerformance
    
    constraints:
        target_clock: 4GHz
    
    # Define a 64-bit Core
    define module "ProcessingCore":
        pins: Clk, Data[64], Result[64]
        
        behavior:
            on Clk.rising_edge do
                Result = (Data * 2) + 0xFF
            end
    
    # Instantiate the core
    add ProcessingCore named Core1
    
    # Layout (Glass Box Domain chunking for parallel routing)
    layout Core1:
        at [x: 1mm, y: 1mm, z: 1]
```

**What happens behind the scenes:**

1. **Parser:** Reads the `.hw` file, builds AST
2. **Behavioral Synthesis:** Expands `(Data * 2) + 0xFF` into:
   - 64 multiplier modules (shift left by 1)
   - 64 adder modules (ripple carry adder)
   - Thousands of XOR, AND, OR gates
3. **Module Flattening:** Recursively expands all modules into atomic components
4. **Domain Partitioning:** Creates glass box for `Core1` module
5. **Parallel Routing:** Routes all internal nets using Rayon threads
6. **Physics Validation:** Checks RC delay, current density, thermal limits
7. **Auto-Fix:** Inserts buffers if timing violations detected
8. **Export:** Generates GDSII file for TSMC 5nm fabrication

**Result:** A manufacturable silicon chip design, created entirely within Hardware Script, with zero external tools.

### Why This Is Unstoppable

1. **It is entirely self-contained:** From the Ruby-like `behavior:` block down to the 5nm Voxel Grid, everything stays inside Hardware Script.

2. **It is massively parallel:** The `layout` block creates a Bounding Box. The `behavior` is unrolled inside that box, and Rayon threads route the voxels simultaneously without touching other cores.

3. **It is deterministic:** Because the compiler is "dumb"—it just substitutes text, counts voxels, and multiplies numbers by constants from `materials.hw`—the exact same code will produce the exact same GDSII file every single time, down to the atomic nanometer.

4. **It is universal:** The same compiler binary can design:
   - A blinking LED board (2 components, 2 nets)
   - A Mac motherboard (10,000 components, 10,000 nets)
   - An Apple M4 SoC (100 billion transistors)
   - A quantum computing qubit array (exotic materials, cryogenic constraints)

By realizing that **Logic Synthesis is just AST Macro Expansion** and **Static Timing Analysis is just Voxel Algebra**, you have completely eliminated the need for the fragmented legacy EDA industry. Hardware Script covers reality from end to end.

---

## Implementation Roadmap

### Phase 1: Parallel Routing (v0.1.4.2)

**Duration:** 3-4 weeks  
**Prerequisites:** FxHashMap ✅, Morton encoding, i64 coordinates, module flattening

1. **Week 1: Domain Partitioning**
   - Implement bounding box calculation from layout blocks
   - Implement net classification (internal vs global)
   - Build interface pin detection
   - Test: Partition simple board into 2-3 domains

2. **Week 2: Parallel Routing Core**
   - Integrate Rayon dependency
   - Implement isolated domain routing
   - Implement coordinate translation (global ↔ local)
   - Test: Route 2 domains in parallel, verify isolation

3. **Week 3: Assembly & Global Routing**
   - Implement grid merging with Morton offset translation
   - Implement global net routing
   - Test: Complete 3-phase pipeline on moderate board

4. **Week 4: Optimization & Testing**
   - Benchmark performance on 4, 8, 16 core systems
   - Test nested modules (domains within domains)
   - Verify determinism (100 runs produce identical output)
   - Document performance characteristics

**Success Criteria:**
- ✅ 1000-net board compiles 5-10x faster on 8-core CPU
- ✅ Output is deterministic across all runs
- ✅ No race conditions or deadlocks
- ✅ Memory usage scales linearly with board complexity

### Phase 2: Behavioral Synthesis (v0.3.0)

**Duration:** 6-8 weeks  
**Prerequisites:** Parallel routing, complete module system

1. **Weeks 1-2: Parser Extensions**
   - Add `behavior:` block parsing
   - Implement event syntax (on rising_edge, on falling_edge, always)
   - Add behavioral statement parsing (assignments, conditionals)
   - Test: Parse complex behavioral blocks

2. **Weeks 3-4: Operator Overloading**
   - Design operator definition syntax
   - Implement operator lookup system
   - Create standard library operator definitions
   - Test: Basic arithmetic operators (+, -, *, /)

3. **Weeks 5-6: AST Macro Expansion**
   - Implement behavioral statement expansion
   - Implement expression tree traversal
   - Implement module instantiation from operators
   - Test: Expand simple behavioral blocks to components

4. **Weeks 7-8: Standard Library & Integration**
   - Create comprehensive operator library (arithmetic, logic, comparison)
   - Implement parameterized operators (bit-width aware)
   - Integrate with module flattening pass
   - Test: Compile complete ALU from behavioral description

**Success Criteria:**
- ✅ `Sum = A + B` expands to ripple carry adder
- ✅ `Result = (A * B) + C` expands to multiplier + adder
- ✅ Behavioral blocks integrate seamlessly with manual component placement
- ✅ Standard library covers all common operations

### Phase 3: Physics Engine (v0.3.0)

**Duration:** 4-6 weeks  
**Prerequisites:** Behavioral synthesis, complete routing

1. **Weeks 1-2: Material Database Extensions**
   - Add electrical properties to materials (resistivity, permittivity)
   - Add thermal properties (conductivity, specific heat)
   - Update material parser
   - Test: Load materials with full property sets

2. **Weeks 3-4: Physics Calculations**
   - Implement RC delay calculation
   - Implement current density calculation
   - Implement thermal rise calculation
   - Test: Calculate properties for simple traces

3. **Weeks 5-6: Validation & Auto-Fix**
   - Implement constraint checking against profile limits
   - Implement buffer insertion for timing fixes
   - Implement trace widening for current fixes
   - Test: Auto-fix timing and current violations

**Success Criteria:**
- ✅ Physics engine detects timing violations (RC delay > max)
- ✅ Physics engine detects current violations (density > max)
- ✅ Auto-fix successfully resolves violations
- ✅ Physics calculations match SPICE simulation results (±5%)

### Phase 4: Integration & SoC Capability (v0.3.0)

**Duration:** 2-3 weeks  
**Prerequisites:** All Phase 1-3 features complete

1. **Week 1: End-to-End Testing**
   - Test complete pipeline on simple SoC design
   - Verify behavioral synthesis → routing → physics → export
   - Benchmark compilation time and memory usage
   - Test: Compile 64-bit ALU with timing constraints

2. **Week 2: Standard Library Expansion**
   - Add common digital logic modules (multiplexers, decoders, registers)
   - Add memory modules (SRAM, register files)
   - Add I/O modules (UART, SPI, I2C)
   - Test: Build simple microcontroller from standard library

3. **Week 3: Documentation & Examples**
   - Write SoC design tutorial
   - Create example designs (CPU core, DSP, memory controller)
   - Document behavioral synthesis syntax
   - Document physics engine capabilities

**Success Criteria:**
- ✅ User can design custom SoC without external tools
- ✅ Compilation time is reasonable (<10 min for moderate SoC)
- ✅ Physics validation catches real design errors
- ✅ Exported GDSII is manufacturable

**Total Estimated Effort:** 15-21 weeks (4-5 months)

---

## Success Criteria Summary

### v0.1.4.2 (Parallel Routing)
- ✅ Multi-threaded routing with domain isolation
- ✅ 5-10x speedup on 8-core systems
- ✅ Deterministic output (no race conditions)
- ✅ Handles 1000+ net boards efficiently

### v0.3.0 (Universal Compiler)
- ✅ Behavioral synthesis replaces Verilog
- ✅ Physics engine replaces SPICE/STA
- ✅ Complete SoC design without external tools
- ✅ Designs range from PCBs to silicon chips
- ✅ Compilation is deterministic and reproducible
- ✅ Standard library covers common digital logic

---

## Conclusion

These three parts transform Hardware Script from a basic compiler into a universal hardware design platform:

1. **Hierarchical Parallel Routing** enables efficient compilation of large designs through domain isolation and multi-threading.

2. **Behavioral Synthesis** eliminates the need for external logic synthesis tools by treating operators as AST macros that expand into geometric components.

3. **Universal Compiler Philosophy** keeps the core "dumb" while achieving universal power through generic transformation passes and physics-based validation.

Together, they provide the foundation for designing anything from simple PCBs to complex SoCs, all within a single, deterministic, version-controlled workflow.

**The Vision Realized:** Hardware Script becomes the LLVM of hardware design—a universal intermediate representation that can target any physical substrate, from printed circuit boards to silicon chips to quantum computing arrays, all through the same elegant, Ruby-inspired syntax.
