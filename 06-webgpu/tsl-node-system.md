# TSL â€” Three.js Shading Language

TSL is Three.js's node-based shader system that compiles to WGSL (WebGPU) or GLSL (WebGL2 fallback).
Source: `three/src/nodes/` â€” r184

## Core Primitives

```js
import {
  Fn, uniform, vec2, vec3, vec4,
  float, int, uint, bool,
  positionWorld, normalWorld,
  uv, time, modelWorldMatrix,
  instancedArray, storageBuffer,
  If, Loop, Continue, Break,
  mix, smoothstep, clamp, length, normalize
} from 'three/tsl';
```

## Declaring Uniforms

```js
const uTime    = uniform(0.0);           // float uniform
const uColor   = uniform(new THREE.Color('#ff6600'));  // vec3
const uTex     = uniform(texture);       // sampler2D

// Update every frame â€” no glUniform* calls needed
renderer.setAnimationLoop(() => {
  uTime.value += 0.016;
});
```

## Writing a Node Material

```js
import { MeshStandardNodeMaterial } from 'three/nodes';

const mat = new MeshStandardNodeMaterial();

// colorNode replaces map + color
mat.colorNode = Fn(() => {
  const t = uTime.mul(0.5);             // WGSL: time * 0.5
  const n = positionWorld.x.add(t);     // position.x + t
  return vec4(sin(n).mul(0.5).add(0.5), 0.8, 1.0, 1.0);
})();
```

## Compute Shaders (GPGPU)

```js
import { computeShader, instancedArray } from 'three/tsl';

const PARTICLE_COUNT = 512 * 512;

// Typed storage arrays
const positions = instancedArray(PARTICLE_COUNT, 'vec3');
const velocities = instancedArray(PARTICLE_COUNT, 'vec3');

// Compute node â€” runs on GPU
const updateParticles = Fn(() => {
  const i     = instanceIndex;
  const pos   = positions.element(i);
  const vel   = velocities.element(i);

  vel.addAssign(vec3(0, -0.001, 0));    // gravity
  pos.addAssign(vel);

  // Bounds check
  If(pos.y.lessThan(-5.0), () => {
    pos.y.assign(-5.0);
    vel.y.assign(vel.y.abs());
  });
})();

const computeNode = computeShader(updateParticles, PARTICLE_COUNT);

// Each frame
await renderer.computeAsync(computeNode);
```

## Instanced Arrays vs Storage Buffers

| API | Use Case | Read/Write |
|-----|----------|------------|
| `instancedArray(count, type)` | Per-instance data (positions, colors) | R/W from compute, R from vertex |
| `storageBuffer(buffer, layout)` | Arbitrary structured data | R/W |
| `uniform(value)` | Single shared value | R from shader |

## Built-in Node Variables

```js
positionWorld      // vec3 â€” world-space position
positionLocal      // vec3 â€” local-space position
normalWorld        // vec3 â€” world-space normal
uv()               // vec2 â€” UV coords (default channel 0)
uv(1)              // vec2 â€” UV channel 1
instanceIndex      // uint â€” gl_InstanceID equivalent
vertexIndex        // uint â€” gl_VertexID equivalent
time               // float â€” elapsed seconds
deltaTime          // float â€” frame delta
```

## Sources
- Three.js TSL docs: https://github.com/mrdoob/three.js/wiki/Three.js-Shading-Language
- TSL examples: https://threejs.org/examples/?q=tsl
- Node source: three/src/nodes/ (r184)
