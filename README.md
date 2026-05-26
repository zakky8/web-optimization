# Three.js & WebGL Performance Optimization — Complete Reference 2026

> The most comprehensive Three.js, WebGL, and WebGPU performance knowledge base on GitHub.
> 48 topic folders. Every claim sourced and validated against live repos and browser specs.

[![Stars](https://img.shields.io/github/stars/zakky8/web-optimization?style=flat-square&color=yellow)](https://github.com/zakky8/web-optimization/stargazers)
[![Last Commit](https://img.shields.io/github/last-commit/zakky8/web-optimization?style=flat-square)](https://github.com/zakky8/web-optimization/commits/main)
[![License: MIT](https://img.shields.io/badge/License-MIT-blue.svg?style=flat-square)](LICENSE)
[![Topics](https://img.shields.io/badge/topics-threejs%20webgl%20webgpu%20r3f%20gsap-informational?style=flat-square)](https://github.com/zakky8/web-optimization)

**Keywords:** Three.js optimization · WebGL performance · React Three Fiber · WebGPU · GLSL shaders · GPGPU particles · GSAP animation · Canvas 2D · Web performance · Core Web Vitals · mobile 3D · TypeScript Three.js

---

## What Is This?

A battle-tested reference built from primary sources — official docs, Awwwards SOTD/SOTY winners,
production benchmarks, and live repo audits. Covers every layer of the 3D web stack from raw WebGL
extension detection to high-level R3F reconciler internals, validated against browser specs as of
May 2026.

**Who it's for:** developers building award-quality 3D websites, creative technologists optimizing
WebGL scenes for mobile, engineers integrating Three.js r184 with React 19 / Next.js 15 / Astro 6.

---

## Table of Contents

- [Foundations](#foundations)
- [GPU & Shaders](#gpu--shaders)
- [Build & Pipeline](#build--pipeline)
- [Frameworks & Rendering](#frameworks--rendering)
- [Concurrency & Memory](#concurrency--memory)
- [CSS & Platform](#css--platform)
- [Three.js Deep Dive](#threejs-deep-dive)
- [Animation & QA](#animation--quality-assurance)
- [TypeScript & Advanced Frameworks](#typescript--frameworks-advanced)
- [Visual Effects & Post-processing](#visual-effects--post-processing)
- [XR, Audio & Media](#xr-audio--media)
- [2D & Typography](#2d--typography)
- [Pipeline & Tooling](#pipeline--tooling)
- [Camera & Interaction](#camera--interaction)
- [Validation & Corrections](#validation--corrections)
- [Quick Start](#quick-start)
- [Key Numbers](#key-numbers-validated)
- [Library Versions](#library-versions-2026)
- [Key Corrections](#key-corrections)
- [Top Sites to Study](#top-sites-to-study)

---

## Foundations

Essential Three.js performance fundamentals, mobile limits, and Core Web Vitals for 3D.

| Folder | What's Inside |
|--------|---------------|
| [`01-3d-animation/`](01-3d-animation/) | Rendering pipeline, draw calls, textures, shaders, scroll, canvas setup |
| [`02-mobile/`](02-mobile/) | iOS/Android GPU limits, thermal throttling, context loss, touch events, network |
| [`03-performance-metrics/`](03-performance-metrics/) | Real benchmark numbers, Core Web Vitals for 3D, FPS targets |
| [`04-resources/`](04-resources/) | Case studies, reading list, primary sources, Awwwards analysis |
| [`05-checklists/`](05-checklists/) | Ship-ready checklists for desktop and mobile 3D |

---

## GPU & Shaders

WebGPU renderer, GLSL noise, GPGPU compute shaders, and shader authoring tools.

| Folder | What's Inside |
|--------|---------------|
| [`06-webgpu/`](06-webgpu/) | WebGPURenderer setup, TSL node system, WGSL, render bundles, GPU-driven rendering |
| [`07-shaders/`](07-shaders/) | GLSL noise (simplex/FBM/Worley), domain warping, lygia library, PBR injection |
| [`48-gpgpu-particles/`](48-gpgpu-particles/) | GPUComputationRenderer, ping-pong buffers, compute shaders (262K+ particles) |

---

## Build & Pipeline

Vite configuration for 3D, glTF/KTX2/Draco compression, and CI/CD for WebGL projects.

| Folder | What's Inside |
|--------|---------------|
| [`08-physics-interaction/`](08-physics-interaction/) | Rapier.js, cannon-es, raycasting, physics-aware drag |
| [`09-build-pipeline/`](09-build-pipeline/) | gltfjsx CLI, Vite 3D config, texture compression (KTX2/Draco/Meshopt), CI/CD |

---

## Frameworks & Rendering

React Three Fiber, drei, TresJS, Threlte, page transitions, accessibility, and advanced techniques.

| Folder | What's Inside |
|--------|---------------|
| [`10-frameworks/`](10-frameworks/) | R3F fundamentals, drei helpers, Zustand, TresJS (Vue), Threlte (Svelte) |
| [`11-page-transitions/`](11-page-transitions/) | Barba.js persistent canvas, View Transitions API, SceneManager dispose |
| [`12-accessibility-seo/`](12-accessibility-seo/) | ARIA for canvas, WCAG animation, Core Web Vitals for 3D |
| [`13-advanced-techniques/`](13-advanced-techniques/) | GPGPU ping-pong, MRT G-Buffer, SSAO, post-processing chains |
| [`14-caching-cdn/`](14-caching-cdn/) | Cache strategies, service workers, Cloudflare, content-hashed assets |
| [`15-animation-formats/`](15-animation-formats/) | Lottie (dotLottie), Rive state machines, Canvas 2D, SVG SMIL/CSS/MorphSVG |

---

## Concurrency & Memory

Web Workers, OffscreenCanvas, SharedArrayBuffer, and Three.js memory leak prevention.

| Folder | What's Inside |
|--------|---------------|
| [`16-workers-parallel/`](16-workers-parallel/) | OffscreenCanvas Three.js, SharedArrayBuffer + Atomics, worker pool, Comlink, PWA/Workbox |
| [`17-memory-management/`](17-memory-management/) | dispose() pattern, renderer.info ground truth, common leaks (r180-r183 fixes) |

---

## CSS & Platform

RenderingNG, CSS Houdini, scroll-driven animations, and WebAssembly physics.

| Folder | What's Inside |
|--------|---------------|
| [`18-css-performance/`](18-css-performance/) | RenderingNG compositor thread, View Transitions API, CSS Houdini, scroll-driven animations |
| [`19-wasm-physics/`](19-wasm-physics/) | Rapier + Vite WASM config, cannon-es, fixed timestep, physics worker pattern |

---

## Three.js Deep Dive

Three.js render pipeline internals, WebGLPrograms cache, BatchedMesh, and open-source reference repos.

| Folder | What's Inside |
|--------|---------------|
| [`20-threejs-internals/`](20-threejs-internals/) | Render pipeline internals, WebGLPrograms cache, shader explosion, BatchedMesh, MRT deferred |
| [`21-open-source-repos/`](21-open-source-repos/) | Award-winning portfolios (Bruno Simon, HamishMW), pmndrs ecosystem, learning resources |

---

## Animation & Quality Assurance

GSAP ScrollTrigger, Lenis smooth scroll, Framer Motion, and validated technical claims.

| Folder | What's Inside |
|--------|---------------|
| [`22-animation-engines/`](22-animation-engines/) | GSAP advanced (ScrollTrigger, SplitText, MotionPath), Lenis v1.3.23, Framer Motion |
| [`23-validation-notes/`](23-validation-notes/) | Corrected technical claims, repo status 2026, verified browser support tables |

---

## TypeScript & Frameworks (Advanced)

TypeScript + Three.js type safety, Next.js 15 + R3F, React 19, Astro 6, and R3F performance patterns.

| Folder | What's Inside |
|--------|---------------|
| [`24-typescript-threejs/`](24-typescript-threejs/) | Strict TypeScript + Three.js r184, typed loaders, custom material types, shader type safety |
| [`25-nextjs-r3f/`](25-nextjs-r3f/) | Next.js 15 + R3F v9, dynamic import, App Router SSR patterns, React 19 compatibility |
| [`43-react-patterns/`](43-react-patterns/) | React 19 concurrent patterns, Suspense boundaries, useTransition for 3D, R3F v9 hooks |
| [`44-astro-threejs/`](44-astro-threejs/) | Astro 6 + Three.js, island architecture, ClientRouter, content collections for 3D assets |
| [`45-r3f-performance/`](45-r3f-performance/) | R3F useFrame optimization, instancing via drei, reconciler internals |

---

## Visual Effects & Post-processing

SSAO, Bloom, DOF, LUT, PBR materials, instancing, GPU picking, and shader dev tooling.

| Folder | What's Inside |
|--------|---------------|
| [`26-postprocessing/`](26-postprocessing/) | SSAO, Bloom, DOF, LUT, EffectComposer chains, postprocessing v7 vs three stdlib |
| [`29-shadows-advanced/`](29-shadows-advanced/) | PCF/PCSS/CSM, baked lightmaps, Three.js r168+ shadow deprecations |
| [`30-instancing-gpu/`](30-instancing-gpu/) | InstancedMesh, BatchedMesh r155+, GPU picking, frustum culling, LOD |
| [`42-advanced-materials/`](42-advanced-materials/) | PBR deep-dive, custom ShaderMaterial, MeshPhysicalMaterial, transmission, anisotropy |
| [`46-shader-tools/`](46-shader-tools/) | ShaderForge, glslify, lygia, vite-plugin-glsl, hot-reload GLSL dev workflow |

---

## XR, Audio & Media

WebXR immersive sessions, Web Audio API spatial sound, and video texture streaming.

| Folder | What's Inside |
|--------|---------------|
| [`27-webxr/`](27-webxr/) | WebXR Device API, immersive-ar/vr sessions, Safari NO support confirmed, polyfills |
| [`28-web-audio/`](28-web-audio/) | Web Audio API spatial audio, AudioWorklet, Three.js PositionalAudio, iOS unlock |
| [`34-video-streaming/`](34-video-streaming/) | Video textures, HLS.js, adaptive bitrate, VideoFrame API, WebCodecs |

---

## 2D & Typography

SVG animation, Canvas 2D performance, variable fonts, and kinetic typography.

| Folder | What's Inside |
|--------|---------------|
| [`31-svg-animation/`](31-svg-animation/) | GSAP MorphSVG (now free), SMIL (not deprecated), SVG filters, DrawSVG |
| [`32-canvas2d-performance/`](32-canvas2d-performance/) | OffscreenCanvas, Canvas 2D compositing, ImageBitmap, pixel manipulation |
| [`33-typography-kinetic/`](33-typography-kinetic/) | Variable fonts, SplitText v3 (free), kinetic typography, scroll-driven text |

---

## Pipeline & Tooling

Blender 4.x glTF export, BVH scene optimization, WebGL debugging, extensions, and asset loading.

| Folder | What's Inside |
|--------|---------------|
| [`35-blender-pipeline/`](35-blender-pipeline/) | glTF export settings, KTX2 baking, Blender 4.x scripts, normal map workflows |
| [`36-scene-optimization/`](36-scene-optimization/) | BVH raycasting, draw call batching, occlusion culling, scene graph flattening |
| [`37-webgl-debugging/`](37-webgl-debugging/) | Spector.js, Chrome WebGL Inspector, gl.getError patterns, shader compile errors |
| [`38-webgl-extensions/`](38-webgl-extensions/) | EXT_color_buffer_float, OES_texture_float, WEBGL_draw_buffers, extension feature detection |
| [`39-asset-loading/`](39-asset-loading/) | GLTFLoader, KTX2Loader, DRACOLoader, progressive loading, asset caching strategies |
| [`40-error-recovery/`](40-error-recovery/) | WebGL context loss/restore, graceful degradation, fallback pipelines |

---

## Camera & Interaction

camera-controls, orbit controls, cinematic transitions, and pointer event handling.

| Folder | What's Inside |
|--------|---------------|
| [`41-camera-systems/`](41-camera-systems/) | camera-controls v3.1.2, orbit controls, cinematic transitions, pointer events |

---

## Validation & Corrections

Sourced corrections to widely-repeated wrong claims across Three.js and WebGL documentation.

| Folder | What's Inside |
|--------|---------------|
| [`47-validation-round2/`](47-validation-round2/) | GSAP licensing, SMIL status, BVH claims, CSM repos, PCFSoftShadowMap deprecation |

---

## Quick Start

**Fast project audit — start here:**
```
05-checklists/desktop-checklist.md
05-checklists/mobile-checklist.md
```

**New 3D project setup:**
```
09-build-pipeline/vite-config-3d.md        ← Vite + Three.js config
09-build-pipeline/texture-compression.md   ← KTX2 / Draco / Meshopt
24-typescript-threejs/                     ← Type-safe Three.js
25-nextjs-r3f/                             ← Next.js 15 + R3F v9
```

**Performance bottleneck?**
```
03-performance-metrics/                    ← Benchmarks + Core Web Vitals
17-memory-management/                      ← dispose() and leak detection
36-scene-optimization/                     ← Draw calls, BVH, culling
20-threejs-internals/                      ← WebGLPrograms, BatchedMesh
```

**Mobile performance:**
```
02-mobile/                                 ← iOS/Android limits
19-wasm-physics/                           ← Physics on a worker
16-workers-parallel/                       ← OffscreenCanvas patterns
```

---

## Key Numbers (Validated)

| Number | Rule | Source |
|--------|------|--------|
| **< 100** | Draw calls for 60 fps on mobile | Verified — Three.js community benchmarks |
| **<= 2.0x** | Max DPR on mobile | Community convention — renderer.setPixelRatio |
| **4-8x** | VRAM reduction from KTX2 ETC1S/UASTC | Verified — Khronos KTX2 spec |
| **70-98%** | Draco geometry compression ratio | Verified — Google Draco benchmarks |
| **70-90%** | gltfjsx scene size reduction | Verified — pmndrs/gltfjsx docs |
| **256 MB** | iOS WebGL safe memory budget | Device-dependent — WebKit limits |
| **r168** | PCFSoftShadowMap deprecated (use PCFShadowMap) | Verified — Three.js CHANGELOG |

---

## Library Versions (2026-05-26)

| Library | Version | Notes |
|---------|---------|-------|
| [three.js](https://threejs.org) | **r184 (0.184.0)** | Active |
| [react-three-fiber](https://github.com/pmndrs/react-three-fiber) | **v9.6.1** | Requires React 19 |
| [drei](https://github.com/pmndrs/drei) | **v10.7.7** | Active |
| [lenis](https://github.com/darkroomengineering/lenis) | **v1.3.23** | Active |
| [GSAP](https://gsap.com) | **3.x** | Now 100% free — DrawSVG/MorphSVG/SplitText included |
| [camera-controls](https://github.com/yomotsu/camera-controls) | **v3.1.2** | Active |
| [Astro](https://astro.build) | **6.3.8** | `<ViewTransitions />` → `<ClientRouter />` in v5+ |
| [detect-gpu](https://github.com/TimvanScherpenzeel/detect-gpu) | **v5.x** | Active |

Full status (including dead/marginal repos): [`23-validation-notes/repo-status-2026.md`](23-validation-notes/repo-status-2026.md)

---

## Key Corrections

Widely-repeated claims in tutorials that are wrong or outdated:

| Claim | Reality | Source |
|-------|---------|--------|
| GSAP Club plugins cost money | **No — 100% free since 2025** | gsap.com pricing page |
| SMIL is deprecated | **No — Chrome reversed deprecation in 2019** | Chrome Status |
| PCFSoftShadowMap is the soft shadow type | **Deprecated in r168** — use PCFShadowMap | Three.js CHANGELOG |
| Safari supports WebXR | **No support** on macOS, iOS, iPadOS | MDN browser compat |
| R3F v8 works with React 19 | **Incompatible** — use R3F v9 | pmndrs/react-three-fiber |
| three-mesh-bvh gives 100x speedup | **Unverified** — not in README | gkjohnson/three-mesh-bvh |

Full corrections: [`47-validation-round2/`](47-validation-round2/) and [`23-validation-notes/claim-corrections.md`](23-validation-notes/claim-corrections.md)

---

## Top Sites to Study

| Site | Award | Why It Matters |
|------|-------|---------------|
| [bruno-simon.com](https://bruno-simon.com) | Awwwards SOTM | Three.js portfolio benchmark |
| [lusion.co](https://lusion.co) | Awwwards SOTY 2024 | Studio engine architecture |
| [igloo-inc.com](https://igloo-inc.com) | Awwwards SOTY 2025 | Latest production patterns |
| [365.cartier.com](https://365.cartier.com) | Enterprise | High-load 3D on production scale |
| [activetheory.net](https://activetheory.net) | Studio | Custom engine benchmark |

Technical breakdowns: [`21-open-source-repos/award-winning-portfolios.md`](21-open-source-repos/award-winning-portfolios.md)

---

## Contributing

Found an error or outdated claim? Open an issue or PR. All claims must reference a primary source
(official docs, changelog, browser spec, or live repo). Affiliate review sites are not accepted
as sources. See [`47-validation-round2/`](47-validation-round2/) for the sourcing standard used
throughout this repo.

---

*Last validated: 2026-05-26 | three.js r184 | browser specs cross-checked against MDN, Chrome Status, WebKit Bugzilla*
