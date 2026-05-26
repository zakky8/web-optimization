# Audio Performance

## AudioContext State Management

Every `AudioContext` carries a `state` property that governs whether the audio graph
is running and consuming resources.

Source: https://developer.mozilla.org/en-US/docs/Web/API/AudioContext/state
Accessed: 2026-05-26

| State | Cause | Resource cost |
|---|---|---|
| `"suspended"` | Created before user gesture; or `suspend()` called | Graph frozen; no DSP |
| `"running"` | Normal operation | Full CPU/GPU audio DSP active |
| `"closed"` | `close()` called | All resources released (permanent) |
| `"interrupted"` | OS audio session taken by another app (iOS, Android) | Graph paused by browser |

### suspend() and resume()

```js
const ctx = listener.context;

// Free audio hardware when the page is not visible
document.addEventListener('visibilitychange', () => {
  if (document.hidden) {
    ctx.suspend();
  } else {
    ctx.resume();
  }
});
```

`suspend()` and `resume()` both return `Promise<void>`. Await them if you need to
confirm the state transition before proceeding.

```js
await ctx.suspend();
console.log(ctx.state);   // "suspended"
await ctx.resume();
console.log(ctx.state);   // "running"
```

### onstatechange event

```js
ctx.addEventListener('statechange', () => {
  console.log('AudioContext state:', ctx.state);
});
// or
ctx.onstatechange = () => console.log(ctx.state);
```

### iOS "interrupted" state

iOS Safari interrupts the audio context when the user locks the screen, takes a call,
or switches apps. Handle it explicitly:

```js
function safeResume(ctx) {
  if (ctx.state === 'interrupted' || ctx.state === 'suspended') {
    return ctx.resume();
  }
  return Promise.resolve();
}

// Re-attach on any user interaction
document.addEventListener('touchstart', () => safeResume(listener.context));
```

Source: https://developer.mozilla.org/en-US/docs/Web/API/AudioContext/state
Accessed: 2026-05-26

---

## Autoplay Policy: User Gesture Requirement

Browsers will not start audio rendering until the user has interacted with the page.
An `AudioContext` created before a gesture enters `"suspended"` state.

Source: https://developer.mozilla.org/en-US/docs/Web/API/Web_Audio_API/Best_practices
Accessed: 2026-05-26

### Three.js pattern

```js
const listener = new THREE.AudioListener();
camera.add(listener);

// Gate — resume on first click/key/touch
const resumeCtx = () => {
  const ctx = THREE.AudioContext.getContext();
  if (ctx.state !== 'running') ctx.resume();
};
window.addEventListener('click',     resumeCtx, { once: false });
window.addEventListener('keydown',   resumeCtx, { once: false });
window.addEventListener('touchstart',resumeCtx, { once: false });
```

Using `{ once: false }` rather than `{ once: true }` handles the iOS interrupted state
recovery as well.

---

## Buffer Pre-loading vs Streaming

### AudioBufferSourceNode (pre-loaded)

The entire audio file is decoded into memory as a `Float32Array`-backed `AudioBuffer`.
Playback has zero seek latency and supports arbitrary looping, playback rate, and
sample-accurate scheduling.

**Use for:** Sound effects, short music loops, any audio that must play instantly and
repeatedly.

```js
async function loadBuffer(ctx, url) {
  const response = await fetch(url);
  const arrayBuffer = await response.arrayBuffer();
  return ctx.decodeAudioData(arrayBuffer);
  // Returns Promise<AudioBuffer> — resampled to ctx.sampleRate automatically
}

// Pre-load at startup
const [shootBuf, explosionBuf] = await Promise.all([
  loadBuffer(ctx, 'audio/shoot.ogg'),
  loadBuffer(ctx, 'audio/explosion.ogg'),
]);

// Fire on demand — instant, no decode overhead
function playShoot() {
  const src = ctx.createBufferSource();
  src.buffer = shootBuf;
  src.connect(ctx.destination);
  src.start();
}
```

Each call to `ctx.createBufferSource()` is cheap. The `AudioBuffer` is shared; the
source node is single-use (play it once and discard).

Source: https://developer.mozilla.org/en-US/docs/Web/API/BaseAudioContext/decodeAudioData
Accessed: 2026-05-26

### MediaElementAudioSourceNode (streaming)

An HTML `<audio>` element is wrapped as a source node. The browser streams and decodes
the file incrementally. No large upfront memory allocation, but:

- Seeking is not sample-accurate.
- Looping may have a brief gap at the loop point (browser/codec dependent).
- The `<audio>` element must have appropriate CORS headers for cross-origin media.

**Use for:** Long-form music tracks, podcasts, ambient streams where memory cost matters.

```js
const el   = document.createElement('audio');
el.src     = 'audio/music.mp3';
el.loop    = true;
const src  = ctx.createMediaElementSource(el);
src.connect(ctx.destination);
el.play();
```

Three.js wraps this as `sound.setMediaElementSource(el)`.

---

## THREE.AudioLoader Pre-loading Pattern

