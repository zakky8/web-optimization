# Web Optimization — Top 1% Reference

A comprehensive knowledge base for building industry-level 3D and animation websites.
All data sourced from primary sources: official docs, Awwwards SOTD winners, production benchmarks,
and validated against live repos and browser specs (2025-2026).

## Folders

### Foundations
| Folder | What's Inside |
|--------|---------------|
| `01-3d-animation/` | Rendering pipeline, textures, shaders, scroll, canvas setup |
| `02-mobile/` | iOS/Android limits, thermal, context loss, touch, network |
| `03-performance-metrics/` | Real benchmark numbers, Core Web Vitals |
| `04-resources/` | Case studies, reading list, primary sources |
| `05-checklists/` | Ship-ready checklists for desktop and mobile |

### GPU & Shaders
| Folder | What's Inside |
|--------|---------------|
| `06-webgpu/` | WebGPURenderer setup, TSL node system, WGSL, render bundles, GPU-driven rendering |
| `07-shaders/` | GLSL noise (simplex/FBM/Worley), domain warping, lygia library, surface shaders, PBR injection |
| `48-gpgpu-particles/` | GPUComputationRenderer, ping-pong buffers, compute shaders (262K+ particles) |

### Build & Pipeline
| Folder | What's Inside |
|--------|---------------|
| `08-physics-interaction/` | Rapier.js, cannon-es, raycasting, physics-aware drag |
| `09-build-pipeline/` | gltfjsx CLI, Vite 3D config, texture compression (KTX2/Draco/Meshopt), CI/CD |

### Frameworks & Rendering
| Folder | What's Inside |
|--------|---------------|
| `10-frameworks/` | R3F fundamentals, drei helpers, Zustand, TresJS (Vue), Threlte (Svelte) |
| `11-page-transitions/` | Barba.js persistent canvas, View Transitions API, SceneManager dispose |
| `12-accessibility-seo/` | ARIA for canvas, WCAG animation, Core Web Vitals for 3D |
| `13-advanced-techniques/` | GPGPU ping-pong, MRT G-Buffer, SSAO, post-processing chains |
| `14-caching-cdn/` | Cache strategies, service workers, Cloudflare, content-hashed assets |
| `15-animation-formats/` | Lottie (dotLottie), Rive state machines, Canvas 2D, SVG SMIL/CSS/MorphSVG |

### Concurrency & Memory
| Folder | What's Inside |
|--------|---------------|
| `16-workers-parallel/` | OffscreenCanvas Three.js, SharedArrayBuffer + Atomics, worker pool, Comlink, PWA/Workbox |
| `17-memory-management/` | dispose() pattern, renderer.info ground truth, common leaks (r180-r183 fixes) |

### CSS & Platform
| Folder | What's Inside |
|--------|---------------|
| `18-css-performance/` | RenderingNG compositor thread, View Transitions API, CSS Houdini, scroll-driven animations |
| `19-wasm-physics/` | Rapier + Vite WASM config, cannon-es, fixed timestep, physics worker pattern |

### Three.js Deep Dive
| Folder | What's Inside |
|--------|---------------|
| `20-threejs-internals/` | Render pipeline internals, WebGLPrograms cache, shader explosion, BatchedMesh, MRT deferred |
| `21-open-source-repos/` | Award-winning portfolios (Bruno Simon, HamishMW), pmndrs ecosystem, learning resources |

### Animation & Quality Assurance
| Folder | What's Inside |
|--------|---------------|
| `22-animation-engines/` | GSAP advanced (ScrollTrigger, SplitText, MotionPath), Lenis v1.3.23, Framer Motion |
| `23-validation-notes/` | Corrected technical claims, repo status 2026, verified browser support tables |

### TypeScript & Frameworks (Advanced)
| Folder | What's Inside |
|--------|---------------|
| `24-typescript-threejs/` | Strict TypeScript + Three.js r184, typed loaders, custom material types, shader type safety |
| `25-nextjs-r3f/` | Next.js 15 + R3F v9, dynamic import, App Router SSR patterns, React 19 compatibility |
| `43-react-patterns/` | React 19 concurrent patterns, Suspense boundaries, useTransition for 3D, R3F v9 hooks |
| `44-astro-threejs/` | Astro 6 + Three.js, island architecture, ClientRouter, content collections for 3D assets |
| `45-r3f-performance/` | R3F-specific perf: useFrame optimization, instancing via drei, reconciler internals |

### Visual Effects & Post-processing
| Folder | What's Inside |
|--------|---------------|
| `26-postprocessing/` | SSAO, Bloom, DOF, LUT, EffectComposer chains, postprocessing v7 vs three stdlib |
| `29-shadows-advanced/` | PCF/PCSS/CSM, baked lightmaps, Three.js r168+ shadow deprecations |
| `30-instancing-gpu/` | InstancedMesh, BatchedMesh r155+, GPU picking, frustum culling, LOD |
| `42-advanced-materials/` | PBR deep-dive, custom ShaderMaterial, MeshPhysicalMaterial, transmission, anisotropy |
| `46-shader-tools/` | ShaderForge, glslify, lygia, vite-plugin-glsl, hot-reload GLSL dev workflow |

