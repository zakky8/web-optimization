# SSR / SSG Patterns with React Three Fiber in Next.js 15 App Router

> Sources verified 2026-05-26 against Next.js 16.2.6 docs and R3F docs.

---

## The Fundamental Rule

**Three.js and R3F cannot execute on the server.** They depend on browser APIs (`window`, `WebGLRenderingContext`, `HTMLCanvasElement`, `requestAnimationFrame`) that do not exist in Node.js. This is not a bug to work around — it is a structural fact to design around.

What this means in practice:

| Part of the app | Runs on server | Runs on client |
|---|---|---|
| Next.js App Router pages and layouts | Yes | Yes (hydration) |
| Server Components (default) | Yes only | No |
| Client Components (`'use client'`) | Yes (prerender) | Yes |
| R3F `<Canvas>` and scene contents | **No** | Yes |
| Three.js object instantiation | **No** | Yes |
| Drei hooks (`useGLTF`, `useThree`, etc.) | **No** | Yes |

The server cannot render pixels from a WebGL canvas. What SSR can contribute is:
- The HTML shell around the canvas
- Page metadata (`<title>`, Open Graph tags)
- Static content alongside the 3D scene (headings, text, UI controls)
- Loading placeholder markup

---

## What Can Be SSR'd

### The page layout

Server Components handle everything outside the canvas. This is the correct split:

```tsx
// app/product/[id]/page.tsx  (Server Component)
// This entire file runs on the server and produces HTML.
import { getProduct } from '@/lib/api'
import ProductScene from '@/components/product-scene'   // Client Component
import ProductDetails from '@/components/product-details' // Server Component

export async function generateMetadata({ params }: { params: { id: string } }) {
  const product = await getProduct(params.id)
  return {
    title: product.name,
    description: product.description,
    openGraph: {
      images: [product.previewImageUrl], // static image, not a screenshot of the canvas
    },
  }
}

export default async function ProductPage({ params }: { params: { id: string } }) {
  const product = await getProduct(params.id)

  return (
    <div className="product-layout">
      {/* Server-rendered text — great for SEO */}
      <ProductDetails product={product} />

      {/* Canvas — client-only, SSR produces only the container */}
      <div style={{ width: '100%', height: '500px' }}>
        <ProductScene modelUrl={product.modelUrl} />
      </div>
    </div>
  )
}
```

### Static generation (SSG) with `generateStaticParams`

Use `generateStaticParams` to pre-build product pages at build time. The canvas still loads client-side, but the surrounding HTML is fully static:

```tsx
// app/product/[id]/page.tsx
export async function generateStaticParams() {
  const products = await getProductList()
  return products.map((p) => ({ id: p.id }))
}

// The page function is unchanged — Next.js calls it at build time for each id.
```

The rendered HTML for each product page contains the product details and an empty container div. When the browser loads it, the Three.js bundle loads and fills in the canvas.

---

## `useIsomorphicLayoutEffect`

`useLayoutEffect` fires synchronously after DOM mutations, before the browser paints. On the server (Node.js), there is no DOM, so React logs a warning when it sees `useLayoutEffect` in a component that could server-render:

```
Warning: useLayoutEffect does nothing on the server because its effect cannot be
encoded into the server renderer's output format.
```

The standard fix is `useIsomorphicLayoutEffect`: it uses `useLayoutEffect` on the client and `useEffect` on the server.

```tsx
// lib/use-isomorphic-layout-effect.ts
'use client'

import { useLayoutEffect, useEffect } from 'react'

// useEffect on server (no-op, no warning), useLayoutEffect on client
export const useIsomorphicLayoutEffect =
  typeof window !== 'undefined' ? useLayoutEffect : useEffect
```

Usage inside an R3F component:

```tsx
'use client'

import { useRef } from 'react'
import { useFrame } from '@react-three/fiber'
import { useIsomorphicLayoutEffect } from '@/lib/use-isomorphic-layout-effect'
import * as THREE from 'three'

export function AnimatedBox() {
  const meshRef = useRef<THREE.Mesh>(null)

  // Safe to use in components that might SSR
  useIsomorphicLayoutEffect(() => {
    if (meshRef.current) {
      // Setup code that touches the DOM/WebGL context
      meshRef.current.position.set(0, 0.5, 0)
    }
  }, [])

  useFrame((_, delta) => {
    if (meshRef.current) {
      meshRef.current.rotation.y += delta
    }
  })

  return (
    <mesh ref={meshRef}>
      <boxGeometry args={[1, 1, 1]} />
      <meshStandardMaterial color="orange" />
    </mesh>
  )
}
```

