# WASM Load Failure Handling

## Failure Modes

A WASM module can fail to load for distinct reasons, each requiring a different response:

| Failure type | Root cause | How it surfaces |
|---|---|---|
| Network error | `.wasm` file not found, CDN timeout, CORS misconfiguration | `fetch()` rejects with `TypeError` or `NetworkError` |
| MIME type mismatch | Server sends wrong `Content-Type` (not `application/wasm`) | `WebAssembly.instantiateStreaming()` rejects with `TypeError` |
| Unsupported browser | No `WebAssembly` global (IE 11, very old Android WebView) | `typeof WebAssembly === 'undefined'` |
| Compilation error | Malformed binary, wrong target (WASM64 on WASM32-only browser) | `WebAssembly.CompileError` |
| Link error | Import/export mismatch between JS and WASM module | `WebAssembly.LinkError` |
| Runtime error | Trap inside WASM (stack overflow, OOB memory access) | `WebAssembly.RuntimeError` |
| CSP blocked | Content Security Policy disallows `wasm-eval` or `unsafe-eval` | `CompileError` with CSP-related message |

Source: MDN `WebAssembly.instantiateStreaming()`, accessed 2026-05-26.

---

## Feature Detection

Check before attempting to load:

```js
function isWasmAvailable() {
  return (
    typeof WebAssembly === 'object' &&
    typeof WebAssembly.instantiateStreaming === 'function' &&
    typeof WebAssembly.validate === 'function'
  );
}
```

The `typeof WebAssembly === 'object'` guard is the canonical pattern (MDN WebAssembly namespace, accessed 2026-05-26). Checking for `instantiateStreaming` specifically gates on the streaming API; fall back to `instantiate` with `arrayBuffer()` if it is missing.

Baseline browser support:
- `WebAssembly` (compile + instantiate): widely available since October 2017
- `WebAssembly.instantiateStreaming`: widely available since September 2021

Source: MDN `WebAssembly.instantiateStreaming()`, accessed 2026-05-26.

---

## `WebAssembly.instantiateStreaming()` Error Types

The function signature:

```js
WebAssembly.instantiateStreaming(source, importObject?, compileOptions?)
// Returns: Promise<{ module: WebAssembly.Module, instance: WebAssembly.Instance }>
```

Thrown exceptions (MDN `instantiateStreaming`, accessed 2026-05-26):
- `TypeError` — wrong parameter types, or incorrect MIME type on the `.wasm` response
- `WebAssembly.CompileError` — binary failed to compile
- `WebAssembly.LinkError` — import/export contract not satisfied
- `WebAssembly.RuntimeError` — runtime trap during initialization

---

## try/catch Around WASM Init

```js
async function loadPhysicsEngine() {
  if (!isWasmAvailable()) {
    console.warn('WebAssembly unavailable — using JS fallback');
    return loadJsFallback();
  }

  try {
    const { instance } = await WebAssembly.instantiateStreaming(
      fetch('/wasm/rapier3d.wasm'),
      importObject
    );
    return buildRapierBindings(instance);

  } catch (err) {
    if (err instanceof TypeError) {
      // Likely MIME type issue or network failure
      console.error('WASM fetch/type error:', err.message);
    } else if (err instanceof WebAssembly.CompileError) {
      console.error('WASM compile error — binary may be corrupt or wrong target:', err.message);
    } else if (err instanceof WebAssembly.LinkError) {
      console.error('WASM link error — import mismatch:', err.message);
    } else if (err instanceof WebAssembly.RuntimeError) {
      console.error('WASM runtime trap during init:', err.message);
    } else {
      console.error('Unknown WASM load error:', err);
    }

    console.warn('Falling back to JS physics engine');
    return loadJsFallback();
  }
}
```

### MIME type fallback

`instantiateStreaming` requires the server to send `Content-Type: application/wasm`. If it does not (e.g., a plain static server sending `application/octet-stream`), the browser rejects the call with `TypeError`. Fall back to the two-step `arrayBuffer` path:

```js
async function loadWasmRobust(url, importObject) {
  // Prefer streaming (single round-trip)
  if (typeof WebAssembly.instantiateStreaming === 'function') {
    try {
      return await WebAssembly.instantiateStreaming(fetch(url), importObject);
    } catch (err) {
      if (!(err instanceof TypeError)) throw err;
      // TypeError often means wrong MIME type — try arrayBuffer fallback
      console.warn('instantiateStreaming failed (likely MIME type), retrying with arrayBuffer');
    }
  }

  // Fallback: fetch → ArrayBuffer → instantiate
  const response = await fetch(url);
  const bytes    = await response.arrayBuffer();
  return WebAssembly.instantiate(bytes, importObject);
}
```

Source: MDN Loading and Running WebAssembly, accessed 2026-05-26.

---

## Fallback: cannon-es Instead of Rapier

Rapier (https://rapier.rs) is a Rust/WASM physics engine. cannon-es (https://github.com/pmndrs/cannon-es) is a pure-JS port of Cannon.js maintained by pmdrs. It has no WASM dependency and runs everywhere.

### Abstraction layer pattern

Define a common physics interface and swap implementations behind it:

```js
// physics-interface.js
export class PhysicsWorld {
  constructor(impl) {
    this._impl = impl; // either RapierWorld or CannonWorld
  }

  step(dt) { this._impl.step(dt); }

  createRigidBody(desc) { return this._impl.createRigidBody(desc); }

  // etc.
}

// physics-loader.js
import RAPIER from '@dimforge/rapier3d';
import * as CANNON from 'cannon-es';
import { PhysicsWorld } from './physics-interface.js';

export async function createPhysicsWorld(gravity = { x: 0, y: -9.81, z: 0 }) {
  if (!isWasmAvailable()) {
    return createCannonWorld(gravity);
  }

  try {
    await RAPIER.init(); // internally calls WebAssembly.instantiateStreaming
    const world = new RAPIER.World(gravity);
    return new PhysicsWorld(new RapierAdapter(world));
  } catch (err) {
    console.warn('Rapier WASM load failed, falling back to cannon-es:', err.message);
    return createCannonWorld(gravity);
  }
}

function createCannonWorld(gravity) {
  const world = new CANNON.World({ gravity: new CANNON.Vec3(gravity.x, gravity.y, gravity.z) });
  return new PhysicsWorld(new CannonAdapter(world));
}
```

### Trade-offs

| | Rapier (WASM) | cannon-es (JS) |
|---|---|---|
| Performance | High (Rust, SIMD) | Moderate |
| Bundle size | ~1.2 MB WASM + ~80 KB JS bindings | ~180 KB |
| Browser support | WebAssembly required | Universal |
| Maintenance | Active (dimforge) | Active (pmndrs) |
| Features | Full rigid body, joints, character controller | Rigid body, basic constraints |

---

## Loading Sequence with User Feedback

```js
async function init() {
  const loadingEl = document.getElementById('physics-loading');

  loadingEl.textContent = 'Loading physics engine...';

  let physics;
  try {
    physics = await createPhysicsWorld({ x: 0, y: -9.81, z: 0 });
    loadingEl.hidden = true;
  } catch (err) {
    loadingEl.textContent = 'Physics engine failed to load. Reloading may help.';
    console.error('Fatal physics load failure:', err);
    return;
  }

  startScene(physics);
}
```

---

## Testing WASM Failure Paths

```js
// In development: mock a WASM load failure
if (process.env.NODE_ENV === 'development' && location.search.includes('no-wasm')) {
  window.WebAssembly = undefined; // forces isWasmAvailable() === false
}
```

Disable WebAssembly in Chrome DevTools: Settings > Experiments > "WebAssembly" toggle. This lets you verify the cannon-es fallback path without code changes.

---

## Sources

| Source | URL | Tier | Accessed |
|---|---|---|---|
| MDN — `WebAssembly.instantiateStreaming()` | https://developer.mozilla.org/en-US/docs/WebAssembly/JavaScript_interface/instantiateStreaming_static | 1 | 2026-05-26 |
| MDN — WebAssembly namespace (feature detection) | https://developer.mozilla.org/en-US/docs/WebAssembly/JavaScript_interface | 1 | 2026-05-26 |
| MDN — Loading and running WebAssembly | https://developer.mozilla.org/en-US/docs/WebAssembly/Guides/Loading_and_running | 1 | 2026-05-26 |
