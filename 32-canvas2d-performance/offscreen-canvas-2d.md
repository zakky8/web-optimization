# OffscreenCanvas for 2D Rendering

**Sources:** MDN Web Docs  
**Accessed:** 2026-05-26  
**Browser support:** Baseline Widely Available since March 2023

---

## 1. What OffscreenCanvas Is

`OffscreenCanvas` decouples the Canvas API from the DOM. Rendering operations run in any context that has access to the object — including Web Workers — without blocking the main UI thread.

**Key property:** `OffscreenCanvas` is a [Transferable Object](https://developer.mozilla.org/en-US/docs/Web/API/Web_Workers_API/Transferable_objects). Ownership can be moved to a worker via `postMessage` at zero copy cost.

> Source: MDN OffscreenCanvas — accessed 2026-05-26  
> https://developer.mozilla.org/en-US/docs/Web/API/OffscreenCanvas

---

## 2. `transferControlToOffscreen()` in a 2D Context

### Method signature

```javascript
HTMLCanvasElement.transferControlToOffscreen()
// Parameters: none
// Returns: OffscreenCanvas
```

### What it does

Transfers rendering control from the `<canvas>` DOM element to an `OffscreenCanvas` instance. After the call:
- The `HTMLCanvasElement` on the main thread becomes a passive display surface.
- All drawing must happen through the returned `OffscreenCanvas` (or through a worker that owns it).
- Calling `getContext()` directly on the original element after `transferControlToOffscreen()` throws `InvalidStateError`.

### Constraints

`transferControlToOffscreen()` must be called **before** any call to `getContext()` on the same element. Calling `getContext()` first and then attempting `transferControlToOffscreen()` throws `InvalidStateError`.

Similarly, calling `transferControlToOffscreen()` twice on the same element throws `InvalidStateError`.

> Source: MDN HTMLCanvasElement.transferControlToOffscreen() — accessed 2026-05-26  
> https://developer.mozilla.org/en-US/docs/Web/API/HTMLCanvasElement/transferControlToOffscreen

### Basic 2D worker setup

**Main thread:**
```javascript
const htmlCanvas = document.getElementById('canvas');
const offscreen = htmlCanvas.transferControlToOffscreen();

const worker = new Worker('render-worker.js');
// Transfer ownership — offscreen is now owned by the worker
worker.postMessage({ canvas: offscreen }, [offscreen]);
```

**render-worker.js:**
```javascript
onmessage = (event) => {
  const canvas = event.data.canvas;
  const ctx = canvas.getContext('2d'); // OffscreenCanvasRenderingContext2D

  function render(timestamp) {
    ctx.clearRect(0, 0, canvas.width, canvas.height);
    // ... draw
    requestAnimationFrame(render);
  }
  requestAnimationFrame(render);
};
```

`requestAnimationFrame` is available inside workers when the canvas was transferred via `transferControlToOffscreen`. The browser schedules worker frames in sync with the display.

---

## 3. `OffscreenCanvasRenderingContext2D`

The 2D context returned by `offscreenCanvas.getContext('2d')` is an `OffscreenCanvasRenderingContext2D`. It inherits virtually the entire `CanvasRenderingContext2D` surface area:

- All path methods: `beginPath`, `moveTo`, `lineTo`, `arc`, `bezierCurveTo`, etc.
- All style properties: `fillStyle`, `strokeStyle`, `globalAlpha`, `globalCompositeOperation`, etc.
- All image/pixel methods: `drawImage`, `getImageData`, `putImageData`, `createImageData`
- State management: `save`, `restore`, `reset`
- Text rendering: `fillText`, `strokeText`, `measureText`

**Only difference:** `drawFocusIfNeeded()` is absent (no DOM focus concept in a worker).

The `canvas` property on the context references the `OffscreenCanvas` object, not an `HTMLCanvasElement`.

> Source: MDN OffscreenCanvasRenderingContext2D — accessed 2026-05-26  
> https://developer.mozilla.org/en-US/docs/Web/API/OffscreenCanvasRenderingContext2D

---

## 4. `ImageBitmap` Transfer Back to the Main Thread

When the offscreen canvas is **not** transferred via `transferControlToOffscreen` — i.e., it was constructed directly (`new OffscreenCanvas(w, h)`) — you must manually push rendered frames back to the main thread using `ImageBitmap`.

### `transferToImageBitmap()`

```javascript
// Inside worker
const bitmap = offscreenCanvas.transferToImageBitmap();
// Returns: ImageBitmap from the most recently rendered frame
// Side effect: the OffscreenCanvas is now blank and ready for next frame
```

The returned `ImageBitmap` is also a Transferable Object, so it can be posted to the main thread at zero copy cost.

### Displaying with `ImageBitmapRenderingContext`

On the main thread, use a canvas with context type `"bitmaprenderer"`:

```javascript
// Main thread setup
const displayCanvas = document.getElementById('display');
const bitmapRenderer = displayCanvas.getContext('bitmaprenderer');

// Worker sends frames
worker.onmessage = (event) => {
  bitmapRenderer.transferFromImageBitmap(event.data.bitmap);
  // bitmap ownership is now consumed; worker must produce a new one next frame
};
```

**Worker render loop:**
```javascript
// Worker thread
const offscreen = new OffscreenCanvas(800, 600);
const ctx = offscreen.getContext('2d');

function renderFrame(timestamp) {
  ctx.clearRect(0, 0, offscreen.width, offscreen.height);
  // ... draw scene

  const bitmap = offscreen.transferToImageBitmap();
  // Transfer bitmap to main thread — zero copy
  postMessage({ bitmap }, [bitmap]);

  requestAnimationFrame(renderFrame);
}
requestAnimationFrame(renderFrame);
```

One `OffscreenCanvas` can fan frames out to multiple `ImageBitmapRenderingContext` objects.

> Source: MDN OffscreenCanvas — accessed 2026-05-26  
> https://developer.mozilla.org/en-US/docs/Web/API/OffscreenCanvas

---

## 5. Worker-Based 2D Rendering Pipeline

### Architecture diagram

```
Main Thread                          Worker Thread
─────────────────────────────────────────────────────────
HTMLCanvasElement                    OffscreenCanvas
   │                                      │
   │ transferControlToOffscreen()         │
   └─────────── postMessage ─────────────►│
                [offscreen]               │ getContext('2d')
                                          │
                                    OffscreenCanvasRenderingContext2D
                                          │
                                    requestAnimationFrame loop
                                          │
                                    draw scene
                                          │
                                    (display update pushed
                                     automatically by browser)
```

### Standalone OffscreenCanvas with manual frame push

```
Main Thread                          Worker Thread
─────────────────────────────────────────────────────────
new OffscreenCanvas(w, h)            receives canvas
   │                                      │
   │ postMessage({ canvas }, [canvas])    │ getContext('2d')
   └─────────────────────────────────────►│
                                          │ draw scene
                                          │ transferToImageBitmap()
                ◄─────── postMessage ─────┘
                [bitmap]
                   │
            bitmaprenderer
                   │
            transferFromImageBitmap(bitmap)
```

### When to use each approach

| Approach | Use when |
|----------|----------|
| `transferControlToOffscreen` | You want the browser to handle frame pacing automatically; single display canvas |
| Manual `ImageBitmap` transfer | You need to drive multiple canvases from one worker; you want explicit control over when frames land |

---

## 6. OffscreenCanvas 2D vs OffscreenCanvas WebGL

Both use the same `OffscreenCanvas` object. The difference is in `getContext()`:

```javascript
const ctx2d  = offscreen.getContext('2d');      // OffscreenCanvasRenderingContext2D
const gl     = offscreen.getContext('webgl');   // WebGLRenderingContext
const gl2    = offscreen.getContext('webgl2');  // WebGL2RenderingContext
```

| Dimension | OffscreenCanvas 2D | OffscreenCanvas WebGL |
|-----------|--------------------|-----------------------|
| **API complexity** | Low — same as familiar `CanvasRenderingContext2D` | High — full GLSL shader pipeline |
| **Suited for** | UI rendering, text, charts, 2D game logic | 3D scenes, GPU particle systems, post-processing |
| **CPU overhead** | Higher per draw call (JS→GPU path per operation) | Lower at scale — batches via GPU buffers |
| **Worker compatibility** | Full (`OffscreenCanvasRenderingContext2D`) | Full (`WebGLRenderingContext` in worker) |
| **Frame transfer** | `transferToImageBitmap()` or auto via `transferControlToOffscreen` | Same |
| **GPU memory** | Depends on canvas size | Depends on buffers/textures |

For 2D use cases — dynamic text overlays, HUD elements, chart rendering — OffscreenCanvas 2D in a worker is sufficient and avoids the complexity of WebGL. For particle systems above ~50,000 elements per frame, WebGL (or WebGPU) in a worker is the correct choice.

---

## 7. Performance Characteristics

### Why off-main-thread matters

The browser's main thread handles JavaScript execution, layout, style recalculation, and user input. A canvas animation loop that runs on the main thread competes with all of those. A 16ms frame budget shared between a complex render loop and DOM work often results in dropped frames.

`OffscreenCanvas` moves the render loop entirely off the main thread. The main thread remains responsive to input even if the render worker momentarily takes longer than 16ms.

### Limitations

- Worker threads have no access to the DOM; `document`, `window`, and DOM APIs are unavailable. Pure drawing logic only.
- Context loss events (`contextlost`, `contextrestored`) fire on the `OffscreenCanvas` object; worker code must handle them.
- Browser support for `transferControlToOffscreen` became Baseline Widely Available in March 2023. For Safari versions before 16.4, a fallback to main-thread canvas rendering is required.

> Source: MDN OffscreenCanvas — accessed 2026-05-26  
> https://developer.mozilla.org/en-US/docs/Web/API/OffscreenCanvas
