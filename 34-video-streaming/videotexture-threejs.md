# THREE.VideoTexture — Setup and Behavior

**Last verified:** 2026-05-26  
**Three.js docs:** https://threejs.org/docs/pages/VideoTexture.html

---

## What VideoTexture Is

`THREE.VideoTexture` is a subclass of `THREE.Texture` that wraps an `HTMLVideoElement` as a GPU texture. Instead of uploading a static image once, it interrogates the video element's readiness every frame and, when a new frame is available, marks the texture dirty so the renderer re-uploads it. The class overrides `generateMipmaps` to `false` by default because video dimensions change between streams and mipmap chains are prohibitively expensive to rebuild every frame.

---

## Constructor

```js
const texture = new THREE.VideoTexture(
  video,                          // required: HTMLVideoElement
  mapping,                        // default: THREE.UVMapping
  wrapS,                          // default: THREE.ClampToEdgeWrapping
  wrapT,                          // default: THREE.ClampToEdgeWrapping
  magFilter,                      // default: THREE.LinearFilter
  minFilter,                      // default: THREE.LinearFilter
  format,                         // default: THREE.RGBAFormat
  type,                           // default: THREE.UnsignedByteType
  anisotropy                      // default: Texture.DEFAULT_ANISOTROPY
);
```

Most callers use only the first argument. The rest are relevant when you need NearestFilter pixel-perfect scaling or a specific texture format (see format section below).

---

## The needsUpdate Pattern

`THREE.VideoTexture.update()` is called automatically by the renderer each frame. It checks `video.readyState >= video.HAVE_CURRENT_DATA` (readyState >= 2) before setting `texture.needsUpdate = true`. When `needsUpdate` is `true`, the renderer calls `texImage2D` / `texSubImage2D` to push the new frame to the GPU, then resets the flag to `false`.

**Practical consequence:** if the video is paused, `readyState` stays at `HAVE_CURRENT_DATA` and `needsUpdate` is still set every frame — causing a redundant upload of the same pixels. This is a known inefficiency tracked in three.js issues [#16946](https://github.com/mrdoob/three.js/issues/16946) and [#22380](https://github.com/mrdoob/three.js/issues/22380). On browsers that support `requestVideoFrameCallback`, three.js uses it instead to fire updates only on genuine new frames, eliminating this waste.

**If you need manual control**, disable auto-updates and drive it yourself:

```js
texture.needsUpdate = false; // prevent the auto loop

function tick() {
  if (video.readyState >= video.HAVE_CURRENT_DATA) {
    texture.needsUpdate = true;
  }
  renderer.render(scene, camera);
  requestAnimationFrame(tick);
}
```

---

## Autoplay + Muted Requirement

All major browsers enforce an **Autoplay Policy**: a page may not start video playback with audio before the user has interacted. The rule is simple — `video.autoplay` only fires silently when the element is muted.

```html
<!-- Required attributes for autoplay without user gesture -->
<video
  id="vid"
  src="background.mp4"
  autoplay
  muted
  loop
  playsinline
  crossorigin="anonymous"
></video>
```

```js
const video = document.getElementById('vid');
const texture = new THREE.VideoTexture(video);
```

If `muted` is absent, Chrome, Firefox, Edge, and Safari all block autoplay and `video.play()` returns a rejected Promise. The texture will show the first decoded frame (or black) and never advance.

**Explicit play call with error handling:**

```js
video.muted = true;
video.play().catch(err => {
  // Autoplay blocked — attach a UI prompt to trigger play() on user gesture
  console.warn('Autoplay blocked:', err);
});
```

---

## iOS-specific: playsinline

On iOS Safari, `<video>` without `playsinline` enters fullscreen and leaves your Three.js canvas behind. `playsinline` keeps the video rendering inline (and invisible, since it is used only as a texture source).

```html
<video autoplay muted loop playsinline crossorigin="anonymous" src="..."></video>
```

`crossorigin="anonymous"` is required when the video is on a different origin, otherwise `texImage2D` throws a tainted-canvas security error.

---

## loop vs. Programmatic Restart

`loop` on the element is the simplest way to keep a background video cycling. If you need more control (e.g., crossfade to a second clip at the end), handle the `ended` event:

```js
video.addEventListener('ended', () => {
  video.currentTime = 0;
  video.play();
});
```

---

## Texture Format Considerations

| Scenario | Recommended format / colorSpace |
|---|---|
| Standard sRGB video (most cases) | `THREE.RGBAFormat` + `texture.colorSpace = THREE.SRGBColorSpace` |
| WebGPURenderer | Must set `colorSpace = THREE.SRGBColorSpace` explicitly |
| Alpha channel video (VP9 with alpha, WebM) | `THREE.RGBAFormat` (default) — alpha channel is passed through |
| HDR video / linear pipeline | `THREE.RGBAFormat`, `type = THREE.HalfFloatType`, `colorSpace = THREE.LinearSRGBColorSpace` |

`generateMipmaps` defaults to `false` for VideoTexture. If you force it `true`, you must call `texture.needsUpdate = true` AND ensure dimensions are power-of-two. In practice, leave it `false` — video frames are too expensive to mipmap at runtime.

---

## Minimal Working Example

```js
// 1. Invisible video element in DOM (or created programmatically)
const video = document.createElement('video');
video.src = '/assets/background.mp4';
video.loop = true;
video.muted = true;
video.playsInline = true;
video.crossOrigin = 'anonymous';
video.play();

// 2. Wrap in VideoTexture
const texture = new THREE.VideoTexture(video);
texture.colorSpace = THREE.SRGBColorSpace;

// 3. Assign to a material
const material = new THREE.MeshBasicMaterial({ map: texture });
const mesh = new THREE.Mesh(new THREE.PlaneGeometry(16, 9), material);
scene.add(mesh);

// 4. Render loop — VideoTexture.update() is called automatically
function animate() {
  requestAnimationFrame(animate);
  renderer.render(scene, camera);
}
animate();
```

---

## Disposal

VideoTexture holds a GPU texture allocation. Always dispose when done:

```js
texture.dispose();
video.pause();
video.src = '';
video.load(); // releases browser media resources
```

Once disposed, call `texture.dispose()` and recreate — you cannot resize or change the format of an existing instance.

---

## Sources

- https://threejs.org/docs/pages/VideoTexture.html — accessed 2026-05-26
- https://github.com/mrdoob/three.js/issues/16946 — VideoTexture uploads on pause
- https://github.com/mrdoob/three.js/issues/22380 — needsUpdate when paused
- https://developer.mozilla.org/en-US/docs/Web/API/HTMLVideoElement/requestVideoFrameCallback — frame callback API
