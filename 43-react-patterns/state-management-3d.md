# State Management for 3D Scenes

**Sources verified:** 2026-05-26
**Primary sources:** r3f.docs.pmnd.rs/advanced/pitfalls, r3f.docs.pmnd.rs/api/hooks, github.com/pmndrs/zustand

---

## The Core Problem: React State vs Frame-Rate Updates

React's state model is designed for event-driven UI. Three.js runs a render loop
at 60+ fps. These two models conflict fundamentally:

- `setState` schedules a reconciliation pass through React's scheduler.
- At 60fps, calling `setState` 60 times per second triggers 60 reconciliation
  passes per second — re-renders every component subscribed to that state.
- R3F documentation states: "You should never setState in [useFrame]!"

Source: r3f.docs.pmnd.rs/advanced/pitfalls, accessed 2026-05-26.

The solution: **React manages scene configuration; refs and direct mutation
manage per-frame values.**

---

## Refs vs State: Decision Rule

| Use case | Use |
|---|---|
| Object position/rotation updated every frame | `useRef` + direct mutation |
| Animation progress (not exposed to React) | `useRef` |
| Camera target following a physics body | `useRef` |
| UI toggle that shows/hides a mesh | `useState` or Zustand |
| Selected object ID (drives UI outside Canvas) | Zustand |
| Loaded asset URL (drives `useGLTF`) | `useState` or Zustand |
| Material color set once on load | `useMemo` |
| Whether a cutscene is playing | Zustand |

**Rule:** if the value changes more than a few times per second, it must not be
React state. Put it in a ref and mutate directly.

---

## useFrame: the Render Loop

`useFrame` executes a callback on every rendered frame. It receives the R3F
state object and a delta (seconds since last frame):

```typescript
useFrame((state: RootState, delta: number) => void)
```

Delta-based animation is required for refresh-rate independence. A 144Hz monitor
would animate at 2.4x the speed of 60Hz if you use fixed increments.

Source: r3f.docs.pmnd.rs/advanced/pitfalls (Use deltas section), accessed 2026-05-26.

### Basic ref mutation pattern

```tsx
import { useRef } from 'react';
import { useFrame } from '@react-three/fiber';
import type { Mesh } from 'three';

function SpinningBox() {
  const meshRef = useRef<Mesh>(null);

  useFrame((_state, delta) => {
    if (!meshRef.current) return;
    // Direct mutation — no React re-render, no scheduler involvement
    meshRef.current.rotation.y += delta;
  });

  return (
    <mesh ref={meshRef}>
      <boxGeometry />
      <meshStandardMaterial />
    </mesh>
  );
}
```

---

## Zustand for Scene State

Zustand is the recommended state manager for R3F projects. It is built by the
same organization (pmndrs) and is designed to coexist with the ref-mutation
pattern via `getState()`.

Source: r3f.docs.pmnd.rs/advanced/pitfalls (State management section), accessed 2026-05-26.

### Setting up a scene store

```typescript
// stores/sceneStore.ts
import { create } from 'zustand';

interface SceneState {
  selectedId: string | null;
  isPlaying: boolean;
  cameraTarget: [number, number, number];

  // Actions
  setSelected: (id: string | null) => void;
  setPlaying: (v: boolean) => void;
  setCameraTarget: (pos: [number, number, number]) => void;
}

export const useSceneStore = create<SceneState>((set) => ({
  selectedId: null,
  isPlaying: false,
  cameraTarget: [0, 0, 0],

  setSelected: (id) => set({ selectedId: id }),
  setPlaying: (v) => set({ isPlaying: v }),
  setCameraTarget: (pos) => set({ cameraTarget: pos }),
}));
```

### Reactive subscription (causes re-render — use for UI)

```tsx
// A DOM overlay showing the selected object name — re-rendering is fine here
function SelectionPanel() {
  const selectedId = useSceneStore((state) => state.selectedId);
  return <div>Selected: {selectedId ?? 'none'}</div>;
}
```

### Zero-re-render reads inside useFrame

`getState()` retrieves the current Zustand state at call time without
establishing a React subscription. Inside `useFrame`, this is the correct
access pattern.

R3F pitfalls documentation:

