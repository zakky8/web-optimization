# Chrome DevTools for WebGL Debugging

Chrome DevTools is the first-stop profiling tool for WebGL running in the browser. It does not give you draw-call-level inspection (use Spector.js for that), but it answers three critical questions: How many frames per second? Where is time going? How much GPU memory is in use?

---

## Frame Rendering Stats Overlay

The quickest way to see live FPS and GPU state without opening any panel.

### Enable via Command Menu

1. Open DevTools (`F12` or `Ctrl+Shift+I`).
2. Open Command Menu: `Ctrl+Shift+P` (Windows/Linux) / `Cmd+Shift+P` (Mac).
3. Type `Show frame rendering stats` and press Enter.

A small overlay appears in the **top-right corner of the viewport** (not inside DevTools).

### Enable via Rendering tab

1. Open DevTools → More tools (⋮) → **Rendering**.
2. Check **Frame rendering stats**.

### What the overlay shows

| Indicator | Meaning |
|---|---|
| FPS counter | Rolling real-time estimate of frames per second |
| Blue bars | Frames that were fully presented on time |
| Yellow bars | Partially presented frames (missed vsync but not dropped) |
| Red bars | Dropped frames — the GPU did not complete the frame in time |
| GPU raster | `on` / `off` — whether GPU rasterization is active for 2D layers |
| GPU memory | Current GPU memory usage in MB |

**Reading tip:** Sustained yellow + red bars with GPU memory climbing toward the device limit is the classic sign of a texture memory leak. Red bars with low GPU memory but high CPU usage point to JS being the bottleneck rather than the GPU.

---

## Performance Tab — GPU and Frame Profiling

### Recording a trace

1. DevTools → **Performance** tab.
2. Click the record button (circle) or press `Ctrl+E`.
3. Interact with the page for 3–10 seconds.
4. Stop recording.

### Key tracks for WebGL work

#### Frames track
Located near the top of the timeline. Each block is one rendered frame. Height indicates duration; tall blocks are slow frames. Hover a block to see:
- Frame duration in ms
- Estimated FPS for that frame
- Whether it was dropped

#### Main thread track
Shows the JS call stack over time. For WebGL apps, look for:
- Long `requestAnimationFrame` callbacks
- Shader compilation spikes (`compileShader`, `linkProgram`) — these are synchronous and block the main thread
- `getParameter` or `getUniform` calls mid-frame (driver readbacks stall the pipeline)

#### GPU track
The GPU track shows GPU-side work. It is not always visible by default:

1. After recording, look in the track list on the left for **GPU**.
2. If not present: click the gear icon → ensure "Advanced paint instrumentation" is enabled, then re-record.

The GPU track displays:
- Rasterization tasks
- Compositor frames
- Time the GPU was busy per frame

For WebGL, GPU work appears as a contiguous block per frame. A gap between the end of the JS frame and the start of GPU work indicates CPU-GPU synchronization latency. A GPU block that extends past the vsync boundary is your dropped frame culprit.

### Flame chart reading for WebGL apps

```
[Frame boundary]
  └─ requestAnimationFrame
       ├─ JS: update uniforms, matrix math
       ├─ gl.drawElements / gl.drawArrays  ← many small calls = CPU overhead
       └─ gl.flush / gl.finish             ← if present, driver sync stall
```

Seeing hundreds of `drawElements` calls in the flame chart means your scene needs draw call batching or instancing.

---

## Memory Tab — Heap and GPU Memory Tracking

### JS Heap Snapshot

DevTools → **Memory** → **Heap snapshot** → **Take snapshot**

Useful for catching:
- Leaked `WebGLTexture`, `WebGLBuffer`, `WebGLProgram` objects that were never `gl.delete*`'d
- Accumulated `Float32Array` / `ImageData` objects from per-frame allocations

Search the snapshot for `WebGL` to list all live WebGL objects. A growing count across multiple snapshots = leak.

### Allocation Timeline

DevTools → **Memory** → **Allocation instrumentation on timeline** → Record and interact → Stop.

Shows new allocations over time. Useful for spotting frame-by-frame allocations (e.g., `new Float32Array(...)` inside the render loop).

### GPU Memory in Performance Counter Pane

During or after a Performance recording:

1. In the recorded trace, look for the **Counter** pane (below the main flame chart).
2. Check the **GPU Memory** checkbox if it is not already enabled.

This plots GPU memory usage over the recording duration, letting you correlate memory growth with specific interactions or scene loads.

> Note: GPU Memory here is what the browser process reports; it does not distinguish between WebGL textures, buffers, and browser compositor memory. It is a useful trend indicator, not a precise VRAM breakdown.

---

## Layers Panel — Compositor Layer Inspection

### Open

DevTools → More tools (⋮) → **Layers**

### What it shows

The Layers panel visualizes the browser's compositor layer tree as a 3D diagram. Each layer is a separate composited surface.

| View | Use |
|---|---|
| 3D diagram | Drag to rotate and inspect layer stacking and z-ordering |
| Layer list | Click a layer to highlight it in the 3D view |
| Details panel | Size, compositing reason, paint count, memory estimate |

### Compositing reasons relevant to WebGL

- **Canvas2D / WebGL canvas** — `<canvas>` elements with a WebGL context are automatically promoted to their own compositor layer. Look for "Has a WebGL context" or "Explicit layer created from accelerated canvas" in the Compositing reason field.
- **will-change: transform** — Elements with this CSS property become separate layers; useful to confirm your canvas is isolated from DOM repaints.
- **Paint count** — A non-zero, incrementing paint count on a canvas layer is unexpected; the canvas should be painting via WebGL/GPU, not the 2D software rasterizer.

### What the Layers panel does NOT do

It does not profile draw calls, show shader state, or expose WebGL-specific metrics. Use it only to confirm that your canvas is running on a hardware-accelerated compositor layer and is not inadvertently triggering DOM repaint.

---

## Additional Rendering Tab Options

Beyond Frame rendering stats, the **Rendering** tab contains:

| Option | Use for WebGL |
|---|---|
| Paint flashing | Green flash on repaint. A WebGL canvas should NOT flash — if it does, the canvas is being software-rasterized |
| Layer borders | Orange/olive borders show compositor layer boundaries; cyan shows tiles. Confirm your canvas has its own border |
| Scrolling performance issues | Highlights scroll-blocking event listeners — relevant if your canvas captures scroll events |
| Layout shift regions | Purple highlight on CLS events — usually not WebGL-specific |

---

## Quick Reference: Shortcuts

| Action | Shortcut |
|---|---|
| Open DevTools | `F12` / `Ctrl+Shift+I` |
| Command Menu | `Ctrl+Shift+P` |
| Frame rendering stats | Command Menu → "Show frame rendering stats" |
| Start/stop Performance recording | `Ctrl+E` |
| Toggle Rendering tab | More tools → Rendering |

---

*Sources: https://developer.chrome.com/docs/devtools/rendering/performance (accessed 2026-05-26), https://developer.chrome.com/docs/devtools/memory-problems (accessed 2026-05-26), https://developer.chrome.com/docs/devtools/layers (accessed 2026-05-26)*
