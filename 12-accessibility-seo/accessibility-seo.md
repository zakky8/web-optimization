# Accessibility and SEO

WebGL and 3D content present unique challenges for accessibility and search engine indexing. A 3D scene is invisible to screen readers and search crawlers by default. The goal: make the content accessible and indexable without sacrificing the visual experience.

## The Core Problem

- `<canvas>` elements have no semantic meaning. Screen readers skip them.
- JavaScript-rendered text is increasingly indexed by Google, but complex 3D scenes with no fallback content still score poorly.
- Heavy 3D scenes hurt Core Web Vitals (LCP, INP, CLS), which are ranking signals.

## ARIA for Canvas Content

### Accessible Canvas Wrapper

```html
<!-- The canvas is decorative/supplementary — hide it from AT -->
<canvas
  id="hero-canvas"
  aria-hidden="true"
  focusable="false"
></canvas>

<!-- Provide real semantic content alongside the canvas -->
<section aria-labelledby="hero-heading">
  <h1 id="hero-heading">Product Name</h1>
  <p>A 360-degree interactive view of our flagship product.</p>
  <button
    id="canvas-fallback-btn"
    aria-label="View product images in gallery instead"
  >
    View static gallery
  </button>
</section>
```

### Live Region for Dynamic 3D State

```html
<!-- Announce state changes from the 3D scene to screen readers -->
<div
  id="scene-status"
  role="status"
  aria-live="polite"
  aria-atomic="true"
  class="sr-only"
></div>
```

```js
function updateSceneStatus(message) {
  document.getElementById("scene-status").textContent = message;
}

// Call when meaningful state changes
controls.addEventListener("change", () => {
  updateSceneStatus("Product rotated. Use arrow keys to continue rotating.");
});
```

### Keyboard Navigation for 3D Controls

```js
const canvas = document.querySelector("#hero-canvas");

// Make canvas focusable
canvas.setAttribute("tabindex", "0");
canvas.setAttribute("role", "img");
canvas.setAttribute("aria-label", "Interactive 3D product viewer. Use arrow keys to rotate.");

canvas.addEventListener("keydown", (e) => {
  const speed = 0.05;
  switch (e.key) {
    case "ArrowLeft":  rotateScene(-speed, 0);  e.preventDefault(); break;
    case "ArrowRight": rotateScene( speed, 0);  e.preventDefault(); break;
    case "ArrowUp":    rotateScene(0, -speed);  e.preventDefault(); break;
    case "ArrowDown":  rotateScene(0,  speed);  e.preventDefault(); break;
    case "Home":       resetScene();            e.preventDefault(); break;
  }
});

// Focus ring visible for keyboard users
canvas.addEventListener("focus",  () => canvas.classList.add("canvas-focused"));
canvas.addEventListener("blur",   () => canvas.classList.remove("canvas-focused"));
```

```css
.sr-only {
  position: absolute;
  width: 1px; height: 1px;
  padding: 0; margin: -1px;
  overflow: hidden;
  clip: rect(0, 0, 0, 0);
  white-space: nowrap;
  border: 0;
}

.canvas-focused {
  outline: 3px solid #0066cc;
  outline-offset: 2px;
}
```

## Reduced Motion

Users who have set `prefers-reduced-motion: reduce` should get a static or minimal-animation version of the scene.

```js
const prefersReducedMotion = window.matchMedia("(prefers-reduced-motion: reduce)").matches;

function animate(delta) {
  if (!prefersReducedMotion) {
    mesh.rotation.y += delta * 0.5;
    particles.forEach(p => updateParticle(p, delta));
  }
  renderer.render(scene, camera);
}

// React Three Fiber
import { useReducedMotion } from "framer-motion"; // or detect manually

function RotatingMesh() {
  const reduced = useReducedMotion();
  useFrame((_, delta) => {
    if (!reduced) meshRef.current.rotation.y += delta;
  });
  return <mesh ref={meshRef}>...</mesh>;
}
```

```css
/* CSS fallback for static canvas background */
@media (prefers-reduced-motion: reduce) {
  #hero-canvas { display: none; }
  #hero-static-bg { display: block; }
}
```

## Color Contrast

Even within 3D scenes, UI overlays, labels, and captions must meet WCAG contrast ratios: 4.5:1 for normal text, 3:1 for large text and UI components.

```js
// If you render text via Three.js Text (Troika) or CSS2DRenderer,
// ensure the text color passes contrast checks against its background.

// CSS2DRenderer labels
const label = document.createElement("div");
label.className = "three-label";
label.textContent = "Component Name";
// In CSS: color: #ffffff; background: rgba(0,0,0,0.75);
// White on dark overlay = 12.6:1 — passes AAA
```

## SEO for 3D Scenes

### The Crawlability Problem

Googlebot renders JavaScript but has limited GPU. Three.js scenes are typically empty `<canvas>` elements to crawlers. Solutions:

**Strategy 1: Semantic HTML alongside the canvas**

```html
<!-- Invisible to visual users, indexed by crawlers -->
<div class="sr-only">
  <h1>Nike Air Max 2026 — Interactive 3D Viewer</h1>
  <p>Explore the Nike Air Max 2026 in full 3D. Available in 8 colorways.
     Lightweight mesh upper, 40mm Air unit, carbon fiber plate.</p>
  <ul>
    <li>Weight: 285g</li>
    <li>Stack height: 40mm heel, 32mm foreground</li>
    <li>Available sizes: EU 38–48</li>
  </ul>
</div>
<canvas id="product-canvas" aria-hidden="true"></canvas>
```

