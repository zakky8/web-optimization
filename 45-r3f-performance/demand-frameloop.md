# R3F `frameloop="demand"` Mode

## Overview

By default React Three Fiber renders at up to 60 fps continuously (`frameloop="always"`). Setting `frameloop="demand"` switches to an on-demand model where the engine only renders when explicitly told to. The primary motivation is battery and thermal management on mobile and laptop devices — not raw throughput — though it also cuts CPU/GPU work to zero during genuinely static periods.

**Source:** `scaling-performance.mdx` in pmndrs/react-three-fiber master branch, accessed 2026-05-26.
**Source:** r3f.docs.pmnd.rs/advanced/scaling-performance, accessed 2026-05-26.

---

## Enabling Demand Mode

```jsx
<Canvas frameloop="demand">
  {/* your scene */}
</Canvas>
```

R3F automatically requests a new frame whenever **React-managed props on native Three.js elements change** (e.g., position, rotation, scale passed as JSX props). Anything that mutates the scene outside React's awareness — OrbitControls, custom event handlers, scroll listeners — must call `invalidate()` manually.

---

## The `invalidate()` Function

`invalidate()` flags the canvas for a render on the next animation frame. It does **not** render immediately; it queues exactly one frame. Calling it multiple times before that frame fires is safe — redundant calls collapse.

**Exact quote from official docs:** "Calling `invalidate()` will not render immediately, it merely requests a new frame to be rendered out."

Obtain `invalidate` from `useThree`:

```jsx
import { useThree } from '@react-three/fiber'

function Controls() {
  const orbitControlsRef = React.useRef()
  const { invalidate, camera, gl } = useThree()

  React.useEffect(() => {
    orbitControlsRef.current.addEventListener('change', invalidate)
    return () => orbitControlsRef.current.removeEventListener('change', invalidate)
  }, [])

  return <orbitControls ref={orbitControlsRef} args={[camera, gl.domElement]} />
}
```

Drei's built-in controls (OrbitControls, MapControls, etc.) call `invalidate` automatically on their `change` event, so you do not need to wire it up manually when using Drei controls.

### Pre-scheduling renders before programmatic animation

When starting a programmatic animation immediately after a user interaction, call `invalidate()` first so the first animation frame is not visually skipped:

```jsx
<mesh
  onClick={() => {
    invalidate()
    requestAnimationFrame(() => controls.dolly(1, true))
  }}
/>
```

---

## `invalidate` vs `advance`

| API | Behaviour | Typical use |
|-----|-----------|-------------|
| `invalidate()` | Queues one frame at natural vsync | Demand mode — user events, control changes |
| `advance(timestamp)` | Renders one frame synchronously right now | `frameloop="never"` — testing, XR tick loops |

`frameloop="never"` disables the automatic loop entirely; the clock only advances via `advance()` calls. This is primarily for off-screen rendering or unit tests.

---

## `useFrame` with Demand Mode

`useFrame` callbacks still run on every frame that R3F renders. In demand mode this means they run only when a frame has been requested. If you have a component that needs to keep rendering while something is happening (e.g., a running GLTF animation) it must call `invalidate()` from inside `useFrame` for as long as that work continues:

```jsx
import { useFrame, useThree } from '@react-three/fiber'

function AnimatedModel({ action }) {
  const { invalidate } = useThree()

  useFrame((_, delta) => {
    if (action.isRunning()) {
      action.getMixer().update(delta)
      invalidate()          // keep requesting frames while animation plays
    }
  })

  return null
}
```

**Source:** pmndrs/react-three-fiber Discussion #1800, accessed 2026-05-26.

The maintainer confirmed: calling `invalidate()` inside `useFrame` has no meaningful performance overhead; the check itself is trivial.

---

## Known Bug in v9 (loop.ts — January 2025)

A regression in the v9 branch causes a single call to `invalidate()` to invoke `useFrame` callbacks **twice** per tick. The root cause is a typo in `loop.ts` where the second assignment of `useFrameInProgress` is set to `true` instead of `false`.

**Source:** pmndrs/react-three-fiber Issue #3436, accessed 2026-05-26.

Check your r3f version and the upstream fix status before relying on exact frame counts in demand mode.

---

## When to Use Demand Mode

**Good fit:**
- Static presentation scenes (product viewers, portfolio sites) where the model sits still until the user interacts
- Dashboard UIs that embed a 3D widget alongside HTML — the 3D is decorative and should not burn battery while the user reads text
- Any scene where you can enumerate all events that cause visual change and call `invalidate()` from each

**Poor fit:**
- Continuous particle systems, fluid simulations, or procedurally animated scenes — you end up calling `invalidate()` every frame anyway, giving you demand mode overhead with no benefit
- Real-time multiplayer games where server state streams in unpredictably
- Scenes with third-party animation libraries that do not natively support `invalidate` (though react-spring does)

---

## Combining with Scroll Animations

Scroll-driven scenes are one of the most common demand-mode patterns. The browser fires `scroll` events faster than 60 fps on some devices; calling `invalidate()` from inside the handler is correct because R3F collapses duplicate requests:

```jsx
import { useThree } from '@react-three/fiber'
import { useEffect, useRef } from 'react'

function ScrollScene() {
  const { invalidate } = useThree()
  const scrollY = useRef(0)

  useEffect(() => {
    const onScroll = () => {
      scrollY.current = window.scrollY
      invalidate()
    }
    window.addEventListener('scroll', onScroll, { passive: true })
    return () => window.removeEventListener('scroll', onScroll)
  }, [invalidate])

  const meshRef = useRef()
  useFrame(() => {
    if (meshRef.current) {
      meshRef.current.position.y = -scrollY.current * 0.01
    }
  })

  return <mesh ref={meshRef}><boxGeometry /><meshStandardMaterial /></mesh>
}
```

For production scroll rigs, the `@14islands/r3f-scroll-rig` library implements this pattern with viewport-intersection awareness, calling `invalidate()` only while the canvas is visible.

**Source:** github.com/14islands/r3f-scroll-rig, accessed 2026-05-26.

---

## Pausing When Off-Screen (Intersection Observer)

Combine demand mode with an Intersection Observer to fully pause rendering when the canvas scrolls out of the viewport:

```jsx
import { useEffect, useRef } from 'react'
import { Canvas, useThree } from '@react-three/fiber'

function PauseWhenHidden() {
  const { set } = useThree()

  useEffect(() => {
    const canvas = document.querySelector('canvas')
    const observer = new IntersectionObserver(
      ([entry]) => {
        set({ frameloop: entry.isIntersecting ? 'demand' : 'never' })
      },
      { threshold: 0 }
    )
    observer.observe(canvas)
    return () => observer.disconnect()
  }, [set])

  return null
}
```

**Source:** pmndrs/react-three-fiber Discussion #1701, accessed 2026-05-26.

---

## Frameloop Mode Comparison

| Mode | Renders when | Use case |
|------|-------------|---------|
| `"always"` | Every vsync tick | Continuously animated scenes, games |
| `"demand"` | `invalidate()` called or JSX props change | Static/interactive scenes, scroll animations |
| `"never"` | Only via `advance(timestamp)` | Off-screen rendering, XR custom loops, tests |

---

*Sources verified against pmndrs/react-three-fiber master branch and r3f.docs.pmnd.rs on 2026-05-26.*
