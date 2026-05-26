# anime.js for SVG Animation

> Sources fetched 2026-05-26.
> Primary sources:
> - https://github.com/juliangarnier/anime (repository page)
> - https://animejs.com/documentation/ (SVG section, v4 docs)
> - https://animejs.com/documentation/svg/morphto
> - https://animejs.com/documentation/svg/createdrawable

---

## Library Status (verified 2026-05-26)

| Field | Value |
|-------|-------|
| Latest release | **v4.4.1** (released 2026-04-30) |
| Total releases | 36 |
| Commits on master | 904 |
| GitHub stars | 68,900+ |
| Maintenance | Actively maintained |
| Language | TypeScript |
| Distribution | npm (ESM, UMD, CJS, IIFE) |

The project is actively maintained. v4 was a major rewrite introducing the Web Animations API (WAAPI) integration, TypeScript, and a revised SVG API. The v3 API (`anime({ targets, ... })`) is not forward-compatible with v4.

---

## Installation

```bash
npm install animejs
```

```js
// ESM
import { animate, svg, stagger, createTimeline } from "animejs";

// CDN (IIFE)
<script src="https://cdn.jsdelivr.net/npm/animejs@4/lib/anime.iife.min.js"></script>
// Then: anime.animate(...), anime.svg.morphTo(...)
```

---

## SVG Transforms

anime.js v4 animates CSS `transform` properties on SVG elements using WAAPI for hardware acceleration. Standard transform shorthand properties work:

```js
import { animate } from "animejs";

// Translate, rotate, scale on SVG elements
animate("#circle", {
  translateX: 200,
  translateY: 50,
  rotate: "1turn",
  scale: 1.5,
  duration: 1000,
  ease: "inOutQuad"
});
```

**Important:** Apply `transform-box: fill-box; transform-origin: center;` in CSS to get correct rotation/scale pivots (same requirement as plain CSS animations on SVG — see `css-svg-animation.md`).

---

## Animating SVG Attributes

anime.js v4 can animate SVG DOM attributes alongside CSS properties:

```js
animate("#myCircle", {
  cx: 300,
  cy: 150,
  r: 60,
  duration: 800,
  ease: "outElastic(1, 0.5)"
});

animate("#myRect", {
  x: 100,
  width: 200,
  height: 80,
  duration: 600
});
```

Numeric SVG attributes are interpolated as numbers. String attributes (like `d` or `points`) are handled via the SVG utilities described below.

---

## `svg.morphTo` — Path Morphing

### Signature

```js
svg.morphTo(shapeTarget, precision?)
```

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `shapeTarget` | CSS selector, `SVGPathElement`, `SVGPolylineElement`, `SVGPolygonElement` | required | The shape to morph into |
| `precision` | Number (0–1) | `0.33` | Amount of interpolation points generated. `0` disables point extrapolation |

**Returns:** An `Array` containing `[startValue, endValue]` strings for the relevant attribute (`d` or `points`).

### How It Works

`morphTo` generates intermediate points between start and end shapes so that paths with different numbers of points can morph smoothly. The `precision` value controls how densely points are sampled.

### Supported Element Types

| Element | Attribute animated |
|---------|-------------------|
| `SVGPathElement` | `d` |
| `SVGPolylineElement` | `points` |
| `SVGPolygonElement` | `points` |

### Examples

**Basic path morph:**
```js
import { animate, svg } from "animejs";

animate("#sourcePath", {
  d: svg.morphTo("#targetPath"),
  duration: 1200,
  ease: "inOutCubic"
});
```

**Loop morph (yoyo):**
```js
animate("#blob", {
  d: svg.morphTo("#blobAlt"),
  duration: 2000,
  ease: "inOutSine",
  loop: true,
  alternate: true
});
```

**Polygon to polygon:**
```js
animate("#poly1", {
  points: svg.morphTo("#poly2"),
  duration: 800,
  ease: "outBack(1.5)"
});
```

**Lower precision for performance:**
```js
// precision: 0 — no extra points, instant snap or poor morph for very different shapes
// precision: 1 — maximum smoothness, higher CPU cost
animate("#complexPath", {
  d: svg.morphTo("#otherComplex", 0.5),
  duration: 1500
});
```

---

## `svg.createDrawable` — Stroke Drawing Animation

