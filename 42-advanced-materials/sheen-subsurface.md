# Sheen & Subsurface Scattering — MeshPhysicalMaterial

**Source revision:** Three.js r184
**Accessed:** 2026-05-26
**Primary sources:**
- `src/materials/MeshPhysicalMaterial.js` (r184) — property defaults
- `src/materials/nodes/MeshPhysicalNodeMaterial.js` (r184) — node overrides
- `src/materials/nodes/MeshSSSNodeMaterial.js` (r184) — SSS approximation implementation
- `docs/pages/MeshPhysicalMaterial.html.md` (r184) — official descriptions

---

## Part 1: Sheen (Fabric / Cloth)

### What it does

Sheen adds a retro-reflective, grazing-angle highlight that reads as a fabric microfiber catch. Physically it approximates the microgeometry of textile fibres, which scatter light back toward the viewer at steep angles — the characteristic "velvet halo." It follows the KHR_materials_sheen glTF extension and uses an Ashikhmin sheen BRDF.

Without sheen, a rough dielectric sphere looks like matte plastic. With `sheen = 1.0` and a coloured `sheenColor`, the same sphere reads as velvet or microsuede.

### Properties

#### `sheen`

| Field | Value |
|-------|-------|
| Type | `Float` |
| Default | `0` |
| Range | `0.0 – 1.0` |

"The intensity of the sheen layer." At `0` there is no sheen. Enabling sheen at any value > 0 activates the sheen BRDF in the fragment shader.

```js
material.sheen = 1.0;
```

#### `sheenColor`

| Field | Value |
|-------|-------|
| Type | `Color` |
| Default | `Color(0, 0, 0)` (black — no tint) |

"The sheen tint." Controls the spectral colour of the sheen highlight. A white `sheenColor` produces a neutral near-white catch light (cotton). A coloured value shifts the specular fringe chromatically.

```js
material.sheenColor = new THREE.Color(0.8, 0.3, 0.1); // rust-orange velvet
```

A `sheenColorMap` (Texture) multiplies against `sheenColor` per-texel.

#### `sheenRoughness`

| Field | Value |
|-------|-------|
| Type | `Float` |
| Default | `1` |
| Range | `0.0 – 1.0` |

Roughness of the sheen lobe. **Default is `1.0` (fully rough)** — intentional, as sheen is always a soft diffuse-like highlight, not a sharp specular. Lower values compress the lobe toward grazing angles more sharply. Values below `0.3` produce an unrealistic ring artefact on most meshes.

```js
material.sheenRoughness = 0.8; // soft velvet
material.sheenRoughness = 0.4; // tighter fibre highlight
```

### Node material overrides

```js
// MeshPhysicalNodeMaterial
material.sheenNode = tslVec3Node;           // overrides sheenColor (as vec3)
material.sheenRoughnessNode = tslFloatNode; // overrides sheenRoughness
```

### Velvet recipe

```js
const velvet = new THREE.MeshPhysicalMaterial({
  color: 0x1a0033,           // deep purple base
  roughness: 0.95,
  metalness: 0.0,
  sheen: 1.0,
  sheenColor: new THREE.Color(0.6, 0.2, 0.8),
  sheenRoughness: 0.7,
});
```

### Fabric / microfibre recipe

```js
const cotton = new THREE.MeshPhysicalMaterial({
  color: 0xf5f0e8,
  roughness: 1.0,
  metalness: 0.0,
  sheen: 0.6,
  sheenColor: new THREE.Color(1, 1, 1),
  sheenRoughness: 0.9,
});
```

### Performance

Sheen activates an additional Ashikhmin BRDF evaluation per fragment. Cost is approximately equivalent to adding one extra specular lobe. No extra render pass or render target required.

---

## Part 2: Subsurface Scattering in Three.js

### The honest answer: native SSS does not exist in core `MeshPhysicalMaterial`

Genuine subsurface scattering (SSS) — where light enters a surface, scatters inside the volume, and exits at a different point — requires either a multi-pass screen-space approach or a volumetric ray-march. Neither is available in Three.js r184's `MeshPhysicalMaterial`.

There are two approaches available:

1. **`thicknessMap` + `attenuationColor` trick** — a static, zero-cost approximation using the transmission volume model.
2. **`MeshSSSNodeMaterial`** — a proper analytical SSS approximation based on a published technique, available in the node material system.

---

### Approach 1: thicknessMap + attenuationColor (MeshPhysicalMaterial)

This is not SSS. It is Beer–Lambert attenuation applied to the transmitted light path. The effect appears similar to SSS for thin-shell objects (ears, fingers, petals) but has no actual light scattering; it is purely colour absorption.

**How to set it up:**

```js
const textureLoader = new THREE.TextureLoader();
const thicknessMap = textureLoader.load('thickness.jpg'); // R channel: thin=dark, thick=bright

const skin = new THREE.MeshPhysicalMaterial({
  color: 0xffe4c4,
  roughness: 0.6,
  metalness: 0.0,
  // Transmission must be > 0 to activate the volume model
  transmission: 0.1,          // subtle — mostly opaque
  thickness: 1.0,             // base thickness (world units)
  thicknessMap: thicknessMap, // per-texel thickness variation
  attenuationColor: new THREE.Color(1.0, 0.3, 0.1), // red-orange scatter tint
  attenuationDistance: 0.08,  // short path = strong colour shift
});
```

