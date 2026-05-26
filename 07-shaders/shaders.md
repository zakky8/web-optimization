# Shaders

Shaders are programs that run on the GPU. In the context of Three.js and WebGL they are written in GLSL ES 3.0. Understanding shader patterns, reusable libraries, and noise functions is the difference between static meshes and living, breathing visuals.

## GLSL Fundamentals in Three.js

### ShaderMaterial Boilerplate

```js
import * as THREE from "three";

const mat = new THREE.ShaderMaterial({
  uniforms: {
    uTime:       { value: 0 },
    uResolution: { value: new THREE.Vector2(window.innerWidth, window.innerHeight) },
    uTexture:    { value: texture },
  },
  vertexShader: /* glsl */ `
    uniform float uTime;
    varying vec2 vUv;
    varying vec3 vNormal;

    void main() {
      vUv     = uv;
      vNormal = normalize(normalMatrix * normal);

      vec3 pos = position;
      pos.y += sin(pos.x * 4.0 + uTime) * 0.1;

      gl_Position = projectionMatrix * modelViewMatrix * vec4(pos, 1.0);
    }
  `,
  fragmentShader: /* glsl */ `
    uniform float uTime;
    uniform sampler2D uTexture;
    varying vec2 vUv;
    varying vec3 vNormal;

    void main() {
      vec3 color = texture2D(uTexture, vUv).rgb;
      float fresnel = pow(1.0 - dot(vNormal, vec3(0.0, 0.0, 1.0)), 3.0);
      color = mix(color, vec3(0.3, 0.6, 1.0), fresnel);
      gl_FragColor = vec4(color, 1.0);
    }
  `,
});

// Update time each frame
renderer.setAnimationLoop(() => {
  mat.uniforms.uTime.value += 0.016;
  renderer.render(scene, camera);
});
```

### GLSL Data Types Quick Reference

```glsl
// Scalars
float  x = 1.0;      // always use decimal point
int    n = 3;
bool   b = true;

// Vectors
vec2 uv  = vec2(0.5, 0.5);
vec3 pos = vec3(1.0, 2.0, 3.0);
vec4 col = vec4(pos, 1.0);      // w = alpha

// Swizzling
vec3 rgb = col.rgb;             // .xyz .rgb .stp all valid
vec2 yx  = uv.yx;              // reorder components
float r  = col.r;

// Matrices
mat4 mvp = projectionMatrix * modelViewMatrix;

// Texture sampling
vec4 t = texture2D(uTex, vUv);
```

## Procedural Noise

Noise is the backbone of organic-looking shaders — terrain, smoke, water, skin, fire.

### Value Noise

```glsl
float hash(vec2 p) {
  return fract(sin(dot(p, vec2(127.1, 311.7))) * 43758.5453);
}

float valueNoise(vec2 p) {
  vec2 i = floor(p);
  vec2 f = fract(p);
  vec2 u = f * f * (3.0 - 2.0 * f); // smoothstep

  float a = hash(i);
  float b = hash(i + vec2(1.0, 0.0));
  float c = hash(i + vec2(0.0, 1.0));
  float d = hash(i + vec2(1.0, 1.0));

  return mix(mix(a, b, u.x), mix(c, d, u.x), u.y);
}
```

### Gradient Noise (Classic Perlin)

Perlin noise uses gradient vectors instead of random values, giving smoother, more natural results.

```glsl
vec2 gradient(vec2 p) {
  float angle = hash(p) * 6.283185;
  return vec2(cos(angle), sin(angle));
}

float perlin(vec2 p) {
  vec2 i = floor(p);
  vec2 f = fract(p);
  vec2 u = f * f * f * (f * (f * 6.0 - 15.0) + 10.0); // quintic

  float a = dot(gradient(i + vec2(0.0, 0.0)), f - vec2(0.0, 0.0));
  float b = dot(gradient(i + vec2(1.0, 0.0)), f - vec2(1.0, 0.0));
  float c = dot(gradient(i + vec2(0.0, 1.0)), f - vec2(0.0, 1.0));
  float d = dot(gradient(i + vec2(1.0, 1.0)), f - vec2(1.0, 1.0));

  return mix(mix(a, b, u.x), mix(c, d, u.x), u.y) * 0.5 + 0.5;
}
```

