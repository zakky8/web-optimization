# Blender PBR to Three.js MeshStandardMaterial Mapping

**Source:** KhronosGroup/glTF-Blender-IO (exporter `__init__.py` + extension files); Three.js `GLTFLoader.js`  
**Accessed:** 2026-05-26

---

## Overview

The Blender glTF exporter only understands one shader node: **Principled BSDF**. Every other node tree — procedural shaders, mix nodes, custom outputs — is either ignored, approximated, or requires texture baking before export. The Principled BSDF inputs map to the glTF PBR metallic-roughness model, which Three.js loads into `MeshStandardMaterial` (for base properties) or `MeshPhysicalMaterial` (for extension properties).

---

## Core Principled BSDF → glTF → Three.js Channel Map

| Blender Input (Principled BSDF) | glTF Channel | Three.js Property | Notes |
|----------------------------------|--------------|-------------------|-------|
| **Base Color** (solid value) | `pbrMetallicRoughness.baseColorFactor` | `material.color` (`Color`) | Linear-space RGB. Blender converts from sRGB to linear automatically. |
| **Base Color** (Image Texture node) | `pbrMetallicRoughness.baseColorTexture` | `material.map` (`Texture`) | sRGB texture. Three.js sets `texture.colorSpace = THREE.SRGBColorSpace` automatically on load. |
| **Alpha** (Principled BSDF Alpha input) | `pbrMetallicRoughness.baseColorTexture` alpha channel | `material.alphaMap` | Packed into the alpha channel of the base color texture when the image supports it (PNG/WebP). |
| **Metallic** (value) | `pbrMetallicRoughness.metallicFactor` | `material.metalness` | 0.0–1.0. |
| **Metallic** (Image Texture, B channel) | `pbrMetallicRoughness.metallicRoughnessTexture` | `material.metalnessMap` | Blender packs Metallic into the **blue channel** and Roughness into the **green channel** of a single combined ORM texture. |
| **Roughness** (value) | `pbrMetallicRoughness.roughnessFactor` | `material.roughness` | 0.0–1.0. |
| **Roughness** (Image Texture, G channel) | `pbrMetallicRoughness.metallicRoughnessTexture` | `material.roughnessMap` | Same combined texture as metalness. Three.js reads G channel for roughness, B channel for metalness from the one `metalnessMap` / `roughnessMap` texture object. |
| **Normal Map** (Normal Map node → Normal) | `normalTexture` | `material.normalMap` | Requires a **Normal Map** node between the Image Texture and Principled BSDF Normal input. The exporter reads the `Strength` input of the Normal Map node as `normalTexture.scale`. |
| **Emission Color** (value) | `emissiveFactor` | `material.emissive` (`Color`) | |
| **Emission Color** (Image Texture) | `emissiveTexture` | `material.emissiveMap` (`Texture`) | |
| **Emission Strength** | `KHR_materials_emissive_strength.emissiveStrength` | `material.emissiveIntensity` | Values > 1.0 require the `KHR_materials_emissive_strength` extension (Blender 3.4+, Three.js r147+). |
| **Ambient Occlusion** | No direct Principled BSDF input — must be baked or connected via a Shader to RGB node hack | `material.aoMap` | glTF stores AO in the **red channel** of `occlusionTexture`. Requires UV2 in Three.js (see `blender-baking.md`). |
| **Alpha Mode** (Blend Mode setting on material) | `alphaMode`: `OPAQUE`, `MASK`, `BLEND` | `material.transparent`, `material.alphaTest` | Set via material Properties > Settings > Blend Mode in Blender. |
| **Alpha Cutoff** | `alphaCutoff` | `material.alphaTest` | Only relevant when `alphaMode = MASK`. |
| **Double Sided** (Backface Culling toggle) | `doubleSided: true` | `material.side = THREE.DoubleSide` | |

---

## Packed Texture Conventions

The Blender exporter combines channels to minimize texture count:

| Map | Channel | glTF Texture |
|-----|---------|-------------|
| Ambient Occlusion | R | `occlusionTexture` |
| Roughness | G | `metallicRoughnessTexture` |
| Metalness | B | `metallicRoughnessTexture` |

This ORM packing is standard across the industry. When creating source textures in Substance Painter or Quixel, export using "glTF Metallic Roughness" preset to match this layout.

---

## Extended Properties via MeshPhysicalMaterial

When extension nodes are present on the Principled BSDF, the exporter writes the corresponding KHR extension. Three.js `GLTFLoader` upgrades the material from `MeshStandardMaterial` to `MeshPhysicalMaterial` to accommodate these properties.

