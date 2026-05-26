# AnalyserNode for Music Visualization

## What AnalyserNode Does

`AnalyserNode` is a pass-through audio node that captures a snapshot of the signal
at each animation frame without altering the audio graph. It provides two data
views of the signal:

- **Frequency domain** (`getByteFrequencyData`, `getFloatFrequencyData`) — FFT magnitude
  spectrum, useful for bar charts, ring visualizations, and frequency-reactive shaders.
- **Time domain** (`getByteTimeDomainData`, `getFloatTimeDomainData`) — raw waveform
  samples, useful for oscilloscope-style visuals and amplitude envelopes.

Source: https://developer.mozilla.org/en-US/docs/Web/API/AnalyserNode
Accessed: 2026-05-26

---

## AnalyserNode Properties and Defaults

Source: https://developer.mozilla.org/en-US/docs/Web/API/AnalyserNode/AnalyserNode
Accessed: 2026-05-26

| Property | Default | Notes |
|---|---|---|
| `fftSize` | `2048` | Power of 2, range 32–32768. Larger = more frequency resolution, more CPU. |
| `frequencyBinCount` | `1024` | **Read-only.** Always `fftSize / 2`. This is the array length you allocate. |
| `minDecibels` | `-100` | Lower bound of FFT output mapped to byte value `0` in `getByteFrequencyData`. |
| `maxDecibels` | `-30` | Upper bound mapped to byte value `255`. |
| `smoothingTimeConstant` | `0.8` | 0 = no smoothing (jagged); 1 = maximum smoothing (sluggish). Range 0–1. |

### fftSize valid values

`32, 64, 128, 256, 512, 1024, 2048, 4096, 8192, 16384, 32768`

Larger `fftSize` gives finer frequency resolution but requires more computation and
increases `frequencyBinCount`. A common visualization value is `2048`
(yielding `1024` bins) or `512` (yielding `256` bins) for performance-sensitive work.

---

## Creating and Connecting an AnalyserNode

### Standalone (vanilla Web Audio API)

```js
const ctx      = new AudioContext();
const analyser = ctx.createAnalyser();

analyser.fftSize             = 2048;
analyser.minDecibels         = -90;
analyser.maxDecibels         = -10;
analyser.smoothingTimeConstant = 0.85;

// Wire: source → analyser → destination
// AnalyserNode is pass-through; audio continues to the speakers.
sourceNode.connect(analyser);
analyser.connect(ctx.destination);

// Allocate once — reuse every frame
const dataArray = new Uint8Array(analyser.frequencyBinCount);   // 1024 elements
```

### With THREE.Audio

```js
const listener = new THREE.AudioListener();
camera.add(listener);

const sound   = new THREE.Audio(listener);
const analyser = listener.context.createAnalyser();
analyser.fftSize = 512;

// Insert after the Audio's gain node, before the listener's input
sound.setFilter(analyser);    // or use sound.connect() / disconnect() manually
// Alternatively, connect the analyser in parallel:
// sound.gain.connect(analyser);
// analyser connects to nothing (no destination needed for capture-only)

const freqData = new Uint8Array(analyser.frequencyBinCount);   // 256 elements
```

If you only need analysis and not to reroute audio, tap the node without
disconnecting the main path:

```js
// Tap in parallel without interrupting existing routing
sound.gain.connect(analyser);
// analyser does NOT need to connect to a destination — it still fills buffers
```

---

## Reading Data

### getByteFrequencyData(array)

Copies the current FFT magnitude values into a `Uint8Array`. Each element is a byte
(0–255) where `0` maps to `minDecibels` and `255` maps to `maxDecibels`. Call once per
animation frame immediately before consuming the data.

```js
const dataArray = new Uint8Array(analyser.frequencyBinCount);

function draw() {
  requestAnimationFrame(draw);
  analyser.getByteFrequencyData(dataArray);
  // dataArray[0]   = sub-bass magnitude (lowest frequency bin)
  // dataArray[511] = near-Nyquist magnitude (for fftSize 1024)
}
```

### getFloatFrequencyData(array)

Same as above but writes `Float32Array` values in decibels. More accurate for threshold
detection and level metering; no loss from the 0–255 byte quantization.

```js
const floatData = new Float32Array(analyser.frequencyBinCount);
analyser.getFloatFrequencyData(floatData);
// Values in dBFS — e.g., -80.5, -45.2, -12.0
```

### getByteTimeDomainData(array)

Copies the current waveform (time-domain) data. Values are centered at `128`
(representing `0.0` amplitude). Range `0–255` maps to `[-1.0, 1.0]`.

```js
const waveData = new Uint8Array(analyser.fftSize);   // note: fftSize, not frequencyBinCount
analyser.getByteTimeDomainData(waveData);
```

---

## Connecting to Three.js Geometry (Bar Visualizer)

