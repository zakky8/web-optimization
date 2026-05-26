# React Server Components and Three.js

**Sources verified:** 2026-05-26
**Primary sources:** nextjs.org/docs/app/getting-started/server-and-client-components (last updated 2026-05-19), react.dev/reference/react/use, react.dev/blog/2024/12/05/react-19

---

## The Fundamental Split

Three.js uses the WebGL API. WebGL is accessed through `HTMLCanvasElement`, which
requires `document` and `window` — browser-only globals that do not exist in a
Node.js server rendering environment.

**Three.js and everything that wraps it (R3F, drei) must run in Client
Components.**

React Server Components (RSC) run exclusively in Node.js at build time or on the
server per-request. They have no DOM, no browser APIs, and no WebGL context.

Source: nextjs.org/docs/app/getting-started/server-and-client-components
(When to use Server and Client Components table), accessed 2026-05-26.

---

## What Runs Where

| Concern | Environment | Reason |
|---|---|---|
| `new THREE.WebGLRenderer()` | Client only | Requires WebGL context |
| `<Canvas>` from R3F | Client only | Mounts to a `<canvas>` element |
| `useGLTF`, `useTexture`, `useLoader` | Client only | Use hooks + browser fetch |
| `useFrame`, `useThree` | Client only | Run inside React renderer |
| Model metadata fetch (URLs, names, positions) | Server preferred | No browser API needed |
| Static scene configuration (light params, camera FOV) | Server preferred | Plain data, no side effects |
| Database queries for 3D asset manifests | Server only | Credentials must stay server-side |
| User interaction state (selected object, camera target) | Client only | Requires useState/event handlers |
| `use()` reading a promise | Either — but see caveats | Server prefers async/await |

---

## Marking the Client Boundary

Add `'use client'` to any file that imports Three.js, R3F, or drei. Once a
file is marked, **all of its imports** are included in the client bundle. You
only need the directive at the entry point of the client subtree.

```tsx
// components/ThreeScene.tsx
'use client'; // This boundary pulls in three, @react-three/fiber, @react-three/drei

import { Canvas } from '@react-three/fiber';
import { useGLTF, OrbitControls } from '@react-three/drei';
import { Suspense } from 'react';

export default function ThreeScene({ modelUrl }: { modelUrl: string }) {
  return (
    <Canvas>
      <Suspense fallback={null}>
        <Model url={modelUrl} />
      </Suspense>
      <OrbitControls />
    </Canvas>
  );
}

function Model({ url }: { url: string }) {
  const { scene } = useGLTF(url);
  return <primitive object={scene} />;
}
```

Source: nextjs.org/docs/app/getting-started/server-and-client-components
(Using Client Components section), accessed 2026-05-26.

---

## Pattern: Fetch Model Metadata on the Server, Render on the Client

The Server Component fetches from a database or API (no API key exposure, runs
close to the data source). The Client Component receives serialized props and
handles all Three.js rendering.

```tsx
// app/scene/page.tsx — Server Component (default in Next.js App Router)
import ThreeScene from '@/components/ThreeScene';
import { getModelConfig } from '@/lib/models'; // Server-only DB call

export default async function ScenePage({
  params,
}: {
  params: Promise<{ id: string }>;
}) {
  const { id } = await params;

  // Runs on server — can use DB credentials, no exposure to client bundle
  const config = await getModelConfig(id);
  // config: { modelUrl: string, position: [x,y,z], lightPreset: string }

  return (
    <main>
      <h1>{config.name}</h1>
      {/* ThreeScene is a Client Component — receives serialized props */}
      <ThreeScene
        modelUrl={config.modelUrl}
        position={config.position}
        lightPreset={config.lightPreset}
      />
    </main>
  );
}
```

Props passed across the server/client boundary must be serializable: strings,
numbers, arrays, plain objects. No functions, no class instances, no Promises
(unless using the `use()` streaming pattern below).

Source: nextjs.org/docs/app/getting-started/server-and-client-components
(Passing data section, "Props... need to be serializable"), accessed 2026-05-26.

---

## Streaming Pattern: Pass a Promise to the Client

React 19 enables an optimized variant: the Server Component initiates the fetch
without awaiting, passes the raw promise to the Client Component, and the client
reads it with `use()`. Data streaming begins before the server renders
the parent component tree.

```tsx
// app/gallery/page.tsx — Server Component
import SceneStream from '@/components/SceneStream';

async function fetchAssetManifest(id: string) {
  const res = await fetch(`https://api.example.com/assets/${id}`, {
    cache: 'force-cache', // RSC fetch caching
  });
  if (!res.ok) throw new Error('Manifest fetch failed');
  return res.json();
}

export default function GalleryPage({
  params,
}: {
  params: { id: string };
}) {
  // NOT awaited — promise passed directly to client
  const manifestPromise = fetchAssetManifest(params.id);

  return (
    <Suspense fallback={<p>Loading gallery...</p>}>
      <SceneStream manifestPromise={manifestPromise} />
    </Suspense>
  );
}
```

```tsx
// components/SceneStream.tsx — Client Component
'use client';

