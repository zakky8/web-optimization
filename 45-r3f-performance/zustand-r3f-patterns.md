# Zustand Store Patterns for React Three Fiber

## Why Zustand Is the Idiomatic Choice

Zustand is maintained by the same team (pmndrs) as R3F and is designed from the ground up to avoid React's reconciler when you do not need it. Its `getState()` and `subscribe()` APIs operate outside React's scheduler entirely, making it the right tool for bridging application state with the Three.js render loop.

**Source:** r3f.docs.pmnd.rs/advanced/pitfalls, accessed 2026-05-26.
**Source:** github.com/pmndrs/zustand, accessed 2026-05-26.

---

## The Core Anti-Pattern: Reactive Binding at 60 fps

Using `useSelector` (or `useBearStore(s => s.x)`) inside a component that reads that value in `useFrame` causes 60 React re-renders per second — one for every frame the value changes. React's reconciler diffs the entire component tree each time.

```jsx
// BAD — pumps 60 re-renders/second through React
const x = useStore(state => state.x)
return <mesh position-x={x} />
```

The correct approach reads state directly from the store inside `useFrame`, keeping the update entirely outside React:

```jsx
// GOOD — zero React re-renders
const meshRef = useRef()
useFrame(() => {
  meshRef.current.position.x = useStore.getState().x
})
return <mesh ref={meshRef} />
```

**Exact quote from official pitfalls documentation:** "Don't bind reactive state in useFrame, instead fetch state directly via `api.getState().x`."

**Source:** r3f.docs.pmnd.rs/advanced/pitfalls, accessed 2026-05-26.

---

## `getState()` in `useFrame` (No Re-Render Pattern)

This is the canonical pattern for high-frequency state reads. `getState()` is synchronous and allocation-free — it returns a reference to the current state object.

```jsx
import { create } from 'zustand'
import { useRef } from 'react'
import { useFrame } from '@react-three/fiber'

const useGameStore = create(set => ({
  playerX: 0,
  playerY: 0,
  speed: 5,
  setPosition: (x, y) => set({ playerX: x, playerY: y }),
}))

function PlayerMesh() {
  const meshRef = useRef()

  useFrame((_, delta) => {
    const { playerX, playerY } = useGameStore.getState()
    meshRef.current.position.x = playerX
    meshRef.current.position.y = playerY
  })

  return (
    <mesh ref={meshRef}>
      <boxGeometry args={[1, 1, 1]} />
      <meshStandardMaterial color="royalblue" />
    </mesh>
  )
}
```

The component mounts once, never re-renders from store changes, and reads the latest state on every frame via `getState()`.

---

## `subscribe()` for Side Effects

`subscribe()` fires a callback whenever a state slice changes. Because it runs outside React, it is useful for:
- Triggering Three.js operations that are not per-frame (e.g., rebuilding a path on waypoint change)
- Logging or telemetry without React overhead
- Syncing with non-React imperative APIs

Basic form — fires on any state change:

```jsx
useEffect(() => {
  const unsub = useGameStore.subscribe(state => {
    if (myRef.current) {
      myRef.current.material.color.set(state.teamColor)
    }
  })
  return unsub   // auto-unsubscribes on component unmount
}, [])
```

### `subscribeWithSelector` middleware for targeted subscriptions

The base `subscribe` fires on *every* state change. The `subscribeWithSelector` middleware adds a selector argument and optional equality function so the callback only fires when a specific slice changes:

```jsx
import { create } from 'zustand'
import { subscribeWithSelector } from 'zustand/middleware'

const useStore = create(
  subscribeWithSelector(set => ({
    phase: 'intro',
    score: 0,
    setPhase: phase => set({ phase }),
    addScore: n => set(s => ({ score: s.score + n })),
  }))
)

// Subscribe only to `phase` changes — selector + optional equality fn
const unsub = useStore.subscribe(
  state => state.phase,
  (phase, prevPhase) => {
    console.log(`Phase changed from ${prevPhase} to ${phase}`)
    if (phase === 'game') startGame()
  }
)
```

The callback signature is `(newValue, previousValue)`, giving you the diff without any extra comparison logic.

**Source:** github.com/pmndrs/zustand/blob/main/src/middleware/subscribeWithSelector.ts, accessed 2026-05-26.

---

## Split Stores for Render-Critical vs UI State

Keeping all application state in one store means every subscriber — including UI components that call expensive React renders — re-renders on every frame update. The solution is to split stores by update frequency:

### Render-critical store (high frequency, no React subscribers)

Updated every frame or at high event rate. Components never `useSelector` this store; they read it with `getState()` inside `useFrame`.

```jsx
import { create } from 'zustand'
import { subscribeWithSelector } from 'zustand/middleware'

// High-frequency store — only accessed via getState() or subscribe()
export const usePhysicsStore = create(
  subscribeWithSelector(() => ({
    positions: new Float32Array(1000 * 3),
    velocities: new Float32Array(1000 * 3),
  }))
)

// Worker or physics tick writes to it imperatively:
export function writePhysics(index, x, y, z) {
  const { positions } = usePhysicsStore.getState()
  positions[index * 3 + 0] = x
  positions[index * 3 + 1] = y
  positions[index * 3 + 2] = z
  // No set() call — avoids triggering React subscribers
}
```

