# React 19 with R3F v9: Setup and Concurrent Mode

**Sources verified:** 2026-05-26
**Primary sources:** r3f.docs.pmnd.rs/tutorials/v9-migration-guide, react.dev/blog/2024/12/05/react-19, github.com/pmndrs/react-three-fiber CHANGELOG.md

---

## Version Pairing

`@react-three/fiber@9` pairs with `react@19`. This is not optional — R3F v9 is a
dedicated compatibility release. R3F v8 will not work correctly with React 19.

| Package | Version |
|---|---|
| react | 19.x |
| react-dom | 19.x |
| @react-three/fiber | 9.x |
| @react-three/drei | 9.x |
| three | 0.170+ |

R3F v9.5.0 extended compatibility to React 19.2. When React 19.2 bumped its
internal reconciler version (not backward-compatible with 19.1), R3F bundled the
reconciler to support the full 19.0–19.2 range from a single package version.

Source: github.com/pmndrs/react-three-fiber CHANGELOG, v9.5.0 entry, accessed 2026-05-26.

---

## Concurrent Mode Compatibility

React 19 ships concurrent rendering as the default. R3F v9 is built against it.
Key behavioral implications for 3D work:

- React may interrupt and restart renders. Three.js object construction inside
  render (not `useMemo`/`useRef`) can run more than once.
- `useFrame` runs outside React's scheduler entirely — it is not affected by
  concurrent interruptions. Frame callbacks always complete their current tick.
- The `gl.render()` call inside R3F's loop is synchronous and atomic relative
  to Three.js; React concurrency only affects component tree reconciliation,
  not the WebGL draw calls.

**Practical rule:** never construct `BufferGeometry`, `Material`, or `Texture`
instances directly in the component body. Wrap them in `useMemo` so concurrent
mode re-renders do not create orphaned GPU resources.

```tsx
// BAD — geometry created every render, orphaned on concurrent restarts
function Box() {
  const geo = new THREE.BoxGeometry(1, 1, 1); // leaks on concurrent re-render
  return <mesh geometry={geo} />;
}

// GOOD
function Box() {
  const geo = useMemo(() => new THREE.BoxGeometry(1, 1, 1), []);
  return <mesh geometry={geo} />;
}
```

---

## StrictMode Double-Effect Behavior

React 19 (like React 18) runs `useEffect` twice in `<StrictMode>` in development:
mount → unmount → mount. This surfaces cleanup bugs that would otherwise only
appear in production after a component is removed and re-added.

R3F v9 changed StrictMode inheritance: **"StrictMode is now correctly inherited
from a parent renderer."** You no longer need to redeclare `<StrictMode>` inside
the Canvas. If your app wraps the whole tree in `<StrictMode>`, the Canvas
inherits it. This means any missing cleanup inside your 3D components will now
surface as double-mount artifacts during development.

Source: r3f.docs.pmnd.rs/tutorials/v9-migration-guide, StrictMode section, accessed 2026-05-26.

### What double-mount exposes

In strict mode the sequence is:
1. Component mounts — Three.js resources allocated, event listeners attached.
2. Component unmounts — cleanup function runs (or does not).
3. Component mounts again — resources allocated a second time.

If cleanup is missing, you end up with:
- Two geometries in GPU memory for one visible mesh.
- Duplicate event listeners firing twice per pointer event.
- R3F attach effects (e.g. `attach="material"`) firing without being removed.

R3F v9 release notes confirm: side-effects like `attach` and event listeners
"no longer fire repeatedly without proper cleanup during suspension."

Source: r3f.docs.pmnd.rs/tutorials/v9-migration-guide, accessed 2026-05-26.

---

## useEffect Cleanup for Geometry and Material Disposal

Three.js does not participate in JavaScript garbage collection for GPU resources.
`BufferGeometry`, `Material`, and `Texture` objects each hold WebGL handles
(VBOs, shader programs, texture units) that remain allocated until `.dispose()`
is explicitly called. A JavaScript GC sweep frees the JS wrapper object but
leaves the GPU allocation live.

Source: threejs.org/manual/en/cleanup.html, accessed 2026-05-26.

### Correct cleanup pattern

```tsx
import { useEffect, useRef } from 'react';
import * as THREE from 'three';

function CustomMesh() {
  const meshRef = useRef<THREE.Mesh>(null);

  useEffect(() => {
    const geometry = new THREE.SphereGeometry(1, 32, 32);
    const material = new THREE.MeshStandardMaterial({ color: 'hotpink' });
    const mesh = new THREE.Mesh(geometry, material);

    if (meshRef.current) {
      meshRef.current.add(mesh);
    }

    return () => {
      // Called on unmount AND on strict-mode remount cycle
      geometry.dispose();
      material.dispose();
      mesh.parent?.remove(mesh);
    };
  }, []);

  return <group ref={meshRef} />;
}
```

### Traversal pattern for loaded scenes

For GLTF scenes where you do not know every child in advance:

```tsx
useEffect(() => {
  return () => {
    scene.traverse((child) => {
      if (child instanceof THREE.Mesh) {
        child.geometry.dispose();
        const mats = Array.isArray(child.material)
          ? child.material
          : [child.material];
        mats.forEach((m) => {
          Object.values(m).forEach((v) => {
            if (v instanceof THREE.Texture) v.dispose();
          });
          m.dispose();
        });
      }
    });
  };
}, [scene]);
```

Source: threejs.org/manual/en/cleanup.html, ResourceTracker section, accessed 2026-05-26.

---

## Breaking Changes in v9 That Affect Setup

1. **`Props` export removed.** Use `CanvasProps` instead.
2. **`MeshProps`, `GroupProps`, etc. removed as direct exports.** Access via
   `ThreeElements['mesh']`, `ThreeElements['group']`, etc.
3. **`Node`, `Object3DNode`, `BufferGeometryNode`, `MaterialNode`, `LightNode`
   helper types removed.** Consolidated into `ThreeElement<T>`.
4. **Global JSX namespace deprecated.** Use module augmentation of
   `ThreeElements` interface instead.
5. **Automatic sRGB conversion of texture props removed.** Manually set
   `texture.colorSpace = THREE.SRGBColorSpace` on color textures.

Source: r3f.docs.pmnd.rs/tutorials/v9-migration-guide, Breaking Changes section, accessed 2026-05-26.

---

## Minimal Working Setup (React 19 + R3F v9)

```tsx
// app/layout.tsx — Next.js 15 App Router example
import type { ReactNode } from 'react';

export default function RootLayout({ children }: { children: ReactNode }) {
  return (
    <html lang="en">
      <body>{children}</body>
    </html>
  );
}
```

```tsx
// components/Scene.tsx
'use client'; // Required — Canvas uses browser WebGL API

import { Canvas } from '@react-three/fiber';
import { Suspense } from 'react';

export default function Scene() {
  return (
    <Canvas>
      <Suspense fallback={null}>
        {/* 3D content here */}
      </Suspense>
    </Canvas>
  );
}
```

The `'use client'` directive is mandatory. `Canvas` calls `WebGLRenderer` which
accesses `window` and `document` — both absent on the server. Without the
directive in Next.js App Router, the module errors during server rendering.
