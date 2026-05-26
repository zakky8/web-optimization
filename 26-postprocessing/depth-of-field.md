# Depth of Field

> Sources verified 2026-05-26 against:
> - `https://raw.githubusercontent.com/mrdoob/three.js/dev/examples/jsm/postprocessing/BokehPass.js`
> - Three.js dof2 example: `https://raw.githubusercontent.com/mrdoob/three.js/dev/examples/webgl_postprocessing_dof2.html`
> - pmndrs/postprocessing v6.39.1 (April 2026)

---

## Overview

Depth of field (DoF) simulates the lens blur that occurs when objects are
outside a camera's focal plane. Post-processing DoF operates in screen space:
the scene is rendered once, a depth buffer is captured, and a fragment shader
blurs pixels proportionally to their depth deviation from the focus distance.

Three.js ships `BokehPass`. For higher-quality tilt-shift or variable bokeh, the
`postprocessing` library's `DepthOfFieldEffect` is the preferred choice.

---

## BokehPass

### Import

```js
import { BokehPass } from 'three/addons/postprocessing/BokehPass.js';
```

### Constructor

```js
const bokehPass = new BokehPass( scene, camera, params );
```

The `params` object accepts:

| Key | Default | Notes |
|-----|---------|-------|
| `focus` | `1.0` | "distance along the camera's look direction in world units." The focal plane sits at this depth. |
| `aperture` | `0.025` | Controls the circle of confusion radius. Larger values = stronger blur off-focus. |
| `maxblur` | `1.0` | Absolute maximum blur radius in texture-space units. Clamps the effect on very far/near objects. |

### Runtime uniform access

All three parameters are shader uniforms and can be changed each frame:

```js
bokehPass.uniforms[ 'focus'    ].value = 5.0;
bokehPass.uniforms[ 'aperture' ].value = 0.002;
bokehPass.uniforms[ 'maxblur'  ].value = 0.01;
```

### Internal rendering pipeline

BokehPass uses a two-stage approach:

1. **Depth pre-pass**: Renders the scene with `MeshDepthMaterial` (RGBA depth
   packing) to a half-float FBO. Camera `near` and `far` are forwarded as
   uniforms automatically.
2. **Bokeh composite**: `BokehShader` reads both the color texture and the depth
   texture. For each fragment it computes a circle of confusion (CoC) radius
   from the depth deviation relative to `focus`, then performs a radial blur
   clamped to `maxblur`.

### Minimal setup

```js
import { EffectComposer } from 'three/addons/postprocessing/EffectComposer.js';
import { RenderPass }     from 'three/addons/postprocessing/RenderPass.js';
import { BokehPass }      from 'three/addons/postprocessing/BokehPass.js';
import { OutputPass }     from 'three/addons/postprocessing/OutputPass.js';

const composer = new EffectComposer( renderer );
composer.addPass( new RenderPass( scene, camera ) );

const bokehPass = new BokehPass( scene, camera, {
  focus:    10.0,    // focus 10 world units away
  aperture: 0.003,
  maxblur:  0.005,
} );
composer.addPass( bokehPass );

composer.addPass( new OutputPass() );
```

---

## BokehPass Parameter Guide

### `focus` — focal plane distance

This is a world-space distance along the camera's Z axis (the depth at which
objects appear perfectly sharp). It corresponds directly to the camera's
`near`/`far` depth range but in linear world units.

For a camera at the origin looking down -Z:
- `focus = 5.0` puts the focal plane 5 units in front of the camera
- Objects closer or farther than 5 units get progressively blurred

### `aperture` — blur strength per unit of defocus

Aperture controls how fast the blur grows as objects move away from the focal
plane. It behaves like an f-stop in reverse: lower values = deeper depth of
field (more in focus), higher values = shallower (more blur). Typical range
for subtle DoF: `0.001`–`0.005`. For dramatic shallow focus: `0.02`–`0.1`.

### `maxblur` — hard blur cap

Even with a wide aperture, very distant objects should not blur to infinity —
that looks wrong and destroys readability. `maxblur` caps the sample radius.
This is in UV texture space (0–1), so `0.01` is about 1% of the screen width.
Typical useful range: `0.003`–`0.02`.

---

## Advanced DOF: the dof2 example approach

The `webgl_postprocessing_dof2` example in Three.js demonstrates a more
sophisticated multi-parameter approach using a more complex `BokehShader2`:

```js
const effectController = {
  enabled:       true,
  fstop:         2.2,        // f-stop, maps to aperture
  maxblur:       1.0,
  focalDepth:    2.8,        // focus depth
  manualdof:     false,
  focalLength:   35,         // lens focal length (mm) — informational
  rings:         3,          // radial sample rings (quality)
  samples:       4,          // samples per ring
};
```

