# Battery-Aware Rendering

## Page Visibility API - Universal, Stop When Hidden

Works on all browsers including iOS Safari 7+.
Stops render loop when tab is not visible. Saves 75% CPU when hidden.

```javascript
let animId = null

function startRendering() {
  if (!animId) animId = requestAnimationFrame(renderLoop)
}

function stopRendering() {
  if (animId) {
    cancelAnimationFrame(animId)
    animId = null
  }
}

document.addEventListener('visibilitychange', () => {
  document.hidden ? stopRendering() : startRendering()
})

// Also stop when window loses focus (not just tab switch)
window.addEventListener('blur', stopRendering)
window.addEventListener('focus', startRendering)
```

## requestIdleCallback - iOS Has No Support

```javascript
// iOS Safari does NOT support requestIdleCallback
// Always polyfill:
window.requestIdleCallback = window.requestIdleCallback || function(cb, opts) {
  const start = Date.now()
  return setTimeout(() => {
    cb({
      didTimeout: false,
      timeRemaining: () => Math.max(0, 50 - (Date.now() - start))
    })
  }, opts?.timeout || 1)
}

// Use for non-visual work only (never render from here)
requestIdleCallback(() => {
  preloadNextSection()      // pre-warm textures
  computePathfinding()      // heavy math
  cleanupDisposedObjects()  // garbage collection
})
```

## Battery Status API - Do Not Rely On

Deprecated from W3C standards (fingerprinting vector).
iOS Safari: never supported.
Android Chrome: HTTPS only, may be removed.

Use as optional signal only, never as primary quality control:
```javascript
if ('getBattery' in navigator) {
  navigator.getBattery().then(battery => {
    if (!battery.charging && battery.level < 0.2) {
      // Under 20% battery and not charging
      setQualityLevel('low')
    }
    battery.addEventListener('levelchange', () => {
      if (!battery.charging && battery.level < 0.2) {
        setQualityLevel('low')
      }
    })
  })
}
```

## prefers-reduced-motion - Doubles as Performance Signal

Users who enable "Reduce Motion" in iOS/Android accessibility settings also fire this.
Treat as: "give me less GPU work."

```javascript
const prefersReduced = window.matchMedia('(prefers-reduced-motion: reduce)').matches

if (prefersReduced) {
  renderer.setPixelRatio(1)
  disablePostProcessing()
  disableParticles()
  // Still render, but use minimal / static scene
}

// Listen for changes at runtime
window.matchMedia('(prefers-reduced-motion: reduce)')
  .addEventListener('change', (e) => {
    if (e.matches) setLowQuality()
    else restoreQuality()
  })
```

## Complete Battery-Aware Setup

```javascript
async function initBatteryAwareness() {
  // 1. Visibility (universal)
  document.addEventListener('visibilitychange', () => {
    document.hidden ? stopRendering() : startRendering()
  })

  // 2. Reduced motion (universal)
  if (window.matchMedia('(prefers-reduced-motion: reduce)').matches) {
    setQualityLevel('low')
  }

  // 3. Battery (Android Chrome only, optional)
  try {
    if ('getBattery' in navigator) {
      const battery = await navigator.getBattery()
      if (!battery.charging && battery.level < 0.15) {
        setQualityLevel('low')
      }
    }
  } catch (e) {
    // Battery API unavailable - ignore
  }
}
```
