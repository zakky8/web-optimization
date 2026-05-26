# Component Optimization in React Three Fiber

## Core Mental Model

React Three Fiber runs two loops simultaneously:
1. **React's reconciler loop** — re-renders components when state/props change, diffing the virtual DOM and updating the fiber tree.
2. **Three.js render loop** — fires 60 times per second via `useFrame`, completely outside React's scheduler.

Optimization work falls into two separate categories accordingly. Re-render cost is paid once per state change; frame-loop cost is paid 60 times per second. Never conflate them.

**Source:** r3f.docs.pmnd.rs/advanced/pitfalls, accessed 2026-05-26.

---

## `React.memo` for 3D Components

`React.memo` is a higher-order component that memoizes the rendered output and skips re-rendering when props are shallowly equal. It is as applicable to R3F components as to DOM components.

```jsx
import React, { memo } from 'react'
import { useFrame } from '@react-three/fiber'

const StarField = memo(function StarField({ count, seed }) {
  // Expensive geometry computation only runs when count or seed changes
  return (
    <points>
      <bufferGeometry />
      <pointsMaterial size={0.05} />
    </points>
  )
})

// Parent re-renders do not cause StarField to re-render
// unless count or seed actually changes.
```

**Gotcha:** if a parent passes an inline object or function as a prop, `memo` is defeated because the reference changes every render. Fix this by memoizing the prop at the call site.

```jsx
// Broken — new object reference on every parent render
<StarField config={{ count: 1000 }} />

// Fixed — stable reference
const starConfig = useMemo(() => ({ count: 1000 }), [])
<StarField config={starConfig} />
```

**Source:** react.dev/reference/react/memo, accessed 2026-05-26.
**Source:** r3f.docs.pmnd.rs/advanced/pitfalls, accessed 2026-05-26.

---

## `useMemo` for Geometry and Material Creation

Every `new THREE.BufferGeometry()` or `new THREE.Material()` call uploads data to the GPU and triggers shader compilation. Calling them inside a render function without memoization re-uploads on every parent re-render.

### Geometry memoization

```jsx
import { useMemo } from 'react'
import * as THREE from 'three'

function TerrainMesh({ resolution }) {
  const geometry = useMemo(
    () => new THREE.PlaneGeometry(100, 100, resolution, resolution),
    [resolution]   // rebuild only when resolution changes
  )

  return <mesh geometry={geometry}><meshStandardMaterial /></mesh>
}
```

### Material memoization

```jsx
function GlowingSphere({ color }) {
  const material = useMemo(
    () => new THREE.MeshStandardMaterial({ color, emissive: color, emissiveIntensity: 0.3 }),
    [color]
  )

  return (
    <mesh material={material}>
      <sphereGeometry args={[1, 32, 32]} />
    </mesh>
  )
}
```

### Toggling between pre-created geometries

A common pattern caches an array of geometries and switches between them without any GPU upload cost:

```jsx
function MorphingShape() {
  const ref = useRef()
  const [index, setIndex] = useState(0)

  const geometries = useMemo(
    () => [new THREE.BoxGeometry(), new THREE.SphereGeometry(0.785)],
    []
  )

  return (
    <mesh
      ref={ref}
      geometry={geometries[index]}
      onPointerDown={() => setIndex(i => (i + 1) % geometries.length)}
    >
      <meshBasicMaterial color="lime" wireframe />
    </mesh>
  )
}
```

**Source:** sbcode.net/react-three-fiber/use-memo/, accessed 2026-05-26.

### Global-scope resource sharing

Resources defined outside the `Canvas` context are created once per module load, not per component instance. Multiple component instances share the same GPU buffer, eliminating redundancy:

```jsx
import * as THREE from 'three'

// Created once — shared across all Sphere instances
THREE.ColorManagement.enabled = true   // required for r150+ color correctness
const sharedMaterial = new THREE.MeshLambertMaterial({ color: 'red' })
const sharedGeometry = new THREE.SphereGeometry(1, 28, 28)

function Sphere({ position }) {
  return <mesh position={position} geometry={sharedGeometry} material={sharedMaterial} />
}
```

**Caution:** shared materials cannot carry per-instance uniforms without additional setup. Use `useMemo` per component when instances need distinct material properties.

**Source:** r3f.docs.pmnd.rs/advanced/scaling-performance, accessed 2026-05-26.

---

## `useRef` Instead of `useState` for Frequently-Changing Values

The fundamental rule: **never route updates that happen every frame through React state.** Doing so forces R3F through React's reconciler 60 times per second — diffing, executing all hooks, re-rendering children.

**Exact quote from official docs:** "Going through state isn't ideal when dealing with continuous updates... Instead, just mutate, use deltas."

