# ROADMAP v0.1.7: Manifold Export Rethink (The "Game Engine" Path)

## The Problem: The Ghost of CAD Past
Current CAD software treats geometry as a collection of overlapping ghosts. When two surfaces (like the top of a substrate and the bottom of a plate) share the same Z-coordinate, the GPU suffers from **Z-fighting (flickering)**. This is a failure to represent the Order of Truth.

## The Solution: Manifold "Lego-Plug" Export
Hardware Script v0.1.7 will innovate by adopting "Voxel-to-Manifold" meshing, a technique perfected by modern game engines (e.g., *No Man's Sky*, *Minecraft*) and high-end physics simulators.

### 1. Neighbor Culling (Voxel-to-Manifold)
Instead of exporting raw `add` commands as individual boxes, the exporter will reconstruct the geometry from the **Voxel State**.

- **Neighbor Culling**: If a voxel face exists between two identical materials, it is internal and deleted.
- **Interface Ownership**: If a face exists between two different materials, only one set of triangles is generated, assigned to the higher-precedence material.
- **Boolean Subtraction**: The lower-precedence material (substrate) is mathematically "hollowed out" to accommodate the higher-precedence material (plates).

### 1.1 Strategy A: 2D Co-Union / 3D Boolean Union (v0.1.8)
For copper layers (pours, traces, vias), we go beyond simple face culling. We adopt **Strategy A**:
- **2D Polygon Unioning**: Use a polygon clipper (Vatti algorithm) to merge all copper shapes on the same net into a single contour.
- **Single Extrusion**: Extrude the unified contour to the layer thickness.
- **Manifold Result**: The exported GLB contains a single, continuous, non-overlapping solid mesh for the entire net on that layer.

### 2. The Precedence Hierarchy (The Order of Truth)
We define a physical precedence levels to resolve material conflicts at shared boundaries:

| Level | Material Category | Examples |
| :--- | :--- | :--- |
| **1 (Highest)** | Metals / Conductors | Gold, Copper, Aluminum |
| **2** | Active Components | Silicon, Package Plastic |
| **3** | Protective Layers | Solder Mask, Conformal Coating |
| **4 (Lowest)** | Substrates | FR4, Glass, Ceramic |

### 3. Implementation: Pre-Mesh Culling Algorithm
The `hwc-export` crate will implement a culling algorithm to decide which faces to render based on their neighbors.

```rust
// Logic for hwc-export/src/scene_graph/manifold.rs

fn should_render_face(voxel: &Voxel, neighbor: &Voxel) -> bool { 
    // 1. Internal Face: Same material neighbor? Delete it.
    if voxel.material == neighbor.material { return false; } 
    
    // 2. External Face: Air neighbor? Render it.
    if neighbor.is_air() { return true; } 
    
    // 3. Conflict Point: Different material neighbor?
    // Only render the face if current voxel has higher Precedence.
    if voxel.precedence > neighbor.precedence { 
        return true; 
    } 
    
    // 4. Submission: Higher material owns this surface. I stay silent.
    false 
}
```

### 4. Benefits: Beyond the Visual
- **Visual Confidence**: Designers see a "Lego-like" fit where plates are recessed into sockets, providing immediate visual feedback that parts are flush.
- **Geometric Integrity**: Dimensions remain exact (e.g., 1.000mm), but surface ownership is explicitly defined.
- **Performance**: Triangle counts are reduced by culling redundant shared interfaces.

## Summary for v0.1.7 Manual Render Tests
In scripts like `ManualRenderSpace`, the compiler will no longer just dump two boxes on top of each other. It will "punch" holes into the substrate and "plug" the plates in. The flicker stops because the substrate material literally ceases to exist at the interface coordinates.
