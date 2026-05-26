# GSAP with SVG

> Sources fetched 2026-05-26. GSAP is now fully free (all plugins) following Webflow's acquisition — no Club membership required.
> Primary source: https://gsap.com/docs/v3/ and plugin-specific pages.

---

## Animating SVG Attributes

GSAP's core can tween any animatable property, including SVG presentation attributes. Attributes are specified inside an `attr:{}` object to distinguish them from CSS properties.

```js
// Animate cx, cy, r on a <circle>
gsap.to("#myCircle", {
  duration: 1,
  attr: { cx: 200, cy: 150, r: 40 }
});

// Animate points on a <polygon>
gsap.to("#myPoly", {
  duration: 1,
  attr: { points: "100,10 40,198 190,78 10,78 160,198" }
});
```

**`d` attribute (path morphing without MorphSVG):** The `d` attribute is animatable via CSS as of Baseline 2015. GSAP can tween it via `attr: { d: "M..." }` when start and end paths share the same command structure and point count. For arbitrary topology morphing, use MorphSVGPlugin instead.

**CSS property `d`:** The SVG `d` attribute is also exposed as a CSS property in modern browsers, meaning it can technically be targeted by CSS `@keyframes`. However, paths must have identical command sequences for CSS interpolation to work.

**Supported elements and attributes:**

| Element | Common animatable attrs |
|---------|------------------------|
| `<circle>` | `cx`, `cy`, `r` |
| `<ellipse>` | `cx`, `cy`, `rx`, `ry` |
| `<rect>` | `x`, `y`, `width`, `height`, `rx`, `ry` |
| `<line>` | `x1`, `y1`, `x2`, `y2` |
| `<polyline>`, `<polygon>` | `points` |
| `<path>` | `d` |

---

## SVG Coordinate System Quirks

SVG has its own coordinate system that diverges from the HTML/CSS box model in ways that cause subtle bugs.

### Viewport vs. viewBox

- The **viewport** is the actual pixel size (`width`/`height` attributes on `<svg>`).
- The **viewBox** maps an internal coordinate space onto the viewport. Example: `viewBox="0 0 100 100"` on a `500px × 500px` SVG means 1 user unit = 5px.
- Transforms computed against SVG user units will not match pixel offsets unless viewBox is accounted for.

### The `transform` attribute vs. CSS `transform`

- SVG historically used its own `transform` attribute (`translate()`, `rotate()`, `scale()`, `matrix()`), not CSS transforms.
- Modern browsers support CSS `transform` on SVG elements, but behaviour (especially `transform-origin`) was inconsistent until recently.
- GSAP normalises this by using its own matrix-based transform pipeline, writing back a single `transform` attribute rather than relying on browser CSS.

### `transform-box` — the critical fix

```css
/* Without this, transform-origin is relative to the SVG viewport, not the element */
svg * {
  transform-box: fill-box;
  transform-origin: center;
}
```

`transform-box: fill-box` makes `transform-origin` relative to the element's own bounding box (like HTML elements). Without it, `transform-origin: 50% 50%` refers to the entire SVG viewport, causing unexpected rotation/scale pivot points.

GSAP applies an equivalent correction internally when you set `transformOrigin` through it.

---

## `transformOrigin` in SVG with GSAP

### The problem

In SVG, the default `transform-origin` is `0 0` (top-left of the viewport), not the element centre. This means `gsap.to(el, { rotation: 45 })` rotates around the viewport origin, not the shape's centre, unless corrected.

### GSAP's `transformOrigin`

Pass `transformOrigin` as a string in **SVG user-unit coordinates** or as a percentage:

```js
// Rotate around the element's own centre — percentage works with GSAP's normalisation
gsap.to("#star", {
  rotation: 360,
  transformOrigin: "50% 50%",
  duration: 2,
  repeat: -1,
  ease: "none"
});

// Absolute SVG user-unit coordinates
gsap.to("#star", {
  rotation: 90,
  transformOrigin: "100 150",   // pivot at SVG point (100, 150)
  duration: 1
});
```

### `svgOrigin`

GSAP also exposes `svgOrigin`, which sets the pivot point in the **SVG global coordinate system** regardless of where the element sits in the DOM tree:

```js
gsap.to("#planet", {
  rotation: 360,
  svgOrigin: "250 250",  // rotate around SVG point (250, 250) — e.g. a sun
  duration: 4,
  repeat: -1,
  ease: "none"
});
```

This is essential for orbital animations where child elements must revolve around a shared centre point.

---

## `strokeDasharray` / `strokeDashoffset` Animation

The classic "draw on" SVG stroke effect works by manipulating two CSS properties:

- **`stroke-dasharray`**: defines a dash pattern. Setting it to the path's total length creates one solid dash covering the whole path.
- **`stroke-dashoffset`**: shifts the dash pattern along the path. Animating from `totalLength` to `0` reveals the stroke from start to end.

### Manual implementation

```js
const path = document.querySelector("#myPath");
const length = path.getTotalLength();

// Initialise
gsap.set(path, {
  strokeDasharray: length,
  strokeDashoffset: length
});

// Animate
gsap.to(path, {
  strokeDashoffset: 0,
  duration: 2,
  ease: "power2.inOut"
});
```

`getTotalLength()` is a native SVG DOM method available on `SVGGeometryElement` (path, line, polyline, polygon, circle, ellipse, rect).

### Reverse draw (erase effect)

```js
gsap.from(path, {
  strokeDashoffset: -length,  // negative offset draws from end to start
  duration: 2
});
```

### DrawSVGPlugin (preferred)

For production use, GSAP's DrawSVGPlugin (see `drawsvg-morphsvg.md`) wraps this pattern with a cleaner API, handles Firefox's path-length miscalculation, and supports animating a visible window (e.g. a segment travelling along the stroke rather than just revealing from the start).

---

## SVGPlugin — Status

There is no separate "SVGPlugin" in GSAP v3. SVG attribute animation is handled by the GSAP core via the `attr:{}` shorthand. The SVG-specific plugins are:

- **DrawSVGPlugin** — stroke drawing
- **MorphSVGPlugin** — path morphing
- **MotionPathPlugin** — element motion along a path

All three are free as of 2026. See `drawsvg-morphsvg.md` for DrawSVG and MorphSVG details.

---

## Performance Notes

- Prefer CSS `transform` (via GSAP) over animating positional attributes (`x`, `y`, `cx`, `cy`) — transform changes skip layout.
- `will-change: transform` can help, but use sparingly; it promotes the element to its own compositor layer.
- SVG filters (blur, displacement) are expensive. Do not apply them to rapidly animating elements without testing on low-end devices.
- Complex `d` attribute tweens (many path commands) are CPU-heavy; consider Canvas or WebGL for particle-scale animations.
