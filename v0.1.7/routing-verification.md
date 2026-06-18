Here is the actionable, step-by-step **Verification Gauntlet and Test Checklist** for the routing system [Unified-2.5D-3D-Routing-and-Placement.md]. 

These tasks are structured as concrete, test-by-test validation steps that you can run against your `hwc` compiler to prove the physical, spatial, and mathematical correctness of your layout engine [Z-AXIS-ABSTRACTION-IMPLEMENTATION.md, Unified-2.5D-3D-Routing-and-Placement.md, Advanced-Routing-Implementation.md].

---

### Task 1: Verification of Lock-File Determinism & Invalidation

This test proves that once a layout is compiled, the mathematical routes are frozen and can be reused instantly unless a physical change invalidates them [Unified-2.5D-3D-Routing-and-Placement.md].

- [ ] **1.1 First-Time Compilation (Lock Generation)**
  *   **Action:** Compile `tests/pcb/test_complex_hybrid_pcb.hw` for the first time [Unified-2.5D-3D-Routing-and-Placement.md].
  *   **Verify:** 
      - The console logs output: `[LOCK] Generating project.routes.lock...` [Unified-2.5D-3D-Routing-and-Placement.md]
      - A file named `tests/pcb/build/PCB_Complex_Space/project.routes.lock` is written to disk [Unified-2.5D-3D-Routing-and-Placement.md].
      - The JSON file contains the explicit nanometer $i64$ waypoints for `NET_MANUAL` and `NET_AUTO` [Unified-2.5D-3D-Routing-and-Placement.md].

- [ ] **1.2 Zero-Search Compile (Lock Hit)**
  *   **Action:** Run the identical build command again immediately [Unified-2.5D-3D-Routing-and-Placement.md].
  *   **Verify:** 
      - The console logs output: `[LOCK] Match found for 'PCB_Complex_Space'. Bypassing A* solver. Loading routes directly from project.routes.lock` [Unified-2.5D-3D-Routing-and-Placement.md].
      - Compilation finishes in under $10\text{ms}$ (instead of $40\text{ms}$), proving the routing engine was bypassed completely [Unified-2.5D-3D-Routing-and-Placement.md].

- [ ] **1.3 Non-Geometric Modification (Preserved Lock)**
  *   **Action:** Modify a non-geometric property inside the `.hw` file (e.g., change `description` in the profile, or change a part number in the metadata) [LANGUAGE-SPEC.md]. Recompile.
  *   **Verify:** 
      - The compiler reads the lock file and bypasses routing [LAZY-REALIZATION-ARCHITECTURE.md].
      - The generated GDSII/DXF outputs are bit-for-bit identical to the first run [Unified-2.5D-3D-Routing-and-Placement.md, Manufacturing-Fidelity-RETHINK.md].

- [ ] **1.4 Geometric Shift (Lock Invalidation)**
  *   **Action:** Shift the position of one connector in the space, for example:
      Change `add Connector named J1_S at [x: 2mm, y: 10mm]` to `[x: 3mm, y: 10mm]` [Z-AXIS-ABSTRACTION-IMPLEMENTATION.md, Unified-2.5D-3D-Routing-and-Placement.md].
  *   **Verify:** 
      - The compiler detects that the component anchors or checksums have changed.
      - The console logs output: `[LOCK] Invalidation detected: Component 'J1_S' has shifted. Discarding lock file and re-running A* pathfinder` [Unified-2.5D-3D-Routing-and-Placement.md].
      - A fresh `.routes.lock` file is generated with updated coordinates [Unified-2.5D-3D-Routing-and-Placement.md].

---

### Task 2: Multi-Layer Barrier-Hop (Auto-Via & Plane-Hopping)

This test proves the auto-router can transition down to the bottom layer, bypass an impenetrable physical barrier on the top layer, and pop back up to the top layer [Z-AXIS-ABSTRACTION-IMPLEMENTATION.md, ADVANCED-ROUTING-IMPLEMENTATION.md].

- [ ] **2.1 Prepare the Barrier Space**
  *   **Action:** Open `tests/pcb/test_complex_hybrid_pcb.hw` and place a physical different-net metal pour (`Top_Blocker`) on `layer: top` directly between the source and destination terminals [Z-AXIS-ABSTRACTION-IMPLEMENTATION.md, Unified-2.5D-3D-Routing-and-Placement.md].

- [ ] **2.2 Execute Auto-Route (`NET_AUTO`)**
  *   **Action:** Compile the file.
  *   **Verify:** 
      - The console outputs: `[PROCESS] Transition L0→L4 at z 0nm→1270000nm` [Z-AXIS-ABSTRACTION-IMPLEMENTATION.md].
      - The unroller places exactly **two** through-hole vias (`AutoVia_NET_AUTO_0_4_1` and `AutoVia_NET_AUTO_0_4_2`) on either side of the `Top_Blocker` pour boundary [ADVANCED-ROUTING-IMPLEMENTATION.md].
      - No shorts or clearance violations are triggered during the DRC pass [ROUTING-AND-MANUFACTURING-ARCHITECTURE.md, Advanced-Routing-Implementation.md].

