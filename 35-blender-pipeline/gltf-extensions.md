# glTF Material Extensions: Blender Export & Three.js Support

**Sources:**  
- KhronosGroup/glTF-Blender-IO `__init__.py` (extension property definitions)  
- KhronosGroup/glTF extension READMEs (transmission, volume, ior, clearcoat, sheen)  
- Three.js `GLTFLoader.js` (registered extension handlers)  
**Accessed:** 2026-05-26

---

## Extension Overview

These five extensions go beyond the base metallic-roughness model. In Blender they are driven by Principled BSDF socket values. In Three.js they require `MeshPhysicalMaterial` — `GLTFLoader` automatically upgrades a material from `MeshStandardMaterial` to `MeshPhysicalMaterial` when it detects any of these extensions in the file.

All five are fully supported in current Three.js (r150+). They are registered as named plugin classes in `GLTFLoader.js` constructor.

---

## KHR_materials_clearcoat

### Purpose

Simulates a transparent coating layer on top of a base material — varnished wood, lacquered paint, carbon fibre with resin, polished plastic. The clearcoat has its own roughness and can have its own normal map for micro-scratches on the coat surface independent of the base normal map.

### Blender Source Inputs (Principled BSDF)

| Blender Input | Blender Version Introduced | Notes |
|--------------|---------------------------|-------|
| Coat Weight | Blender 4.0 (renamed from "Clearcoat") | Formerly "Clearcoat" in Blender ≤ 3.6 |
| Coat Roughness | Blender 4.0 | Formerly "Clearcoat Roughness" |
| Coat Normal | Blender 4.0 | Via Normal Map node |

The exporter has supported KHR_materials_clearcoat since **Blender 2.83** (initial extension support), with the socket rename handled automatically in the exporter's compatibility layer.

### glTF Properties

| glTF Property | Range | Description |
|--------------|-------|-------------|
| `clearcoatFactor` | 0.0–1.0 | Intensity of the coating layer |
| `clearcoatRoughnessFactor` | 0.0–1.0 | Roughness of the coating surface |
| `clearcoatTexture` | Texture | Multiplies `clearcoatFactor` (R channel) |
| `clearcoatRoughnessTexture` | Texture | Multiplies `clearcoatRoughnessFactor` (G channel) |
| `clearcoatNormalTexture` | Normal texture | Normal map for the coating layer only |

### Three.js Mapping (MeshPhysicalMaterial)

| glTF Property | Three.js Property |
|--------------|-------------------|
| `clearcoatFactor` | `material.clearcoat` |
| `clearcoatRoughnessFactor` | `material.clearcoatRoughness` |
| `clearcoatTexture` | `material.clearcoatMap` |
| `clearcoatRoughnessTexture` | `material.clearcoatRoughnessMap` |
| `clearcoatNormalTexture` | `material.clearcoatNormalMap` |
| `clearcoatNormalTexture.scale` | `material.clearcoatNormalScale` |

### Constraints

- Cannot be combined with `KHR_materials_pbrSpecularGlossiness` or `KHR_materials_unlit`.
- Meshes using `clearcoatNormalTexture` must have `NORMAL` and `TANGENT` vertex attributes defined (enable `export_tangents = true` in Blender export settings).

---

## KHR_materials_transmission

### Purpose

Physically-based optical transparency. Unlike standard alpha blending (which is coverage-based and does not simulate refraction), transmission allows light to pass through the surface with tinting, roughness-driven blurring of transmitted light, and proper Fresnel reflection. Use for glass, crystal, ice, and clear liquids.

### Key Distinction from Alpha Blending

Alpha blending (`alphaMode: BLEND`) cuts geometry away. Transmission keeps the surface opaque to reflections while letting light through. A material with `transmissionFactor = 1.0` should still have `alphaMode: OPAQUE`.

### Blender Source Inputs (Principled BSDF)

| Blender Input | Notes |
|--------------|-------|
| Transmission Weight | Principled BSDF "Transmission" socket (Blender 3.0+, formerly "Transmission" in older builds) |
| Roughness | Controls blurring of transmitted light (existing roughness socket) |
| Base Color | Tints transmitted light |

Blender has exported this extension since approximately **Blender 3.0** (glTF-Blender-IO 3.x release cycle). Check exporter release notes for exact version.

### glTF Properties

