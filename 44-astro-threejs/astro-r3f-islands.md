# R3F as an Astro Island

> Verified against: https://docs.astro.build/  
> Astro version: 6.x (current as of 2026-05-26)  
> Sources accessed: 2026-05-26

---

## What "Island" Means in Astro

Astro's Islands Architecture renders the page shell in HTML with zero JavaScript by default. Each interactive component is an isolated "island" that hydrates independently. R3F (React Three Fiber) is a React renderer, so it ships as a React island inside an otherwise static Astro page.

---

## 1. Prerequisites

```bash
# React integration
npx astro add react

# Three.js + R3F
npm install three @react-three/fiber @react-three/drei
npm install --save-dev @types/three
```

`npx astro add react` automatically edits `astro.config.mjs` and `tsconfig.json`.

---

## 2. `client:only="react"` — Skip SSR Entirely

R3F uses `canvas`, `WebGLRenderer`, and the browser's `ResizeObserver`. None of these exist in Node.js. Use `client:only="react"` to prevent Astro from attempting a server render.

```tsx
// src/components/Scene.tsx
import { Canvas } from '@react-three/fiber';
import { OrbitControls, Box } from '@react-three/drei';

export default function Scene() {
  return (
    <Canvas camera={{ position: [0, 0, 5], fov: 75 }}>
      <ambientLight intensity={0.5} />
      <directionalLight position={[10, 10, 5]} />
      <Box args={[1, 1, 1]}>
        <meshStandardMaterial color="#6366f1" />
      </Box>
      <OrbitControls />
    </Canvas>
  );
}
```

```astro
---
// src/pages/index.astro
import Scene from '../components/Scene';
---
<html>
  <head><title>3D Scene</title></head>
  <body style="margin:0;">
    <!-- client:only="react" — no SSR, renders only in browser -->
    <Scene client:only="react" />
  </body>
</html>
```

`client:only` behavior (from Astro reference docs, accessed 2026-05-26):
- Skips server-side HTML rendering entirely
- Renders and hydrates exclusively on the client at page load
- Requires specifying the framework: `"react"`, `"preact"`, `"svelte"`, `"vue"`, or `"solid-js"`
- Supports a `slot="fallback"` for content shown while the component loads

```astro
<Scene client:only="react">
  <p slot="fallback">Loading 3D scene…</p>
</Scene>
```

---

## 3. `client:visible` — Lazy Load 3D When in Viewport

For 3D sections below the fold, defer loading until the user scrolls to them. This avoids downloading Three.js (~600 KB min+gzip) for users who never reach the section.

```astro
---
import HeroScene from '../components/HeroScene';
import ProductViewer from '../components/ProductViewer';
---
<html>
  <body>
    <!-- Hero: load immediately because it is visible on arrival -->
    <HeroScene client:only="react" />

    <!-- Long marketing section — pure HTML, no JS -->
    <section><!-- ... --></section>

    <!-- Product viewer: defer until user scrolls down -->
    <div style="height:600px;">
      <ProductViewer client:visible />
    </div>
  </body>
</html>
```

`client:visible` behavior (from Astro reference docs, accessed 2026-05-26):
- Uses `IntersectionObserver` to detect viewport entry
- Hydration begins the moment the component enters view
- Intended for "resource-intensive elements you would prefer not to load if the user never saw the element"
- Accepts an optional `rootMargin` parameter (added in Astro 4.1.0) to trigger hydration slightly before the element enters view, reducing pop-in:

```astro
<!-- Hydrate 200px before the element enters view -->
<ProductViewer client:visible={{ rootMargin: "200px" }} />
```

**`client:visible` vs `client:only` for R3F**

`client:visible` still triggers SSR for the component's markup (returning an empty placeholder, since R3F produces nothing server-side). For components that throw during SSR, `client:only="react"` with `client:visible`-style lazy loading is achieved by lazy-importing the component:

```tsx
// src/components/LazyScene.tsx
import { lazy, Suspense } from 'react';

const Scene = lazy(() => import('./Scene'));

export default function LazyScene() {
  return (
    <Suspense fallback={<div>Loading…</div>}>
      <Scene />
    </Suspense>
  );
}
```

```astro
<LazyScene client:only="react" />
```

---

## 4. Combining SSR Content with Client-Side 3D

Astro's Islands model means the HTML structure is server-rendered but the canvas is client-only. This is the correct pattern for combining SEO content with 3D:

