# Three.js Audio Setup

## Overview

Three.js wraps the Web Audio API through four classes:
`THREE.AudioListener`, `THREE.Audio`, `THREE.PositionalAudio`, and `THREE.AudioLoader`.
The listener models the human ear; it must be attached to the camera so the audio graph
knows the observer's position and orientation. Global audio sources (`THREE.Audio`) are
panned in stereo only; positional sources (`THREE.PositionalAudio`) participate in full
3D spatial processing via a `PannerNode`.

---

## AudioContext and the Autoplay Policy

**Every interaction with Web Audio starts with an `AudioContext`.** Browsers enforce an
autoplay policy: an `AudioContext` created before a user gesture enters `"suspended"`
state and produces no sound until it is explicitly resumed.

### Rule
> "Create or resume context from inside a user gesture."
> — MDN Web Audio API Best Practices
> Source: https://developer.mozilla.org/en-US/docs/Web/API/Web_Audio_API/Best_practices
> Accessed: 2026-05-26

Three.js creates the `AudioContext` lazily the first time `THREE.AudioListener` needs it
(via `THREE.AudioContext.getContext()`). That call will still land outside any user
gesture in most scene setups, so you must resume manually.

### Recommended pattern

```js
// Resume on first user interaction (click, keydown, touchstart, etc.)
document.addEventListener('click', () => {
  const ctx = THREE.AudioContext.getContext();   // or listener.context
  if (ctx.state === 'suspended') {
    ctx.resume();
  }
}, { once: true });
```

### Alternative: create the context inside the handler

```js
let listener;

document.getElementById('start').addEventListener('click', () => {
  listener = new THREE.AudioListener();
  camera.add(listener);               // context is now 'running'
  loadAndPlayAudio(listener);
});
```

`AudioContext.state` values:
- `"suspended"` — created outside a gesture, or after `suspend()`.
- `"running"` — actively processing.
- `"closed"` — permanently torn down via `close()`.
- `"interrupted"` — paused by OS (e.g., phone call, tab switch on iOS).

Source: https://developer.mozilla.org/en-US/docs/Web/API/AudioContext/state
Accessed: 2026-05-26

---

## THREE.AudioListener

`AudioListener` extends `Object3D`. It creates and owns the `AudioContext` and a master
`GainNode`, then calls the Web Audio listener API (`AudioListener.positionX/Y/Z`,
`AudioListener.forwardX/Y/Z`, `AudioListener.upX/Y/Z`) inside `updateMatrixWorld()` so
that Three.js scene transforms drive spatial audio automatically.

### Constructor

```js
const listener = new THREE.AudioListener();
camera.add(listener);   // REQUIRED — keeps position in sync with camera
```

Adding the listener as a child of the camera is the canonical pattern. `updateMatrixWorld`
decomposes the world matrix and writes to the underlying `AudioContext.listener` every
frame.

### Key methods

| Method | Returns | Notes |
|---|---|---|
| `getContext()` | `AudioContext` | The shared Web Audio context |
| `getInput()` | `GainNode` | Master gain node; audio sources connect here |
| `getMasterVolume()` | `number` | Current gain value |
| `setMasterVolume(value)` | `this` | Smooth transition via `setTargetAtTime` |
| `getFilter()` | `AudioNode \| null` | Currently inserted filter |
| `setFilter(node)` | `this` | Insert any `AudioNode` into master path |
| `removeFilter()` | `this` | Remove inserted filter |

Source: https://github.com/mrdoob/three.js/blob/dev/src/audio/AudioListener.js
Accessed: 2026-05-26

### Master volume

```js
listener.setMasterVolume(0.5);   // 50% — affects every audio node in the scene
```

---

## THREE.Audio (Global / Non-Positional)

`THREE.Audio` extends `Object3D` and wraps an `AudioBufferSourceNode` (or
`MediaElementAudioSourceNode` / `MediaStreamAudioSourceNode`) connected to the listener's
gain node. Because it connects directly to the master gain without a `PannerNode`,
volume does not change with distance or position.

### Constructor

```js
const sound = new THREE.Audio(listener);
```

### Core workflow

```js
const loader = new THREE.AudioLoader();
loader.load('audio/ambient.ogg', (buffer) => {
  sound.setBuffer(buffer);
  sound.setLoop(true);
  sound.setVolume(0.8);
  sound.play();
});
```

### Key methods

| Method | Description |
|---|---|
| `setBuffer(audioBuffer)` | Assign decoded `AudioBuffer`; triggers autoplay if enabled |
| `play(delay?)` | Start playback; `delay` in seconds |
| `pause()` | Suspend playback, preserving position |
| `stop(delay?)` | Stop and reset position to zero |
| `setLoop(bool)` | Enable/disable looping |
| `setLoopStart(s)` / `setLoopEnd(s)` | Loop region in seconds |
| `setVolume(value)` | Set gain (0–1 typical) |
| `setPlaybackRate(value)` | Playback speed multiplier |
| `setFilters([nodes])` | Insert effects chain between source and output |
| `getFilters()` | Return current filter array |
| `connect()` / `disconnect()` | Manage routing manually |
| `setMediaElementSource(el)` | Use `<audio>` element as source |
| `setMediaStreamSource(stream)` | Use `MediaStream` as source |

