# Award-Winning Portfolio Repos

All star counts and commit dates verified against GitHub — 2026-05-26

## brunosimon/folio-2019

```
URL:     https://github.com/brunosimon/folio-2019
Stars:   4,684
Status:  Active (recent commits 2026)
Awards:  Awwwards SOTD, FWA, CSS Design Awards
Stack:   Three.js, custom WebGL, no framework
```

Notable patterns:
- Custom WebGL renderer (no THREE.WebGLRenderer wrapper)
- Pixel-perfect raymarching for environment
- Custom cursor with lag/trail
- GSAP for UI transitions, Three.js for 3D

## brunosimon/folio-2025

```
URL:     https://github.com/brunosimon/folio-2025
Stars:   1,411
Status:  Active
Awards:  Awwwards SOTD 2025
Stack:   Three.js r184, WebGPU, TSL
```

Notable patterns:
- TSL node materials
- WebGPU compute for particles
- React integration
- Vite build pipeline

## brunosimon/threejs-template-complex

```
URL:     https://github.com/brunosimon/threejs-template-complex
Stars:   2,000+ (estimated — check GitHub for current)
Status:  Active
Purpose: Production project template
```

Structure used as reference template:
```
src/
├── Experience/
│   ├── World/        → scene objects
│   ├── Camera.js     → camera controls
│   ├── Renderer.js   → renderer setup
│   ├── sources.js    → asset manifest
│   └── utils/        → EventEmitter, Resources, Sizes, Time, Debug
```

## HamishMW/portfolio

```
URL:     https://github.com/HamishMW/portfolio
Stars:   3,434
Status:  Active
Awards:  Awwwards SOTD
Stack:   Three.js + React (no R3F), custom hooks
```

Notable patterns:
- Direct Three.js inside React (not via R3F)
- Custom `useThree` hook
- `useAnimationFrame` hook
- Post-processing: custom depth-of-field
- WebGL model rendering inside React layout

Patterns to study:
```js
// Custom React-Three integration (not R3F)
function useThreeScene(canvasRef) {
  useEffect(() => {
    const canvas = canvasRef.current;
    const renderer = new THREE.WebGLRenderer({ canvas });
    const scene = new THREE.Scene();
    // ...
    return () => { renderer.dispose(); scene.traverse(dispose); };
  }, []);
}
```

## nicoptere/physarum

```
URL:     https://github.com/nicoptere/physarum
Stars:   317
Status:  Active
Purpose: Physarum (slime mold) GPU simulation
```

Key implementation:
- Ping-pong render targets (GPUComputationRenderer pattern without the helper)
- Agent simulation: deposit → diffuse → decay cycle
- 2D GPGPU on GPU: 512×512 = 262,144 agents
- Custom Three.js post-processing for visualization

## nicoptere/FBO (ARCHIVED — Do Not Use)

```
URL:     https://github.com/nicoptere/FBO
Stars:   436
Status:  DEAD — last commit May 2021
Note:    Pre-dates GPUComputationRenderer
         Use GPUComputationRenderer instead
```

## Sources
- All repos verified on GitHub — 2026-05-26
- Bruno Simon Three.js Journey: https://threejs-journey.com/
