# WebXR Setup with Three.js

**Last verified:** 2026-05-26  
**Sources:** Three.js manual (threejs.org/manual/en/webxr-basics.html), WebXRManager docs (threejs.org/docs/#api/en/renderers/webxr/WebXRManager), MDN WebXR Device API

---

## What WebXR Is

The WebXR Device API provides access to VR and AR hardware from the browser. Three.js wraps this API in its `WebXRManager` class, exposed via `renderer.xr`. You do not call `navigator.xr` directly in a standard Three.js app — Three.js handles session creation, frame loops, and camera management for you.

---

## 1. Enable WebXR in the Renderer

```js
import * as THREE from 'three';

const renderer = new THREE.WebGLRenderer({ antialias: true });
renderer.setPixelRatio(window.devicePixelRatio);
renderer.setSize(window.innerWidth, window.innerHeight);
document.body.appendChild(renderer.domElement);

// Required: tell Three.js to hand frame control to the XR system
renderer.xr.enabled = true;
```

Without `renderer.xr.enabled = true` the VR/AR buttons will fail silently and `setAnimationLoop` will not receive `XRFrame` data.

---

## 2. VRButton — Immersive VR

```js
import { VRButton } from 'three/addons/webxr/VRButton.js';

document.body.appendChild(VRButton.createButton(renderer));
```

`VRButton.createButton(renderer)` returns a DOM button element. When the user clicks it, it calls `navigator.xr.requestSession('immersive-vr')` and wires the session to the renderer. The button is grayed out automatically on browsers/devices that do not support `immersive-vr`.

**VR session features requested by default:** `local-floor` reference space.

---

## 3. ARButton — Immersive AR

```js
import { ARButton } from 'three/addons/webxr/ARButton.js';

const renderer = new THREE.WebGLRenderer({
  antialias: true,
  alpha: true,   // required: transparent background so camera feed shows through
});

document.body.appendChild(
  ARButton.createButton(renderer, {
    requiredFeatures: ['hit-test'],         // request hit testing
    optionalFeatures: ['dom-overlay'],      // HTML overlay over AR view
    domOverlay: { root: document.body },
  })
);
```

Key differences from VRButton:
- Renderer must be created with `alpha: true`.
- Session mode is `immersive-ar`.
- Optional and required WebXR features are passed as the second argument.

---

## 4. XRSession Types

| Type | Description | Typical Use |
|------|-------------|-------------|
| `immersive-vr` | Full VR headset, hides real world | Head-mounted displays (Quest, Vive, etc.) |
| `immersive-ar` | AR on phone screen or passthrough headset | Android Chrome, Quest 3 passthrough |
| `inline` | Renders in a standard `<canvas>`, no immersion | 360° previews, non-headset preview |

Requesting a session type that the device does not support throws a `NotSupportedError`. Always check support first:

```js
if (navigator.xr) {
  const supported = await navigator.xr.isSessionSupported('immersive-vr');
  // true | false
}
```

---

## 5. setAnimationLoop with XRFrame

**Do not use `requestAnimationFrame` for XR apps.** The browser's standard rAF loop is paused when an XR session is active. Use `renderer.setAnimationLoop` instead — Three.js calls this correctly in both XR and non-XR contexts.

```js
renderer.setAnimationLoop(animate);

function animate(timestamp, frame) {
  // `timestamp` — DOMHighResTimeStamp (milliseconds), same as rAF
  // `frame`     — XRFrame | undefined
  //               XRFrame is present only when an XR session is active;
  //               undefined during normal (non-XR) rendering

  if (frame) {
    // XR-specific work, e.g. hit testing, pose queries
    const referenceSpace = renderer.xr.getReferenceSpace();
    const session = renderer.xr.getSession();
    // frame.getHitTestResults(...), frame.getPose(...), etc.
  }

  renderer.render(scene, camera);
}
```

The `XRFrame` object provides:
- `frame.getPose(space, referenceSpace)` — position/orientation of an XR space
- `frame.getHitTestResults(hitTestSource)` — AR surface intersections
- `frame.getViewerPose(referenceSpace)` — viewer head pose

---

## 6. WebXRManager API (renderer.xr)

```js
renderer.xr.enabled          // boolean — must be true before session starts
renderer.xr.isPresenting      // boolean — true when an XR session is active

renderer.xr.getSession()      // returns XRSession | null
renderer.xr.getReferenceSpace() // returns XRReferenceSpace | null

renderer.xr.setReferenceSpaceType(type)
// type: 'viewer' | 'local' | 'local-floor' | 'bounded-floor' | 'unbounded'
// default: 'local-floor'
// Must be called BEFORE session starts

renderer.xr.getController(index)     // returns THREE.Group — ray space
renderer.xr.getControllerGrip(index) // returns THREE.Group — grip space
renderer.xr.getHand(index)           // returns THREE.Group — hand tracking

// Events on renderer.xr
renderer.xr.addEventListener('sessionstart', () => { ... })
renderer.xr.addEventListener('sessionend',   () => { ... })
```

---

## 7. Camera in VR

In VR, Three.js automatically replaces the camera with an `ArrayCamera` (one sub-camera per eye). Do not manually set camera position inside the render loop when in VR — the XR system provides head-tracked poses. Set an initial standing position before the session starts:

```js
camera.position.set(0, 1.6, 3); // 1.6 m = average eye height standing
```

---

## 8. Minimal Full Example (VR)

```js
import * as THREE from 'three';
import { VRButton } from 'three/addons/webxr/VRButton.js';

const scene    = new THREE.Scene();
const camera   = new THREE.PerspectiveCamera(70, window.innerWidth / window.innerHeight, 0.1, 100);
camera.position.set(0, 1.6, 3);

const renderer = new THREE.WebGLRenderer({ antialias: true });
renderer.setSize(window.innerWidth, window.innerHeight);
renderer.xr.enabled = true;
document.body.appendChild(renderer.domElement);
document.body.appendChild(VRButton.createButton(renderer));

// simple scene content
const box = new THREE.Mesh(
  new THREE.BoxGeometry(0.5, 0.5, 0.5),
  new THREE.MeshStandardMaterial({ color: 0x4488ff })
);
box.position.set(0, 1.6, -2);
scene.add(box);
scene.add(new THREE.AmbientLight(0xffffff, 1));

renderer.setAnimationLoop((time, frame) => {
  box.rotation.y = time * 0.001;
  renderer.render(scene, camera);
});

window.addEventListener('resize', () => {
  camera.aspect = window.innerWidth / window.innerHeight;
  camera.updateProjectionMatrix();
  renderer.setSize(window.innerWidth, window.innerHeight);
});
```

---

## Sources

- [Three.js WebXR Basics manual](https://threejs.org/manual/en/webxr-basics.html) — accessed 2026-05-26, tier 1
- [Three.js WebXRManager docs](https://threejs.org/docs/#api/en/renderers/webxr/WebXRManager) — accessed 2026-05-26, tier 1
- [MDN WebXR Device API](https://developer.mozilla.org/en-US/docs/Web/API/WebXR_Device_API) — accessed 2026-05-26, tier 1