### XR, Audio & Media
| Folder | What's Inside |
|--------|---------------|
| `27-webxr/` | WebXR Device API, immersive-ar/vr sessions, Safari NO support confirmed, polyfills |
| `28-web-audio/` | Web Audio API spatial audio, AudioWorklet, Three.js PositionalAudio, iOS unlock |
| `34-video-streaming/` | Video textures, HLS.js, adaptive bitrate, VideoFrame API, WebCodecs |

### 2D & Typography
| Folder | What's Inside |
|--------|---------------|
| `31-svg-animation/` | GSAP MorphSVG (now free), SMIL (not deprecated), SVG filters, DrawSVG |
| `32-canvas2d-performance/` | OffscreenCanvas, Canvas 2D compositing, ImageBitmap, pixel manipulation |
| `33-typography-kinetic/` | Variable fonts, SplitText v3 (free), kinetic typography, scroll-driven text |

### Pipeline & Tooling
| Folder | What's Inside |
|--------|---------------|
| `35-blender-pipeline/` | glTF export settings, KTX2 baking, Blender 4.x scripts, normal map workflows |
| `36-scene-optimization/` | BVH raycasting, draw call batching, occlusion culling, scene graph flattening |
| `37-webgl-debugging/` | Spector.js, Chrome WebGL Inspector, gl.getError patterns, shader compile errors |
| `38-webgl-extensions/` | EXT_color_buffer_float, OES_texture_float, WEBGL_draw_buffers, extension feature detection |
| `39-asset-loading/` | GLTFLoader, KTX2Loader, DRACOLoader, progressive loading, asset caching strategies |
| `40-error-recovery/` | WebGL context loss/restore, graceful degradation, fallback pipelines |

### Camera & Interaction
| Folder | What's Inside |
|--------|---------------|
| `41-camera-systems/` | camera-controls v3.1.2, orbit controls, cinematic transitions, pointer events |

### Validation & Corrections
| Folder | What's Inside |
|--------|---------------|
| `47-validation-round2/` | Second-pass corrections: GSAP licensing, SMIL status, BVH claims, CSM repos |

## Quick Start

For a fast audit of your project, start with the checklists:
- `05-checklists/desktop-checklist.md`
- `05-checklists/mobile-checklist.md`

For new 3D projects, start with the build pipeline:
- `09-build-pipeline/vite-config-3d.md`
- `09-build-pipeline/texture-compression.md`

## Key Numbers (Validated)

| Number | Rule | Status |
|--------|------|--------|
| 100 | Max draw calls target for 60fps mobile | Verified |
| <=2.0x | Max DPR on mobile (community convention) | Community best practice |
| 2.0x | Max DPR on desktop | Community best practice |
| 4-8x | VRAM reduction from KTX2 (ETC1S/UASTC) | Verified |
| 70-98% | Draco geometry compression | Verified |
| 70-90% | gltfjsx scene size reduction | Verified |
| 256MB | iOS WebGL memory budget (safe working limit) | Device-dependent |

See `23-validation-notes/claim-corrections.md` and `47-validation-round2/` for full corrections log.

## Library Versions (2026-05-26)

| Library | Version | Status |
|---------|---------|--------|
| three.js | r184 (0.184.0) | Active |
| react-three-fiber | v9.6.1 | Active |
| drei | v10.7.7 | Active |
| lenis | v1.3.23 | Active |
| detect-gpu | v5.x | Active |
| camera-controls | v3.1.2 | Active |
| GSAP | 3.x (fully free) | Active — DrawSVG/MorphSVG/SplitText now free |
| Astro | 6.3.8 | Active — ViewTransitions -> ClientRouter in v5+ |

See `23-validation-notes/repo-status-2026.md` for full status including marginal/dead repos.

## Top Sites to Study

- bruno-simon.com — Awwwards SOTM (3D portfolio benchmark)
- lusion.co — SOTY 2024 (studio engine)
- igloo-inc.com — SOTY 2025
- 365.cartier.com — Enterprise 3D
- activetheory.net — Studio engine benchmark

Source repos: `21-open-source-repos/award-winning-portfolios.md`

## Key Corrections (from validation agents)

- GSAP plugins (DrawSVG, MorphSVG, SplitText) are now **100% free** — Club membership no longer required
- SMIL is **NOT deprecated** — Chrome reversed its 2015 deprecation intent; SMIL is fully supported
- PCFSoftShadowMap is **deprecated in r168** — silently redirects to PCFShadowMap; use PCFShadowMap directly
- Safari WebXR: **NO support** on macOS, iOS, or iPadOS (confirmed 2026)
- R3F v9 required for React 19 — v8 is incompatible
- Astro `<ViewTransitions />` renamed to `<ClientRouter />` in v5, removed in v6

See `47-validation-round2/` for full sourced corrections.
