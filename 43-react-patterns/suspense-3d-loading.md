# Suspense for 3D Asset Loading

**Sources verified:** 2026-05-26
**Primary sources:** react.dev/reference/react/Suspense, r3f.docs.pmnd.rs, drei.docs.pmnd.rs/loaders/gltf-use-gltf

---

## How Suspense Works with 3D Assets

R3F's `useLoader` (and `useGLTF` which wraps it) throws a promise during render
when an asset is not yet cached. React catches the thrown promise, shows the
nearest `<Suspense>` fallback, and resumes the component tree once the promise
resolves. This is the same mechanism React 19 formalises with the `use()` hook,
but `useLoader`/`useGLTF` predates `use()` and implements it via the
"throw-a-promise" (Suspense protocol) pattern directly.

Source: r3f.docs.pmnd.rs/api/hooks (useLoader section), accessed 2026-05-26.

---

## useGLTF in Suspense Mode

`useGLTF` from `@react-three/drei` is Suspense-enabled by default. The function
signature:

```typescript
useGLTF<T extends string | string[]>(
  path: T,
  useDraco?: boolean | string,  // default: true (CDN Draco binaries)
  useMeshOpt?: boolean,          // default: true
  extendLoader?: (loader: GLTFLoader) => void
): T extends any[] ? (GLTF & ObjectMap)[] : GLTF & ObjectMap
```

The return type merges the raw GLTF interface with an `ObjectMap` that provides:
- `nodes` — all `THREE.Object3D` instances keyed by name
- `materials` — all `THREE.Material` instances keyed by name

Source: drei.docs.pmnd.rs/loaders/gltf-use-gltf, accessed 2026-05-26.

### Basic usage

```tsx
'use client';

import { Suspense } from 'react';
import { Canvas } from '@react-three/fiber';
import { useGLTF } from '@react-three/drei';

function Robot() {
  // Suspends until loaded; R3F caches by URL so subsequent mounts are instant
  const { scene } = useGLTF('/models/robot.glb');
  return <primitive object={scene} />;
}

export default function Scene() {
  return (
    <Canvas>
      <Suspense fallback={null}>
        <Robot />
      </Suspense>
    </Canvas>
  );
}
```

When `Robot` suspends, React shows `fallback={null}` (no visible placeholder).
The scene reveals atomically once the GLB is parsed and textures are decoded.

---

## Preloading: Starting Fetches Before Mount

`useGLTF.preload(url)` initiates the network request and parse at module load
time, before any component mounts. This eliminates the visible suspension
entirely for assets you know will be needed.

```tsx
// Start the fetch immediately when the module is evaluated
useGLTF.preload('/models/robot.glb');
useGLTF.preload('/models/environment.glb');

function Robot() {
  // By the time this renders, the asset is already in cache
  const { scene } = useGLTF('/models/robot.glb');
  return <primitive object={scene} />;
}
```

Call `useGLTF.preload()` at module scope (outside any component function) so it
runs when the JS bundle is parsed. Works with code-split bundles — call it in
the same file as the component that uses the model.

---

## Nested Suspense Boundaries for Progressive Loading

React processes nested `<Suspense>` boundaries independently. Inner boundaries
show their own fallbacks while outer content has already revealed. This enables
progressive 3D scene loading: render the environment first, then load characters
one by one.

React's documented loading sequence for nested boundaries:

> "If `Biography` hasn't loaded → show `BigSpinner`. Once `Biography` loads →
> replace `BigSpinner` with biography content. If `Albums` still loading →
> show `AlbumsGlimmer` for that section only."

Source: react.dev/reference/react/Suspense, Nested Suspense example, accessed 2026-05-26.

### Applying progressive loading to 3D scenes