> "fetch state directly within useFrame"
> `useFrame(() => (ref.current.position.x = api.getState().x))`

Source: r3f.docs.pmnd.rs/advanced/pitfalls, accessed 2026-05-26.

```tsx
import { useRef } from 'react';
import { useFrame } from '@react-three/fiber';
import { useSceneStore } from '../stores/sceneStore';
import type { PerspectiveCamera } from 'three';

function CameraRig() {
  const cameraRef = useRef<PerspectiveCamera>(null);

  useFrame((_state, delta) => {
    if (!cameraRef.current) return;

    // getState() reads current value without subscribing the component
    const target = useSceneStore.getState().cameraTarget;

    // Lerp toward target — direct mutation, no setState
    cameraRef.current.position.x +=
      (target[0] - cameraRef.current.position.x) * delta * 5;
    cameraRef.current.position.y +=
      (target[1] - cameraRef.current.position.y) * delta * 5;
    cameraRef.current.position.z +=
      (target[2] - cameraRef.current.position.z) * delta * 5;
  });

  return null;
}
```

The `CameraRig` component itself never re-renders due to store changes — it
reads state imperatively inside `useFrame` and mutates the camera directly.

---

## Transient Updates: subscribe Without Re-render

Zustand's `subscribe` lets you react to store changes without causing a
React render. Combine with `useEffect` for proper cleanup on unmount:

```tsx
import { useEffect, useRef } from 'react';
import { useSceneStore } from '../stores/sceneStore';
import type { Mesh } from 'three';

function SelectionHighlight() {
  const meshRef = useRef<Mesh>(null);

  useEffect(() => {
    // Subscribe fires on every state change but does not trigger re-render
    const unsub = useSceneStore.subscribe(
      (state) => state.selectedId,
      (selectedId) => {
        if (!meshRef.current) return;
        // Mutate material directly — no React involvement
        (meshRef.current.material as THREE.MeshStandardMaterial).emissive
          .set(selectedId ? '#4fc3f7' : '#000000');
      }
    );
    return unsub; // Zustand unsubscribe on unmount
  }, []);

  return (
    <mesh ref={meshRef}>
      <boxGeometry />
      <meshStandardMaterial />
    </mesh>
  );
}
```

Source: github.com/pmndrs/zustand, Transient updates section, accessed 2026-05-26.

---

## useThree: Accessing Renderer State Without Re-render

`useThree` from R3F provides the camera, renderer, scene, and viewport. Use
selectors to avoid re-rendering when unrelated state changes:

```tsx
// Only re-renders when camera reference changes (rare)
const camera = useThree((state) => state.camera);

// Use the get() function for non-reactive imperative access
// Useful inside event handlers and callbacks
const { get } = useThree();
function handleSomething() {
  const { camera, scene } = get(); // reads without subscription
}
```

Source: r3f.docs.pmnd.rs/api/hooks (useThree section), accessed 2026-05-26.

---

## Visibility Over Mounting: Avoiding Recompilation Cost

A common mistake is conditionally mounting/unmounting meshes based on state.
Every time a Three.js object mounts in R3F, geometries and materials are
compiled by the GPU driver. This causes frame drops ("jank").

R3F pitfalls:

> "Replace conditional mounting with visibility props to prevent expensive
> material/geometry recompilation."

Source: r3f.docs.pmnd.rs/advanced/pitfalls, accessed 2026-05-26.

```tsx
// BAD — material recompiles every time isOpen flips
{isOpen && <PopupPanel />}

// GOOD — GPU resources stay allocated, just invisible
<PopupPanel visible={isOpen} />
```

`visible={false}` sets `THREE.Object3D.visible = false`, which skips the
draw call but keeps the geometry/material on the GPU.

---

## Pattern Summary

```
Slow-changing scene config (URLs, IDs, toggles)
  → Zustand store → reactive useSceneStore() subscription
  → React re-renders are acceptable here

Fast-changing per-frame values (position, rotation, morph targets)
  → useRef → direct mutation in useFrame
  → zero React re-renders

Responding to Zustand changes without re-render
  → useSceneStore.subscribe() in useEffect
  → mutate ref directly in subscriber callback

Reading Zustand inside useFrame
  → useSceneStore.getState() — imperative, no subscription
```
