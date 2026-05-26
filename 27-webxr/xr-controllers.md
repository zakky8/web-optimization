# XR Controllers in Three.js

**Last verified:** 2026-05-26  
**Sources:** Three.js WebXR controller articles (markhorgan.com/blog/getting-started-with-webxr-and-threejs), Three.js forum posts, timmykokke.com haptics article, Three.js WebXRManager docs

---

## Controller Spaces: Ray vs Grip

Three.js exposes each physical controller as two separate `THREE.Group` objects:

| Space | Method | Purpose |
|-------|--------|---------|
| **Ray space** | `renderer.xr.getController(index)` | Pointing direction — origin at controller, Z-axis pointing forward along the "laser" |
| **Grip space** | `renderer.xr.getControllerGrip(index)` | Physical grip — position and orientation where the controller model sits in the user's hand |

Use **ray space** for raycasting/interaction. Use **grip space** to attach rendered controller models or objects held in the hand.

```js
const controller0 = renderer.xr.getController(0); // right (typically)
const controller1 = renderer.xr.getController(1); // left (typically)

const grip0 = renderer.xr.getControllerGrip(0);
const grip1 = renderer.xr.getControllerGrip(1);

scene.add(controller0, controller1, grip0, grip1);
```

**Index 0 / 1 do not reliably map to right/left.** Use `controller.inputSource.handedness` (`'left'` | `'right'` | `'none'`) inside a `connected` event handler to identify which hand connected.

---

## XRControllerModelFactory

