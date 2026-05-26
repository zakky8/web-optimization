# Texture Compression

## The Single Biggest Performance Impact

A 4K PNG = 64MB of GPU VRAM. KTX2 = 6-8MB for the same texture.
This is the #1 optimization for both performance and iOS crash prevention.

## Format Comparison

| Format | GPU Memory (4K) | File Size | Use Case |
|--------|----------------|-----------|----------|
| PNG/JPEG | 64MB+ | Medium | NEVER in production 3D |
| WebP/AVIF | 64MB+ (decompresses on GPU) | Small | 2D overlays only |
| KTX2 / ETC1S | ~6-8MB | ~JPEG size | Color textures, diffuse maps |
| KTX2 / UASTC | ~8-16MB | 1-2x JPEG | Normal maps, data textures |

KTX2 transcodes at load time to native GPU format:
- ASTC on iOS (A8+, all modern iPhones)
- ETC2 on Android (all OpenGL ES 3.0+)
- BC7 on desktop
- RGBA fallback if nothing else available

## Implementation

```javascript
import { KTX2Loader } from 'three/addons/loaders/KTX2Loader.js'
import { GLTFLoader } from 'three/addons/loaders/GLTFLoader.js'

const ktx2Loader = new KTX2Loader()
  .setTranscoderPath('/libs/basis/')
  .detectSupport(renderer)
  // Auto-selects ASTC on iOS, ETC2 on Android, BC7 on desktop

const gltfLoader = new GLTFLoader()
gltfLoader.setKTX2Loader(ktx2Loader)
```

## Converting Textures

```bash
# Install: npm install -g @ktx/toktx

# ETC1S - for color/diffuse textures (smaller file, lower quality)
toktx --bcmp output.ktx2 input.png

# UASTC - for normal maps and detail textures (larger file, higher quality)
toktx --uastc output.ktx2 input.png

# Via gltfjsx pipeline (handles everything)
npx gltfjsx model.gltf -S -T -t
# -S = simplify  -T = web transforms  -t = Draco + KTX2
# Result: up to 90% asset size reduction (Codrops 2025 benchmark)
```

## Texture Atlasing

Batch multiple images into a single 4096x4096 atlas.
Reduces texture switches (= draw calls).

Case study result: 1,100 -> 17 draw calls, 3x FPS improvement (Dhia Shakiry)

Rules:
- Power-of-2 dimensions always (64, 128, 256, 512, 1024, 2048, 4096)
- Pre-generate mipmaps to eliminate distance shimmer
- UV islands must not overlap

## iOS-Specific Budget

iOS hard limit: 256MB total WebGL canvas memory.

- 2048x2048 RGBA uncompressed = 16MB GPU
- Ten of those = 160MB = 62% of entire iOS budget
- Same in KTX2/ASTC = ~2MB each (8x reduction)

Rules for iOS:
- 1K textures max on older devices
- 2K is risky on iPhone with 3GB RAM
- 4K will crash on many devices
- KTX2 is mandatory, not optional, for iOS

## Confirmed Studio Results

- Cartier 365: converted all WebGL images to KTX2 = "overall faster and smoother page load"
- Chipsa: KTX2 across Timeless and Courier Museum for memory efficiency
- Bruno Simon '25: ETC1S and UASTC throughout entire scene
- Basis texture compression case study: 1.5GB VRAM -> 700MB (53% reduction)