### Overview

`createDrawable` (available since anime.js v4.0.0) creates a proxy SVG element with an additional `draw` property. This property controls the **visible portion of the stroke** as a normalised range (`0` to `1`), internally implemented via stroke dash properties.

### Signature

```js
svg.createDrawable(target)
```

| Parameter | Type | Description |
|-----------|------|-------------|
| `target` | CSS selector or SVG element | `SVGLineElement`, `SVGPathElement`, `SVGPolylineElement`, or `SVGRectElement` |

**Returns:** An `Array` of proxy element(s) with a `draw` property.

### `draw` Property Syntax

The `draw` property takes a string of two space-separated values in the `0–1` range defining the visible stroke window:

| Value | Stroke visible |
|-------|----------------|
| `'0 1'` | Entire stroke |
| `'0 0'` | Nothing visible |
| `'1 1'` | Nothing visible (at end point) |
| `'0 .5'` | First half |
| `'.5 1'` | Second half |
| `'.25 .75'` | Middle 50% |

### Animation Patterns

**Draw on from start:**
```js
import { animate, svg, stagger } from "animejs";

animate(svg.createDrawable(".line"), {
  draw: ["0 0", "0 1"],
  ease: "inOutQuad",
  duration: 1500
});
```

**Draw off to end (erase):**
```js
animate(svg.createDrawable("#path"), {
  draw: ["0 1", "1 1"],
  ease: "inOutSine",
  duration: 1000
});
```

**Animated segment travelling along stroke:**
```js
// '0 0.2' → '0.8 1' — a 20% segment moves from start to end
animate(svg.createDrawable("#scanner"), {
  draw: ["0 0.2", "0.8 1"],
  ease: "linear",
  duration: 2000,
  loop: true
});
```

**Staggered multi-element draw-on:**
```js
animate(svg.createDrawable(".stroke-element"), {
  draw: ["0 0", "0 1"],
  ease: "inOutQuad",
  duration: 2000,
  delay: stagger(100),
  loop: true
});
```

**Draw from centre outward:**
```js
animate(svg.createDrawable("#centerPath"), {
  draw: [".5 .5", "0 1"],
  ease: "outExpo",
  duration: 1200
});
```

### Performance Note

Animating elements with `vector-effect: non-scaling-stroke` causes the plugin to recalculate scale factors on every frame, which increases CPU load. Avoid combining `createDrawable` with `vector-effect: non-scaling-stroke` in tight animation loops.

---

## `svg.createMotionPath` — Motion Along a Path

```js
import { animate, svg } from "animejs";

const motionPath = svg.createMotionPath("#trackPath");

animate("#car", {
  ...motionPath,
  duration: 3000,
  ease: "linear",
  loop: true
});
```

`createMotionPath` returns an object with `translateX` and `translateY` keyframe arrays derived from the path's geometry. The element follows the path trajectory without needing to be inside the SVG document — it works on any DOM element.

---

## Timeline with SVG Animations

```js
import { createTimeline, svg, stagger } from "animejs";

const tl = createTimeline({ loop: true });

tl
  .add(svg.createDrawable(".line-1"), {
    draw: ["0 0", "0 1"],
    duration: 600,
    ease: "outCubic"
  })
  .add("#icon", {
    opacity: [0, 1],
    scale: [0.5, 1],
    duration: 400,
    ease: "outBack"
  }, "-=200")
  .add(svg.createDrawable(".line-2"), {
    draw: ["0 0", "0 1"],
    duration: 600,
    ease: "outCubic"
  });
```

---

## v3 vs v4 API Differences

| Feature | v3 | v4 |
|---------|----|----|
| Entry point | `anime({ targets, ... })` | `animate(targets, { ... })` |
| Stroke draw | `anime({ targets, strokeDashoffset: [...] })` | `animate(svg.createDrawable(el), { draw: [...] })` |
| Path morph | Manual `d` attribute tween | `animate(el, { d: svg.morphTo(target) })` |
| Motion path | `anime({ targets, path: [...] })` | `animate(el, { ...svg.createMotionPath(path) })` |
| WAAPI | Not used | Integrated for hardware-accelerated props |

If your codebase uses v3, do not assume v4 is a drop-in upgrade — the API surface changed substantially.