### Simplex Noise (Faster, No Axis Artifacts)

Simplex noise is O(n^2) in n dimensions vs O(2^n) for Perlin — significant for 3D and 4D noise.

```glsl
// Stefan Gustavson's implementation (public domain)
vec3 mod289(vec3 x) { return x - floor(x * (1.0 / 289.0)) * 289.0; }
vec2 mod289(vec2 x) { return x - floor(x * (1.0 / 289.0)) * 289.0; }
vec3 permute(vec3 x) { return mod289(((x * 34.0) + 10.0) * x); }

float snoise(vec2 v) {
  const vec4 C = vec4(0.211324865405187, 0.366025403784439,
                     -0.577350269189626, 0.024390243902439);
  vec2 i  = floor(v + dot(v, C.yy));
  vec2 x0 = v -   i + dot(i, C.xx);
  vec2 i1 = (x0.x > x0.y) ? vec2(1.0, 0.0) : vec2(0.0, 1.0);
  vec4 x12 = x0.xyxy + C.xxzz;
  x12.xy -= i1;
  i = mod289(i);
  vec3 p = permute(permute(i.y + vec3(0.0, i1.y, 1.0))
                 + i.x + vec3(0.0, i1.x, 1.0));
  vec3 m = max(0.5 - vec3(dot(x0, x0), dot(x12.xy, x12.xy),
                          dot(x12.zw, x12.zw)), 0.0);
  m = m * m; m = m * m;
  vec3 x  = 2.0 * fract(p * C.www) - 1.0;
  vec3 h  = abs(x) - 0.5;
  vec3 ox = floor(x + 0.5);
  vec3 a0 = x - ox;
  m *= 1.79284291400159 - 0.85373472095314 * (a0 * a0 + h * h);
  vec3 g;
  g.x  = a0.x  * x0.x   + h.x  * x0.y;
  g.yz = a0.yz * x12.xz + h.yz * x12.yw;
  return 130.0 * dot(m, g);
}
```

### Fractal Brownian Motion (fBm)

Stack octaves of noise for cloud, terrain, and turbulence effects:

```glsl
float fbm(vec2 p) {
  float value = 0.0;
  float amplitude = 0.5;
  float frequency = 1.0;
  // 5 octaves is typical — each doubles frequency, halves amplitude
  for (int i = 0; i < 5; i++) {
    value     += amplitude * snoise(p * frequency);
    amplitude *= 0.5;
    frequency *= 2.0;
  }
  return value;
}
```

## LYGIA Shader Library

LYGIA is a granular, multi-language shader library (GLSL, HLSL, Metal, WGSL) designed for performance and reusability. Import only what you need.

### CDN Import via #include

```glsl
#ifdef GL_ES
precision mediump float;
#endif

// CDN-based import (resolves at shader compile time via lygia-glsl package or cdn.lygia.xyz)
#include "lygia/space/ratio.glsl"
#include "lygia/math/PI.glsl"
#include "lygia/generative/noised.glsl"
#include "lygia/color/palette/spectral.glsl"

uniform float u_time;
uniform vec2  u_resolution;

void main() {
  vec2 st = gl_FragCoord.xy / u_resolution;
  st = ratio(st, u_resolution);

  float n = noised(st + u_time * 0.2).x;
  vec3  c = spectral_zucconi6(n * 0.5 + 0.5);

  gl_FragColor = vec4(c, 1.0);
}
```

### Using LYGIA with Three.js

```js
import { loadLygia } from "lygia";  // npm install lygia

const fragmentShader = await loadLygia(`
  #include "lygia/generative/curl.glsl"
  #include "lygia/color/blend/screen.glsl"

  uniform float uTime;
  varying vec2 vUv;

  void main() {
    vec3 curl = curlNoise(vec3(vUv * 3.0, uTime * 0.1));
    gl_FragColor = vec4(curl * 0.5 + 0.5, 1.0);
  }
`);
```

### LYGIA Module Categories

