# Canvas 2D Performance Fundamentals

**Sources:** MDN Web Docs, web.dev/canvas-performance  
**Accessed:** 2026-05-26  
**Applies to:** `CanvasRenderingContext2D` in all evergreen browsers (Baseline Widely Available since 2015)

---

## 1. Clearing the Canvas: `clearRect` vs `fillRect`

### `clearRect(x, y, width, height)`

The standard, semantically correct way to erase a region. Sets all pixels in the rectangle to transparent black (`rgba(0,0,0,0)`).

```javascript
// Clear the entire canvas each frame
ctx.clearRect(0, 0, canvas.width, canvas.height);
```

**Important:** After calling `clearRect`, call `beginPath()` before drawing new shapes. Failure to do so can produce unintended rendering artifacts from accumulated path state.

> Source: MDN CanvasRenderingContext2D: clearRect() — accessed 2026-05-26  
> https://developer.mozilla.org/en-US/docs/Web/API/CanvasRenderingContext2D/clearRect

### `fillRect` as a clear substitute

When the canvas has `alpha: false` (opaque background), painting over the entire canvas with `fillRect` can serve as a clear. This avoids the compositing pass that `clearRect` triggers on transparent canvases, but it only makes sense when no transparency is needed in the final output.

```javascript
const ctx = canvas.getContext('2d', { alpha: false });
// Equivalent to clearRect in an opaque canvas context
ctx.fillStyle = '#000000';
ctx.fillRect(0, 0, canvas.width, canvas.height);
```

### Resize-based clear (avoid)

Setting `canvas.width = canvas.width` clears the canvas but also resets **all context state** (transforms, styles, clipping regions). This is an old trick with non-obvious side effects and inconsistent performance across implementations. Do not use it in production code.

### Which is faster?

Performance varies by browser and GPU driver. `clearRect` is generally preferred because:
- It communicates intent to the browser (transparent erase vs. opaque fill).
- On transparent canvases, it is the only correct choice.
- On opaque canvases (`alpha: false`), `fillRect` may be marginally faster because the browser skips alpha compositing.

Benchmark both in your specific scenario if clearing is a measured bottleneck.

> Source: web.dev/canvas-performance — accessed 2026-05-26  
> https://web.dev/articles/canvas-performance

---

## 2. Context Creation Options That Affect Performance

Pass a second argument to `getContext('2d', options)` to hint the browser:

```javascript
const ctx = canvas.getContext('2d', {
  alpha: false,           // opaque canvas — skip alpha compositing
  desynchronized: true,   // low-latency; decouples paint from event loop
  willReadFrequently: true // forces software rasteriser; faster for getImageData loops
});
```

| Option | Default | Effect |
|--------|---------|--------|
| `alpha` | `true` | `false` removes alpha channel; browser may skip compositing passes |
| `desynchronized` | `false` | Reduces latency for pointer/pen-input apps by allowing canvas to paint outside the normal frame cycle |
| `willReadFrequently` | `false` | Switches to CPU-side rendering; avoids GPU→CPU readback stalls when `getImageData()` is called frequently |
| `colorSpace` | `"srgb"` | `"display-p3"` for wide-gamut; slight overhead |
| `colorType` | `"unorm8"` | `"float16"` for HDR; heavier memory footprint |

> Source: MDN HTMLCanvasElement.getContext() — accessed 2026-05-26  
> https://developer.mozilla.org/en-US/docs/Web/API/HTMLCanvasElement/getContext

---

## 3. `save()` / `restore()` Stack Cost

`save()` pushes a snapshot of the **entire drawing state** onto a stack. `restore()` pops it. Neither touches pixel data — only the state machine.

### What is saved

