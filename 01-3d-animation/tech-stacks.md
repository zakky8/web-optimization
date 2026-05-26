# Tech Stacks Used by Top Studios

## Standard 2025-2026 Stack

| Layer | Choices |
|-------|---------|
| Renderer | Three.js (most common), R3F (React), TresJs (Vue), WebGPU (emerging) |
| Framework | Vanilla JS, Astro, Next.js, Nuxt, Svelte |
| Animation | GSAP (ScrollTrigger, SplitText, Flip, quickTo) |
| Smooth Scroll | Lenis (3KB) - replaced Locomotive Scroll |
| 3D Authoring | Blender + Houdini -> GLTF export |
| Physics | Cannon.js / Matter.js / Rapier (WASM, fastest) |
| GPU Detection | @pmndrs/detect-gpu |
| Page Transitions | Barba.js |

## Per-Studio Breakdown

| Studio | Renderer | Framework | Animation | Special |
|--------|----------|-----------|-----------|---------|
| Bruno Simon '25 | Three.js + WebGPU | Vanilla JS | - | TSL shaders, ETC1S/UASTC textures |
| Igloo Inc | Three.js + BVH | Svelte | GSAP | Houdini, custom geometry exporters |
| Lusion v3 | Three.js | Angular | TweenLite | Houdini cloth sim, ArrayBuffer vertex anim |
| Stas Bondar '25 | Three.js | Astro | GSAP full suite | Matter.js physics, GLSL Bayer dithering |
| Active Theory | Hydra (custom) | Custom | Custom | WebWorkers, WebGL2/WebGPU, Aura engine |
| Cartier 365 | R3F | Next.js | Framer Motion | 14islands r3f-scroll-rig |
| Chipsa | Three.js | - | - | KTX2, WebGPU compute particles |

## TSL (Three.js Shading Language)
Bruno Simon '25 uses TSL - shaders written in JavaScript that compile to both
GLSL (WebGL) and WGSL (WebGPU). Single codebase, dual renderer support.
Available: Three.js r155+
