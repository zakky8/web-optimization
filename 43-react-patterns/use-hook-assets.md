# React 19 `use()` Hook for Loading 3D Assets

**Sources verified:** 2026-05-26
**Primary sources:** react.dev/reference/react/use, react.dev/blog/2024/12/05/react-19, drei.docs.pmnd.rs/loaders/gltf-use-gltf

---

## What `use()` Is

`use()` is a new React 19 API that reads a resource (Promise or Context) during
render. Unlike hooks, it can be called inside conditionals and loops.

Function signature:

```typescript
const value = use(resource);
// resource: Promise<T> | Context<T>
// returns: T (resolved value, or context value)
```

When passed a pending Promise, `use()` suspends the component — identical
behavior to throwing a promise in the Suspense protocol, but with an official
API surface. The nearest `<Suspense>` boundary shows its fallback. When the
promise resolves, React re-renders the component with the resolved value.

Source: react.dev/reference/react/use, accessed 2026-05-26.

---

## Critical Rule: Promise Stability

The promise passed to `use()` must be **stable across re-renders**. Creating a
promise inside a Client Component function body means it is recreated on every
render, causing `use()` to suspend indefinitely (new promise = perpetually
pending from React's perspective).

React docs state explicitly: "Avoid creating Promises in Client Components" and
"Prefer creating Promises in Server Components."

Source: react.dev/reference/react/use, Caveats section, accessed 2026-05-26.

Solutions for client-only contexts:

1. Create the promise at module scope (outside any component).
2. Use `useMemo` to stabilize a promise derived from props.
3. Use a caching layer (React Query, SWR, Zustand) that returns the same
   promise reference for the same key.

```tsx
// BAD — new promise every render, infinite suspension
function ModelViewer() {
  const gltfPromise = loadGLTF('/model.glb'); // recreated each render
  const gltf = use(gltfPromise);
  // ...
}

// GOOD — stable promise, created once
const stableGltfPromise = loadGLTF('/model.glb'); // module scope

function ModelViewer() {
  const gltf = use(stableGltfPromise);
  // ...
}

// ALSO GOOD — stable via useMemo when URL comes from props
function ModelViewer({ url }: { url: string }) {
  const gltfPromise = useMemo(() => loadGLTF(url), [url]);
  const gltf = use(gltfPromise);
  // ...
}
```

---

## Wrapping GLTFLoader in a Promise for `use()`

To use `use()` with raw Three.js `GLTFLoader`, wrap the loader callback in a
promise and cache it by URL to ensure stability:

```tsx
import { GLTFLoader, GLTF } from 'three/examples/jsm/loaders/GLTFLoader.js';
import { DRACOLoader } from 'three/examples/jsm/loaders/DRACOLoader.js';

// Singleton loader with Draco support
const dracoLoader = new DRACOLoader();
dracoLoader.setDecoderPath('https://www.gstatic.com/draco/v1/decoders/');

const gltfLoader = new GLTFLoader();
gltfLoader.setDRACOLoader(dracoLoader);

// Module-level cache ensures promise stability across re-renders
const cache = new Map<string, Promise<GLTF>>();

function loadGLTF(url: string): Promise<GLTF> {
  if (!cache.has(url)) {
    const promise = new Promise<GLTF>((resolve, reject) => {
      gltfLoader.load(url, resolve, undefined, reject);
    });
    cache.set(url, promise);
  }
  return cache.get(url)!;
}
```

Usage with `use()`:

```tsx
import { use, Suspense } from 'react';
import * as THREE from 'three';

// Promise created at module scope — stable
const robotPromise = loadGLTF('/models/robot.glb');

function Robot() {
  // Suspends until resolved; throws to ErrorBoundary if rejected
  const gltf = use(robotPromise);
  return <primitive object={gltf.scene} />;
}

export default function Scene() {
  return (
    <Canvas>
      <Suspense fallback={<LoadingSpinner />}>
        <Robot />
      </Suspense>
    </Canvas>
  );
}
```

---

## `use()` Cannot Be Called in try-catch

React docs state: "`use()` cannot be called in a try-catch block."

Source: react.dev/reference/react/use, Caveats, accessed 2026-05-26.

Handle promise rejections with either:

1. `ErrorBoundary` (declarative, catches render-time throws)
2. `.catch()` on the promise before passing it to `use()`:

```tsx
// Option 2: resolve errors to a fallback value instead of throwing
const safeRobotPromise = loadGLTF('/models/robot.glb').catch(() => null);

function Robot() {
  const gltf = use(safeRobotPromise); // null on failure, no ErrorBoundary needed
  if (!gltf) return <FallbackMesh />;
  return <primitive object={gltf.scene} />;
}
```

---

## `use()` vs `useGLTF` from Drei: Comparison

| Feature | `use()` + manual GLTFLoader | `useGLTF` from drei |
|---|---|---|
| API surface | React 19 built-in | Drei v9 helper |
| Suspense integration | Manual (via stable promise) | Automatic |
| Draco support | Manual setup required | Built-in (CDN default) |
| MeshOpt support | Manual setup required | Built-in |
| `nodes`/`materials` map | Manual | Provided via `ObjectMap` |
| Preloading | Manual cache at module scope | `useGLTF.preload(url)` |
| Dispose helper | Manual | `useGLTF.clear(url)` |
| Code volume | Higher | Lower |
| Control over loader | Full | Via `extendLoader` callback |

**When to use `use()` + manual loader:**
- You need to integrate with a non-standard loader not in drei.
- You are passing promises from a Server Component to a Client Component
  (the recommended React 19 pattern for data that originates on the server).
- You need custom caching or request deduplication logic.

**When to use `useGLTF`:**
- Standard GLB/GLTF loading in R3F contexts.
- You want Draco/MeshOpt decompression without boilerplate.
- You need the `nodes`/`materials` destructuring pattern for parametric scenes.

---

## Server Component + Client Component Pattern with `use()`

React 19 docs recommend: "Create Promises in Server Components and pass them
to Client Components."

Source: react.dev/reference/react/use, Server Components section, accessed 2026-05-26.

For 3D scenes, the model file itself lives on the client (WebGL). But model
metadata (name, position, variants, LOD URLs) can be fetched on the server
and streamed down via `use()`:

```tsx
// app/scene/page.tsx — Server Component
async function getModelManifest() {
  const res = await fetch('https://api.example.com/models/robot');
  return res.json(); // { url: '/models/robot-v3.glb', lodUrls: [...], ... }
}

export default function ScenePage() {
  // Promise not awaited — passed to client component for streaming
  const manifestPromise = getModelManifest();

  return (
    <Suspense fallback={<p>Loading scene...</p>}>
      <SceneClient manifestPromise={manifestPromise} />
    </Suspense>
  );
}
```

```tsx
// components/SceneClient.tsx — Client Component
'use client';

import { use } from 'react';
import { Canvas } from '@react-three/fiber';
import { useGLTF } from '@react-three/drei';

interface ModelManifest {
  url: string;
  position: [number, number, number];
}

function Model({ url, position }: ModelManifest) {
  const { scene } = useGLTF(url);
  return <primitive object={scene} position={position} />;
}

export default function SceneClient({
  manifestPromise,
}: {
  manifestPromise: Promise<ModelManifest>;
}) {
  // use() reads the promise; component suspends until manifest resolves
  const manifest = use(manifestPromise);

  return (
    <Canvas>
      <Suspense fallback={null}>
        <Model {...manifest} />
      </Suspense>
    </Canvas>
  );
}
```

Note: Props passed from Server Components to Client Components must be
serializable. Functions, class instances, and non-serializable objects
cannot cross the server/client boundary.

Source: nextjs.org/docs/app/getting-started/server-and-client-components
(Passing data section), accessed 2026-05-26.