| Blender Principled BSDF Input | KHR Extension | Three.js Property |
|-------------------------------|---------------|-------------------|
| **Transmission** | `KHR_materials_transmission` | `material.transmission` |
| **IOR** | `KHR_materials_ior` | `material.ior` |
| **Coat Weight** (Blender 4.0+) / **Clearcoat** (≤3.6) | `KHR_materials_clearcoat` | `material.clearcoat`, `material.clearcoatRoughness` |
| **Sheen Weight**, **Sheen Color**, **Sheen Roughness** | `KHR_materials_sheen` | `material.sheen`, `material.sheenColor`, `material.sheenRoughness` |
| Thickness / Attenuation Color / Attenuation Distance (via Volume Absorption node in shader) | `KHR_materials_volume` | `material.thickness`, `material.attenuationColor`, `material.attenuationDistance` |

See `gltf-extensions.md` for full per-extension property tables and Blender version requirements.

---

## What Does NOT Export

The following node types and features are silently dropped or approximated at export time:

### Node Types That Are Ignored

| Node | Why It Fails | Workaround |
|------|-------------|------------|
| **Shader Mix / Add Shader** | glTF has no multi-layer material blending. The exporter can only handle one Principled BSDF per material output. | Bake the combined result to a texture before export. |
| **Procedural textures** (Noise, Wave, Voronoi, Musgrave, etc.) | These exist only in Blender's shader graph; there is no glTF equivalent. | Bake to an Image Texture node before export (see `blender-baking.md`). |
| **Texture Coordinate node (Generated/Object/Camera)** | glTF only supports UV-mapped and environment-mapped textures. | Re-UV-map the object using the relevant UV coordinates. |
| **Color Ramp** | Evaluated at bake time only. | Bake the full chain to a flat image. |
| **Math / RGB Curves** | No glTF equivalent for per-channel math operations. | Bake. |
| **Displacement node** (true geometry displacement) | glTF does not support mesh displacement at runtime. | Bake the displaced shape into a static mesh or use a normal map approximation. |
| **Subsurface Scattering** | Not in glTF PBR metallic-roughness model. `KHR_materials_subsurface` is a draft extension not yet in Three.js. | Use a Translucent workaround or approximate with base color + emission. |
| **Anisotropic BSDF** | Not the same as `KHR_materials_anisotropy`; the exporter uses the Principled BSDF Anisotropic input for that extension. Custom Anisotropic nodes are dropped. | Use the Principled BSDF Anisotropic input. |
| **Holdout / Background shaders** | Scene-level constructs, not material properties. | N/A. |
| **Volume Scatter / Volume Absorption** (standalone, not via Principled BSDF) | Partially supported via `KHR_materials_volume` only when connected through Principled BSDF. | Connect through Principled BSDF where possible. |

### Limitations Even with Principled BSDF

- **Multiple UV maps in a single material:** The exporter supports `TEXCOORD_0` and `TEXCOORD_1` (UV0 and UV1), but managing which texture uses which UV set requires explicit UV Map node connections in the Blender shader.
- **Specular tint node group:** Blender 4.0 changed the Principled BSDF to use Specular Tint as a color input; the exporter maps this to `KHR_materials_specular` but only when the node version matches what the exporter expects.
- **Texture rotation beyond UV Transform:** Complex UV manipulation via Vector Math nodes is not exportable; use Blender's **Mapping** node (which maps to `KHR_texture_transform`).

---

## Practical Material Setup for Three.js

### Minimal Correct Node Graph

```
[Image Texture: albedo.png]  ─── Base Color ─┐
[Image Texture: orm.png]     ─── AO: (R)     │
                              ── Roughness:(G)├─ [Principled BSDF] ─── [Material Output]
                              ── Metalness:(B)│
[Normal Map node]
  └─ [Image Texture: normal.png] ─ Color ──── Normal ┘
```

### Ensuring AO Exports Correctly

AO via Principled BSDF is not a direct socket — you must use the **Bake** workflow to generate an AO map and reference it as `occlusionTexture` in glTF (red channel of the ORM texture). Three.js reads it via `material.aoMap` which requires the mesh to have a second UV set assigned to `uv2` attribute. See `blender-baking.md`.

### Alpha Blend Mode Settings

Set in **Material Properties > Settings > Blend Mode**:
- `Opaque` → `alphaMode: OPAQUE`
- `Alpha Clip` → `alphaMode: MASK` (set cutoff under Clip Threshold)
- `Alpha Blend` → `alphaMode: BLEND`

Use `BLEND` sparingly on web — `BLEND` requires painter's-algorithm sorting and has performance implications in Three.js with many overlapping transparent surfaces.
