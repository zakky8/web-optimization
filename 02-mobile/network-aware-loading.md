# Network-Aware Asset Loading

## Network Information API (Chrome/Android Only)

Not supported on iOS Safari or Firefox. Always provide fallback.

```javascript
const conn = navigator.connection ||
             navigator.mozConnection ||
             navigator.webkitConnection

function getNetworkTier() {
  if (!conn) return 'high'           // no info = assume good (iOS users)
  if (conn.saveData) return 'low'    // user enabled Lite/Data Saver mode

  const map = {
    'slow-2g': 'low',
    '2g':      'low',
    '3g':      'medium',
    '4g':      'high'
  }
  return map[conn.effectiveType] ?? 'high'
}

// Listen for changes mid-session (WiFi -> cellular switch)
if (conn) {
  conn.addEventListener('change', () => {
    const newTier = getNetworkTier()
    if (newTier !== currentTier) {
      updateAssetQuality(newTier)
    }
  })
}
```

effectiveType is measured RTT/downlink by Chrome, NOT carrier-advertised speed.
saveData: true is a strong signal - user explicitly requested reduced data. Always honor it.

## Connection-Adaptive Model Loading

```javascript
async function loadModel() {
  const tier = getNetworkTier()

  const models = {
    high:   '/models/scene-hd.glb',    // 8MB, 4K KTX2 textures, full poly
    medium: '/models/scene-md.glb',    // 3MB, 2K KTX2 textures, LOD1
    low:    '/models/scene-ld.glb',    // 1MB, 1K KTX2 textures, LOD2
  }

  const textureRes = { low: 512, medium: 1024, high: 2048 }[tier]
  const enableEnvMap = tier === 'high'
  const shadowMapSize = { low: 0, medium: 512, high: 1024 }[tier]

  if (shadowMapSize === 0) {
    renderer.shadowMap.enabled = false
  } else {
    renderer.shadowMap.enabled = true
    dirLight.shadow.mapSize.setScalar(shadowMapSize)
  }

  return gltfLoader.loadAsync(models[tier])
}
```

## Progressive Loading Order

```
1. Inline CSS skeleton visible before any JS
2. Renderer initialization (canvas with bg color)
3. Scene structure (empty scene, lights, camera)
4. Low-res proxy geometry (boxes at correct positions)
5. Critical textures (512px versions of above-fold materials)
6. Primary GLB (Draco + KTX2 compressed)
7. High-res texture upgrades (after primary model visible)
8. Ambient effects (envmap, post-processing if device allows)
```

## Loading Progress UI

```javascript
const overlay = createLoadingScreen()

gltfLoader.load(
  modelUrl,
  (gltf) => {
    scene.add(gltf.scene)
    overlay.remove()
    startRenderLoop()
  },
  (progress) => {
    const pct = Math.round(progress.loaded / progress.total * 100)
    overlay.querySelector('.fill').style.width = pct + '%'
  },
  (error) => {
    console.error('Load failed:', error)
    overlay.querySelector('.text').textContent = 'Failed to load. Try refreshing.'
  }
)

function createLoadingScreen() {
  const div = document.createElement('div')
  div.innerHTML = `
    <div class="loading" style="position:fixed;inset:0;background:#000;z-index:9999;
         display:flex;align-items:center;justify-content:center;">
      <div style="width:200px">
        <div style="background:#333;height:2px;width:100%">
          <div class="fill" style="background:#fff;height:100%;width:0%;transition:width 0.1s">
          </div>
        </div>
        <p class="text" style="color:#fff;margin-top:8px;font-size:12px">Loading...</p>
      </div>
    </div>
  `
  document.body.appendChild(div)
  return div.firstElementChild
}
```

## prefers-reduced-data (Experimental)

```css
@media (prefers-reduced-data: reduce) {
  /* serve static image instead of 3D canvas */
  .webgl-canvas { display: none; }
  .static-fallback { display: block; }
}
```

Not widely supported yet. Use as progressive enhancement only.