```js
const NUM_BARS     = 64;
const geometry     = new THREE.BoxGeometry(0.4, 1, 0.4);
const bars         = [];

for (let i = 0; i < NUM_BARS; i++) {
  const mesh = new THREE.Mesh(geometry, new THREE.MeshStandardMaterial());
  mesh.position.x = (i - NUM_BARS / 2) * 0.5;
  scene.add(mesh);
  bars.push(mesh);
}

const analyser = listener.context.createAnalyser();
analyser.fftSize = NUM_BARS * 2;     // frequencyBinCount === NUM_BARS
sound.gain.connect(analyser);

const freqData = new Uint8Array(analyser.frequencyBinCount);

function animate() {
  requestAnimationFrame(animate);

  analyser.getByteFrequencyData(freqData);

  for (let i = 0; i < NUM_BARS; i++) {
    const scale = 1 + (freqData[i] / 255) * 10;
    bars[i].scale.y = scale;
    bars[i].position.y = scale * 0.5;
  }

  renderer.render(scene, camera);
}
animate();
```

---

## Connecting to Shader Uniforms

Upload the frequency array as a `DataTexture` to avoid uploading 1D data as individual
uniforms. The texture's `needsUpdate` flag triggers a GPU upload each frame.

```js
const FFT_SIZE   = 512;
const BIN_COUNT  = FFT_SIZE / 2;     // 256

const analyser = listener.context.createAnalyser();
analyser.fftSize = FFT_SIZE;
sound.gain.connect(analyser);

const freqData    = new Uint8Array(BIN_COUNT);
const audioTexture = new THREE.DataTexture(
  freqData,          // data reference — same Uint8Array updated each frame
  BIN_COUNT,         // width
  1,                 // height
  THREE.RedFormat,   // one channel
  THREE.UnsignedByteType
);

// Assign to a ShaderMaterial uniform
const material = new THREE.ShaderMaterial({
  uniforms: {
    uAudioTex: { value: audioTexture },
    uTime:     { value: 0 },
  },
  vertexShader:   vertSrc,
  fragmentShader: fragSrc,
});

function animate() {
  requestAnimationFrame(animate);

  analyser.getByteFrequencyData(freqData);
  audioTexture.needsUpdate = true;          // upload to GPU

  material.uniforms.uTime.value += 0.016;
  renderer.render(scene, camera);
}
```

Fragment shader sample:

```glsl
uniform sampler2D uAudioTex;
varying vec2 vUv;

void main() {
  float freq = texture2D(uAudioTex, vec2(vUv.x, 0.5)).r;  // 0.0–1.0
  gl_FragColor = vec4(vec3(freq), 1.0);
}
```

---

## requestAnimationFrame Timing

`getByteFrequencyData` and `getByteTimeDomainData` return a snapshot computed at the
time of the call. The audio engine updates its internal FFT independently at
approximately every audio render quantum (128 samples ≈ 2.9 ms at 44100 Hz).
`requestAnimationFrame` fires at display refresh rate (typically 60 Hz ≈ 16.7 ms).

There is **no need** to throttle or debounce `getByteFrequencyData` calls — calling it
once per rAF frame is correct and cheap. The `smoothingTimeConstant` already handles
temporal averaging.

**Do not call from a `setInterval`** — it decouples the read from the render loop and
creates a stale-data race condition with the draw calls.

---

## Performance Notes

- Allocate `dataArray` **once** before the loop. `new Uint8Array()` inside rAF is a GC
  pressure source.
- `getByteFrequencyData` is faster than `getFloatFrequencyData`; use `Float32Array`
  only when you need dB-accurate values.
- `fftSize 2048` costs roughly 2× more than `fftSize 1024`. Profile before increasing.
- `smoothingTimeConstant 0.0` is valid and correct for reactive, jittery visuals.
  Increase toward `1.0` for smooth slow animations.

---

## THREE.AudioAnalyser Helper

Three.js ships a `THREE.AudioAnalyser` convenience wrapper:

```js
import { AudioAnalyser } from 'three';

// constructor(audio, fftSize)
const threeAnalyser = new AudioAnalyser(sound, 32);

// In render loop:
const data = threeAnalyser.getFrequencyData();    // returns the Uint8Array
const avg  = threeAnalyser.getAverageFrequency(); // single average 0–255
```

Internally it creates an `AnalyserNode`, wires it into the audio graph, and manages the
data buffer. For custom `minDecibels` / `maxDecibels` / `smoothingTimeConstant`,
access the raw node:

```js
threeAnalyser.analyser.minDecibels = -90;
threeAnalyser.analyser.smoothingTimeConstant = 0.7;
```

---

## Sources

| URL | Tier | Accessed |
|---|---|---|
| https://developer.mozilla.org/en-US/docs/Web/API/AnalyserNode | 1 (MDN) | 2026-05-26 |
| https://developer.mozilla.org/en-US/docs/Web/API/AnalyserNode/AnalyserNode | 1 (MDN) | 2026-05-26 |