| glTF Property | Range | Description |
|--------------|-------|-------------|
| `transmissionFactor` | 0.0–1.0 | Percentage of light transmitted through the surface |
| `transmissionTexture` | Texture | Per-texel transmission factor (R channel), multiplied by `transmissionFactor` |

### Three.js Mapping (MeshPhysicalMaterial)

| glTF Property | Three.js Property |
|--------------|-------------------|
| `transmissionFactor` | `material.transmission` |
| `transmissionTexture` | `material.transmissionMap` |

### Three.js Rendering Notes

Transmission in Three.js requires the renderer's `transmissionPass` to work. Use:

```js
const renderer = new THREE.WebGLRenderer();
renderer.physicallyCorrectLights = true; // legacy flag, ensure tone mapping is set
// Transmission uses a render target internally — no special setup required in r152+
```

For fully correct transmission with refraction blur, the scene must be rendered with `WebGLRenderer` (not `WebGPURenderer` in early builds). Performance cost is non-trivial for many transmissive objects.

---

## KHR_materials_volume

### Purpose

Transforms a surface from a thin-walled boundary into a volumetric medium. Enables simulation of light attenuation (absorption) as it travels through the volume, creating coloured glass, amber, milk, and similar materials. Requires a closed mesh to function correctly.

**Mandatory dependency:** `KHR_materials_transmission` must also be present. Volume effects cannot occur without transmission.

### Blender Source Inputs

Volume parameters do not map directly to Principled BSDF sockets in the standard sense. The Blender exporter reads:

| Blender Node/Input | glTF Property |
|-------------------|--------------|
| Volume Absorption node (Color) in the Volume socket of Material Output | `attenuationColor` |
| Volume Absorption node (Density) | Approximated into `attenuationDistance` |
| Principled BSDF geometry thickness (approximated) | `thicknessFactor` |

Blender 3.3+ improves volume export fidelity. For precise control, set values directly via a custom glTF properties panel or via the glTF Material Extras extension.

### glTF Properties

| glTF Property | Description |
|--------------|-------------|
| `thicknessFactor` | Depth of the volume in metres; 0 = thin-walled. Non-zero requires a closed mesh. |
| `thicknessTexture` | Per-texel thickness (G channel), multiplied by `thicknessFactor` |
| `attenuationDistance` | Distance (metres) for light to lose half its intensity to absorption |
| `attenuationColor` | RGB colour light tints toward after travelling `attenuationDistance` |

### Three.js Mapping (MeshPhysicalMaterial)

| glTF Property | Three.js Property |
|--------------|-------------------|
| `thicknessFactor` | `material.thickness` |
| `thicknessTexture` | `material.thicknessMap` |
| `attenuationDistance` | `material.attenuationDistance` |
| `attenuationColor` | `material.attenuationColor` |

---

## KHR_materials_ior

### Purpose

Overrides the fixed IOR of 1.5 used in the base glTF metallic-roughness model. IOR affects Fresnel reflectance at grazing angles and the degree of refraction (in combination with `KHR_materials_transmission` / `KHR_materials_volume`). Without this extension, all dielectrics behave as if they are standard glass/plastic (1.5).

### Blender Source Inputs

| Blender Input | Notes |
|--------------|-------|
| IOR (Principled BSDF) | Direct socket. Default in Blender is 1.45 (glass). Blender 3.0+ exports this as `KHR_materials_ior` when the value differs from 1.5. |

### glTF Properties

| glTF Property | Default | Range | Description |
|--------------|---------|-------|-------------|
| `ior` | 1.5 | ≥ 1.0 (0 = legacy compat mode) | Index of refraction |

**Reference IOR values:**

| Material | IOR |
|----------|-----|
| Air | 1.00 |
| Water | 1.33 |
| Standard glass | 1.50 |
| Crystal glass | 1.60 |
| Sapphire | 1.76 |
| Diamond | 2.42 |

### Three.js Mapping (MeshPhysicalMaterial)

| glTF Property | Three.js Property |
|--------------|-------------------|
| `ior` | `material.ior` |

### Constraints

- Cannot be combined with `KHR_materials_pbrSpecularGlossiness` or `KHR_materials_unlit`.
- IOR alone has no visible effect without transmission. Its primary purpose is to adjust Fresnel at grazing angles and to drive refraction in volumetric materials.

---

## KHR_materials_sheen

### Purpose

Adds a microfibre sheen layer for fabric and cloth simulation. The sheen model is based on a separate BRDF lobe that simulates back-scattering of velvet-like fibres. Sheen sits on top of the base metallic-roughness layer and below clearcoat when all three are combined.

