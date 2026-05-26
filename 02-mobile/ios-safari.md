# iOS Safari Specific Limitations and Fixes

## Hard Memory Limit: 256MB WebGL Canvas

iOS enforces a hard 256MB limit for all WebGL allocations on a canvas.
When exceeded: tab crashes and force-reloads.

```
Error: "Total canvas memory use exceeds the maximum limit (256mb)"
WebGL: INVALID_VALUE: texImage2D: no canvas
```

Budget math:
- 2048x2048 RGBA uncompressed = 16MB GPU
- Ten textures = 160MB = 62% of entire budget
- Same in KTX2/ASTC = ~2MB each (8x reduction)

Rules:
- 1K textures max on older iOS devices
- 2K risky on iPhone 3GB RAM
- 4K crashes most devices
- KTX2 is MANDATORY on iOS, not optional

## Canvas Resize Memory Leak (WebKit Bug #219780)

Resizing a WebGL canvas by writing .width or .height leaks GPU memory on iOS.
Affects every orientation change. Fixed in iOS 14.3 but resurfaces.

Workaround:
```javascript
// Render into offscreen canvas, blit to 2D canvas
const offscreen = document.createElement('canvas')
const glCtx = offscreen.getContext('webgl2')
const display = document.getElementById('display-canvas')
const ctx2d = display.getContext('2d')

function onResize() {
  offscreen.width = window.innerWidth
  offscreen.height = window.innerHeight
  renderer.setSize(offscreen.width, offscreen.height)
}

function render() {
  renderer.render(scene, camera)
  ctx2d.drawImage(offscreen, 0, 0)  // blit per frame
  requestAnimationFrame(render)
}
```

Performance cost: one extra blit per frame. Only apply for iOS 14 and earlier.

## Viewport Units - Use svh Not vh

Traditional 100vh includes address bar, causing layout shifts.
Fixed units available since iOS 15.4 / Safari 15.4 (March 2022):

- svh = small viewport: toolbar always visible (use for canvas)
- lvh = large viewport: toolbar hidden
- dvh = dynamic: updates in real-time but throttled (NOT for animations)

```css
canvas {
  width: 100%;
  height: 100svh;   /* safe default */
  display: block;
}
```

## Missing WebGL Extensions on iOS

Always query before use, never assume:

```javascript
const gl = renderer.getContext()

// OES_texture_float_linear - absent on many iOS
// Breaks: SSAO, DOF, custom blur passes using float render targets
const hasFloatLinear = gl.getExtension('OES_texture_float_linear')

// WEBGL_draw_buffers - absent on most mobile
const hasDrawBuffers = gl.getExtension('WEBGL_draw_buffers')

// Check max texture size (4096 on older, 8192+ on newer)
const maxTexSize = gl.getParameter(gl.MAX_TEXTURE_SIZE)

// Check vertex texture units (can be 0 on mobile WebGL1)
const maxVtxTex = gl.getParameter(gl.MAX_VERTEX_TEXTURE_IMAGE_UNITS)
```

If OES_texture_float_linear is absent:
- SSAO pass will fail or produce artifacts
- Must sample float textures manually and interpolate in shader
- Simplest fix: disable SSAO on mobile entirely

## iOS Shader Precision

Desktop silently upgrades mediump to highp. iOS does not.
Bugs invisible in desktop dev will appear on iOS only.

```glsl
precision highp float;    // positions, depth, normals - always highp
// mediump safe ONLY for 0-1 color values
// Never use lowp for anything visual
```

## iOS-Specific Renderer Settings

```javascript
// Detect iOS
const isIOS = /iPhone|iPad|iPod/.test(navigator.userAgent) ||
              (navigator.platform === 'MacIntel' && navigator.maxTouchPoints > 1)

if (isIOS) {
  renderer.setPixelRatio(Math.min(window.devicePixelRatio, 1.5)) // never above 1.5
  renderer.shadowMap.enabled = false   // shadows double render cost
  // Disable: SSAO, DOF, Bloom
  // Reduce: particle count by 50%
  THREE.Object3D.DEFAULT_MATRIX_AUTO_UPDATE = false // Bruno Simon confirmed gain
}
```
