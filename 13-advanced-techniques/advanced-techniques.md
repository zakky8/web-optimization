# Advanced Techniques

This section covers techniques that go beyond standard Three.js usage: GPU-based particle simulation via GPGPU, deferred rendering pipelines, and general Framebuffer Object (FBO) patterns. These are the tools for scenes with hundreds of thousands of animated particles, complex lighting with many dynamic lights, and off-screen render targets.

## GPGPU Particles

General-Purpose GPU Computing on particle systems uses the GPU's fragment shader stage to simulate particle physics — position and velocity are stored in floating-point textures and updated each frame without touching the CPU.

### The Ping-Pong Pattern

GPGPU uses two render targets alternated each frame:

```
Frame 0: read from TextureA, write to TextureB
Frame 1: read from TextureB, write to TextureA
Frame 2: read from TextureA, write to TextureB
...
```

This avoids reading and writing the same texture simultaneously (undefined behavior in GLSL).

### GPUComputationRenderer (Three.js Built-in)

```js
import { GPUComputationRenderer } from "three/addons/misc/GPUComputationRenderer.js";

const PARTICLE_COUNT = 512; // sqrt gives texture dimension: 512x512 = 262,144 particles
const WIDTH = PARTICLE_COUNT;

const gpuCompute = new GPUComputationRenderer(WIDTH, WIDTH, renderer);

// Initialize position texture (RGBA = x, y, z, age)
const dtPosition = gpuCompute.createTexture();
const dtVelocity = gpuCompute.createTexture();

// Scatter initial positions
const posArr = dtPosition.image.data;
const velArr = dtVelocity.image.data;
for (let i = 0; i < posArr.length; i += 4) {
  posArr[i    ] = (Math.random() - 0.5) * 10; // x
  posArr[i + 1] = (Math.random() - 0.5) * 10; // y
  posArr[i + 2] = (Math.random() - 0.5) * 10; // z
  posArr[i + 3] = Math.random();               // age/life

  velArr[i    ] = (Math.random() - 0.5) * 0.1; // vx
  velArr[i + 1] = Math.random() * 0.05;         // vy (upward drift)
  velArr[i + 2] = (Math.random() - 0.5) * 0.1; // vz
  velArr[i + 3] = Math.random();                // mass
}

// Position simulation shader
const positionShader = `
  uniform sampler2D textureVelocity;
  uniform float uTime;
  uniform float uDelta;

  void main() {
    vec2 uv = gl_FragCoord.xy / resolution.xy;

    vec4 pos = texture2D(texturePosition, uv);  // texturePosition is auto-injected
    vec4 vel = texture2D(textureVelocity, uv);

    // Euler integration
    pos.xyz += vel.xyz * uDelta;

    // Bounce off floor
    if (pos.y < 0.0) {
      pos.y = 0.0;
    }

    // Age particle — reset when old
    pos.w -= uDelta * 0.3;
    if (pos.w <= 0.0) {
      pos.xyz = vec3(
        (rand(uv + uTime) - 0.5) * 10.0,
        5.0,
        (rand(uv + uTime * 1.3) - 0.5) * 10.0
      );
      pos.w = 1.0;
    }

    gl_FragColor = pos;
  }
`;

const velocityShader = `
  uniform float uDelta;

  void main() {
    vec2 uv = gl_FragCoord.xy / resolution.xy;
    vec4 vel = texture2D(textureVelocity, uv);

    // Gravity
    vel.y -= 9.81 * uDelta * 0.01;

    // Damping
    vel.xyz *= 0.999;

    gl_FragColor = vel;
  }
`;

// Register variables
const posVar = gpuCompute.addVariable("texturePosition", positionShader, dtPosition);
const velVar = gpuCompute.addVariable("textureVelocity", velocityShader, dtVelocity);

// Set dependencies
gpuCompute.setVariableDependencies(posVar, [posVar, velVar]);
gpuCompute.setVariableDependencies(velVar, [velVar]);

// Pass extra uniforms
posVar.material.uniforms.uTime  = { value: 0 };
posVar.material.uniforms.uDelta = { value: 0 };
velVar.material.uniforms.uDelta = { value: 0 };

const error = gpuCompute.init();
if (error !== null) console.error(error);
```

### Rendering GPGPU Particles

