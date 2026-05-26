# CSS SVG Animation

> Sources fetched 2026-05-26.
> Primary sources:
> - https://developer.mozilla.org/en-US/docs/Web/CSS/animation (Baseline widely available since 2015)
> - https://developer.mozilla.org/en-US/docs/Web/SVG/SVG_animation_with_SMIL
> - https://caniuse.com/svg-smil (global usage: 96%)
> - https://developer.mozilla.org/en-US/docs/Web/SVG/Reference/Attribute/d

---

## SMIL vs CSS — Current Status (2026)

### SMIL — Not Deprecated, Still Supported

SMIL (Synchronized Multimedia Integration Language) SVG animations (`<animate>`, `<animateTransform>`, `<animateMotion>`) were **proposed for deprecation in Chrome circa 2015** and Chrome briefly showed deprecation warnings. That intent was reversed. As of 2026:

- **Global browser support: 96%** (caniuse.com, fetched 2026-05-26)
- Chrome, Firefox, Safari, and modern Edge all support SMIL.
- The only notable non-supporter is Internet Explorer (all versions).

**Verdict:** SMIL is alive and usable. It is not a safe choice for IE-targeting projects, but IE is effectively extinct. The "SMIL is deprecated" claim that circulates in tutorials is stale — it was a 2015 proposal that was abandoned.

### Why CSS (or JS) is Still Preferred Over SMIL

Despite SMIL not being removed, the industry consensus favours CSS animations and the Web Animations API because:

1. **SMIL cannot animate CSS properties** — only SVG presentation attributes.
2. **No JavaScript integration** — you cannot control SMIL from JS (pause, seek, respond to events easily).
3. **Limited easing options** — SMIL supports only `linear`, `ease`, `ease-in`, `ease-out`, `ease-in-out`, and `cubic-bezier()` via `calcMode`.
4. **Tooling and debugging** — browser DevTools have mature CSS animation inspection; SMIL has none.
5. **No `@keyframes` composition** — you cannot reuse SMIL timing across elements cleanly.

SMIL's niche where it is still genuinely useful: inline SVG icons with self-contained animation that must work without any JavaScript.

---

## CSS Animations on SVG Elements

### What CSS Can and Cannot Animate on SVG

CSS animations work on **CSS properties** applied to SVG elements. They do **not** directly animate SVG presentation attributes (e.g., `cx`, `cy`, `r`, `d`) unless those attributes are also exposed as CSS properties.

| Property | CSS-animatable | Notes |
|----------|----------------|-------|
| `transform` | Yes | Requires `transform-box: fill-box` for correct origin |
| `opacity` | Yes | |
| `fill` | Yes | |
| `stroke` | Yes | |
| `stroke-width` | Yes | |
| `stroke-dasharray` | Yes | Key for draw-on effects |
| `stroke-dashoffset` | Yes | Key for draw-on effects |
| `d` | Yes (modern browsers) | Baseline widely available since 2015; paths must have same command structure |
| `cx`, `cy`, `r` | No | Must use SMIL or JS |
| `points` | No | Must use SMIL or JS |

### `transform-box: fill-box` — Critical Rule

```css
svg * {
  transform-box: fill-box;
  transform-origin: center;
}
```

Without `transform-box: fill-box`, `transform-origin` is calculated relative to the SVG viewport's bounding box, not the individual element. This causes rotations and scales to pivot at an unexpected point.

### Basic CSS Animation on SVG

```css
@keyframes spin {
  from { transform: rotate(0deg); }
  to   { transform: rotate(360deg); }
}

#gear {
  transform-box: fill-box;
  transform-origin: center;
  animation: spin 3s linear infinite;
}
```

```css
@keyframes pulse {
  0%, 100% { opacity: 1; transform: scale(1); }
  50%       { opacity: 0.5; transform: scale(0.9); }
}

#indicator {
  transform-box: fill-box;
  transform-origin: center;
  animation: pulse 1.5s ease-in-out infinite;
}
```

---

## The `stroke-dasharray` / `stroke-dashoffset` Draw-On Trick

This is the canonical CSS technique for making SVG paths appear to draw themselves.

### How It Works

