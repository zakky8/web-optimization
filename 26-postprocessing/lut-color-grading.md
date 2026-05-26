# LUT Color Grading & Tone Mapping

> Sources verified 2026-05-26 against:
> - `https://raw.githubusercontent.com/mrdoob/three.js/dev/examples/jsm/postprocessing/LUTPass.js`
> - `https://raw.githubusercontent.com/mrdoob/three.js/dev/examples/jsm/loaders/LUTCubeLoader.js`
> - `https://raw.githubusercontent.com/mrdoob/three.js/dev/examples/jsm/loaders/LUTImageLoader.js`
> - `https://raw.githubusercontent.com/mrdoob/three.js/dev/examples/jsm/postprocessing/OutputPass.js`
> - Three.js r160 release notes (AgXToneMapping added)
> - Three.js r162 release notes (NeutralToneMapping added)
> - Three.js r184 release notes (NeutralToneMapping made default)

---

## Concepts

### LUT vs Tone Mapping — What's the Difference?

**Tone mapping** compresses a high-dynamic-range linear scene into a displayable
range, typically [0, 1] sRGB. It is a mathematical operator applied to all
pixels uniformly. Three.js applies it inside `OutputPass` as the final step.

**LUT (Look-Up Table) color grading** maps each RGB input color to a different
output color by sampling a 3D cube of pre-computed color transformations. A LUT
does not replace tone mapping — it is applied after tone mapping and is used for
creative look adjustment, film emulation, or brand-specific color grades. The
correct position in the pipeline is: HDR rendering → tone mapping → LUT grade.

Think of LUTs as the digital equivalent of a film stock choice or a colorist's
grade. Tone mapping is the physics; LUT is the art.

---

## LUTPass

`LUTPass` extends `ShaderPass` and applies a 3D LUT texture to the rendered
frame.

### Import

```js
import { LUTPass }      from 'three/addons/postprocessing/LUTPass.js';
import { LUTCubeLoader } from 'three/addons/loaders/LUTCubeLoader.js';
```

### Constructor

```js
const lutPass = new LUTPass( {
  lut:       null,   // THREE.Data3DTexture — required before the pass is useful
  intensity: 1,      // 0.0–1.0 blend factor
} );
```

`intensity` drives a `mix()` in the fragment shader: `0.0` = original colors,
`1.0` = fully graded. This is set as a shader uniform and can be animated in
real time.

### Correct placement in chain

```
RenderPass      ← render scene (linear)
UnrealBloomPass ← bloom on linear/HDR values
LUTPass         ← grade AFTER OutputPass applies tone mapping
OutputPass      ← tone map + sRGB conversion
```

Wait — LUTPass must go BEFORE OutputPass if you want to grade the linear result
before sRGB encoding. Most real-world pipelines grade in log or linear space.
But if your LUT was designed in a video editor on a tonemapped image (the
common case for downloaded free LUTs), insert LUTPass AFTER OutputPass.

The safest mental model: **match the LUT's design space to where you insert
it in the chain.** When in doubt, insert after OutputPass and test both positions.

---

## Loading .cube Files

`.cube` is the industry-standard text format for LUTs (Adobe/DaVinci Resolve).
Three.js provides `LUTCubeLoader` to parse them into a `Data3DTexture`.

```js
import { LUTCubeLoader } from 'three/addons/loaders/LUTCubeLoader.js';

const loader = new LUTCubeLoader();

loader.load( '/luts/kodak_2383.cube', ( result ) => {
  lutPass.lut = result.texture3D;
} );
```

The loader returns an object with these fields:

| Field | Type | Notes |
|-------|------|-------|
| `texture3D` | `Data3DTexture` | Cubic 3D texture, LinearFilter, ClampToEdgeWrapping |
| `size` | number | Cube dimension (e.g. 33 for a 33×33×33 LUT) |
| `domainMin` | `Vector3` | Input value floor (default: 0,0,0) |
| `domainMax` | `Vector3` | Input value ceiling (default: 1,1,1) |
| `title` | string | Optional TITLE metadata from file |

### LUTCubeLoader precision

By default the loader creates `UnsignedByteType` (8-bit) textures. For
production quality with minimal banding, request float precision:

```js
const loader = new LUTCubeLoader();
loader.setDataType( THREE.FloatType );   // 32-bit per channel

loader.load( '/luts/cinematic_grade.cube', ( result ) => {
  lutPass.lut = result.texture3D;
} );
```

