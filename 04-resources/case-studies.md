# Primary Case Studies

All Tier 1 sources (written by the developers who built the sites).
Read these first - they contain more detail than any tutorial.

## Must-Read (Start Here)

### Bruno Simon Portfolio Case Study (2019)
- URL: https://medium.com/@bruno_simon/bruno-simon-portfolio-case-study-960402cc259b
- Covers: DRACO, matcap lighting, Cannon.js physics, mobile optimizations (DPR clamping,
  disabling matrix auto-update, removing blur effects), VAO optimization
- Why read: foundational techniques, clearly explained

### 60 to 1500 FPS - Optimising a WebGL Visualisation (Dhia Shakiry)
- URL: https://medium.com/@dhiashakiry/60-to-1500-fps-optimising-a-webgl-visualisation-d79705b33af4
- Covers: exact progression of instancing -> Basis compression -> texture atlasing
- Why read: most quantitatively documented optimization study in the industry

### Faster WebGL with OffscreenCanvas and Web Workers (Evil Martians)
- URL: https://evilmartians.com/chronicles/faster-webgl-three-js-3d-graphics-with-offscreencanvas-and-web-workers
- Covers: OffscreenCanvas pattern, bundle splitting, DOM API limitations, Lighthouse 95 -> 100
- Why read: definitive guide to moving Three.js off main thread

## Studio Case Studies

### Lusion (SOTY 2024) - Awwwards
- URL: https://www.awwwards.com/case-study-for-lusion-by-lusion-winner-of-site-of-the-month-may.html
- Covers: Houdini cloth sim to ArrayBuffer, 11-keyframe interpolation, 16-bit integer encoding,
  mobile vertex count reduction, 220KB gzip animation data

### Igloo Inc (SOTY 2025) - Awwwards
- URL: https://www.awwwards.com/igloo-inc-case-study.html
- Covers: UI-in-WebGL, custom geometry exporters, VDB compression,
  background shader compilation, tiered texture loading

### Bruno Simon '25 Portfolio - Awwwards
- URL: https://www.awwwards.com/brunos-portfolio-case-study.html
- Covers: WebGPU + Three.js + TSL, grass instancing (78,400 blades), ETC1S/UASTC,
  frustum culling, mobile DoF/shadow disabling

### Active Theory: Story of Technology
- URL: https://medium.com/active-theory/the-story-of-technology-built-at-active-theory-5d17ae0e3fb4
- Covers: Hydra engine, WebWorker parallelization, WebGL2/1 fallback,
  designer-code workflow via GUI, Aura cross-platform engine

### Chipsa WebGL Evolution
- URL: https://medium.com/@Chipsa/the-evolution-of-webgl-at-chipsa-c68dca54d538
- Covers: 6-year studio evolution, GPU instancing (80% scene weight reduction),
  KTX2 adoption, WebGPU compute for particles, gyroscope mobile interaction

### Cartier 365 Behind the Scenes
- URL: https://www.awwwards.com/behind-the-scenes-designing-and-building-365-a-year-of-cartier.html
- Covers: React + R3F + 14islands scroll rig, KTX2 conversion results

## Technical Deep Dives

### Building Efficient Three.js Scenes (Codrops, Feb 2025)
- URL: https://tympanus.net/codrops/2025/02/11/building-efficient-three-js-scenes-optimize-performance-while-maintaining-quality/
- Covers: DPR management, PerformanceMonitor, InstancedMesh, KTX2, gltfjsx pipeline
- Benchmark: 27 meshes, 184 textures, 40K triangles, 2.1MB, stable FPS

### Stas Bondar '25 Code Techniques (Codrops, March 2025)
- URL: https://tympanus.net/codrops/2025/03/25/stas-bondar-25-the-code-techniques-behind-a-next-level-portfolio/
- Covers: gsap.quickTo(), Bayer dithering shader, Matter.js physics, Flip transitions

### The Monolith: 13-Scene WebGL Deferred Rendering (Codrops, Nov 2025)
- URL: https://tympanus.net/codrops/2025/11/29/building-the-monolith-composable-rendering-systems-for-a-13-scene-webgl-epic/
- Covers: G-Buffer deferred rendering, octahedron normal encoding, composable shader modules

### Web Texture Formats for WebGL/WebGPU (Don McCurdy)
- URL: https://www.donmccurdy.com/2024/02/11/web-texture-formats/
- Covers: Definitive guide to ETC1S vs UASTC vs PNG/JPEG for WebGL
- Author: Three.js core contributor

### Figma: Rendering Powered by WebGPU
- URL: https://www.figma.com/blog/figma-rendering-powered-by-webgpu/
- Covers: WebGL -> WebGPU migration, GLSL -> WGSL, explicit draw-call architecture

## Curated Index

### luruke/awesome-casestudy (GitHub)
- URL: https://github.com/luruke/awesome-casestudy
- The most comprehensive index of WebGL/creative development case studies
- Updated by the community, filterable by technique
