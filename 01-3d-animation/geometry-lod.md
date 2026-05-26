# Geometry Compression and LOD

## DRACO Compression

Reduces geometry file size by 50-70% via quantization.
No quality loss at standard settings.

```bash
# Export via gltfjsx (recommended pipeline)
npx gltfjsx model.gltf -S -T -t
# Handles Draco + KTX2 + simplification in one command

# Manual Draco export from Blender
# File -> Export -> glTF -> Draco mesh compression: ON
```

Three.js loading:
```javascript
import { DRACOLoader } from 'three/addons/loaders/DRACOLoader.js'
import { GLTFLoader } from 'three/addons/loaders/GLTFLoader.js'

const dracoLoader = new DRACOLoader()
dracoLoader.setDecoderPath('/libs/draco/')
dracoLoader.preload()  // start WASM decoder download immediately

const gltfLoader = new GLTFLoader()
gltfLoader.setDRACOLoader(dracoLoader)
```

Bruno Simon: "DRACO was a real game changer."

## LOD - Level of Detail

Switch geometry complexity based on camera distance.

```javascript
// Three.js built-in
const lod = new THREE.LOD()
lod.addLevel(highPolyMesh, 0)    // < 10 units
lod.addLevel(midPolyMesh, 10)    // 10-50 units
lod.addLevel(lowPolyMesh, 50)    // > 50 units
scene.add(lod)
lod.update(camera)  // call every frame

// R3F / Drei
import { Detailed } from '@react-three/drei'
<Detailed distances={[0, 10, 20]}>
  <HighPoly />
  <MidPoly />
  <LowPoly />
</Detailed>
```

Production gain: 40% draw call reduction in large environments (Three.js forum)

## Lusion Cloth Simulation Technique

Problem: 66-frame Houdini cloth animation is too large for web.
Solution: 11 keyframes stored, values interpolated in real-time in the shader.
Encoding: 32-bit floats -> 16-bit integers with divisor, stored in ArrayBuffer.

Results:
- Desktop model: 983KB (4,096 vertices)
- Mobile model: 246KB (1,024 vertices)
- Full animation data: 220KB gzipped

This technique applies to any pre-baked animation (character, fluid, cloth):
1. Bake in Houdini/Blender
2. Export sparse keyframes only
3. Quantize to 16-bit integers
4. Store in ArrayBuffer binary
5. Interpolate in vertex shader at runtime

## Geometry Merging

Merge static geometries that share a material into a single BufferGeometry.

```javascript
import { mergeGeometries } from 'three/addons/utils/BufferGeometryUtils.js'

const merged = mergeGeometries([geomA, geomB, geomC])
const mesh = new THREE.Mesh(merged, sharedMaterial)
// 3 draw calls -> 1 draw call
```

Only for truly static objects. Cannot merge objects that animate independently.