Every call to `save()` captures:
- Current transformation matrix
- Clipping region and dash list
- `fillStyle`, `strokeStyle`, `globalAlpha`, `globalCompositeOperation`
- `lineWidth`, `lineCap`, `lineJoin`, `miterLimit`, `lineDashOffset`
- `shadowColor`, `shadowBlur`, `shadowOffsetX`, `shadowOffsetY`
- `font`, `fontKerning`, `fontStretch`, `fontVariantCaps`, `textAlign`, `textBaseline`, `textRendering`, `letterSpacing`, `wordSpacing`
- `filter`, `imageSmoothingEnabled`, `imageSmoothingQuality`
- `direction`

That is approximately 25 fields. Each `save()` allocates this record on the stack.

### Performance implications

- A single save/restore pair has negligible cost in isolation.
- **Nested save/restore in tight loops** (e.g., called per particle per frame) accumulates allocation and GC pressure.
- MDN does not specify a hard stack depth limit; in practice browsers allow hundreds of levels, but deep stacks signal a design problem.

### Alternatives for shallow state changes

If you only need to change `fillStyle` and restore it afterward, save/restore is unnecessary overhead. Manually record and reset the property:

```javascript
// Instead of save/restore for a single property change:
const previousFill = ctx.fillStyle;
ctx.fillStyle = 'red';
ctx.fillRect(10, 10, 50, 50);
ctx.fillStyle = previousFill;
```

Use `save()` / `restore()` when multiple properties change together, or when a transform + clip combination needs clean isolation.

```javascript
// Correct idiomatic use: transform + clip isolation
ctx.save();
ctx.translate(cx, cy);
ctx.rotate(angle);
ctx.clip();
drawComplexShape(ctx);
ctx.restore();
```

> Source: MDN CanvasRenderingContext2D: save() — accessed 2026-05-26  
> https://developer.mozilla.org/en-US/docs/Web/API/CanvasRenderingContext2D/save

---

## 4. Path Batching and `beginPath()`

### What `beginPath()` does

`beginPath()` **empties the list of sub-paths**, starting a fresh path. Without it, every subsequent `moveTo`, `lineTo`, `arc`, etc. appends to the existing path.

```javascript
beginPath()  // no parameters, returns undefined
```

Failure to call `beginPath()` means:
- Sub-paths accumulate indefinitely in the context.
- `stroke()` / `fill()` replays the entire accumulated path each call — O(n) cost that grows per frame.
- Different segments cannot have different `strokeStyle` / `fillStyle`.

> Source: MDN CanvasRenderingContext2D: beginPath() — accessed 2026-05-26  
> https://developer.mozilla.org/en-US/docs/Web/API/CanvasRenderingContext2D/beginPath

### Path batching pattern

**Anti-pattern** — one stroke call per line:
```javascript
for (const line of lines) {
  ctx.beginPath();
  ctx.moveTo(line.x1, line.y1);
  ctx.lineTo(line.x2, line.y2);
  ctx.stroke(); // GPU draw call per line
}
```

**Batched pattern** — one `beginPath`, many sub-paths, one `stroke`:
```javascript
ctx.beginPath();
for (const line of lines) {
  ctx.moveTo(line.x1, line.y1);
  ctx.lineTo(line.x2, line.y2);
}
ctx.stroke(); // single GPU draw call
```

The batched version issues one draw call regardless of line count. The performance delta becomes significant above ~100 objects per frame.

### When batching does not apply

Batching requires all shapes in the batch to share the same `strokeStyle`, `fillStyle`, `lineWidth`, and other style properties. When styles vary per element, you must start a new batch per style group:

```javascript
// Group by color, not by position
for (const [color, items] of groupByColor(elements)) {
  ctx.strokeStyle = color;
  ctx.beginPath();
  for (const item of items) {
    ctx.moveTo(item.x, item.y);
    // ... path commands
  }
  ctx.stroke();
}
```

> Source: web.dev/canvas-performance — accessed 2026-05-26  
> "It's cheaper to render by color rather than by placement"

---

## 5. `globalCompositeOperation` Modes and GPU Cost

### Default: `source-over`

Every pixel written blends the source color over the existing destination:

