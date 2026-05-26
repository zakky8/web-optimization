# Instancing in React Three Fiber

## Why Instancing

Every discrete Three.js mesh produces one draw call. GPUs are throughput machines — the CPU-to-GPU dispatch overhead of thousands of separate draw calls dominates frame time long before the actual vertex or pixel work does. Instancing collapses N draw calls into one by sending the geometry and material to the GPU once and issuing a single draw command with a buffer of per-instance transform matrices.

**Target from official docs:** Keep total draw calls below 1000; optimally a few hundred or less for interactive scenes.

**Source:** r3f.docs.pmnd.rs/advanced/scaling-performance, accessed 2026-05-26.

---

## Raw `instancedMesh` in R3F (Lowest Overhead)

The native Three.js `<instancedMesh>` JSX element accepts `args={[geometry, material, count]}` and exposes a `ref` for imperative matrix updates. This path has zero CPU overhead per instance at render time — the GPU does all the work.

### Static placement

```jsx
import { useRef, useEffect } from 'react'
import { useFrame } from '@react-three/fiber'
import * as THREE from 'three'

const TEMP = new THREE.Object3D()   // reuse — never allocate inside loop

function Forest({ count = 100_000 }) {
  const meshRef = useRef()

  useEffect(() => {
    for (let i = 0; i < count; i++) {
      TEMP.position.set(
        (Math.random() - 0.5) * 200,
        0,
        (Math.random() - 0.5) * 200
      )
      TEMP.scale.setScalar(0.5 + Math.random())
      TEMP.updateMatrix()
      meshRef.current.setMatrixAt(i, TEMP.matrix)
    }
    meshRef.current.instanceMatrix.needsUpdate = true
  }, [count])

  return (
    <instancedMesh ref={meshRef} args={[null, null, count]}>
      <coneGeometry args={[0.5, 2, 6]} />
      <meshLambertMaterial color="forestgreen" />
    </instancedMesh>
  )
}
```

`args={[null, null, count]}` passes `null` for geometry and material because they are defined as children via JSX. The third argument is the max instance count — it pre-allocates the matrix buffer.

**Source:** r3f.docs.pmnd.rs/advanced/scaling-performance, accessed 2026-05-26.

### Per-instance animation with `useFrame` and `setMatrixAt`

```jsx
import { useRef, useEffect } from 'react'
import { useFrame } from '@react-three/fiber'
import * as THREE from 'three'

const TEMP = new THREE.Object3D()
const MATRIX = new THREE.Matrix4()

function AnimatedParticles({ count = 5_000 }) {
  const meshRef = useRef()

  // Initialise positions once
  useEffect(() => {
    for (let i = 0; i < count; i++) {
      TEMP.position.set(
        (Math.random() - 0.5) * 40,
        (Math.random() - 0.5) * 40,
        (Math.random() - 0.5) * 40
      )
      TEMP.updateMatrix()
      meshRef.current.setMatrixAt(i, TEMP.matrix)
    }
    meshRef.current.instanceMatrix.needsUpdate = true
  }, [count])

  // Update per frame
  useFrame(({ clock }) => {
    const t = clock.elapsedTime
    for (let i = 0; i < count; i++) {
      meshRef.current.getMatrixAt(i, MATRIX)
      // Decompose, mutate Y, recompose
      MATRIX.decompose(TEMP.position, TEMP.quaternion, TEMP.scale)
      TEMP.position.y += Math.sin(t + i * 0.1) * 0.002
      TEMP.updateMatrix()
      meshRef.current.setMatrixAt(i, TEMP.matrix)
    }
    meshRef.current.instanceMatrix.needsUpdate = true
  })

  return (
    <instancedMesh ref={meshRef} args={[null, null, count]}>
      <sphereGeometry args={[0.1, 8, 8]} />
      <meshBasicMaterial color="cyan" />
    </instancedMesh>
  )
}
```

**Key requirement:** set `instanceMatrix.needsUpdate = true` after every batch of `setMatrixAt` calls, or Three.js will not upload the updated buffer to the GPU.

---

## Drei `<Instances>` + `<Instance>` Pattern (Declarative)

`@react-three/drei` wraps `THREE.InstancedMesh` with a declarative API. Each `<Instance>` is a React component that can receive position, rotation, scale, color, and event handlers as props. This unlocks standard React patterns (conditional rendering, arrays, context) on instanced objects at the cost of per-instance CPU reconciliation overhead.

**When to use:** hundreds to low thousands of instances that need React-managed state, pointer events, or conditional rendering. For tens of thousands of stationary objects, prefer raw `instancedMesh`.

