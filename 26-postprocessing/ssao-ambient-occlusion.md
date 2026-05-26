# SSAO & Ambient Occlusion

> Sources verified 2026-05-26 against:
> - `https://raw.githubusercontent.com/mrdoob/three.js/dev/examples/jsm/postprocessing/SSAOPass.js`
> - `https://raw.githubusercontent.com/mrdoob/three.js/dev/examples/jsm/postprocessing/GTAOPass.js`
> - `https://raw.githubusercontent.com/pmndrs/postprocessing/main/src/effects/SSAOEffect.js`
> - Three.js r184 release tag

---

## What SSAO Does

Screen Space Ambient Occlusion darkens crevices, contact points, and concave
areas where ambient light is geometrically blocked. It is computed entirely from
the depth and normal buffers already on the GPU — no shadow maps, no geometry
passes. The trade-off is that occluders invisible to the camera do not contribute,
and the effect can ring or halo at silhouettes.

Three.js ships two SSAO implementations:

| Pass | Quality | Cost | Notes |
|------|---------|------|-------|
| `SSAOPass` | Medium | Low–Medium | Simplex noise kernel, built-in blur |
| `GTAOPass` | High | Medium–High | Ground Truth AO with Poisson denoising; added r161 |

---

## SSAOPass

### Import

```js
import { SSAOPass } from 'three/addons/postprocessing/SSAOPass.js';
```

### Constructor

```js
const ssaoPass = new SSAOPass(
  scene,           // THREE.Scene
  camera,          // THREE.Camera
  width,           // number — effect resolution width  (default: 512)
  height,          // number — effect resolution height (default: 512)
  kernelSize        // number — samples per pixel       (default: 32)
);
```

Resolution does not need to match the canvas. Running at 50% resolution
(divide canvas dimensions by 2) saves ~75% of fragment work with a modest
quality loss. Upscale at the blend step is usually invisible.

### Parameters

| Property | Type | Default | Practical range | Effect |
|----------|------|---------|-----------------|--------|
| `kernelRadius` | number | 8 | 1–32 | World-space radius of the AO hemisphere. Too small: misses broad contact shadows. Too large: the hemisphere clips through walls and causes light-leaking. |
| `minDistance` | number | 0.005 | 0.001–0.05 | Minimum depth delta for a sample to count as occluder. Prevents self-occlusion on flat surfaces. |
| `maxDistance` | number | 0.1 | 0.01–0.5 | Maximum depth delta beyond which a sample is ignored. Keeps the effect local; prevents distant geometry occluding the foreground. |
| `output` | number | 0 | see table | Debug visualisation mode |

### Output modes

```js
import { SSAOPass } from 'three/addons/postprocessing/SSAOPass.js';

ssaoPass.output = SSAOPass.OUTPUT.Default;    // 0 — blurred AO composited with scene
ssaoPass.output = SSAOPass.OUTPUT.SSAO;       // 1 — raw AO texture (white = lit, black = occluded)
ssaoPass.output = SSAOPass.OUTPUT.Blur;       // 2 — blurred AO only
ssaoPass.output = SSAOPass.OUTPUT.Depth;      // 3 — depth buffer visualisation
ssaoPass.output = SSAOPass.OUTPUT.Normal;     // 4 — normal buffer visualisation
```

`SSAO` and `Normal` output modes are invaluable for diagnosing halos, incorrect
normals from flat-shaded geometry, and kernel-radius tuning.

### Minimal setup

```js
import { EffectComposer } from 'three/addons/postprocessing/EffectComposer.js';
import { RenderPass }     from 'three/addons/postprocessing/RenderPass.js';
import { SSAOPass }       from 'three/addons/postprocessing/SSAOPass.js';
import { OutputPass }     from 'three/addons/postprocessing/OutputPass.js';

const composer = new EffectComposer( renderer );
composer.addPass( new RenderPass( scene, camera ) );

const ssaoPass = new SSAOPass( scene, camera, window.innerWidth, window.innerHeight );
ssaoPass.kernelRadius = 16;
ssaoPass.minDistance  = 0.002;
ssaoPass.maxDistance  = 0.08;
composer.addPass( ssaoPass );

composer.addPass( new OutputPass() );
```

### What SSAOPass cannot handle

The documentation states SSAOPass "honors only meshes — points and lines do not
contribute to SSAO." This means point clouds or line-based geometry will not
cast or receive AO from the SSAO pass.

During normal/depth pre-pass, SSAOPass temporarily forces mesh visibility,
which means transparent objects without depth writes may cause artefacts.

---

## GTAOPass — Ground Truth Ambient Occlusion

GTAOPass was added in Three.js r161. The docs state: *"GTAOPass provides better
quality than SSAOPass but is also more expensive."*

### Import

```js
import { GTAOPass } from 'three/addons/postprocessing/GTAOPass.js';
```

### Constructor

```js
const gtaoPass = new GTAOPass(
  scene,
  camera,
  width,         // default: 512
  height,        // default: 512
  parameters,    // { depthTexture?, normalTexture? } — optional external GBuffer
  aoParameters,  // GTAO shader config (see below)
  pdParameters   // Poisson Denoise config (see below)
);
```

### aoParameters

