# Learning Resources & Real-World Case Studies

## Primary Learning Paths

### Three.js Journey (Bruno Simon)
- URL: https://threejs-journey.com/
- Format: 93+ hours of video lessons
- Level: Beginner → Production-advanced
- Topics: Three.js, WebGL shaders, GLSL, React Three Fiber, Blender
- Status: Updated for r184, actively maintained
- Used by: Industry professionals worldwide
- Note: Paid course — de facto industry standard for Three.js

### WebGPU Fundamentals
- URL: https://webgpufundamentals.org/
- Format: Written lessons with live examples
- By: Gregg Tavares (Google Chrome team)
- Status: Active 2026
- Topics: WebGPU API, WGSL, compute shaders, rendering

### Codrops (Frontend Pearls)
- URL: https://tympanus.net/codrops/
- Format: Tutorials + demo code
- Quality: High — used as reference by industry
- Notable tags: WebGL, Three.js, GSAP, canvas
- Always check publish date — older articles may reference deprecated APIs

## Case Studies

### Google Chrome Developers
- URL: https://developer.chrome.com/case-studies/
- Covers: WebGL, WebGPU, WebXR, performance
- Tier 1 source — official engineering team authorship

### WebGL Insights (Book)
- URL: https://www.webglinsights.com/
- Format: Collected essays from WebGL implementors
- Note: 2015 — conceptually still valid, but verify APIs

## GitHub Stars as Quality Signal

Track these repos for production patterns:

| Repo | Stars | Why Follow |
|------|-------|------------|
| mrdoob/three.js | 103k+ | Source of truth |
| pmndrs/react-three-fiber | 27k+ | R3F architecture |
| pmndrs/drei | 8k+ | Helper patterns |
| greggman/twgl.js | 3k+ | Minimal WebGL |
| regl-project/regl | 5k+ | Functional WebGL |
| oframe/ogl | 4k+ | Minimal alternative to Three.js |

Stars verified approximately — check GitHub for current counts.

## Awwwards Technical Teardowns

Sites awarded SOTD with WebGL:
1. https://www.awwwards.com/sites/with-webgl — filter for WebGL
2. View source + DevTools Network tab to identify stack
3. Find GitHub repo (many are open source)

Key signals in DevTools:
- `three.module.js` → Three.js
- `@react-three` → R3F
- `gsap.min.js` → GSAP
- Custom `.vert`/`.frag` downloads → Custom shaders

## Open Source Inspiration

```
luruke/awesome-casestudy    DEAD — last commit Sep 2022, do not reference
nicoptere/FBO               DEAD — last commit May 2021, use GPUComputationRenderer
nicoptere/physarum          Active — excellent GPGPU reference
brunosimon/folio-2019       Active — SOTD winner, production WebGL
brunosimon/folio-2025       Active — modern WebGPU/TSL
HamishMW/portfolio          Active — Three.js + React integration
```

Source: GitHub — verified 2026-05-26

## Sources
- Three.js Journey: https://threejs-journey.com/
- Codrops: https://tympanus.net/codrops/
- WebGPU Fundamentals: https://webgpufundamentals.org/
- Awwwards WebGL: https://www.awwwards.com/awwwards/collections/webgl/
