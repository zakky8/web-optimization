# Next.js 15 App Router + React Three Fiber Setup

> Sources verified 2026-05-26 against Next.js 16.2.6 docs and R3F installation docs.

---

## Version Compatibility

Before anything else, match your package versions. R3F split on the React 18/19 boundary:

| `@react-three/fiber` | React | Next.js |
|---|---|---|
| v8.x | 18.x | 14.x and earlier |
| v9.x | 19.x | 15.x + |

Next.js 15 ships React 19 as a peer dependency. If you are on Next.js 15, install R3F v9:

```bash
npm install three @react-three/fiber
# @react-three/fiber@9 resolves automatically with react@19
```

For drei helpers:

```bash
npm install @react-three/drei
```

> **Known issue**: `@react-three/fiber@8` with Next.js 15 / React 19 throws `TypeError: Cannot read properties of undefined (reading 'ReactCurrentOwner')`. The fix is upgrading to v9, not adding `'use client'` or adjusting imports.

---

## Why the App Router Breaks Naive Three.js Usage

In the App Router, every file is a **Server Component by default**. Server Components render on Node.js — there is no `window`, no `WebGLRenderingContext`, no DOM. Three.js and R3F import browser globals at module evaluation time. Importing them in a server-executed module causes a build error or a silent runtime failure.

The two failure modes you will actually hit:

1. **Top-level import in a Server Component** — `import * as THREE from 'three'` evaluated on the server throws or produces an empty bundle because Three.js references `window` during module init.
2. **Canvas rendered server-side** — R3F's `<Canvas>` creates a WebGL context. On the server this produces a hydration mismatch: the server sends no canvas markup, the client renders one, React throws.

Both are solved with the `'use client'` directive, possibly combined with `next/dynamic` + `{ ssr: false }`.

---

## Pattern 1 — `'use client'` alone (simplest)

Mark any file that touches Three.js or R3F as a Client Component. The directive at the top of the file pushes the entire module graph of that file to the client bundle. You do **not** need to repeat the directive in every imported sub-component.

```tsx
// components/scene.tsx
'use client'

import { Canvas } from '@react-three/fiber'
import { OrbitControls } from '@react-three/drei'

export default function Scene() {
  return (
    <Canvas camera={{ position: [0, 0, 5], fov: 60 }}>
      <ambientLight intensity={0.5} />
      <directionalLight position={[10, 10, 5]} intensity={1} />
      <mesh>
        <boxGeometry args={[1, 1, 1]} />
        <meshStandardMaterial color="royalblue" />
      </mesh>
      <OrbitControls />
    </Canvas>
  )
}
```

Use it from a Server Component page without any special wrapper:

```tsx
// app/page.tsx  (Server Component — no 'use client')
import Scene from '@/components/scene'

export default function HomePage() {
  return (
    <main>
      <h1>My 3D App</h1>
      <div style={{ width: '100%', height: '500px' }}>
        <Scene />
      </div>
    </main>
  )
}
```

Next.js sees `Scene` is a Client Component and ships its bundle to the browser. The server-rendered HTML contains the `<div>` wrapper but no canvas markup, which is correct.

---

## Pattern 2 — `'use client'` + `next/dynamic` + `{ ssr: false }`

Use this when you want to **completely skip prerendering** the component on the server. This suppresses the server-side HTML for the component entirely, so React never attempts reconciliation for it during hydration.

```tsx
// app/page.tsx  (Server Component)
import dynamic from 'next/dynamic'

const Scene = dynamic(() => import('@/components/scene'), {
  ssr: false,
  loading: () => <div style={{ width: '100%', height: '500px' }}>Loading 3D scene...</div>,
})

export default function HomePage() {
  return (
    <main>
      <h1>My 3D App</h1>
      <div style={{ width: '100%', height: '500px' }}>
        <Scene />
      </div>
    </main>
  )
}
```

`scene.tsx` still needs `'use client'` at the top:

```tsx
// components/scene.tsx
'use client'

import { Canvas } from '@react-three/fiber'
// ... rest of the component
```

> **Important**: The `ssr: false` option only works inside a Client Component or a Server Component that dynamically imports the component. You **cannot** use `ssr: false` directly inside a Server Component file — Next.js will throw: `ssr: false is not allowed with next/dynamic in Server Components. Please move it into a Client Component.`

The correct pattern for `ssr: false` from a Server Component is to create an intermediate client wrapper:

```tsx
// components/scene-loader.tsx  <-- this is the Client Component
'use client'

import dynamic from 'next/dynamic'

const Scene = dynamic(() => import('./scene'), { ssr: false })

export default function SceneLoader() {
  return <Scene />
}
```

```tsx
// app/page.tsx  (Server Component — imports the wrapper)
import SceneLoader from '@/components/scene-loader'

export default function HomePage() {
  return (
    <main>
      <SceneLoader />
    </main>
  )
}
```

