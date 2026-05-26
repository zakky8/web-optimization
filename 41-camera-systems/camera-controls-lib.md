# camera-controls Library (yomotsu)

## Source Verification

| Field | Value |
|---|---|
| Repository | https://github.com/yomotsu/camera-controls |
| Current version | **v3.1.2** (released 2025-11-17) |
| License | MIT |
| Stars | ~2,401 |
| Last push | 2026-02-02 |
| Open issues | 91 |
| Default branch | `dev` |
| Topics | camera, orbitcontrols, threejs |

Source accessed: 2026-05-26. Archive: https://web.archive.org/web/20260526/https://github.com/yomotsu/camera-controls

---

## Why camera-controls vs THREE.OrbitControls

`THREE.OrbitControls` ships in the Three.js examples folder — it is intentionally minimal and carries no stability guarantee. `camera-controls` is a purpose-built, independently maintained library with a stable npm release cycle.

### Feature comparison

| Feature | OrbitControls | camera-controls v3 |
|---|---|---|
| Smooth transitions (lerp/spring) | None — snaps instantly | `smoothTime`, `draggingSmoothTime` |
| Fit to bounding box | No | `fitToBox()` |
| Fit to bounding sphere | No | `fitToSphere()` |
| Collision detection | No | `colliderMeshes` + raycasters |
| Promise-based animation | No | All transition methods return Promises |
| Dolly-to-cursor | No | `dollyToCursor = true` |
| Boundary enforcement | Basic polar limits | Full box boundaries + polar + azimuth |
| Orthographic support | Partial | Full `zoom` path for ortho cams |
| Bundle-tree-shakeable install | N/A | Subset install supported |
| Configurable action map | No | Per-mouse-button + per-touch-count |

---

## Installation

```bash
npm install camera-controls
```

### Required bootstrap — `CameraControls.install({ THREE })`

camera-controls does **not** import Three.js internally. You must inject the Three.js namespace before instantiating any `CameraControls`. This keeps it version-agnostic and avoids bundling Two copies of Three.js.

```js
import * as THREE from 'three';
import CameraControls from 'camera-controls';

// Must be called once, before any new CameraControls(...)
CameraControls.install({ THREE: THREE });

const camera = new THREE.PerspectiveCamera(60, aspect, 0.01, 1000);
const controls = new CameraControls(camera, renderer.domElement);
```

**Subset install** (smaller bundle — only provide what you actually use):

```js
import {
  Vector2, Vector3, Vector4, Quaternion, Matrix4,
  Spherical, Box3, Sphere, Raycaster, MathUtils,
} from 'three';

const subsetOfTHREE = {
  Vector2, Vector3, Vector4, Quaternion, Matrix4,
  Spherical, Box3, Sphere, Raycaster, MathUtils,
};
CameraControls.install({ THREE: subsetOfTHREE });
```

---

## Dolly vs. Zoom — the critical distinction

These two concepts are frequently confused. camera-controls makes the distinction explicit in its API and documentation.

| | Dolly | Zoom |
|---|---|---|
| What moves | The **camera position** — physically closer/farther along the view axis | **Nothing moves** — the lens FOV (perspective) or orthographic frustum size changes |
| Three.js mechanism | `camera.position` is updated | `camera.fov` (PerspectiveCamera) or `camera.zoom` (OrthographicCamera) is updated |
| Visual result | Objects maintain relative sizes; perspective distortion changes | Field of view narrows/widens; perspective distortion stays constant |
| API | `controls.dolly(delta)` / `controls.dollyTo(distance)` | `controls.zoom(delta)` / `controls.zoomTo(value)` |
| Constraints | `minDistance` / `maxDistance` | `minZoom` / `maxZoom` |

**When to use which:**
- Dolly = cinematic camera movement, game camera, walk-through — parallax is part of the experience.
- Zoom = map view, product configurator, 2D-ish top-down — you want to crop in without distorting depth relationships.

---

## fit / fitToSphere / fitToBox

These methods automatically position and orient the camera to frame an object. All return a `Promise<void>` that resolves when the transition completes.

