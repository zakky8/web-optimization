# Android Chrome vs iOS Safari

## Key Differences

| Issue | iOS Safari | Android Chrome |
|-------|-----------|----------------|
| WebGL memory limit | 256MB hard (WebKit enforced) | None - OOM killer at process level |
| Context loss on background | Yes - iOS 17 bug (fixed 17.1.1) | Rare, usually resource pressure |
| ASTC texture support | A8+ (all modern iPhone/iPad) | Adreno 4xx+, Mali T624+ |
| ETC2 support | Yes (all iOS via Metal) | All Android OpenGL ES 3.0+ |
| OES_texture_float_linear | Frequently absent | Present on most Adreno/Mali |
| WEBGL_draw_buffers | Absent on most | Present on Adreno 5xx+ |
| Canvas resize leak | Documented WebKit bug | Not on Blink |
| requestIdleCallback | NOT SUPPORTED | Chrome 47+ |
| Battery Status API | NOT SUPPORTED | Chrome 79+ (HTTPS only) |
| Network Information API | NOT SUPPORTED | Chrome only |
| Gyroscope permission | requestPermission() required | Auto-granted |
| powerPreference hint | No-op (single GPU iPhone) | Honored on Snapdragon dual-GPU |

## Android Advantage: No Canvas Memory Cap

Android's GPU process is killed by OS OOM killer at the process level.
This gives significantly more headroom than iOS's 256MB hard limit.
Mid-range 2024 Android 6GB RAM: typically 2-3GB browser GPU usage allowed.

But: FRAGMENTATION. Cannot assume any extension present on Android.
Always query, never assume.

```javascript
const gl = renderer.getContext()

// Check every extension before use
const ext = {
  floatLinear:  gl.getExtension('OES_texture_float_linear'),
  drawBuffers:  gl.getExtension('WEBGL_draw_buffers'),
  colorBuffer:  gl.getExtension('EXT_color_buffer_float'),
  instancing:   gl.getExtension('ANGLE_instanced_arrays'),  // WebGL1 only
}

// Example: conditionally enable SSAO only if float texture supported
if (ext.floatLinear && ext.colorBuffer) {
  enableSSAO()
} else {
  disableSSAO()
}
```

## Android-Specific Performance Factors

Snapdragon Adreno GPUs (high-end Android):
- Excellent OpenCL/compute shader support
- Good ASTC support on 400 series and up
- WebGPU partially supported in Chrome 113+

Mali GPUs (Samsung mid-range):
- Lower clock speeds
- ETC2 always supported (OpenGL ES 3.0+)
- Some ASTC gaps on older Mali T-series

MediaTek/Dimensity GPUs:
- Treat as low-tier for WebGL
- Minimal extension support
- Use tier 1 settings

## Detecting Android GPU Type

```javascript
const gl = renderer.getContext()
const debugInfo = gl.getExtension('WEBGL_debug_renderer_info')
if (debugInfo) {
  const renderer_str = gl.getParameter(debugInfo.UNMASKED_RENDERER_WEBGL)
  // "Adreno (TM) 650"
  // "Mali-G77"
  // "Apple GPU"

  const isAdreno = /Adreno/i.test(renderer_str)
  const isMali = /Mali/i.test(renderer_str)
  const isApple = /Apple/i.test(renderer_str)
}
// Note: some browsers block this for privacy reasons
// Use @pmndrs/detect-gpu as the reliable cross-platform solution instead
```

## Android Chrome Permissions Model

Unlike iOS, Android Chrome auto-grants:
- DeviceOrientationEvent (gyroscope)
- DeviceMotionEvent (accelerometer)
- No user gesture required

But Android still requires HTTPS for:
- Battery Status API
- Network Information API
- Vibration API
- Web Bluetooth / Web USB