| Category | Examples |
|---|---|
| `math/` | PI, TWO_PI, PHI, saturate, remap, quat |
| `space/` | ratio, rotate, scale, mirror |
| `color/` | luma, saturation, hue, blend modes, palette |
| `generative/` | noise, curl, fbm, voronoi, random |
| `lighting/` | diffuse, specular, shadow, BRDF |
| `filter/` | blur, sharpen, edge detection, median |

## Advanced Shader Patterns

### Signed Distance Functions (SDFs)

SDFs define geometry as a math function rather than polygons. Perfect for procedural shapes, ray marching.

```glsl
float sdCircle(vec2 p, float r) {
  return length(p) - r;
}
float sdBox(vec2 p, vec2 b) {
  vec2 d = abs(p) - b;
  return length(max(d, 0.0)) + min(max(d.x, d.y), 0.0);
}
float sdRoundedBox(vec2 p, vec2 b, float r) {
  vec2 d = abs(p) - b + r;
  return min(max(d.x, d.y), 0.0) + length(max(d, 0.0)) - r;
}

// Smooth union — blend two SDFs
float sdfUnion(float d1, float d2, float k) {
  float h = clamp(0.5 + 0.5 * (d2 - d1) / k, 0.0, 1.0);
  return mix(d2, d1, h) - k * h * (1.0 - h);
}
```

### Fresnel Effect

```glsl
// vNormal must be in view space
float fresnel = pow(1.0 - max(0.0, dot(vNormal, normalize(-vViewPos))), 3.0);
vec3 fresnelColor = mix(baseColor, rimColor, fresnel);
```

### UV Distortion / Displacement

```glsl
uniform sampler2D uNormalMap;
uniform float uStrength;

vec2 distort(vec2 uv, float strength) {
  vec3 n = texture2D(uNormalMap, uv + uTime * 0.05).rgb * 2.0 - 1.0;
  return uv + n.xy * strength;
}

void main() {
  vec2 distortedUv = distort(vUv, 0.02);
  vec4 color = texture2D(uTexture, distortedUv);
  gl_FragColor = color;
}
```

### Color Palette (Inigo Quilez)

Compact parametric palettes without texture lookups:

```glsl
vec3 palette(float t, vec3 a, vec3 b, vec3 c, vec3 d) {
  return a + b * cos(6.28318 * (c * t + d));
}

// Usage
vec3 col = palette(
  t,
  vec3(0.5, 0.5, 0.5),   // a: brightness
  vec3(0.5, 0.5, 0.5),   // b: contrast
  vec3(1.0, 1.0, 1.0),   // c: frequency
  vec3(0.0, 0.33, 0.67)  // d: phase offset
);
```

## Performance Implications

- **Branch divergence**: `if` statements inside fragment shaders can halve throughput. Prefer `mix()`, `step()`, `smoothstep()`, and `clamp()`.
- **Texture lookups**: each `texture2D` call is expensive on mobile. Combine multiple data channels in one texture (r=roughness, g=metallic, b=ao).
- **Precision qualifiers**: `lowp` and `mediump` are faster on mobile but introduce visible artifacts with gradients. Use `mediump` as default, `highp` only where needed.
- **Loop unrolling**: GLSL compilers typically unroll `for` loops with constant bounds. Variable-bound loops prevent this.
- **Discard**: `discard` stalls the early depth test optimization. Avoid it unless alpha cutout requires it.
- **Varying count**: limit varyings. More than 8 varyings can exceed limits on some mobile GPUs.

## Tools and Libraries

| Tool | Purpose |
|---|---|
| [LYGIA](https://github.com/patriciogonzalezvivo/lygia) | Multi-language shader library |
| [The Book of Shaders](https://thebookofshaders.com) | Canonical GLSL learning resource |
| [Shadertoy](https://www.shadertoy.com) | Community shader examples |
| [GLSL Noise gist](https://gist.github.com/patriciogonzalezvivo/670c22f3966e662d2f83) | Noise algorithm collection |
| [glslify](https://github.com/glslify/glslify) | Node-style GLSL module system |
| [Inigo Quilez SDF functions](https://iquilezles.org/articles/distfunctions/) | SDF reference |
| Chrome GLSL DevTools | Shader editor inline |
