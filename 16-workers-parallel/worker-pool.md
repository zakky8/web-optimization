# Worker Pool Pattern

## Basic Pool with Task Queue

```js
// worker-pool.js
export class WorkerPool {
  constructor(workerUrl, size = navigator.hardwareConcurrency ?? 4) {
    this.workers  = Array.from({ length: size }, () => ({
      worker: new Worker(workerUrl, { type: 'module' }),
      busy:   false,
    }));
    this.queue    = [];
    this.size     = size;

    this.workers.forEach((entry, i) => {
      entry.worker.onmessage = ({ data }) => {
        entry.busy = false;
        // Resolve pending promise
        if (entry._resolve) {
          entry._resolve(data);
          entry._resolve = null;
        }
        // Pick next task from queue
        this._drain();
      };
    });
  }

  // Returns a promise that resolves with worker response
  exec(payload) {
    return new Promise((resolve, reject) => {
      this.queue.push({ payload, resolve, reject });
      this._drain();
    });
  }

  _drain() {
    const free = this.workers.find(w => !w.busy);
    if (!free || this.queue.length === 0) return;

    const { payload, resolve, reject } = this.queue.shift();
    free.busy      = true;
    free._resolve  = resolve;
    free._reject   = reject;
    free.worker.postMessage(payload);
  }

  terminate() {
    this.workers.forEach(w => w.worker.terminate());
  }
}
```

```js
// Usage
const pool = new WorkerPool('./compress.worker.js', 4);

// Process 100 tasks across 4 workers
const results = await Promise.all(
  images.map(img => pool.exec({ imageData: img }))
);
```

## Comlink â€” RPC-Style Workers

```bash
npm install comlink
```

```js
// math.worker.js
import { expose } from 'comlink';

const api = {
  async computeNoise(x, y, octaves) {
    // CPU-intensive noise computation
    return fbm(x, y, octaves);
  },
  async processMesh(vertices) {
    // Heavy mesh processing
    return computeNormals(vertices);
  }
};

expose(api);
```

```js
// main.js â€” call worker like a normal async function
import { wrap } from 'comlink';

const worker = new Worker('./math.worker.js', { type: 'module' });
const api    = wrap(worker);

// Looks synchronous, runs in worker
const noiseValue = await api.computeNoise(1.5, 2.3, 8);
const normals    = await api.processMesh(vertexArray);
```

## Transferable Objects (Zero-Copy)

```js
// Without transfer â€” SLOW (copies buffer)
worker.postMessage({ data: largeBuffer });

// With transfer â€” FAST (moves ownership, zero copy)
worker.postMessage({ data: largeBuffer }, [largeBuffer]);
// largeBuffer is now detached in main thread

// Common transferables:
// - ArrayBuffer (and typed array .buffer)
// - MessagePort
// - ImageBitmap
// - OffscreenCanvas
// - ReadableStream, WritableStream, TransformStream (Chrome)
```

## Parallel Texture Processing Example

```js
const pool = new WorkerPool('./image.worker.js');

// Process texture atlas in parallel strips
const STRIP_HEIGHT = 64;
const strips = [];
for (let y = 0; y < textureHeight; y += STRIP_HEIGHT) {
  const slice = imageData.data.slice(
    y * textureWidth * 4,
    (y + STRIP_HEIGHT) * textureWidth * 4
  );
  strips.push(pool.exec({ slice, y, width: textureWidth }));
}

const processed = await Promise.all(strips);
```

## Sources
- MDN Web Workers: https://developer.mozilla.org/en-US/docs/Web/API/Web_Workers_API
- Comlink: https://github.com/GoogleChromeLabs/comlink
- MDN Transferable: https://developer.mozilla.org/en-US/docs/Web/API/Web_Workers_API/Transferable_objects
