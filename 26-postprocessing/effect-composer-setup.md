# EffectComposer — Complete Setup Guide

> Sources verified 2026-05-26 against Three.js r184 source at
> `https://raw.githubusercontent.com/mrdoob/three.js/dev/examples/jsm/postprocessing/EffectComposer.js`
> and
> `https://raw.githubusercontent.com/mrdoob/three.js/dev/examples/jsm/postprocessing/RenderPass.js`

---

## What EffectComposer Does

EffectComposer manages an ordered chain of `Pass` objects. Each pass reads from a
**readBuffer** (a `WebGLRenderTarget`), does its work, and writes to a **writeBuffer**.
After a pass that has `needsSwap = true`, the buffers swap. The final enabled pass
routes its output to the screen (or to any render target you override
`renderToScreen` for).

Internally the composer allocates **two** render targets — `rt1` and `rt2` — so
reads and writes never race each other. Both default to `HalfFloatType` for precision
headroom in HDR pipelines.

---

## Import Paths (Three.js addons)

```js
import { EffectComposer } from 'three/addons/postprocessing/EffectComposer.js';
import { RenderPass }     from 'three/addons/postprocessing/RenderPass.js';
import { OutputPass }     from 'three/addons/postprocessing/OutputPass.js';
import { ShaderPass }     from 'three/addons/postprocessing/ShaderPass.js';
import { Pass }           from 'three/addons/postprocessing/Pass.js';
```

> `three/addons` was introduced as a clean npm alias in r158 (October 2023).
> The older `three/examples/jsm/…` path still works but is now discouraged.

---

## Constructor

```js
const composer = new EffectComposer( renderer, renderTarget? );
```

| Parameter | Type | Required | Notes |
|-----------|------|----------|-------|
| `renderer` | `WebGLRenderer` | yes | The renderer this composer drives |
| `renderTarget` | `WebGLRenderTarget` | no | Custom internal buffer. When omitted, composer creates two targets using `HalfFloatType`. Provide one if you need stencil bits, custom filtering, or a specific format. |

### Manually supplying HalfFloat targets

The composer auto-uses `HalfFloatType` when it builds its internal targets. If
you supply your own target, match that format to keep precision consistent:

```js
import { HalfFloatType } from 'three';

const rt = new THREE.WebGLRenderTarget(
  window.innerWidth,
  window.innerHeight,
  {
    type: HalfFloatType,
    format: THREE.RGBAFormat,
    depthBuffer: true,
  }
);

const composer = new EffectComposer( renderer, rt );
```

Why `HalfFloatType`? Standard `UnsignedByteType` clamps values to [0, 1], which
truncates specular highlights and emissive areas before bloom/tonemapping can act
on them. Half-float keeps values above 1.0 alive through the pipe.

---

## Core Properties

| Property | Default | Purpose |
|----------|---------|---------|
| `renderer` | — | The WebGLRenderer instance |
| `writeBuffer` | auto | Current write target |
| `readBuffer` | auto | Current read target |
| `passes` | `[]` | Ordered array of Pass objects |
| `renderToScreen` | `true` | Whether the last pass blits to canvas |

---

## All Methods

### Pass management

```js
composer.addPass( pass )            // Append to end of chain
composer.insertPass( pass, index )  // Insert at arbitrary index
composer.removePass( pass )         // Remove a pass
composer.isLastEnabledPass( i )     // → boolean; true if passes[i] is final enabled
```

### Render

```js
composer.render( deltaTime? )
```

Iterates `passes`. For each enabled pass:
1. Calls `pass.render( renderer, writeBuffer, readBuffer, deltaTime, maskActive )`.
2. If `pass.needsSwap` is true, swaps the read/write buffers.
3. Tracks `MaskPass`/`ClearMaskPass` to correctly maintain stencil state.

### Sizing and pixel ratio

```js
composer.setSize( width, height )
composer.setPixelRatio( ratio )     // Triggers setSize internally
```

Both methods propagate the new dimensions to every pass in the chain that has
its own render targets (e.g. SSAOPass, UnrealBloomPass each own internal FBOs).
**Always call `composer.setSize` inside your resize handler — not just
`renderer.setSize`.**

