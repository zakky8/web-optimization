# Dynamic Imports, Suspense, and Code Splitting for Three.js in Next.js 15

> Sources verified 2026-05-26 against Next.js 16.2.6 docs, R3F docs, and pmndrs discussions.

---

## Why Three.js Needs Explicit Code Splitting

Three.js is a large library. The full build is approximately 600 KB minified (160 KB gzipped). `@react-three/drei` adds another 200+ KB depending on which helpers you use. Without deliberate code splitting, all of this lands in your initial page bundle — even for users who never scroll to the 3D section.

Goals when splitting:
- Defer Three.js loading until the user needs it (or until the rest of the page is interactive)
- Load heavy assets (GLTF models, HDR environment maps, textures) after initial render
- Give users a fallback UI while 3D content loads

---

## `next/dynamic` — The Primary Tool

`next/dynamic` is Next.js's wrapper around `React.lazy` + `Suspense`. It handles the module boundary between Server and Client Components cleanly.

### Basic pattern

```tsx
// app/page.tsx  (Server Component)
import dynamic from 'next/dynamic'

// The Scene component is shipped in a separate JS chunk.
// It will not be included in the initial page bundle.
const Scene = dynamic(() => import('@/components/scene'), {
  ssr: false,
  loading: () => (
    <div
      style={{
        width: '100%',
        height: '500px',
        display: 'grid',
        placeItems: 'center',
        background: '#111',
        color: '#fff',
      }}
    >
      Loading 3D scene...
    </div>
  ),
})

export default function Page() {
  return (
    <div style={{ width: '100%', height: '500px' }}>
      <Scene />
    </div>
  )
}
```

The `loading` prop renders synchronously on the server and during the client's pre-hydration phase. It is replaced by `<Scene>` once the chunk downloads and the component mounts.

---

## Suspense Boundaries Inside the Canvas

R3F uses React `<Suspense>` for **asset loading** (models, textures, fonts, audio). The `useLoader` hook and Drei's asset hooks (`useGLTF`, `useTexture`, `useEnvironment`) all suspend while their assets are in flight.

### Basic Suspense with a fallback mesh

```tsx
'use client'

import { Suspense } from 'react'
import { Canvas } from '@react-three/fiber'
import { useGLTF, OrbitControls } from '@react-three/drei'

function Model({ url }: { url: string }) {
  // useGLTF suspends until the file is loaded
  const { scene } = useGLTF(url)
  return <primitive object={scene} />
}

function FallbackBox() {
  return (
    <mesh>
      <boxGeometry args={[1, 1, 1]} />
      <meshStandardMaterial color="#888" wireframe />
    </mesh>
  )
}

export default function Scene() {
  return (
    <Canvas camera={{ position: [0, 2, 5] }}>
      <ambientLight intensity={0.5} />
      <directionalLight position={[5, 5, 5]} />
      {/*
        Suspense inside Canvas catches suspensions from asset loaders.
        The fallback renders inside the WebGL scene, not in the DOM.
      */}
      <Suspense fallback={<FallbackBox />}>
        <Model url="/models/robot.glb" />
      </Suspense>
      <OrbitControls />
    </Canvas>
  )
}
```

> **Two types of Suspense to distinguish:**
> - **DOM-level Suspense** (wraps the `<Canvas>` itself) — shows a DOM fallback while the R3F bundle downloads
> - **Canvas-level Suspense** (inside `<Canvas>`) — shows a 3D fallback object while an asset loads inside the scene

### Nested Suspense boundaries

For complex scenes with multiple independently-loading assets, nest boundaries:

```tsx
'use client'

import { Suspense } from 'react'
import { Canvas } from '@react-three/fiber'
import { useGLTF, useEnvironment, Environment } from '@react-three/drei'

function Character({ url }: { url: string }) {
  const { scene } = useGLTF(url)
  return <primitive object={scene} />
}

function SceneEnvironment() {
  const envTexture = useEnvironment({ files: '/hdri/sunset.hdr' })
  return <Environment map={envTexture} background />
}

export default function ComplexScene() {
  return (
    <Canvas>
      {/* Environment and character load independently */}
      <Suspense fallback={null}>
        <SceneEnvironment />
      </Suspense>

      <Suspense fallback={<BoxPlaceholder />}>
        <Character url="/models/character.glb" />
      </Suspense>
    </Canvas>
  )
}
```

---

## Lazy Loading 3D Scenes on User Interaction

Do not load the Three.js bundle at all until the user triggers it. This keeps your initial bundle minimal.

```tsx
// components/lazy-scene-trigger.tsx
'use client'

import { useState, lazy, Suspense } from 'react'

// React.lazy does not support SSR. Use this inside a 'use client' file only.
const Scene = lazy(() => import('./heavy-scene'))

export default function LazySceneTrigger() {
  const [show, setShow] = useState(false)

  return (
    <div>
      {!show && (
        <button onClick={() => setShow(true)}>
          Launch 3D Experience
        </button>
      )}

      {show && (
        <div style={{ width: '100%', height: '600px' }}>
          <Suspense fallback={<div>Loading Three.js...</div>}>
            <Scene />
          </Suspense>
        </div>
      )}
    </div>
  )
}
```

### Preloading on hover (user intent signal)

Preload the chunk when the user hovers the trigger button — by the time they click, the download is likely complete:

```tsx
'use client'

import { useState } from 'react'
import dynamic from 'next/dynamic'

// next/dynamic returns a component with a .preload() method
const Scene = dynamic(() => import('./heavy-scene'), { ssr: false })

export default function PreloadOnHover() {
  const [show, setShow] = useState(false)

  return (
    <div>
      <button
        onMouseEnter={() => Scene.preload()}
        onClick={() => setShow(true)}
      >
        Open 3D View
      </button>
      {show && <Scene />}
    </div>
  )
}
```

