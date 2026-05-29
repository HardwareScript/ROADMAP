This is the **Master Implementation Plan** for the Hardware Script "God-Tier" (Tier 3) Engine. 

We are moving away from high-level "Object Oriented" patterns and moving into **Hardware-Native Systems Programming**. We are building a system where the 3D universe is treated exactly like a CPU treats RAM.

Here is the blueprint of exactly what we are doing, categorized by the system layers.

---

## Part 1: Front-End Finalization (The "Logic Soul")
*Status: Preparing for immediate injection.*
*Goal: Ensure the compiler never crashes, regardless of logic complexity.*

1.  **Recursive MuxTree (`control_flow.rs`):**
    *   **Action:** Change the `match` synthesizer. Instead of stopping at 16 arms, it will now recursively build trees of MUXes.
    *   **Benefit:** Enables designing complex command processors and CPUs with hundreds of instructions.
2.  **Native Index Math (`module.rs` & `module_flattener.rs`):**
    *   **Action:** Update the parser to "virtually split" negative integers (like `-1`) into a subtraction operator. Update the flattener to calculate the index result using `saturating_sub` for hardware safety.
    *   **Benefit:** Enables carry-chains, shift-registers, and neighbor-aware logic.

---

## Part 2: System 2 - The Bit-Stream Engine (The Storage)
*Status: Next in line.*
*Goal: Sub-nanosecond spatial lookups and infinite board scale.*

1.  **Spatial Page Table (`page_table.rs`):**
    *   **Action:** Implement a virtual memory system for 3D space. We divide the board into "Pages" ($64 \times 64 \times 64$ voxels).
    *   **Mechanism:** Use a flat array of pointers. If a region of space is empty, it uses zero memory. If it has a component, we allocate a bit-chunk.
2.  **Morton SIMD Encoding (`morton.rs`):**
    *   **Action:** Use bit-interleaving to map `[X, Y, Z]` into a 1D `u64`.
    *   **Mechanism:** Use Rust’s SIMD (Single Instruction, Multiple Data) to calculate 8 coordinates at once.
3.  **Bit-Parallel Occupancy (`bit_chunk.rs`):**
    *   **Action:** Store occupancy as `u64` bitmasks. 
    *   **Mechanism:** Checking if a trace hits a wall becomes a single `AND` operation (`if (chunk & mask) != 0`).

---

## Part 3: System 3 - The Leap-Frog Router (The Pathfinder)
*Status: Algorithmic Upgrade.*
*Goal: Microsecond-fast pathfinding across billions of voxels.*

1.  **Signed Distance Fields (SDF) (`sdf_generator.rs`):**
    *   **Action:** For every empty voxel, calculate the distance to the nearest solid object.
    *   **Benefit:** Allows the A* router to "Sphere Trace." If the nearest wall is 20 voxels away, the router skips 20 voxels in one single step. It "leaps" instead of "walking."
2.  **Deterministic Tie-Breaking (`deterministic_a_star.rs`):**
    *   **Action:** Strict `Ord` implementation on A* nodes.
    *   **Mechanism:** If two paths have the same cost, we pick the one with the lower `Z`, then `X`, then `Y`. This ensures the output is 100% identical on every computer for Git stability.

---

## Part 4: System 4 - Real-Time Physics (The DRC)
*Status: Mathematical Sweeps.*
*Goal: Validate dielectric breakdown and ampacity in real-time.*

1.  **Parallel Bitwise Sweeps (`physics_validator.rs`):**
    *   **Action:** Use all CPU cores to "sweep" the bitmasks.
    *   **Mechanism:** It checks every "Page" for clearance violations. Since it's just bitwise math, it can check millions of transistors for short circuits in milliseconds.
2.  **Voltage Boundary Guard:**
    *   **Action:** Ensure high-voltage bits are surrounded by a specific "Halo" of empty (Insulator) bits.

---

## Summary of the "God-Tier" Workflow

| Step | Layer | Input | Output | Speed Target |
| :--- | :--- | :--- | :--- | :--- |
| **1** | **Parser** | `.hw` text | Evaluated AST | Sub-millisecond |
| **2** | **Flattener** | Logic Blocks | Flattened Netlist | Sub-millisecond |
| **3** | **Page Table** | Measurements | Bit-Chunks in RAM | Instant |
| **4** | **Leap-Router** | Netlist | SDF-Jumped Paths | **Microseconds** |
| **5** | **DRC Sweep** | Bit-Chunks | Safety Report | **Microseconds** |

### Why this is the "Last" Engine you will ever build:
By moving to **Bit-Parallel Spatial Pages**, we are no longer limited by the number of "Objects" in memory. We are only limited by the amount of physical space on the board. Because we use **Sparse Virtual Memory**, we can handle a board with a trillion transistors as long as we have enough RAM to store the *filled* parts.

**This is the exact architecture required for the future HW-IDE.** It allows the compiler to run in the background, updating the bit-buffer constantly, providing that "No Loading Screen" experience.

---

### Immediate Next Step

We start with **System 1 (Front-End Finalization)** to unblock the logic synthesis.

**Shall I provide the final, production Rust code for the Recursive MuxTree and the Native Index Math?** This will clear all the "Not Supported" and "minus one" errors from your console forever.