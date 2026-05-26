# @react-three/xr — React Three Fiber XR Integration

**Last verified:** 2026-05-26  
**Package:** `@react-three/xr`  
**Latest version:** `6.6.0` (released 2025-01-30)  
**Repository:** `github.com/pmndrs/xr`  
**Sources:** pmndrs/xr GitHub repo docs (tier 1, accessed 2026-05-26), pmnd.rs blog reintroduction post (tier 1), GitHub releases page (tier 1)

---

## What Changed in v6 (vs v5)

v6 is a major rewrite published in 2024. The core philosophy shifted to align with the rest of the react-three ecosystem:

- **No more `XRCanvas`** as a separate component — use standard `<Canvas>` from `@react-three/fiber` wrapped by `<XR>`.
- **Controllers, hands, and transient pointers are added by default** — no `<Controllers>` or `<Hands>` import needed.
- **Pointer events work the same as in r3f** — `onClick`, `onPointerDown`, etc., fire in XR the same way they fire on desktop.
- **`createXRStore()`** replaces the old context pattern.
- A compatibility shim exists (`XRButton`, `ARButton`, `VRButton`, `useInteraction`, `useXREvent`, `Interactive`, `RayGrab`) for migrating v5 code, but it is deprecated — use the new API.

---

## Installation

```bash
npm install three @react-three/fiber @react-three/xr@latest
```

Peer dependencies: `three >=0.168.0`, `react >=18.0.0`, `@react-three/fiber >=8.0.0`.

---

## Core Setup

### 1. Create the XR Store

The store is the central control object. Create it once outside your component tree.

```tsx
import { createXRStore } from '@react-three/xr';

const store = createXRStore();
```

`createXRStore()` accepts an options object — see the Options section below.

### 2. Wrap Your Canvas

```tsx
import { Canvas } from '@react-three/fiber';
import { XR } from '@react-three/xr';

export function App() {
  return (
    <>
      <button onClick={() => store.enterAR()}>Enter AR</button>
      <button onClick={() => store.enterVR()}>Enter VR</button>

      <Canvas>
        <XR store={store}>
          {/* All r3f scene content here */}
          <ambientLight />
          <mesh>
            <boxGeometry />
            <meshStandardMaterial />
          </mesh>
        </XR>
      </Canvas>
    </>
  );
}
```

### 3. Entering Sessions

```ts
store.enterAR();  // requests immersive-ar
store.enterVR();  // requests immersive-vr
store.enterXR('immersive-ar'); // explicit mode string
```

All return `Promise<XRSession | undefined>`. `undefined` means the session could not start (device not supported, permissions denied, etc.).

---

## useXR Hook

Access the XR store state from any component inside `<XR>`:

```tsx
import { useXR } from '@react-three/xr';

function SceneComponent() {
  // Pass a selector to subscribe to a slice of state
  const session     = useXR(state => state.session);
  const mode        = useXR(state => state.mode);        // 'immersive-vr' | 'immersive-ar' | 'inline' | null
  const isPresenting = useXR(state => state.mode != null);

  return <mesh visible={isPresenting}>...</mesh>;
}
```

**Available state properties:**

| Property | Type | Description |
|----------|------|-------------|
| `session` | `XRSession \| null` | Active XR session |
| `mode` | `'immersive-vr' \| 'immersive-ar' \| 'inline' \| null` | Current session mode; `null` when not in XR |
| `originReferenceSpace` | `XRReferenceSpace \| null` | Origin reference space |
| `origin` | `THREE.Object3D \| undefined` | The XR origin object in the scene |
| `domOverlayRoot` | `HTMLElement \| null` | DOM element for AR overlay |
| `visibilityState` | string | Session visibility state (e.g. `'visible-blurred'`) |
| `frameRate` | number | Configured frame rate |
| `detectedPlanes` | `ReadonlyArray<XRPlane>` | Planes detected in current session |
| `detectedMeshes` | `ReadonlyArray<XRMesh>` | Meshes detected in current session |

---

## Reading Controller Input (Gamepad)

```tsx
import { useXRInputSourceState, XROrigin } from '@react-three/xr';
import { useFrame } from '@react-three/fiber';
import { useRef, Group } from 'react';

function Locomotion() {
  const controller = useXRInputSourceState('controller', 'right');
  const ref = useRef<Group>(null);

  useFrame((_, delta) => {
    if (!ref.current || !controller) return;
    const thumbstick = controller.gamepad['xr-standard-thumbstick'];
    if (!thumbstick) return;
    ref.current.position.x += (thumbstick.xAxis ?? 0) * delta;
    ref.current.position.z += (thumbstick.yAxis ?? 0) * delta;
  });

  return <XROrigin ref={ref} />;
}
```

`useXRInputSourceState('controller', 'left' | 'right')` — returns the gamepad state for the specified hand or `null` if that controller is not connected.

---

## Interactions (Pointer Events)

In v6, interactions use standard r3f pointer events — no special XR wrapper needed:

```tsx
<mesh
  onClick={(e) => console.log('clicked in XR or desktop', e)}
  onPointerDown={(e) => { /* start drag */ }}
  onPointerUp={(e) => { /* end drag */ }}
  onPointerEnter={(e) => { /* hover */ }}
>
  <boxGeometry />
</mesh>
```

This works identically whether the user is in VR with a controller, AR with a tap, or on desktop with a mouse.

---

## Hit Testing in @react-three/xr v6

Three hooks are available. All require `hitTest: true` in store options (enabled by default).

### useXRHitTest — Continuous (most common)