```js
// Create point cloud — one vertex per particle
const positions = new Float32Array(WIDTH * WIDTH * 3);
const uvCoords  = new Float32Array(WIDTH * WIDTH * 2);

for (let i = 0; i < WIDTH * WIDTH; i++) {
  const x = i % WIDTH;
  const y = Math.floor(i / WIDTH);
  uvCoords[i * 2    ] = x / (WIDTH - 1);
  uvCoords[i * 2 + 1] = y / (WIDTH - 1);
}

const geometry = new THREE.BufferGeometry();
geometry.setAttribute("position", new THREE.BufferAttribute(positions, 3));
geometry.setAttribute("uv",       new THREE.BufferAttribute(uvCoords, 2));

const particleMaterial = new THREE.ShaderMaterial({
  uniforms: {
    texturePosition: { value: null },
    uPointSize:      { value: 3.0 },
  },
  vertexShader: `
    uniform sampler2D texturePosition;
    uniform float uPointSize;
    varying float vLife;

    void main() {
      // Read world position from GPGPU texture
      vec4 pos = texture2D(texturePosition, uv);
      vLife = pos.w;
      gl_Position = projectionMatrix * modelViewMatrix * vec4(pos.xyz, 1.0);
      gl_PointSize = uPointSize * vLife;
    }
  `,
  fragmentShader: `
    varying float vLife;
    void main() {
      // Circular point
      vec2 c = gl_PointCoord - vec2(0.5);
      if (length(c) > 0.5) discard;
      gl_FragColor = vec4(vec3(1.0), vLife);
    }
  `,
  transparent: true,
  depthWrite: false,
  blending: THREE.AdditiveBlending,
});

const particles = new THREE.Points(geometry, particleMaterial);
scene.add(particles);

// Simulation loop
const clock = new THREE.Clock();
renderer.setAnimationLoop(() => {
  const delta = clock.getDelta();
  const elapsed = clock.getElapsedTime();

  posVar.material.uniforms.uTime.value  = elapsed;
  posVar.material.uniforms.uDelta.value = delta;
  velVar.material.uniforms.uDelta.value = delta;

  gpuCompute.compute();

  particleMaterial.uniforms.texturePosition.value =
    gpuCompute.getCurrentRenderTarget(posVar).texture;

  renderer.render(scene, camera);
});
```

### WebGPU Compute Shader Particles (Modern Approach)

With WebGPURenderer + TSL, GPGPU becomes native compute:

```js
import { WebGPURenderer, StorageBufferAttribute } from "three/webgpu";
import { Fn, storage, instanceIndex, float, vec3, uniform } from "three/tsl";

const COUNT = 262144; // 512 * 512

// GPU-side storage buffers (no CPU-GPU copy needed)
const positionBuffer = new StorageBufferAttribute(COUNT, 4); // xyzw
const velocityBuffer = new StorageBufferAttribute(COUNT, 4);

// Compute node: runs COUNT times in parallel
const updateParticles = Fn(() => {
  const i   = instanceIndex;
  const pos = storage(positionBuffer, "vec4", COUNT).element(i);
  const vel = storage(velocityBuffer, "vec4", COUNT).element(i);

  vel.y = vel.y.sub(float(9.81).mul(0.01)); // gravity
  pos.xyz = pos.xyz.add(vel.xyz);            // euler integration
})().compute(COUNT);

renderer.setAnimationLoop(() => {
  renderer.compute(updateParticles);
  renderer.render(scene, camera);
});
```

## Framebuffer Objects (FBO)

An FBO is a render target: instead of drawing to the screen canvas, you draw to a texture. That texture can then be used as input to another draw call. This is the foundation of post-processing, reflections, shadow maps, and deferred rendering.

### WebGLRenderTarget

```js
// Create render target
const rt = new THREE.WebGLRenderTarget(
  window.innerWidth,
  window.innerHeight,
  {
    minFilter: THREE.LinearFilter,
    magFilter: THREE.LinearFilter,
    format:    THREE.RGBAFormat,
    type:      THREE.HalfFloatType, // HDR precision
    samples:   4, // MSAA (WebGL 2 only)
  }
);

// Render scene into render target
renderer.setRenderTarget(rt);
renderer.render(scene, camera);
renderer.setRenderTarget(null); // back to screen

// Use as texture
screenQuad.material.uniforms.uScene.value = rt.texture;
```

### Full-Screen Post-Processing Pass

```js
// Minimal post-processing without a library
const quad = new THREE.Mesh(
  new THREE.PlaneGeometry(2, 2),
  new THREE.ShaderMaterial({
    uniforms: {
      uTexture:    { value: rt.texture },
      uResolution: { value: new THREE.Vector2(innerWidth, innerHeight) },
    },
    vertexShader: `
      void main() {
        gl_Position = vec4(position.xy, 0.0, 1.0); // fullscreen clip-space quad
      }
    `,
    fragmentShader: `
      uniform sampler2D uTexture;
      uniform vec2 uResolution;

      // Example: vignette
      void main() {
        vec2 uv = gl_FragCoord.xy / uResolution;
        vec4 color = texture2D(uTexture, uv);
        float vig = smoothstep(0.8, 0.3, length(uv - 0.5));
        gl_FragColor = vec4(color.rgb * vig, 1.0);
      }
    `,
    depthTest:  false,
    depthWrite: false,
  })
);

// The quad renders over everything — camera is unused
const orthoCamera = new THREE.OrthographicCamera(-1, 1, 1, -1, 0, 1);
```

