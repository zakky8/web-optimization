# Baked Shadows in Three.js

**Knowledge base entry for:** `web-optimization / 29-shadows-advanced`
**Last verified:** 2026-05-26
**Sources:** Three.js r168 source (`src/materials/MeshStandardMaterial.js`, `src/renderers/shaders/ShaderChunk/lights_fragment_maps.glsl.js`, `examples/jsm/lights/LightProbeGenerator.js`), Three.js docs (accessed 2026-05-26)

---

## What Baked Shadows Are

Baked shadows pre-compute lighting and shadow information offline (during content creation) and store the result in a texture. At runtime, Three.js multiplies the baked texture against the surface color — zero realtime shadow cost, zero light render passes.

Trade-off: baked shadows are static. A moving object will not cast a correct real-time shadow on a baked surface. Common strategy: bake static geometry (walls, floor, furniture) and overlay a dynamic technique (ContactShadows, CSM, native shadow maps) for characters and interactive objects.

---

## Blender Baking Overview

### Cycles bake pipeline

1. **UV unwrap** — Every mesh that will receive baked shadows needs a dedicated UV channel (UV2, also called the lightmap UV). UVs must not overlap within a single object. Islands should fill as much UV space as possible without overlapping other objects.

2. **Bake target** — In the Shader Editor, add an Image Texture node to each material and create a new image. Name it consistently (`mesh_lightmap`, `floor_baked`, etc.). The node does NOT need to be connected to any socket — it just needs to be selected/active.

3. **Select bake type** — In Render Properties → Bake panel:
   - `Shadow` — Bakes only shadow information (white = lit, dark = shadowed). Fastest. Good for lightmaps.
   - `AO` (Ambient Occlusion) — Bakes screen-space-like contact darkening. No directional light component.
   - `Combined` — Full lighting pass: diffuse, specular, shadows, AO, indirect. Largest textures. Requires all lights to be set up correctly.
   - `Diffuse` — Albedo + lighting without specular. Good for PBR lightmaps.

4. **Bake settings for lightmaps**
   - Output format: `OpenEXR` (32-bit float, `LinearSRGBColorSpace`) for maximum precision, especially if you'll blend at runtime. `PNG` 16-bit for storage-limited cases.
   - Margin: increase (4–16px) to prevent seams at UV island edges.
   - `Selected to Active`: for baking multiple objects onto a single lightmap atlas.

5. **Export** — Save the image. For Three.js, save as PNG (8-bit, sRGB) or EXR (float, linear).

### Bakery (third-party Blender addon)
Bakery by Mr F is a commercial addon that replaces Cycles' bake pipeline with a GPU-accelerated path tracer bake system. Advantages over built-in bake:
- Batch bakes multiple objects to a single atlas automatically
- Produces directional lightmaps (stores light direction in a second texture for specular reconstruction)
- Supports HDR lightmaps natively
- Significantly faster for large scenes

Bakery outputs are compatible with Three.js's `lightMap` property directly — export as PNG or EXR.

---

## UV2 — The Lightmap UV Channel

Three.js uses a second UV set (historically called `uv2`, now accessed as a named attribute) for light maps and ambient occlusion maps.

### Why a separate UV channel
Main UVs (`uv`) are used for albedo and normal maps. They can overlap (tiled textures, mirrored symmetry) because the same texture value applies everywhere. Lightmap UVs must be unique (no overlaps) across the entire mesh so every surface point maps to a unique texel in the baked texture.

### Setting up UV2 in Three.js

**GLB/GLTF loaded models** with a second UV set are automatically available:
```js
const gltf = await loader.loadAsync('model.glb');
// UV2 is preserved if the Blender model has a second UV channel named 'UVMap.001'
// Three.js GLTFLoader maps it to the 'uv2' attribute automatically (r136+)
```

**Manually for procedural geometry:**
```js
const geometry = new THREE.BoxGeometry(1, 1, 1);
// Copy UV0 to UV2 (only appropriate if they happen to be non-overlapping)
geometry.setAttribute('uv2', geometry.getAttribute('uv').clone());
```