`XRControllerModelFactory` loads the correct 3D model for the connected hardware using the [WebXR Input Profiles](https://github.com/immersive-web/webxr-input-profiles) library maintained by the W3C Immersive Web Working Group. Models have animated buttons and joysticks that reflect actual hardware.

```js
import { XRControllerModelFactory } from 'three/addons/webxr/XRControllerModelFactory.js';

const controllerModelFactory = new XRControllerModelFactory();

// Attach the model to GRIP space (not ray space)
const grip0 = renderer.xr.getControllerGrip(0);
grip0.add(controllerModelFactory.createControllerModel(grip0));
scene.add(grip0);

const grip1 = renderer.xr.getControllerGrip(1);
grip1.add(controllerModelFactory.createControllerModel(grip1));
scene.add(grip1);
```

The factory fetches model/profile JSON from a CDN (`cdn.jsdelivr.net/npm/@webxr-input-profiles/assets`) when the controller connects. Pass a custom `path` to the constructor to self-host:

```js
const factory = new XRControllerModelFactory(null, '/assets/xr-profiles/');
```

---

## Visual Ray Line

A common pattern is to add a visible pointing ray to the ray-space controller:

```js
import * as THREE from 'three';

const geometry = new THREE.BufferGeometry().setFromPoints([
  new THREE.Vector3(0, 0, 0),
  new THREE.Vector3(0, 0, -1),
]);
const line = new THREE.Line(geometry);
line.scale.z = 5; // ray length in metres

const controller0 = renderer.xr.getController(0);
controller0.add(line.clone());
scene.add(controller0);
```

---

## Controller Events

Listen for events on the **ray-space** controller (returned by `getController`).

### Primary Action (Trigger)

```js
controller0.addEventListener('selectstart', onSelectStart);
controller0.addEventListener('selectend',   onSelectEnd);
controller0.addEventListener('select',      onSelect);     // fires after selectend
```

### Squeeze (Grip Button)

```js
controller0.addEventListener('squeezestart', onSqueezeStart);
controller0.addEventListener('squeezeend',   onSqueezeEnd);
controller0.addEventListener('squeeze',      onSqueeze);
```

### Connection Lifecycle

```js
controller0.addEventListener('connected', (event) => {
  const inputSource = event.data; // XRInputSource
  console.log('handedness:', inputSource.handedness); // 'left' | 'right' | 'none'
  console.log('profiles:', inputSource.profiles);     // e.g. ['oculus-touch-v3', 'generic-trigger-squeeze']
});

controller0.addEventListener('disconnected', (event) => {
  // clean up anything attached to this controller
});
```

### Tracking State

Controllers can lose tracking. Access `inputSource.gripSpace` only after `connected` fires. Missing tracking shows `null` pose — do not crash if pose queries return null.

---

## Example: Toggle Object Color on Select

```js
const box = new THREE.Mesh(
  new THREE.BoxGeometry(0.2, 0.2, 0.2),
  new THREE.MeshStandardMaterial({ color: 0xff0000 })
);
box.position.set(0, 1.4, -0.5);
scene.add(box);

function onSelectStart() {
  box.material.color.set(0x00ff00);
}
function onSelectEnd() {
  box.material.color.set(0xff0000);
}

[controller0, controller1].forEach(ctrl => {
  ctrl.addEventListener('selectstart', onSelectStart);
  ctrl.addEventListener('selectend',   onSelectEnd);
});
```

---

## Raycasting with Controllers

```js
const raycaster = new THREE.Raycaster();
const tempMatrix = new THREE.Matrix4();
const objects = [box]; // meshes to test against

function checkIntersections(controller) {
  tempMatrix.identity().extractRotation(controller.matrixWorld);
  raycaster.ray.origin.setFromMatrixPosition(controller.matrixWorld);
  raycaster.ray.direction.set(0, 0, -1).applyMatrix4(tempMatrix);

  const intersects = raycaster.intersectObjects(objects);
  if (intersects.length > 0) {
    intersects[0].object.material.emissive.set(0x555555);
  }
}

// call inside animation loop
renderer.setAnimationLoop(() => {
  checkIntersections(controller0);
  checkIntersections(controller1);
  renderer.render(scene, camera);
});
```

---

## Haptic Feedback

The WebXR Device API itself has no haptics API. Haptics are accessed through the **Gamepad API**, reachable via the `XRInputSource`. Support is experimental but works in Meta Quest Browser.

```js
controller0.addEventListener('selectstart', (event) => {
  // event is an XRInputSourceEvent; event.data is the XRInputSource
  // However, Three.js controller events don't expose inputSource directly.
  // Use the session's inputSources instead:
  const session = renderer.xr.getSession();
  for (const source of session.inputSources) {
    if (source.handedness === 'right') {
      triggerHaptic(source.gamepad, 0.5, 100);
    }
  }
});

function triggerHaptic(gamepad, strength, durationMs) {
  if (!gamepad || !gamepad.hapticActuators || gamepad.hapticActuators.length === 0) {
    return;
  }
  // pulse(strength: 0–1, durationMs: milliseconds)
  gamepad.hapticActuators[0].pulse(strength, durationMs);
}
```

**Caveats:**
- `gamepad.hapticActuators` may be empty on non-haptic devices — always guard.
- `pulse()` is part of the Gamepad Haptics extension, not the base Gamepad API spec. Check before calling.
- The duration is in **milliseconds**; strength is **0–1** (0 = off, 1 = maximum).
- Not supported on PICO Browser or older Quest firmware.

---

## Hand Tracking

Three.js also exposes hand tracking via `renderer.xr.getHand(index)` which returns a `THREE.Group` with 25 joint bones. See `XRHandModelFactory` (same import path pattern) for model loading. Hand tracking is separate from controller tracking — hardware switches between modes.

---

## Sources

- [Getting started with WebXR and Three.js — Mark Horgan](https://markhorgan.com/blog/getting-started-with-webxr-and-threejs/) — accessed 2026-05-26, tier 1 (official example walkthrough)
- [VR Controller Haptics in WebXR — Timmy Kokke](https://timmykokke.com/blog/2022/2022-03-14-controller-haptics-in-webxr/) — accessed 2026-05-26, tier 1 (primary implementation source)
- [Three.js WebXRManager docs](https://threejs.org/docs/#api/en/renderers/webxr/WebXRManager) — accessed 2026-05-26, tier 1
- [Three.js forum: controller events discussion](https://discourse.threejs.org/t/webxr-controller-events-selectstart-selectend-fired-at-same-time/12064) — accessed 2026-05-26, tier 4
