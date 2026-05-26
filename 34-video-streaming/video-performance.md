# Video Performance in Three.js

**Last verified:** 2026-05-26

---

## requestVideoFrameCallback — Precise Frame Timing

### What It Is

`HTMLVideoElement.requestVideoFrameCallback(callback)` fires the callback exactly when a new decoded video frame is committed to the compositor. This is strictly more precise than `requestAnimationFrame` for video-synchronized work because:

- `requestAnimationFrame` runs at the monitor refresh rate (60 / 90 / 120 Hz)
- A 24 fps video fires callbacks at 24 Hz, not 60 Hz — three.js needlessly re-uploads the same frame up to 5× per second without this API

The method is not yet a W3C standard (still a WICG draft as of 2026-05-26) but is Baseline since October 2024.

### Browser Support (2026-05-26)

Data from caniuse.com — 94.74% global coverage.

| Browser | Minimum version |
|---|---|
| Chrome | 83 |
| Edge | 83 |
| Firefox | 132 |
| Safari | 15.4 |
| Safari iOS | 15.4 |
| Samsung Internet | 13.0 |
| Opera | 69 |
| Chrome for Android | 148 |
| Firefox for Android | 150 |

Source: https://caniuse.com/mdn-api_htmlvideoelement_requestvideoframecallback

### Callback Signature and Metadata

```js
video.requestVideoFrameCallback((now, metadata) => {
  // now             — DOMHighResTimeStamp, same origin as performance.now()
  // metadata.presentationTime   — when frame was committed to compositor
  // metadata.expectedDisplayTime — when frame is expected to be shown
  // metadata.mediaTime          — position in the video timeline (seconds)
  // metadata.presentedFrames    — monotonically increasing counter; gaps = dropped frames
  // metadata.width              — decoded frame width in pixels
  // metadata.height             — decoded frame height in pixels
  // metadata.processingDuration — decode latency
});
```

`presentedFrames` is the dropped-frame detector. If consecutive callbacks show a gap greater than 1 (e.g., jumps from 42 to 44), a frame was dropped somewhere between decode and compositing.

### Using It with Three.js VideoTexture

Three.js natively uses `requestVideoFrameCallback` when available (since r148). No action is needed in the default setup. But if you are managing updates manually or need the metadata:

```js
function onVideoFrame(now, metadata) {
  // Force texture update exactly on this new frame
  texture.needsUpdate = true;

  // Optional: log frame-accurate media time for sync
  console.log('video frame at media time:', metadata.mediaTime.toFixed(3), 's');

  // Re-register for the next frame
  video.requestVideoFrameCallback(onVideoFrame);
}

video.requestVideoFrameCallback(onVideoFrame);
```

**Without this API** (fallback): Three.js falls back to checking `video.readyState` in the renderer's render loop, which uploads on every rAF tick regardless of whether the frame is new.

---

## Video Preloading Strategies

### The preload Attribute

The HTML `preload` attribute is a hint to the browser — it may be overridden by browser heuristics, data-saver mode, or battery status.

| Value | Behavior | When to use |
|---|---|---|
| `none` | Nothing downloaded until `play()` is called | Videos unlikely to be played; mobile data budgets |
| `metadata` | Fetches duration, dimensions, possibly first frame; typically up to ~3% of file | Default — most Three.js texture uses |
| `auto` | Browser downloads entire file | Only when playback is near-certain; background loops |

```html
<!-- For a video used as a guaranteed Three.js background texture -->
<video preload="auto" autoplay muted loop playsinline src="bg.mp4"></video>

<!-- For a conditionally loaded texture -->
<video preload="metadata" muted playsinline src="hover-effect.mp4"></video>
```

`preload` is ignored when `autoplay` is set on many browsers, because autoplay implies the video will play and the browser downloads accordingly.

### Programmatic Preloading

For Three.js scenes that load video assets in a loading manager:

```js
function preloadVideo(src) {
  return new Promise((resolve, reject) => {
    const video = document.createElement('video');
    video.preload  = 'auto';
    video.muted    = true;
    video.playsInline = true;
    video.crossOrigin = 'anonymous';

    video.addEventListener('canplaythrough', () => resolve(video), { once: true });
    video.addEventListener('error', reject, { once: true });

    video.src = src;
    video.load();
  });
}

// Usage
const video = await preloadVideo('/assets/bg-loop.mp4');
const texture = new THREE.VideoTexture(video);
```

`canplaythrough` fires when the browser estimates it can play to the end without buffering. This is the right gate for starting a background that must play smoothly.

### Lazy Loading

For textures on meshes that are off-camera on load:

```js
// IntersectionObserver pattern — load only when mesh enters camera frustum
// (use a proxy DOM element positioned at the mesh's projected screen position)
const observer = new IntersectionObserver((entries) => {
  entries.forEach(entry => {
    if (entry.isIntersecting) {
      video.src = '/assets/section2-bg.mp4';
      video.load();
      observer.disconnect();
    }
  });
});
observer.observe(proxyElement);
```

---

## Video Sprite Sheets vs. Actual Video

### Video Sprite Sheets

A video sprite sheet (or flipbook texture) is a single texture atlas where each frame of an animation is a tile in a grid. Instead of uploading a new texture per frame, you upload once and advance a UV offset.

```
Frame 0: UV (0.0, 0.0) → (0.25, 0.25)
Frame 1: UV (0.25, 0.0) → (0.50, 0.25)
...
```

**When to use sprite sheets:**

