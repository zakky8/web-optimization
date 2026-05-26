# Iridescence & Clearcoat — MeshPhysicalMaterial

**Source revision:** Three.js r184
**Accessed:** 2026-05-26
**Primary sources:**
- `src/materials/MeshPhysicalMaterial.js` (r184) — property defaults
- `src/materials/nodes/MeshPhysicalNodeMaterial.js` (r184) — node override properties
- `docs/pages/MeshPhysicalMaterial.html.md` (r184) — official property descriptions

---

## Overview

Iridescence and clearcoat are two layered-material features in `MeshPhysicalMaterial` that together can produce car paint, CD/soap-bubble, and coated-lens appearances. Both features are additive layers computed on top of the base PBR BRDF and do not replace it.

---

## Iridescence

### What it does

Iridescence (thin-film interference) simulates the angle-dependent spectral color shift seen on soap bubbles, CDs, hummingbird feathers, and automotive flake paint. As the viewing angle changes, the perceived color cycles through the visible spectrum. The effect is modelled using the KHR_materials_iridescence glTF extension formulation.

### Properties

#### `iridescence`

| Field | Value |
|-------|-------|
| Type | `Float` |
| Default | `0` |
| Range | `0.0 – 1.0` |

"The intensity of the iridescence layer, simulating RGB color shift based on angle." At `0` there is no effect. At `1.0` the thin-film layer is fully visible.

```js
material.iridescence = 1.0;
```

#### `iridescenceIOR`

| Field | Value |
|-------|-------|
| Type | `Float` |
| Default | `1.3` |
| Range | `1.0 – 2.333` |

Index of refraction of the thin iridescent coating. Controls the strength and spectral distribution of the color shift. Higher values produce more compressed fringes (more rainbow cycles visible).

```js
material.iridescenceIOR = 1.8; // pronounced rainbow
```

#### `iridescenceThicknessRange`

| Field | Value |
|-------|-------|
| Type | `Array` |
| Default | `[100, 400]` |
| Units | Nanometres (nm) |

Two-element array `[min, max]` specifying the physical thickness of the thin film. The renderer maps 0–1 of `iridescenceThicknessMap` (R channel) to this range at each texel. Where no map is provided, the material uses the `max` value uniformly.

Reference thicknesses for known effects:

| Effect | Approximate range (nm) |
|--------|----------------------|
| Soap bubble iridescence | 100 – 800 |
| CD/DVD surface | 120 – 500 |
| Oil slick on water | 50 – 600 |
| Automotive flake paint | 200 – 1000 |

```js
material.iridescenceThicknessRange = [200, 800]; // wider color sweep
```

#### `iridescenceThicknessMap` (Texture)

R channel encodes thickness as a fraction of the `iridescenceThicknessRange`, allowing spatial variation — e.g., varying rainbow bands across a surface.

### Node material override

In `MeshPhysicalNodeMaterial`, all three values can be driven by TSL nodes:

```js
// MeshPhysicalNodeMaterial
material.iridescenceNode = tslFloatNode;           // replaces iridescence
material.iridescenceIORNode = tslFloatNode;        // replaces iridescenceIOR
material.iridescenceThicknessNode = tslFloatNode;  // replaces thickness lookup
```

All default to `null` (falls back to uniform values).

### Layered appearance: CD / car paint

For a CD effect:
```js
const material = new THREE.MeshPhysicalMaterial({
  color: 0x111111,        // near-black base
  metalness: 0.9,
  roughness: 0.05,
  iridescence: 1.0,
  iridescenceIOR: 1.5,
  iridescenceThicknessRange: [100, 500],
});
```

For automotive flake paint (layered with clearcoat):
```js
const material = new THREE.MeshPhysicalMaterial({
  color: 0x1a1a2e,        // deep blue base
  metalness: 0.6,
  roughness: 0.3,
  iridescence: 0.7,
  iridescenceIOR: 1.8,
  iridescenceThicknessRange: [300, 900],
  clearcoat: 1.0,
  clearcoatRoughness: 0.05,
});
```

---

## Clearcoat

### What it does

Clearcoat models a thin transparent lacquer layer applied over the base material — the defining feature of automotive paint, lacquered wood, and some plastics. It adds a second specular lobe computed with its own roughness and normal, composited over the base BRDF. The formulation follows the KHR_materials_clearcoat glTF extension.

### Properties

#### `clearcoat`

| Field | Value |
|-------|-------|
| Type | `Float` |
| Default | `0` |
| Range | `0.0 – 1.0` |

"Represents the intensity of the clear coat layer." At `0` there is no coating. At `1.0` a fully specular lacquer layer is present.

```js
material.clearcoat = 1.0;
```

A `clearcoatMap` (Texture, R channel) multiplies against `clearcoat` per-texel, useful for worn paint with patchy coating.

#### `clearcoatRoughness`

| Field | Value |
|-------|-------|
| Type | `Float` |
| Default | `0` |
| Range | `0.0 – 1.0` |