### UI store (low frequency, React components subscribe normally)

Updated at interaction speed — button clicks, menu navigation, settings changes.

```jsx
export const useUIStore = create(set => ({
  menuOpen: false,
  selectedLevel: 1,
  highScore: 0,
  openMenu: () => set({ menuOpen: true }),
  closeMenu: () => set({ menuOpen: false }),
  selectLevel: level => set({ selectedLevel: level }),
  recordScore: score => set(s => ({ highScore: Math.max(s.highScore, score) })),
}))

// React UI component — re-renders only when menuOpen changes
function HUD() {
  const menuOpen = useUIStore(s => s.menuOpen)
  return menuOpen ? <MenuOverlay /> : null
}
```

### Bridge pattern

A `subscribe` call in a Three.js component bridges rare UI-store changes into the render world without re-renders:

```jsx
function LevelGeometry() {
  const meshRef = useRef()

  useEffect(() => {
    return useUIStore.subscribe(
      s => s.selectedLevel,
      level => {
        // Imperatively swap geometry when level changes
        meshRef.current.geometry.dispose()
        meshRef.current.geometry = levelGeometries[level]
      }
    )
  }, [])

  return <mesh ref={meshRef}><meshStandardMaterial /></mesh>
}
```

---

## Zustand Slices Pattern for Large Stores

For projects with many distinct domains, the slices pattern splits store logic into factory functions and composes them:

```jsx
// sceneSlice.js
export const createSceneSlice = (set) => ({
  fogDensity: 0.02,
  ambientIntensity: 0.5,
  setFog: density => set({ fogDensity: density }),
  setAmbient: intensity => set({ ambientIntensity: intensity }),
})

// playerSlice.js
export const createPlayerSlice = (set, get) => ({
  health: 100,
  ammo: 30,
  takeDamage: amount => set(s => ({ health: Math.max(0, s.health - amount) })),
  reload: () => set({ ammo: 30 }),
})

// store.js — combine with middleware
import { create } from 'zustand'
import { subscribeWithSelector } from 'zustand/middleware'
import { createSceneSlice } from './sceneSlice'
import { createPlayerSlice } from './playerSlice'

export const useBoundStore = create(
  subscribeWithSelector((...a) => ({
    ...createSceneSlice(...a),
    ...createPlayerSlice(...a),
  }))
)
```

**Source:** github.com/pmndrs/zustand slices-pattern.md, accessed 2026-05-26.

---

## Temporal / Transient Store Pattern

For values that must pass through `useFrame` (camera shake magnitude, audio beat value, etc.) without ever triggering a React render, use a plain module-level `ref`-style object. No Zustand needed — just a mutable object read by `getState` or accessed directly:

```jsx
// temporal.js — not a React hook, not a store, just a plain object
export const temporal = {
  cameraShake: 0,
  beatPulse: 0,
}

// AudioAnalyzer writes to it outside React
export function onAudioFrame(analyserNode) {
  const data = new Uint8Array(analyserNode.frequencyBinCount)
  analyserNode.getByteFrequencyData(data)
  temporal.beatPulse = data[0] / 255
}

// R3F component reads it in useFrame — zero subscriptions, zero re-renders
function CameraShaker() {
  useFrame(({ camera }) => {
    camera.position.x = Math.sin(Date.now() * 0.01) * temporal.cameraShake
  })
  return null
}
```

When you do need React components to react to temporal changes (e.g., showing a beat visualizer in HTML), keep the temporal object for the Three.js side and separately call Zustand `set()` at a throttled rate (e.g., every 100 ms) for the React side.

---

## Using `useStore` to Access R3F's Internal Zustand Store

R3F itself uses Zustand for its root state (camera, renderer, size, performance, etc.). Access the internal store via `useStore()`:

```jsx
import { useStore } from '@react-three/fiber'

function ResizeWatcher() {
  useEffect(() => {
    const store = useStore()
    let prevSize = store.getState().size

    const unsub = store.subscribe(() => {
      const { size } = store.getState()
      if (size !== prevSize) {
        console.log('canvas resized to', size)
        prevSize = size
      }
    })
    return unsub
  }, [])

  return null
}
```

The maintainer recommended this pattern over direct `_roots` access.

**Source:** pmndrs/react-three-fiber Discussion #1922, accessed 2026-05-26.

---

## Decision Flowchart

```
Does the value change every frame (or > 10 Hz)?
  Yes → getState() in useFrame, do NOT useSelector
  No  → useSelector is fine, updates React normally

Does a Three.js object need to react to rare store changes?
  Yes → subscribe() + useEffect, mutate via ref imperatively

Do you need two stores with different update frequencies?
  Yes → split into render store (getState/subscribe only) + UI store (normal React hooks)

Do you need per-value subscriptions?
  Yes → add subscribeWithSelector middleware
```

---

*Sources verified against pmndrs/react-three-fiber, pmndrs/zustand master branches, and r3f.docs.pmnd.rs on 2026-05-26.*
