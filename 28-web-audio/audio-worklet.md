# AudioWorklet

## Why ScriptProcessorNode is Deprecated

`ScriptProcessorNode` processed audio on the **main JavaScript thread** via an
`onaudioprocess` event. Any main-thread jank — a GC pause, a layout recalculation, a
heavy render frame — could starve the audio callback, producing audible glitches, pops,
and dropouts. Buffer sizes had to be a power of two between 256 and 16384; small buffers
minimized latency but maximized the chance of starvation.

MDN deprecation notice (exact language):
> "This feature is no longer recommended. Though some browsers might still support it,
> it may have already been removed from the relevant web standards, may be in the process
> of being dropped, or may only be kept for compatibility purposes. Avoid using it, and
> update existing code if possible."
> — https://developer.mozilla.org/en-US/docs/Web/API/ScriptProcessorNode
> Accessed: 2026-05-26

**AudioWorklet replaces `ScriptProcessorNode`** by running processor code in a dedicated
audio rendering thread (the same thread as the native audio nodes), isolated from main-
thread jank.

---

## AudioWorklet Architecture

```
Main thread                        Audio rendering thread
─────────────────                  ──────────────────────────────
AudioContext.audioWorklet          AudioWorkletGlobalScope
  .addModule('proc.js')   ──────>    registerProcessor('my-proc', MyProc)
                                      ↕ MessagePort
AudioWorkletNode          ←────────  AudioWorkletProcessor
  .port.postMessage()                   .process(inputs, outputs, parameters)
  .parameters.get('gain')
```

Key objects:

| Object | Thread | Role |
|---|---|---|
| `AudioWorklet` | main | Entry point; `audioContext.audioWorklet` |
| `AudioWorkletNode` | main | Audio node connected into the graph |
| `AudioWorkletProcessor` | audio | User-defined DSP class |
| `AudioWorkletGlobalScope` | audio | Execution environment for the worklet |

Source: https://developer.mozilla.org/en-US/docs/Web/API/AudioWorklet
Accessed: 2026-05-26

---

## Setup: addModule()

The processor script must be loaded before any `AudioWorkletNode` is constructed.
`addModule()` returns a `Promise` that resolves when the script is registered.

```js
const audioCtx = new AudioContext();

// Load the processor (HTTPS or localhost only)
await audioCtx.audioWorklet.addModule('worklets/gain-processor.js');

// Now safe to instantiate
const gainNode = new AudioWorkletNode(audioCtx, 'gain-processor');
gainNode.connect(audioCtx.destination);
```

**Security requirement:** `addModule()` is only available in secure contexts (HTTPS or
`localhost`). The `audioWorklet` property itself requires a secure context.

Source: https://developer.mozilla.org/en-US/docs/Web/API/AudioWorkletGlobalScope/registerProcessor
Accessed: 2026-05-26

---

## Defining an AudioWorkletProcessor

The processor file runs entirely inside `AudioWorkletGlobalScope`. It has access to
`sampleRate`, `currentTime`, `currentFrame`, and `registerProcessor()`. It does **not**
have access to the DOM, `window`, or `fetch`.

### Minimal example

```js
// worklets/gain-processor.js

class GainProcessor extends AudioWorkletProcessor {

  // Declare AudioParams — visible as AudioWorkletNode.parameters
  static get parameterDescriptors() {
    return [
      {
        name: 'gain',
        defaultValue: 1.0,
        minValue: 0.0,
        maxValue: 1.0,
        automationRate: 'a-rate',  // 128 values/block (per-sample automation)
      },
    ];
  }

  process(inputs, outputs, parameters) {
    const input  = inputs[0];
    const output = outputs[0];
    const gain   = parameters['gain'];

    for (let ch = 0; ch < output.length; ch++) {
      const inCh  = input[ch];
      const outCh = output[ch];

      for (let i = 0; i < outCh.length; i++) {
        // gain may be k-rate (length 1) or a-rate (length 128)
        outCh[i] = inCh[i] * (gain.length > 1 ? gain[i] : gain[0]);
      }
    }

    return true;  // keep processor alive
  }
}

registerProcessor('gain-processor', GainProcessor);
```

Source: https://developer.mozilla.org/en-US/docs/Web/API/AudioWorkletProcessor/process
Accessed: 2026-05-26

---

## process() Method Reference

### Signature

```js
process(inputs, outputs, parameters) → boolean
```

### inputs / outputs

Both are `Array<Array<Float32Array>>`:
- `inputs[nodeInput][channel][sampleIndex]`
- `outputs[nodeInput][channel][sampleIndex]`
- Buffer size is **128 frames** per block (current spec; treat as `channel.length` in
  code — the spec permits variable sizes in future).
- Sample range: `[-1.0, 1.0]`.
- Unconnected inputs arrive as empty arrays, not zero-filled buffers.

### parameters

`{ [name]: Float32Array }`

