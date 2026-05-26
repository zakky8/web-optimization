# Thermal Throttling

## The Problem

Mobile GPUs throttle clock speeds when the SoC gets hot.
A scene at 60fps on cold boot can drop to 20fps after 30 seconds of sustained load.
There is NO web API to detect device temperature.

## Indirect Detection via Frame Time Monitoring

```javascript
const frameTimes = []
let lastFrame = performance.now()
let thermalMode = false

function renderLoop(timestamp) {
  const delta = timestamp - lastFrame
  lastFrame = timestamp

  frameTimes.push(delta)
  if (frameTimes.length > 60) frameTimes.shift()

  if (frameTimes.length === 60) {
    const avgFPS = 1000 / (frameTimes.reduce((a,b) => a+b) / frameTimes.length)

    // Was running well, now collapsed = thermal throttle
    if (avgFPS < 25 && !thermalMode) {
      thermalMode = true
      enterLowQualityMode()
    }
  }

  renderer.render(scene, camera)
  requestAnimationFrame(renderLoop)
}

function enterLowQualityMode() {
  // Priority order - kill most expensive first
  composer.passes.forEach(p => p.enabled = false)  // kill all post-processing
  renderer.shadowMap.enabled = false
  renderer.setPixelRatio(1)  // drop to 1x
  particles.count = Math.floor(particles.count / 2)
}
```

## Proactive Fix: Cap Mobile at 30fps

Prevents the GPU from ever hitting full throttle in the first place.

```javascript
const isMobile = /iPhone|iPad|iPod|Android/i.test(navigator.userAgent)
const TARGET_MS = isMobile ? 33.33 : 16.67  // 30fps vs 60fps

let lastRender = 0

function loop(time) {
  requestAnimationFrame(loop)
  if (time - lastRender < TARGET_MS) return  // skip frame
  lastRender = time
  renderer.render(scene, camera)
}
requestAnimationFrame(loop)
```

## R3F Performance.regress() During Interaction

Tell R3F to drop quality temporarily during camera movement:

```javascript
import { useThree } from '@react-three/fiber'

function CameraController() {
  const { performance } = useThree()

  useEffect(() => {
    const controls = new OrbitControls(camera, gl.domElement)
    controls.addEventListener('change', () => {
      performance.regress()  // drops dpr temporarily while moving
    })
  }, [])
}
```

## Thermal Throttling Mitigation Checklist

- Cap DPR at 1.5 on mobile (never use raw devicePixelRatio which is 3x on iPhone)
- Target 30fps on mobile, 60fps on desktop
- Kill ALL post-processing on mobile (biggest thermal source)
- Reduce shadow map: 512 on mobile vs 2048 on desktop
- Monitor rolling 60-frame FPS average
- When FPS < 25 for 60 frames: drop to low quality mode
- Stop render loop when tab is hidden (Page Visibility API)