Float LUTs double the texture memory vs 8-bit but eliminate all quantization
banding, especially in dark areas.

---

## Loading Image-Based LUTs

For LUTs distributed as PNG strips (common from Unity/Unreal workflows):

```js
import { LUTImageLoader } from 'three/addons/loaders/LUTImageLoader.js';

const loader = new LUTImageLoader();
loader.load( '/luts/orange_teal.png', ( result ) => {
  lutPass.lut = result.texture3D;
} );
```

`LUTImageLoader` handles both vertical strips and horizontal strips. The `flip`
property corrects green-channel orientation differences between DCC tools:

```js
loader.flip = true;   // for LUTs exported from Unreal Engine
```

---

## Fragment Shader Mechanics

The LUTPass fragment shader uses a half-pixel offset to ensure sampling begins
at cube centers, not edges — this prevents clamping artefacts at the extreme
values of the LUT:

```glsl
vec3 halfPixelWidth = vec3( 0.5 / lutSize );
vec3 lookup = halfPixelWidth + inputColor.rgb * ( 1.0 - pixelWidth );
gl_FragColor = mix( inputColor, texture( lut, lookup ), intensity );
```

The `mix()` with `intensity` means at `intensity = 0.5`, you get a 50/50 blend
of the original and graded color. Alpha is preserved from the input texture.

---

## Full Pipeline Example

```js
import * as THREE from 'three';
import { EffectComposer }  from 'three/addons/postprocessing/EffectComposer.js';
import { RenderPass }      from 'three/addons/postprocessing/RenderPass.js';
import { UnrealBloomPass } from 'three/addons/postprocessing/UnrealBloomPass.js';
import { OutputPass }      from 'three/addons/postprocessing/OutputPass.js';
import { LUTPass }         from 'three/addons/postprocessing/LUTPass.js';
import { LUTCubeLoader }   from 'three/addons/loaders/LUTCubeLoader.js';
import { HalfFloatType }   from 'three';

// --- Renderer ---
const renderer = new THREE.WebGLRenderer( { antialias: true } );
renderer.toneMapping         = THREE.ACESFilmicToneMapping;
renderer.toneMappingExposure = 1.0;
renderer.outputColorSpace    = THREE.SRGBColorSpace;
renderer.setSize( window.innerWidth, window.innerHeight );

// --- Composer ---
const composer = new EffectComposer( renderer );

// Pass 1: render scene (linear HDR)
composer.addPass( new RenderPass( scene, camera ) );

// Pass 2: bloom (needs linear values above threshold)
const bloom = new UnrealBloomPass(
  new THREE.Vector2( window.innerWidth, window.innerHeight ),
  0.5, 0.4, 0.8
);
composer.addPass( bloom );

// Pass 3: tone mapping + sRGB encode
const outputPass = new OutputPass();
composer.addPass( outputPass );

// Pass 4: creative LUT grade (applied to tonemapped sRGB)
const lutPass = new LUTPass( { intensity: 1.0 } );
composer.addPass( lutPass );

// --- Load LUT ---
new LUTCubeLoader()
  .setDataType( THREE.FloatType )
  .load( '/luts/cinematic.cube', ( result ) => {
    lutPass.lut = result.texture3D;
  } );
```

---

## Tone Mapping Operators

All tone mapping in Three.js is applied through `OutputPass`, which reads
`renderer.toneMapping` at render time and recompiles shader defines only when
the value changes.

### Available operators

| Constant | Added | Characteristics |
|----------|-------|-----------------|
| `THREE.NoToneMapping` | — | Raw linear output. Clips above 1.0. |
| `THREE.LinearToneMapping` | — | Exposure scale only. `exposure * color`. Still clips. |
| `THREE.ReinhardToneMapping` | — | Soft shoulder. Compressed highlights. Desaturates at high values. |
| `THREE.CineonToneMapping` | — | Film-print emulation. Warm shadows, slightly crushed blacks. |
| `THREE.ACESFilmicToneMapping` | — | Academy Color Encoding System S-curve. Industry standard. High contrast. |
| `THREE.AgXToneMapping` | r160 | Perceptually accurate, avoids hue shifts in highlights. Best for photorealistic scenes. |
| `THREE.NeutralToneMapping` | r162; default since r184 | Khronos standard. Minimal creative interpretation. Preserves hues better than ACES. |
| `THREE.CustomToneMapping` | — | Reads GLSL from `ShaderChunk.tonemapping_pars_fragment`. |