**Checking what's present:**
```js
console.log(Object.keys(geometry.attributes));
// ['position', 'normal', 'uv', 'uv2'] if second set is present
```

---

## Three.js `lightMap` and `lightMapIntensity`

Available on: `MeshStandardMaterial`, `MeshPhysicalMaterial`, `MeshLambertMaterial`, `MeshPhongMaterial`, and their node-material variants.

### `lightMap`
**Type:** `THREE.Texture | null`
**Default:** `null`

```
"The light map. Requires a second set of UVs."
— MeshStandardMaterial.js source, r168
```

The texture is sampled using the `uv2` attribute. Its RGB values are multiplied into the surface's indirect diffuse irradiance:

```glsl
// lights_fragment_maps.glsl
vec3 lightMapIrradiance = lightMapTexel.rgb * lightMapIntensity;
irradiance += lightMapIrradiance;
```

The `irradiance` term feeds into the PBR diffuse BRDF. This means the lightmap contribution is physically plausible — it adds to, not replaces, the direct lighting calculation.

### `lightMapIntensity`
**Type:** `float`
**Default:** `1.0`

Described as "Intensity of the baked light." A pure multiplicative scalar. At `0`, the lightmap has no effect. At `2`, the baked contribution is doubled. Use to calibrate the baked brightness relative to any remaining realtime lights.

### Color space
The lightmap texture's `colorSpace` must be set correctly:
```js
lightMapTexture.colorSpace = THREE.LinearSRGBColorSpace; // for EXR/float textures
lightMapTexture.colorSpace = THREE.SRGBColorSpace;       // for PNG (gamma-corrected)
```

Three.js does not automatically detect color space from texture data — incorrect setting causes either washed-out or too-dark lightmaps.

### Full material setup

```js
import { TextureLoader, MeshStandardMaterial, LinearSRGBColorSpace } from 'three';

const loader = new TextureLoader();
const lightMap = await loader.loadAsync('/textures/floor_lightmap.png');
lightMap.colorSpace = LinearSRGBColorSpace;

const material = new MeshStandardMaterial({
  map: colorTexture,
  lightMap: lightMap,
  lightMapIntensity: 1.0,
});
```

For GLTF models, set after load:
```js
gltf.scene.traverse(obj => {
  if (obj.isMesh) {
    obj.material.lightMap = lightMap;
    obj.material.lightMapIntensity = 1.0;
  }
});
```

---

## THREE.LightProbe

A `LightProbe` stores pre-integrated ambient lighting as spherical harmonic (SH) coefficients. It is not a shadow technique per se, but part of the baked lighting workflow: it captures the low-frequency directional character of ambient light at a point in space and feeds it into the PBR shader's ambient term.

### Class

```js
import { LightProbe, LightProbeHelper } from 'three';
```

**Properties:**
- `sh` — `SphericalHarmonics3` — 9 coefficients (L0, L1, L2) × 3 color channels. Encodes the irradiance distribution from all directions.
- `intensity` — `float`, default `1.0` — Multiplier on SH contribution.

### LightProbeGenerator

```js
import { LightProbeGenerator } from 'three/addons/lights/LightProbeGenerator.js';

// From a pre-loaded CubeTexture (environment map):
const lightProbe = LightProbeGenerator.fromCubeTexture(envMap);
scene.add(lightProbe);

// From a live CubeRenderTarget (dynamic capture):
const lightProbe = await LightProbeGenerator.fromCubeRenderTarget(renderer, cubeRenderTarget);
scene.add(lightProbe);
```

`fromCubeTexture`: Synchronous. Iterates all 6 faces, maps each pixel to a unit sphere direction, computes the solid angle weight (`4 / (√lengthSq × lengthSq)`), evaluates 9 SH basis functions per pixel, accumulates weighted RGB contributions, normalizes by `4π / totalWeight`.

