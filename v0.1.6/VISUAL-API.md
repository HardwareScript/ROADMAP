# God-Tier Material Visual API (v0.1.6)

**Date**: 2026-04-18  
**Status**: Implementation Ready  
**Philosophy**: From "Basic Color" to Physically Based Rendering (PBR)

---

## Overview

Hardware Script is adopting industry-standard visual properties to enable **Silicon-Photo-Realistic** 3D exports. This upgrade brings CSS-style transparency control and PBR material properties to hardware design.

---

## API Specification

### Visual Properties

All properties are **optional**. If omitted, defaults to "Standard Opaque Engineering Material."

| Property | Type | Default | Description |
|----------|------|---------|-------------|
| `color` | HEX | `#808080` | Base color of the material |
| `opacity` | 0.0 - 1.0 | `1.0` | Transparency of the material volume |
| `outline_opacity` | 0.0 - 1.0 | `0.0` | Visibility of physical edges |
| `roughness` | 0.0 - 1.0 | `0.5` | 0.0 = Mirror/Glossy, 1.0 = Matte/Diffuse |
| `metallic` | 0.0 - 1.0 | `0.0` | 1.0 = Conductive metal, 0.0 = Ceramic/Plastic |

---

## Outline Logic

The `outline_opacity` enables three engineering visualization modes:

**Ghost View**: `opacity: 0.0, outline_opacity: 0.2`  
- Faint box outline, transparent interior
- See through components without obstruction

**Minecraft View**: `opacity: 0.1, outline_opacity: 1.0`  
- Tinted glass with sharp edges
- Clear boundary definition

**Solid View**: `opacity: 1.0, outline_opacity: 0.0`  
- No visible edges, solid mass
- Traditional CAD appearance

---

## Syntax Examples

```hardware
material SiliconDioxide:
    category: insulator
    properties:
        color: "#E0F7FA"
        opacity: 0.15          # See-through glass
        outline_opacity: 0.8   # Strong edges
        roughness: 0.05        # High gloss (refractive look)
        metallic: 0.0          # Glass, not metal

material Aluminum:
    category: conductor
    properties:
        color: "#D1D5DB"
        opacity: 1.0           # Solid metal
        outline_opacity: 0.0   # No lines needed
        roughness: 0.2         # Polished metal finish
        metallic: 1.0          # Metallic reflection

material Polysilicon:
    category: conductor
    properties:
        color: "#808080"
        opacity: 1.0
        outline_opacity: 0.0
        roughness: 0.7         # Matte semiconductor
        metallic: 0.3          # Semi-metallic
```

---

## Implementation Plan

### Phase 1: Core Data Structures
**Crates**: `hwc-parser`, `hwc-compiler`

- [ ] Update `MaterialDefinition` struct with 5 optional visual fields
- [ ] Update `MaterialState` struct for runtime storage
- [ ] Implement `Default` trait for backward compatibility
- [ ] Update material parser to accept new properties

### Phase 2: Outline Generator
**Crate**: `hwc-export`

- [ ] Modify `SceneGraph` to check `outline_opacity`
- [ ] Generate line primitives for bounding box edges when `outline_opacity > 0.0`
- [ ] Add 12 edge lines per material volume
- [ ] Export lines to OBJ/GLB formats

### Phase 3: PBR Shader Integration
**Crate**: `hwc-export`

**OBJ (.mtl)**:
- [ ] Map `roughness` to `Ns` (specular exponent)
- [ ] Map `opacity` to `d` (dissolve)
- [ ] Map `metallic` to `Pm` (metallic factor)

**GLB (GLTF)**:
- [ ] Use `KHR_materials_pbrMetallicRoughness` extension
- [ ] Map `metallic` to `metallicFactor`
- [ ] Map `roughness` to `roughnessFactor`
- [ ] Map `opacity` to `alphaMode` and `alphaCutoff`

**Blender**:
- [ ] Map to Principled BSDF node inputs:
  - `metallic` → Metallic
  - `roughness` → Roughness
  - `opacity` → Alpha (Transmission)
  - `color` → Base Color

---

## Validation Plan

### Test Case: `test_contact.hw`

Update `materials.hw` with new visual properties:

1. **Gate (Polysilicon)**: Matte semiconductor appearance
2. **Metal (Aluminum)**: Glossy conductor with metallic reflection
3. **Via (Tungsten)**: Solid pillar with polished finish
4. **Substrate (Optional)**: Transparent glass to see internal structure

### Success Criteria

- [ ] OBJ export includes `.mtl` file with PBR properties
- [ ] GLB export uses GLTF PBR extensions
- [ ] Blender script generates Principled BSDF nodes
- [ ] Outline edges visible when `outline_opacity > 0.0`
- [ ] Existing `.hw` files compile without modification (optional properties use defaults)

---

## Export Format Mapping

| Property | OBJ (.mtl) | GLB (GLTF) | Blender |
|----------|------------|------------|---------|
| `color` | `Kd` (diffuse) | `baseColorFactor` | Base Color |
| `opacity` | `d` (dissolve) | `alphaMode` | Alpha |
| `outline_opacity` | Line primitives | Line primitives | Edge modifier |
| `roughness` | `Ns` (inverse) | `roughnessFactor` | Roughness |
| `metallic` | `Pm` | `metallicFactor` | Metallic |

---

## Forward Compatibility

**Note**: Hardware Script is pre-release. There is no "backward compatibility" obligation.

However, all new visual properties are **optional with sensible defaults**. This means:

**Existing files** (without visual properties):
```hardware
material Copper:
    category: conductor
    properties:
        color: "#B87333"
```
✅ **Compiles successfully** - Uses default visual properties

**Enhanced files** (with visual properties):
```hardware
material Copper:
    category: conductor
    properties:
        color: "#B87333"
        opacity: 1.0
        roughness: 0.3
        metallic: 1.0
```
✅ **Compiles successfully** - Uses specified visual properties

**Result**: Every previously written `.hw` file will compile without modification, even though they lack the new visual API.

---

## Priority

**HIGH** - Enables professional-grade 3D visualization for marketing, documentation, and design review.
