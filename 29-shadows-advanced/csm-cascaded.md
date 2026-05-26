# Cascaded Shadow Maps in Three.js

**Knowledge base entry for:** `web-optimization / 29-shadows-advanced`
**Last verified:** 2026-05-26
**Sources:** Three.js r168 `examples/jsm/csm/CSM.js` (built-in addon), Microsoft DirectX CSM article (https://learn.microsoft.com/en-us/windows/win32/dxtecharts/cascaded-shadow-maps, accessed 2026-05-26)

> **Note on `nicktindall/three-csm`:** The repository referenced in the task prompt (`https://github.com/nicktindall/three-csm`) returned HTTP 404 on 2026-05-26. The nicktindall GitHub user exists but three-csm is not present. The Three.js project ships its own CSM addon at `examples/jsm/csm/CSM.js` as of r151+. A separate package named `three-csm` by `strandedkitty` was historically available on GitHub (`github.com/strandedkitty/three-csm`) and is the origin of the Three.js addon. Both implementations share the same architecture.

---

## Why CSM Exists: The Perspective Aliasing Problem

A single shadow map covering an entire camera frustum wastes resolution. Objects near the camera require high shadow-map density (many texels per world unit), while distant objects need far fewer. A fixed shadow camera covering the full scene frustum allocates equally to near and far — the near region is severely under-sampled, producing blocky aliased shadows on close geometry regardless of map resolution.

CSM partitions the camera frustum into N sub-frusta (cascades). Each cascade gets its own shadow map, tightly fitted to its sub-frustum. Near cascades cover less world area at higher effective texel density. Far cascades cover more area at lower density — acceptable because distant shadow detail is less perceptible.

Result: smooth near shadows without prohibitive memory cost.

---

## Three.js Built-in CSM Addon

### Import

```js
import { CSM } from 'three/addons/csm/CSM.js';
import { CSMHelper } from 'three/addons/csm/CSMHelper.js'; // optional visualizer
```

Or via npm:
```js
import { CSM } from 'three/examples/jsm/csm/CSM.js';
```

### Constructor

```js
const csm = new CSM({
  camera,                  // required — THREE.PerspectiveCamera
  parent,                  // required — usually scene

  cascades: 3,             // number of cascade levels (default: 3)
  maxFar: 100000,          // cap on shadow distance, clamped to camera.far (default: 100000)

  mode: 'practical',       // split algorithm: 'uniform' | 'logarithmic' | 'practical' | 'custom'
  fade: false,             // blend between cascades to hide seams (default: false)

  shadowMapSize: 2048,     // shadow texture resolution per cascade (default: 2048)
  shadowBias: 0.000001,    // shadow.bias applied to each cascade light

  lightDirection: new THREE.Vector3(1, -1, 1).normalize(), // directional light angle
  lightIntensity: 3,       // light intensity
  lightNear: 1,            // cascade lights' near plane (default: 1)
  lightFar: 2000,          // cascade lights' far plane (default: 2000)
  lightMargin: 200,        // extra padding on shadow camera extents (default: 200)

  customSplitsCallback: undefined, // function(cascades, near, far, breaks) for mode:'custom'
});
```

### What the constructor creates
For each cascade, CSM creates one `THREE.DirectionalLight` and adds it to `parent`. Each light's shadow camera is computed to tightly cover its sub-frustum slice. This means `cascades=3` adds 3 directional lights to your scene — budget accordingly.

---

## Update Loop

`csm.update()` **must** be called every frame before `renderer.render()`. It recomputes cascade frustum bounds based on the camera's current position and orientation.

```js
function animate() {
  requestAnimationFrame(animate);
  csm.update();               // recalculate cascade splits each frame
  renderer.render(scene, camera);
}
```

Call `csm.updateFrustums()` whenever the camera's `near`, `far`, or `fov` changes (e.g., on window resize):
```js
window.addEventListener('resize', () => {
  camera.aspect = window.innerWidth / window.innerHeight;
  camera.updateProjectionMatrix();
  csm.updateFrustums();
});
```

---

## Material Setup

Materials must be registered with CSM to receive the custom shadow shader injected via `onBeforeCompile`. Call `setupMaterial` for every material that should receive CSM shadows:

```js
const material = new THREE.MeshStandardMaterial({ color: 0xffffff });
csm.setupMaterial(material);
```

`setupMaterial` injects `#define USE_CSM 1` and `#define CSM_CASCADES N` into the material's shader via `material.onBeforeCompile`. It also sets the `CSM_cascades` and `cameraNear`/`shadowFar` uniforms for cascade selection.

Without calling `setupMaterial`, objects will still cast shadows but cascade selection in the fragment shader will be incorrect.

---

## Cascade Split Modes

### `'uniform'`
Equal depth intervals along the camera Z axis. Simple but wasteful — near cascade covers a large depth range when the scene has significant Z extent.

```
near=0.1, far=100, cascades=4:
  cascade 0: [0.1,  25.1]
  cascade 1: [25.1, 50.1]
  cascade 2: [50.1, 75.1]
  cascade 3: [75.1, 100]
```

### `'logarithmic'`
Exponential distribution. Near cascade is very thin; each subsequent cascade roughly doubles in depth range. Favors near-camera quality heavily.

```
λ = 1.0 (pure log):
  split_i = near × (far/near)^(i/N)
```

Best for outdoor scenes with deep Z range (hundreds of meters).

### `'practical'` (default)
Blends uniform and logarithmic using a 50/50 lerp (`λ = 0.5` in Engel's notation). Provides good near-camera quality without completely wasting the far cascades. Most scenes should start here.

```
split_i = λ × log_split_i + (1-λ) × uniform_split_i
```

### `'custom'`
Provide your own split callback:
```js
csm = new CSM({
  mode: 'custom',
  customSplitsCallback(cascades, near, far, breaks) {
    // breaks is an array; push (cascades - 1) normalized values [0..1]
    // representing split positions between near and far
    breaks.push(0.05);  // 5% of range = first split
    breaks.push(0.2);   // 20%
    // last cascade implicitly ends at far
  }
});
```

---

## `fade` Parameter

**Default:** `false`

When `true`, the shadow extents of each cascade are expanded by a margin computed as:

```
linearDepth = frustum.vertices.far[0].z / (far - camera.near)
margin = 0.25 × linearDepth² × (far - near)
```

This creates an overlap zone between adjacent cascades. The fragment shader linearly blends between the two cascade shadow samples in the overlap zone, eliminating the visible seam that otherwise appears at cascade boundaries (especially noticeable when the shadow map type is BasicShadowMap or the cascade count is low).

Cost: each fragment in the blend zone performs two shadow map lookups instead of one.

```js
const csm = new CSM({
  fade: true,
  cascades: 3,
  mode: 'practical',
  // ...
});
```

---

## Number of Cascades — Guidelines

| Scene type | Recommended cascades | Notes |
|---|---|---|
| Indoor / small area | 1–2 | Single cascade may suffice if scene Z < 20m |
| Outdoor mid-scale | 3 (default) | Good balance, standard choice |
| Open world / large terrain | 4 | Diminishing returns beyond 4 for most hardware |
| Mobile WebGL | 2 | Each cascade = one directional light + shadow pass |

Each additional cascade adds:
- 1 directional light render pass per frame
- `shadowMapSize × shadowMapSize × 4 bytes` (for 32-bit depth) of GPU memory

---

## Debug Visualization

```js
const helper = new CSMHelper(csm);
scene.add(helper);

// In animation loop:
helper.update();
```

The `CSMHelper` renders colored frustum slices for each cascade, making split planes and coverage visible.

---

## Complete Minimal Setup

```js
import { CSM } from 'three/addons/csm/CSM.js';

const renderer = new THREE.WebGLRenderer();
renderer.shadowMap.enabled = true;
renderer.shadowMap.type = THREE.PCFShadowMap;

const csm = new CSM({
  camera,
  parent: scene,
  cascades: 3,
  mode: 'practical',
  fade: true,
  shadowMapSize: 2048,
  shadowBias: -0.0001,
  lightDirection: new THREE.Vector3(-1, -1, -1).normalize(),
  lightIntensity: 2,
  maxFar: 200,
});

// Register all shadow-receiving materials
scene.traverse(obj => {
  if (obj.isMesh) csm.setupMaterial(obj.material);
});

function animate() {
  requestAnimationFrame(animate);
  csm.update();
  renderer.render(scene, camera);
}
```

---

## Limitations

- All CSM directional lights are treated as parallel (directional), so point or spot light CSM is not supported by this addon.
- `setupMaterial` modifies the material's `onBeforeCompile` callback — incompatible with materials that already use custom `onBeforeCompile` unless you chain carefully.
- Each cascade light contributes to Three.js's light count limit (`WebGL2: up to 4 directional shadow lights` by default in the PBR shader). For `cascades=4`, ensure the scene has no other shadow-casting directional lights.

---

## Sources

```yaml
- claim: "CSM constructor params: camera, parent, cascades=3, maxFar=100000, mode='practical', fade=false, shadowMapSize=2048, shadowBias=0.000001, lightDirection, lightIntensity=3, lightNear=1, lightFar=2000, lightMargin=200"
  sources:
    - url: "https://raw.githubusercontent.com/mrdoob/three.js/dev/examples/jsm/csm/CSM.js"
      tier: 1
      accessed_at_utc: "2026-05-26T00:00:00Z"
      quote_location: "CSM constructor data destructuring"
  status: verified

- claim: "fade formula: 0.25 * pow(linearDepth, 2.0) * (far - near)"
  sources:
    - url: "https://raw.githubusercontent.com/mrdoob/three.js/dev/examples/jsm/csm/CSM.js"
      tier: 1
      accessed_at_utc: "2026-05-26T00:00:00Z"
  status: verified

- claim: "split modes: uniform, logarithmic, practical (blend of uniform+log), custom"
  sources:
    - url: "https://raw.githubusercontent.com/mrdoob/three.js/dev/examples/jsm/csm/CSM.js"
      tier: 1
      accessed_at_utc: "2026-05-26T00:00:00Z"
  status: verified

- claim: "nicktindall/three-csm GitHub repo returns 404 as of 2026-05-26"
  sources:
    - url: "https://github.com/nicktindall/three-csm"
      tier: 1
      accessed_at_utc: "2026-05-26T00:00:00Z"
  status: verified
  notes: "Task prompt specified https://github.com/nicktindall/three-csm — repo does not exist. Documentation is based on the Three.js built-in CSM addon which shares the same design."

- claim: "CSM concept and split scheme formulas (uniform, log, practical)"
  sources:
    - url: "https://learn.microsoft.com/en-us/windows/win32/dxtecharts/cascaded-shadow-maps"
      tier: 3
      accessed_at_utc: "2026-05-26T00:00:00Z"
  status: verified
```
