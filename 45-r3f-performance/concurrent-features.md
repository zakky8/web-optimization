# React 18/19 Concurrent Features with React Three Fiber

## Background: How R3F Uses Concurrent Mode

Since R3F v8, the `<Canvas>` component runs React in **concurrent mode by default**. There is no `mode="concurrent"` prop to pass — it is automatic when paired with React 18+.

**Exact quote from maintainer (Discussion #2164):** "Concurrency is now part of React 18, which automatically switches between blocking (default) and concurrent (async)."

In concurrent mode React can **pause, interrupt, and restart renders** to keep the UI responsive. For R3F this means:
- Expensive scene-graph reconciliation can be deferred without blocking canvas frame delivery.
- React prioritizes user-input responses over background scene updates.
- Suspense boundaries can display fallbacks while async resources (GLTFs, textures, HDRs) load.

**Source:** pmndrs/react-three-fiber Discussion #2164 (v8.x.x release notes), accessed 2026-05-26.
**Source:** r3f.docs.pmnd.rs/advanced/scaling-performance, accessed 2026-05-26.

---

## `startTransition` for Heavy Scene Changes

`startTransition` marks a state update as a **non-urgent transition**. React will start the render but can interrupt it to handle urgent updates (pointer events, keyboard input) and restart when idle. This prevents a heavy scene swap from causing visible input lag.

### Scene swap without `startTransition` (blocks UI)

```jsx
function SceneSelector() {
  const [scene, setScene] = useState('simple')

  return (
    <>
      <button onClick={() => setScene('heavy')}>Load Heavy Scene</button>
      {/* Clicking this button may freeze the UI for several frames
          while React reconciles the new 3D scene graph */}
      <Canvas>
        {scene === 'heavy' ? <HeavyScene /> : <SimpleScene />}
      </Canvas>
    </>
  )
}
```

### Scene swap with `startTransition` (non-blocking)

```jsx
import { startTransition, useState } from 'react'

function SceneSelector() {
  const [scene, setScene] = useState('simple')

  const switchScene = () => {
    startTransition(() => {
      setScene('heavy')   // React treats this as interruptible, low-priority
    })
  }

  return (
    <>
      <button onClick={switchScene}>Load Heavy Scene</button>
      <Canvas>
        {scene === 'heavy' ? <HeavyScene /> : <SimpleScene />}
      </Canvas>
    </>
  )
}
```

The canvas keeps rendering at its existing framerate while React progressively reconciles the new scene graph in background slices.

**Source:** pmndrs/react-three-fiber Discussion #2164, accessed 2026-05-26.
**Source:** r3f.docs.pmnd.rs/advanced/pitfalls, accessed 2026-05-26.

---

## `useTransition` for Loading State UI

`useTransition` is the hook form of `startTransition`. It returns `[isPending, startTransition]`, where `isPending` is `true` while the transition is still in progress. Use it to show a loading indicator without blocking the canvas.

```jsx
import { useTransition, useState, Suspense } from 'react'
import { useGLTF } from '@react-three/drei'

function ModelViewer() {
  const [modelUrl, setModelUrl] = useState('/models/low.glb')
  const [isPending, startTransition] = useTransition()

  const loadHighRes = () => {
    startTransition(() => {
      setModelUrl('/models/high.glb')
    })
  }

  return (
    <>
      <button onClick={loadHighRes} disabled={isPending}>
        {isPending ? 'Loading...' : 'Load High-Res Model'}
      </button>
      <Canvas>
        <Suspense fallback={<LowResModel />}>
          <Model url={modelUrl} />
        </Suspense>
      </Canvas>
    </>
  )
}

function Model({ url }) {
  const { scene } = useGLTF(url)
  return <primitive object={scene} />
}
```

`isPending` is `true` from the moment `startTransition` fires until the new `<Model>` finishes loading and mounting. The `Suspense` boundary keeps the old model visible until then.

---

## Practical Pattern: Geometry Computation Deferral

When user input drives an expensive geometry recomputation (e.g., adjusting a terrain resolution slider), wrapping the update in `startTransition` keeps the input responsive:

```jsx
import { startTransition, useMemo, useState } from 'react'

function TerrainControls() {
  const [resolution, setResolution] = useState(64)

  const handleSlider = (e) => {
    const value = Number(e.target.value)
    startTransition(() => {
      setResolution(value)
    })
  }

  return (
    <>
      <input type="range" min={16} max={512} step={16} onChange={handleSlider} />
      <Canvas>
        <Terrain resolution={resolution} />
      </Canvas>
    </>
  )
}

function Terrain({ resolution }) {
  // This expensive computation is deferred — React can interrupt and restart it
  const geometry = useMemo(
    () => new THREE.PlaneGeometry(100, 100, resolution, resolution),
    [resolution]
  )
  return <mesh geometry={geometry}><meshStandardMaterial wireframe /></mesh>
}
```

**Source:** dawchihliou.github.io/articles/stress-testing-concurrent-features-in-react-18, accessed 2026-05-26.

---

## `useDeferredValue` for Texture/Model Transitions

`useDeferredValue` keeps the previous value rendering while React prepares the new one in the background. It is a lower-level alternative to `useTransition` when you do not control where the state update originates.

```jsx
import { useDeferredValue, useState } from 'react'
import { useTexture } from '@react-three/drei'

function TextureSwitcher() {
  const [textureUrl, setTextureUrl] = useState('/textures/default.jpg')
  const deferredUrl = useDeferredValue(textureUrl)
  // deferredUrl lags behind textureUrl while the new texture loads
  // The mesh continues rendering with the old texture during the transition

  return (
    <>
      <button onClick={() => setTextureUrl('/textures/high-res.jpg')}>
        Switch Texture
      </button>
      <Canvas>
        <Suspense fallback={null}>
          <TexturedMesh url={deferredUrl} />
        </Suspense>
      </Canvas>
    </>
  )
}

function TexturedMesh({ url }) {
  const texture = useTexture(url)
  return (
    <mesh>
      <planeGeometry args={[2, 2]} />
      <meshBasicMaterial map={texture} />
    </mesh>
  )
}
```

---

## Nested `Suspense` for Progressive Loading

R3F supports nested Suspense boundaries. The outer boundary handles the page shell; inner boundaries handle progressive quality upgrades within the canvas:

```jsx
function App() {
  return (
    <Suspense fallback={<LoadingScreen />}>      {/* outer: page-level */}
      <Canvas>
        <Suspense fallback={<LowQualityScene />}>  {/* inner: quality upgrade */}
          <HighQualityScene />
        </Suspense>
      </Canvas>
    </Suspense>
  )
}

function HighQualityScene() {
  // useGLTF suspends until the file is loaded and parsed
  const { scene } = useGLTF('/models/high-res.glb')
  return <primitive object={scene} />
}

function LowQualityScene() {
  const { scene } = useGLTF('/models/low-res.glb')
  return <primitive object={scene} />
}
```

**Source:** r3f.docs.pmnd.rs/advanced/scaling-performance, accessed 2026-05-26.

---

## Priority-Based Rendering Strategy

R3F's `useFrame` hook accepts a **render priority** as its second argument. When any subscriber has a priority greater than zero, **automatic rendering is disabled** and you are responsible for issuing render calls. Priorities are executed in ascending order (lowest number first), so you can chain post-processing passes:

```jsx
import { useFrame, useThree } from '@react-three/fiber'
import { EffectComposer } from 'three/examples/jsm/postprocessing/EffectComposer'

function PostProcessingPipeline() {
  const { gl, scene, camera } = useThree()
  const composer = useRef()

  // Priority 1 — renders the scene
  useFrame(() => {
    gl.render(scene, camera)
  }, 1)

  // Priority 2 — runs post-processing after the scene render
  useFrame(() => {
    composer.current?.render()
  }, 2)

  return null
}
```

**Caution:** as soon as any `useFrame` in the tree has a non-zero priority, ALL auto-rendering stops. Every pass that needs to render must now explicitly call `gl.render()` or the composer. This is an advanced escape hatch, not a routine optimization.

**Source:** gracious-keller-98ef35.netlify.app/docs/api/hooks/useframe/ (legacy R3F docs, still accurate for priority mechanics), accessed 2026-05-26.

---

## Deferring Non-Critical Updates (Practical Checklist)

These patterns are the correct React 18 concurrent approach for common R3F scenarios:

| Scenario | Recommended pattern |
|----------|---------------------|
| Swapping a heavy scene on button click | `startTransition(() => setScene(next))` |
| Showing "loading" while scene changes | `useTransition` → `isPending` for spinner |
| Texture/model quality upgrade | `useDeferredValue(url)` keeps old version visible |
| LOD swap on slider change | `startTransition` around slider state update |
| Canvas visible while heavy GLTF loads | Nested `Suspense` with low-res fallback |
| Post-processing pipeline ordering | `useFrame(fn, priority)` — non-zero priority disables auto-render |
| Pointer events during scene rebuild | Automatic in concurrent mode — React yields to pointer events |

---

## What Concurrent Features Do Not Cover

Concurrent features handle React reconciliation cost — the JS work of building the scene graph and diffing components. They do **not** help with:

- GPU frame time (overdraw, fill rate, shader complexity)
- Three.js operations inside `useFrame` — these always run synchronously per frame
- Asset decompression latency on the main thread (use `useLoader` with Workers or offload to a worker via `draco-loader` async decode)
- Memory pressure from keeping both old and new scenes alive during a `Suspense` transition

For frame-time problems, use `PerformanceMonitor` (drei), adaptive DPR, and instancing. For asset decompression, use Draco/Meshopt compression with worker-based loaders.

---

*Sources verified against pmndrs/react-three-fiber Discussion #2164, r3f.docs.pmnd.rs, and react.dev on 2026-05-26.*