import { use, Suspense } from 'react';
import { Canvas } from '@react-three/fiber';
import { useGLTF } from '@react-three/drei';

interface AssetManifest {
  modelUrl: string;
  environmentUrl: string;
  cameraPosition: [number, number, number];
}

function Scene({ manifest }: { manifest: AssetManifest }) {
  const { scene: model } = useGLTF(manifest.modelUrl);
  return <primitive object={model} />;
}

export default function SceneStream({
  manifestPromise,
}: {
  manifestPromise: Promise<AssetManifest>;
}) {
  // use() suspends until the server-initiated promise resolves
  const manifest = use(manifestPromise);

  return (
    <Canvas camera={{ position: manifest.cameraPosition }}>
      <Suspense fallback={null}>
        <Scene manifest={manifest} />
      </Suspense>
    </Canvas>
  );
}
```

The promise is stable because it originates in the Server Component, not inside
a Client Component render function. React docs: "prefer creating Promises in
Server Components and passing to Client Components."

Source: react.dev/reference/react/use (Promise stability section), accessed 2026-05-26.

---

## Static Scene Configuration on the Server

Scene configuration that does not change — default camera FOV, ambient light
intensity, fog density — can live in the Server Component and be passed down
as props. This has two benefits: it keeps configuration in a single authoritative
location (server/DB), and it does not add to the client JS bundle.

```tsx
// Server Component
const SCENE_CONFIG = {
  fog: { near: 10, far: 100, color: '#1a1a2e' },
  ambientLight: { intensity: 0.4 },
  camera: { fov: 60, near: 0.1, far: 200 },
};

export default function Page() {
  return <ThreeScene config={SCENE_CONFIG} />;
}
```

```tsx
// Client Component
'use client';

export default function ThreeScene({ config }: { config: typeof SCENE_CONFIG }) {
  return (
    <Canvas
      camera={{
        fov: config.camera.fov,
        near: config.camera.near,
        far: config.camera.far,
      }}
    >
      <fog attach="fog" color={config.fog.color} near={config.fog.near} far={config.fog.far} />
      <ambientLight intensity={config.ambientLight.intensity} />
      <Suspense fallback={null}>
        <SceneContent />
      </Suspense>
    </Canvas>
  );
}
```

---

## Preventing Environment Poisoning

If a utility module imports Three.js, importing it in a Server Component will
cause a build error (or silently fail at runtime in some configurations). Use
the `client-only` package to enforce the boundary:

```typescript
// lib/three-utils.ts
import 'client-only'; // Build-time error if imported in a Server Component

import * as THREE from 'three';

export function createStandardMaterial(color: string) {
  return new THREE.MeshStandardMaterial({ color });
}
```

The complementary `server-only` package protects modules that contain secrets
or server-only logic from being accidentally bundled into client code.

Source: nextjs.org/docs/app/getting-started/server-and-client-components
(Preventing environment poisoning section), accessed 2026-05-26.

---

## Context Providers Must Be Client Components

React context is not supported in Server Components. If you use a Zustand
provider, a theme context, or an R3F context wrapper, it must be a Client
Component. The recommended pattern: a thin `'use client'` wrapper that provides
context to Server Component children via the `children` prop.

```tsx
// providers/SceneProvider.tsx
'use client';

import { createContext, useContext, useState } from 'react';

const SceneContext = createContext<{ selectedId: string | null }>({ selectedId: null });

export function SceneProvider({ children }: { children: React.ReactNode }) {
  const [selectedId, setSelectedId] = useState<string | null>(null);
  return (
    <SceneContext.Provider value={{ selectedId }}>
      {children}
    </SceneContext.Provider>
  );
}
```

```tsx
// app/layout.tsx — Server Component wrapping the client provider
import { SceneProvider } from '@/providers/SceneProvider';

export default function RootLayout({ children }: { children: React.ReactNode }) {
  return (
    <html>
      <body>
        <SceneProvider>{children}</SceneProvider>
      </body>
    </html>
  );
}
```

Source: nextjs.org/docs/app/getting-started/server-and-client-components
(Context providers section), accessed 2026-05-26.

---

## Summary Decision Tree

```
Does this code touch Three.js, WebGL, canvas, or browser APIs?
  YES → 'use client' required. No exceptions.

Does this code fetch data from a DB or API?
  YES and no browser APIs needed → Server Component preferred.

Does this code need useState, useEffect, or event handlers?
  YES → Client Component required.

Is this static configuration passed to the scene?
  YES → Define in Server Component, pass as serialized props.

Does a Client Component need dynamic data that loads asynchronously?
  Use use() + promise from Server Component for streaming pattern.
  Use useGLTF/useLoader for 3D asset files specifically.
```
