# PBR Injection — onBeforeCompile + glslify

## THREE.MeshStandardMaterial.onBeforeCompile

Three.js's built-in PBR shader uses named `#include` chunks. `onBeforeCompile` lets you
inject custom code via `String.replace()`.

```javascript
const material = new THREE.MeshStandardMaterial({
  color: 0xffffff,
  roughness: 0.5,
  metalness: 0.0,
});

material.onBeforeCompile = (shader) => {
  shader.uniforms.uTime     = { value: 0 };
  shader.uniforms.uNoiseAmp = { value: 0.1 };

  // Store ref for animation loop updates
  material.userData.shader = shader;

  // Inject uniforms + helpers after #include <common>
  shader.vertexShader = shader.vertexShader.replace(
    '#include <common>',
    `#include <common>
     uniform float uTime;
     uniform float uNoiseAmp;
     // ... paste snoise() here
    `
  );

  // Displace vertices
  shader.vertexShader = shader.vertexShader.replace(
    '#include <begin_vertex>',
    `#include <begin_vertex>
     float n = snoise(vec3(transformed.xy * 2.0, uTime));
     transformed += normal * n * uNoiseAmp;
    `
  );

  // Modify fragment emissive
  shader.fragmentShader = shader.fragmentShader.replace(
    '#include <emissivemap_fragment>',
    `#include <emissivemap_fragment>
     totalEmissiveRadiance += vec3(n * 0.5 + 0.5) * 0.3;
    `
  );
};

// In animation loop:
if (material.userData.shader) {
  material.userData.shader.uniforms.uTime.value = clock.getElapsedTime();
}
```

## Key Injection Hooks

| Hook token | Purpose |
|---|---|
| `#include <common>` | Declare uniforms, varyings, helper functions |
| `#include <begin_vertex>` | Access `transformed` vertex position |
| `#include <project_vertex>` | After clip-space projection |
| `#include <color_fragment>` | Modify `diffuseColor` |
| `#include <emissivemap_fragment>` | Modify `totalEmissiveRadiance` |
| `#include <roughnessmap_fragment>` | Modify `roughnessFactor` |
| `#include <normal_fragment_begin>` | Modify `normal` in fragment |
| `#include <dithering_fragment>` | Last hook before output |

## Fresnel Rim via onBeforeCompile

```javascript
material.onBeforeCompile = (shader) => {
  shader.uniforms.uRimColor = { value: new THREE.Color(0x0088ff) };
  shader.uniforms.uRimPower = { value: 3.0 };
  material.userData.shader = shader;

  shader.fragmentShader = shader.fragmentShader.replace(
    '#include <common>',
    `#include <common>
    uniform vec3 uRimColor;
    uniform float uRimPower;
    `
  );

  shader.fragmentShader = shader.fragmentShader.replace(
    '#include <dithering_fragment>',
    `#include <dithering_fragment>
    float NdotV = dot(vNormal, normalize(cameraPosition - vViewPosition));
    float rim = pow(1.0 - abs(NdotV), uRimPower);
    gl_FragColor.rgb += uRimColor * rim;
    `
  );
};
```

## glslify — npm Module System

```bash
npm install glslify vite-plugin-glslify
```

```javascript
// vite.config.js
import glslify from 'vite-plugin-glslify';
export default { plugins: [glslify()] };
```

```glsl
// shader.frag
#pragma glslify: snoise3 = require(glsl-noise/simplex/3d)
#pragma glslify: fog     = require(glsl-fog/exp2)

void main() {
  float n = snoise3(vec3(vUv, uTime));
  gl_FragColor = vec4(vec3(n), 1.0);
}
```

Export your own:
```glsl
// myNoise.glsl
#pragma glslify: snoise = require(glsl-noise/simplex/2d)
float myNoise(vec2 uv) { return snoise(uv) * 0.5 + 0.5; }
#pragma glslify: export(myNoise)
```

## SDF Raymarching Reference

```glsl
// Signed distance primitives (Inigo Quilez)
// Source: https://iquilezles.org/articles/distfunctions/

float sdSphere(vec3 p, float r)           { return length(p) - r; }
float sdBox(vec3 p, vec3 b)               {
  vec3 q = abs(p) - b;
  return length(max(q, 0.0)) + min(max(q.x, max(q.y, q.z)), 0.0);
}
float sdTorus(vec3 p, vec2 t)             {
  vec2 q = vec2(length(p.xz) - t.x, p.y);
  return length(q) - t.y;
}
float opSmoothUnion(float d1, float d2, float k) {
  float h = clamp(0.5 + 0.5*(d2-d1)/k, 0.0, 1.0);
  return mix(d2, d1, h) - k*h*(1.0-h);
}

// Minimal raymarcher
float map(vec3 p) {
  return opSmoothUnion(
    sdSphere(p - vec3(0.5, 0.0, 0.0), 0.5),
    sdBox(p + vec3(0.5, 0.0, 0.0), vec3(0.4)),
    0.2
  );
}

vec3 calcNormal(vec3 p) {
  const vec2 e = vec2(0.001, 0.0);
  return normalize(vec3(
    map(p + e.xyy) - map(p - e.xyy),
    map(p + e.yxy) - map(p - e.yxy),
    map(p + e.yyx) - map(p - e.yyx)
  ));
}
```

## GitHub Repos
- https://github.com/patriciogonzalezvivo/lygia
- https://github.com/stegu/webgl-noise
- https://github.com/glslify/glslify
- https://github.com/hughsk/glsl-noise
- https://github.com/pmndrs/three-stdlib