---

## Canvas Placement in App Router

### Canvas needs an explicit size

`<Canvas>` fills its container via `position: absolute`. Always give the parent an explicit `width` and `height`:

```tsx
'use client'

import { Canvas } from '@react-three/fiber'

export default function Scene() {
  return (
    // height must be explicit — '100%' only works if the parent also has a height
    <div style={{ width: '100%', height: '100vh', position: 'relative' }}>
      <Canvas>
        {/* scene contents */}
      </Canvas>
    </div>
  )
}
```

### Avoid placing Canvas inside the root layout

Placing `<Canvas>` in `app/layout.tsx` (which is a Server Component by default) causes issues. Wrap it instead:

```tsx
// app/layout.tsx  (Server Component)
import CanvasProvider from '@/components/canvas-provider'

export default function RootLayout({ children }: { children: React.ReactNode }) {
  return (
    <html lang="en">
      <body>
        {children}
      </body>
    </html>
  )
}
```

### Full-page 3D layout example

```tsx
// app/(3d)/layout.tsx
import dynamic from 'next/dynamic'

const SceneCanvas = dynamic(() => import('@/components/scene-canvas'), {
  ssr: false,
})

export default function ThreeDLayout({ children }: { children: React.ReactNode }) {
  return (
    <div style={{ position: 'relative', width: '100vw', height: '100vh' }}>
      <SceneCanvas />
      {/* DOM overlay rendered on top of the canvas */}
      <div style={{ position: 'absolute', inset: 0, pointerEvents: 'none' }}>
        {children}
      </div>
    </div>
  )
}
```

---

## Avoiding Hydration Errors — Checklist

| Cause | Fix |
|---|---|
| `<Canvas>` rendered on server | Add `{ ssr: false }` or wrap in a client component that delays mount |
| `window` / `document` accessed at module level | Move access inside `useEffect` or `useLayoutEffect` |
| `useLayoutEffect` in a component that SSRs | Use `useIsomorphicLayoutEffect` (see `ssr-ssg-patterns.md`) |
| Three.js import at top of a server file | Move the import into the `'use client'` file; never import `three` in a Server Component |
| `Math.random()` or `Date.now()` at render time | Seed in `useEffect`, not during render |

---

## Common Mistake: Importing Three.js in a Server Component

This is the most common error new developers make:

```tsx
// app/page.tsx — WRONG
// This is a Server Component. This import will either fail at build time
// or produce broken output because Three.js references browser globals.
import * as THREE from 'three'

export default function Page() {
  const geometry = new THREE.BoxGeometry()  // ReferenceError on server
  // ...
}
```

The fix: never instantiate Three.js objects in Server Components. Move all Three.js code into `'use client'` files:

```tsx
// app/page.tsx — CORRECT (Server Component)
import SceneLoader from '@/components/scene-loader'

export default function Page() {
  return <SceneLoader />
}

// components/scene-loader.tsx — CORRECT (Client Component)
'use client'
import * as THREE from 'three'
import { Canvas } from '@react-three/fiber'

export default function SceneLoader() {
  return (
    <Canvas>
      <mesh geometry={new THREE.BoxGeometry()}>
        <meshStandardMaterial />
      </mesh>
    </Canvas>
  )
}
```

---

## TypeScript: Extending JSX for R3F Elements

R3F uses declarative JSX for Three.js objects. TypeScript needs the type extension from R3F:

```tsx
// types/three-fiber.d.ts
import '@react-three/fiber'

// If you use custom elements, extend the ThreeElements interface here
// This is usually handled automatically by @react-three/fiber types
```

R3F v9 ships types that cover all core Three.js classes as JSX elements. No manual augmentation needed for standard objects.

---

## Installation Summary

```bash
# Next.js 15 + React 19 + R3F v9
npm install three @react-three/fiber @react-three/drei

# TypeScript types for three.js
npm install --save-dev @types/three
```

`next.config.js` minimum for R3F (see `next-config-3d.md` for full config):

```js
// next.config.js
/** @type {import('next').NextConfig} */
const nextConfig = {
  transpilePackages: ['three', '@react-three/fiber', '@react-three/drei'],
}

module.exports = nextConfig
```

---

## References

- [React Three Fiber — Installation](https://r3f.docs.pmnd.rs/getting-started/installation) — accessed 2026-05-26
- [Next.js — Server and Client Components](https://nextjs.org/docs/app/getting-started/server-and-client-components) — accessed 2026-05-26
- [Next.js — Lazy Loading](https://nextjs.org/docs/app/guides/lazy-loading) — accessed 2026-05-26
- [GitHub — R3F ReactCurrentOwner error with Next.js 15](https://github.com/pmndrs/react-three-fiber/issues/3417) — accessed 2026-05-26
- [GitHub — Next.js + R3F compatibility issue #71836](https://github.com/vercel/next.js/issues/71836) — accessed 2026-05-26
