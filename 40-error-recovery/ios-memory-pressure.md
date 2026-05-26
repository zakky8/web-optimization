# iOS Memory Pressure and WebGL Context Management

## iOS-Specific WebGL Behavior

iOS Safari manages GPU memory aggressively compared to desktop browsers. Several behaviors are unique to iOS that require explicit handling:

1. **Context loss on tab background.** iOS Safari destroys the WebGL context when a tab is backgrounded — not just when the tab is inactive, but when the user switches to another app or the screen locks. This is an iOS-level GPU resource reclamation, not a browser-level throttle.
2. **No recovery without `preventDefault()`.** The standard `event.preventDefault()` requirement applies. Without it the context is not restored when the tab is foregrounded.
3. **Texture memory limits.** iOS devices have strict per-tab GPU memory budgets (varies by device generation). Exceeding the limit triggers context loss even while the tab is active.
4. **`requestAnimationFrame` stops in background.** When `document.hidden === true`, iOS suspends `rAF` callbacks entirely. Source: MDN Page Visibility API, accessed 2026-05-26.

---

## `visibilitychange` Event: Pause and Resume

The `visibilitychange` event fires on the `document` when the page transitions between visible and hidden states. Source: MDN `visibilitychange` event, accessed 2026-05-26.

```js
document.addEventListener('visibilitychange', (event) => {
  if (document.hidden) {
    // page is hidden — rAF is suspended on iOS anyway,
    // but explicit cancellation avoids any race on re-show
    pauseRenderLoop();
  } else {
    // page is visible again
    resumeRenderLoop();
  }
});
```

`document.visibilityState` values (MDN Page Visibility API, accessed 2026-05-26):

| Value | Meaning |
|---|---|
| `"visible"` | Page is at least partially visible in a non-minimized window |
| `"hidden"` | Tab is in background, window is minimized, or device screen is off |

`document.hidden` is a boolean shorthand: `true` when `visibilityState === "hidden"`.

---

## Render Loop Pause/Resume Pattern

```js
let animFrameId  = null;
let isRendering  = false;

function startRenderLoop() {
  if (isRendering) return;
  isRendering = true;
  loop();
}

function stopRenderLoop() {
  isRendering = false;
  if (animFrameId !== null) {
    cancelAnimationFrame(animFrameId);
    animFrameId = null;
  }
}

function loop() {
  if (!isRendering) return;
  animFrameId = requestAnimationFrame(loop);
  renderer.render(scene, camera);
}

function pauseRenderLoop() {
  stopRenderLoop();
}

function resumeRenderLoop() {
  // On iOS, the context may have been lost during backgrounding.
  // Check before resuming.
  if (renderer.getContext().isContextLost()) {
    // Context lost — wait for webglcontextrestored to restart
    console.warn('WebGL context was lost while backgrounded. Waiting for restore.');
    return;
  }
  startRenderLoop();
}

// Visibility-based pause/resume
document.addEventListener('visibilitychange', () => {
  if (document.hidden) {
    pauseRenderLoop();
  } else {
    resumeRenderLoop();
  }
});

// Context lifecycle — must be registered before calling getContext()
const canvas = document.getElementById('canvas');

canvas.addEventListener('webglcontextlost', (event) => {
  event.preventDefault(); // Required to allow restoration
  stopRenderLoop();
  console.warn('WebGL context lost (likely iOS background or memory pressure).');
}, false);

canvas.addEventListener('webglcontextrestored', () => {
  reinitializeGPUResources();
  startRenderLoop();
  console.info('WebGL context restored.');
}, false);
```

---

## iOS Context Loss During Tab Background: What Actually Happens

On iOS Safari (all current versions as of 2026-05-26), the sequence when backgrounding is:

1. User switches to another app or the screen locks.
2. iOS fires `visibilitychange` (document becomes hidden).
3. Shortly after (timing varies by device memory state), iOS sends `webglcontextlost` to the canvas.
4. When the user foregrounds the tab again, iOS fires `visibilitychange` (document becomes visible).
5. Then iOS fires `webglcontextrestored` — but only if `preventDefault()` was called in step 3.

The gap between steps 3 and 5 means `resumeRenderLoop()` called in the `visibilitychange` handler may run before `webglcontextrestored`. This is why the `resumeRenderLoop` implementation above checks `isContextLost()` before restarting — if lost, it defers restart to the `webglcontextrestored` handler.

---

## Texture Quality Reduction on Memory Pressure

iOS does not expose a JavaScript `memory pressure` event (unlike native iOS apps which receive `applicationDidReceiveMemoryWarning`). The closest signal available to web code is context loss itself (indicating the GPU ran out of memory) or monitoring texture upload failures.

Proactive mitigation: reduce texture resolution before memory pressure causes a loss.

```js
function getTextureScale() {
  // Heuristic: reduce on low-memory devices or when devicePixelRatio suggests a mid-range device
  const isIOS = /iphone|ipad|ipod/i.test(navigator.userAgent);
  const dpr   = window.devicePixelRatio;

  if (isIOS && dpr <= 2) {
    return 0.5; // Half-resolution textures on older iOS (iPhone 6S–X era)
  }
  return 1.0;
}

function loadTexture(url) {
  const scale   = getTextureScale();
  const loader  = new THREE.TextureLoader();

  return new Promise((resolve, reject) => {
    loader.load(url, (texture) => {
      if (scale < 1.0) {
        // Downscale via canvas
        const img    = texture.image;
        const canvas = document.createElement('canvas');
        canvas.width  = Math.floor(img.width  * scale);
        canvas.height = Math.floor(img.height * scale);
        const ctx     = canvas.getContext('2d');
        ctx.drawImage(img, 0, 0, canvas.width, canvas.height);
        texture.image = canvas;
        texture.needsUpdate = true;
      }
      resolve(texture);
    }, undefined, reject);
  });
}
```