**Layering order:** base metallic-roughness → sheen → clearcoat.

### Blender Source Inputs (Principled BSDF)

| Blender Input | Notes |
|--------------|-------|
| Sheen Weight | Controls sheen intensity (Blender 4.0+; "Sheen" in earlier versions) |
| Sheen Tint | Blender 4.0+; maps to sheen colour |
| Sheen Roughness | Blender 4.0+ |

Blender has exported `KHR_materials_sheen` since approximately **Blender 3.3** (glTF-Blender-IO 3.3.x). Pre-4.0 builds used different socket names that the exporter handled via a compatibility layer.

### glTF Properties

| glTF Property | Description |
|--------------|-------------|
| `sheenColorFactor` | RGB colour of the sheen layer. Default: black (disabled) |
| `sheenColorTexture` | RGB texture multiplied by `sheenColorFactor` |
| `sheenRoughnessFactor` | 0.0–1.0. Controls sharpness of the microfibre highlight |
| `sheenRoughnessTexture` | Roughness in alpha channel, multiplied by `sheenRoughnessFactor` |

Sheen is disabled when `sheenColorFactor` is black (0, 0, 0).

### Three.js Mapping (MeshPhysicalMaterial)

| glTF Property | Three.js Property |
|--------------|-------------------|
| `sheenColorFactor` | `material.sheenColor` (`Color`) |
| `sheenColorTexture` | `material.sheenColorMap` |
| `sheenRoughnessFactor` | `material.sheenRoughness` |
| `sheenRoughnessTexture` | `material.sheenRoughnessMap` |

Three.js exposes a scalar `material.sheen` (0–1) as the overall sheen intensity, mapped from the magnitude of `sheenColorFactor`.

---

## Additional Extensions Supported by Three.js GLTFLoader

The following are also registered in `GLTFLoader.js` (Three.js r150+):

| Extension | Purpose | Three.js Property |
|-----------|---------|-------------------|
| `KHR_materials_emissive_strength` | Emissive multiplier > 1.0 (HDR emissives) | `material.emissiveIntensity` |
| `KHR_materials_specular` | Tinted specular F0 for dielectrics | `material.specularIntensity`, `material.specularColor` |
| `KHR_materials_iridescence` | Thin-film interference (soap bubble, oil slick) | `material.iridescence`, `material.iridescenceIOR`, etc. |
| `KHR_materials_anisotropy` | Directional highlights (brushed metal) | `material.anisotropy`, `material.anisotropyVector` |
| `KHR_materials_dispersion` | Chromatic IOR dispersion | `material.dispersion` |
| `KHR_lights_punctual` | Point, spot, directional lights | `THREE.PointLight`, `THREE.SpotLight`, `THREE.DirectionalLight` |
| `KHR_draco_mesh_compression` | Draco geometry compression | Requires external `DRACOLoader` |
| `KHR_mesh_quantization` | Vertex attribute quantization | Handled internally |
| `KHR_texture_transform` | UV offset/rotation/scale | `texture.offset`, `texture.repeat`, `texture.rotation` |
| `EXT_texture_webp` | WebP texture format | Handled by browser WebP support |
| `KHR_texture_basisu` | Basis Universal GPU texture compression | Requires `KTX2Loader` |
| `EXT_meshopt_compression` / `KHR_meshopt_compression` | Meshopt geometry compression | Requires `MeshoptDecoder` |
| `EXT_mesh_gpu_instancing` | Hardware instancing | `THREE.InstancedMesh` |

---

## Blender Version Summary for Key Extensions

| Extension | First Exported (Approx.) | Notes |
|-----------|------------------------|-------|
| `KHR_materials_clearcoat` | Blender 2.83 | Socket renamed in Blender 4.0 from "Clearcoat" to "Coat Weight" |
| `KHR_materials_transmission` | Blender 3.0 | Requires Principled BSDF Transmission socket |
| `KHR_materials_ior` | Blender 3.0 | Exported when IOR ≠ 1.5 |
| `KHR_materials_volume` | Blender 3.0–3.3 (improving) | Volume Absorption node required |
| `KHR_materials_sheen` | Blender 3.3 | Socket renamed in Blender 4.0 from "Sheen" to "Sheen Weight" |

For verified version data, consult the [glTF-Blender-IO release changelog](https://github.com/KhronosGroup/glTF-Blender-IO/releases) for the exact Blender build each extension was introduced in.
