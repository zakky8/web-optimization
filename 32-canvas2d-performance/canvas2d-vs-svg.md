# Canvas 2D vs SVG vs CSS: Performance Comparison

**Sources:** MDN Web Docs  
**Accessed:** 2026-05-26  
**Scope:** Runtime rendering performance for interactive web graphics

---

## 1. Fundamental Architecture Differences

### Canvas 2D — retained-nothing raster

The `<canvas>` element is a single DOM node that exposes a **bitmap**. All drawing is immediate mode: you issue commands, pixels change, and the API forgets. There is no retained scene graph. Hit testing, animation, and accessibility are entirely the application's responsibility.

```javascript
// Canvas "remembers" nothing — you own the entire state machine
ctx.fillRect(x, y, w, h);  // pixels written; Canvas does not know a rect exists
```

### SVG — retained DOM vector

Every SVG shape (`<circle>`, `<rect>`, `<path>`, `<text>`) is a DOM element. The browser maintains a retained scene graph, re-renders it when attributes change, and exposes each element to CSS, JavaScript events, and the accessibility tree.

```xml
<circle cx="50" cy="50" r="40" fill="blue" />
<!-- This element lives in the DOM; it can receive click events, be styled with CSS,
     and be queried with querySelector -->
```

### CSS — layout-integrated compositing

CSS animations and transitions are resolved by the browser's style engine and, for `transform` and `opacity`, executed entirely on the GPU compositor thread. No JavaScript is involved once the animation is declared. CSS applies to HTML elements only; it has no native "draw a circle" primitive.

---

## 2. Particle Systems — Canvas Wins

A particle system with 10,000 particles updating position every frame requires:
- Updating 10,000 × (x, y, vx, vy, life) state values per tick.
- Rendering 10,000 shapes per frame.

**SVG at this scale is unusable.** Each particle would be a DOM element. 10,000 DOM nodes trigger:
- Layout and style recalculation on every attribute mutation.
- Memory overhead of ~1–2 KB of DOM object per node.
- Event system overhead even if no events are registered.

**Canvas 2D scales to tens of thousands of objects** because drawing is a series of `arc()` / `fillRect()` calls with no DOM involvement:

```javascript
// 10,000 particles — single pass, no DOM nodes
ctx.clearRect(0, 0, W, H);
ctx.beginPath();
for (const p of particles) {
  ctx.moveTo(p.x + p.r, p.y);
  ctx.arc(p.x, p.y, p.r, 0, TWO_PI);
}
ctx.fill(); // one draw call for all 10,000
```

For particle counts above ~50,000 at 60 fps, Canvas 2D's CPU-bound draw loop itself becomes a bottleneck; that threshold calls for WebGL or WebGPU.

> Source: MDN SVG element documentation — accessed 2026-05-26  
> "Large Datasets: Hundreds of DOM nodes = memory overhead. Canvas: Better for thousands of objects"  
> https://developer.mozilla.org/en-US/docs/Web/SVG/Element/svg

---

## 3. Static Graphics — SVG Wins

SVG is the correct format for graphics that do not animate or animate rarely:

- **Resolution independence.** SVG paths are re-rasterized at every display size. A 16px icon and a 512px icon from the same SVG file are equally sharp.
- **No JavaScript required.** Declarative markup renders without any JS execution.
- **CSS styling.** `fill`, `stroke`, `opacity` respond to CSS classes, hover pseudo-classes, and media queries.
- **Accessibility.** `<title>`, `<desc>`, and `aria-label` on SVG elements are read by screen readers.
- **Caching.** An SVG file used as an `<img src>` is cached by the browser like any image; canvas always requires re-execution.

Canvas for static graphics requires:
1. Executing JavaScript to draw.
2. No accessibility without a `<canvas aria-label>` workaround.
3. Pixelation at higher DPI unless the canvas is explicitly scaled to `devicePixelRatio`.

```javascript
// Required for crisp canvas at high DPI — SVG avoids this entirely
canvas.width  = logicalWidth  * devicePixelRatio;
canvas.height = logicalHeight * devicePixelRatio;
ctx.scale(devicePixelRatio, devicePixelRatio);
```

**Verdict:** Use SVG for icons, logos, illustrations, charts, infographics. Use Canvas only when SVG's vector rendering performance is insufficient.

---

## 4. Animated Illustrations — It Depends

The crossover point depends on the number of independently animated elements and their complexity.

### SVG wins when

- Fewer than ~200 elements are animating simultaneously.
- Animations are **declarative** — `transition`, `animation`, CSS `transform` — executed on the GPU compositor thread with no JS involvement.
- Individual elements need **event handlers** (hover states, click effects on specific paths).
- Animations are driven by CSS keyframes or SMIL `<animate>`.

```css
/* GPU-compositor animation — no JS, no layout, smooth even on busy main thread */
.node {
  transition: transform 0.3s ease;
}
.node:hover {
  transform: scale(1.1);
}
```

### Canvas wins when

- Animations involve hundreds of simultaneously moving elements.
- Each element's motion is computed in JavaScript per frame (physics, procedural paths).
- You need to composite partial frames efficiently (dirty-rectangle clearing).
- The illustration includes per-pixel effects (blur trails, additive blending across many objects).

### Practical threshold (indicative)

| Element count | Recommendation |
|---------------|----------------|
| 1–50 | SVG with CSS animation |
| 50–500 | Benchmark both; SVG may still win if animations are CSS-driven |
| 500–5,000 | Canvas 2D |
| 5,000+ | WebGL / WebGPU |

