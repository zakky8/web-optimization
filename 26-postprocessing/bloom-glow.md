# Bloom & Glow

> Sources verified 2026-05-26 against:
> - `https://raw.githubusercontent.com/mrdoob/three.js/dev/examples/jsm/postprocessing/UnrealBloomPass.js`
> - Three.js selective bloom example: `https://raw.githubusercontent.com/mrdoob/three.js/dev/examples/webgl_postprocessing_unreal_bloom_selective.html`
> - `https://raw.githubusercontent.com/pmndrs/postprocessing/main/src/effects/BloomEffect.js`

---

## How UnrealBloomPass Works

Three.js implements an approximation of Unreal Engine's bloom algorithm. The
pipeline is:

1. **Luminance threshold pass** — extracts pixels above `threshold` into a
   separate render target. Uses a smooth knee curve so the cutoff is not a hard
   step function.
2. **Mip chain** — 5 progressive blur levels (mip 0 = full resolution,
   mip 4 = 1/16th). Each level applies a separable Gaussian blur
   (horizontal then vertical). Kernel sizes are `[6, 10, 14, 18, 22]` pixels
   per mip.
3. **Composite** — all 5 mips blend additively onto the input frame, weighted
   by the per-mip factors `[1.0, 0.8, 0.6, 0.4, 0.2]` and multiplied by
   `strength`. The `radius` parameter scales the blend weights at composite time.

Because `needsSwap = false`, UnrealBloomPass manages its own FBOs and writes
additively into the scene frame without going through the standard ping-pong
mechanism.

---

## Import

```js
import { UnrealBloomPass } from 'three/addons/postprocessing/UnrealBloomPass.js';
```

---

## Constructor

```js
const bloomPass = new UnrealBloomPass(
  resolution,   // THREE.Vector2 — effect render dimensions
  strength,     // number — default: 1
  radius,       // number — clamped to [0, 1]
  threshold     // number — luminance cutoff
);
```

Typical initialization from the official example:

```js
const bloomPass = new UnrealBloomPass(
  new THREE.Vector2( window.innerWidth, window.innerHeight ),
  1.5,    // strength
  0.4,    // radius
  0.85    // threshold
);
```

---

## Parameters

| Property | Type | Range | Effect |
|----------|------|-------|--------|
| `threshold` | number | 0.0–1.0 | Luminance cutoff. At `0.0` the entire frame blooms. At `1.0` only the brightest pixels bloom. With HDR materials (emissive values > 1), lower thresholds are appropriate. Typical: `0.0`–`0.3` for pure emissive glow; `0.7`–`0.9` for subtle natural lighting. |
| `strength` | number | 0.0–3.0 | Overall bloom brightness multiplier. `1.0` is the original unscaled composite. Values > 2 quickly wash out the image. |
| `radius` | number | 0.0–1.0 | Controls how "spread" the composite looks by scaling per-mip blend weights. Low values give tight star-like blooms; high values give wide, soft halos. |
| `resolution` | `THREE.Vector2` | — | FBO dimensions. Initialized from the canvas size. Updated via `setSize()`. |
| `bloomTintColors` | `THREE.Vector3[5]` | — | Per-mip RGB tint. Default is white for all. Set individual mip tints for color aberration effects. |
| `clearColor` | `THREE.Color` | — | Background for the luminance extraction step. Default: black. |

Runtime changes to `strength`, `radius`, and `threshold` take effect immediately
without shader recompile — they are uniforms.

---

## Minimal Working Pipeline

```js
import { EffectComposer }   from 'three/addons/postprocessing/EffectComposer.js';
import { RenderPass }       from 'three/addons/postprocessing/RenderPass.js';
import { UnrealBloomPass }  from 'three/addons/postprocessing/UnrealBloomPass.js';
import { OutputPass }       from 'three/addons/postprocessing/OutputPass.js';

renderer.toneMapping = THREE.ACESFilmicToneMapping;

const composer = new EffectComposer( renderer );
composer.addPass( new RenderPass( scene, camera ) );

const bloom = new UnrealBloomPass(
  new THREE.Vector2( window.innerWidth, window.innerHeight ),
  1.0,  // strength
  0.5,  // radius
  0.0   // threshold — bloom everything above 0 luminance (emissive-only if using HDR materials)
);
composer.addPass( bloom );

composer.addPass( new OutputPass() );

function onResize() {
  renderer.setSize( window.innerWidth, window.innerHeight );
  composer.setSize( window.innerWidth, window.innerHeight );
}
```

**The pass order is critical.** Bloom must run before OutputPass. OutputPass
applies tone mapping; if you tone-map first, bright HDR values are compressed
to ≤1 and the bloom threshold filter sees no bright content.

---

## Bloom on Emissive Materials Only

The cleanest pattern for "only emissive objects bloom" is to keep
`threshold` at `0.0` and rely on the fact that non-emissive objects in a
correctly-lit scene have luminance < 1.0 in linear space, while emissive
materials with `emissiveIntensity > 1.0` have linear values above 1.

```js
mesh.material.emissive     = new THREE.Color( 0xffffff );
mesh.material.emissiveIntensity = 3.0;   // pushes luminance well above threshold

bloom.threshold = 0.9;   // only catch extreme highlights
```

This works well if your scene uses HDR lighting. In darker scenes where
non-emissive surfaces may peak near 1.0, this threshold will accidentally
bloom highlights on regular geometry.

---

## Selective Bloom via Layers

For surgical control over which objects bloom regardless of luminance, use
Two-Pass Selective Bloom. This is the technique from the Three.js
`webgl_postprocessing_unreal_bloom_selective` example.

### Concept

