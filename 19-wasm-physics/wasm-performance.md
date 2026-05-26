# WASM Performance Patterns for Physics

## Loading WASM in Vite Projects

```js
// vite.config.js — REQUIRED for any WASM package
import { defineConfig } from 'vite';
import wasm from 'vite-plugin-wasm';
import topLevelAwait from 'vite-plugin-top-level-await';

export default defineConfig({
  plugins: [
    wasm(),
    topLevelAwait(),  // Required for async WASM initialization
  ],
  build: {
    target: 'esnext',  // Required for top-level await in output
  },
  optimizeDeps: {
    // Exclude WASM modules from Vite's pre-bundler (esbuild can't handle WASM)
    exclude: [
      '@dimforge/rapier3d',
      '@dimforge/rapier2d',
      'ammo.js',
    ],
  },
  worker: {
    format: 'es',
    plugins: () => [wasm(), topLevelAwait()],
  },
});
```

## Physics in a Worker (Recommended Architecture)

Run physics simulation in a dedicated worker, zero main-thread blocking:

```js
// physics.worker.js
import RAPIER from '@dimforge/rapier3d';
import { expose } from 'comlink';

class PhysicsEngine {
  async init(gravity = { x: 0, y: -9.81, z: 0 }) {
    await RAPIER.init();
    this.world = new RAPIER.World(gravity);
    this.bodies = new Map();
  }

  addRigidBody(id, desc) {
    const body = this.world.createRigidBody(desc);
    this.bodies.set(id, body);
    return id;
  }

  step() {
    this.world.step();
  }

  getPositions() {
    const out = [];
    for (const [id, body] of this.bodies) {
      const t = body.translation();
      const r = body.rotation();
      out.push(id, t.x, t.y, t.z, r.x, r.y, r.z, r.w);
    }
    return out;
  }
}

expose(new PhysicsEngine());
```

```js
// main.js
import { wrap } from 'comlink';

const worker  = new Worker('./physics.worker.js', { type: 'module' });
const physics = wrap(worker);

await physics.init({ x: 0, y: -9.81, z: 0 });

// Each frame
renderer.setAnimationLoop(async () => {
  await physics.step();
  const positions = await physics.getPositions();

  // Parse and apply to Three.js meshes
  for (let i = 0; i < positions.length; i += 8) {
    const id = positions[i];
    const mesh = meshMap.get(id);
    if (mesh) {
      mesh.position.set(positions[i+1], positions[i+2], positions[i+3]);
      mesh.quaternion.set(positions[i+4], positions[i+5], positions[i+6], positions[i+7]);
    }
  }

  renderer.render(scene, camera);
});
```

## Fixed vs Variable Timestep

```js
// Fixed timestep (recommended — deterministic)
const FIXED_DT = 1 / 60;  // 60Hz physics
const MAX_SUB  = 3;        // Max substeps per frame (prevents spiral of death)

let accumulator = 0;

function animate(timestamp) {
  const dt = Math.min((timestamp - lastTime) / 1000, 0.1);
  lastTime = timestamp;
  accumulator += dt;

  let steps = 0;
  while (accumulator >= FIXED_DT && steps < MAX_SUB) {
    world.step(FIXED_DT);
    accumulator -= FIXED_DT;
    steps++;
  }

  // Interpolate render state between physics steps
  const alpha = accumulator / FIXED_DT;
  interpolatePositions(alpha);

  renderer.render(scene, camera);
  requestAnimationFrame(animate);
}
```

## WASM Memory

```js
// WASM memory is a fixed ArrayBuffer — allocate once
// Rapier manages its own heap internally

// Check WASM memory usage
const wasmMemory = RAPIER.__wasm.memory;
const usedMB = wasmMemory.buffer.byteLength / 1024 / 1024;
console.log(`Rapier WASM memory: ${usedMB.toFixed(1)}MB`);

// Rapier default: 16MB initial, grows as needed
// For large scenes (1000+ rigid bodies), can reach 64-128MB
```

## Sources
- vite-plugin-wasm: https://github.com/Menci/vite-plugin-wasm
- Rapier performance: https://rapier.rs/docs/user_guides/javascript/getting_started_js
- MDN WebAssembly: https://developer.mozilla.org/en-US/docs/WebAssembly