**Limitations:**
- Light does not actually scatter inside the mesh; only the view ray is attenuated.
- Back-lit translucency is not reproduced unless `transmission > 0`.
- No wrap-lighting or bleed across shadow terminator.

---

### Approach 2: MeshSSSNodeMaterial (node renderer)

`MeshSSSNodeMaterial` (in `src/materials/nodes/MeshSSSNodeMaterial.js`, r184) extends `MeshPhysicalNodeMaterial` and uses a custom `SSSLightingModel` that extends `PhysicalLightingModel`. It implements the technique from:

> Colin Barré-Brisebois, "Approximating Translucency for a Fast, Cheap and Convincing Subsurface Scattering Look," GDC 2011.

This produces a back-lit wrap and scatter without any extra render passes.

#### How it works

For each direct light, the lighting model:
1. Computes a "scattering half-vector" by blending the light direction with a distorted surface normal.
2. Calculates a view-dependent dot-product falloff, raised to a power and scaled.
3. Multiplies by the `thicknessColorNode` (the SSS tint) and `thicknessAttenuationNode`.
4. Adds the result to the direct diffuse contribution.

This produces a glow on back-lit surfaces that responds to actual light positions.

#### Properties (all are TSL node inputs, not scalar uniforms)

| Property | Type | Default | Purpose |
|----------|------|---------|---------|
| `thicknessColorNode` | `vec3 Node` | `null` | SSS tint colour; must be set |
| `thicknessDistortionNode` | `float Node` | `0.1` | Normal distortion for scatter direction |
| `thicknessAmbientNode` | `float Node` | `0.0` | Ambient SSS contribution |
| `thicknessAttenuationNode` | `float Node` | `0.1` | Light attenuation per scattering event |
| `thicknessPowerNode` | `float Node` | `2.0` | Falloff sharpness (higher = tighter) |
| `thicknessScaleNode` | `float Node` | `10.0` | Overall scatter intensity |

#### SSS recipe (node renderer required)

```js
import { MeshSSSNodeMaterial } from 'three/webgpu';
import { color, float, texture } from 'three/tsl';

const thicknessMap = new THREE.TextureLoader().load('thickness.jpg');

const skin = new MeshSSSNodeMaterial({
  color: 0xffe4c4,
  roughness: 0.6,
});

skin.thicknessColorNode = color(1.0, 0.4, 0.2);      // warm SSS tint
skin.thicknessDistortionNode = float(0.1);
skin.thicknessAmbientNode = float(0.05);
skin.thicknessAttenuationNode = texture(thicknessMap).r;  // per-texel attenuation
skin.thicknessPowerNode = float(3.0);
skin.thicknessScaleNode = float(8.0);
```

**Requirements:** `MeshSSSNodeMaterial` only works with the WebGPU renderer or the WebGL node renderer (`WebGLRenderer` with `WebGPU.compatibilityMode`). It does not work with the legacy `WebGLRenderer` + `MeshPhysicalMaterial` pipeline.

---

## Comparison: The Three Approaches

| Approach | SSS accuracy | Back-lit translucency | Extra passes | Renderer |
|----------|--------------|-----------------------|--------------|----------|
| `attenuationColor` + `thicknessMap` | None (attenuation only) | Partial (requires `transmission > 0`) | None | WebGLRenderer |
| `MeshSSSNodeMaterial` | Analytical approximation (GDC 2011) | Yes | None | WebGPU / Node renderer |
| Screen-space SSS (custom) | Good | Yes | 1–2 extra passes | Any (manual) |

---

## Sources

```yaml
- claim: "sheen default 0, range 0.0–1.0; sheenColor default Color(0,0,0); sheenRoughness default 1"
  source: src/materials/MeshPhysicalMaterial.js (r184) + docs/pages/MeshPhysicalMaterial.html.md (r184)
  tier: 1
  accessed_at_utc: "2026-05-26T00:00:00Z"
  status: verified

- claim: "sheenNode (vec3) and sheenRoughnessNode (float) default null in MeshPhysicalNodeMaterial"
  source: src/materials/nodes/MeshPhysicalNodeMaterial.js (r184)
  tier: 1
  accessed_at_utc: "2026-05-26T00:00:00Z"
  status: verified

- claim: "MeshSSSNodeMaterial extends MeshPhysicalNodeMaterial, uses SSSLightingModel"
  source: src/materials/nodes/MeshSSSNodeMaterial.js (r184)
  tier: 1
  accessed_at_utc: "2026-05-26T00:00:00Z"
  status: verified

- claim: "SSS technique referenced: Barré-Brisebois GDC 2011 'Approximating Translucency'"
  source: src/materials/nodes/MeshSSSNodeMaterial.js (r184) — inline comment
  tier: 1
  accessed_at_utc: "2026-05-26T00:00:00Z"
  status: verified

- claim: "thicknessDistortionNode default 0.1, thicknessAmbientNode default 0.0, thicknessAttenuationNode default 0.1, thicknessPowerNode default 2.0, thicknessScaleNode default 10.0"
  source: src/materials/nodes/MeshSSSNodeMaterial.js (r184)
  tier: 1
  accessed_at_utc: "2026-05-26T00:00:00Z"
  status: verified
```