### Reset and dispose

```js
composer.reset( renderTarget? )     // Reinitialise internal buffers
composer.dispose()                  // Free GPU memory: render targets + copy pass
```

`dispose()` only cleans the composer's own allocations. Each pass owns its own
targets; call `pass.dispose()` separately for passes that allocate (e.g.
UnrealBloomPass, SSAOPass, BokehPass).

---

## Minimal Working Setup

```js
import * as THREE from 'three';
import { EffectComposer } from 'three/addons/postprocessing/EffectComposer.js';
import { RenderPass }     from 'three/addons/postprocessing/RenderPass.js';
import { OutputPass }     from 'three/addons/postprocessing/OutputPass.js';

const renderer = new THREE.WebGLRenderer({ antialias: true });
renderer.setPixelRatio( window.devicePixelRatio );
renderer.setSize( window.innerWidth, window.innerHeight );
// Do NOT set renderer.toneMapping or outputColorSpace here if using OutputPass —
// OutputPass reads them at render time.
document.body.appendChild( renderer.domElement );

const composer = new EffectComposer( renderer );

// 1. Render the scene
const renderPass = new RenderPass( scene, camera );
composer.addPass( renderPass );

// 2. … insert effect passes here …

// 3. Final pass: applies renderer.toneMapping + color-space conversion
const outputPass = new OutputPass();
composer.addPass( outputPass );

// Animation loop — use composer.render() instead of renderer.render()
function animate() {
  requestAnimationFrame( animate );
  composer.render();
}
animate();
```

---

## RenderPass

```js
const renderPass = new RenderPass(
  scene,            // THREE.Scene
  camera,           // THREE.Camera
  overrideMaterial, // optional — apply one material to all mesh objects
  clearColor,       // optional — custom clear color for this pass
  clearAlpha        // optional — clear alpha
);
```

Key properties after construction:

| Property | Default | Notes |
|----------|---------|-------|
| `clear` | `true` | Clears color, depth, stencil before render |
| `clearDepth` | `false` | Depth-only clear when `clear = false` |
| `needsSwap` | `false` | Intentionally false — RenderPass writes into writeBuffer but the scene result must stay in readBuffer for the next pass |
| `isRenderPass` | `true` (readonly) | Identifies this pass to other systems |

RenderPass deliberately sets `needsSwap = false`. The scene ends up in
`writeBuffer`, but no swap occurs, so the *next* pass reads from `readBuffer`
which at that moment still points to the same target. This is the intended
design — later passes that need the scene color read `readBuffer`.

---

## OutputPass

OutputPass is **always the last pass** in a Three.js composer chain. It:

1. Applies `renderer.toneMapping` (reads the value each frame, recompiles shader
   defines only when it changes).
2. Converts from linear working space to `renderer.outputColorSpace` (adds an
   sRGB transfer function when needed).
3. Applies `renderer.toneMappingExposure` as a pre-multiply.

Supported tone mapping operators (as of r184):

| Constant | Notes |
|----------|-------|
| `THREE.NoToneMapping` | Pass-through |
| `THREE.LinearToneMapping` | Exposure-scale only |
| `THREE.ReinhardToneMapping` | Classic |
| `THREE.CineonToneMapping` | Film-style curve |
| `THREE.ACESFilmicToneMapping` | S-curve, industry standard |
| `THREE.AgXToneMapping` | Added r160; perceptually accurate, avoids hue shifts |
| `THREE.NeutralToneMapping` | Added r162; Khronos standard; default since r184 |
| `THREE.CustomToneMapping` | Requires custom GLSL in `ShaderChunk.tonemapping_pars_fragment` |

```js
renderer.toneMapping         = THREE.ACESFilmicToneMapping;
renderer.toneMappingExposure = 1.2;
renderer.outputColorSpace    = THREE.SRGBColorSpace;
```

OutputPass has a `dispose()` method. Call it when tearing down.

---

## Pass Ordering

A well-structured chain looks like this:

```
RenderPass          ← always first
SSAOPass / GTAOPass ← AO before any lighting manipulation
UnrealBloomPass     ← bloom reads luminance, must see HDR values
BokehPass / DOFPass ← DoF needs depth, add before tone mapping
LUTPass             ← color grading on tonemapped-but-linear data
OutputPass          ← ALWAYS last; applies tone mapping + sRGB
```

Putting tone mapping (OutputPass) before bloom is the single most common
ordering mistake. Bloom needs the HDR values > 1.0 that survive only in the
half-float buffer.

---

## Resize Handling

```js
window.addEventListener( 'resize', onResize );

function onResize() {
  const w = window.innerWidth;
  const h = window.innerHeight;

  camera.aspect = w / h;
  camera.updateProjectionMatrix();

  renderer.setSize( w, h );           // Resize canvas + renderer
  composer.setSize( w, h );           // Resize ALL composer pass targets
}
```

`composer.setSize` iterates every pass and calls `pass.setSize( w, h )` on
those that implement it. Passes that manage their own FBOs (Bloom, SSAO,
BokehPass) all implement `setSize`, so a single call handles the chain.

When using `renderer.setPixelRatio( window.devicePixelRatio )`, pass the CSS
pixel dimensions to `setSize` (not scaled by devicePixelRatio) — the renderer
and composer handle the scaling internally.

---

## Dispose Pattern

```js
function teardown() {
  // 1. Dispose passes that own FBOs
  ssaoPass.dispose();
  bloomPass.dispose();
  bokehPass.dispose();
  outputPass.dispose();

  // 2. Dispose the composer (frees internal ping-pong targets + copy pass)
  composer.dispose();

  // 3. Dispose the renderer
  renderer.dispose();
}
```

Not calling `dispose()` leaks textures and framebuffers on the GPU. In SPAs
where the canvas may be unmounted and remounted, this will drain VRAM quickly.

---

## pmndrs/postprocessing EffectComposer — Key Differences

The `postprocessing` library (v6.39.1, April 2026) exports its own
`EffectComposer` with a different constructor:

```js
import { EffectComposer } from 'postprocessing';

const composer = new EffectComposer( renderer, {
  frameBufferType: HalfFloatType,   // recommended
  multisampling:  4,                // MSAA via WebGL2 resolve (Three.js built-in lacks this)
  stencilBuffer:  false,
  depthBuffer:    true,
} );
```

Key differences from Three.js built-in:

| Feature | Three.js built-in | pmndrs/postprocessing |
|---------|-------------------|-----------------------|
| Effect merging | No — each pass is a separate draw | Yes — `EffectPass` merges multiple effects into one shader |
| Fullscreen geometry | Screen-filling quad (2 triangles) | Single triangle (eliminates diagonal fragment overdraw) |
| MSAA support | Not supported | Native via `multisampling` option (WebGL2 only) |
| Stable depth texture | No | Yes — `blitDepthBuffer` prevents feedback loops |
| Auto-sizing | Manual `setSize` | Integrated `Resolution` objects per effect |
| Tone mapping / color space | Handled by OutputPass | Follows `renderer.outputColorSpace` automatically |

The single-triangle optimization alone cuts fullscreen fragment shading by ~15%
on large viewports compared to a quad.

---

## Performance Notes

- **Pixel ratio**: Do not call `renderer.setPixelRatio` and then forget to call
  `composer.setPixelRatio` — the internal targets will be undersized.
- **HalfFloat vs UnsignedByte**: Half-float uses 2x the bandwidth of 8-bit but
  is essential for HDR-safe bloom and tonemapping. On mobile where bandwidth is
  the bottleneck, consider testing `UnsignedByteType` if no effects require HDR.
- **Pass count**: Every pass except the last writes to an offscreen FBO and then
  does a full-screen blit. Minimize passes; where possible merge via `ShaderPass`
  or use pmndrs/postprocessing's effect-merging `EffectPass`.
- **Disabled passes**: Set `pass.enabled = false` rather than removing and
  re-adding passes; `isLastEnabledPass` skips disabled passes correctly.