These numbers are not from MDN benchmarks. They reflect widely observed industry heuristics. Always profile in your specific browser/hardware target.

---

## 5. DOM Interaction — SVG Wins

"DOM interaction" means: users click, hover, or keyboard-navigate individual graphical elements; the application responds to those specific elements.

### SVG native event model

Every SVG shape is a DOM element. Standard event handling requires no extra code:

```javascript
const circle = document.querySelector('circle');
circle.addEventListener('click', () => {
  circle.setAttribute('fill', 'red');
});
circle.addEventListener('mouseover', () => {
  circle.style.cursor = 'pointer';
});
```

Event bubbling, `stopPropagation`, `preventDefault`, focus management, and ARIA roles all work as in HTML. SVG supports the full WCAG accessibility toolchain.

### Canvas manual hit testing

Canvas has no concept of shapes after they are drawn. Every interaction requires the application to implement hit testing from scratch:

```javascript
canvas.addEventListener('click', (e) => {
  const rect = canvas.getBoundingClientRect();
  const x = e.clientX - rect.left;
  const y = e.clientY - rect.top;

  // Must test every shape manually
  for (const shape of shapes) {
    if (x >= shape.x && x <= shape.x + shape.w &&
        y >= shape.y && y <= shape.y + shape.h) {
      shape.onClick();
      break;
    }
  }
});
```

For circular or arbitrary path shapes, hit testing requires `ctx.isPointInPath()`, which re-evaluates the path on the CPU:

```javascript
ctx.beginPath();
ctx.arc(shape.cx, shape.cy, shape.r, 0, Math.PI * 2);
if (ctx.isPointInPath(x, y)) {
  shape.onClick();
}
```

Calling `isPointInPath` for hundreds of shapes per mouse-move event is expensive. Libraries like Konva.js and Fabric.js implement optimised spatial indexing (quadtrees, R-trees) to manage this at scale, but the complexity cost is real.

**Accessibility on canvas is near-impossible to retrofit.** The only mechanism is `canvas.getContext('2d').drawFocusIfNeeded()` combined with hidden fallback DOM elements inside `<canvas>`. SVG provides this for free.

> Source: MDN SVG element documentation — accessed 2026-05-26  
> https://developer.mozilla.org/en-US/docs/Web/SVG/Element/svg

---

## 6. CSS for Graphics

CSS (with HTML elements, pseudo-elements, and `clip-path`) can produce surprisingly complex visuals:

- Circles, rounded rects, arbitrary polygons via `clip-path`.
- Gradients, box shadows, borders.
- Fully GPU-compositor-driven `transform` and `opacity` animations.

### CSS wins when

- Visual effects are decorative (buttons, cards, loaders).
- Motion is triggered by state change (hover, focus, class toggle).
- The animation is `transform` / `opacity` only — these composite on the GPU without touching layout.
- Accessibility and DOM integration are required by design.

### CSS loses when

- You need per-pixel procedural patterns (noise, fractals, custom shaders).
- You have more than ~50 independently positioned animated elements (each CSS-animated element requires a GPU layer, and layer count has memory limits).
- You need to read back pixel data.

---

## 7. Summary Decision Matrix

| Use case | Best choice | Why |
|----------|-------------|-----|
| Particle system, 100–50,000 particles | **Canvas 2D** | No DOM overhead; single draw-call batching |
| Particle system, 50,000+ particles | **WebGL / WebGPU** | Canvas 2D CPU loop becomes bottleneck |
| Static icon / logo | **SVG** | Resolution-independent, cached, accessible |
| Static data chart (bars, lines, pie) | **SVG** | Individual elements are queryable; accessibility |
| Animated chart (smooth transitions) | **SVG + CSS** | CSS transforms execute on GPU compositor |
| Interactive diagram (clickable nodes) | **SVG** | Native event model, no hit-test code needed |
| Animated illustration, < 200 elements | **SVG** | CSS animation stays off main thread |
| Animated illustration, 200–5,000 elements | **Canvas 2D** | SVG DOM overhead exceeds Canvas loop cost |
| Pixel-level image manipulation | **Canvas 2D** | `getImageData`, typed array loops |
| Text rendering (rich, multi-line) | **HTML + CSS** | Full layout engine; fallback: Canvas for texture |
| Text in 3D scene (Three.js) | **CanvasTexture** or **troika-three-text** | Depends on update frequency (see canvas-as-texture.md) |
| UI buttons, loaders, card animations | **CSS** | GPU compositor; no JS animation loop needed |
| GPU shader effects | **WebGL / WebGPU** | Canvas 2D cannot express GLSL |

---

## 8. Mixed Approaches

SVG and Canvas are not mutually exclusive. Common hybrids:

- **SVG background + Canvas overlay** — SVG handles the static scene graph; a transparent canvas layered on top renders dynamic particles or paint effects via `pointer-events: none`.
- **Canvas pre-renders into an `<img src="data:…">`** or Blob URL, which SVG then `<image href>` references — useful for procedural textures inside an SVG scene.
- **CSS animation on a Canvas element** — `transform: rotate(45deg)` applied to `<canvas>` is GPU-composited even though the canvas content is rasterized.

> Source: MDN Web Docs, SVG element — accessed 2026-05-26  
> https://developer.mozilla.org/en-US/docs/Web/SVG/Element/svg
