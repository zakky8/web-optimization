# SharedArrayBuffer + Atomics

Zero-copy shared memory between main thread and workers.
**Use case:** Physics sync, audio processing, particle data â€” anything that needs >60 updates/sec.

## Setup (Requires COOP + COEP Headers)

```
Cross-Origin-Opener-Policy: same-origin
Cross-Origin-Embedder-Policy: require-corp
```

## Basic Shared Memory

```js
// main.js
const sharedBuffer = new SharedArrayBuffer(
  4 * Float32Array.BYTES_PER_ELEMENT  // 4 floats
);
const sharedView = new Float32Array(sharedBuffer);

const worker = new Worker('./worker.js', { type: 'module' });
// Pass buffer reference â€” no copy, zero overhead
worker.postMessage({ buffer: sharedBuffer });

// Read physics result every frame (no postMessage needed)
renderer.setAnimationLoop(() => {
  mesh.position.x = sharedView[0];
  mesh.position.y = sharedView[1];
  mesh.position.z = sharedView[2];
});
```

```js
// worker.js â€” physics simulation
let physicsData;

self.onmessage = ({ data }) => {
  physicsData = new Float32Array(data.buffer);
  runPhysics();
};

function runPhysics() {
  let t = 0;
  setInterval(() => {
    t += 0.016;
    // Write directly to shared memory â€” visible to main thread immediately
    physicsData[0] = Math.sin(t) * 2;
    physicsData[1] = Math.cos(t * 0.7);
    physicsData[2] = Math.sin(t * 1.3);
  }, 16);
}
```

## Atomics API

Thread-safe operations on shared integer arrays (Int32Array or BigInt64Array only).

```js
const sab  = new SharedArrayBuffer(256);
const i32  = new Int32Array(sab);

// Atomic operations â€” guaranteed single-instruction, no race conditions
Atomics.store(i32, 0, 42);           // Write
const val = Atomics.load(i32, 0);    // Read
Atomics.add(i32, 1, 1);             // Increment
Atomics.sub(i32, 1, 1);             // Decrement
Atomics.and(i32, 2, 0xFF);          // Bitwise AND
Atomics.or(i32, 2, 0x01);           // Bitwise OR
Atomics.xor(i32, 2, 0x01);          // Bitwise XOR

// Compare-and-swap (CAS) â€” returns old value
const old = Atomics.compareExchange(i32, 0, expected, newValue);

// Wait/notify â€” mutex-style synchronization
// Worker: sleep until i32[0] !== 0
const result = Atomics.wait(i32, 0, 0, 1000);  // timeout 1000ms
// result = 'ok' | 'not-equal' | 'timed-out'

// Main/other worker: wake sleeping workers
Atomics.notify(i32, 0, 1);  // wake 1 waiter
Atomics.notify(i32, 0, Infinity);  // wake all
```

## Physics Sync Pattern (Zero postMessage Per Frame)

```js
// Shared layout: [stateFlag, x0, y0, z0, x1, y1, z1, ...]
// stateFlag: 0 = busy (physics writing), 1 = ready (main thread can read)
const BODY_COUNT = 1000;
const SAB_SIZE   = (1 + BODY_COUNT * 3) * Float32Array.BYTES_PER_ELEMENT;
const sab        = new SharedArrayBuffer(SAB_SIZE);
const flag       = new Int32Array(sab, 0, 1);
const positions  = new Float32Array(sab, 4, BODY_COUNT * 3);

// Physics worker writes positions, flips flag
Atomics.store(flag, 0, 0);  // busy
updatePhysics(positions);   // write all body positions
Atomics.store(flag, 0, 1);  // ready

// Main thread reads only when flag = 1
if (Atomics.load(flag, 0) === 1) {
  for (let i = 0; i < BODY_COUNT; i++) {
    bodies[i].position.set(
      positions[i * 3],
      positions[i * 3 + 1],
      positions[i * 3 + 2]
    );
  }
}
```

## Sources
- MDN SharedArrayBuffer: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/SharedArrayBuffer
- MDN Atomics: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Atomics
- web.dev: https://web.dev/articles/coop-coep