Roughness of the clearcoat layer, independent of the base `roughness`. A fresh car has `clearcoatRoughness ≈ 0.05`; weathered paint approaches `0.3+`. A `clearcoatRoughnessMap` (Texture, G channel) provides per-texel roughness.

```js
material.clearcoatRoughness = 0.05; // fresh lacquer
```

#### `clearcoatNormalMap` (Texture)

Applies a separate normal map to the clearcoat layer only, leaving the base surface normals unchanged. Use case: orange-peel texture visible in the lacquer that does not exist in the geometry normals.

```js
material.clearcoatNormalMap = loader.load('clearcoat_normal.jpg');
material.clearcoatNormalScale = new THREE.Vector2(0.3, 0.3);
```

`clearcoatNormalScale` (default `Vector2(1, 1)`) controls the normal map intensity independently from `normalScale`.

### Node material override

```js
// MeshPhysicalNodeMaterial
material.clearcoatNode = tslFloatNode;           // replaces clearcoat
material.clearcoatRoughnessNode = tslFloatNode;  // replaces clearcoatRoughness
material.clearcoatNormalNode = tslVec3Node;      // replaces normal lookup
```

---

## Layered Material Appearance — How the Shader Composes

The physical shader evaluates layers from bottom to top:

1. **Base BRDF** — diffuse + specular from `color`, `metalness`, `roughness`, `normalMap`
2. **Sheen** (if `sheen > 0`) — added on top of base
3. **Iridescence** (if `iridescence > 0`) — modulates the base specular with thin-film spectral weighting
4. **Clearcoat** (if `clearcoat > 0`) — second specular lobe composited with Fresnel blend over all layers below

Each layer can be independently textured and node-driven. They are not mutually exclusive — a car-paint material legitimately uses all four simultaneously.

---

## Performance Notes

- `iridescence > 0` activates the thin-film Fresnel calculation, which involves a per-fragment spectral integration approximation. Negligible cost on modern hardware; no extra render pass.
- `clearcoat > 0` adds a second GGX specular lobe per fragment. Cost roughly doubles specular evaluation time; still a fragment-shader cost, no extra pass.
- `clearcoatNormalMap` adds a texture sample per fragment.
- Neither feature requires an additional render target.

---

## Minimal Code Example

```js
import * as THREE from 'three';

// Soap bubble
const bubble = new THREE.MeshPhysicalMaterial({
  color: 0xffffff,
  roughness: 0.0,
  metalness: 0.0,
  transmission: 0.95,
  thickness: 0.01,
  ior: 1.33,
  iridescence: 1.0,
  iridescenceIOR: 1.4,
  iridescenceThicknessRange: [100, 600],
});

// Car paint
const carPaint = new THREE.MeshPhysicalMaterial({
  color: 0x8b0000,
  roughness: 0.4,
  metalness: 0.2,
  iridescence: 0.5,
  iridescenceIOR: 1.6,
  iridescenceThicknessRange: [250, 700],
  clearcoat: 1.0,
  clearcoatRoughness: 0.04,
});
```

---

## Sources

```yaml
- claim: "iridescence default 0, range 0.0–1.0"
  exact_quote: "The intensity of the iridescence layer, simulating RGB color shift based on angle"
  source: docs/pages/MeshPhysicalMaterial.html.md (r184)
  tier: 1
  accessed_at_utc: "2026-05-26T00:00:00Z"
  status: verified

- claim: "iridescenceIOR default 1.3, range 1.0–2.333"
  source: src/materials/MeshPhysicalMaterial.js (r184) + docs/pages/MeshPhysicalMaterial.html.md (r184)
  tier: 1
  accessed_at_utc: "2026-05-26T00:00:00Z"
  status: verified

- claim: "iridescenceThicknessRange default [100, 400], units nanometres"
  source: src/materials/MeshPhysicalMaterial.js (r184) + docs/pages/MeshPhysicalMaterial.html.md (r184)
  tier: 1
  accessed_at_utc: "2026-05-26T00:00:00Z"
  status: verified

- claim: "clearcoat default 0, range 0.0–1.0"
  exact_quote: "Represents the intensity of the clear coat layer"
  source: docs/pages/MeshPhysicalMaterial.html.md (r184)
  tier: 1
  accessed_at_utc: "2026-05-26T00:00:00Z"
  status: verified

- claim: "clearcoatRoughness default 0"
  source: src/materials/MeshPhysicalMaterial.js (r184)
  tier: 1
  accessed_at_utc: "2026-05-26T00:00:00Z"
  status: verified

- claim: "iridescenceNode, iridescenceIORNode, iridescenceThicknessNode default null in MeshPhysicalNodeMaterial"
  source: src/materials/nodes/MeshPhysicalNodeMaterial.js (r184)
  tier: 1
  accessed_at_utc: "2026-05-26T00:00:00Z"
  status: verified

- claim: "clearcoatNode, clearcoatRoughnessNode, clearcoatNormalNode default null in MeshPhysicalNodeMaterial"
  source: src/materials/nodes/MeshPhysicalNodeMaterial.js (r184)
  tier: 1
  accessed_at_utc: "2026-05-26T00:00:00Z"
  status: verified
```