- [ ] **2.3 Physical and Visual Verification**
  *   **Action:** Open the generated `board.glb` file [HARDWARE-SCRIPT-MONITOR.md].
  *   **Verify:** 
      - The `top_copper` layer has two separate, discontinuous trace segments leading up to the via annular rings [Z-AXIS-ABSTRACTION-IMPLEMENTATION.md, Via-Engine-Implementation.md].
      - A single continuous copper trace runs horizontally along the `bottom_copper` layer directly underneath the `Top_Blocker` zone, connected by two hollow via tubes [Solder-Mask-Opening.md, Volumetric-Solid-Modeling-via-Boundary-Representation.md].

---

### Task 3: Hybrid Routing Coexistence (Manual + Auto)

This test proves that manually defined 3D trace coordinates are treated as static physical obstacles, forcing the auto-router to navigate around them without collisions [ROUTING-AND-MANUFACTURING-ARCHITECTURE.md, Unified-2.5D-3D-Routing-and-Placement.md].

- [ ] **3.1 Manual Trace Blitting**
  *   **Action:** Declare a manual route path with 3D coordinates for `NET_MANUAL` [Unified-2.5D-3D-Routing-and-Placement.md].
  *   **Verify:**
      - The compiler processes the manual route *first*, committing the copper geometries to the voxel grid prior to running the pathfinder [ROUTING-AND-MANUFACTURING-ARCHITECTURE.md, Unified-2.5D-3D-Routing-and-Placement.md].
      - In the logs, verify the manual trace's Z-boundaries sit exactly on `top_copper` [Z-AXIS-ABSTRACTION-IMPLEMENTATION.md].

- [ ] **3.2 Obstacle Inflation**
  *   **Action:** Run the compiler and monitor the Minkowski inflation pass [ROUTING-AND-MANUFACTURING-ARCHITECTURE.md, Unified-2.5D-3D-Routing-and-Placement.md].
  *   **Verify:**
      - The manual trace's 3D bounding box is inflated by $\frac{\text{Trace Width}}{2} + \text{Clearance}$ [ROUTING-AND-MANUFACTURING-ARCHITECTURE.md, Unified-2.5D-3D-Routing-and-Placement.md].
      - This inflated zone is marked as `Cost::INFINITE` in the router's cost map [ROUTING-AND-MANUFACTURING-ARCHITECTURE.md].

- [ ] **3.3 Collision-Free Auto-Routing**
  *   **Action:** Auto-route `NET_AUTO` across the same layer [Unified-2.5D-3D-Routing-and-Placement.md].
  *   **Verify:**
      - The auto-routed trace steers away from the manual trace, maintaining the minimum clearance specified in the profile [ROUTING-AND-MANUFACTURING-ARCHITECTURE.md, Unified-2.5D-3D-Routing-and-Placement.md].
      - Running `cargo run -- build` completes with `Violations: 0` [The-SoC-Engine.md, Advanced-Routing-Implementation.md].

---

### Task 4: Dynamic Pattern Integration & Length Matching

This test proves that a routing pattern (such as a serpentine) is imported and applied to the A* cost calculation dynamically via the pattern warping penalty [Unified-2.5D-3D-Routing-and-Placement.md, Base-Implementation-Roadmap.md].

- [ ] **4.1 Pattern Importation**
  *   **Action:** Import a standard routing pattern in your script [Unified-2.5D-3D-Routing-and-Placement.md, AUTHORITY-AND-LIBRARY-ARCHITECTURE.md]:
      `import Serpentine from @std/routing/patterns` [HPM-ARCHITECTURE.md]
  *   **Verify:**
      - The parser successfully registers the `Serpentine` symbol in the Symbol Table's HPM layer [AUTHORITY-AND-LIBRARY-ARCHITECTURE.md].

- [ ] **4.2 Warp Cost Application**
  - **Action:** Apply the pattern to the `NET_AUTO` route:
    `pattern: Serpentine(amplitude: 1mm, pitch: 0.5mm, target_length: 22mm)` [Unified-2.5D-3D-Routing-and-Placement.md]
  - **Verify:**
    - The A* cost evaluator calculates the step penalty $w(n)$ [Unified-2.5D-3D-Routing-and-Placement.md].
    - Inside unobstructed 2D zones, the step cost for deviating into serpentine folds drops to 0 [Unified-2.5D-3D-Routing-and-Placement.md].
    - The pathfinder generates a winding, step-like pattern matching the target length [Routing-&-Manufacturing-Roadmap.md].

- [ ] **4.3 2D Clipper Weld (Strategy A)**
  - **Action:** Export to GLB [HARDWARE-SCRIPT-MONITOR.md].
  - **Verify:**
    - The 2D Clipper engine executes a boolean union on the serpentine segments using the **Non-Zero Winding Rule** [2D-POLYGON-UNIONING-IMPLEMENTATION.md].
    - The exported GLB contains a single, continuous, non-overlapping copper mesh for the entire serpentine net, with zero Z-fighting or rendering anomalies [2D-POLYGON-UNIONING-IMPLEMENTATION.md, Z_FIGHTING_FIX.md].