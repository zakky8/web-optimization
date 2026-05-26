# Converting Shadertoy Shaders to Three.js

**Last verified:** 2026-05-26

---

## Overview

Shadertoy shaders are self-contained GLSL fragment programs that run inside Shadertoy's fixed rendering harness. Three.js uses raw WebGL shaders where you control the harness yourself. The mapping is mechanical but every piece must be translated explicitly — nothing is injected automatically.

---

## Uniform Mapping Reference

| Shadertoy uniform | GLSL type | Three.js equivalent | Notes |
|-------------------|-----------|---------------------|-------|
| `iTime` | `float` | `uTime` | Pass `clock.getElapsedTime()` each frame |
| `iTimeDelta` | `float` | `uTimeDelta` | Frame delta in seconds |
| `iFrame` | `int` | `uFrame` | Increment manually each render call |
| `iResolution` | `vec3` | `uResolution` | `.xy` = width/height in px; `.z` = pixel ratio (Shadertoy) — usually only `.xy` needed |
| `iMouse` | `vec4` | `uMouse` | `.xy` = current pos; `.zw` = click pos; map from `pointermove` events |
| `iChannel0` | `sampler2D` (or `samplerCube`) | `uChannel0` / `uTexture` | Set via `THREE.TextureLoader` |
| `iChannel1–3` | same | `uChannel1–3` | Same pattern |
| `iChannelResolution[N]` | `vec3[4]` | `uChannelResolution[N]` | Texture dimensions per channel |
| `iChannelTime[N]` | `float[4]` | `uChannelTime[N]` | Playback time for video textures |
| `iDate` | `vec4` | `uDate` | `.xyzw` = year, month, day, seconds-since-midnight |
| `iSampleRate` | `float` | `uSampleRate` | Audio sample rate (44100 typically) |

---

## Entry Point: `mainImage` → `void main()`

**Shadertoy signature:**

```glsl
void mainImage(out vec4 fragColor, in vec2 fragCoord) {
  // ...
  fragColor = vec4(color, 1.0);
}
```

**Three.js fragment shader:**

```glsl
void main() {
  vec2 fragCoord = gl_FragCoord.xy;
  // ... same body ...
  gl_FragColor = vec4(color, 1.0);
}
```

Steps:
1. Rename `mainImage(out vec4 fragColor, in vec2 fragCoord)` to `void main()`.
2. Replace all `fragColor =` with `gl_FragColor =`.
3. Replace all `fragCoord` with `gl_FragCoord.xy` (or declare `vec2 fragCoord = gl_FragCoord.xy;` at the top of `main()`).

---

## Coordinate System: Y-Axis Flip

Shadertoy's `fragCoord` origin is **bottom-left** (standard OpenGL). Three.js `gl_FragCoord` is also bottom-left, so there is no flip needed for `gl_FragCoord` itself.

However, Shadertoy normalizes UV coordinates as:

```glsl
vec2 uv = fragCoord / iResolution.xy;  // (0,0) bottom-left, (1,1) top-right
```

If your Shadertoy shader writes `uv.y` in a way that assumes top-left origin (common when sampling textures that were authored top-to-bottom), flip manually:

```glsl
vec2 uv = gl_FragCoord.xy / uResolution.xy;
uv.y = 1.0 - uv.y;  // flip if texture was loaded with flipY:false
```

Three.js `TextureLoader` loads textures with `flipY = true` by default (matching WebGL convention), so most Shadertoy texture reads will not need a manual flip.

---

## Full Conversion Example

### Original Shadertoy

```glsl
// Shadertoy — paste into https://www.shadertoy.com/new
uniform sampler2D iChannel0;

void mainImage(out vec4 fragColor, in vec2 fragCoord) {
    vec2 uv  = fragCoord / iResolution.xy;
    vec3 col = texture(iChannel0, uv).rgb;
    col     *= 0.5 + 0.5 * sin(iTime * 2.0 + uv.xyx + vec3(0, 2, 4));
    fragColor = vec4(col, 1.0);
}
```

### Converted Three.js Fragment Shader (`shader.frag`)

```glsl
precision mediump float;

uniform float     uTime;
uniform vec2      uResolution;
uniform sampler2D uChannel0;

void main() {
    vec2 uv  = gl_FragCoord.xy / uResolution;
    vec3 col = texture2D(uChannel0, uv).rgb;
    col     *= 0.5 + 0.5 * sin(uTime * 2.0 + uv.xyx + vec3(0.0, 2.0, 4.0));
    gl_FragColor = vec4(col, 1.0);
}
```

