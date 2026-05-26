# Touch Events and Input Optimization

## Passive Listeners - Critical for Scroll Performance

Non-passive touch listeners block the browser's scroll compositor thread.
Browser must wait for preventDefault() before scrolling. Causes jank.

```javascript
// BAD - blocks scroll thread (Chrome shows warning in DevTools)
canvas.addEventListener('touchmove', handler)
canvas.addEventListener('touchstart', handler)

// GOOD - non-blocking (when NOT calling preventDefault)
canvas.addEventListener('touchmove', handler, { passive: true })
canvas.addEventListener('touchstart', handler, { passive: true })

// Required ONLY if you need to block scroll (e.g., orbit controls)
canvas.addEventListener('touchmove', handler, { passive: false })
```

Note: Three.js OrbitControls calls preventDefault() internally to block scroll during orbit.
Chrome will show the passive listener warning. Cannot suppress without patching OrbitControls.

## CSS touch-action - The Single Most Impactful Line

```css
canvas {
  touch-action: none;      /* eliminates 300ms tap delay + no scroll conflicts */
  user-select: none;
  -webkit-user-select: none;
  -webkit-tap-highlight-color: transparent;  /* removes tap flash on mobile */
}
```

touch-action: none tells the browser this element handles all its own touch input.
No scroll speculation, no delay, no conflicts with orbit controls.

## Pointer Events - Modern Unified API

Single handler for mouse + touch + stylus.

```javascript
// Pointer capture keeps events flowing even if pointer leaves canvas
canvas.addEventListener('pointerdown', (e) => {
  canvas.setPointerCapture(e.pointerId)
  startDrag(e.clientX, e.clientY)
})

canvas.addEventListener('pointermove', (e) => {
  if (e.buttons > 0) {
    updateDrag(e.clientX, e.clientY)
  }
})

canvas.addEventListener('pointerup', (e) => {
  canvas.releasePointerCapture(e.pointerId)
  endDrag()
})

// Multi-touch pinch zoom - track multiple pointers
const pointers = new Map()

canvas.addEventListener('pointerdown', (e) => {
  pointers.set(e.pointerId, { x: e.clientX, y: e.clientY })
})

canvas.addEventListener('pointermove', (e) => {
  if (!pointers.has(e.pointerId)) return
  pointers.set(e.pointerId, { x: e.clientX, y: e.clientY })

  if (pointers.size === 2) {
    const [p1, p2] = [...pointers.values()]
    const dist = Math.hypot(p2.x - p1.x, p2.y - p1.y)
    handlePinch(dist)
  }
})

canvas.addEventListener('pointerup', (e) => {
  pointers.delete(e.pointerId)
})
```

## Prevent iOS Bounce Scroll on Canvas

```css
/* Nuclear option - prevents all body scroll */
html, body {
  overflow: hidden;
  position: fixed;
  width: 100%;
  height: 100%;
}

canvas {
  touch-action: none;
  overscroll-behavior: none;
}
```

If page has other scrollable sections, use targeted JS:
```javascript
canvas.addEventListener('touchmove', (e) => {
  e.preventDefault()
}, { passive: false })  // must be non-passive to call preventDefault
```

## Gyroscope - iOS 13+ Requires Permission

```javascript
async function enableGyroscope() {
  // iOS 13+ requires explicit user permission
  // Must be called from a user gesture (tap/click) - NOT on page load
  if (typeof DeviceOrientationEvent?.requestPermission === 'function') {
    const permission = await DeviceOrientationEvent.requestPermission()
    if (permission !== 'granted') return
  }
  // Android auto-grants (no requestPermission method exists)

  window.addEventListener('deviceorientation', (e) => {
    // e.alpha = compass heading (0-360)
    // e.beta  = front-back tilt (-180 to 180)
    // e.gamma = left-right tilt (-90 to 90)

    const euler = new THREE.Euler(
      THREE.MathUtils.degToRad(e.beta),
      THREE.MathUtils.degToRad(e.alpha),
      THREE.MathUtils.degToRad(-e.gamma),
      'YXZ'
    )
    camera.quaternion.setFromEuler(euler)
  })
}

// Wire to user-gesture button only
document.getElementById('enable-gyro').addEventListener('click', enableGyroscope)
```
