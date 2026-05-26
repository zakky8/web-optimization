# Emissive & Bloom — MeshStandardMaterial / MeshPhysicalMaterial

**Source revision:** Three.js r184
**Accessed:** 2026-05-26
**Primary sources:**
- `src/materials/MeshStandardMaterial.js` (r184) — emissive / emissiveIntensity defaults
- `src/materials/Material.js` (r184) — needsUpdate / version
- `src/renderers/WebGLRenderer.js` (r184) — HalfFloatType render target, outputBufferType
- `examples/jsm/postprocessing/UnrealBloomPass.js` (r184) — bloom pass implementation
- `examples/webgl_postprocessing_unreal_bloom_selective.html` (r184) — selective bloom pattern
- `docs/pages/WebGLRenderer.html.md` (r184) — outputBufferType / HalfFloatType

---

## Overview

`MeshStandardMaterial` and `MeshPhysicalMaterial` share the same emissive model: a colour + intensity multiplier that adds to the fragment output independently of lighting. By default `emissive` is black and the material adds nothing. When enabled, emissive makes an object appear to glow regardless of light placement.

HDR emissive (intensity above `1.0`) requires a floating-point render target to avoid clamping; combine with `UnrealBloomPass` for the full glowing corona effect.

---

## Emissive Properties

### `emissive`

| Field | Value |
|-------|-------|
| Type | `Color` |
| Default | `Color(0x000000)` — black (no emission) |

"A solid color unaffected by other lighting." Defines the spectral colour of the emitted light. This does not make the object a light source — other objects are not lit by it. It only affects the material's own pixels.

```js
material.emissive = new THREE.Color(0xff6600); // orange glow
```

An `emissiveMap` (Texture) multiplies per-texel against `emissive × emissiveIntensity`.

### `emissiveIntensity`

| Field | Value |
|-------|-------|
| Type | `Float` |
| Default | `1` |

"Modulates the emissive color." In a standard SDR pipeline (8-bit buffer, no tone mapping) the practical ceiling is `1.0` — higher values saturate to white without visible difference. In an HDR pipeline (see below) values above `1.0` produce sub-pixel fireflies that bloom passes can expand into glowing coronas.

```js
material.emissiveIntensity = 3.0; // HDR — overdriven
material.emissiveIntensity = 8.0; // strong bloom source
```

---

## HDR Emissive: HalfFloatType Render Target

To make `emissiveIntensity > 1.0` meaningful, the renderer's output buffer must be floating-point so values are not clamped to `[0, 1]` before post-processing.

### Setting up the HDR pipeline

```js
import * as THREE from 'three';
import { EffectComposer } from 'three/examples/jsm/postprocessing/EffectComposer.js';
import { RenderPass } from 'three/examples/jsm/postprocessing/RenderPass.js';
import { UnrealBloomPass } from 'three/examples/jsm/postprocessing/UnrealBloomPass.js';

const renderer = new THREE.WebGLRenderer({ antialias: true });

// HalfFloatType render target for the composer
const renderTarget = new THREE.WebGLRenderTarget(
  window.innerWidth,
  window.innerHeight,
  {
    type: THREE.HalfFloatType,  // 16-bit float — preserves >1.0 values
    format: THREE.RGBAFormat,
  }
);

const composer = new EffectComposer(renderer, renderTarget);
composer.addPass(new RenderPass(scene, camera));
composer.addPass(new UnrealBloomPass(
  new THREE.Vector2(window.innerWidth, window.innerHeight),
  1.5,   // strength
  0.4,   // radius
  0.85,  // threshold — only values > 0.85 (in linear space) bloom
));
```

### Why HalfFloatType matters

The `WebGLRenderer.outputBufferType` documentation (r184) confirms: "HDR rendering with tone mapping and post-processing support" is the use case for `HalfFloatType`. Without it, `emissiveIntensity = 3.0` is stored as `1.0` in an 8-bit buffer — the bloom pass sees a solid white pixel and cannot distinguish it from a `1.0`-intensity pixel.

With `HalfFloatType`, the buffer stores `3.0` exactly. The `UnrealBloomPass` threshold filter then extracts values above `threshold` (e.g., `0.85`) and processes them through 5 mip-level Gaussian blurs.

---

## UnrealBloomPass

### Constructor

```js
new UnrealBloomPass(resolution, strength, radius, threshold)
```

| Parameter | Type | Default | Range | Description |
|-----------|------|---------|-------|-------------|
| `resolution` | `Vector2` | — | — | Size of the bloom render targets |
| `strength` | `Float` | `1` | `0 – 3+` | Overall bloom intensity multiplier |
| `radius` | `Float` | — | `0 – 1` | Spread of the bloom (clamped to `[0, 1]`) |
| `threshold` | `Float` | — | `0 – 1` | Luminance threshold: only pixels above this bloom |

### Internal pipeline

The pass uses a five-level mipmap chain:

1. **High-pass filter** — extracts pixels with luminance > `threshold`.
2. **Progressive blur** — each of the 5 mip levels receives a separable Gaussian blur with kernel sizes `[6, 10, 14, 18, 22]` pixels.
3. **Weighted composite** — blends the 5 blurred textures with weights `[1.0, 0.8, 0.6, 0.4, 0.2]` and optional per-level tint colours.
4. **Additive blend** — the composite is additively blended over the original scene.