- **k-rate** (`automationRate: 'k-rate'`): `length === 1`, one value per 128-frame block.
- **a-rate** (`automationRate: 'a-rate'`): `length === 128` when automation is scheduled;
  `length === 1` when no automation is active (optimization).

Always branch on `gain.length > 1`:

```js
// safe k-rate / a-rate handling
const g = parameters['gain'];
const gVal = (i) => g.length > 1 ? g[i] : g[0];
```

### Return value

| Return | Effect |
|---|---|
| `true` | Processor stays alive even with no inputs — correct for source nodes |
| `false` | Processor may be garbage-collected when inputs disconnect |
| `undefined` | Treated as `false` — avoid; causes subtle lifecycle bugs |

Source: https://developer.mozilla.org/en-US/docs/Web/API/AudioWorkletProcessor/process
Accessed: 2026-05-26

---

## AudioWorkletGlobalScope Properties

Available inside the worklet without any import:

| Property | Type | Description |
|---|---|---|
| `sampleRate` | `number` | Sample rate of the audio context |
| `currentTime` | `number` | Context time in seconds at start of current block |
| `currentFrame` | `number` | Frame index since context start |
| `registerProcessor(name, ctor)` | `void` | Register a processor class |

Source: https://developer.mozilla.org/en-US/docs/Web/API/AudioWorkletGlobalScope/registerProcessor
Accessed: 2026-05-26

---

## MessagePort: Main Thread ↔ Worklet Communication

Both `AudioWorkletNode` (main thread) and `AudioWorkletProcessor` (audio thread) expose
a `.port` property (`MessagePort`). Use it to send parameter snapshots, configuration
changes, or analysis data that should not be `AudioParam` automations.

### Main thread → worklet

```js
// main.js
const node = new AudioWorkletNode(audioCtx, 'gain-processor');
node.port.postMessage({ type: 'setGain', value: 0.75 });
```

### Worklet → main thread

```js
// worklets/gain-processor.js
class GainProcessor extends AudioWorkletProcessor {
  constructor(options) {
    super(options);
    this.port.onmessage = (e) => {
      if (e.data.type === 'setGain') {
        this._gain = e.data.value;
      }
    };
    this._gain = 1.0;
  }

  process(inputs, outputs, parameters) {
    const output = outputs[0];
    output[0]?.forEach((ch) => {
      for (let i = 0; i < ch.length; i++) {
        ch[i] = (inputs[0][0]?.[i] ?? 0) * this._gain;
      }
    });
    // Send metering back to main thread
    this.port.postMessage({ rms: computeRms(output[0][0]) });
    return true;
  }
}
registerProcessor('gain-processor', GainProcessor);
```

### Main thread listener

```js
node.port.onmessage = (e) => {
  vuMeter.value = e.data.rms;
};
```

Source: https://developer.mozilla.org/en-US/docs/Web/API/AudioWorkletNode
Accessed: 2026-05-26

---

## AudioWorkletNode Constructor Options

```js
const node = new AudioWorkletNode(audioCtx, 'gain-processor', {
  numberOfInputs:  1,
  numberOfOutputs: 1,
  outputChannelCount: [2],   // stereo output
  parameterData: {
    gain: 0.5,               // initial value for declared AudioParams
  },
  processorOptions: {        // arbitrary object passed to processor constructor
    mode: 'soft-clip',
  },
});
```

`parameterData` sets initial values for `AudioParam` objects declared via
`parameterDescriptors`. `processorOptions` is a plain object made available in the
processor's `constructor(options)` as `options.processorOptions`.

---

## Integration with Three.js

Three.js does not provide native AudioWorklet helpers. Wire worklets directly through
the listener's context and insert the node into the audio graph manually.

```js
const ctx  = listener.context;                        // AudioContext
const gain = listener.getInput();                     // master GainNode

await ctx.audioWorklet.addModule('worklets/reverb.js');

const reverb = new AudioWorkletNode(ctx, 'reverb');
gain.disconnect();
gain.connect(reverb);
reverb.connect(ctx.destination);
```

---

## Error Handling

```js
node.addEventListener('processorerror', (e) => {
  console.error('AudioWorkletProcessor threw:', e);
  // The processor outputs silence for the rest of its lifetime.
  // Re-create the node if recovery is needed.
});
```

---

## Sources

| URL | Tier | Accessed |
|---|---|---|
| https://developer.mozilla.org/en-US/docs/Web/API/ScriptProcessorNode | 1 (MDN) | 2026-05-26 |
| https://developer.mozilla.org/en-US/docs/Web/API/AudioWorklet | 1 (MDN) | 2026-05-26 |
| https://developer.mozilla.org/en-US/docs/Web/API/AudioWorkletNode | 1 (MDN) | 2026-05-26 |
| https://developer.mozilla.org/en-US/docs/Web/API/AudioWorkletProcessor/process | 1 (MDN) | 2026-05-26 |
| https://developer.mozilla.org/en-US/docs/Web/API/AudioWorkletGlobalScope/registerProcessor | 1 (MDN) | 2026-05-26 |
