# Shadow Bias Tuning in Three.js

**Knowledge base entry for:** `web-optimization / 29-shadows-advanced`
**Last verified:** 2026-05-26
**Sources:** Three.js r168 source (`src/lights/LightShadow.js`, `src/renderers/shaders/ShaderChunk/shadowmap_pars_fragment.glsl.js`, `src/renderers/shaders/ShaderChunk/shadowmap_vertex.glsl.js`)

---

## The Two Bias Properties

Every `LightShadow` instance exposes two bias controls:

```js
light.shadow.bias       // default: 0
light.shadow.normalBias // default: 0
```

These are independent mechanisms operating at different pipeline stages. Understanding where each is applied is essential for tuning.

---

## `shadow.bias` — Fragment-Stage Depth Offset

### Where it applies
In the GLSL fragment shader, after the shadow coordinate is projected into shadow map UV space. The bias shifts the fragment's depth value before comparison against the stored depth.

```glsl
// For PCF / Basic (standard depth buffer):
shadowCoord.z += shadowBias;

// For reversed depth buffer (WebGL2 with depth clamping):
#ifdef USE_REVERSED_DEPTH_BUFFER
    shadowCoord.z -= shadowBias;
#else
    shadowCoord.z += shadowBias;
#endif
```

Point lights apply an equivalent offset after converting view-space depth to perspective depth.

### Effect: Shadow Acne (self-shadowing)
When `shadow.bias = 0`, floating-point precision in the depth comparison causes a surface to randomly shadow itself — producing a pattern of dark speckles across lit faces. This is **shadow acne**.

Root cause: The shadow map stores the depth at discrete texel centers. A fragment between texel centers compares its depth against the nearest texel, which may represent a slightly different depth due to surface curvature and the discrete sampling grid.

### Effect: Peter-Panning
Increasing bias too far pushes the shadow receiver's effective depth toward the light until it no longer intersects the stored depth, causing the shadow to detach from the caster's base and appear to float. This is **peter-panning**.

Named after the fictional character — the shadow floats freely from its owner.

### Sign convention
- **Negative bias** (`shadow.bias = -0.0005`): Pushes the fragment's depth closer to the light camera (makes it easier to be shadowed). Reduces acne on receivers facing the light.
- **Positive bias** (`shadow.bias = +0.0005`): Pushes depth away from the light (harder to be shadowed). Can help on back-facing surfaces but increases peter-panning risk.

In practice, negative values in the range `-0.0001` to `-0.001` are the most common starting point for `PCFShadowMap` with a directional light.

---

## `shadow.normalBias` — Vertex-Stage World-Space Offset

### Where it applies
In the GLSL vertex shader, **before** the shadow matrix multiplication. The vertex position is displaced along its world-space normal by `shadowNormalBias` units before the shadow coordinate is computed.

```glsl
// Directional lights:
shadowWorldPosition = worldPosition + vec4(
    shadowWorldNormal * directionalLightShadows[i].shadowNormalBias, 0.0
);
vDirectionalShadowCoord[i] = directionalShadowMatrix[i] * shadowWorldPosition;

// Spot lights (applied when shadow is active):
shadowWorldPosition.xyz += shadowWorldNormal * spotLightShadows[i].shadowNormalBias;

// Point lights: equivalent pattern via pointLightShadows[i].shadowNormalBias
```

### Why this is better than `shadow.bias` for most cases
`shadow.bias` applies a uniform depth offset that is independent of surface orientation. On surfaces that are nearly parallel to the light direction (grazing angle), the projected depth values vary rapidly across shadow map texels, requiring a large `shadow.bias` to compensate — which then causes peter-panning on normal-facing surfaces.

`shadow.normalBias` offsets the shadow receiver outward along its normal before projection. This naturally handles grazing-angle surfaces without needing an extreme uniform depth shift. The offset is in world units, so its effect scales correctly with scene scale.

### Effect on geometry
At high values, `shadow.normalBias` causes shadow contact to be visually incorrect — the shadow edge will float away from the actual geometry silhouette in world space rather than in depth space. The visual artifact is still peter-panning, but it's geometrically more localized than the `shadow.bias` variant.

---

## Interaction Between the Two

They compound. Both are applied independently:
1. Vertex shader: position += normal × `normalBias` (world space)
2. Fragment shader: depth += `bias` (NDC depth space)

Having both non-zero is common:
```js
light.shadow.bias = -0.0001;       // small depth nudge for residual acne
light.shadow.normalBias = 0.02;    // larger geometric offset for angled surfaces
```

---

## Camera `near` and `far` Impact on Bias

This is the most commonly misunderstood factor. Shadow bias operates in the normalized depth range `[0, 1]` (or `[-1, 1]` in reversed buffers). The precision of that range is determined by `shadow.camera.near` and `shadow.camera.far`.