1. `stroke-dasharray` sets a repeating dash pattern. If you set it to the path's total length, you get one single dash equal to the full path.
2. `stroke-dashoffset` shifts that dash along the path. At `offset = length`, the dash is fully shifted off the visible area (invisible). At `offset = 0`, the dash sits exactly over the path (fully visible).
3. Animating `stroke-dashoffset` from `length` to `0` progressively reveals the stroke.

### Step 1 — Measure the path

```js
const path = document.querySelector("#myPath");
const len = path.getTotalLength();
console.log(len); // e.g. 842
```

### Step 2 — Set initial state in CSS

```css
#myPath {
  stroke-dasharray: 842;
  stroke-dashoffset: 842;
  animation: draw 2s ease-in-out forwards;
}
```

### Step 3 — Define `@keyframes`

```css
@keyframes draw {
  to {
    stroke-dashoffset: 0;
  }
}
```

### Hardcode vs. Custom Property

Because `getTotalLength()` must be called in JS, the cleanest approach uses CSS custom properties set from JS:

```js
const path = document.querySelector("#myPath");
const len = path.getTotalLength();
path.style.setProperty("--path-length", len);
```

```css
#myPath {
  stroke-dasharray: var(--path-length);
  stroke-dashoffset: var(--path-length);
  animation: draw 2s ease-in-out forwards;
}

@keyframes draw {
  to { stroke-dashoffset: 0; }
}
```

### Scroll-driven draw-on (CSS Scroll Timeline — 2024+)

```css
@keyframes draw {
  to { stroke-dashoffset: 0; }
}

#myPath {
  stroke-dasharray: var(--path-length);
  stroke-dashoffset: var(--path-length);
  animation: draw linear;
  animation-timeline: scroll();
  animation-range: entry 0% entry 100%;
}
```

---

## Animating the `d` Attribute via CSS

The SVG `d` attribute is a CSS property in modern browsers (Baseline widely available since July 2015). This enables CSS transitions and animations on path shapes — but with a hard constraint: **start and end paths must have identical path command sequences** (same number and type of commands). If they differ, interpolation silently snaps.

```css
#morph {
  transition: d 0.4s ease;
}

#morph:hover {
  d: path("M 10,80 Q 95,10 180,80");
}
```

```css
@keyframes morphHeart {
  0%   { d: path("M 10,30 A 20,20 0,0,1 50,30 A 20,20 0,0,1 90,30 Q 90,60 50,90 Q 10,60 10,30 Z"); }
  100% { d: path("M 50,10 L 90,50 L 50,90 L 10,50 Z"); }
  /* This will snap, not interpolate, because command counts differ */
}
```

For reliable cross-shape morphing, use MorphSVGPlugin or anime.js `morphTo`.

---

## SMIL Quick Reference (for legacy or no-JS contexts)

```xml
<!-- Animate cx attribute on a circle -->
<circle cx="50" cy="50" r="20" fill="red">
  <animate attributeName="cx" from="50" to="200" dur="2s" repeatCount="indefinite" />
</circle>

<!-- Animate rotation transform -->
<rect x="10" y="10" width="40" height="40" fill="blue">
  <animateTransform attributeName="transform"
    type="rotate"
    from="0 30 30"
    to="360 30 30"
    dur="3s"
    repeatCount="indefinite" />
</rect>

<!-- Animate motion along a path -->
<circle r="5" fill="green">
  <animateMotion path="M10,50 Q50,0 90,50" dur="2s" repeatCount="indefinite" />
</circle>
```

SMIL `calcMode` easing values: `linear` | `discrete` | `paced` | `spline` (with `keySplines` for custom cubic-bezier).

---

## Browser Compatibility Summary

| Feature | Chrome | Firefox | Safari | Notes |
|---------|--------|---------|--------|-------|
| CSS `animation` on SVG elements | Full | Full | Full | Baseline 2015 |
| `transform-box: fill-box` | 64+ | 55+ | 9.1+ | Required for correct origin |
| CSS `d` property animation | Full | Full | Full | Baseline 2015 |
| SMIL (`<animate>`, etc.) | Full | Full | Full | 96% global; no IE |
| CSS scroll-driven animations | 115+ | 110+ (partial) | Partial | Check caniuse for current state |