```js
// Frame a bounding sphere
await controls.fitToSphere(mesh, true);  // (object, enableTransition)

// Frame a bounding box with padding
await controls.fitToBox(mesh, true, {
  paddingTop:    0.1,
  paddingBottom: 0.1,
  paddingLeft:   0.1,
  paddingRight:  0.1,
});
```

Internally these compute the required camera distance from the target's angular size at the current FOV, then call `setLookAt()` with the result.

---

## lerpFactor / smoothTime

camera-controls uses a **critically-damped spring** approach (not a fixed lerp per frame) to avoid frame-rate-dependent easing.

```js
controls.smoothTime         = 0.25; // seconds to reach target (default)
controls.draggingSmoothTime = 0.125; // while user is dragging
```

`smoothTime` maps to a spring damping coefficient. At 60 fps a value of `0.25` gives roughly 4-5 frames of visible lag on a short move. Lower = snappier.

**Do not** multiply by `delta` yourself; camera-controls handles the delta internally in its `update(delta)` call:

```js
// In your render loop:
const delta = clock.getDelta();
const updated = controls.update(delta); // returns true if camera moved
if (updated) renderer.render(scene, camera);
```

---

## Collision Detection

```js
controls.colliderMeshes = [floorMesh, wallMesh, ceilingMesh];
```

Internally camera-controls casts rays from the target toward the camera. If a collider is hit, the camera is pulled closer. This prevents geometry clipping but is **not** a physics simulation — it prevents the camera from entering a mesh, but the target point can still pass through geometry.

Performance note: raycasts fire on every frame when `colliderMeshes` is non-empty and the camera is moving. Keep the collider array small; use simplified proxy meshes where possible.

---

## Action Map (mouse / touch configuration)

```js
import { ACTION } from 'camera-controls';

controls.mouseButtons.left   = ACTION.ROTATE;
controls.mouseButtons.right  = ACTION.TRUCK;
controls.mouseButtons.middle = ACTION.DOLLY;
controls.mouseButtons.wheel  = ACTION.DOLLY;

controls.touches.one   = ACTION.TOUCH_ROTATE;
controls.touches.two   = ACTION.TOUCH_DOLLY_TRUCK;
controls.touches.three = ACTION.TOUCH_TRUCK;
```

Available actions: `ROTATE`, `TRUCK`, `OFFSET`, `DOLLY`, `ZOOM`, `SCREEN_PAN`, `NONE` (and touch equivalents).

---

## Event System

```js
controls.addEventListener('controlstart', () => { /* user grabbed */ });
controls.addEventListener('control',      () => { /* mid-drag */ });
controls.addEventListener('controlend',   () => { /* released */ });
controls.addEventListener('transitionstart', () => { /* programmatic move started */ });
controls.addEventListener('update',       () => { /* camera matrix updated */ });
controls.addEventListener('wake',         () => { /* camera started moving */ });
controls.addEventListener('rest',         () => { /* transition settled */ });
controls.addEventListener('sleep',        () => { /* fully stopped */ });
```

`rest` fires when velocity is below threshold but the user may still be holding. `sleep` fires when both velocity and user input are zero — safe to pause the render loop.

---

## Key Properties Reference

```js
controls.enabled            = true;
controls.smoothTime         = 0.25;
controls.draggingSmoothTime = 0.125;
controls.minDistance        = 1;
controls.maxDistance        = Infinity;
controls.minZoom            = 0.01;
controls.maxZoom            = Infinity;
controls.minPolarAngle      = 0;                // radians from top
controls.maxPolarAngle      = Math.PI;          // radians from top
controls.minAzimuthAngle    = -Infinity;
controls.maxAzimuthAngle    = Infinity;
controls.dollyToCursor      = false;
controls.infinityDolly      = false;
controls.colliderMeshes     = [];
```

---

## Notes

- The `dev` branch is the active development branch; releases are tagged from it.
- v3.x introduced the Promise-based transition API; v2.x used callbacks.
- TypeScript types are bundled; no `@types/` package needed.
- Works with both `PerspectiveCamera` and `OrthographicCamera`; zoom path auto-detects camera type.
