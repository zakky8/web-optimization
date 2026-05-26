# Shadow Map Types in Three.js

**Knowledge base entry for:** `web-optimization / 29-shadows-advanced`
**Last verified:** 2026-05-26
**Sources:** Three.js r168 source (`src/constants.js`, `src/renderers/webgl/WebGLShadowMap.js`, `src/renderers/shaders/ShaderChunk/shadowmap_pars_fragment.glsl.js`)

---

## Overview

Three.js exposes four shadow map algorithms through `renderer.shadowMap.type`. They are defined as integer constants in `src/constants.js`:

| Constant | Integer value | Enum name |
|---|---|---|
| `THREE.BasicShadowMap` | 0 | BasicShadowMap |
| `THREE.PCFShadowMap` | 1 | PCFShadowMap |
| `THREE.PCFSoftShadowMap` | 2 | PCFSoftShadowMap (**deprecated r168**) |
| `THREE.VSMShadowMap` | 3 | VSMShadowMap |

Enable shadow rendering with:

```js
renderer.shadowMap.enabled = true;
renderer.shadowMap.type = THREE.PCFShadowMap; // default
```

---

## 1. BasicShadowMap (`= 0`)

### Algorithm
Hard-edged nearest-filter depth comparison. One depth sample per fragment. No filtering. Renders depth maps with `NearestFilter` on both min and mag.

### Quality
Produces aliased, staircase shadow edges at any resolution below ~4096. Blocky artifacts scale with light distance and camera frustum size.

### Performance
Cheapest possible shadow evaluation — single texture fetch and direct comparison in the fragment shader. Saves ~4–20 texture reads per fragment compared to PCF.

### Bias requirements
Most prone to shadow acne because the raw depth comparison has zero tolerance for floating-point imprecision. A non-zero `shadow.bias` is almost always required.

Typical starting value: `shadow.bias = -0.0005` (negative biases push the shadow surface toward the light, reducing acne on receivers facing the light).

### Use cases
- Pixel-art / retro aesthetics where hard shadows are intentional
- Off-screen or very distant shadow casters where edge quality is invisible
- Mobile / embedded WebGL where fill-rate is critical
- Debug / preview pass where you need maximum shadow throughput

---

## 2. PCFShadowMap (`= 1`) — Default

### Algorithm
Percentage-Closer Filtering. The fragment shader samples the shadow map multiple times around the lookup coordinate and averages the binary pass/fail results, producing a soft penumbra. Three.js r168 uses Vogel disk sampling (5 taps with interleaved gradient noise rotation) that effectively delivers ~20 filtered taps through hardware PCF bilinear interpolation. Depth texture is allocated with `LinearFilter`.

### Quality
Smooth, anti-aliased shadow edges. The softness radius is proportional to `shadow.radius` (default `1`). Higher radius = softer but potentially over-blurred edges. Quality degrades gracefully at lower `mapSize`.

### Performance
5 texture fetches in the shader (hardware bilinear on each = ~20 effective samples). Roughly 3–5× heavier fragment cost than BasicShadowMap. Still GPU-friendly on desktop. Acceptable on mid-tier mobile at `mapSize 512–1024`.

### Bias requirements
Less aggressive bias needed than BasicShadowMap because the averaged samples reduce acne. A small bias is still required for flat receivers.

Typical starting values:
- `shadow.bias = -0.0001`
- `shadow.normalBias = 0.02`

### `shadow.radius`
Controls the spread of the Vogel disk kernel in UV space. Default `1`. Increasing to `2–4` produces softer penumbras at the cost of visible noise at low sample counts.

### Use cases
Standard choice for most WebGL scenes. Default for a reason — best balance of quality and cost.

---

## 3. PCFSoftShadowMap (`= 2`) — DEPRECATED

### Status as of r168
`PCFSoftShadowMap` now triggers a deprecation warning in the console:

```
"PCFSoftShadowMap has been deprecated. Using PCFShadowMap instead."
```

The internal implementation automatically substitutes `PCFShadowMap`. Existing code setting `renderer.shadowMap.type = THREE.PCFSoftShadowMap` continues to work but should be migrated to `THREE.PCFShadowMap`.

### Migration
Replace all instances:
```js
// Before (deprecated)
renderer.shadowMap.type = THREE.PCFSoftShadowMap;

// After
renderer.shadowMap.type = THREE.PCFShadowMap;
// Tune softness via:
light.shadow.radius = 2; // increase for softer edges
```

---

## 4. VSMShadowMap (`= 3`)

### Algorithm
Variance Shadow Maps. Two-pass approach:

1. **Depth pass** — Scene is rendered into an `RGFormat` / `HalfFloatType` render target storing both depth (`r`) and depth-squared (`g`). A native `DepthTexture` is attached with `NearestFilter`.
2. **Blur pass** — Two-pass separable Gaussian blur (vertical then horizontal) is applied to the depth/depth² texture using the `shadow.blurSamples` (default `8`) and `shadow.radius` parameters.
3. **Shadow evaluation** — In the fragment shader, the mean and variance are read from the blurred texture. Chebyshev's inequality computes an upper-bound probability `p_max = variance / (variance + d²)` where `d = depth - mean`. A light-bleeding reduction remap then constrains the result to `[amount, 1]` → `[0, 1]`.