```jsx
import { Instances, Instance } from '@react-three/drei'

function Crowd({ positions }) {
  return (
    <Instances limit={positions.length}>
      <capsuleGeometry args={[0.25, 1, 4, 8]} />
      <meshStandardMaterial />
      {positions.map((pos, i) => (
        <Instance
          key={i}
          position={pos}
          color={i % 2 === 0 ? 'tomato' : 'skyblue'}
          onClick={() => console.log('clicked instance', i)}
        />
      ))}
    </Instances>
  )
}
```

**Props on `<Instances>`:**
- `limit` — maximum instance count; sets the buffer size. Required when instance count changes dynamically.
- `range` — how many instances to actually draw (can be less than `limit`).

**Props on `<Instance>`:**
- `position`, `rotation`, `scale` — transform values (same signature as `<mesh>`)
- `color` — per-instance color (requires `vertexColors` on the material — Drei handles this automatically)
- All standard R3F event handlers: `onClick`, `onPointerOver`, `onPointerOut`, etc.
- Custom attributes when combined with `<InstancedAttribute>`

**Source:** drei.docs.pmnd.rs/performances/instances, accessed 2026-05-26.

---

## `createInstances()` for Multiple Instance Types

When a scene needs two or more distinct instanced meshes that must coexist without interference, `createInstances()` provides isolated `[Provider, Component]` pairs:

```jsx
import { createInstances } from '@react-three/drei'

const [CubeInstances, Cube] = createInstances()
const [SphereInstances, Sphere] = createInstances()

function Scene() {
  return (
    <>
      <CubeInstances limit={500}>
        <boxGeometry />
        <meshStandardMaterial color="orange" />
        {cubePositions.map((pos, i) => <Cube key={i} position={pos} />)}
      </CubeInstances>

      <SphereInstances limit={200}>
        <sphereGeometry args={[0.5, 16, 16]} />
        <meshStandardMaterial color="teal" />
        {spherePositions.map((pos, i) => <Sphere key={i} position={pos} />)}
      </SphereInstances>
    </>
  )
}
```

**Source:** drei.docs.pmnd.rs/performances/instances, accessed 2026-05-26.

---

## `<InstancedAttribute>` for Custom Shader Data

Drei's `<InstancedAttribute>` lets you pass arbitrary per-instance data to GLSL shaders without manual buffer management:

```jsx
import { Instances, Instance, InstancedAttribute } from '@react-three/drei'

function ShaderInstances({ data }) {
  return (
    <Instances limit={data.length}>
      <boxGeometry />
      <shaderMaterial
        vertexShader={`
          attribute float brightness;
          varying float vBrightness;
          void main() {
            vBrightness = brightness;
            gl_Position = projectionMatrix * modelViewMatrix * instanceMatrix * vec4(position, 1.0);
          }
        `}
        fragmentShader={`
          varying float vBrightness;
          void main() {
            gl_FragColor = vec4(vec3(vBrightness), 1.0);
          }
        `}
      />
      <InstancedAttribute name="brightness" defaultValue={1.0} />
      {data.map((d, i) => (
        <Instance key={i} position={d.position} brightness={d.brightness} />
      ))}
    </Instances>
  )
}
```

`defaultValue` determines the stride of the buffer (scalar float, `[r, g, b]` array, etc.).

**Source:** drei.docs.pmnd.rs/performances/instances, accessed 2026-05-26.

---

## Dynamic Instance Count Changes

Changing the number of instances at runtime requires `limit` to be set to the **maximum possible count** upfront, with `range` used to control how many are drawn. Changing `limit` after mount deallocates and reallocates the GPU buffer.

```jsx
function DynamicCloud({ positions }) {
  const MAX = 10_000   // pre-allocate for worst case

  return (
    <Instances limit={MAX} range={positions.length}>
      <sphereGeometry args={[0.05, 4, 4]} />
      <meshBasicMaterial />
      {positions.map((p, i) => <Instance key={i} position={p} />)}
    </Instances>
  )
}
```

For raw `instancedMesh`, set `count` (not `args[2]`) dynamically and mark `instanceMatrix.needsUpdate`:

```jsx
useEffect(() => {
  meshRef.current.count = newCount
  meshRef.current.instanceMatrix.needsUpdate = true
}, [newCount])
```

---

## Performance Comparison

| Approach | Draw calls | CPU overhead per frame | React events | Custom shader attribs |
|----------|-----------|----------------------|--------------|----------------------|
| Separate `<mesh>` per object | N | Low per mesh | Yes | Yes |
| Raw `<instancedMesh>` | 1 | Minimal | No (manual raycasting) | Manual |
| Drei `<Instances>` + `<Instance>` | 1 | Per-instance reconciliation | Yes | Via `InstancedAttribute` |
| `createInstances()` | 1 per type | Same as Instances | Yes | Via `InstancedAttribute` |

---

*Sources verified against pmndrs/react-three-fiber master, drei.docs.pmnd.rs, and r3f.docs.pmnd.rs on 2026-05-26.*
