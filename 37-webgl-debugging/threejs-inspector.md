# Three.js Inspector & Performance Overlays

---

## 1. Three.js Editor (editor.threejs.org)

**URL:** https://editor.threejs.org/  
**Source:** https://github.com/mrdoob/three.js/tree/dev/editor  
**Status:** Actively maintained as part of the Three.js monorepo.

The Three.js Editor is a browser-based scene editor and inspector. It is useful for debugging an existing scene when you can serialize it, or for building test scenes to isolate a problem.

### Key debugging-relevant features

| Panel | What it shows |
|---|---|
| Scene graph (left) | Full object hierarchy; click any mesh, light, camera, or group to select it |
| Properties (right) | Transform (position, rotation, scale), material settings, geometry info, visibility toggle |
| Geometry stats | Vertex count, triangle count, morph targets per selected object |
| Material inspector | Texture slots, color pickers, roughness/metalness/etc. — changes apply live |
| Script editor | Attach and run JavaScript on scene objects; useful for testing logic in isolation |
| Viewport | Orbit/pan/zoom; wireframe toggle, grid, helpers |

### Loading your own scene

```js
// Export from your app at runtime, paste into editor
const exporter = new THREE.ObjectExporter();  // or GLTFExporter
const json = scene.toJSON();
```

Alternatively, export as GLTF and use File → Import in the editor.

### Limitations for debugging

The editor operates on a *copy* of the scene — it does not attach to a live running page. For inspecting a live Three.js context you need one of the tools below.

---

## 2. three-devtools (Browser Extension)

**GitHub:** https://github.com/threejs/three-devtools  
**Status: ARCHIVED** — repository archived 2022-12-12, read-only, alpha/experimental.

three-devtools was an official Three.js DevTools extension for Chrome and Firefox. It surfaced a dedicated "Three.js" tab in DevTools showing the live scene graph, object properties, and texture previews.

**Do not use for new projects.** The extension has not been updated for Three.js r150+ and will likely fail to attach to modern builds that use ES modules without the `three/examples/jsm` import paths the extension expected.

> UNVERIFIED whether a fork with current compatibility exists as of 2026-05-26.

---

## 3. r3f-perf — Performance Overlay for React Three Fiber

**GitHub:** https://github.com/utsuboco/r3f-perf  
**Status:** Actively maintained; 775 stars, 38 forks, 307 commits as of 2026-05-26.  
**Use case:** React Three Fiber (R3F) projects only.

### Install

```bash
# npm
npm install --save-dev r3f-perf

# yarn
yarn add --dev r3f-perf
```

### Basic usage

```jsx
import { Canvas } from "@react-three/fiber";
import { Perf } from "r3f-perf";

function App() {
  return (
    <Canvas>
      <Perf />
      {/* your scene */}
    </Canvas>
  );
}
```

The `<Perf />` component renders a HUD overlay in the top-left corner (position configurable).

### Metrics displayed

| Metric | Notes |
|---|---|
| FPS | Live rolling average |
| GPU | Frame GPU time in ms (requires `EXT_disjoint_timer_query_webgl2`) |
| CPU | JS main-thread time per frame in ms |
| Memory | JS heap usage in MB |
| GL programs | Number of compiled shader programs (needs `deepAnalyze` prop) |
| Matrix updates | `matrixWorld` computations per frame (needs `matrixUpdate` prop) |

### Props

```jsx
<Perf
  position="top-left"     // "top-left" | "top-right" | "bottom-left" | "bottom-right"
  deepAnalyze={true}      // count GL programs (adds slight overhead)
  matrixUpdate={true}     // count matrixWorld updates
  minimal={false}         // compact single-line mode
/>
```

### Conditional render (dev only)

```jsx
{process.env.NODE_ENV === "development" && <Perf />}
```

### GPU timing caveat

GPU time requires the `EXT_disjoint_timer_query_webgl2` extension, which Chrome enables by default. Firefox may not expose it. When unavailable, the GPU column shows `N/A`.

---

## 4. drei Perf Component

**Package:** `@react-three/drei`  
**GitHub:** https://github.com/pmndrs/drei  

`@react-three/drei` re-exports r3f-perf as a convenience component. If you already have drei installed there is no need to add r3f-perf as a separate dependency.

```jsx
import { Perf } from "@react-three/drei";

// Usage is identical to r3f-perf directly
<Canvas>
  <Perf position="top-left" />
</Canvas>
```

Under the hood, drei wraps r3f-perf — the metrics, props, and GPU timing caveats are the same as documented in section 3 above.

---

## Comparison Table

| Tool | Live attach | R3F required | Archived |
|---|---|---|---|
| Three.js Editor | No (import only) | No | No |
| three-devtools extension | Yes | No | **Yes (2022)** |
| r3f-perf | Yes | **Yes** | No |
| drei `<Perf>` | Yes | **Yes** | No |

---

*Sources: https://github.com/threejs/three-devtools (accessed 2026-05-26), https://github.com/utsuboco/r3f-perf (accessed 2026-05-26), https://threejs.org/editor/ (accessed 2026-05-26)*