**Notable change:** `texture()` → `texture2D()` if targeting WebGL 1 (GLSL ES 1.00). In WebGL 2 (GLSL ES 3.00, with `#version 300 es`) `texture()` is correct.

---

### Three.js Material Setup

```js
import * as THREE from 'three';
import vertSrc from './shader.vert';
import fragSrc from './shader.frag';

const clock    = new THREE.Clock();
const texture  = new THREE.TextureLoader().load('/assets/noise.png');
texture.wrapS  = THREE.RepeatWrapping;
texture.wrapT  = THREE.RepeatWrapping;

const uniforms = {
  uTime:       { value: 0 },
  uResolution: { value: new THREE.Vector2(window.innerWidth, window.innerHeight) },
  uChannel0:   { value: texture },
};

const material = new THREE.ShaderMaterial({
  vertexShader:   vertSrc,
  fragmentShader: fragSrc,
  uniforms,
});

// In your animation loop:
function animate() {
  uniforms.uTime.value = clock.getElapsedTime();
  renderer.render(scene, camera);
  requestAnimationFrame(animate);
}
```

---

## WebGL 1 vs WebGL 2 GLSL Differences

Shadertoy defaults to GLSL ES 3.00 (WebGL 2). Three.js defaults to WebGL 2 as well but falls back to WebGL 1. Key differences:

| Shadertoy (GLSL ES 3.00) | WebGL 1 equivalent (GLSL ES 1.00) |
|--------------------------|-----------------------------------|
| `texture(sampler, uv)` | `texture2D(sampler, uv)` |
| `in` / `out` qualifiers | `varying` |
| `#version 300 es` required for ES 3.00 | No version directive needed |
| `layout(location=0) out vec4 fragColor` | Use `gl_FragColor` |

When targeting WebGL 2 explicitly in Three.js:

```js
const renderer = new THREE.WebGLRenderer();
// Three.js uses WebGL 2 by default in r150+
```

Add `#version 300 es` as the first line of your shader and use ES 3.00 syntax throughout.

---

## Common Pitfalls

| Symptom | Cause | Fix |
|---------|-------|-----|
| Black screen, no errors | `mainImage` not renamed | Replace with `void main()` |
| Texture appears vertically flipped | `flipY` mismatch | Set `texture.flipY = false` or flip UV manually |
| `iResolution.z` used | Pixel ratio dependency | Replace with `1.0` or pass `window.devicePixelRatio` as a separate uniform |
| Integer literals in float expressions | Strict GLSL typing | Change `1` → `1.0`, `2` → `2.0` throughout |
| `iMouse.zw` for click state | Click position not wired | Add a `pointerdown` listener and update uniform |
| Gradient banding | Missing `precision` directive | Add `precision highp float;` at shader top |

---

## iMouse Wiring Example

```js
const mouse = new THREE.Vector4(0, 0, 0, 0);
const uniforms = { uMouse: { value: mouse } };

renderer.domElement.addEventListener('pointermove', (e) => {
  mouse.x = e.clientX;
  mouse.y = window.innerHeight - e.clientY;  // flip Y to match OpenGL origin
});

renderer.domElement.addEventListener('pointerdown', (e) => {
  mouse.z = e.clientX;
  mouse.w = window.innerHeight - e.clientY;
});
```

---

## Sources

| Source | URL | Tier | Accessed |
|--------|-----|------|----------|
| Shadertoy howto (uniforms reference) | https://www.shadertoy.com/howto | 1 | 2026-05-26 — returned 403; uniform list cross-checked against Shadertoy API documentation |
| Three.js ShaderMaterial docs | https://threejs.org/docs/#api/en/materials/ShaderMaterial | 1 | 2026-05-26 |
| glslCanvas built-in uniforms (patriciogonzalezvivo) | https://github.com/patriciogonzalezvivo/glslCanvas | 1 | 2026-05-26 |

**Note:** The Shadertoy uniform list (`iTime`, `iResolution`, `iMouse`, `iChannel0-3`, `iDate`, `iFrame`, `iChannelTime`, `iChannelResolution`, `iSampleRate`) is drawn from the publicly documented Shadertoy shader input specification. The Shadertoy howto page returned HTTP 403 on 2026-05-26; the uniform names and types are consistent with the documented API and widely corroborated by independent sources (UNVERIFIED for exact current wording from primary page).
