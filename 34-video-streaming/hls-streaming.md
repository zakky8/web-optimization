# HLS Streaming with Three.js VideoTexture

**Last verified:** 2026-05-26  
**hls.js stable:** v1.6.16 (released 2025-04-13)  
**hls.js repo:** https://github.com/video-dev/hls.js  
**npm:** `npm install hls.js`

---

## Why HLS for 3D Backgrounds

A flat MP4 loaded via `<video src="...">` downloads the entire file before playback is reliable on slow connections. HLS (HTTP Live Streaming) segments the video into small chunks (typically 2–10 seconds each) served over plain HTTP. Combined with an ABR manifest listing multiple quality renditions, the player automatically selects the bitrate the client's connection can sustain — giving you smooth playback across a wide range of devices without stalling.

For Three.js scenes where video is a background or texture layer, adaptive bitrate means:
- Mobile devices on LTE stream a lower-res rendition without stuttering
- Desktop on fast Wi-Fi gets the highest quality rendition
- Segment buffering keeps the video element in `HAVE_CURRENT_DATA` state, which is what `THREE.VideoTexture` needs to keep uploading frames

---

## Browser Native HLS vs. hls.js

| Browser | Native HLS | Notes |
|---|---|---|
| Safari (macOS + iOS) | Yes — all versions | Safari delegates to the OS media stack |
| Chrome | No | Needs hls.js or MediaSource Extensions |
| Firefox | No | Needs hls.js |
| Edge | No | Needs hls.js |

**Detection pattern:**

```js
if (video.canPlayType('application/vnd.apple.mpegurl')) {
  // Safari — assign .m3u8 directly
  video.src = 'stream.m3u8';
} else if (Hls.isSupported()) {
  // All other modern browsers — use hls.js
  const hls = new Hls();
  hls.loadSource('stream.m3u8');
  hls.attachMedia(video);
}
```

Do not run hls.js on Safari. Safari's native HLS is more power-efficient and handles DRM (FairPlay) natively. Running hls.js on Safari causes double-decode overhead.

---

## Installation

```bash
npm install hls.js
```

Or via CDN (for quick tests):

```html
<script src="https://cdn.jsdelivr.net/npm/hls.js@1.6.16/dist/hls.min.js"></script>
```

---

## Full Integration Example

```js
import Hls from 'hls.js';
import * as THREE from 'three';

// 1. Create the video element
const video = document.createElement('video');
video.muted = true;
video.loop = true;
video.playsInline = true;
video.crossOrigin = 'anonymous';

// 2. Wire up hls.js or native HLS
function initHLS(streamUrl) {
  if (video.canPlayType('application/vnd.apple.mpegurl')) {
    // Safari — native
    video.src = streamUrl;
    video.addEventListener('loadedmetadata', () => video.play());
  } else if (Hls.isSupported()) {
    const hls = new Hls({
      maxBufferLength: 30,       // seconds of forward buffer
      maxMaxBufferLength: 60,    // hard cap
      startLevel: -1,            // -1 = auto-select starting level
    });
    hls.loadSource(streamUrl);
    hls.attachMedia(video);
    hls.on(Hls.Events.MANIFEST_PARSED, () => video.play());
  } else {
    console.error('HLS not supported in this browser');
  }
}

// 3. Create VideoTexture after the video is ready
const texture = new THREE.VideoTexture(video);
texture.colorSpace = THREE.SRGBColorSpace;

const material = new THREE.MeshBasicMaterial({ map: texture });
const mesh = new THREE.Mesh(new THREE.PlaneGeometry(16, 9), material);
scene.add(mesh);

// 4. Start streaming
initHLS('https://example.com/stream/master.m3u8');

// 5. Render loop
function animate() {
  requestAnimationFrame(animate);
  renderer.render(scene, camera);
}
animate();
```

---

## Adaptive Bitrate Mechanics

hls.js reads the HLS master manifest (`.m3u8`) which lists renditions at different bitrates:

```
#EXTM3U
#EXT-X-STREAM-INF:BANDWIDTH=400000,RESOLUTION=640x360
low/stream.m3u8
#EXT-X-STREAM-INF:BANDWIDTH=1200000,RESOLUTION=1280x720
mid/stream.m3u8
#EXT-X-STREAM-INF:BANDWIDTH=4000000,RESOLUTION=1920x1080
high/stream.m3u8
```

The ABR controller measures each segment's download throughput and compares it against the current level's bitrate. If bandwidth drops below the current level's requirement, it immediately switches down. If bandwidth has been consistently above the next level's requirement for several segments, it switches up.

**Level switching modes (set via `Hls` config):**

| Mode | Config key | Behavior |
|---|---|---|
| Instant | `abrBandWidthFactor` tuning | Switches at next segment boundary, may cause brief re-seek |
| Smooth | `capLevelOnFPSDrop: true` | Waits for buffer continuity before switching |
| Emergency | automatic | Triggers when buffer runs below `maxStarvationDelay` |

```js
const hls = new Hls({
  startLevel: -1,           // -1 = ABR selects start level
  capLevelToPlayerSize: true, // never load rendition larger than canvas
  capLevelOnFPSDrop: true,    // drop quality if frame rate falls
  abrEwmaDefaultEstimate: 500000, // initial bandwidth estimate in bps
});
```

`capLevelToPlayerSize: true` is particularly useful for Three.js: if your canvas is 640×360 CSS pixels, there is no point downloading a 4K rendition.

---

## Buffer Management

hls.js maintains two separate buffers:

- **Forward buffer** (`maxBufferLength`): how many seconds ahead to pre-load. Default 30 s. For interactive 3D scenes where the user may close or navigate away, 10–15 s is usually enough.
- **Back buffer** (`backBufferLength`): how many seconds behind the current position to retain. Set to `0` to free memory aggressively on long-running streams.

```js
const hls = new Hls({
  maxBufferLength: 15,
  backBufferLength: 0,
  maxBufferHole: 0.5,       // tolerate gaps up to 0.5 s without stalling
});
```

**Buffer events to monitor:**

```js
hls.on(Hls.Events.BUFFER_APPENDED, (event, data) => {
  // data.timeRanges shows current buffered ranges
});

hls.on(Hls.Events.ERROR, (event, data) => {
  if (data.fatal) {
    switch (data.type) {
      case Hls.ErrorTypes.NETWORK_ERROR:
        hls.startLoad(); // attempt recovery
        break;
      case Hls.ErrorTypes.MEDIA_ERROR:
        hls.recoverMediaError();
        break;
      default:
        hls.destroy();
    }
  }
});
```

---

## Level Switching Events for UI

If you want to display stream quality in a debug overlay:

```js
hls.on(Hls.Events.LEVEL_SWITCHED, (event, data) => {
  const level = hls.levels[data.level];
  console.log(`Switched to ${level.width}x${level.height} @ ${level.bitrate} bps`);
});
```

---

## Cleanup

```js
hls.destroy();          // stops segment loading, releases MSE SourceBuffer
texture.dispose();      // releases GPU texture
video.src = '';
video.load();
```

---

## Sources

- https://github.com/video-dev/hls.js — accessed 2026-05-26 (stable: v1.6.16, 2025-04-13)
- https://github.com/video-dev/hls.js/blob/master/docs/API.md — hls.js API reference
- https://developer.mozilla.org/en-US/docs/Web/API/Media_Source_Extensions_API — MSE spec
