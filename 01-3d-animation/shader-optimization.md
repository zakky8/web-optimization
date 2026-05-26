# Shader Optimization

## Precompilation - Eliminate First-Frame Stutter

Without this: GPU freezes for 1-4 seconds on first frame with complex materials.

```javascript
// Async shader compilation - uses KHR_parallel_shader_compile
await renderer.compileAsync(scene, camera)
// Call this behind a loading screen, before the scene is revealed
// Returns Promise - resolves when all shaders are compiled
```

Igloo Inc: implemented background shader compilation as part of loading strategy.
Named as one of their key optimizations in the Awwwards case study.

## Precision - Mobile vs Desktop

Desktop silently upgrades mediump -> highp. Mobile does not.
Bugs you write will NOT appear in desktop dev, only on device.

```glsl
// Always declare explicitly at top of every fragment shader
precision highp float;   // for positions, depth, normals

// Safe to drop for color calculations (0-1 range values)
// mediump is ~2x faster than highp on mobile
precision mediump float;

// NEVER use lowp for anything visual
```

## Replace Branching with Math

GPU parallelism is destroyed by if/else (divergent branches).

```glsl
// BAD - creates divergent shader paths
if (value > 0.5) {
  color = colorA;
} else {
  color = colorB;
}

// GOOD - single math path, fully parallel
color = mix(colorB, colorA, step(0.5, value));

// BAD
float result;
if (flag > 0.0) {
  result = a * b;
} else {
  result = a + b;
}

// GOOD
float result = mix(a + b, a * b, step(0.0, flag));
```

## Varying Variables - Mobile Limit

Keep varying variables under 3 on mobile.
Pack multiple values into RGBA channels to stay within limits.

```glsl
// BAD - 4 varyings
varying float vNoise;
varying float vElevation;
varying float vShadow;
varying float vAO;

// GOOD - pack into one vec4 varying
varying vec4 vData;
// vData.r = noise
// vData.g = elevation
// vData.b = shadow
// vData.a = AO
```

## TSL (Three.js Shading Language) - Bruno Simon '25

Write shaders in JavaScript, compile to GLSL (WebGL) or WGSL (WebGPU).

```javascript
import { color, uniform, texture, normalLocal, positionLocal } from 'three/tsl'

const material = new THREE.MeshStandardNodeMaterial()
material.colorNode = texture(colorMap)
material.normalNode = normalMap(texture(normalMapTex))
// Compiles to GLSL for WebGL, WGSL for WebGPU automatically
```

Available Three.js r155+. Used in production by Bruno Simon '25.

## Matcap Textures - Zero Cost Lighting

Replace all lights and shadow calculations with a single texture lookup.

```javascript
const material = new THREE.MeshMatcapMaterial({
  matcap: matcapTexture
})
// Camera-relative normal lookup on 2D image
// Simulates complex lighting with zero GPU computation
// No lights needed in scene
```

Bruno Simon: no lights, no dynamic shadows, matcap throughout entire scene.
Lusion: matcap for all translucent objects, pre-rendered material maps for everything else.

## Bayer Matrix Dithering (Stas Bondar '25)

Ordered dithering shader for stylized rendering at lower bit depths.

```glsl
// Bayer matrix dithering - creates halftone/grain effect
const float bayer[64] = float[64](
   0.0/64.0,  32.0/64.0, ...
);

void main() {
  vec2 coord = gl_FragCoord.xy;
  int index = int(mod(coord.x, 8.0)) + int(mod(coord.y, 8.0)) * 8;
  float threshold = bayer[index];
  float dithered = step(threshold, luminance);
  gl_FragColor = vec4(vec3(dithered), 1.0);
}
```

Used for: stylized B&W scenes, retro aesthetics, reducing color banding.