| Property | Default | Notes |
|----------|---------|-------|
| `radius` | — | Sample radius for hemisphere |
| `distanceExponent` | — | Falloff curve steepness |
| `thickness` | — | Assumed surface thickness; affects backface occlusion |
| `distanceFallOff` | — | Distance-based attenuation |
| `scale` | — | AO intensity multiplier |
| `samples` | — | Sample count — **changes trigger shader recompile** |
| `screenSpaceRadius` | `false` | `true` = radius in screen pixels; `false` = world units |

### pdParameters (Poisson Denoise)

| Property | Default | Notes |
|----------|---------|-------|
| `pdRings` | 2 | Poisson ring count |
| `pdRadiusExponent` | 2 | Radius scaling per ring |
| `pdSamples` | 16 | Samples per denoise pass |
| `lumaPhi` | — | Luminance weight for bilateral filter |
| `depthPhi` | — | Depth weight |
| `normalPhi` | — | Normal weight |
| `blendIntensity` | 1.0 | Final AO mix strength |

The Poisson denoiser is what separates GTAOPass visually from SSAOPass. Instead
of a simple box/Gaussian blur, it uses bilateral filtering weighted by depth and
normals, which preserves sharp contact edges while smoothing noisy AO in open
areas.

### GTAOPass output modes

```js
gtaoPass.output = GTAOPass.OUTPUT.Default;    // composited result
gtaoPass.output = GTAOPass.OUTPUT.AO;         // raw AO
gtaoPass.output = GTAOPass.OUTPUT.Denoise;    // denoised AO only
gtaoPass.output = GTAOPass.OUTPUT.Normal;     // normal buffer
gtaoPass.output = GTAOPass.OUTPUT.Depth;      // depth buffer
```

### Scene clip box

GTAOPass supports a `clipBox` (`THREE.Box3`) to restrict AO computation to a
region of the scene:

```js
gtaoPass.clipBox.setFromObject( targetMesh );
```

---

## Tips for Reducing Noise

SSAO noise is the most common complaint in production. Ranked by effectiveness:

### 1. Increase `kernelSize` (SSAOPass) or `samples` (GTAOPass)

More samples per pixel = less variance per pixel. Going from 32 to 64 samples
halves the noise standard deviation but doubles GPU cost. Usually 32–48 is a
good ceiling before switching to a denoised approach like GTAOPass.

### 2. Tune `minDistance` / `maxDistance` carefully

Most "sparkle" noise on flat surfaces comes from `minDistance` being too low.
Start at `0.005` and raise it until flat-surface noise disappears. If you have
thin geometry (paper, grass cards), do not raise it too far or the thin surfaces
stop self-occluding.

### 3. Use half-resolution with blur

```js
// Construct at half resolution
const w = window.innerWidth  / 2;
const h = window.innerHeight / 2;
const ssaoPass = new SSAOPass( scene, camera, w, h );
```

The built-in blur in SSAOPass runs on the half-res result. The upscale blur acts
as a free denoiser. This halves GPU cost with minimal perceptual quality loss.

### 4. Switch to GTAOPass for production quality

GTAOPass's Poisson denoise pass with `lumaPhi / depthPhi / normalPhi` weighting
produces significantly cleaner results at equal sample counts. Its
`blendIntensity` knob lets you dial in subtlety without changing the core parameters.

### 5. Temporal accumulation (manual — not built-in)

Neither Three.js pass includes TAA for AO. If you are already running
`TemporalAAPass` or `SAOPass`-based TAA, you can pipe the AO texture through
the accumulator. This is an advanced topic beyond the scope of the built-in passes.

---

## pmndrs/postprocessing SSAOEffect

```js
import { EffectComposer, EffectPass, RenderPass, SSAOEffect, NormalPass } from 'postprocessing';

const normalPass = new NormalPass( scene, camera );
const ssaoEffect = new SSAOEffect( camera, normalPass.texture, {
  blendFunction:    BlendFunction.MULTIPLY,
  samples:          9,
  rings:            7,
  radius:           0.1825,
  intensity:        1.0,
  bias:             0.025,
  fade:             0.01,
  luminanceInfluence: 0.7,
  worldDistanceThreshold:  20,
  worldDistanceFalloff:    5,
  worldProximityThreshold: 0.4,
  worldProximityFalloff:   0.1,
  resolutionScale: 0.5,   // render AO at half res
  depthAwareUpsampling: true,
} );

const composer = new EffectComposer( renderer, { frameBufferType: HalfFloatType } );
composer.addPass( new RenderPass( scene, camera ) );
composer.addPass( normalPass );
composer.addPass( new EffectPass( camera, ssaoEffect ) );
```

The `depthAwareUpsampling` option is the pmndrs equivalent of bilateral upscale —
it reconstructs the half-resolution AO using the full-res depth buffer, avoiding
depth-discontinuity bleed that plagues naive upscaling.

---

## Performance Reference

| Configuration | Relative GPU Cost |
|---------------|------------------|
| SSAOPass 512×512 / 32 samples | 1× (baseline) |
| SSAOPass 512×512 / 64 samples | ~1.9× |
| SSAOPass full canvas / 32 samples | ~3.5× (canvas-dependent) |
| GTAOPass 512×512 default | ~2.5× |
| pmndrs SSAOEffect 0.5× scale + depth-aware upsample | ~0.8× |

These are rough ratios. Actual cost depends heavily on GPU fill-rate, canvas
resolution, and scene depth complexity.
