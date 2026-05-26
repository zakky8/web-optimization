# OffscreenCanvas â€” Three.js in a Web Worker

## Why Use OffscreenCanvas

- Main thread fully decoupled from GPU rendering
- Long-running JS (physics, AI, data processing) won't cause frame drops
- Enables true 60fps even during heavy computation
- Required for: large particle systems, real-time physics, ML inference

## Basic Setup

```js
// main.js
const canvas = document.getElementById('gl-canvas');
const offscreen = canvas.transferControlToOffscreen();

const worker = new Worker('./renderer.worker.js', { type: 'module' });

// Transfer ownership of canvas to worker (one-way, irreversible)
worker.postMessage({ type: 'init', canvas: offscreen }, [offscreen]);

// Input forwarding (pointer events, keyboard, resize)
window.addEventListener('pointermove', (e) => {
  worker.postMessage({
    type: 'pointermove',
    x: e.clientX, y: e.clientY
  });
});

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
// renderer.worker.js
import * as THREE from 'three';

let renderer, scene, camera, clock;

self.onmessage = ({ data }) => {
  switch (data.type) {
    case 'init':
      initRenderer(data.canvas);
      break;
    case 'resize':
      handleResize(data.width, data.height, data.dpr);
      break;
    case 'pointermove':
      handlePointer(data.x, data.y);
      break;
  }
};

function initRenderer(canvas) {
  renderer = new THREE.WebGLRenderer({ canvas, antialias: true });
  renderer.setPixelRatio(2);

  scene  = new THREE.Scene();
  camera = new THREE.PerspectiveCamera(60, canvas.width / canvas.height, 0.1, 100);
  clock  = new THREE.Clock();

  // Build scene...
  scene.add(new THREE.AmbientLight(0xffffff, 0.5));
  const mesh = new THREE.Mesh(
    new THREE.BoxGeometry(),
    new THREE.MeshStandardMaterial({ color: 0x0080ff })
  );
  scene.add(mesh);
  camera.position.z = 3;

  tick();
}

function tick() {
  self.requestAnimationFrame(tick);
  const dt = clock.getDelta();
  renderer.render(scene, camera);
}

function handleResize(w, h, dpr) {
  renderer.setSize(w, h, false);
  renderer.setPixelRatio(dpr);
  camera.aspect = w / h;
  camera.updateProjectionMatrix();
}
```

## Required HTTP Headers

```
Cross-Origin-Opener-Policy: same-origin
Cross-Origin-Embedder-Policy: require-corp
```

Without COEP, OffscreenCanvas works but SharedArrayBuffer is unavailable.

## Browser Support

| Browser | OffscreenCanvas | Notes |
|---------|----------------|-------|
| Chrome 69+ | Full | Stable |
| Firefox 105+ | Full | Stable |
| Safari 16.4+ | Full | Added 2023 |
| iOS Safari 16.4+ | Full | Same as desktop |

## Sources
- MDN OffscreenCanvas: https://developer.mozilla.org/en-US/docs/Web/API/OffscreenCanvas
- Three.js offscreen example: https://threejs.org/examples/#webgl_worker_offscreencanvas
- COOP/COEP: https://web.dev/articles/coop-coep