The `rings` and `samples` parameters control quality: total samples = `rings *
samples * 8` (approximately). Higher values reduce the characteristic pattern
of discrete blur circles but cost proportionally more fill rate.

---

## Dynamic Focus with Raycasting

A common interaction pattern is to move the focal plane to the object under the
cursor. The `dof2` example demonstrates this:

```js
const raycaster = new THREE.Raycaster();
const mouse     = new THREE.Vector2();
let   distance  = camera.far / 2;

window.addEventListener( 'mousemove', ( e ) => {
  mouse.x =  ( e.clientX / window.innerWidth  ) * 2 - 1;
  mouse.y = -( e.clientY / window.innerHeight ) * 2 + 1;
} );

function updateFocus() {
  raycaster.setFromCamera( mouse, camera );
  const intersects = raycaster.intersectObjects( scene.children, true );

  const targetDistance = intersects.length > 0
    ? intersects[ 0 ].distance
    : camera.far;

  // Smooth lerp to avoid jarring jumps
  distance += ( targetDistance - distance ) * 0.03;
  bokehPass.uniforms[ 'focus' ].value = distance;
}
```

Call `updateFocus()` inside the animation loop.

---

## Debugging the Focus Plane

When `focus` is set correctly, sharp objects should be centered on the focal
plane. To visualize:

1. Temporarily render a `PlaneGeometry` perpendicular to the camera at the
   target distance, coloured bright red with `depthTest: false`. If it aligns
   with the sharp region, the focus value is correct.
2. Add a GUI slider connected to `bokehPass.uniforms['focus'].value` and drag
   it while watching the scene. Fastest way to calibrate.
3. Use `SSAOPass.OUTPUT.Depth` (or a similar ShaderPass that outputs depth) to
   confirm the depth buffer is correct before blaming the DoF parameters.

---

## pmndrs/postprocessing — DepthOfFieldEffect

The pmndrs `DepthOfFieldEffect` is more production-ready than BokehPass. It
implements a Circle of Confusion (CoC) pipeline with separate near and far blur
fields, auto-focus capability, and Bokeh shape control.

```js
import {
  EffectComposer,
  EffectPass,
  RenderPass,
  DepthOfFieldEffect,
  BlendFunction
} from 'postprocessing';

const dof = new DepthOfFieldEffect( camera, {
  blendFunction:    BlendFunction.NORMAL,
  worldFocusDistance:  10,    // world-space focus distance
  worldFocusRange:     5,     // half-range of in-focus zone
  bokehScale:          4.0,   // bokeh radius scale
  height:              480,   // effect render height (lower = cheaper)
} );

const composer = new EffectComposer( renderer, {
  frameBufferType: HalfFloatType,
} );
composer.addPass( new RenderPass( scene, camera ) );
composer.addPass( new EffectPass( camera, dof ) );
```

Key differences from BokehPass:

| Feature | Three.js BokehPass | pmndrs DepthOfFieldEffect |
|---------|-------------------|--------------------------|
| Near field blur | Partial (maxblur clamps both directions) | Separate near/far blur passes |
| Bokeh shape | Circular radial blur | Hexagonal/custom via `BokehMaterial` |
| Auto-focus | Not built-in (manual raycasting) | Built-in via `target` Vector3 |
| Resolution control | Shares composer size | `height` parameter (independent) |
| Vignette support | No | Yes (separate VignetteEffect) |
| CoC visualization | No | `cocMaterial.defines.VISUALIZE_COC = ''` |

### Auto-focus in pmndrs

```js
// Focus on a scene object automatically
dof.target = mesh.position;   // updates worldFocusDistance each frame
```

---

## Performance Notes

- BokehPass requires a full depth pre-pass (extra scene render). This is the
  dominant cost, not the bokeh blur itself.
- The blur quality in BokehPass (`rings` × `samples`) grows quadratically. On
  a budget, `rings = 2, samples = 2` gives 32 samples per fragment and looks
  acceptable at small `maxblur` values.
- DoF is generally incompatible with MSAA — the depth buffer from an MSAA FBO
  must be resolved before the bokeh shader reads it. The EffectComposer's ping-
  pong buffers handle this, but verify the depth texture is from the resolved
  target, not the multisample one.
- On mobile, disable DoF. It is one of the most fill-rate-intensive effects.
  Alternative: a subtle vignette darkening the corners achieves a similar
  compositional effect at near-zero cost.
- `maxblur` values above `0.02` produce visible banding in the BokehShader
  because sample count is fixed. If you need wide bokeh, switch to pmndrs
  `DepthOfFieldEffect` which uses a proper CoC accumulation buffer.