### Renderer pixel ratio capping

Also cap `devicePixelRatio` on iOS to limit framebuffer size:

```js
// Cap DPR at 2.0 — retina is sufficient; 3x does not visibly improve quality
// but triples fill rate and memory usage
renderer.setPixelRatio(Math.min(window.devicePixelRatio, 2));
```

---

## Testing with Safari Web Inspector

Safari Web Inspector on macOS can connect to an iOS device over USB (requires Develop menu enabled in Safari preferences).

### Simulate context loss

```js
// In Safari Web Inspector console, connected to the iOS device:
const canvas = document.querySelector('canvas');
const gl     = canvas.getContext('webgl') || canvas.getContext('webgl2');
const ext    = gl.getExtension('WEBGL_lose_context');
if (ext) {
  ext.loseContext();   // Triggers webglcontextlost
  // Later:
  ext.restoreContext(); // Triggers webglcontextrestored
} else {
  console.warn('WEBGL_lose_context extension not available on this device/browser');
}
```

### Simulate background-to-foreground cycle

1. On the iOS device, press the Home button (or swipe up) while the web page is visible.
2. Wait 2–5 seconds.
3. Return to Safari.
4. Observe whether `webglcontextlost` and `webglcontextrestored` fired in the Web Inspector console.

### Memory timeline

Safari Web Inspector > Memory tab shows the GPU memory footprint. Watch for spikes when loading textures. If total memory usage climbs above ~300–400 MB on an iPhone 12 or older, context loss is likely.

---

## Three.js + iOS: Complete Setup

```js
import * as THREE from 'three';

const canvas   = document.getElementById('canvas');
const renderer = new THREE.WebGLRenderer({ canvas, antialias: false }); // antialias off saves ~30% GPU mem on iOS

// Cap pixel ratio
renderer.setPixelRatio(Math.min(window.devicePixelRatio, 2));

// Small shadow maps — iOS GPU has limited bandwidth
renderer.shadowMap.enabled = true;
renderer.shadowMap.type    = THREE.PCFShadowMap; // not PCFSoft — cheaper

let animFrameId  = null;
let isRendering  = false;

function startLoop() {
  if (isRendering) return;
  isRendering = true;
  (function loop() {
    if (!isRendering) return;
    animFrameId = requestAnimationFrame(loop);
    renderer.render(scene, camera);
  })();
}

function stopLoop() {
  isRendering = false;
  cancelAnimationFrame(animFrameId);
  animFrameId = null;
}

// Visibility API — pause on background
document.addEventListener('visibilitychange', () => {
  if (document.hidden) {
    stopLoop();
  } else if (!renderer.getContext().isContextLost()) {
    startLoop();
    // else: webglcontextrestored handler will restart
  }
});

// Context loss — iOS may send this on background or low-memory condition
canvas.addEventListener('webglcontextlost', (e) => {
  e.preventDefault();
  stopLoop();
}, false);

canvas.addEventListener('webglcontextrestored', () => {
  // Three.js reinitializes GL state internally; mark resources for re-upload
  scene.traverse((obj) => {
    if (obj.isMesh) {
      if (obj.material.map)       obj.material.map.needsUpdate       = true;
      if (obj.material.normalMap) obj.material.normalMap.needsUpdate  = true;
      obj.material.needsUpdate = true;
    }
  });
  startLoop();
}, false);

startLoop();
```

---

## Key Differences: iOS vs Desktop Context Loss

| Behavior | Desktop | iOS Safari |
|---|---|---|
| Context loss trigger | GPU timeout, driver crash, tab inactive in heavy-use scenario | Tab background, screen lock, memory pressure |
| How predictable | Rare in normal use | Happens on every background on older devices |
| `webglcontextrestored` reliability | High | Requires `preventDefault()` — without it, rarely fires |
| Memory limit | System RAM dependent, usually generous | Per-tab budget, strict (128–512 MB GPU depending on device) |
| `WEBGL_lose_context` available | Yes, widely supported | Yes — use for testing |

---

## Sources

| Source | URL | Tier | Accessed |
|---|---|---|---|
| MDN — `visibilitychange` event | https://developer.mozilla.org/en-US/docs/Web/API/Document/visibilitychange_event | 1 | 2026-05-26 |
| MDN — Page Visibility API | https://developer.mozilla.org/en-US/docs/Web/API/Page_Visibility_API | 1 | 2026-05-26 |
| MDN — `webglcontextlost` event | https://developer.mozilla.org/en-US/docs/Web/API/HTMLCanvasElement/webglcontextlost_event | 1 | 2026-05-26 |
| MDN — `webglcontextrestored` event | https://developer.mozilla.org/en-US/docs/Web/API/HTMLCanvasElement/webglcontextrestored_event | 1 | 2026-05-26 |
| MDN — `WEBGL_lose_context` | https://developer.mozilla.org/en-US/docs/Web/API/WEBGL_lose_context | 1 | 2026-05-26 |
| Three.js — `WebGLRenderer.js` (dev) | https://raw.githubusercontent.com/mrdoob/three.js/dev/src/renderers/WebGLRenderer.js | 1 | 2026-05-26 |
