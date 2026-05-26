# Core Web Vitals for 3D Sites

## Targets

| Metric | Good | Needs Improvement | Poor |
|--------|------|------------------|------|
| LCP (Largest Contentful Paint) | < 2.5s | 2.5-4s | > 4s |
| INP (Interaction to Next Paint) | < 200ms | 200-500ms | > 500ms |
| CLS (Cumulative Layout Shift) | < 0.1 | 0.1-0.25 | > 0.25 |

## LCP - WebGL Canvas Does NOT Count

The canvas element is not a valid LCP element.
The browser looks for the largest image or text block instead.

Strategy: make your real LCP element an actual HTML element.

```html
<!-- This heading IS a valid LCP element -->
<h1 class="hero-title">Studio Name</h1>

<!-- Canvas is invisible to LCP algorithm -->
<canvas id="webgl"></canvas>
```

```javascript
// Defer WebGL bundle past initial paint
// Load Three.js only when canvas is visible
const observer = new IntersectionObserver((entries) => {
  if (entries[0].isIntersecting) {
    import('./webgl-scene.js').then(({ init }) => init())
    observer.disconnect()
  }
})
observer.observe(document.getElementById('webgl'))
```

LCP target: under 2.5 seconds.
Technique: defer 3D bundle past first paint, make real HTML element the LCP.

## INP - Three.js on Main Thread Will Fail This

INP measures input latency. Three.js render on main thread blocks input handling.

```
Three.js main thread -> INP typically 500-2000ms -> FAIL

Three.js + OffscreenCanvas -> INP typically < 50ms -> PASS
```

Fix: move Three.js to Web Worker via OffscreenCanvas.
Lighthouse 95 -> 100 confirmed by Evil Martians.

```javascript
// Quick INP test in DevTools:
// Performance tab -> Record -> Click rapidly -> Check "Interaction" entries
// > 200ms = failing INP
```

## CLS - Prevent Canvas Layout Shift

Canvas without explicit size causes layout shift when renderer sets dimensions.

```css
/* ALWAYS give canvas explicit dimensions before JS loads */
#webgl-canvas {
  width: 100%;
  height: 100svh;
  display: block;      /* removes inline 4px gap */
  aspect-ratio: auto;  /* let CSS control it */
}
```

```html
<!-- Or use width/height attributes -->
<canvas id="webgl" width="1920" height="1080" style="width:100%;height:100svh"></canvas>
```

CLS target: under 0.1. Canvas without explicit size = immediate CLS fail.

## Lighthouse Scores for 3D Sites - Realistic Expectations

| Optimization Level | Mobile Lighthouse | Desktop Lighthouse |
|-------------------|------------------|-------------------|
| None (Three.js on main thread) | 20-40 | 60-80 |
| Basic (DPR cap, KTX2, deferred load) | 40-60 | 80-90 |
| Advanced (OffscreenCanvas, lazy load) | 70-85 | 90-95 |
| Full (worker rendering, inline LCP) | 90-100 | 95-100 |

Achieving 90+ on mobile for a heavy 3D site requires OffscreenCanvas + deferred loading.
Achieving 100 on mobile is possible (Evil Martians confirmed) but requires worker rendering.

## Core Web Vitals Checklist for 3D Sites

```
LCP:
[ ] Real HTML element (heading/image) is the LCP element - not canvas
[ ] Canvas/Three.js bundle deferred past first paint
[ ] LCP element has explicit width/height (no size shift)
[ ] LCP element preloaded: <link rel="preload">

INP:
[ ] Three.js on Web Worker via OffscreenCanvas (or at minimum: deferred load)
[ ] No synchronous Three.js init on main thread at page load
[ ] Event handlers are lightweight (heavy work in worker)

CLS:
[ ] Canvas has explicit CSS dimensions (100svh, not 0 then resized)
[ ] Fonts loaded before render (font-display: optional or preload)
[ ] No elements inserted above canvas after load
```
