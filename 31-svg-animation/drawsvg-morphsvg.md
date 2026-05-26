# DrawSVG and MorphSVG Plugins (GSAP)

> Sources fetched 2026-05-26.
> Primary sources:
> - https://gsap.com/docs/v3/Plugins/DrawSVGPlugin/
> - https://gsap.com/docs/v3/Plugins/MorphSVGPlugin/
> - https://gsap.com/pricing/ (confirmed: both plugins are free as of Webflow's acquisition)

---

## DrawSVGPlugin

### What It Does

DrawSVGPlugin animates the visible portion of an SVG stroke by controlling `stroke-dasharray` and `stroke-dashoffset` under the hood. It works with `<path>`, `<line>`, `<polyline>`, `<polygon>`, `<rect>`, and `<ellipse>` elements.

Unlike manually computing `getTotalLength()`, DrawSVG exposes a percentage-based API and handles several browser edge cases automatically.

### Installation

```js
// CDN
<script src="https://cdn.jsdelivr.net/npm/gsap@3/dist/DrawSVGPlugin.min.js"></script>

// npm (register after import)
import { gsap } from "gsap";
import { DrawSVGPlugin } from "gsap/DrawSVGPlugin";
gsap.registerPlugin(DrawSVGPlugin);
```

### `drawSVG` Value Syntax

The `drawSVG` value defines **the visible segment of the stroke**, not the animation range. It uses percentage positions along the path:

| Value | Meaning |
|-------|---------|
| `0` | Fully hidden |
| `"100%"` | Fully visible (equivalent to `"0 100%"`) |
| `"20% 80%"` | Show the segment between 20% and 80% of the path length |
| `"50% 50%"` | Zero-length segment at the midpoint (invisible) |

**Single value shorthand:** `drawSVG: "70%"` is equivalent to `drawSVG: "0 70%"` — shows from start to 70%.

### Common Animation Patterns

**Draw on from start:**
```js
gsap.from(".path", { drawSVG: 0, duration: 2, ease: "power2.inOut" });
```

**Draw on from end (reverse):**
```js
// Animate a visible window travelling to the end and disappearing
gsap.to(".path", {
  drawSVG: "100% 100%",
  duration: 2,
  ease: "power1.in"
});
```

**Draw outward from centre:**
```js
// Start: zero-width segment at midpoint. End: full stroke.
gsap.fromTo(".path",
  { drawSVG: "50% 50%" },
  { drawSVG: "0% 100%", duration: 1.5 }
);
```

**Staggered multi-path draw:**
```js
gsap.from(".draw-me", {
  drawSVG: 0,
  duration: 1,
  stagger: 0.15,
  ease: "power2.out"
});
```

**Travelling segment (scanner / loading effect):**
```js
gsap.to(".scanner", {
  drawSVG: "80% 100%",   // 20% segment moves from 80% to 100%
  duration: 1,
  ease: "none",
  repeat: -1
});
```

### `live` Modifier

Append `"live"` to a value when the SVG element's dimensions may change during animation (e.g. responsive layouts, viewport resize):

```js
gsap.to(".responsive-path", {
  drawSVG: "0 100% live",
  duration: 2
});
```

Without `live`, path lengths are computed once at tween creation time.

### Utility Methods

```js
DrawSVGPlugin.getLength(element);    // Returns total stroke length in user units
DrawSVGPlugin.getPosition(element);  // Returns current [start, end] DrawSVG position
```

### Known Limitations and Bugs

| Issue | Workaround |
|-------|-----------|
| Firefox occasionally miscalculates `getTotalLength()`, resulting in a gap at 100% | Use `drawSVG: "102%"` instead of `"100%"` |
| iOS Safari has rendering bugs with `<rect>` elements | Convert `<rect>` to a `<path>` with equivalent geometry |
| Cannot affect contents of `<use>` elements | Browser restriction — use actual elements or inline the symbol |
| Only animates **strokes**, not fills | Use GSAP's `attr: { d: "..." }` or MorphSVG for fill-based shape changes |

### SVG Element Requirements

The element must have a stroke defined before DrawSVG will work:

```css
.draw-target {
  stroke: #e74c3c;
  stroke-width: 3px;
  fill: none;  /* usually needed so fill doesn't obscure the stroke */
}
```

---

