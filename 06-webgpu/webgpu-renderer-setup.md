# WebGPU Renderer Setup

## Browser Support Matrix

| Browser | Version | Status |
|---------|---------|--------|
| Chrome / Edge | 113+ | Stable |
| Safari | 26+ (iOS 26+) | Stable |
| Firefox | 141+ | Stable (behind flag until 141) |
| Chrome Android | 121+ | Stable |

**Verified:** caniuse.com/webgpu â€” accessed 2026-05-26

## Three.js WebGPURenderer

```js
import WebGPURenderer from 'three/addons/renderers/webgpu/WebGPURenderer.js';
import WebGPU from 'three/addons/capabilities/WebGPU.js';

// 1. Capability check â€” ALWAYS guard before init
if (!WebGPU.isAvailable()) {
  // Fallback to WebGLRenderer
  document.body.appendChild(WebGPU.getErrorMessage());
  throw new Error('WebGPU not supported');
}

const renderer = new WebGPURenderer({ antialias: true });

// 2. await init() is MANDATORY â€” WebGPU is async
await renderer.init();

renderer.setSize(window.innerWidth, window.innerHeight);
renderer.setPixelRatio(Math.min(devicePixelRatio, 2));
document.body.appendChild(renderer.domElement);
```

### Key Differences from WebGLRenderer

| Aspect | WebGLRenderer | WebGPURenderer |
|--------|--------------|----------------|
| Init | Synchronous | `await renderer.init()` required |
| Shaders | GLSL | TSL (compiles to WGSL/GLSL) |
| Compute | Not native | `renderer.computeAsync(computeNode)` |
| Pipelines | Implicit | Explicit via TSL nodes |
| Multi-draw | `BatchedMesh` | Native indirect draws |

## Device Capabilities

```js
const info = renderer.info;
// renderer.capabilities object exposes:
// - maxTextureSize
// - maxUniformBufferBindingSize
// - timestampQuery (bool)
// - floatFilterable (bool)
// - dualSourceBlending (bool)

// Check specific feature at runtime
const adapter = await navigator.gpu.requestAdapter();
const canTimestamp = adapter.features.has('timestamp-query');
```

## Fallback Pattern (Recommended)

```js
async function createRenderer() {
  if (WebGPU.isAvailable()) {
    const r = new WebGPURenderer({ antialias: true });
    await r.init();
    return r;
  }
  // WebGL2 fallback
  return new THREE.WebGLRenderer({ antialias: true });
}
```

## Sources
- three.js examples: https://threejs.org/examples/?q=webgpu
- Three.js WebGPURenderer source: src/renderers/webgpu/WebGPURenderer.js (r184)
- WebGPU spec: https://www.w3.org/TR/webgpu/