| Scenario | Sprite sheet | Actual video |
|---|---|---|
| Short loop (< 2 s), < 30 frames | Preferred | Overkill |
| Needs to run at a specific frame rate not matching video decode | Preferred | Harder to control |
| Must work on platforms without video decode (e.g. headless canvas) | Preferred | Not available |
| Long loop (> 10 s) | Impractical (atlas too large) | Preferred |
| Needs adaptive bitrate | No | Yes (HLS.js) |
| Needs audio | No | Yes |

**Sprite sheet GPU math:**  
A 10-frame sprite sheet of 512×512 frames arranged in a 4×4 grid = 2048×2048 texture = 16 MB VRAM (RGBA8). Uploaded once. Advancing frames costs zero GPU bandwidth.

**Actual video GPU math:**  
A 512×512 video frame = 1 MB. At 30 fps = 30 MB/s of GPU texture upload bandwidth consumed continuously.

### Sprite Sheet Shader Pattern

```glsl
// Vertex or fragment: advance UV by frame index
uniform float  frameIndex;  // 0, 1, 2, ...
uniform vec2   gridSize;    // e.g. vec2(4.0, 4.0) for a 4×4 grid

varying vec2 vUv;

void main() {
  float col     = mod(frameIndex, gridSize.x);
  float row     = floor(frameIndex / gridSize.x);
  vec2 frameUv  = (vUv + vec2(col, row)) / gridSize;
  gl_FragColor  = texture2D(spriteSheet, frameUv);
}
```

Drive `frameIndex` from a clock in the JavaScript render loop:

```js
const fps      = 24;
const numFrames = 16;

function animate(time) {
  requestAnimationFrame(animate);
  const frame = Math.floor((time / 1000) * fps) % numFrames;
  material.uniforms.frameIndex.value = frame;
  renderer.render(scene, camera);
}
```

---

## Memory Considerations for Multiple Video Textures

### GPU Texture Memory

Each `THREE.VideoTexture` allocates a GPU texture the size of the video's decoded resolution. At RGBA8:

| Resolution | VRAM per frame |
|---|---|
| 640×360 | ~0.9 MB |
| 1280×720 | ~3.7 MB |
| 1920×1080 | ~8.3 MB |
| 3840×2160 | ~33 MB |

**Each active VideoTexture uploads its frame every decode interval to the GPU.** With 4 simultaneous 1080p VideoTextures, you are pushing ~33 MB/s of texture data continuously, on top of whatever the scene is drawing.

### Practical Limits

- **Mobile (integrated GPU shared RAM):** 2–3 simultaneous 720p textures is a reasonable ceiling
- **Desktop discrete GPU:** 6–8 simultaneous 1080p textures is generally fine; monitor GPU memory usage with `renderer.info.memory.textures`
- **WebXR / VR:** Halve mobile limits; each eye renders independently

### Strategies to Reduce Memory Pressure

**1. Pause videos not in the camera frustum**

```js
function updateVisibility(meshes, camera) {
  const frustum = new THREE.Frustum();
  frustum.setFromProjectionMatrix(
    new THREE.Matrix4().multiplyMatrices(
      camera.projectionMatrix, camera.matrixWorldInverse
    )
  );
  meshes.forEach(({ mesh, video, texture }) => {
    const visible = frustum.intersectsObject(mesh);
    if (!visible && !video.paused) {
      video.pause();           // stops decode; texture upload stops
    } else if (visible && video.paused) {
      video.play();
    }
  });
}
```

A paused VideoTexture still holds GPU memory but stops uploading new data — so VRAM is occupied but bandwidth pressure drops to zero.

**2. Use lower-resolution sources for off-axis meshes**

Load a 360p stream for a video mesh that is 200 CSS pixels wide. The eye cannot see the quality difference at that size, but the texture upload cost drops ~22× vs 1080p.

**3. Dispose textures when off-screen**

```js
function disposeVideoTexture(texture, video) {
  texture.dispose();
  video.pause();
  video.src = '';
  video.load();
}
```

`texture.dispose()` immediately releases the GPU allocation. You can recreate the texture and reload the video when the object comes back into view.

**4. Limit simultaneous decode with a pool**

```js
const MAX_ACTIVE = 3;
const pool = [];  // { video, texture, mesh, priority }

function tick() {
  // Sort by screen-space area (larger = higher priority)
  pool.sort((a, b) => screenArea(b.mesh) - screenArea(a.mesh));
  pool.forEach((item, i) => {
    if (i < MAX_ACTIVE && item.video.paused) item.video.play();
    if (i >= MAX_ACTIVE && !item.video.paused) item.video.pause();
  });
}
```

**5. Consider sprite sheets for short loops** (see above)

For 0–3 second loops used as particle effects or UI elements, a sprite sheet eliminates continuous decode entirely.

---

## Monitoring Performance

```js
// Three.js renderer stats
console.log(renderer.info.memory.textures);   // count of GPU textures
console.log(renderer.info.render.calls);       // draw calls per frame

// Video element decode health
video.requestVideoFrameCallback((now, metadata) => {
  if (metadata.presentedFrames > prevFrames + 1) {
    console.warn('Dropped frame detected');
  }
  prevFrames = metadata.presentedFrames;
  video.requestVideoFrameCallback(arguments.callee);
});
```

---

## Sources

- https://developer.mozilla.org/en-US/docs/Web/API/HTMLVideoElement/requestVideoFrameCallback — accessed 2026-05-26
- https://caniuse.com/mdn-api_htmlvideoelement_requestvideoframecallback — 94.74% global, Baseline October 2024
- https://web.dev/articles/requestvideoframecallback-rvfc — frame metadata documentation
- https://web.dev/fast-playback-with-preload/ — preload strategies
- https://web.dev/learn/performance/video-performance — video performance guide
- https://github.com/mrdoob/three.js/issues/28980 — VideoFrame + CanvasTexture/VideoTexture performance discussion
