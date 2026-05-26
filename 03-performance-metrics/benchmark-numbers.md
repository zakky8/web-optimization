# Benchmark Numbers

## Numbers to Memorize

| Number | What It Is |
|--------|-----------|
| 100 | Draw calls target for 60fps |
| 1,000 | Absolute draw call ceiling (any hardware) |
| 1.5x | Max DPR on mobile |
| 2.0x | Max DPR on desktop |
| 64MB+ | VRAM cost of one 4K PNG/JPEG |
| 6-8MB | VRAM cost of same texture in KTX2/ETC1S |
| 50-70% | DRACO geometry compression rate |
| 80% | Scene weight reduction from GPU instancing (Chipsa confirmed) |
| 3.5s | Shader compile stall without compileAsync() |
| 3KB | Lenis library footprint |
| 2.8MB | Bruno Simon's entire 2019 portfolio |
| 220KB | Lusion's 66-frame cloth simulation (gzipped) |
| 256MB | iOS WebGL canvas hard limit |
| 8x | ASTC memory reduction vs uncompressed RGBA |

## Optimization Case Study Results

| Optimization | Before | After | Source |
|---|---|---|---|
| Instancing + atlasing | 2,200 draw calls, 60fps | 17 draw calls, 1,500fps | Dhia Shakiry |
| GPU instancing | baseline | 80% scene weight reduction | Chipsa RSI case study |
| Basis texture compression | 1.5GB VRAM | 700MB VRAM (-53%) | Dhia case study |
| KTX2 adoption | standard PNG | Faster GPU upload, less VRAM | Cartier 365 case study |
| DRACO compression | baseline geometry | 50-70% smaller file | Bruno Simon |
| gltfjsx full pipeline | source assets | 90% asset size reduction | Codrops 2025 |
| Lusion cloth sim | large keyframe data | 220KB gzip, 4,096 verts desktop | Lusion case study |
| OffscreenCanvas | Lighthouse 95 | Lighthouse 100 | Evil Martians |
| Draco + LOD on mobile | 20-30 avg FPS | 45-55 avg FPS | Krapton blog 2026 |
| + OffscreenCanvas | 45-55 FPS | Consistent 60 FPS | Krapton blog 2026 |
| matcap vs real lights | multiple draw calls | zero GPU lighting cost | Bruno Simon |

## Frame Rate Targets

| Platform | Target | Acceptable Minimum |
|----------|--------|--------------------|
| Desktop | 60fps | 60fps |
| Desktop high-refresh | 120fps | 60fps |
| Mobile | 60fps | 30fps |
| Mobile thermal throttled | 30fps | 20fps |

R3F PerformanceMonitor default bounds: 30-500fps

## Asset Size Reference Points

| Asset | Size | Notes |
|-------|------|-------|
| Bruno Simon '19 full site | 2.8MB | Images dominate |
| Codrops demo scene | 2.1MB | 27 meshes, 184 textures, 49 shaders |
| Lusion desktop model | 983KB | 4,096 vertices |
| Lusion mobile model | 246KB | 1,024 vertices |
| Lusion animation data | 220KB gzip | 66 frames, 11 keyframes stored |
| Lenis library | 3KB | Complete smooth scroll |
| detect-gpu library | ~15KB | GPU tier detection |

## Shader Compilation Times (Without compileAsync)

| Scene Complexity | First-Frame Stall |
|-----------------|------------------|
| Simple (5-10 materials) | 200-500ms |
| Medium (20-40 materials) | 1-2 seconds |
| Complex (50+ materials, custom shaders) | 3-5 seconds |

With renderer.compileAsync(): 0ms stall (compiled behind loading screen)
