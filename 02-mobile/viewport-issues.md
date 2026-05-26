# Mobile Viewport Issues

## Address Bar Show/Hide

Mobile browsers' dynamic toolbar changes the viewport height as user scrolls.
100vh is based on the large viewport (toolbar hidden) but toolbar is visible on load.
Result: canvas overshoots by ~56px (Safari toolbar height) on initial load.

Fix with viewport units (iOS 15.4+, Safari 15.4+):
```css
canvas {
  height: 100svh;   /* small viewport: always assumes toolbar visible */
}
```

Or with CSS custom property updated via JS:
```javascript
function setVhVariable() {
  const vh = window.innerHeight * 0.01
  document.documentElement.style.setProperty('--vh', `${vh}px`)
}
setVhVariable()
window.addEventListener('resize', setVhVariable)
```
```css
canvas { height: calc(var(--vh, 1vh) * 100); }
```

## Only React to Width Changes on Mobile

Address bar showing/hiding fires resize events constantly.
Only reinit renderer when actual width changes (orientation change).

```javascript
let lastWidth = window.innerWidth

window.addEventListener('resize', () => {
  if (window.innerWidth === lastWidth) return  // height-only change, ignore
  lastWidth = window.innerWidth

  camera.aspect = window.innerWidth / window.innerHeight
  camera.updateProjectionMatrix()
  renderer.setSize(window.innerWidth, window.innerHeight)
}, { passive: true })
```

## visualViewport - More Stable Than window.innerHeight

```javascript
function getCanvasSize() {
  // visualViewport accounts for zoom, keyboard, and on-screen UI
  const vp = window.visualViewport
  return {
    width: vp ? vp.width : window.innerWidth,
    height: vp ? vp.height : window.innerHeight
  }
}

// Supported: iOS 13+, Android Chrome 61+
window.visualViewport?.addEventListener('resize', () => {
  const { width, height } = getCanvasSize()
  renderer.setSize(width, height)
  camera.aspect = width / height
  camera.updateProjectionMatrix()
})
```

## Orientation Change

For orientation change (portrait <-> landscape), use screen.orientation:

```javascript
screen.orientation?.addEventListener('change', () => {
  // Small delay to let browser settle new dimensions
  setTimeout(() => {
    const w = window.innerWidth
    const h = window.innerHeight
    renderer.setSize(w, h)
    camera.aspect = w / h
    camera.updateProjectionMatrix()
  }, 100)
})

// Fallback for older browsers
window.addEventListener('orientationchange', () => {
  setTimeout(() => {
    renderer.setSize(window.innerWidth, window.innerHeight)
    camera.aspect = window.innerWidth / window.innerHeight
    camera.updateProjectionMatrix()
  }, 200)  // longer delay needed for orientationchange event
})
```

## iOS Safe Area Insets (Notch / Home Bar)

```css
canvas {
  /* Extend behind notch but pad content away from it */
  padding-top: env(safe-area-inset-top);
  padding-bottom: env(safe-area-inset-bottom);
  padding-left: env(safe-area-inset-left);
  padding-right: env(safe-area-inset-right);
}

/* For full-screen canvas behind notch */
body {
  background: #000;
}
meta viewport: viewport-fit=cover  /* allows extending behind notch */
```

```html
<meta name="viewport" content="width=device-width, initial-scale=1, viewport-fit=cover">
```