### Depth precision formula
For a standard WebGL depth buffer with a perspective or orthographic shadow camera, the depth precision at distance `z` degrades roughly as:

```
effective_depth_precision ≈ (far - near) / (2^depthBits)
```

For a 24-bit depth buffer:
- `near=0.1, far=1000` → precision ≈ `0.000060` per step near the camera
- `near=1.0, far=100` → precision ≈ `0.000006` per step — **10× better**

### Consequence
A tightly fitted shadow camera (small `far - near` ratio) means:
- Finer depth precision
- Less shadow acne for the same `shadow.bias`
- You can use smaller (less distorting) bias values

### Directional light shadow camera tuning
```js
const d = 10; // scene half-size
light.shadow.camera.left   = -d;
light.shadow.camera.right  =  d;
light.shadow.camera.top    =  d;
light.shadow.camera.bottom = -d;
light.shadow.camera.near   = 0.1;  // as large as possible without clipping near geometry
light.shadow.camera.far    = 50;   // as small as possible, just past the farthest caster
light.shadow.camera.updateProjectionMatrix();
```

Visualize with:
```js
import { CameraHelper } from 'three';
const helper = new CameraHelper(light.shadow.camera);
scene.add(helper);
```

### Spot light
Uses a `PerspectiveCamera`. The `fov` is linked to `light.angle`. Keep `shadow.camera.near` as large as possible to maximize depth precision.

### Point light
Uses a cube camera — only `near` and `far` are configurable. Six render passes per frame. Most expensive shadow type.

---

## Per-Light Bias Settings

Each light has its own independent `shadow` object. Settings do not bleed between lights.

```js
// Directional light — flat terrain, orthographic projection
dirLight.shadow.bias = -0.0001;
dirLight.shadow.normalBias = 0.04;

// Spot light — closer frustum, more depth precision available
spotLight.shadow.bias = -0.00005;
spotLight.shadow.normalBias = 0.02;

// Point light — cube map, grazing angles on all faces
pointLight.shadow.bias = -0.001;     // typically needs more bias due to cube projection
pointLight.shadow.normalBias = 0.05;
```

---

## Practical Tuning Workflow

1. Set `shadow.mapSize` to your final resolution first. Bias interacts with texel density.
2. Start with both at `0`. Observe acne.
3. Increase `shadow.normalBias` in steps of `0.01` until acne on main surfaces disappears.
4. If acne persists on flat/horizontal receivers, add a small negative `shadow.bias` (`-0.0001`).
5. Check for peter-panning: the shadow base should touch geometry. If it floats, reduce both values.
6. Tighten `shadow.camera.near` and `shadow.camera.far` to improve depth precision and revisit.
7. Test at target shadow map resolution — acne reappears when resolution drops.

---

## Shadow Type — Bias Sensitivity Comparison

| Shadow type | Acne without bias | Recommended bias range | normalBias sensitive |
|---|---|---|---|
| BasicShadowMap | Severe | `bias: -0.001` to `-0.0005` | Yes |
| PCFShadowMap | Moderate | `bias: -0.0001` to `-0.00001` | Yes |
| VSMShadowMap | Low (blur masks acne) | `bias: 0` to `0.0001` | Yes (vertex stage still active) |

---

## Sources

```yaml
- claim: "shadow.bias is applied in the fragment shader to shadowCoord.z before depth comparison"
  exact_quote: "shadowCoord.z += shadowBias"
  sources:
    - url: "https://raw.githubusercontent.com/mrdoob/three.js/dev/src/renderers/shaders/ShaderChunk/shadowmap_pars_fragment.glsl.js"
      tier: 1
      accessed_at_utc: "2026-05-26T00:00:00Z"
  status: verified

- claim: "shadow.normalBias offsets vertex position along world normal before shadow matrix multiplication"
  exact_quote: "shadowWorldPosition = worldPosition + vec4( shadowWorldNormal * directionalLightShadows[ i ].shadowNormalBias, 0 )"
  sources:
    - url: "https://raw.githubusercontent.com/mrdoob/three.js/dev/src/renderers/shaders/ShaderChunk/shadowmap_vertex.glsl.js"
      tier: 1
      accessed_at_utc: "2026-05-26T00:00:00Z"
  status: verified

- claim: "LightShadow defaults: bias=0, normalBias=0, radius=1, blurSamples=8, mapSize=512x512"
  sources:
    - url: "https://raw.githubusercontent.com/mrdoob/three.js/dev/src/lights/LightShadow.js"
      tier: 1
      accessed_at_utc: "2026-05-26T00:00:00Z"
  status: verified

- claim: "DirectionalLightShadow uses OrthographicCamera"
  sources:
    - url: "https://threejs.org/docs/#api/en/lights/shadows/DirectionalLightShadow"
      tier: 1
      accessed_at_utc: "2026-05-26T00:00:00Z"
  status: verified
```
