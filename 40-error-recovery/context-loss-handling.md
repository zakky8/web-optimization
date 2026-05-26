# WebGL Context Loss Handling

## What Context Loss Is

A WebGL context can be lost at any time. The browser fires `webglcontextlost` on the `<canvas>` element and all WebGL state—textures, buffers, shaders, framebuffers—is invalidated. The GPU process was reset or stolen by another client. Without handling this event, your render loop silently breaks and the canvas goes blank.

Common causes (from MDN, accessed 2026-05-26):
- Multiple pages or tabs competing for GPU resources; the browser tears down lower-priority contexts
- The system switches between GPUs (e.g., discrete to integrated on battery)
- A long-running shader or draw call triggers a GPU timeout reset
- The OS or driver performs a live graphics driver update

---

## The Two Events

Both events fire on the `HTMLCanvasElement`. Their interface is `WebGLContextEvent` (extends `Event`). Neither bubbles. Source: MDN `webglcontextlost` and `webglcontextrestored` event pages, accessed 2026-05-26.

| Event | Cancelable | Key action required |
|---|---|---|
| `webglcontextlost` | Yes — call `preventDefault()` to allow restore | Stop render loop; set lost flag |
| `webglcontextrestored` | No | Re-initialize all GPU resources |

### `event.statusMessage`

`WebGLContextEvent` exposes one additional read-only property: `statusMessage` (string). It may carry a human-readable reason from the browser/driver. Log it; it is empty more often than not.

---

## The Critical Role of `preventDefault()`

By default the browser **will not restore** a lost context. You must call `event.preventDefault()` inside the `webglcontextlost` handler to opt in to restoration. If you omit it, `webglcontextrestored` never fires.

```js
canvas.addEventListener('webglcontextlost', (event) => {
  event.preventDefault(); // Required — without this, context is never restored
  // ...
}, false);
```

Source: MDN `webglcontextlost` event, accessed 2026-05-26.

---

## `WebGLRenderingContext.isContextLost()`

Poll this inside the render loop as a guard. Returns `true` when the context is lost. Also useful when checking shader/program link errors — an apparent link failure during context loss produces unreliable error logs, so check `isContextLost()` first.

```js
if (!gl.getProgramParameter(program, gl.LINK_STATUS) && !gl.isContextLost()) {
  console.error(gl.getProgramInfoLog(program));
}
```

Source: MDN `isContextLost()`, accessed 2026-05-26.

---

## WEBGL_lose_context Extension (Testing)

The `WEBGL_lose_context` extension lets you trigger context loss and restoration programmatically. Baseline widely available since April 2018 (MDN, accessed 2026-05-26).

```js
const ext = gl.getExtension('WEBGL_lose_context');
// Simulate loss — fires webglcontextlost on the canvas
ext.loseContext();
// Simulate restore — fires webglcontextrestored on the canvas
ext.restoreContext();
```

Always null-guard: `ext` is `null` if the extension is unavailable.

---

## Three.js: Built-in Helpers

Three.js `WebGLRenderer` registers the three context lifecycle listeners internally during construction (source: `three.js/src/renderers/WebGLRenderer.js`, `dev` branch, accessed 2026-05-26):

```js
canvas.addEventListener('webglcontextlost', onContextLost, false);
canvas.addEventListener('webglcontextrestored', onContextRestore, false);
canvas.addEventListener('webglcontextcreationerror', onContextCreationError, false);
```

### Three.js internal `onContextLost`

Extracted verbatim from `WebGLRenderer.js` (`dev`, accessed 2026-05-26):

```js
function onContextLost(event) {
  event.preventDefault();
  log('WebGLRenderer: Context Lost.');
  _isContextLost = true;
}
```

### Three.js internal `onContextRestore`

```js
function onContextRestore(/* event */) {
  log('WebGLRenderer: Context Restored.');
  _isContextLost = false;

  const infoAutoReset        = info.autoReset;
  const shadowMapEnabled     = shadowMap.enabled;
  const shadowMapAutoUpdate  = shadowMap.autoUpdate;
  const shadowMapNeedsUpdate = shadowMap.needsUpdate;
  const shadowMapType        = shadowMap.type;

  initGLContext();   // Reinitializes all internal WebGL state

  info.autoReset          = infoAutoReset;
  shadowMap.enabled       = shadowMapEnabled;
  shadowMap.autoUpdate    = shadowMapAutoUpdate;
  shadowMap.needsUpdate   = shadowMapNeedsUpdate;
  shadowMap.type          = shadowMapType;
}
```

Three.js handles the GPU-side re-initialization through `initGLContext()` but it does **not** reload your scene's textures and geometries. You are responsible for re-uploading application-level GPU resources.

### Three.js internal `onContextCreationError`