> R3F v9 internally uses `useIsomorphicLayoutEffect` in several places, so you mainly need this when writing custom hooks that call `useLayoutEffect` directly.

---

## The Server/Client Component Boundary for 3D

### Rule: the boundary lives at the file that imports R3F or Three.js

Once a file has `'use client'`, everything it imports is part of the client module graph. This means the boundary can be narrow:

```
app/page.tsx              ← Server Component (can be async, can fetch)
  └── ProductDetails.tsx  ← Server Component (fetches and renders text)
  └── SceneLoader.tsx     ← 'use client' BOUNDARY ← imports Canvas, Three.js
        └── Scene.tsx     ← also client (pulled in by SceneLoader's graph)
        └── Model.tsx     ← also client
        └── Lights.tsx    ← also client
```

### Passing server-fetched data to the canvas

Server Components can pass serializable props (strings, numbers, plain objects, arrays) to Client Components. Use this to fetch model URLs, configuration, or product data on the server and pass it to the canvas:

```tsx
// app/page.tsx  (Server Component)
import { getModelConfig } from '@/lib/api'
import SceneLoader from '@/components/scene-loader'

export default async function Page({ params }: { params: { id: string } }) {
  // Fetch happens on the server — no API key exposed to the client
  const config = await getModelConfig(params.id)

  return (
    <SceneLoader
      modelUrl={config.modelUrl}
      backgroundColor={config.backgroundColor}
      cameraFov={config.cameraFov}
    />
  )
}
```

```tsx
// components/scene-loader.tsx
'use client'

import { Canvas } from '@react-three/fiber'
import { Suspense } from 'react'
import Model from './model'

interface Props {
  modelUrl: string
  backgroundColor: string
  cameraFov: number
}

export default function SceneLoader({ modelUrl, backgroundColor, cameraFov }: Props) {
  return (
    <Canvas
      camera={{ fov: cameraFov }}
      style={{ background: backgroundColor }}
    >
      <Suspense fallback={null}>
        <Model url={modelUrl} />
      </Suspense>
    </Canvas>
  )
}
```

> **Constraint**: You cannot pass React Server Components as children of R3F components. The children of `<Canvas>` are in the WebGL tree, not the DOM tree.

### Server Components alongside the canvas (children prop pattern)

Pass Server Component output as DOM-level children that overlay the canvas, not as three.js scene children:

```tsx
// app/page.tsx  (Server Component)
import CanvasWrapper from '@/components/canvas-wrapper'
import ProductInfo from '@/components/product-info'

export default async function Page() {
  return (
    <CanvasWrapper>
      {/* ProductInfo is a Server Component — renders to HTML */}
      <ProductInfo />
    </CanvasWrapper>
  )
}
```

```tsx
// components/canvas-wrapper.tsx
'use client'

import { Canvas } from '@react-three/fiber'

export default function CanvasWrapper({ children }: { children: React.ReactNode }) {
  return (
    <div style={{ position: 'relative', width: '100%', height: '100vh' }}>
      {/* WebGL canvas */}
      <Canvas style={{ position: 'absolute', inset: 0 }}>
        {/* scene objects here */}
      </Canvas>

      {/* DOM overlay — Server-rendered children go here */}
      <div style={{ position: 'absolute', inset: 0, pointerEvents: 'none' }}>
        {children}
      </div>
    </div>
  )
}
```

---

## `client-only` Package: Guarding Against Accidental Server Imports

Install the `client-only` package to make any accidental import of Three.js in a Server Component a **build-time error** rather than a runtime failure:

```bash
npm install client-only
```

Add the import at the top of any file containing Three.js or R3F code:

```tsx
// components/scene.tsx
import 'client-only'   // Build error if imported from a Server Component
'use client'

import { Canvas } from '@react-three/fiber'
import * as THREE from 'three'
// ...
```

This is particularly useful in shared utility files:

```tsx
// lib/three-utils.ts
import 'client-only'   // Prevents accidental server usage

import * as THREE from 'three'

export function createDefaultMaterial(color: string) {
  return new THREE.MeshStandardMaterial({ color })
}
```

---

## Static Metadata and Open Graph Images for 3D Pages

You cannot capture a WebGL canvas render server-side to use as an Open Graph image. Use these alternatives:

1. **Pre-rendered thumbnails**: Render your models to images in a build script and store the URLs. Reference them in `generateMetadata`.
2. **`next/og` with Canvas 2D**: Use Next.js Image Response for OG images. Canvas 2D (Skia-based) works server-side; WebGL does not.
3. **Placeholder images**: Use static images representing the 3D content.

```tsx
// app/product/[id]/page.tsx
import { ImageResponse } from 'next/og'

export async function generateImageMetadata() {
  return [
    {
      id: 'og',
      contentType: 'image/png',
      alt: 'Product preview',
      size: { width: 1200, height: 630 },
    },
  ]
}

// This function is server-executed. It can use Canvas 2D but NOT WebGL.
export default async function og() {
  return new ImageResponse(
    <div style={{ /* 2D layout */ }}>
      {/* static image or 2D canvas — no Three.js here */}
    </div>
  )
}
```

---

## SSG Pre-render Timing: When Does the Canvas First Paint?

For statically generated pages, the timeline is:

1. **Build time**: Server renders HTML. Canvas container `<div>` is in the HTML. Canvas itself is not.
2. **Browser load**: HTML and critical CSS stream to browser. User sees page shell.
3. **JS bundle download**: Three.js chunk downloads (deferred if you used `next/dynamic`).
4. **React hydration**: React attaches to the SSR'd HTML.
5. **Canvas mount**: R3F creates the WebGL context, starts rendering.
6. **Asset load**: GLTF models, textures stream in. Suspense fallbacks shown until resolved.

There is no way to eliminate steps 3–6. The goal is to minimize their impact: make steps 1–2 fast (they are, because Three.js is excluded), and handle the gap with appropriate loading UI.

---

## Streaming with React Suspense at the Page Level

Use `loading.tsx` or `<Suspense>` around server components that fetch data for the page, so text content streams in fast even if the 3D section takes longer:

```tsx
// app/product/[id]/page.tsx
import { Suspense } from 'react'
import dynamic from 'next/dynamic'
import { ProductDetailsFromServer } from '@/components/product-details-server'

const Scene = dynamic(() => import('@/components/scene'), { ssr: false })

export default function Page({ params }: { params: { id: string } }) {
  return (
    <div>
      {/* Text content streams immediately as server data resolves */}
      <Suspense fallback={<div>Loading product details...</div>}>
        <ProductDetailsFromServer id={params.id} />
      </Suspense>

      {/* Canvas loads independently on the client */}
      <Scene />
    </div>
  )
}
```

---

## Summary: What Lives Where

| Code | Server Component | Client Component |
|---|---|---|
| `generateMetadata()` | Yes | No |
| `generateStaticParams()` | Yes | No |
| Database / API fetches | Yes (preferred) | Yes (via SWR/React Query) |
| `<Canvas>` | **No** | Yes |
| `useThree()`, `useFrame()` | **No** | Yes |
| `new THREE.Mesh()` | **No** | Yes |
| `useGLTF()`, `useTexture()` | **No** | Yes |
| `useLayoutEffect` (as-is) | Warning | Yes |
| `useIsomorphicLayoutEffect` | Safe (becomes `useEffect`) | Yes |
| `'client-only'` guard | Build error | N/A |

---

## References

- [Next.js — Server and Client Components](https://nextjs.org/docs/app/getting-started/server-and-client-components) — accessed 2026-05-26
- [Next.js — Lazy Loading](https://nextjs.org/docs/app/guides/lazy-loading) — accessed 2026-05-26
- [React Three Fiber — Installation](https://r3f.docs.pmnd.rs/getting-started/installation) — accessed 2026-05-26
- [usehooks-ts — useIsomorphicLayoutEffect](https://usehooks-ts.com/react-hook/use-isomorphic-layout-effect) — accessed 2026-05-26
- [R3F Discussion — Architecture for routes with canvases](https://github.com/pmndrs/react-three-fiber/discussions/3221) — accessed 2026-05-26
