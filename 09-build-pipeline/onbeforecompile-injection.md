# onBeforeCompile â€” Shader Injection into PBR Materials

## What It Does

`onBeforeCompile` lets you inject GLSL code into Three.js's built-in PBR shaders
(MeshStandardMaterial, MeshPhysicalMaterial, etc.) without rewriting the entire shader.

## Pattern

```js
const mat = new THREE.MeshStandardMaterial({ color: '#ffffff' });

mat.onBeforeCompile = (shader) => {
  // 1. Add uniforms
  shader.uniforms.uTime  = { value: 0 };
  shader.uniforms.uNoise = { value: noiseTexture };

  // Keep reference for per-frame updates
  mat.userData.shader = shader;

  // 2. Inject into vertex shader
  // Insert declarations before main()
  shader.vertexShader = shader.vertexShader.replace(
    '#include <common>',
    `#include <common>
    uniform float uTime;
    varying vec2 vCustomUv;`
  );

  // Inject into main() body
  shader.vertexShader = shader.vertexShader.replace(
    '#include <begin_vertex>',
    `#include <begin_vertex>
    // Displace geometry
    transformed.y += sin(transformed.x * 3.0 + uTime) * 0.1;
    vCustomUv = uv;`
  );

  // 3. Inject into fragment shader
  shader.fragmentShader = shader.fragmentShader.replace(
    '#include <common>',
    `#include <common>
    uniform sampler2D uNoise;
    varying vec2 vCustomUv;`
  );

  shader.fragmentShader = shader.fragmentShader.replace(
    '#include <dithering_fragment>',
    `#include <dithering_fragment>
    // Tint final color with noise
    float n = texture2D(uNoise, vCustomUv).r;
    gl_FragColor.rgb *= mix(0.8, 1.2, n);`
  );
};

// Update uniforms per frame
renderer.setAnimationLoop(() => {
  if (mat.userData.shader) {
    mat.userData.shader.uniforms.uTime.value += 0.016;
  }
});
```

## Common Injection Points

### Vertex Shader
| Token | When to Use |
|-------|-------------|
| `#include <common>` | Add declarations, uniforms, varyings |
| `#include <begin_vertex>` | Access/modify `transformed` (local position) |
| `#include <project_vertex>` | After MVP transform, before gl_Position |
| `#include <worldpos_vertex>` | Access `worldPosition` |

### Fragment Shader
| Token | When to Use |
|-------|-------------|
| `#include <common>` | Add declarations, uniforms, varyings |
| `#include <map_fragment>` | After base color, modify `diffuseColor` |
| `#include <roughnessmap_fragment>` | Modify `roughnessFactor` |
| `#include <emissivemap_fragment>` | Add emission |
| `#include <dithering_fragment>` | Last step â€” modify final `gl_FragColor` |
| `vec4 diffuseColor` | Access base color variable |

## Finding All Tokens

```js
// Print the full shader source to see all #include tokens
mat.onBeforeCompile = (shader) => {
  console.log('--- VERTEX ---');
  console.log(shader.vertexShader);
  console.log('--- FRAGMENT ---');
  console.log(shader.fragmentShader);
};
```

Alternatively, browse `three/src/renderers/shaders/ShaderLib/` in source.

## customProgramCacheKey

When variants exist (e.g., with/without effect), prevent shader cache collisions:

```js
mat.customProgramCacheKey = () => {
  return `myMat-${mat.userData.variant}-${mat.userData.quality}`;
};
```

## Sources
- Three.js Material.onBeforeCompile: https://threejs.org/docs/#api/en/materials/Material.onBeforeCompile
- ShaderLib sources: three/src/renderers/shaders/ShaderLib/
- Three.js journey lesson on custom shaders (Bruno Simon)