```astro
---
// src/pages/product/[slug].astro
import { getCollection } from 'astro:content';
import ProductViewer from '../../components/ProductViewer';

export async function getStaticPaths() {
  const products = await getCollection('products');
  return products.map(p => ({ params: { slug: p.id }, props: { product: p } }));
}

const { product } = Astro.props;
---
<html>
  <head>
    <title>{product.data.name}</title>
    <!-- This meta tag is rendered server-side — crawlers can read it -->
    <meta name="description" content={product.data.description} />
  </head>
  <body>
    <!-- SSR: fully crawlable, loads instantly -->
    <h1>{product.data.name}</h1>
    <p>{product.data.description}</p>

    <!-- Island: Three.js canvas, client-only -->
    <div style="height:500px;">
      <ProductViewer
        client:visible
        modelPath={product.data.modelPath}
      />
    </div>

    <!-- SSR: more content below -->
    <section>
      <h2>Specifications</h2>
      <!-- ... -->
    </section>
  </body>
</html>
```

Props passed from `.astro` to an island component are serialized at build time and injected as JSON. They must be serializable (no functions, no class instances). File paths and primitive values work fine.

---

## 5. Hydration Strategy Decision Tree

| Scenario | Directive |
|---|---|
| Canvas is above the fold, visible on arrival | `client:only="react"` |
| Canvas is below the fold | `client:only="react"` + lazy React import, OR `client:visible` on a safe SSR wrapper |
| Canvas needs to be interactive immediately | `client:load` (hydrates on page load; fine for React Three Fiber) |
| Canvas is in a sidebar only shown on desktop | `client:media="(min-width: 768px)"` |
| Canvas is low-priority background element | `client:idle` (hydrates during browser idle time) |

`client:load` behavior (from Astro reference docs, accessed 2026-05-26): "Immediately loads and hydrates component JavaScript on page load. Best for immediately-visible UI elements that need to be interactive as soon as possible."

`client:idle` behavior (from Astro reference docs, accessed 2026-05-26): "Hydrates after initial page load completes and the `requestIdleCallback` event fires." Accepts an optional `timeout` parameter (added in Astro 4.15.0) specifying the maximum milliseconds to wait.

---

## 6. Passing Data into the R3F Island

```astro
---
import Viewer from '../components/Viewer';

// These values are computed server-side and serialized into the island
const modelUrl = '/assets/product.glb';
const envMapUrl = '/assets/studio.hdr';
const initialRotation: [number, number, number] = [0, Math.PI / 4, 0];
---

<Viewer
  client:only="react"
  modelUrl={modelUrl}
  envMapUrl={envMapUrl}
  initialRotation={initialRotation}
/>
```

```tsx
// src/components/Viewer.tsx
interface ViewerProps {
  modelUrl: string;
  envMapUrl: string;
  initialRotation: [number, number, number];
}

export default function Viewer({ modelUrl, envMapUrl, initialRotation }: ViewerProps) {
  // ...
}
```

---

## 7. Multiple R3F Islands on One Page

Each `<Canvas>` creates a separate WebGL context. Most devices support 8–16 simultaneous contexts; browsers begin dropping oldest contexts past that limit. For pages with many 3D elements, consider:

1. One shared `<Canvas>` that renders multiple scenes using portals
2. Canvas pooling (advanced — share renderer across islands)
3. Limiting `client:visible` islands so only 2–3 are active simultaneously

```tsx
// Multiple scenes in one canvas using @react-three/fiber portals
import { createPortal, useThree } from '@react-three/fiber';
import { useMemo } from 'react';
import * as THREE from 'three';

function InsetScene({ children }: { children: React.ReactNode }) {
  const scene = useMemo(() => new THREE.Scene(), []);
  return createPortal(children, scene);
}
```

---

## Key Points

- `client:only="react"` is the standard directive for all R3F components — it prevents SSR that would fail on WebGL APIs.
- `client:visible` is ideal for below-the-fold 3D sections; combine with a React `lazy()` import to defer the JS bundle itself.
- Props passed to islands must be JSON-serializable. Pass URLs and primitives; compute anything complex server-side.
- Each `<Canvas>` is a separate WebGL context. Keep the total count low per page.
- Astro 6 removed `<ViewTransitions />` — use `<ClientRouter />` from `astro:transitions` if you need SPA-style navigation between pages with 3D.
