# Main Thread & Web Workers

## The Problem

Three.js on the main thread blocks the browser's UI thread.
Every heavy frame = frozen scroll, dropped inputs, failing Core Web Vitals (INP).

Result without optimization: Lighthouse INP fails, site feels laggy on interaction.
Result with OffscreenCanvas: Lighthouse 95 -> 100 (Evil Martians confirmed).

## OffscreenCanvas - Move Three.js Off Main Thread

```javascript
// main.js - runs on main thread
const canvas = document.getElementById('webgl')
const offscreen = canvas.transferControlToOffscreen()
const worker = new Worker('./webgl-worker.js', { type: 'module' })

worker.postMessage(
  { type: 'init', canvas: offscreen, width: window.innerWidth, height: window.innerHeight },
  [offscreen]  // transfer ownership
)

// Forward resize events to worker
window.addEventListener('resize', () => {
  worker.postMessage({
    type: 'resize',
    width: window.innerWidth,
    height: window.innerHeight
  })
})

// Forward mouse/touch to worker
window.addEventListener('mousemove', (e) => {
  worker.postMessage({ type: 'mousemove', x: e.clientX, y: e.clientY })
})
```

```javascript
// webgl-worker.js - runs on separate thread, full Three.js here
import * as THREE from 'three'

let renderer, scene, camera

self.onmessage = ({ data }) => {
  switch (data.type) {
    case 'init':
      renderer = new THREE.WebGLRenderer({ canvas: data.canvas })
      renderer.setSize(data.width, data.height)
      scene = new THREE.Scene()
      camera = new THREE.PerspectiveCamera(75, data.width / data.height, 0.1, 100)
      animate()
      break
    case 'resize':
      renderer.setSize(data.width, data.height)
      camera.aspect = data.width / data.height
      camera.updateProjectionMatrix()
      break
    case 'mousemove':
      // update scene based on mouse
      break
  }
}

function animate() {
  requestAnimationFrame(animate)
  renderer.render(scene, camera)
}

// IMPORTANT: No DOM APIs in worker
// Use ImageBitmapLoader instead of ImageLoader
// Use ImageBitmapLoader instead of TextureLoader
```

CAVEAT: No DOM APIs in workers.
Replace: TextureLoader -> ImageBitmapLoader
Replace: ImageLoader -> ImageBitmapLoader

## Web Workers for Parallel Tasks (Active Theory Pattern)

Active Theory's Hydra engine parallelizes all heavy non-render work:

```javascript
// geometry-worker.js
self.onmessage = ({ data: { vertices, count } }) => {
  // Heavy computation off main thread
  const positions = new Float32Array(count * 3)
  for (let i = 0; i < count; i++) {
    // compute particle positions, physics, etc.
    positions[i * 3] = Math.random() * 100
    positions[i * 3 + 1] = Math.random() * 100
    positions[i * 3 + 2] = Math.random() * 100
  }
  self.postMessage({ positions }, [positions.buffer]) // transfer, don't copy
}

// main.js
const geoWorker = new Worker('./geometry-worker.js')
geoWorker.postMessage({ count: 100000 })
geoWorker.onmessage = ({ data: { positions } }) => {
  geometry.setAttribute('position', new THREE.BufferAttribute(positions, 3))
}
```

Tasks to move off main thread (Active Theory confirmed):
- Loading and parsing geometry
- Computing particle system positions
- Physics collision detection
- Texture atlas generation
- Path finding / AI
- Large data processing

## React 18 startTransition for R3F

Wrap expensive scene changes to keep frame rate stable:

```javascript
import { startTransition } from 'react'

function onNavigate(newSection) {
  startTransition(() => {
    setActiveSection(newSection)  // triggers heavy scene rebuild
    // React yields to browser between chunks, preventing frame drops
  })
}
```

## Bundle Splitting Strategy

```javascript
// vite.config.js
export default {
  build: {
    rollupOptions: {
      output: {
        manualChunks: {
          'three': ['three'],
          'drei': ['@react-three/drei'],
          'postprocessing': ['postprocessing', '@react-three/postprocessing'],
          'webgl-worker': ['./src/webgl-worker.js']
        }
      }
    }
  }
}
// Result: three.js only loads when WebGL section is visible
// Use dynamic import() with IntersectionObserver
```