## MorphSVGPlugin

### What It Does

MorphSVGPlugin morphs any SVG `<path>` element into another shape by interpolating the `d` attribute. It automatically handles paths with different numbers of points by converting all paths to cubic beziers and subdividing them as needed.

### Installation

```js
import { MorphSVGPlugin } from "gsap/MorphSVGPlugin";
gsap.registerPlugin(MorphSVGPlugin);
```

### Basic Usage

```js
// Morph #circle into the shape defined by #star's d attribute
gsap.to("#circle", {
  duration: 1,
  morphSVG: "#star"
});

// Inline path data
gsap.to("#shape", {
  morphSVG: "M10,30 A20,20 0,0,1 50,30 A20,20 0,0,1 90,30 Q90,60 50,90 Z",
  duration: 1
});
```

The target element keeps its own styles (fill, stroke, etc.) — only the path geometry changes.

### How MorphSVG Handles Different Point Counts

1. All path data is converted to **cubic beziers** internally.
2. The plugin **subdivides** the path with fewer points, adding anchors until both paths have an equal number of points.
3. It then interpolates anchor positions frame by frame.

This means any two closed or open paths can morph into each other, regardless of their original command structure.

### `shapeIndex` Parameter

`shapeIndex` controls which point on the start path aligns with the first point of the end path. This affects the "route" the morph takes and can eliminate unwanted crossing or twisting artefacts.

- Default: GSAP picks the best alignment automatically.
- Range: `0` to `n-1` (where `n` = number of resulting points), or a negative value to **reverse** the path direction.
- Only applies to **closed paths** (paths ending with `Z`/`z`).

```js
gsap.to("#blob", {
  morphSVG: { shape: "#target", shapeIndex: 3 },
  duration: 1
});
```

**Finding the right value:** Use the `findShapeIndex()` utility (see below).

### `type` Parameter

Controls the interpolation algorithm:

| Value | Behaviour |
|-------|-----------|
| `"linear"` (default) | Interpolates raw x/y coordinates of anchor points |
| `"rotational"` | Interpolates using angle and length of bezier handles — produces more natural organic morphs, prevents kinks in smooth anchors |

```js
gsap.to("#organic", {
  morphSVG: { shape: "#otherBlob", type: "rotational" },
  duration: 1.5
});
```

### Additional Options

```js
gsap.to("#shape", {
  morphSVG: {
    shape: "#target",
    shapeIndex: 2,
    type: "rotational",
    map: "size",        // "size" (default) | "position" | "complexity"
    precision: 2,       // decimal rounding of generated path data
  },
  duration: 1
});
```

- **`map`**: Algorithm priority for automatic point alignment. `"size"` matches by segment size, `"position"` by proximity, `"complexity"` by curvature complexity.
- **`precision`**: Lower = smaller DOM strings; higher = smoother arcs on complex paths.

### `findShapeIndex()` Utility

A standalone development tool that provides an interactive drag UI for previewing `shapeIndex` values before committing to one in code.

```html
<!-- Load in development only — not for production -->
<script src="https://s3-us-west-2.amazonaws.com/s.cdpn.io/16327/findShapeIndex.js"></script>
```

After loading, call:
```js
findShapeIndex("#sourceShape", "#targetShape");
```

A UI overlay renders both shapes and lets you drag to cycle through `shapeIndex` values, previewing the morph live.

### Morphing Non-Path Elements

MorphSVG can convert basic shapes (`<circle>`, `<rect>`, `<ellipse>`, `<polygon>`, `<polyline>`) to path data automatically, so you can morph between them:

```js
// Morph a circle into a rect — MorphSVG converts both to path data
gsap.to("#circle", {
  morphSVG: "#rect",
  duration: 1
});
```

### Bi-directional Morph with Timeline

```js
const tl = gsap.timeline({ repeat: -1, yoyo: true });
tl.to("#blob", { morphSVG: "#targetBlob", duration: 1.5, ease: "power2.inOut" });
```

### Performance Notes

- MorphSVG recalculates path strings on **every frame**, which is CPU-bound. Avoid morphing dozens of complex paths simultaneously.
- Pre-compute paths at load time and cache the `morphSVG` plugin's output where possible.
- For particle-scale morphing (100+ shapes), consider Canvas 2D or WebGL approaches instead.