Processing at multiple downsampled resolutions achieves wide-radius blooms efficiently: the 22-pixel kernel runs at 1/32 resolution.

### Post-creation parameter updates

```js
bloomPass.threshold = 0.5;
bloomPass.strength = 2.0;
bloomPass.radius = 0.5;
// No needsUpdate required for UnrealBloomPass uniform changes
```

---

## Selective Bloom via Layers

`UnrealBloomPass` cannot natively bloom only selected objects (it runs on the full scene texture). The standard workaround uses Three.js `Layers` to separate "bloom objects" from the rest.

### Pattern (from three.js selective bloom example, r184)

```js
const BLOOM_SCENE = 1; // layer index reserved for bloom

// Mark objects that should bloom
sphere.layers.enable(BLOOM_SCENE);  // add to bloom layer
// sphere.layers.disable(BLOOM_SCENE) to remove

// One camera renders only bloom objects
const bloomLayer = new THREE.Layers();
bloomLayer.set(BLOOM_SCENE);
```

**Two-composer technique:**

```js
// bloomComposer — renders only bloom objects
const bloomComposer = new EffectComposer(renderer, renderTarget);
bloomComposer.renderToScreen = false;
bloomComposer.addPass(new RenderPass(scene, camera));
bloomComposer.addPass(bloomPass);

// finalComposer — composites bloom over full scene
const finalComposer = new EffectComposer(renderer);
finalComposer.addPass(new RenderPass(scene, camera));
finalComposer.addPass(finalPass); // custom ShaderPass that mixes bloom texture
```

**Dark material masking:**

Before rendering the bloom pass, non-bloom objects are temporarily swapped to a black `MeshBasicMaterial` so they do not contribute to the luminance extraction:

```js
const darkMaterial = new THREE.MeshBasicMaterial({ color: 'black' });
const materials = {}; // cache original materials

function darkenNonBloomed(obj) {
  if (obj.isMesh && !bloomLayer.test(obj.layers)) {
    materials[obj.uuid] = obj.material;
    obj.material = darkMaterial;
  }
}

function restoreMaterial(obj) {
  if (materials[obj.uuid]) {
    obj.material = materials[obj.uuid];
    delete materials[obj.uuid];
  }
}

function render() {
  scene.traverse(darkenNonBloomed);
  bloomComposer.render();
  scene.traverse(restoreMaterial);
  finalComposer.render();
}
```

`bloomLayer.test(obj.layers)` — the `Layers` class test method returns `true` if the object's layer bitmask overlaps the bloom layer bitmask.

---

## Emissive as Light Source Appearance

Emissive does not emit light in Three.js — it has no effect on other objects in the scene. To create the appearance of a glowing light source:

1. Set the mesh `emissiveIntensity` high enough to bloom.
2. Place an actual `PointLight` or `SpotLight` at the same position with matching colour.
3. The bloom halo is visual; the `PointLight` provides actual illumination.

```js
// Glowing lamp
const bulbMaterial = new THREE.MeshStandardMaterial({
  color: 0xffee88,
  emissive: new THREE.Color(0xffee88),
  emissiveIntensity: 5.0,    // HDR — drives bloom
});

const bulbLight = new THREE.PointLight(0xffee88, 2.0, 10);
bulbLight.position.copy(bulb.position);
scene.add(bulbLight);
```

---

## Performance Notes

| Factor | Cost |
|--------|------|
| `emissive` alone | Free — one extra colour add in fragment shader |
| `emissiveMap` | One additional texture sample per fragment |
| `UnrealBloomPass` | 5 mip-level render passes + 1 composite pass per frame |
| Selective bloom masking | One full-scene material-swap traversal per frame |
| `HalfFloatType` render target | 2× memory vs. 8-bit; negligible performance difference on desktop GPU |

---

## Sources

```yaml
- claim: "emissive default Color(0x000000), emissiveIntensity default 1"
  source: src/materials/MeshStandardMaterial.js (r184)
  tier: 1
  accessed_at_utc: "2026-05-26T00:00:00Z"
  status: verified

- claim: "HalfFloatType render target for HDR output buffer"
  exact_quote: "HDR rendering with tone mapping and post-processing support"
  source: docs/pages/WebGLRenderer.html.md (r184)
  tier: 1
  accessed_at_utc: "2026-05-26T00:00:00Z"
  status: verified

- claim: "UnrealBloomPass uses 5 mip levels, kernel sizes [6,10,14,18,22], blend weights [1.0,0.8,0.6,0.4,0.2]"
  source: examples/jsm/postprocessing/UnrealBloomPass.js (r184)
  tier: 1
  accessed_at_utc: "2026-05-26T00:00:00Z"
  status: verified

- claim: "UnrealBloomPass constructor: strength default 1, radius clamped [0,1], threshold controls luminance extraction"
  source: examples/jsm/postprocessing/UnrealBloomPass.js (r184)
  tier: 1
  accessed_at_utc: "2026-05-26T00:00:00Z"
  status: verified

- claim: "Selective bloom uses BLOOM_SCENE = 1, darkMaterial trick, bloomLayer.test(obj.layers), two-composer approach"
  source: examples/webgl_postprocessing_unreal_bloom_selective.html (r184)
  tier: 1
  accessed_at_utc: "2026-05-26T00:00:00Z"
  status: verified
```