### Multiple Render Targets (MRT) — G-Buffer

MRT writes to multiple textures in a single pass. Foundation of deferred rendering.

```js
// Requires WebGL 2 (default in Three.js r155+)
const gBuffer = new THREE.WebGLMultipleRenderTargets(
  innerWidth, innerHeight, 3, // 3 color attachments
  {
    minFilter: THREE.NearestFilter,
    magFilter: THREE.NearestFilter,
    type:      THREE.HalfFloatType,
  }
);
// gBuffer.texture[0] = albedo
// gBuffer.texture[1] = normal (world space)
// gBuffer.texture[2] = position (world space) OR reconstruct from depth

// Geometry pass shader — writes to 3 outputs simultaneously
const gPassMat = new THREE.ShaderMaterial({
  glslVersion: THREE.GLSL3,
  fragmentShader: `
    layout(location = 0) out vec4 gAlbedo;
    layout(location = 1) out vec4 gNormal;
    layout(location = 2) out vec4 gPosition;

    in vec3 vWorldNormal;
    in vec3 vWorldPos;
    uniform sampler2D uAlbedoMap;
    in vec2 vUv;

    void main() {
      gAlbedo   = texture(uAlbedoMap, vUv);
      gNormal   = vec4(normalize(vWorldNormal) * 0.5 + 0.5, 1.0);
      gPosition = vec4(vWorldPos, 1.0);
    }
  `,
  // ...
});
```

### Screen-Space Ambient Occlusion (SSAO) via FBO

```js
// 1. Render scene to G-Buffer (position, normal, depth)
// 2. Sample hemisphere around each fragment using G-Buffer
// 3. Count how many samples are occluded by geometry (below depth)
// 4. Blur the AO result
// 5. Multiply into final color

// Three.js has a built-in SSAO pass
import { SSAOPass } from "three/addons/postprocessing/SSAOPass.js";
import { EffectComposer } from "three/addons/postprocessing/EffectComposer.js";
import { RenderPass } from "three/addons/postprocessing/RenderPass.js";

const composer = new EffectComposer(renderer);
composer.addPass(new RenderPass(scene, camera));

const ssaoPass = new SSAOPass(scene, camera, innerWidth, innerHeight);
ssaoPass.kernelRadius = 16;
ssaoPass.minDistance  = 0.005;
ssaoPass.maxDistance  = 0.1;
composer.addPass(ssaoPass);

renderer.setAnimationLoop(() => composer.render());
```

## Performance Implications

**GPGPU**
- CPU cost for 262K particles is effectively zero — all simulation runs on GPU.
- The bottleneck is GPU memory bandwidth, not compute. Keep particle data as compact as possible (RGBA16F not RGBA32F unless needed).
- `GPUComputationRenderer` is WebGL 1/2. The WebGPU compute path is 2-5x faster for the same particle count due to better memory access patterns.

**FBOs**
- Each render target allocation consumes GPU VRAM. On mobile, total VRAM may be 1–2 GB shared with the OS. Budget carefully.
- Half-float (`THREE.HalfFloatType`) uses half the memory of full float. Use it unless you need > ~65,000 range.
- MSAA render targets (samples > 1) cannot be used as texture inputs directly. Resolve to a non-MSAA target first.
- `THREE.WebGLMultipleRenderTargets` requires WebGL 2. Check `renderer.capabilities.isWebGL2`.

**Post-processing chains**
- Each pass is a full-screen render. 5 passes × 1080p = 5 × 2M pixels × 4 bytes = 40 MB of texture bandwidth per frame.
- Combine passes where possible. A single custom pass doing tonemapping + vignette + color grading is faster than three separate passes.

## Tools and Libraries

| Tool | Purpose |
|---|---|
| [GPUComputationRenderer](https://threejs.org/docs/#examples/en/misc/GPUComputationRenderer) | Three.js GPGPU helper |
| [Three.js postprocessing](https://threejs.org/docs/#examples/en/postprocessing/EffectComposer) | Built-in composer |
| [postprocessing (npm)](https://github.com/pmndrs/postprocessing) | High-performance effects library (pmndrs) |
| [Three.js Journey GPGPU lesson](https://threejs-journey.com/lessons/gpgpu-flow-field-particles-shaders) | Detailed GPGPU tutorial |
| [Codrops GPGPU tutorial](https://tympanus.net/codrops/2024/12/19/crafting-a-dreamy-particle-effect-with-three-js-and-gpgpu/) | Visual GPGPU particles walkthrough |
| [WebGPU compute shaders MDN](https://developer.mozilla.org/en-US/docs/Web/API/WebGPU_API) | Native compute reference |
