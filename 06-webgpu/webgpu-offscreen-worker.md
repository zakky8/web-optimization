# WebGPU with OffscreenCanvas + Workers

## Setup

```js
// main.js
const canvas = document.getElementById('canvas');

// Transfer canvas control to worker
const offscreen = canvas.transferControlToOffscreen();

const worker = new Worker('./render.worker.js', { type: 'module' });
worker.postMessage({ type: 'init', canvas: offscreen }, [offscreen]);

// Handle resize
window.addEventListener('resize', () => {
  worker.postMessage({
    type: 'resize',
    width: window.innerWidth,
    height: window.innerHeight,
    dpr: Math.min(devicePixelRatio, 2)
  });
});
```

```js
// render.worker.js
import WebGPURenderer from 'three/addons/renderers/webgpu/WebGPURenderer.js';
import * as THREE from 'three';

let renderer, scene, camera;

self.addEventListener('message', async ({ data }) => {
  if (data.type === 'init') {
    renderer = new WebGPURenderer({
      canvas: data.canvas,
      antialias: true
    });

    // REQUIRED: async init
    await renderer.init();

    scene  = new THREE.Scene();
    camera = new THREE.PerspectiveCamera(60, 1, 0.1, 1000);
    camera.position.z = 5;

    // Build scene...
    buildScene(scene);

    animate();
  }

  if (data.type === 'resize') {
    renderer.setSize(data.width, data.height, false);
    renderer.setPixelRatio(data.dpr);
    camera.aspect = data.width / data.height;
    camera.updateProjectionMatrix();
  }
});

function animate() {
  self.requestAnimationFrame(animate);
  renderer.render(scene, camera);
}
```

## Required HTTP Headers

OffscreenCanvas + SharedArrayBuffer requires:

```
Cross-Origin-Opener-Policy: same-origin
Cross-Origin-Embedder-Policy: require-corp
```

Without these, `transferControlToOffscreen()` still works but `SharedArrayBuffer` will be undefined.

## Vite Dev Server Config

```js
// vite.config.js
export default {
  server: {
    headers: {
      'Cross-Origin-Opener-Policy': 'same-origin',
      'Cross-Origin-Embedder-Policy': 'require-corp',
    }
  },
  plugins: [
    // Ensure worker bundling
  ]
}
```

## Performance Notes

- WebGPU in worker: main thread JS fully decoupled from render work
- GPU-CPU sync (readback) still crosses thread boundary via postMessage
- Use `SharedArrayBuffer` for high-frequency data (input state, physics) without postMessage overhead
- Comlink simplifies worker RPC if needed: `npm install comlink`

## Sources
- MDN OffscreenCanvas: https://developer.mozilla.org/en-US/docs/Web/API/OffscreenCanvas
- Three.js offscreen example: https://threejs.org/examples/#webgpu_compute_particles
- COOP/COEP explainer: https://web.dev/articles/coop-coep
