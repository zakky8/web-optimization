# Canvas and Renderer Setup

## Correct Defaults

Every flag in WebGLRenderer has a memory/performance cost.

```javascript
// Three.js - production defaults
const renderer = new THREE.WebGLRenderer({
  canvas: document.getElementById('webgl'),
  powerPreference: 'high-performance',  // request discrete GPU on dual-GPU systems
  alpha: false,          // saves framebuffer memory (no transparency needed usually)
  antialias: false,      // handle via FXAA post-pass instead (more control)
  stencil: false,        // free GPU memory unless using stencil effects
  depth: true,           // keep unless you have no depth-tested geometry
})

renderer.setPixelRatio(Math.min(window.devicePixelRatio, 2))  // never above 2
renderer.setSize(window.innerWidth, window.innerHeight)
renderer.toneMapping = THREE.ACESFilmicToneMapping
renderer.toneMappingExposure = 1
renderer.outputColorSpace = THREE.SRGBColorSpace

// Disable matrix auto-update globally (Bruno Simon confirmed perf gain)
THREE.Object3D.DEFAULT_MATRIX_AUTO_UPDATE = false
// Then manually update only objects that move:
// mesh.updateMatrix() when position/rotation changes
```

## R3F Production Defaults

```jsx
<Canvas
  gl={{
    powerPreference: 'high-performance',
    alpha: false,
    antialias: false,
    stencil: false,
    depth: true,
  }}
  dpr={[1, 2]}            // min 1, max 2 - never render above 2x
  frameloop="demand"      // only render when invalidate() called (non-animated scenes)
  camera={{ fov: 75, near: 0.1, far: 100 }}
>
```

## DPR Management

```javascript
// Mobile: cap at 1.5 (not devicePixelRatio which can be 3)
// Desktop: cap at 2.0
const isMobile = /iPhone|iPad|iPod|Android/i.test(navigator.userAgent)
renderer.setPixelRatio(
  Math.min(window.devicePixelRatio, isMobile ? 1.5 : 2)
)

// Adaptive DPR (R3F) - dynamically reduces when FPS drops
const [dpr, setDpr] = useState(2)
<PerformanceMonitor onDecline={() => setDpr(d => Math.max(1, d - 0.2))}>
  <Canvas dpr={dpr}>
```

Why this matters: iPhone 14 has 3x DPR. Rendering at 3x = 9x the pixel count vs 1x.
Capping at 1.5 = 2.25x pixels. 4x less work than uncapped.

## Anti-Aliasing Strategy

```javascript
// When using post-processing composer:
renderer = new THREE.WebGLRenderer({ antialias: false })  // disable hardware MSAA
// Then add FXAA pass at end of composer chain

// FXAA (Shader pass) - 9 texture reads per pixel, negligible cost
import { ShaderPass } from 'three/addons/postprocessing/ShaderPass.js'
import { FXAAShader } from 'three/addons/shaders/FXAAShader.js'
const fxaaPass = new ShaderPass(FXAAShader)
fxaaPass.uniforms['resolution'].value.set(1/window.innerWidth, 1/window.innerHeight)
composer.addPass(fxaaPass)

// When NOT using post-processing (mobile):
renderer = new THREE.WebGLRenderer({ antialias: true })  // hardware MSAA - free on mobile tile GPUs
```

## Resize Handler

```javascript
window.addEventListener('resize', () => {
  const w = window.innerWidth
  const h = window.innerHeight
  camera.aspect = w / h
  camera.updateProjectionMatrix()
  renderer.setSize(w, h)
  renderer.setPixelRatio(Math.min(devicePixelRatio, 2))

  // Update FXAA resolution uniform if used
  if (fxaaPass) {
    fxaaPass.uniforms['resolution'].value.set(1/w, 1/h)
  }
}, { passive: true })
```

## Post-Processing Composer - When to Use

```javascript
import { EffectComposer } from 'three/addons/postprocessing/EffectComposer.js'
import { RenderPass } from 'three/addons/postprocessing/RenderPass.js'

const composer = new EffectComposer(renderer)
composer.addPass(new RenderPass(scene, camera))
// Add effect passes here
// Replace renderer.render(scene, camera) with composer.render()

// Order matters - most expensive passes last
// RenderPass -> SSAO -> Bloom -> FXAA (always last)
```