**Strategy 2: Structured Data**

```html
<script type="application/ld+json">
{
  "@context": "https://schema.org",
  "@type": "Product",
  "name": "Nike Air Max 2026",
  "description": "Lightweight running shoe with 40mm Air unit",
  "image": [
    "https://example.com/product-front.jpg",
    "https://example.com/product-side.jpg"
  ],
  "brand": { "@type": "Brand", "name": "Nike" },
  "offers": {
    "@type": "Offer",
    "price": "189.99",
    "priceCurrency": "USD",
    "availability": "https://schema.org/InStock"
  }
}
</script>
```

**Strategy 3: Static OG Image + SSR fallback**

```html
<!-- Open Graph — used by social sharing, crawlers -->
<meta property="og:title"       content="Nike Air Max 2026 — 3D Viewer">
<meta property="og:description" content="Explore every angle. Lightweight, fast, precise.">
<meta property="og:image"       content="https://cdn.example.com/og/air-max-2026.jpg">
<meta property="og:type"        content="product">

<!-- Twitter Card -->
<meta name="twitter:card"       content="summary_large_image">
<meta name="twitter:image"      content="https://cdn.example.com/og/air-max-2026.jpg">
```

### Canvas Screenshot for OG Image

Automate OG image generation from your 3D scene using Playwright or Puppeteer in CI:

```js
// scripts/generate-og-images.mjs
import { chromium } from "playwright";

const browser = await chromium.launch();
const page    = await browser.newPage();
await page.setViewportSize({ width: 1200, height: 630 });

await page.goto("http://localhost:3000/product/air-max-2026");
await page.waitForSelector("canvas");
await page.waitForTimeout(2000); // let 3D scene fully render

const canvas = await page.$("canvas");
await canvas.screenshot({ path: "public/og/air-max-2026.jpg", type: "jpeg", quality: 85 });

await browser.close();
console.log("OG image generated");
```

## Core Web Vitals for 3D Sites

### LCP (Largest Contentful Paint) — target < 2.5s

The 3D canvas is typically NOT the LCP element (it's a canvas, not an image/text block). But heavy Three.js scripts delay LCP of the real content.

```html
<!-- Preload critical scripts -->
<link rel="preload" href="/assets/three.min.js"   as="script">
<link rel="preload" href="/assets/hero-model.glb" as="fetch" crossorigin>

<!-- Defer non-critical Three.js -->
<script defer src="/assets/three-scene.js"></script>
```

```js
// Load 3D scene AFTER LCP element is painted
if ("requestIdleCallback" in window) {
  requestIdleCallback(() => initThreeScene(), { timeout: 3000 });
} else {
  setTimeout(() => initThreeScene(), 200);
}
```

### INP (Interaction to Next Paint) — target < 200ms

Pointer events on a Three.js canvas trigger raycasting on the main thread. Move heavy work off:

```js
// Debounce raycasting
let rafId = null;
canvas.addEventListener("pointermove", (e) => {
  if (rafId) return;
  rafId = requestAnimationFrame(() => {
    performRaycast(e);
    rafId = null;
  });
});

// Or use a Web Worker for physics/AI raycasts
```

### CLS (Cumulative Layout Shift) — target < 0.1

Reserve canvas space before Three.js initializes to prevent layout shifts:

```css
/* Reserve aspect ratio before JS loads */
#canvas-container {
  aspect-ratio: 16 / 9;
  width: 100%;
  background: #111; /* placeholder color */
}

#canvas-container canvas {
  display: block;
  width: 100%;
  height: 100%;
}
```

## Real User Monitoring

```js
// web-vitals library (Google's official package)
import { onLCP, onINP, onCLS, onFCP, onTTFB } from "web-vitals";

function sendToAnalytics({ name, value, id, delta, rating }) {
  // rating: "good" | "needs-improvement" | "poor"
  navigator.sendBeacon("/analytics", JSON.stringify({
    metric: name,
    value:  Math.round(name === "CLS" ? delta * 1000 : delta),
    id, rating,
    page: location.pathname,
  }));
}

onLCP(sendToAnalytics);
onINP(sendToAnalytics);
onCLS(sendToAnalytics);
onFCP(sendToAnalytics);
onTTFB(sendToAnalytics);
```

## Performance Implications

- In 2026, Google's March core update strengthened the weight of Core Web Vitals in rankings. 43% of sites fail the 200ms INP threshold — the most commonly failed metric.
- Deferred 3D initialization (after LCP) is the single most impactful change for sites with heavy Three.js hero sections.
- `prefers-reduced-motion` compliance is now a WCAG 2.2 requirement for Level AA. Animated canvas content without a reduced-motion alternative is a legal risk in jurisdictions enforcing WCAG.
- Semantic HTML hidden alongside canvas content costs nothing in performance and significantly improves crawler indexability.

## Tools and Libraries

| Tool | Purpose |
|---|---|
| [web-vitals](https://github.com/GoogleChrome/web-vitals) | Core Web Vitals measurement |
| [axe-core](https://github.com/dequelabs/axe-core) | Automated accessibility testing |
| [WAVE](https://wave.webaim.org) | Manual accessibility evaluation |
| [Lighthouse](https://developer.chrome.com/docs/lighthouse/overview/) | Performance + accessibility audit |
| [PageSpeed Insights](https://pagespeed.web.dev) | Field data CWV dashboard |
| [Schema.org Product](https://schema.org/Product) | Structured data spec |
| [WCAG 2.2](https://www.w3.org/WAI/WCAG22/quickref/) | Accessibility guidelines |
