# Top Benchmark Sites (2025-2026)

Study these - they are the industry ceiling for 3D and animation on the web.

| Site | Award | Why It Matters |
|------|-------|----------------|
| bruno-simon.com | Awwwards SOTM Jan 2026 | Full interactive 3D driving game in browser. Ultimate benchmark. |
| lusion.co | SOTY 2024, Dev Award | Real-time cloth sim, Houdini FX pipeline. |
| igloo-inc.com | SOTY 2025 | Procedural ice 3D world in Svelte. Best shader precompilation example. |
| stabondar.com | GSAP Site of the Month | Physics + Bayer dithering shaders + multi-column WebGL. |
| 365.cartier.com | Awwwards multi-award 2024-25 | React + R3F + KTX2 - best enterprise 3D example. |
| activetheory.net | Ongoing industry benchmark | Custom Hydra engine, full WebWorker parallelization. |
| chipsa.design | Awwwards/Codrops 2025 | GPU instancing + WebGPU compute + KTX2. 80% scene weight reduction. |
| landonauta.com | SOTD + SOTM 2025 | F1 site - 3D helmet rotations + Rive on Webflow. |
| federicopian.com | SOTD 2024 | Nuxt + TresJs + GSAP Flip. |

## What They All Have in Common
- KTX2 compressed textures (never raw PNG/JPEG in GPU)
- DRACO geometry compression
- Device-tiered quality (not one fixed quality level for all devices)
- Baked lighting (matcap / pre-rendered maps, not real-time lights)
- DPR capped at 2.0 on desktop / 1.5 on mobile
- GSAP + Lenis for scroll
- Sub-100 draw calls per frame target
