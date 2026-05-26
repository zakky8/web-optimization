# Particle Rendering — Billboards, Soft Particles, Points vs Instanced

## Particle Geometry Setup

Each particle in the render pass is a vertex. The vertex has a UV that maps to its texel.

```javascript
const COUNT = WIDTH * WIDTH; // 512*512 = 262,144
const geometry = new THREE.BufferGeometry();

// UV coordinates: map each vertex to its simulation texel
const uvs = new Float32Array(COUNT * 2);
for (let i = 0; i < WIDTH; i++) {
  for (let j = 0; j < WIDTH; j++) {
    const idx = (i * WIDTH + j);
    uvs[idx * 2 + 0] = i / (WIDTH - 1); // u
    uvs[idx * 2 + 1] = j / (WIDTH - 1); // v
  }
}
geometry.setAttribute('uv', new THREE.BufferAttribute(uvs, 2));
geometry.setAttribute('position', new THREE.BufferAttribute(new Float32Array(COUNT * 3), 3));
```

## Vertex Shader (Billboard + Size Attenuation)

```glsl
attribute vec2 uv;
uniform sampler2D uTexturePosition;
uniform sampler2D uTextureVelocity;
uniform float uParticleSize;

varying vec2 vUv;
varying vec3 vVelocity;

void main() {
  vUv = uv;

  vec4 posData = texture2D(uTexturePosition, uv);
  vec3 pos = posData.xyz;
  vec4 velData = texture2D(uTextureVelocity, uv);
  vVelocity = velData.xyz;

  vec4 mvPosition = modelViewMatrix * vec4(pos, 1.0);

  // Size attenuation
  float distanceFactor = 1.0 / -mvPosition.z;
  gl_PointSize = uParticleSize * distanceFactor * 600.0;

  // Size by speed
  float speed = length(velData.xyz);
  gl_PointSize *= (1.0 + speed * 20.0);

  gl_Position = projectionMatrix * mvPosition;
}
```

## Fragment Shader (Soft Particles + Depth Fade)

```glsl
uniform sampler2D uDepthTexture;
uniform vec2 uResolution;
uniform float uCameraNear;
uniform float uCameraFar;
uniform float uSoftEdge; // e.g. 0.3

varying vec3 vVelocity;

float linearizeDepth(float depth) {
  float z = depth * 2.0 - 1.0;
  return (2.0 * uCameraNear * uCameraFar) /
         (uCameraFar + uCameraNear - z * (uCameraFar - uCameraNear));
}

void main() {
  // Circular billboard mask
  vec2 coord = gl_PointCoord - 0.5;
  float radius = length(coord);
  if (radius > 0.5) discard;

  float alpha = 1.0 - smoothstep(0.4, 0.5, radius);

  // Soft particles: fade near scene geometry
  vec2 screenUV = gl_FragCoord.xy / uResolution;
  float sceneDepth = linearizeDepth(texture2D(uDepthTexture, screenUV).r);
  float particleDepth = linearizeDepth(gl_FragCoord.z);
  float softFade = clamp((sceneDepth - particleDepth) / uSoftEdge, 0.0, 1.0);
  alpha *= softFade;

  // Color by velocity
  float speed = length(vVelocity);
  vec3 slowColor = vec3(0.2, 0.4, 1.0);
  vec3 fastColor = vec3(1.0, 0.5, 0.1);
  vec3 col = mix(slowColor, fastColor, clamp(speed * 80.0, 0.0, 1.0));

  gl_FragColor = vec4(col, alpha);
}
```

## Material Setup

```javascript
const mat = new THREE.ShaderMaterial({
  uniforms: {
    uTexturePosition: { value: null },
    uTextureVelocity: { value: null },
    uSize: { value: 3.0 }
  },
  vertexShader: RENDER_VERT,
  fragmentShader: RENDER_FRAG,
  transparent: true,
  depthWrite: false,
  blending: THREE.AdditiveBlending
});
```

## THREE.Points vs THREE.InstancedMesh

| Criterion | THREE.Points | THREE.InstancedMesh |
|---|---|---|
| Draw calls | 1 | 1 |
| Billboard | Automatic (gl_PointCoord) | Manual |
| Max at 60fps (desktop) | ~500k | ~100k |
| Max at 60fps (mobile) | ~100k | ~20k |
| Per-particle rotation | Not possible | Full 4x4 matrix |
| GPGPU compatible | Yes | Yes (harder) |
| Use when | Smoke, sparks, stars | 3D debris, foam |

**Crossover point:** GPGPU beats CPU-updated at ~20,000 particles.

## Depth Pre-pass for Soft Particles

```javascript
const depthRT = new THREE.WebGLRenderTarget(width, height);
depthRT.depthTexture = new THREE.DepthTexture(width, height);
depthRT.depthTexture.format = THREE.DepthFormat;
depthRT.depthTexture.type = THREE.UnsignedShortType;

// Each frame: render scene (no particles) to depthRT first
renderer.setRenderTarget(depthRT);
renderer.render(sceneNoParticles, camera);
renderer.setRenderTarget(null);

particleMaterial.uniforms.uDepthTexture.value = depthRT.depthTexture;
```

## Performance Benchmarks

### Desktop (RTX 3070 class)
| Technique | Max particles at 60fps |
|---|---|
| CPU-updated Points | ~50,000 |
| Points + GPGPU | 500,000–1,000,000 |
| InstancedMesh + CPU | ~100,000 |
| WebGPU compute | 1,000,000+ |

### Mobile (Snapdragon 8 Gen 2 / Apple A16)
| Technique | Max particles at 60fps |
|---|---|
| CPU-updated Points | ~10,000 |
| GPGPU (HalfFloatType) | 65,536 (256x256) |
| GPGPU (FloatType) | 16,384 (128x128) |
| WebGPU compute | 100,000-200,000 |

## GitHub References

- GPUComputationRenderer: https://github.com/mrdoob/three.js/blob/dev/examples/jsm/misc/GPUComputationRenderer.js
- Three.js GPGPU birds (flocking): https://github.com/mrdoob/three.js/blob/master/examples/webgl_gpgpu_birds.html
- PhysicsRenderer: https://github.com/cabbibo/PhysicsRenderer
- Codrops GPGPU particles: https://tympanus.net/codrops/2024/12/19/crafting-a-dreamy-particle-effect-with-three-js-and-gpgpu/