```tsx
'use client';

import { Suspense } from 'react';
import { Canvas } from '@react-three/fiber';
import { useGLTF } from '@react-three/drei';

// Preload both, but let them reveal independently
useGLTF.preload('/models/environment.glb');
useGLTF.preload('/models/character.glb');

function Environment() {
  const { scene } = useGLTF('/models/environment.glb');
  return <primitive object={scene} />;
}

function Character() {
  const { scene } = useGLTF('/models/character.glb');
  return <primitive object={scene} position={[0, 0, 0]} />;
}

function LoadingRing() {
  // A lightweight Three.js placeholder — no asset fetch needed
  return (
    <mesh>
      <torusGeometry args={[1, 0.05, 8, 32]} />
      <meshBasicMaterial color="gray" wireframe />
    </mesh>
  );
}

export default function Scene() {
  return (
    <Canvas>
      {/* Outer boundary: nothing visible until environment loads */}
      <Suspense fallback={null}>
        <Environment />

        {/* Inner boundary: spinning ring while character loads */}
        <Suspense fallback={<LoadingRing />}>
          <Character />
        </Suspense>
      </Suspense>
    </Canvas>
  );
}
```

Nesting rules:
- Inner `<Suspense>` catches its subtree's suspension; outer does not see it.
- If the inner component's promise rejects, the nearest `<ErrorBoundary>` above
  it catches the error (not the Suspense boundary).
- A `key` prop on `<Suspense>` resets it, forcing the fallback to show again
  when the key changes — useful when navigating between different scenes.

---

## Custom Loading UI Inside the Canvas

The `fallback` prop accepts any React subtree, including R3F elements. Because
`<Suspense>` fallbacks render in the same context, they have access to the
Canvas renderer and can display Three.js geometry.

```tsx
function AssetLoadingIndicator() {
  const [progress, setProgress] = React.useState(0);

  useEffect(() => {
    // Tie into drei's useProgress for actual load percentage
  }, []);

  return (
    <mesh rotation={[0, 0, progress * Math.PI * 2]}>
      <ringGeometry args={[0.8, 1.0, 32]} />
      <meshBasicMaterial color="#4fc3f7" />
    </mesh>
  );
}

// In practice, use drei's Html or Loader for 2D overlays:
import { Html, useProgress } from '@react-three/drei';

function CanvasLoader() {
  const { progress } = useProgress();
  return (
    <Html center>
      <div style={{ color: 'white', fontSize: 14 }}>
        {Math.round(progress)}% loaded
      </div>
    </Html>
  );
}

function Scene() {
  return (
    <Canvas>
      <Suspense fallback={<CanvasLoader />}>
        <HeavyModel />
      </Suspense>
    </Canvas>
  );
}
```

`useProgress` from drei hooks into `THREE.DefaultLoadingManager` and provides
`progress` (0–100), `item` (currently loading URL), `loaded`, and `total`.

---

## ErrorBoundary + Suspense Pattern

Suspense handles the pending state. `ErrorBoundary` handles the rejected state.
They are separate concerns and should be nested together:

```tsx
import { ErrorBoundary } from 'react-error-boundary';
import { Suspense } from 'react';

function ModelFallback({ error, resetErrorBoundary }: {
  error: Error;
  resetErrorBoundary: () => void;
}) {
  return (
    <Html center>
      <div>
        <p>Failed to load model: {error.message}</p>
        <button onClick={resetErrorBoundary}>Retry</button>
      </div>
    </Html>
  );
}

export default function SafeScene() {
  return (
    <Canvas>
      <ErrorBoundary FallbackComponent={ModelFallback}>
        <Suspense fallback={<CanvasLoader />}>
          <HeavyModel />
        </Suspense>
      </ErrorBoundary>
    </Canvas>
  );
}
```

**Order matters:** `ErrorBoundary` must be the outer wrapper. If Suspense is
outside ErrorBoundary, a thrown error during loading bypasses the boundary.

React docs confirm: "If [a promise] rejects, the nearest Error Boundary fallback
is displayed."

Source: react.dev/reference/react/use (Error Handling section), accessed 2026-05-26.

---

## Suspense Behavior Fixed in R3F v9

R3F v9 migration guide notes that prior to v9, side-effects like `attach` and
event listeners would "fire repeatedly without proper cleanup during suspension."
In v9 this is resolved: suspending a component no longer causes attach effects
to accumulate. This means multiple suspensions during concurrent rendering (e.g.
while streaming multiple assets) will not leave duplicate material bindings or
ghost listeners.

Source: r3f.docs.pmnd.rs/tutorials/v9-migration-guide, accessed 2026-05-26.
