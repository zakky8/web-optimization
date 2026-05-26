# Post-Processing on Mobile

## What to Kill Completely on Mobile

| Effect | Why |
|--------|-----|
| SSAO / SAO | Needs OES_texture_float_linear (absent many iOS), multiple full-screen passes |
| Depth of Field (Bokeh) | Multiple blur passes + depth reads = 2-5ms extra per frame |
| Full-res Bloom | 4+ Gaussian blur passes at native resolution |
| SSR (Screen-Space Reflections) | Ray-march passes, extremely fill-rate heavy |
| SMAA | Not designed for mobile, documented as unusable |
| FXAA (post-pass) | On mobile: use hardware antialias instead |

## What Can Stay (Reduced)

| Effect | Mobile Approach | Cost |
|--------|----------------|------|
| Bloom | Render at 1/4 resolution | Low at reduced res |
| FXAA | Use hardware antialias instead on mobile | Free (tile GPU) |
| Tone mapping | Single pass, keep it | Negligible |
| Vignette | Single pass shader | Negligible |
| Color grading (LUT) | Single 3D texture lookup | Negligible |
| Film grain / noise | Single pass | Negligible |

## Correct Pattern

```javascript
import { EffectComposer } from 'three/addons/postprocessing/EffectComposer.js'
import { RenderPass } from 'three/addons/postprocessing/RenderPass.js'
import { ShaderPass } from 'three/addons/postprocessing/ShaderPass.js'
import { FXAAShader } from 'three/addons/shaders/FXAAShader.js'
import { UnrealBloomPass } from 'three/addons/postprocessing/UnrealBloomPass.js'

const isMobile = /iPhone|iPad|iPod|Android/i.test(navigator.userAgent)

let renderer, composer

if (isMobile) {
  // Mobile: hardware antialias on renderer, no composer needed
  renderer = new THREE.WebGLRenderer({ antialias: true, ... })
  // Direct renderer.render() - no composer
  // Tone mapping on renderer:
  renderer.toneMapping = THREE.ACESFilmicToneMapping

} else {
  // Desktop: software FXAA via composer (more control)
  renderer = new THREE.WebGLRenderer({ antialias: false, ... })
  composer = new EffectComposer(renderer)
  composer.addPass(new RenderPass(scene, camera))

  // Bloom (desktop only)
  const bloom = new UnrealBloomPass(
    new THREE.Vector2(window.innerWidth, window.innerHeight),
    0.5,  // strength
    0.4,  // radius
    0.85  // threshold
  )
  composer.addPass(bloom)

  // SSAO (desktop only, tier 3 only)
  if (gpuTier >= 3) {
    composer.addPass(ssaoPass)
  }

  // FXAA always last
  const fxaa = new ShaderPass(FXAAShader)
  fxaa.uniforms['resolution'].value.set(1/window.innerWidth, 1/window.innerHeight)
  composer.addPass(fxaa)
}

// In render loop:
function render() {
  if (composer) {
    composer.render()
  } else {
    renderer.render(scene, camera)
  }
}
```

## Render Target Memory Cost on iOS

Each full-screen render target at 1080p = ~8MB.
Post-processing uses multiple render targets per effect.
With 5 passes (ping-pong rendering): 40MB just for post-processing buffers.
On iOS 256MB budget, this is 16% of total budget before any textures.

This is why disabling post-processing on mobile is a memory optimization, not just a performance one.

## Half-Resolution Bloom (Mobile-Friendly Version)

```javascript
// Render bloom at 1/4 resolution then upscale
const bloomComposer = new EffectComposer(renderer)
bloomComposer.renderToScreen = false
bloomComposer.addPass(new RenderPass(scene, camera))
bloomComposer.addPass(new UnrealBloomPass(
  new THREE.Vector2(window.innerWidth / 4, window.innerHeight / 4),
  1.5, 0.4, 0.85
))
// Composite bloom onto main render in final pass
```