```tsx
import { useXRHitTest } from '@react-three/xr';
import { Matrix4, Vector3 } from 'three';
import { useRef } from 'react';

const matHelper = new Matrix4();
const pos       = new Vector3();

function ReticleMesh() {
  const meshRef = useRef();

  useXRHitTest(
    (results, getWorldMatrix) => {
      if (results.length === 0) {
        if (meshRef.current) meshRef.current.visible = false;
        return;
      }
      getWorldMatrix(matHelper, results[0]);
      pos.setFromMatrixPosition(matHelper);
      if (meshRef.current) {
        meshRef.current.visible = true;
        meshRef.current.position.copy(pos);
      }
    },
    'viewer', // ray origin: 'viewer' | ref to Object3D | XRSpace
    'plane'   // trackable type: 'plane' | 'point' | 'mesh'
  );

  return (
    <mesh ref={meshRef} rotation={[-Math.PI / 2, 0, 0]}>
      <ringGeometry args={[0.08, 0.10, 32]} />
      <meshBasicMaterial color="white" />
    </mesh>
  );
}
```

### useXRHitTestSource — Manual / Conditional

Lower-level. Returns an object with a `.source` (the native `XRHitTestSource`) that you query manually via `frame.getHitTestResults(source)`. Use when you need conditional or batched hit tests.

### useXRRequestHitTest — One-shot / Event-driven

```tsx
const requestHitTest = useXRRequestHitTest();

const handleTap = async () => {
  const result = await requestHitTest('viewer', ['plane', 'mesh']);
  if (result.results?.length > 0) {
    const mat = new Matrix4();
    result.getWorldMatrix(mat, result.results[0]);
    // place object at matrix position
  }
};
```

### XRHitTest Component

Component wrapper for `useXRHitTest`:

```tsx
import { XRHitTest } from '@react-three/xr';

<XRHitTest
  space={targetRaySpace}
  onResults={(results, getWorldMatrix) => { ... }}
/>
```

---

## createXRStore Options (Key Subset)

```ts
createXRStore({
  controller:    true,        // default XRController rendered on both hands
  hand:          true,        // hand tracking enabled
  gaze:          true,        // gaze input
  screenInput:   true,        // handheld AR tap input
  hitTest:       true,        // enables hit-test feature in session
  anchors:       true,        // enables anchors feature
  planeDetection: true,       // plane detection
  meshDetection:  true,       // mesh detection
  layers:        true,        // WebXR layers
  frameRate:     'high',      // 'high' | number
  foveation:     undefined,   // 0 (none) – 1 (max foveation)
  emulate:       'metaQuest3',// in-browser emulation on localhost via IWER
  offerSession:  true,        // browser shows its own XR entry UI automatically
  domOverlay:    true,        // DOM overlay in handheld AR
})
```

---

## Store Functions

```ts
store.enterAR()               // → Promise<XRSession | undefined>
store.enterVR()               // → Promise<XRSession | undefined>
store.enterXR('immersive-ar') // → Promise<XRSession | undefined>
store.setController(impl, handedness?)
store.setHand(impl, handedness?)
store.setFrameRate(72)
store.requestFrame()          // → Promise<XRFrame>
store.destroy()
```

---

## v5 → v6 Migration Reference

| v5 | v6 equivalent |
|----|--------------|
| `<XRCanvas>` | `<Canvas><XR store={store}>` |
| `<Controllers />` | Automatic — remove it |
| `<Hands />` | Automatic — remove it |
| `useXR().isPresenting` | `useXR(s => s.mode != null)` |
| `useXR().controllers` | `useXR(s => s.controllers)` (UNVERIFIED — check store.md) |
| `useController('right')` | `useXRInputSourceState('controller', 'right')` |
| `useInteraction(mesh, 'onSelect', fn)` | `<mesh onPointerDown={fn}>` |
| `useXREvent('select', fn)` | compatibility shim available; prefer r3f events |
| `<Interactive>` | Not needed; all meshes receive pointer events |
| `<RayGrab>` | Compatibility shim available in `@react-three/xr/compat` |

---

## Version History (Key Releases)

| Version | Date | Key Change |
|---------|------|-----------|
| **6.6.0** | 2025-01-30 | IWER synthetic environment module for debugging; offer-session support |
| 6.4.0 | 2024-11-01 | `PointerEvents` component; `useXRControllerLocomotion` (replaces `useControllerLocomotion`) |
| 6.3.0 | 2024-11-01 | Default controller/hand alignment with WebXR standards |
| 6.2.0 | 2024-08-19 | Unbounded space, depth sensing, secondary input sources |
| 6.1.0 | 2024-08-01 | Anchors, hit testing, DOM overlay, IWER emulator integration |

---

## Sources

- [pmndrs/xr GitHub repository](https://github.com/pmndrs/xr) — accessed 2026-05-26, tier 1
- [pmndrs/xr store tutorial doc](https://github.com/pmndrs/xr/blob/main/docs/tutorials/store.md) — accessed 2026-05-26, tier 1
- [pmndrs/xr hit-test tutorial doc](https://github.com/pmndrs/xr/blob/main/docs/tutorials/hit-test.md) — accessed 2026-05-26, tier 1
- [pmndrs/xr gamepad tutorial doc](https://github.com/pmndrs/xr/blob/main/docs/tutorials/gamepad.md) — accessed 2026-05-26, tier 1
- [pmndrs/xr migration from v5 doc](https://github.com/pmndrs/xr/blob/main/docs/migration/from-react-three-xr-5.md) — accessed 2026-05-26, tier 1
- [pmndrs/xr releases page](https://github.com/pmndrs/xr/releases) — accessed 2026-05-26, tier 1 (v6.6.0 confirmed latest)
- [Reintroducing @react-three/xr — pmnd.rs blog](https://pmnd.rs/blog/reintroducing-react-three-xr/) — accessed 2026-05-26, tier 1