Source: https://github.com/mrdoob/three.js/blob/dev/src/audio/Audio.js
Accessed: 2026-05-26

---

## THREE.PositionalAudio (3D Spatial)

`THREE.PositionalAudio` extends `THREE.Audio` and inserts a `PannerNode` between the
source and the listener's gain. The panner node is created with
`context.createPanner()` and defaults to `panningModel: 'HRTF'`.
`updateMatrixWorld()` decomposes the object's world transform and applies position and
orientation to the panner's `AudioParam` nodes via `linearRampToValueAtTime` (with a
legacy `setPosition`/`setOrientation` fallback).

### Constructor

```js
const posSound = new THREE.PositionalAudio(listener);
```

Attach it to any mesh so its world transform tracks the object:

```js
mesh.add(posSound);
```

### Distance / rolloff methods

| Method | Default value |
|---|---|
| `setRefDistance(v)` / `getRefDistance()` | `1` |
| `setRolloffFactor(v)` / `getRolloffFactor()` | `1` |
| `setMaxDistance(v)` / `getMaxDistance()` | `10000` |
| `setDistanceModel(s)` / `getDistanceModel()` | `"inverse"` |

### Directional cone

```js
posSound.setDirectionalCone(
  360,   // coneInnerAngle — no attenuation inside this arc
  360,   // coneOuterAngle — full attenuation outside this arc
  0      // coneOuterGain — 0 = silent outside the outer cone
);
```

Default `coneInnerAngle: 360`, `coneOuterAngle: 360` means omnidirectional. Narrow the
angles to model a speaker or a directional source.

Source: https://github.com/mrdoob/three.js/blob/dev/src/audio/PositionalAudio.js
Accessed: 2026-05-26

---

## THREE.AudioLoader

`AudioLoader` extends `Loader` and internally:
1. Fetches the URL with `FileLoader` as an `ArrayBuffer`.
2. Calls `AudioContext.decodeAudioData(arrayBuffer)` to decode into an `AudioBuffer`.
3. Passes the `AudioBuffer` to the `onLoad` callback.

### Method signature

```js
audioLoader.load(url, onLoad, onProgress, onError)
```

### Usage

```js
const loader = new THREE.AudioLoader();

loader.load(
  'audio/explosion.mp3',
  (buffer) => {
    posSound.setBuffer(buffer);
    posSound.setLoop(false);
    posSound.play();
  },
  (xhr) => console.log(`${(xhr.loaded / xhr.total * 100).toFixed(1)}% loaded`),
  (err) => console.error('AudioLoader error:', err)
);
```

Source: https://threejs.org/docs/#api/en/loaders/AudioLoader
Accessed: 2026-05-26

---

## Minimal Scene Wiring

```js
import * as THREE from 'three';

// 1. Scene and camera
const scene    = new THREE.Scene();
const camera   = new THREE.PerspectiveCamera(70, window.innerWidth / window.innerHeight);
const renderer = new THREE.WebGLRenderer({ antialias: true });
renderer.setSize(window.innerWidth, window.innerHeight);
document.body.appendChild(renderer.domElement);

// 2. AudioListener — must live on the camera
const listener = new THREE.AudioListener();
camera.add(listener);

// 3. Autoplay gate
renderer.domElement.addEventListener('click', () => {
  if (listener.context.state === 'suspended') listener.context.resume();
}, { once: true });

// 4. Global ambient sound
const ambient = new THREE.Audio(listener);
new THREE.AudioLoader().load('audio/music.mp3', (buf) => {
  ambient.setBuffer(buf);
  ambient.setLoop(true);
  ambient.setVolume(0.4);
  // Play only after user gesture unlocks context
  listener.context.resume().then(() => ambient.play());
});

// 5. Positional sfx on a mesh
const mesh    = new THREE.Mesh(new THREE.BoxGeometry(), new THREE.MeshStandardMaterial());
const possfx  = new THREE.PositionalAudio(listener);
mesh.add(possfx);
scene.add(mesh);

new THREE.AudioLoader().load('audio/beep.ogg', (buf) => {
  possfx.setBuffer(buf);
  possfx.setRefDistance(5);
  possfx.setLoop(true);
  possfx.play();
});

// 6. Render loop
function animate() {
  requestAnimationFrame(animate);
  renderer.render(scene, camera);
}
animate();
```

---

## Sources

| URL | Tier | Accessed |
|---|---|---|
| https://developer.mozilla.org/en-US/docs/Web/API/Web_Audio_API/Best_practices | 1 (MDN) | 2026-05-26 |
| https://developer.mozilla.org/en-US/docs/Web/API/AudioContext/state | 1 (MDN) | 2026-05-26 |
| https://github.com/mrdoob/three.js/blob/dev/src/audio/AudioListener.js | 1 (source) | 2026-05-26 |
| https://github.com/mrdoob/three.js/blob/dev/src/audio/Audio.js | 1 (source) | 2026-05-26 |
| https://github.com/mrdoob/three.js/blob/dev/src/audio/PositionalAudio.js | 1 (source) | 2026-05-26 |
| https://threejs.org/docs/#api/en/loaders/AudioLoader | 1 (official docs) | 2026-05-26 |