### The anti-pattern

```jsx
// BAD — 60 React re-renders per second
const [rotationX, setRotationX] = useState(0)
useFrame((_, delta) => setRotationX(r => r + delta))
return <mesh rotation-x={rotationX} />
```

### The correct pattern

```jsx
// GOOD — zero React re-renders per second
const meshRef = useRef()
useFrame((_, delta) => {
  meshRef.current.rotation.x += delta
})
return <mesh ref={meshRef} />
```

### Lerp animation without state

```jsx
const meshRef = useRef()
const vec = new THREE.Vector3()   // reuse — do not create inside useFrame

useFrame((_, delta) => {
  meshRef.current.position.lerp(
    vec.set(targetX, targetY, targetZ),
    1 - Math.pow(0.001, delta)   // framerate-independent lerp
  )
})
```

**Note:** allocating `new THREE.Vector3()` inside `useFrame` creates a fresh object 60 times per second, stressing the garbage collector. Hoist it to component scope or a module-level constant.

**Source:** r3f.docs.pmnd.rs/advanced/pitfalls, accessed 2026-05-26.
**Source:** r3f.docs.pmnd.rs/tutorials/basic-animations, accessed 2026-05-26.

### What `useState` is still correct for

Use `useState` for discrete, infrequent state changes that are intended to trigger a visible re-render: `hovered`, `selected`, `visible`, `modelUrl`. These happen at human interaction speed (~1–5 Hz) not render speed.

```jsx
function InteractiveMesh() {
  const [hovered, setHovered] = useState(false)   // fine — fires rarely
  const meshRef = useRef()

  useFrame((_, delta) => {                         // never touches React state
    meshRef.current.rotation.y += delta * (hovered ? 2 : 0.5)
  })

  return (
    <mesh
      ref={meshRef}
      onPointerOver={() => setHovered(true)}
      onPointerOut={() => setHovered(false)}
    >
      <boxGeometry />
      <meshStandardMaterial color={hovered ? 'hotpink' : 'orange'} />
    </mesh>
  )
}
```

---

## Component Structure for Minimal Re-Renders

### Isolate frequently-changing state

If one child needs to hold rapidly-changing state (e.g., a cursor tracker), put that state as close as possible to the consuming component. A state update at a high node in the tree re-renders everything below it.

```jsx
// BAD — App state forces all children to re-render
function App() {
  const [mousePos, setMousePos] = useState({ x: 0, y: 0 })
  return (
    <Canvas>
      <HeavyScene />        {/* re-renders on every mouse move */}
      <Cursor pos={mousePos} />
    </Canvas>
  )
}

// GOOD — mouse state lives only inside Cursor
function App() {
  return (
    <Canvas>
      <HeavyScene />        {/* never re-renders from mouse moves */}
      <Cursor />
    </Canvas>
  )
}

function Cursor() {
  const [mousePos, setMousePos] = useState({ x: 0, y: 0 })
  // ...
}
```

### Avoid conditional mounting of heavy components

Unmounting and re-mounting causes geometry upload, shader recompilation, and texture binding every time. Prefer toggling `visible`:

```jsx
// BAD — recompiles materials on every toggle
{stage === 'detail' && <DetailModel />}

// GOOD — stays in GPU memory
<DetailModel visible={stage === 'detail'} />
```

**Source:** r3f.docs.pmnd.rs/advanced/pitfalls, accessed 2026-05-26.

### Selector-based subscriptions with `useThree`

`useThree` accepts a selector to prevent re-renders when unrelated canvas state changes:

```jsx
// Subscribes to entire state — re-renders on any canvas state change
const state = useThree()

// Only re-renders when camera changes
const camera = useThree(state => state.camera)
```

### Prevent GC pressure in callbacks

Pointer event handlers that use vector math should reuse objects:

```jsx
const point = new THREE.Vector3()

function ClickTracker() {
  return (
    <mesh
      onPointerMove={e => {
        point.copy(e.point)   // reuse the same Vector3
        doSomethingWith(point)
      }}
    />
  )
}
```

---

## Summary Decision Table

| Scenario | Tool |
|----------|------|
| Component renders only when specific props change | `React.memo` |
| Expensive Three.js object created in component | `useMemo` with dependency array |
| Value updated 60× per second in render loop | `useRef` + direct mutation in `useFrame` |
| Discrete UI event (hover, click, visibility) | `useState` |
| Subscribing to canvas state | `useThree(selector)` |
| Shared geometry/material across many instances | Module-scope constant |

---

*Sources verified against pmndrs/react-three-fiber master branch and r3f.docs.pmnd.rs on 2026-05-26.*
