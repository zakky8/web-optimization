# GPUComputationRenderer — Complete API Reference

Source: https://github.com/mrdoob/three.js/blob/dev/examples/jsm/misc/GPUComputationRenderer.js

## Constructor
```javascript
new GPUComputationRenderer(sizeX, sizeY, renderer)
```
`sizeX * sizeY` = total particle count. Each texel holds one particle's RGBA floats.
- 128×128 = 16,384 particles
- 256×256 = 65,536 particles
- 512×512 = 262,144 particles

Creates a `FullScreenQuad` for rendering, a pass-through shader for texture copies.

## Method Reference

### `setDataType(type)`
```javascript
gpuCompute.setDataType(THREE.HalfFloatType); // or THREE.FloatType
```
- `FloatType` — 32-bit, wider hardware support
- `HalfFloatType` — 16-bit, halves VRAM, good for positions if normalized
- Must be called before `init()`

### `addVariable(name, fragmentShader, initialTexture)`
Returns a variable object:
```javascript
{
  name: variableName,          // uniform name in shaders
  material: ShaderMaterial,    // wraps your fragment shader
  dependencies: null,          // set via setVariableDependencies()
  renderTargets: [],           // two per variable (ping-pong)
  wrapS, wrapT,                // default ClampToEdgeWrapping
  minFilter, magFilter         // default NearestFilter (CRITICAL - no interpolation)
}
```

### `setVariableDependencies(variable, dependencies)`
Declares which textures a variable reads. Auto-injects `uniform sampler2D <name>` into shader.
```javascript
gpuCompute.setVariableDependencies(positionVariable, [positionVariable, velocityVariable]);
gpuCompute.setVariableDependencies(velocityVariable, [positionVariable, velocityVariable]);
```

### `init()`
- Checks `renderer.capabilities.maxVertexTextures === 0` (fails on old GPUs)
- Allocates two `WebGLRenderTarget` per variable
- Returns `null` on success, error string on failure — always check:
```javascript
const error = gpuCompute.init();
if (error !== null) console.error('GPUComputationRenderer init error:', error);
```

### `compute()`
Called once per animation frame. Flips ping-pong buffers.
```
Frame N:   reads from RT[0], writes to RT[1]
Frame N+1: reads from RT[1], writes to RT[0]
```

### `getCurrentRenderTarget(variable)`
Returns render target written during most recent `compute()`:
```javascript
particleMaterial.uniforms.uPositions.value =
  gpuCompute.getCurrentRenderTarget(positionVariable).texture;
```

### `createTexture()`
Allocates a `DataTexture` backed by `Float32Array(sizeX * sizeY * 4)`.
Use this to seed initial particle state.

### `dispose()`
Frees all render targets, textures, materials. Essential for SPA cleanup.

## Complete Setup Pattern

```javascript
import * as THREE from 'three';
import { GPUComputationRenderer } from 'three/examples/jsm/misc/GPUComputationRenderer.js';

const WIDTH = 512; // 262,144 particles

const gpuCompute = new GPUComputationRenderer(WIDTH, WIDTH, renderer);

// Use HalfFloat on mobile to save bandwidth
if (renderer.capabilities.isWebGL2) {
  gpuCompute.setDataType(THREE.HalfFloatType);
}

// Seed initial textures
const dtPosition = gpuCompute.createTexture();
const dtVelocity = gpuCompute.createTexture();

const posArr = dtPosition.image.data;
const velArr = dtVelocity.image.data;
for (let i = 0; i < posArr.length; i += 4) {
  posArr[i]   = (Math.random() - 0.5) * 10.0;
  posArr[i+1] = (Math.random() - 0.5) * 10.0;
  posArr[i+2] = (Math.random() - 0.5) * 10.0;
  posArr[i+3] = Math.random(); // age seed
  velArr[i]   = (Math.random() - 0.5) * 0.02;
  velArr[i+1] = (Math.random() - 0.5) * 0.02;
  velArr[i+2] = (Math.random() - 0.5) * 0.02;
  velArr[i+3] = 1.0;
}

const positionVariable = gpuCompute.addVariable('texturePosition', positionShader, dtPosition);
const velocityVariable = gpuCompute.addVariable('textureVelocity', velocityShader, dtVelocity);

gpuCompute.setVariableDependencies(positionVariable, [positionVariable, velocityVariable]);
gpuCompute.setVariableDependencies(velocityVariable, [positionVariable, velocityVariable]);

velocityVariable.material.uniforms.uTime  = { value: 0.0 };
velocityVariable.material.uniforms.uDelta = { value: 0.0 };

const error = gpuCompute.init();
if (error !== null) console.error(error);
```
