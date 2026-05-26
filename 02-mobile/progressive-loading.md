# Progressive Loading for Mobile

## Load Order Strategy

Load in this exact order for best perceived performance:

```
1. HTML skeleton (inline CSS, visible immediately)
2. Critical JS (renderer init, empty scene)
3. Draco + KTX2 decoders (start WASM download early)
4. Low-res proxy geometry (boxes at correct world positions)
5. Critical above-fold textures (512px versions)
6. Primary GLB model (Draco compressed geometry + KTX2 textures)
7. High-res texture upgrades (1K/2K versions swap in)
8. Environmental effects (envmap, post-processing if device supports)
```

## Draco + KTX2 Full Setup

```javascript
import { GLTFLoader } from 'three/addons/loaders/GLTFLoader.js'
import { DRACOLoader } from 'three/addons/loaders/DRACOLoader.js'
import { KTX2Loader } from 'three/addons/loaders/KTX2Loader.js'

// Start decoder downloads immediately (before model load is called)
const dracoLoader = new DRACOLoader()
dracoLoader.setDecoderPath('/libs/draco/')
dracoLoader.preload()  // begins WASM download now

const ktx2Loader = new KTX2Loader()
  .setTranscoderPath('/libs/basis/')
  .detectSupport(renderer)

const gltfLoader = new GLTFLoader()
gltfLoader.setDRACOLoader(dracoLoader)
gltfLoader.setKTX2Loader(ktx2Loader)
```

## Texture LOD Upgrade Pattern

Show low-res textures immediately, upgrade when idle:

```javascript
async function loadWithTextureUpgrade(modelUrl) {
  // Load low-res version first
  const lowRes = await gltfLoader.loadAsync(modelUrl.replace('.glb', '-low.glb'))
  scene.add(lowRes.scene)

  // Immediately visible - start rendering

  // Upgrade textures when browser is idle
  requestIdleCallback(async () => {
    const highRes = await gltfLoader.loadAsync(modelUrl)
    // Swap textures on existing objects
    lowRes.scene.traverse((obj) => {
      if (!obj.isMesh) return
      const newObj = highRes.scene.getObjectByName(obj.name)
      if (newObj?.material) {
        obj.material = newObj.material
      }
    })
  })
}
```

## Shader Compile Behind Loading Screen

Always compile shaders before revealing scene (prevents 1-4s first-frame freeze):

```javascript
async function loadAndReveal() {
  const overlay = showLoadingOverlay()

  // Load model
  const gltf = await gltfLoader.loadAsync('/scene.glb')
  scene.add(gltf.scene)

  // Update loading progress (75%)
  overlay.setProgress(75)

  // Compile all shaders - prevents first-frame stutter
  await renderer.compileAsync(scene, camera)

  // Progress to 100%
  overlay.setProgress(100)

  // Small delay for smooth transition
  await new Promise(resolve => setTimeout(resolve, 200))

  // Fade out overlay, start render loop
  overlay.fadeOut()
  startRenderLoop()
}
```

## Connection-Aware Model Selection

```javascript
async function selectAndLoad() {
  const conn = navigator.connection
  const savingData = conn?.saveData
  const slowConnection = ['slow-2g', '2g', '3g'].includes(conn?.effectiveType)

  const quality = (savingData || slowConnection) ? 'low' : 'high'

  const assets = {
    high: { model: '/scene-hd.glb', envMap: '/hdri-2k.hdr' },
    low:  { model: '/scene-ld.glb', envMap: null },
  }[quality]

  if (assets.envMap) {
    const envMap = await rgbeLoader.loadAsync(assets.envMap)
    scene.environment = envMap
  }

  return gltfLoader.loadAsync(assets.model)
}
```

## Asset Pipeline Commands

```bash
# Full optimization pipeline for any GLTF model:
npx gltfjsx model.gltf -S -T -t
# -S = mesh simplification
# -T = web-friendly transforms (draco, ktx2 compatible)
# -t = apply Draco compression + KTX2 texture conversion

# Result: typically 80-95% smaller than source
# Bruno Simon '19: 2.8MB total site
# Codrops demo (27 meshes, 184 textures): 2.1MB total
```