`fromCubeRenderTarget`: Async (returns `Promise<LightProbe>`). Same algorithm but reads back from a GPU render target — supports `FloatType`, `HalfFloatType`, `UnsignedByteType`.

### Role in baked lighting workflow

For fully baked scenes:
1. Bake static geometry lightmaps in Blender (see above).
2. Capture environment cube map from a reference camera position.
3. Generate `LightProbe` from the cube map to provide ambient term for dynamic objects.
4. Dynamic objects (characters, moving props) use the `LightProbe` for ambient and a simple runtime shadow (ContactShadows or a single PCF shadow map) for their contact shadow.

```js
const lightProbe = LightProbeGenerator.fromCubeTexture(envTexture);
lightProbe.intensity = 0.8;
scene.add(lightProbe);
```

---

## Baking vs. Runtime: Decision Guide

| Factor | Baked | Runtime |
|---|---|---|
| Objects | Static architecture | Moving objects, characters |
| Light sources | Fixed (sun angle, interior fixtures) | Dynamic (day/night, flashlights) |
| Initial build time | High (minutes to hours in Blender) | None |
| GPU runtime cost | Near-zero (texture sample) | High (shadow passes per light) |
| Storage | +lightmap texture per mesh (0.5–4 MB) | No extra storage |
| Update when scene changes | Re-bake required | Automatic |
| Shadow softness | Arbitrary (Cycles path traced) | Limited by map resolution and PCF radius |

---

## Common Pitfalls

**UV2 missing or incorrect**
The most common bug. Symptoms: lightmap shows as black or not applied. Check `geometry.attributes.uv2` exists. GLTFLoader requires the Blender model to have a second UV channel named `UVMap.001` or similar.

**Color space mismatch**
EXR lightmaps loaded as sRGB appear burned out. PNG lightmaps loaded as Linear appear too dark. Always set `texture.colorSpace` explicitly.

**LightMap adds to, not replaces, realtime light**
If ambient lights or `HemisphereLight` are in the scene, the lightmap contribution compounds with them. Disable or reduce realtime ambient light when using a full-color combined bake to avoid double-brightness.

**`lightMapIntensity` vs. material `envMapIntensity`**
These are independent. Both can cause unexpected brightness if not calibrated against each other.

---

## Sources

```yaml
- claim: "lightMap requires a second set of UVs; default null"
  exact_quote: "The light map. Requires a second set of UVs."
  sources:
    - url: "https://raw.githubusercontent.com/mrdoob/three.js/dev/src/materials/MeshStandardMaterial.js"
      tier: 1
      accessed_at_utc: "2026-05-26T00:00:00Z"
  status: verified

- claim: "lightMapIntensity default is 1.0; description is 'Intensity of the baked light.'"
  sources:
    - url: "https://raw.githubusercontent.com/mrdoob/three.js/dev/src/materials/MeshStandardMaterial.js"
      tier: 1
      accessed_at_utc: "2026-05-26T00:00:00Z"
  status: verified

- claim: "Shader applies lightmap as: vec3 lightMapIrradiance = lightMapTexel.rgb * lightMapIntensity; irradiance += lightMapIrradiance;"
  exact_quote: "vec3 lightMapIrradiance = lightMapTexel.rgb * lightMapIntensity;"
  sources:
    - url: "https://raw.githubusercontent.com/mrdoob/three.js/dev/src/renderers/shaders/ShaderChunk/lights_fragment_maps.glsl.js"
      tier: 1
      accessed_at_utc: "2026-05-26T00:00:00Z"
  status: verified

- claim: "LightProbeGenerator.fromCubeTexture normalizes by 4pi/totalWeight and uses solid angle weight 4/(sqrt(lengthSq)*lengthSq)"
  sources:
    - url: "https://raw.githubusercontent.com/mrdoob/three.js/dev/examples/jsm/lights/LightProbeGenerator.js"
      tier: 1
      accessed_at_utc: "2026-05-26T00:00:00Z"
  status: verified
```