Pre-load all sound effects at the start of the experience, before the gameplay loop
begins, to eliminate decode hitches during play.

```js
const loader = new THREE.AudioLoader();

// Pre-load into a map
const buffers = {};

async function preloadSounds(listener) {
  const files = {
    shoot:     'audio/shoot.ogg',
    explosion: 'audio/explosion.ogg',
    ambient:   'audio/ambient.mp3',
  };

  await Promise.all(
    Object.entries(files).map(([key, url]) =>
      new Promise((resolve, reject) => {
        loader.load(url, (buf) => { buffers[key] = buf; resolve(); }, undefined, reject);
      })
    )
  );
}

// Play from cache — no decode cost at runtime
function spawnSfx(listener, key) {
  const sound = new THREE.Audio(listener);
  sound.setBuffer(buffers[key]);
  sound.play();
}
```

Source: https://threejs.org/docs/#api/en/loaders/AudioLoader
Accessed: 2026-05-26

---

## latencyHint: Tuning AudioContext Latency

Set at construction time; cannot be changed afterwards.

Source: https://developer.mozilla.org/en-US/docs/Web/API/AudioContext/AudioContext
Accessed: 2026-05-26

| Value | Latency | Use case |
|---|---|---|
| `"interactive"` (default) | Lowest possible | Sound effects that must respond to input |
| `"balanced"` | Medium | Mixed interactive + background music |
| `"playback"` | Highest | Music streaming; minimises battery drain |
| `number` (seconds) | Requested | Browser honours best-effort only |

```js
// Sound-effects-heavy game: minimise latency
const ctx = new AudioContext({ latencyHint: 'interactive' });

// Music player: favour battery life
const musicCtx = new AudioContext({ latencyHint: 'playback' });
```

Check what the browser actually granted:

```js
console.log(ctx.baseLatency);    // e.g. 0.00290249...  (seconds)
console.log(ctx.outputLatency);  // total output latency including hardware buffer
```

Three.js does not expose `latencyHint` in its constructor API. To override the default,
create the context manually before constructing `AudioListener`:

```js
// Must be called before any THREE.AudioListener is created
const ctx = new AudioContext({ latencyHint: 'interactive', sampleRate: 48000 });
THREE.AudioContext.setContext(ctx);

const listener = new THREE.AudioListener();   // reuses the context you provided
camera.add(listener);
```

---

## sampleRate

The sample rate cannot be changed after context creation. Mismatched sample rates
between the context and the hardware output cause silent resampling. `decodeAudioData`
automatically resamples loaded buffers to the context's rate.

```js
const ctx = new AudioContext({ sampleRate: 48000 });
// 48000 Hz common on professional audio hardware
```

Default is the output device's preferred rate (usually 44100 or 48000 Hz).

---

## Memory: Managing AudioBuffers

An `AudioBuffer` holds PCM audio in `Float32Array`. A 4-minute stereo file at 44100 Hz
consumes roughly:

```
4 * 60 * 44100 * 2 channels * 4 bytes = ~84 MB
```

For long files, prefer `MediaElementAudioSourceNode` (streaming). For short sound
effects, pre-loading is fine.

Release `AudioBuffer` references when they are no longer needed — let GC collect them.
There is no explicit `dispose()` method.

---

## Closing an AudioContext

Once `close()` is called the context is permanently unusable. Do this when the user
leaves the page or the scene is torn down.

```js
window.addEventListener('beforeunload', () => {
  listener.context.close();
});

// Or in Three.js teardown
function disposeScene() {
  renderer.dispose();
  listener.context.close();
}
```

---

## Common Performance Mistakes

| Mistake | Impact | Fix |
|---|---|---|
| `new AudioContext()` on every play | Context limit (Chrome: 6/tab); memory leak | Create once; reuse |
| `decodeAudioData` at play time | CPU spike causes frame drop | Pre-load at load screen |
| `new Uint8Array(analyser.frequencyBinCount)` inside rAF | GC pressure | Allocate once before loop |
| Leaving context running on hidden tab | Background CPU drain | `suspend()` on `visibilitychange` |
| Not handling `"interrupted"` state | Silent audio on mobile | `resume()` on user interaction |
| Streaming a short SFX file | Network latency on first play | Pre-load short files as `AudioBuffer` |

---

## Sources

| URL | Tier | Accessed |
|---|---|---|
| https://developer.mozilla.org/en-US/docs/Web/API/AudioContext/state | 1 (MDN) | 2026-05-26 |
| https://developer.mozilla.org/en-US/docs/Web/API/Web_Audio_API/Best_practices | 1 (MDN) | 2026-05-26 |
| https://developer.mozilla.org/en-US/docs/Web/API/AudioContext/AudioContext | 1 (MDN) | 2026-05-26 |
| https://developer.mozilla.org/en-US/docs/Web/API/BaseAudioContext/decodeAudioData | 1 (MDN) | 2026-05-26 |
| https://threejs.org/docs/#api/en/loaders/AudioLoader | 1 (official docs) | 2026-05-26 |