```js
function onContextCreationError(event) {
  error('WebGLRenderer: A WebGL context could not be created. Reason: ',
        event.statusMessage);
}
```

### `forceContextLoss()` and `forceContextRestore()` (testing)

```js
renderer.forceContextLoss();    // calls WEBGL_lose_context.loseContext()
renderer.forceContextRestore(); // calls WEBGL_lose_context.restoreContext()
```

Both are no-ops if the extension is unavailable. Use these in development to verify your recovery logic.

---

## Full Re-initialization Pattern

This pattern works with plain WebGL. For Three.js, adapt step 4 to reload textures via `TextureLoader` and mark geometries as needing upload.

```js
const canvas = document.getElementById('canvas');
let gl;
let animFrameId;
let isLost = false;

function initGL() {
  gl = canvas.getContext('webgl');
  if (!gl) {
    handleNoWebGL();
    return;
  }

  // Create all GPU resources here
  // e.g. programs, buffers, textures
  setupShaders();
  setupBuffers();
  setupTextures();
}

function render() {
  if (isLost) return; // Guard: never draw into a lost context
  animFrameId = requestAnimationFrame(render);
  draw(gl);
}

function stopRenderLoop() {
  cancelAnimationFrame(animFrameId);
  animFrameId = null;
}

canvas.addEventListener('webglcontextlost', (event) => {
  event.preventDefault();       // (1) Allow the browser to restore later
  isLost = true;                // (2) Set guard flag
  stopRenderLoop();             // (3) Cancel rAF — never call gl commands on lost context
  console.warn('WebGL context lost. Waiting for restore.');
}, false);

canvas.addEventListener('webglcontextrestored', () => {
  isLost = false;               // (4) Clear flag
  // (5) All previous WebGL objects are invalid — rebuild everything
  initGL();
  // (6) Restart render loop
  render();
  console.info('WebGL context restored.');
}, false);

// Bootstrap
initGL();
render();
```

### Why you cannot reuse old objects after restore

After `webglcontextrestored` fires, all objects created before the loss (textures, buffers, programs, framebuffers, renderbuffers, vertex array objects) are invalid. Calling any method on them produces a `INVALID_OPERATION` error. Every resource must be recreated from scratch. Source: MDN `webglcontextrestored` event, accessed 2026-05-26.

---

## Three.js Specific Re-initialization

When using `WebGLRenderer` directly (not R3F), the renderer's internal state is rebuilt by `initGLContext()`. However your scene's `Texture`, `BufferGeometry`, and material uniforms that hold `WebGLTexture` or `WebGLBuffer` handles are stale. Force re-upload:

```js
canvas.addEventListener('webglcontextrestored', () => {
  // Three.js rebuilds its internal program cache and extension cache via initGLContext()
  // but .needsUpdate forces re-upload on next render

  scene.traverse((obj) => {
    if (obj.isMesh) {
      if (obj.geometry) {
        obj.geometry.attributes.position.needsUpdate = true;
      }
      const mats = Array.isArray(obj.material) ? obj.material : [obj.material];
      mats.forEach((mat) => {
        if (mat.map)         mat.map.needsUpdate         = true;
        if (mat.normalMap)   mat.normalMap.needsUpdate   = true;
        if (mat.roughnessMap) mat.roughnessMap.needsUpdate = true;
        mat.needsUpdate = true;
      });
    }
  });

  renderer.render(scene, camera); // trigger re-upload
}, false);
```

---

## Testing Checklist

1. Open DevTools; run `renderer.forceContextLoss()` in the console. Canvas should go blank.
2. Run `renderer.forceContextRestore()`. Scene should reappear without page reload.
3. Verify no GPU resource leaks by checking `renderer.info.memory.textures` and `.geometries` before and after.
4. Test on a real device: iOS Safari reliably loses context on tab background (see `ios-memory-pressure.md`).

---

## Sources

| Source | URL | Tier | Accessed |
|---|---|---|---|
| MDN — `webglcontextlost` event | https://developer.mozilla.org/en-US/docs/Web/API/HTMLCanvasElement/webglcontextlost_event | 1 | 2026-05-26 |
| MDN — `webglcontextrestored` event | https://developer.mozilla.org/en-US/docs/Web/API/HTMLCanvasElement/webglcontextrestored_event | 1 | 2026-05-26 |
| MDN — `isContextLost()` | https://developer.mozilla.org/en-US/docs/Web/API/WebGLRenderingContext/isContextLost | 1 | 2026-05-26 |
| MDN — `WEBGL_lose_context` | https://developer.mozilla.org/en-US/docs/Web/API/WEBGL_lose_context | 1 | 2026-05-26 |
| Three.js — `WebGLRenderer.js` (dev) | https://raw.githubusercontent.com/mrdoob/three.js/dev/src/renderers/WebGLRenderer.js | 1 | 2026-05-26 |
