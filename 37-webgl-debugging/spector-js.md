# Spector.js

**GitHub:** https://github.com/BabylonJS/Spector.js  
**Current version:** v0.9.27 (released 2021-09-26)  
**npm package:** `spectorjs`  
**Status:** Functionally stable; last release 2021-09-26; 544 commits, 1.6 k stars. No active feature development as of 2026-05-26, but the library works reliably against WebGL 1 and WebGL 2.

---

## What Spector.js Does

Spector.js intercepts every WebGL call made during a single frame and records:

- The full ordered list of draw calls with their state snapshots
- All bound textures, framebuffers, and buffer contents at each draw call
- Shader source (vertex + fragment) attached to each program
- Uniform and attribute values at time of capture
- Visual output after each draw call (rendered to an off-screen canvas)

It is engine-agnostic — works with Three.js, Babylon.js, raw WebGL, PixiJS, etc.

---

## Installation

### CDN (quickest for debugging a live site)

```html
<script src="https://cdn.jsdelivr.net/npm/spectorjs@0.9.30/dist/spector.bundle.js"></script>
<script>
  const spector = new SPECTOR.Spector();
  spector.displayUI();   // shows the capture button overlay
</script>
```

> Note: CDN references v0.9.30, which is higher than the tagged GitHub release v0.9.27. The npm registry and CDN may carry a patch version not formally tagged. Always verify with `npm show spectorjs version`.

### npm / webpack

```bash
npm install spectorjs
```

```js
import { Spector } from "spectorjs";

const spector = new Spector();
spector.displayUI();
```

### Browser Extension

Available for Chrome and Firefox. Install from the respective extension stores; search "Spector.js". The extension injects itself into any page that creates a WebGL context — no code changes needed.

---

## Frame Capture

### Via UI overlay

1. Load your page with the CDN/npm bundle or browser extension active.
2. Click the Spector icon (red circle with "S") that appears in the top-right corner.
3. Press **Start capture**. Spector waits for the next `requestAnimationFrame` tick.
4. After one frame completes, capture stops automatically and the result panel opens.

### Programmatic capture

```js
const spector = new SPECTOR.Spector();

// Capture the next N frames (default 1)
spector.captureCanvas(document.getElementById("myCanvas"), 1 /* frameCount */);

spector.onCapture.add((capture) => {
  console.log(capture);           // full JSON capture object
  spector.displayUI();            // open the viewer
  spector.showCapture(capture);   // display this specific capture
});
```

### Capture of an OffscreenCanvas (worker context)

```js
// Inside the worker
importScripts("spector.bundle.js");
const spector = new SPECTOR.Spector();
spector.captureCanvas(offscreenCanvas);
```

---

## Reading the Capture Timeline

The timeline panel is the central UI after a capture. Layout:

```
[Command list — left panel]    [Visual output — center]    [State / details — right panel]
```

- **Command list:** Every WebGL call in execution order, e.g. `bindTexture`, `useProgram`, `drawElements`. Clicking a command jumps the visual output to what the frame looked like *after* that command executed.
- **Visual output (scrubber):** Lets you step through the frame command-by-command. A thumbnail strip across the top shows visual checkpoints. Red thumbnails signal a draw call; grey are state changes only.
- **State panel:** Shows the full WebGL state at the selected command — active texture units, bound buffers, blend modes, depth test, stencil state, viewport, etc.

Reading tip: jump to the *last* draw call to confirm the final image, then scrub backwards to find where something unexpected first appears or disappears.

---

## Draw Call Inspector

Click any `drawArrays` or `drawElements` command to see:

| Field | What it shows |
|---|---|
| Program | Which shader program was active |
| Attributes | Buffer bindings, stride, offset, type for each `in` variable |
| Uniforms | Current values of all uniforms at time of draw |
| Textures | All texture units with thumbnail previews |
| Draw parameters | `mode`, `count`, `offset`, `instances` |

---

## Texture Viewer

Every texture bound at any draw call is captured as a snapshot. The texture panel shows:

- Full-resolution image preview with mip level selector
- Internal format (e.g. `RGBA8`, `DEPTH24_STENCIL8`, `SRGB8_ALPHA8`)
- Width × height × depth (for 3D/array textures)
- Wrap mode (S/T) and min/mag filters
- Whether mipmaps were generated

SRGB textures are decoded for display so they appear perceptually correct (supported since v0.9.26).

---

## Shader Source View

Click any draw call, then select the **Shader** tab in the right panel:

- **Vertex source** and **Fragment source** are shown with syntax highlighting.
- Spector captures the source at `shaderSource()` call time, so if your build step inlines GLSL you see the final processed source, not the original file.
- Uniforms panel cross-references the uniform names visible in the shader against actual values at draw time — fast way to spot a uniform that was never set.

---

## Tagging Objects for Readability

By default, textures and buffers appear as `WebGLTexture [id=3]`. Add custom tags so captures are readable:

```js
// Requires the SPECTOR extension object injected by the lib
spector.setMarker("Shadow map pass");          // timeline marker
canvas.__SPECTOR_Metadata = { name: "Main canvas" };

// Tag a WebGL object
const tex = gl.createTexture();
spector.addTextureMetadata(tex, { name: "Diffuse albedo" });
```

---

## Export / Import Captures

Captures can be exported as JSON for sharing or archival:

```js
spector.onCapture.add((capture) => {
  const json = JSON.stringify(capture);
  // save to file, send to server, etc.
});

// Re-load a saved capture
spector.loadCapture(savedJsonObject);
```

---

## Limitations (as of v0.9.27)

- No support for WebGPU contexts.
- Compressed texture *contents* cannot be read back by the browser; dimensions and format are captured but pixel data shows as empty.
- Performance overhead during capture is significant — not suitable for real-time profiling; use only for single-frame inspection.
- Captures can be very large (hundreds of MB) on texture-heavy scenes.

---

*Sources: https://github.com/BabylonJS/Spector.js (accessed 2026-05-26), https://spector.babylonjs.com/ (accessed 2026-05-26)*
