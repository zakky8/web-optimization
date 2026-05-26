# `getImageData` / `putImageData` Performance

**Sources:** MDN Web Docs  
**Accessed:** 2026-05-26  
**Applies to:** `CanvasRenderingContext2D` and `OffscreenCanvasRenderingContext2D`

---

## 1. Why Pixel Readback Is Slow

### The GPU→CPU readback problem

When the browser hardware-accelerates a canvas, pixel data lives in GPU VRAM. `getImageData()` must:

1. Stall the GPU pipeline (wait for all pending draw calls to complete).
2. Copy the pixel rectangle from GPU VRAM into a CPU-accessible buffer.
3. Wrap that buffer in a `Uint8ClampedArray` and return it to JavaScript.

This round-trip breaks the GPU's asynchronous execution model. The main thread blocks until the transfer is complete. On a 1920×1080 canvas, that is 1920 × 1080 × 4 = ~8 MB of data moving across the PCIe bus per call.

> Source: MDN Canvas Pixel Manipulation Tutorial — accessed 2026-05-26  
> https://developer.mozilla.org/en-US/docs/Web/API/Canvas_API/Tutorial/Pixel_manipulation_with_canvas  
> "getImageData() is a GPU-to-CPU readback operation — expensive for large canvas areas"

### `willReadFrequently` context option

If your use case **requires** frequent pixel reads (color pickers, software filters applied every frame), inform the browser at context creation:

```javascript
const ctx = canvas.getContext('2d', { willReadFrequently: true });
```

This switches the canvas to **software (CPU-side) rendering**. The GPU→CPU stall is eliminated because the pixel data never moved to the GPU in the first place. The trade-off is that GPU-accelerated drawing operations (`drawImage`, compositing, shadows) become slower.

Use `willReadFrequently: true` only when `getImageData` calls outnumber drawing calls.

> Source: MDN HTMLCanvasElement.getContext() — accessed 2026-05-26  
> https://developer.mozilla.org/en-US/docs/Web/API/HTMLCanvasElement/getContext

---

## 2. `ImageData` Object and Direct Manipulation

### Structure

`getImageData(left, top, width, height)` returns an `ImageData` object:

```javascript
const imageData = ctx.getImageData(left, top, width, height);
// imageData.data    — Uint8ClampedArray, length = width × height × 4
// imageData.width   — pixel width of the rectangle
// imageData.height  — pixel height of the rectangle
```

Each pixel occupies four consecutive bytes in `imageData.data`:

```
Index:   [0]  [1]  [2]  [3]   [4]  [5]  [6]  [7]  ...
Channel:  R    G    B    A     R    G    B    A   ...
          ← pixel 0 ───────►  ← pixel 1 ───────►
```

### Accessing a specific pixel

```javascript
// Pixel at column x, row y in a canvas of width `w`
const index = (y * w + x) * 4;
const r = data[index];
const g = data[index + 1];
const b = data[index + 2];
const a = data[index + 3];
```

### Writing back

```javascript
ctx.putImageData(imageData, dx, dy);
// dx, dy: destination coordinates on the canvas
```

`putImageData` bypasses `globalCompositeOperation`, `globalAlpha`, shadows, and transforms — it is a direct pixel write.

---

## 3. Typed Arrays for Pixel Operations

### `Uint8ClampedArray`

`imageData.data` is a `Uint8ClampedArray`. Writes automatically clamp to [0, 255] — no explicit bounds-checking needed:

```javascript
data[i] = someValue;      // auto-clamped: 300 → 255, -5 → 0
```

### Performance patterns

**Grayscale (iterate every 4th index for alpha skip):**
```javascript
const data = ctx.getImageData(0, 0, w, h).data;
for (let i = 0; i < data.length; i += 4) {
  const avg = (data[i] + data[i + 1] + data[i + 2]) / 3;
  data[i] = avg;      // R
  data[i + 1] = avg;  // G
  data[i + 2] = avg;  // B
  // data[i + 3] = alpha — left unchanged
}
```

**Invert:**
```javascript
for (let i = 0; i < data.length; i += 4) {
  data[i]     = 255 - data[i];
  data[i + 1] = 255 - data[i + 1];
  data[i + 2] = 255 - data[i + 2];
}
```

**Sepia (ITU-R BT.601 approximation):**
```javascript
for (let i = 0; i < data.length; i += 4) {
  const r = data[i], g = data[i + 1], b = data[i + 2];
  data[i]     = Math.min(Math.round(0.393 * r + 0.769 * g + 0.189 * b), 255);
  data[i + 1] = Math.min(Math.round(0.349 * r + 0.686 * g + 0.168 * b), 255);
  data[i + 2] = Math.min(Math.round(0.272 * r + 0.534 * g + 0.131 * b), 255);
}
```

### 32-bit integer trick for faster pixel access

Reading/writing one 32-bit integer per pixel instead of four 8-bit values cuts loop iterations by 4×. Use a shared `ArrayBuffer` with both `Uint8ClampedArray` and `Uint32Array` views:

```javascript
const imageData = ctx.getImageData(0, 0, w, h);
const buf = imageData.data.buffer;
const data8  = new Uint8ClampedArray(buf);  // RGBA bytes
const data32 = new Uint32Array(buf);        // one int per pixel

// Write RGBA 0xFF0000FF (red, fully opaque) to pixel 100
// Note: byte order is little-endian on most systems → ABGR in Uint32
data32[100] = 0xFF0000FF; // verify endianness in your environment
```