### Quality
Produces soft, naturally blurred shadows that can be filtered with hardware bilinear/trilinear. The Gaussian blur is physically separable, so large soft radii are cheap. Handles complex geometry well in isolation.

**Critical flaw — light bleeding:** When two shadow casters overlap (e.g., a character standing near a wall), VSM produces incorrect bright halos in the shadowed region. The Chebyshev bound underestimates occlusion probability when depth variance is high. This is an intrinsic limitation of the algorithm.

### Performance
More expensive setup than PCF: requires an additional render target allocation (RGFormat HalfFloat), two blur draw calls per shadow map, and a more complex fragment shader. However, the blur kernel cost is O(2N) samples vs. O(N²) for an equivalent full-kernel PCF, making very large soft shadows cheaper than PCF.

### Bias requirements
VSM inherently has different bias sensitivity. The blur smears depth values, reducing the precision of the depth comparison and often reducing acne without bias. However, large variance in the depth texture can introduce its own artifacts at geometry edges.

Key setting: `shadow.bias` affects how the raw depth is offset before the variance comparison. Small values (e.g., `0.0001`) are usually sufficient. `shadow.normalBias` still applies at the vertex stage.

**`shadow.blurSamples`** — Default `8`. Controls the sample count of the separable Gaussian blur passes. Higher values produce smoother soft shadows at increased fragment cost. Range: `1` (hard VSM, not recommended) to `25+` (very soft, expensive).

### Use cases
- Outdoor scenes with a single strong directional sun light
- Scenes where shadow softness matters more than perfect correctness
- Stylized renders where light bleeding is either invisible or acceptable
- Combined with CSM to get high-quality cascade coverage at distance

### Anti-use cases
- Interiors with overlapping furniture casting shadows on each other (light bleeding)
- Scenes with thin geometry (fences, wires) — VSM blur smears depth incorrectly
- Mobile targets — HalfFloatType RG render targets require OES_texture_half_float extension

---

## Quality vs. Performance Summary

| Type | Edge Quality | Soft Shadows | Cost (fragment) | Acne Resistance | Light Bleed |
|---|---|---|---|---|---|
| BasicShadowMap | Blocky/aliased | No | 1× (baseline) | Low | None |
| PCFShadowMap | Smooth | Yes (radius) | 3–5× | Medium | None |
| PCFSoftShadowMap | — | — | Deprecated → PCFShadowMap | — | — |
| VSMShadowMap | Smooth + blurred | Yes (blur kernel) | 4–8× + blur passes | High | Yes |

---

## Renderer Configuration Reference

```js
renderer.shadowMap.enabled = true;
renderer.shadowMap.type = THREE.PCFShadowMap; // default
renderer.shadowMap.autoUpdate = true;         // default: recompute every frame
renderer.shadowMap.needsUpdate = false;       // manual trigger when autoUpdate=false
```

Light-level configuration:
```js
light.castShadow = true;
light.shadow.mapSize.set(2048, 2048); // default 512×512
light.shadow.radius = 2;             // PCF penumbra width / VSM blur radius
light.shadow.blurSamples = 8;        // VSM only — Gaussian samples per pass
light.shadow.bias = -0.0001;
light.shadow.normalBias = 0.02;
```

---

## Sources

```yaml
- claim: "BasicShadowMap=0, PCFShadowMap=1, PCFSoftShadowMap=2, VSMShadowMap=3"
  sources:
    - url: "https://raw.githubusercontent.com/mrdoob/three.js/dev/src/constants.js"
      tier: 1
      accessed_at_utc: "2026-05-26T00:00:00Z"
  status: verified

- claim: "PCFSoftShadowMap deprecated in r168, auto-redirects to PCFShadowMap"
  sources:
    - url: "https://raw.githubusercontent.com/mrdoob/three.js/dev/src/renderers/webgl/WebGLShadowMap.js"
      tier: 1
      accessed_at_utc: "2026-05-26T00:00:00Z"
  status: verified

- claim: "VSMShadowMap uses RGFormat/HalfFloatType render target with two-pass Gaussian blur"
  sources:
    - url: "https://raw.githubusercontent.com/mrdoob/three.js/dev/src/renderers/webgl/WebGLShadowMap.js"
      tier: 1
      accessed_at_utc: "2026-05-26T00:00:00Z"
  status: verified

- claim: "PCF uses Vogel disk sampling with interleaved gradient noise in the fragment shader"
  sources:
    - url: "https://raw.githubusercontent.com/mrdoob/three.js/dev/src/renderers/shaders/ShaderChunk/shadowmap_pars_fragment.glsl.js"
      tier: 1
      accessed_at_utc: "2026-05-26T00:00:00Z"
  status: verified
```