1. Assign bloomed objects to a dedicated layer (e.g. layer 1).
2. **Bloom pass**: Hide all non-bloom objects (replace materials with black),
   render through UnrealBloomPass into `bloomRenderTarget`.
3. **Final composite pass**: Render the full scene normally, then blend in the
   bloom texture with a ShaderPass.

### Implementation

```js
import * as THREE from 'three';
import { EffectComposer }  from 'three/addons/postprocessing/EffectComposer.js';
import { RenderPass }      from 'three/addons/postprocessing/RenderPass.js';
import { UnrealBloomPass } from 'three/addons/postprocessing/UnrealBloomPass.js';
import { ShaderPass }      from 'three/addons/postprocessing/ShaderPass.js';
import { OutputPass }      from 'three/addons/postprocessing/OutputPass.js';
import { HalfFloatType }   from 'three';

const BLOOM_LAYER = 1;
const bloomLayer  = new THREE.Layers();
bloomLayer.set( BLOOM_LAYER );

const darkMaterial = new THREE.MeshBasicMaterial({ color: 0x000000 });
const materials    = {};   // stash for original materials

// ---- Bloom composer ----
const bloomComposer = new EffectComposer( renderer );
bloomComposer.renderToScreen = false;
bloomComposer.addPass( new RenderPass( scene, camera ) );

const bloomPass = new UnrealBloomPass(
  new THREE.Vector2( window.innerWidth, window.innerHeight ),
  1.5, 0.4, 0.85
);
bloomComposer.addPass( bloomPass );

// ---- Final composer ----
const finalComposer = new EffectComposer( renderer );
finalComposer.addPass( new RenderPass( scene, camera ) );

const mixShader = {
  uniforms: {
    baseTexture:  { value: null },
    bloomTexture: { value: bloomComposer.renderTarget2.texture },
  },
  vertexShader: `
    varying vec2 vUv;
    void main() {
      vUv = uv;
      gl_Position = projectionMatrix * modelViewMatrix * vec4( position, 1.0 );
    }
  `,
  fragmentShader: `
    uniform sampler2D baseTexture;
    uniform sampler2D bloomTexture;
    varying vec2 vUv;
    void main() {
      gl_FragColor = texture2D( baseTexture, vUv ) + vec4( 1.0 ) * texture2D( bloomTexture, vUv );
    }
  `,
};

const mixPass = new ShaderPass( mixShader, 'baseTexture' );
mixPass.needsSwap = true;
finalComposer.addPass( mixPass );
finalComposer.addPass( new OutputPass() );

// ---- Render loop ----
function darkenNonBloomed( obj ) {
  if ( obj.isMesh && !bloomLayer.test( obj.layers ) ) {
    materials[ obj.uuid ] = obj.material;
    obj.material = darkMaterial;
  }
}

function restoreMaterials( obj ) {
  if ( materials[ obj.uuid ] ) {
    obj.material = materials[ obj.uuid ];
    delete materials[ obj.uuid ];
  }
}

function render() {
  // Pass 1: render only bloom-layer objects
  scene.traverse( darkenNonBloomed );
  bloomComposer.render();
  scene.traverse( restoreMaterials );

  // Pass 2: render full scene composited with bloom
  finalComposer.render();
}
```

### Assigning objects to the bloom layer

```js
// Objects on layer 1 will bloom
glowMesh.layers.enable( BLOOM_LAYER );

// Toggle bloom on a specific object (e.g. on click)
object.layers.toggle( BLOOM_LAYER );
```

Objects on no extra layer (only layer 0) are darkened during the bloom pass and
do not contribute to bloom. Their full materials appear in the final composite.

---

## pmndrs/postprocessing BloomEffect

```js
import { EffectComposer, EffectPass, RenderPass, BloomEffect } from 'postprocessing';

const bloom = new BloomEffect({
  blendFunction:       BlendFunction.ADD,
  luminanceThreshold:  0.9,
  luminanceSmoothing:  0.025,
  mipmapBlur:          true,   // modern path — preferred
  intensity:           1.0,
  radius:              0.85,
  levels:              8,       // MIP levels for mipmap blur
});

const composer = new EffectComposer( renderer, { frameBufferType: HalfFloatType } );
composer.addPass( new RenderPass( scene, camera ) );
composer.addPass( new EffectPass( camera, bloom ) );
```

Key differences from Three.js UnrealBloomPass:

| Feature | Three.js UnrealBloomPass | pmndrs BloomEffect |
|---------|--------------------------|-------------------|
| Blur method | Separable Gaussian per mip | `MipmapBlurPass` (hardware mip chain) |
| MIP levels | Fixed 5 | Configurable `levels` (default 8) |
| Effect merging | Not possible | Multiple effects merge into one EffectPass draw |
| Selective bloom | Manual layer trick | Use `Selection` API |
| Performance | Good | Better on large canvases due to mipmap-blur |

The `MipmapBlurPass` in pmndrs leverages hardware mipmap generation, avoiding
manual separable blur passes per mip level. This reduces draw calls substantially
at higher mip counts.

---

## Performance Notes

- UnrealBloomPass executes 11 draw calls per frame (5 horizontal blurs + 5
  vertical blurs + 1 composite). On a 1920×1080 canvas, this is significant;
  consider halving the resolution: `new THREE.Vector2( width/2, height/2 )`.
- The selective bloom technique doubles total draw calls (one full scene render
  for the bloom pass, one for the final composite). Profile before shipping.
- `threshold = 0.0` + `strength = 0.1` is often more visually accurate for
  natural ambient glow than a high threshold with high strength. Low-threshold
  bloom with subtle strength does not produce harsh halos at emissive edges.
- On mobile, disable bloom entirely or drop to `strength = 0.3` with reduced
  resolution FBOs. Bloom is one of the most expensive effects per pixel.
