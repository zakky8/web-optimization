# Graceful Degradation: WebGL Not Available

## When WebGL Is Absent

WebGL is unavailable in several real scenarios:
- Old Android WebViews with WebGL disabled by policy
- Corporate desktops with locked-down GPU drivers
- Browsers in privacy/hardened mode with canvas fingerprinting blocked
- Headless environments (some SSR hydration paths)
- Users explicitly blocking hardware acceleration

Failing silently produces a blank `<canvas>` with no error feedback. The patterns below ensure users get something useful.

---

## Feature Detection

### Canonical Detection Pattern

Create a canvas off-DOM and attempt `getContext`. The return value is `null` if WebGL is unsupported; it is an instance of `WebGLRenderingContext` when available. Source: MDN "Detect WebGL" example, accessed 2026-05-26.

```js
function isWebGLAvailable() {
  try {
    const canvas = document.createElement('canvas');
    return !!(
      window.WebGLRenderingContext &&
      (canvas.getContext('webgl') || canvas.getContext('experimental-webgl'))
    );
  } catch {
    return false;
  }
}

function isWebGL2Available() {
  try {
    const canvas = document.createElement('canvas');
    return !!(
      window.WebGL2RenderingContext &&
      canvas.getContext('webgl2')
    );
  } catch {
    return false;
  }
}
```

The `try/catch` matters: some browser extensions and privacy tools throw on `getContext` instead of returning `null`.

### Three.js `WebGL.isWebGLAvailable()`

Three.js ships `three/addons/capabilities/WebGL.js` (formerly `THREE.WebGL`). It wraps the same detection logic:

```js
import WebGL from 'three/addons/capabilities/WebGL.js';

if (!WebGL.isWebGLAvailable()) {
  const warning = WebGL.getWebGLErrorMessage();
  document.body.appendChild(warning);
}
```

`getWebGLErrorMessage()` returns a pre-built `<div>` with a human-readable message and a link to https://get.webgl.org. Do not use this string verbatim in production — it is styled for Three.js demo pages.

---

## `<canvas>` Fallback Content

HTML inside `<canvas>` is rendered only when the element itself cannot be displayed or when the browser does not support canvas at all. It does **not** display when WebGL `getContext` fails — the canvas element exists, only the context is missing.

```html
<canvas id="scene" width="800" height="600">
  <!-- This renders only if the browser has no canvas support at all -->
  <p>Your browser does not support the HTML canvas element.
     Please upgrade to a modern browser.</p>
</canvas>
```

For the WebGL failure case, you must use JavaScript to show a fallback:

```js
const container = document.getElementById('app');

if (!isWebGLAvailable()) {
  container.innerHTML = `
    <div class="no-webgl-fallback" role="alert">
      <p>3D rendering is not available in this browser.</p>
      <p>Try Chrome, Firefox, or Edge with hardware acceleration enabled.</p>
    </div>
  `;
} else {
  initThreeJsScene(container);
}
```

---

## CSS-Only Animation Fallback

When WebGL is unavailable, replace animated 3D with a CSS animation that communicates the same visual intent. Honor `prefers-reduced-motion` at the same time.

```css
/* Fallback: shown only when .no-webgl class is on the container */
.no-webgl .hero-fallback {
  display: block;
  background: linear-gradient(135deg, #1a1a2e 0%, #16213e 50%, #0f3460 100%);
  animation: hero-pulse 4s ease-in-out infinite;
}

/* No animation for users who prefer reduced motion */
@media (prefers-reduced-motion: reduce) {
  .no-webgl .hero-fallback {
    animation: none;
  }
}

@keyframes hero-pulse {
  0%, 100% { opacity: 1; }
  50%       { opacity: 0.7; }
}
```

The `prefers-reduced-motion: reduce` value is set by the user at the OS level (macOS: System Settings > Accessibility > Display > Reduce motion; iOS: Settings > Accessibility > Motion). Source: MDN `prefers-reduced-motion`, accessed 2026-05-26.

### JavaScript detection of `prefers-reduced-motion`

```js
const prefersReducedMotion = window.matchMedia('(prefers-reduced-motion: reduce)').matches;

if (isWebGLAvailable() && !prefersReducedMotion) {
  initHeavyThreeJsScene();
} else if (isWebGLAvailable()) {
  // WebGL available but user prefers less motion — use a minimal, static scene
  initStaticThreeJsScene();
} else {
  // No WebGL — CSS fallback only
  showCssFallback();
}
```

Listen for runtime changes (user can toggle accessibility settings without reloading):

```js
window.matchMedia('(prefers-reduced-motion: reduce)')
  .addEventListener('change', (e) => {
    if (e.matches) {
      pauseAnimations();
    } else {
      resumeAnimations();
    }
  });
```

Source: MDN `prefers-reduced-motion`, accessed 2026-05-26.

---

## R3F: `fallback` Prop on `<Canvas>`

React Three Fiber's `<Canvas>` accepts a `fallback` prop for when WebGL is not available. Source: R3F Canvas API docs, accessed 2026-05-26.

```jsx
import { Canvas } from '@react-three/fiber';

function App() {
  return (
    <Canvas
      fallback={
        <div className="no-webgl-fallback">
          <p>WebGL is not supported in this browser.</p>
        </div>
      }
    >
      <Scene />
    </Canvas>
  );
}
```

---

## No-WebGL User Experience Guidelines

1. **Be explicit.** Tell the user what is unavailable and why, not just that "something went wrong."
2. **Offer a path forward.** Link to https://get.webgl.org or https://browsehappy.com with instructions.
3. **Preserve content.** If the WebGL scene is decorative, hide it gracefully and show the underlying text/image content.
4. **Do not break the page.** A missing WebGL context must never cause an unhandled exception that breaks navigation or form submission on the same page.
5. **Respect reduced motion.** Any CSS fallback animation must disable itself under `prefers-reduced-motion: reduce`.
6. **Test without GPU.** Run Chrome with `--disable-gpu` flag or use `--use-gl=swiftshader` to simulate software-only rendering during CI.

```bash
# Test WebGL detection in headless Chrome
google-chrome --headless --disable-gpu --dump-dom https://localhost:3000
```

---

## Putting It Together: Detection + Fallback + Reduced Motion

```js
const container = document.querySelector('#canvas-container');

const webglOk    = isWebGLAvailable();
const prefersMin = window.matchMedia('(prefers-reduced-motion: reduce)').matches;

if (!webglOk) {
  container.classList.add('no-webgl');
  // Mount CSS-only static fallback
  container.insertAdjacentHTML('afterbegin', '<div class="hero-fallback"></div>');
} else if (prefersMin) {
  // WebGL available but animate minimally
  initMinimalScene(container);
} else {
  initFullScene(container);
}
```

---

## Sources

| Source | URL | Tier | Accessed |
|---|---|---|---|
| MDN — Detect WebGL | https://developer.mozilla.org/en-US/docs/Web/API/WebGL_API/By_example/Detect_WebGL | 1 | 2026-05-26 |
| MDN — `prefers-reduced-motion` | https://developer.mozilla.org/en-US/docs/Web/CSS/@media/prefers-reduced-motion | 1 | 2026-05-26 |
| R3F Canvas API | https://r3f.docs.pmnd.rs/api/canvas | 1 | 2026-05-26 |
