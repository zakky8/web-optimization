# Rendering Pipeline Optimization

## The Hard Rule: Draw Call Budget

- Target: under 100 draw calls per frame for 60fps
- Absolute ceiling: 1,000 draw calls before degradation regardless of hardware
- Real case study: 2,200 -> 17 draw calls via instancing + atlasing = 60fps -> 1,500fps (Dhia Shakiry)

## GPU Instancing - Collapse N Objects to 1 Draw Call

```javascript
// Three.js - N identical objects = 1 draw call
const mesh = new THREE.InstancedMesh(geometry, material, count)
mesh.setMatrixAt(index, matrix)
mesh.instanceMatrix.needsUpdate = true

// InstancedMesh2 (third-party) - adds per-instance frustum culling
// (missing from built-in Three.js InstancedMesh)

// R3F equivalent
import { Instances, Instance } from '@react-three/drei'
<Instances limit={1000}>
  <boxGeometry />
  <meshStandardMaterial />
  <Instance position={[x, y, z]} />
</Instances>

// BatchedMesh - for varied geometries sharing one material (Three.js r159+)
const batched = new THREE.BatchedMesh(maxGeometries, maxVertices, maxIndices, material)
```

Production gain: 80%+ scene weight reduction (Chipsa RSI project confirmed)

## Frustum + Occlusion Culling

```javascript
// Built-in frustum culling (enabled by default)
object.frustumCulled = true   // skip off-screen objects

// Per-instance culling requires InstancedMesh2 or manual BVH
// three-mesh-bvh (used by Igloo) for spatial partitioning
import { MeshBVH, acceleratedRaycast } from 'three-mesh-bvh'
THREE.Mesh.prototype.raycast = acceleratedRaycast
const bvh = new MeshBVH(geometry)

// LOD - Level of Detail
// R3F/Drei
import { Detailed } from '@react-three/drei'
<Detailed distances={[0, 10, 20]}>
  <HighPolyMesh />    // 0-10 units
  <MidPolyMesh />     // 10-20 units
  <LowPolyMesh />     // 20+ units
</Detailed>
```

Lusion real numbers: 4,096 verts (desktop) -> 1,024 verts (mobile) = 4x reduction

## Frame Loop Control

```javascript
// R3F - only render when something changes
<Canvas frameloop="demand">
// call invalidate() from useThree() to trigger a render

// Three.js - stop loop when tab hidden (saves battery)
document.addEventListener('visibilitychange', () => {
  if (document.hidden) {
    cancelAnimationFrame(animId)
  } else {
    animId = requestAnimationFrame(loop)
  }
})
```

## Shader Precompilation - Eliminates First-Frame Stutter

Without this: 1-4 second GPU freeze on first render of complex scenes.

```javascript
// Precompile ALL shaders before revealing scene
// Uses KHR_parallel_shader_compile extension - async, no freeze
await renderer.compileAsync(scene, camera)
// Put this behind your loading screen
```

## Adaptive Quality - R3F PerformanceMonitor

```javascript
import { PerformanceMonitor } from '@react-three/drei'

function Scene() {
  const [dpr, setDpr] = useState(2)

  return (
    <PerformanceMonitor
      onDecline={() => setDpr(d => Math.max(1, d - 0.2))}
      onIncline={() => setDpr(d => Math.min(2, d + 0.2))}
      flipflops={3}
      onFallback={() => setLowQuality(true)}
    >
      <Canvas dpr={dpr}>
        ...
      </Canvas>
    </PerformanceMonitor>
  )
}
```

## Baked Lighting (Bruno Simon / Lusion Approach)

No real-time lights or dynamic shadows.
- Bake everything to PNG-8 in Blender
- Use matcap textures for surface lighting (one texture lookup, zero GPU cost)
- Rule: ONE shadow map = rendering the entire scene a second time

Bruno Simon '25: no lights, no dynamic shadows.
Matcap textures simulate lighting via camera-relative normal lookup on a 2D image.
Static shadows baked to PNG-8 files in Blender.

Lusion: pre-rendered material maps (normal, AO, thickness).
Matcap for translucent objects. Only one real-time point light (cursor tracking).
