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
| `06-gpgpu-particles/` | GPUComputationRenderer, ping-pong buffers, compute shaders (262K+ particles) |
| `06-webgpu/` | WebGPURenderer setup, TSL node system, WGSL, render bundles, GPU-driven rendering |
| `07-shaders/` | GLSL noise (simplex/FBM/Worley), domain warping, lygia library, surface shaders, PBR injection |

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
| 100 | Max draw calls target for 60fps mobile | ✅ Verified |
| ≤2.0x | Max DPR on mobile (community convention) | Community best practice |
| 2.0x | Max DPR on desktop | Community best practice |
| 4-8× | VRAM reduction from KTX2 (ETC1S/UASTC) | ✅ Verified |
| 70-98% | Draco geometry compression | ✅ Verified |
| 70-90% | gltfjsx scene size reduction | ✅ Verified |
| 256MB | iOS WebGL memory budget (safe working limit) | ⚠️ Device-dependent |

See `23-validation-notes/claim-corrections.md` for full corrections log.

## Library Versions (2026-05-26)

| Library | Version | Status |
|---------|---------|--------|
| three.js | r184 (0.184.0) | ✅ Active |
| react-three-fiber | v9.6.1 | ✅ Active |
| drei | v10.7.7 | ✅ Active |
| lenis | v1.3.23 | ✅ Active |
| detect-gpu | v5.x | ✅ Active |

See `23-validation-notes/repo-status-2026.md` for full status including marginal/dead repos.

## Top Sites to Study

- bruno-simon.com — Awwwards SOTM (3D portfolio benchmark)
- lusion.co — SOTY 2024 (studio engine)
- igloo-inc.com — SOTY 2025
- 365.cartier.com — Enterprise 3D
- activetheory.net — Studio engine benchmark

Source repos: `21-open-source-repos/award-winning-portfolios.md`
