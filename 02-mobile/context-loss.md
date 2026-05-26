# WebGL Context Loss - iOS and Chrome

## Why Context Loss Happens

Three documented causes:
1. iOS 17 bug (WebKit #261331): backgrounding Safari killed WebGL context
   - Fixed in iOS 17.1.1
2. Memory pressure: approaching 256MB iOS canvas limit
3. Video element in texImage2D: passing paused video caused loss on affected iOS

## Three.js Does NOT Auto-Recover

Three.js calls event.preventDefault() on webglcontextlost (tells browser it will restore)
but does NOT rebuild GPU resources on webglcontextrestored. You must handle this.

```javascript
const canvas = renderer.domElement

canvas.addEventListener('webglcontextlost', (event) => {
  event.preventDefault()               // REQUIRED: tell browser we will restore
  cancelAnimationFrame(animationId)    // stop render loop
  console.warn('WebGL context lost')
}, false)

canvas.addEventListener('webglcontextrestored', () => {
  console.log('WebGL context restored, reinitializing...')
  reinitRenderer()
}, false)

function reinitRenderer() {
  renderer.dispose()

  renderer = new THREE.WebGLRenderer({
    canvas: canvas,
    antialias: false,
    powerPreference: 'high-performance',
  })
  renderer.setPixelRatio(Math.min(devicePixelRatio, 1.5))
  renderer.setSize(window.innerWidth, window.innerHeight)

  // Re-flag ALL textures for re-upload to GPU
  // Textures are lost with context and must be re-uploaded
  scene.traverse((obj) => {
    if (obj.isMesh) {
      if (obj.material.map) obj.material.map.needsUpdate = true
      if (obj.material.normalMap) obj.material.normalMap.needsUpdate = true
      if (obj.material.roughnessMap) obj.material.roughnessMap.needsUpdate = true
    }
  })

  animationId = requestAnimationFrame(renderLoop)
}
```

## Chrome-Specific: Context May Not Recover

Chrome tabs that experience context loss often cannot create a new context in the same session.
The only reliable fix is a page reload.

```javascript
canvas.addEventListener('webglcontextlost', (event) => {
  event.preventDefault()

  const isChrome = /Chrome/.test(navigator.userAgent) && !/Safari/.test(navigator.userAgent)
  if (isChrome) {
    // Show reload prompt instead of trying to reinit
    showReloadPrompt()
  }
}, false)

function showReloadPrompt() {
  const div = document.createElement('div')
  div.innerHTML = `
    <div style="position:fixed;inset:0;display:flex;align-items:center;
                justify-content:center;background:rgba(0,0,0,0.8);z-index:9999;color:white">
      <div>
        <p>Scene connection lost</p>
        <button onclick="location.reload()">Reload</button>
      </div>
    </div>
  `
  document.body.appendChild(div)
}
```

## Prevention: Stop Rendering When Hidden

Prevents context loss from memory pressure and iOS backgrounding bug.

```javascript
let animId = null

function startRender() {
  if (!animId) animId = requestAnimationFrame(renderLoop)
}

function stopRender() {
  cancelAnimationFrame(animId)
  animId = null
}

document.addEventListener('visibilitychange', () => {
  if (document.hidden) {
    stopRender()
  } else {
    // Check if context was lost while hidden
    if (renderer.getContext().isContextLost()) {
      reinitRenderer()
    } else {
      startRender()
    }
  }
})
```
