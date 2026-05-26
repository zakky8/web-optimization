# Mobile Memory Limits

## iOS Hard Limits (Empirically Documented)

| Device | RAM | WebGL Canvas Limit | Process Kill Threshold |
|--------|-----|-------------------|----------------------|
| iPhone 256MB era | 1-2GB | 256MB hard | ~700MB process memory |
| iPhone 4GB RAM | 4GB | 256MB hard | ~1.0-1.5GB process memory |
| iPad Air 3 (iOS 14.2) | 3GB | 256MB hard | Documented at 1.25GB |

The 256MB WebGL canvas limit is hard-enforced by WebKit.
Process kill is done by Jetsam (iOS OOM killer) at the process level.

## Android Memory

No documented hard WebGL canvas limit.
Process is killed at OS level by the low-memory killer.
Mid-range 2024 Android with 6GB RAM: typically allows 2-3GB browser GPU usage.

## Compressed Texture Format Support

| Format | iOS (A8+) | Android Modern | GPU Memory Reduction |
|--------|-----------|----------------|---------------------|
| ASTC | Yes (all modern iPhone) | Adreno 4xx+, Mali T624+ | ~8x vs RGBA |
| ETC2 | Yes (via Metal) | All OpenGL ES 3.0+ | ~4x vs RGBA |
| BC7/BPTC | No | No (desktop only) | ~4x |
| PVRTC | Old iOS only | No | ~4x |

Use Basis Universal (KTX2): transcodes to ASTC on iOS, ETC2 on Android, RGBA fallback.
This is the ONLY practical approach for cross-platform GPU compression.

```javascript
import { KTX2Loader } from 'three/addons/loaders/KTX2Loader.js'
const ktx2Loader = new KTX2Loader()
  .setTranscoderPath('/libs/basis/')
  .detectSupport(renderer)
// Automatically selects correct format per device
```

## What Happens When You Exceed Limits

iOS:
- Error: "WebGL: INVALID_VALUE: texImage2D: no canvas"
- Tab crashes and force-reloads
- NO warning before it happens

Chrome/Android:
- webglcontextlost event fires
- Tab is usually killed after

IMPORTANT: There is NO web API to check remaining GPU memory.
No gl.getFreeMem(), no budget query. You must budget conservatively.

## Texture Budget Calculator

```javascript
function textureMB(width, height, channels = 4, compressed = false) {
  if (compressed) return (width * height * channels) / (8 * 1024 * 1024)
  return (width * height * channels) / (1024 * 1024)
}

// 2048x2048 RGBA uncompressed
console.log(textureMB(2048, 2048, 4))       // 16 MB
// 2048x2048 ASTC compressed (approx 8x)
console.log(textureMB(2048, 2048, 4) / 8)   // 2 MB

// iOS budget: 256MB
// 10 uncompressed 2K textures = 160MB = 62% of budget
// 10 KTX2/ASTC 2K textures = 20MB = 8% of budget
```

## Practical Limits for iOS Scenes

- Max uncompressed textures: 10-12 at 2K before hitting budget
- With KTX2/ASTC: up to 80-100 textures at 2K within budget
- Geometry: less concern than textures (stored in system RAM not GPU VRAM)
- Render targets (post-processing): each full-screen RT at 1x DPR = ~8MB
  - 5 RT passes = 40MB (add to texture budget)
  - This is why disabling post-processing on mobile is so important