```
output = src_alpha * src_color + (1 - src_alpha) * dst_color
```

This is what the GPU executes natively and is cheapest.

### Full list of modes

| Mode | Category | Relative Cost |
|------|----------|---------------|
| `source-over` | Basic | Low (native blend) |
| `copy` | Basic | Low (overwrites) |
| `source-in` | Basic | Low |
| `source-out` | Basic | Low |
| `source-atop` | Basic | Low |
| `destination-over` | Basic | Low |
| `destination-in` | Basic | Low |
| `destination-out` | Basic | Low |
| `destination-atop` | Basic | Low |
| `xor` | Basic | Low–Medium |
| `lighter` | Color add | Medium |
| `multiply` | Photoshop-style | Medium |
| `screen` | Photoshop-style | Medium |
| `overlay` | Photoshop-style | Medium |
| `hard-light` | Photoshop-style | Medium |
| `soft-light` | Photoshop-style | Medium |
| `darken` | Darkening | Medium |
| `lighten` | Lightening | Medium |
| `color-dodge` | Photoshop-style | High |
| `color-burn` | Photoshop-style | High |
| `difference` | Photoshop-style | High |
| `exclusion` | Photoshop-style | High |
| `hue` | HSL component | High |
| `saturation` | HSL component | High |
| `color` | HSL component | High |
| `luminosity` | HSL component | High |

MDN does not publish specific GPU cost benchmarks. The cost ordering above reflects general GPU blend equation complexity:
- Simple modes (`source-over`, `copy`) map to native GPU blend states.
- Photoshop-style modes (`multiply`, `screen`, `overlay`) require more arithmetic per pixel.
- HSL-component modes (`hue`, `saturation`, `color`, `luminosity`) require HSL↔RGB conversion per pixel and may fall back to CPU rendering in some browser/driver combinations.

> Source: MDN CanvasRenderingContext2D: globalCompositeOperation — accessed 2026-05-26  
> https://developer.mozilla.org/en-US/docs/Web/API/CanvasRenderingContext2D/globalCompositeOperation

### Practical guidance

```javascript
// Change composite mode only when necessary; reset immediately after
ctx.save();
ctx.globalCompositeOperation = 'multiply';
drawBlendedLayer(ctx);
ctx.restore(); // resets to source-over
```

- Avoid toggling `globalCompositeOperation` inside per-object loops.
- Batch all objects that share the same composite mode into one rendering pass.
- If using `xor` or `difference` for masking effects, consider pre-rendering the mask to an offscreen canvas instead of compositing live.

---

## 6. Additional Fundamentals

### Integer coordinates

Sub-pixel coordinates trigger anti-aliasing, which costs ~2× the fill work. Round coordinates before drawing:

```javascript
// Bitwise OR truncates to integer (faster than Math.floor for positive values)
const x = (0.5 + rawX) | 0;
const y = (0.5 + rawY) | 0;
```

### Avoid `shadowBlur`

`shadowBlur` is described as "very expensive" in web.dev documentation. It applies a Gaussian blur to a composited shadow layer before rendering. Avoid it in any per-frame animation loop.

### Pre-render to offscreen canvas

For shapes drawn identically each frame, render once to an offscreen canvas and `drawImage` the result:

```javascript
// One-time setup
const cache = document.createElement('canvas');
cache.width = 200; cache.height = 200;
const cacheCtx = cache.getContext('2d');
drawExpensiveShape(cacheCtx);

// Per-frame render
ctx.drawImage(cache, x, y);
```

### Use `requestAnimationFrame`, not `setInterval`

`requestAnimationFrame` synchronises with the display refresh rate, suspends when the tab is hidden, and aligns with the browser's compositing pipeline.

```javascript
function render(timestamp) {
  ctx.clearRect(0, 0, canvas.width, canvas.height);
  update(timestamp);
  draw(ctx);
  requestAnimationFrame(render);
}
requestAnimationFrame(render);
```