Use with caution — endianness varies. Test in your target environments.

### Offloading pixel work to a worker

`ImageData` is **not** transferable, but its `.data.buffer` (`ArrayBuffer`) is. You can transfer the buffer to a worker for processing:

```javascript
// Main thread: get data and transfer buffer to worker
const imageData = ctx.getImageData(0, 0, w, h);
const buffer = imageData.data.buffer;
worker.postMessage({ buffer, width: w, height: h }, [buffer]);

// Worker: process pixels
onmessage = (e) => {
  const data = new Uint8ClampedArray(e.data.buffer);
  // ... process pixels
  postMessage({ buffer: data.buffer }, [data.buffer]);
};

// Main thread: receive processed buffer and put back
worker.onmessage = (e) => {
  const processedData = new Uint8ClampedArray(e.data.buffer);
  const result = new ImageData(processedData, w, h);
  ctx.putImageData(result, 0, 0);
};
```

---

## 4. `createImageBitmap()` for Async Processing

### Method signature

```javascript
createImageBitmap(image)
createImageBitmap(image, options)
createImageBitmap(image, sx, sy, sw, sh)
createImageBitmap(image, sx, sy, sw, sh, options)
```

**Returns:** `Promise<ImageBitmap>`

### Accepted source types

`HTMLImageElement`, `SVGImageElement`, `HTMLVideoElement`, `HTMLCanvasElement`, `Blob`, `ImageData`, `ImageBitmap`, `OffscreenCanvas`, `VideoFrame`

### Options

| Option | Values | Default |
|--------|--------|---------|
| `imageOrientation` | `"from-image"`, `"flipY"`, `"none"` | `"from-image"` |
| `premultiplyAlpha` | `"none"`, `"premultiply"`, `"default"` | `"default"` |
| `colorSpaceConversion` | `"none"`, `"default"` | `"default"` |
| `resizeWidth` | integer | — |
| `resizeHeight` | integer | — |
| `resizeQuality` | `"pixelated"`, `"low"`, `"medium"`, `"high"` | `"low"` |

### Why it is faster than `getImageData`

| Dimension | `getImageData` | `createImageBitmap` |
|-----------|---------------|---------------------|
| **Execution** | Synchronous — blocks main thread | Asynchronous — returns Promise |
| **GPU stall** | Yes — forces GPU→CPU copy immediately | Deferred; browser schedules optimally |
| **Result type** | `Uint8ClampedArray` (CPU memory) | `ImageBitmap` (GPU-ready handle) |
| **Use for drawing** | Must go through `putImageData` | Can pass directly to `drawImage`, WebGL textures, `CanvasTexture` |
| **Worker support** | Available in workers | Available as `WorkerGlobalScope.createImageBitmap()` |

> Source: MDN Window.createImageBitmap() — accessed 2026-05-26  
> https://developer.mozilla.org/en-US/docs/Web/API/Window/createImageBitmap

### Example: async sprite sheet slicing

```javascript
const image = new Image();
image.src = 'spritesheet.png';
image.onload = async () => {
  const [idle, run, jump] = await Promise.all([
    createImageBitmap(image, 0,   0, 64, 64),
    createImageBitmap(image, 64,  0, 64, 64),
    createImageBitmap(image, 128, 0, 64, 64),
  ]);
  // Use in drawImage — no getImageData involved
  ctx.drawImage(idle, playerX, playerY);
};
```

### Example: worker-side image processing with createImageBitmap

```javascript
// Worker
onmessage = async (e) => {
  const { blob } = e.data;
  const bitmap = await createImageBitmap(blob, {
    imageOrientation: 'flipY',
    resizeWidth: 256,
    resizeHeight: 256,
    resizeQuality: 'high'
  });
  // bitmap is ready to drawImage onto OffscreenCanvas
  postMessage({ bitmap }, [bitmap]);
};
```

---

## 5. Optimization Checklist

1. **Never call `getImageData` on every frame** for large canvases unless `willReadFrequently: true` is set.
2. **Read only the region you need** — `getImageData(x, y, smallWidth, smallHeight)` instead of the full canvas.
3. **Prefer `createImageBitmap`** over `getImageData` when the goal is to decode, resize, or transfer an image — it avoids the GPU stall.
4. **Use typed array views** (`Uint32Array` over a shared buffer) to process pixels faster when manipulating raw bytes.
5. **Move pixel-intensive work to a worker** using transferable `ArrayBuffer` to avoid blocking the main thread.
6. **Avoid `getImageData` at all when possible** — canvas CSS filters (`filter: blur(4px)`) or SVG filters applied via `ctx.filter` can be faster for visual effects because they stay GPU-side.

---

## 6. CORS and Tainted Canvases

A canvas is "tainted" if it has drawn any pixel from a cross-origin image that lacks appropriate CORS headers. Tainted canvases throw `SecurityError` on:
- `getImageData()`
- `toDataURL()`
- `toBlob()`

Fix by setting `crossOrigin = "anonymous"` before loading the image:

```javascript
const img = new Image();
img.crossOrigin = 'anonymous';
img.src = 'https://other-origin.example/image.png';
```

The server must respond with `Access-Control-Allow-Origin: *` or the specific origin.
