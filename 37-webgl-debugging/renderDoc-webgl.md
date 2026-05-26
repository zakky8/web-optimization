# RenderDoc for WebGL

**Website:** https://renderdoc.org/  
**GitHub:** https://github.com/baldurk/renderdoc  
**Current version:** v1.44 (released 2026-05-01)  
**License:** MIT  
**Status:** Actively maintained; wide industry adoption (Unity, Unreal Engine integrations).

---

## WebGL Support Status

**RenderDoc does NOT natively support WebGL as of v1.44.**

RenderDoc's supported graphics APIs are:
- Vulkan
- Direct3D 11
- Direct3D 12
- OpenGL (desktop, Windows/Linux)
- OpenGL ES (Android)

WebGL (the browser-sandboxed subset of OpenGL ES) is not in that list. The browser process owns the OpenGL/Vulkan context and does not expose it for external injection by RenderDoc the way a native executable does.

> UNVERIFIED: Some community reports describe attaching RenderDoc to Chrome's GPU process via its OpenGL backend on Windows, using `--use-gl=desktop` Chrome flags. This is an unsupported, undocumented workflow and has not been verified as of 2026-05-26.

---

## When RenderDoc IS Useful for Web/WebGL Work

### 1. Native engine reference builds
If you are developing a WebGL renderer that mirrors a native C++ engine (e.g., debugging why your WebGL port looks different from the native build), capture the native build in RenderDoc to see ground-truth pipeline state, then compare against Spector.js captures from the browser.

### 2. WebGL via Emscripten / WebAssembly
Applications compiled with Emscripten's OpenGL ES layer (`-s USE_WEBGL2=1`) can sometimes be profiled in their *native* (non-browser) build using RenderDoc against the OpenGL ES backend. This requires a separate native build target.

### 3. WebXR / AR applications
VR/AR runtimes that accept OpenGL ES contexts may be profilable with RenderDoc depending on the platform and runtime.

---

## RenderDoc Core Capabilities (for reference when applicable)

### Frame Capture

Launch application via RenderDoc (File → Launch Application), or inject into a running process (File → Inject Into Process). Press `F12` or the capture key in-app to capture a single frame.

Key options:
- **Capture key:** configurable; default `Print Screen` or `F12`
- **Capture delay:** delay N frames before capturing (to reach a steady state)
- **Capture count:** capture N consecutive frames

### Frame Debugger — Panels

| Panel | What it shows |
|---|---|
| Event Browser | Full ordered list of API calls for the frame (equivalent to Spector.js command list) |
| API Inspector | Arguments and return values for the selected API call |
| Texture Viewer | Any texture resource at any point in the frame; slice/mip/channel selectors |
| Mesh Viewer | Input/output vertex data for draw calls; geometry visualizer |
| Pipeline State | Full pipeline state at selected draw call: input assembler, shaders, rasterizer, output merger |
| Shader Viewer | SPIR-V or GLSL source with live variable inspection during shader debugging |
| Pixel History | For a selected pixel: list of every draw call that touched it and what color it contributed |

### Pipeline State Viewer

The Pipeline State panel shows the complete API state for a draw call:

```
Input Assembler → Vertex Shader → [Tessellation] → [Geometry Shader] → Rasterizer → Fragment Shader → Output Merger
```

Each stage is clickable. At the Fragment Shader stage you can see:
- Bound shader program (with link to source)
- All uniform values
- All bound texture units
- Blend state, depth/stencil state

This is the RenderDoc equivalent of reading through a Spector.js state panel, but with deeper hardware-level detail (e.g., buffer layout in GPU memory, actual resource descriptors).

### Shader Debugging

RenderDoc supports stepping through shader execution line by line for Vulkan and D3D12. For OpenGL, shader debugging support is more limited and driver-dependent.

---

## RenderDoc vs Spector.js — Decision Matrix

| Criterion | Spector.js | RenderDoc |
|---|---|---|
| Works with WebGL in browser | **Yes** | No |
| Works with native OpenGL/Vulkan | No | **Yes** |
| Setup complexity | Low (CDN script or extension) | Medium (install + launch app via RenderDoc) |
| Draw call timeline | Yes | **Yes (richer)** |
| Pixel history | No | **Yes** |
| Line-by-line shader debugging | No | Yes (Vulkan/D3D12; limited OpenGL) |
| Texture viewer | Yes | **Yes (more formats)** |
| Mesh/geometry viewer | No | **Yes** |
| Pipeline state viewer | Partial (WebGL state) | **Yes (full hardware state)** |
| Works without source code changes | Via browser extension | Via process injection |
| Platform | Browser | Windows, Linux, Android |

**Summary:** For any WebGL application running in Chrome or Firefox, use Spector.js. RenderDoc is the right tool if you have a native build, are targeting Android OpenGL ES, or need pixel history and line-by-line shader stepping that Spector.js cannot provide.

---

## Installation (for native use)

Download from https://renderdoc.org/builds — available as:
- Windows installer (`.msi`)
- Windows portable (`.zip`)
- Linux `.tar.gz`
- Source on GitHub

No browser plugin or extension is available or officially supported.

---

## Community Workaround: Chrome GPU Process Capture (UNVERIFIED)

Some developers report the following workflow to capture Chrome's GPU process on Windows:

1. Launch Chrome with `--use-gl=desktop` (disables Chrome's default ANGLE abstraction layer, using native OpenGL instead).
2. In RenderDoc, use File → Inject Into Process and select the Chrome GPU process (`chrome.exe` with `--type=gpu-process`).
3. Capture frames as normal.

**Caveats:**
- Officially unsupported by both Google and the RenderDoc team.
- The `--use-gl=desktop` flag is deprecated and may be removed from Chrome.
- Captured state may not map cleanly to WebGL spec behavior due to ANGLE's translation layer being bypassed.
- Does not work on macOS (Metal backend) or with Chrome's Vulkan backend.

This workflow is documented here for awareness. Prefer Spector.js for any browser WebGL debugging.

---

*Sources: https://renderdoc.org/ (accessed 2026-05-26 — v1.44 confirmed). RenderDoc WebGL support status: direct inspection of supported API list from renderdoc.org. Community workaround: UNVERIFIED, no primary source confirmed as of 2026-05-26.*