### Choosing an operator

- **ACES Filmic**: High-contrast look, film-like. Will shift hues toward orange in
  highlights. Preferred for games with warm lighting aesthetics.
- **AgX (r160+)**: Most physically accurate hue preservation. Recommended for
  product visualization, architecture, scientific rendering.
- **Neutral (r162+, default r184)**: Khronos-specified, designed to be
  colorist-friendly — minimal interpretation so LUTs applied after it behave
  predictably. Best choice when heavy LUT grading follows.
- **Reinhard**: Legacy. Looks muddy in scenes with both very dark and very bright
  elements. Avoid in new projects.

### Exposure control

```js
renderer.toneMappingExposure = 1.2;   // >1.0 brightens; <1.0 darkens
```

This is a pre-multiply applied before the tone mapping curve. It is equivalent
to adjusting scene EV in photography. A value of `1.0` is "as rendered."

---

## Gamma Correction

Three.js handles gamma automatically when you set:

```js
renderer.outputColorSpace = THREE.SRGBColorSpace;
```

`OutputPass` detects this and adds an sRGB transfer function (`SRGB_TRANSFER`
shader define) to the final output. You should not manually apply gamma 2.2
correction in shader code if `outputColorSpace` is set correctly.

The common mistake: mixing `renderer.outputColorSpace = THREE.SRGBColorSpace`
(correct) with also doing `pow(color, vec3(1.0/2.2))` in a custom ShaderPass
fragment shader. That double-corrects and produces washed-out output.

Historically (pre-r152), Three.js used `renderer.outputEncoding = sRGBEncoding`
(now removed). As of r152+, only `outputColorSpace` exists.

### Texture color space

For any texture that stores color data (diffuse maps, emissive maps, LUT
outputs), Three.js expects the loader to declare color space:

```js
texture.colorSpace = THREE.SRGBColorSpace;   // for diffuse/emissive textures
// Data3DTexture from LUTCubeLoader: leave as THREE.LinearSRGBColorSpace (default)
```

Normal maps, roughness maps, metalness maps — use `THREE.NoColorSpace` (they
are data, not color).

---

## pmndrs/postprocessing — LUT1DEffect / LUT3DEffect

The pmndrs library splits LUT handling into two effects:

```js
import { LUT3DEffect } from 'postprocessing';
import { LUTCubeLoader } from 'three/addons/loaders/LUTCubeLoader.js';

new LUTCubeLoader().load( '/luts/grade.cube', ( result ) => {
  const lut3d = new LUT3DEffect( result.texture3D, {
    blendFunction: BlendFunction.SET,
    inputEncoding:  SRGBColorSpace,
    outputEncoding: SRGBColorSpace,
    tetrahedralInterpolation: true,   // higher quality than trilinear
  } );

  composer.addPass( new EffectPass( camera, lut3d ) );
} );
```

The `tetrahedralInterpolation` flag is significant: it uses tetrahedral
interpolation within each cube cell rather than trilinear interpolation, which
reduces color shift artefacts in low-size LUTs (e.g. 17×17×17). The Three.js
`LUTPass` always uses hardware trilinear sampling via `LinearFilter`.

---

## Performance Notes

- LUT sampling from a 3D texture is a single `texture3D()` call in the fragment
  shader. It is extremely cheap — typically < 0.1ms on desktop, < 0.3ms on
  mobile.
- LUT texture size affects quality, not performance (GPU sampler handles it at
  fixed cost). A 33³ or 64³ LUT is standard; 128³ is rarely necessary.
- The OutputPass recompiles its internal `RawShaderMaterial` when
  `renderer.toneMapping` or `renderer.outputColorSpace` changes. Changing these
  values each frame causes shader recompile stalls. Set them once at startup.
- For animated LUT blending (e.g. mood transition as time of day changes),
  animate `lutPass.intensity` between 0 and 1 rather than swapping the LUT
  texture. Swapping requires a texture upload; tweening intensity is free.
- Multiple LUTs (e.g. day LUT and night LUT): load both at startup, blend
  between them via intensity. Do not create multiple LUTPass instances; one
  pass with a ShaderPass blending two `texture3D` samples is more efficient.