---

## Code Splitting Individual Three.js Features

### Splitting postprocessing

`@react-three/postprocessing` is heavy. Defer it separately from the core scene:

```tsx
// components/scene.tsx
'use client'

import { Suspense, lazy } from 'react'
import { Canvas } from '@react-three/fiber'

// Postprocessing in its own chunk
const Effects = lazy(() => import('./effects'))

export default function Scene() {
  return (
    <Canvas>
      <ambientLight />
      <mesh>
        <torusKnotGeometry />
        <meshStandardMaterial color="hotpink" />
      </mesh>

      {/* Effects load after the scene is visible */}
      <Suspense fallback={null}>
        <Effects />
      </Suspense>
    </Canvas>
  )
}
```

```tsx
// components/effects.tsx
'use client'

import { EffectComposer, Bloom, ChromaticAberration } from '@react-three/postprocessing'

export default function Effects() {
  return (
    <EffectComposer>
      <Bloom luminanceThreshold={0} luminanceSmoothing={0.9} height={300} />
      <ChromaticAberration offset={[0.0005, 0.0005]} />
    </EffectComposer>
  )
}
```

### Splitting GLTF models into named exports

For scenes with multiple models, use separate chunks per model using named dynamic imports:

```tsx
// components/models/index.ts
import dynamic from 'next/dynamic'

export const RobotModel = dynamic(() =>
  import('./robot').then((m) => ({ default: m.Robot })),
  { ssr: false }
)

export const EnvironmentModel = dynamic(() =>
  import('./environment').then((m) => ({ default: m.Environment })),
  { ssr: false }
)
```

---

## `useGLTF.preload` — Preload GLTF Outside the Component Tree

Drei's `useGLTF.preload` starts the network request before the component that uses the model mounts. Call it at module scope in the file that will eventually render the model:

```tsx
// components/scene.tsx
'use client'

import { Canvas } from '@react-three/fiber'
import { useGLTF } from '@react-three/drei'
import { Suspense } from 'react'

// Fires the fetch immediately when this module loads (client-side only).
// The model is in the cache by the time <Model> mounts.
useGLTF.preload('/models/robot.glb')
useGLTF.preload('/models/environment.glb')

function Robot() {
  const { scene } = useGLTF('/models/robot.glb')
  return <primitive object={scene} />
}

export default function Scene() {
  return (
    <Canvas>
      <Suspense fallback={null}>
        <Robot />
      </Suspense>
    </Canvas>
  )
}
```

---

## Intersection Observer: Load 3D Only When Visible

Avoid loading Three.js for users who never scroll to the canvas:

```tsx
// components/viewport-scene.tsx
'use client'

import { useRef, useState, useEffect } from 'react'
import dynamic from 'next/dynamic'

const Scene = dynamic(() => import('./scene'), {
  ssr: false,
  loading: () => <div style={{ height: '500px', background: '#111' }} />,
})

export default function ViewportScene() {
  const containerRef = useRef<HTMLDivElement>(null)
  const [inViewport, setInViewport] = useState(false)

  useEffect(() => {
    const observer = new IntersectionObserver(
      ([entry]) => {
        if (entry.isIntersecting) {
          setInViewport(true)
          observer.disconnect() // load once, stay loaded
        }
      },
      { rootMargin: '200px' } // pre-load 200px before entering view
    )

    if (containerRef.current) {
      observer.observe(containerRef.current)
    }

    return () => observer.disconnect()
  }, [])

  return (
    <div ref={containerRef} style={{ width: '100%', height: '500px' }}>
      {inViewport ? <Scene /> : <div style={{ height: '500px', background: '#111' }} />}
    </div>
  )
}
```

---

## Bundle Analysis: Verifying Your Split

Install the bundle analyzer to confirm Three.js is not in your initial bundle:

```bash
npm install --save-dev @next/bundle-analyzer
```

```js
// next.config.js
const withBundleAnalyzer = require('@next/bundle-analyzer')({
  enabled: process.env.ANALYZE === 'true',
})

/** @type {import('next').NextConfig} */
const nextConfig = {
  transpilePackages: ['three', '@react-three/fiber', '@react-three/drei'],
}

module.exports = withBundleAnalyzer(nextConfig)
```

```bash
ANALYZE=true npm run build
```

In the treemap output, `three.module.js` and `@react-three/fiber` should appear in a separate chunk from your `app/page` chunk.

---

## Summary Decision Tree

```
Do you need the 3D scene on initial page load?
├── No  → next/dynamic + ssr:false + loading fallback
│         (Three.js not in initial bundle at all)
│
└── Yes → 'use client' on the component
          │
          Does the scene contain heavy async assets (GLTF, HDR, etc.)?
          ├── Yes → Wrap in Suspense inside <Canvas>
          │         useGLTF.preload() at module scope
          │
          └── No  → Render directly inside Canvas with no Suspense
```

---

## References

- [Next.js — Lazy Loading](https://nextjs.org/docs/app/guides/lazy-loading) — accessed 2026-05-26
- [React Three Fiber — Installation](https://r3f.docs.pmnd.rs/getting-started/installation) — accessed 2026-05-26
- [R3F Discussion — Loader inside Suspense fallback](https://github.com/pmndrs/react-three-fiber/discussions/2096) — accessed 2026-05-26
- [pmndrs/react-three-next starter](https://github.com/pmndrs/react-three-next) — accessed 2026-05-26
