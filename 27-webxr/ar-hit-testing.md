# AR Hit Testing with WebXR and Three.js

**Last verified:** 2026-05-26  
**Sources:** Google ARCore WebXR guide (developers.google.com/ar/develop/webxr/hello-webxr), MDN XRSession.requestHitTestSource, W3C WebXR Hit Test Module spec (w3.org/TR/webxr-hit-test-1), Three.js AR hit test example (threejs.org/examples/webxr_ar_hittest.html)

---

## What Hit Testing Is

Hit testing answers the question: "Where does a ray from this position intersect a real-world surface?" The result is a 6DOF pose (position + orientation) on a detected plane, point cloud feature, or mesh. You use this to place virtual objects convincingly on floors, tables, and walls.

---

## Prerequisites

1. An `immersive-ar` session — hit testing is not available in VR.
2. The `hit-test` feature must be declared as **required** when requesting the session.
3. `renderer.xr.enabled = true` and `alpha: true` on the renderer.

```js
import * as THREE from 'three';
import { ARButton } from 'three/addons/webxr/ARButton.js';

const renderer = new THREE.WebGLRenderer({ antialias: true, alpha: true });
renderer.xr.enabled = true;

document.body.appendChild(
  ARButton.createButton(renderer, {
    requiredFeatures: ['hit-test'],
  })
);
```

---

## Session and Reference Spaces

Hit testing requires two reference spaces:

| Space | Purpose |
|-------|---------|
| `local` | Scene coordinate space; used to resolve hit poses into world positions |
| `viewer` | Represents the user's head/phone camera; used as the ray origin |

```js
let localSpace;
let viewerSpace;
let hitTestSource = null;

renderer.xr.addEventListener('sessionstart', async () => {
  const session = renderer.xr.getSession();

  // Space for resolving hit results into world coordinates
  localSpace  = await session.requestReferenceSpace('local');
  // Space for casting the hit-test ray from the viewer/camera
  viewerSpace = await session.requestReferenceSpace('viewer');

  // Create the hit test source — this is asynchronous
  hitTestSource = await session.requestHitTestSource({ space: viewerSpace });
});

renderer.xr.addEventListener('sessionend', () => {
  hitTestSource = null;
});
```

**`requestHitTestSource` signature (MDN verified):**

```js
session.requestHitTestSource({
  space: XRSpace,          // required — origin of the test ray
  entityTypes: ['plane'],  // optional — 'point' | 'plane' | 'mesh', default: ['plane']
  offsetRay: new XRRay(),  // optional — custom ray direction/origin offset
});
// returns Promise<XRHitTestSource>
```

Throws `NotSupportedError` if `hit-test` was not in `requiredFeatures`.

---

## The Reticle

A reticle is a visual indicator (typically a flat ring) that tracks the current hit position in real time. It should be invisible when there is no hit result.

```js
// Use a torus ring as reticle
const reticle = new THREE.Mesh(
  new THREE.RingGeometry(0.08, 0.10, 32).rotateX(-Math.PI / 2),
  new THREE.MeshBasicMaterial({ color: 0xffffff })
);
reticle.matrixAutoUpdate = false; // we set the matrix manually from the hit pose
reticle.visible = false;
scene.add(reticle);
```

The ring is rotated -90° on X so it lies flat on a horizontal surface. `matrixAutoUpdate = false` is important — we will write the matrix directly from the hit result transform.

---

## Updating the Reticle Each Frame

Inside the animation loop, query hit results from the current `XRFrame`:

```js
renderer.setAnimationLoop((timestamp, frame) => {
  if (frame && hitTestSource) {
    const hitTestResults = frame.getHitTestResults(hitTestSource);

    if (hitTestResults.length > 0) {
      const hit      = hitTestResults[0];
      const hitPose  = hit.getPose(localSpace);

      reticle.visible = true;
      reticle.matrix.fromArray(hitPose.transform.matrix);
      // Alternatively: decompose into position/quaternion:
      // reticle.position.setFromMatrixPosition(reticle.matrix);
      // reticle.quaternion.setFromRotationMatrix(reticle.matrix);
    } else {
      reticle.visible = false;
    }
  }

  renderer.render(scene, camera);
});
```

`hitTestResults[0]` is the closest intersection. `getPose(localSpace)` returns an `XRPose` whose `transform.matrix` is a column-major Float32Array of the 4×4 world transform.

---

## Placing Objects on Select

The `select` event fires when the user taps the screen (handheld AR) or pulls the trigger (head-mounted AR). Place a clone of your model at the reticle's current world position:

```js
const session = renderer.xr.getSession();

session.addEventListener('select', () => {
  if (!reticle.visible) return;

  const object = buildObject(); // your mesh/model
  // Copy the reticle's transform
  object.position.setFromMatrixPosition(reticle.matrix);
  object.quaternion.setFromRotationMatrix(reticle.matrix);
  scene.add(object);
});

function buildObject() {
  return new THREE.Mesh(
    new THREE.CylinderGeometry(0.05, 0.05, 0.1, 32),
    new THREE.MeshStandardMaterial({ color: 0xff6600 })
  );
}
```

---

## Entity Types

Control which real-world geometry the ray tests against:

```js
hitTestSource = await session.requestHitTestSource({
  space: viewerSpace,
  entityTypes: ['plane'],         // detected planes only (default)
  // entityTypes: ['point'],      // sparse point-cloud features
  // entityTypes: ['mesh'],       // full mesh (requires mesh-detection feature)
  // entityTypes: ['plane', 'point'], // multiple types
});
```

`'plane'` is the most reliable — floor and table detection is available on all supported AR platforms. `'mesh'` requires additional session features and is only available on Meta Quest 3+ and newer devices.

---

## Anchors (Persistent Placement)

If you need a placed object to stay fixed to a real-world location even as the user moves, use the Anchors API:

```js
// After placing at a hit pose:
const anchor = await frame.createAnchor(hitPose.transform, localSpace);
// anchor.anchorSpace updates each frame with the corrected pose
```

Requires the `anchors` feature in session options. This is currently supported on Chrome Android and Quest Browser; NOT on iOS.

---

## Complete Minimal Example

```js
import * as THREE from 'three';
import { ARButton } from 'three/addons/webxr/ARButton.js';

const scene    = new THREE.Scene();
const camera   = new THREE.PerspectiveCamera(70, innerWidth / innerHeight, 0.01, 20);
const renderer = new THREE.WebGLRenderer({ antialias: true, alpha: true });
renderer.setSize(innerWidth, innerHeight);
renderer.xr.enabled = true;
document.body.appendChild(renderer.domElement);
document.body.appendChild(ARButton.createButton(renderer, { requiredFeatures: ['hit-test'] }));
scene.add(new THREE.AmbientLight(0xffffff, 1.5));

// Reticle
const reticle = new THREE.Mesh(
  new THREE.RingGeometry(0.08, 0.10, 32).rotateX(-Math.PI / 2),
  new THREE.MeshBasicMaterial()
);
reticle.matrixAutoUpdate = false;
reticle.visible = false;
scene.add(reticle);

let localSpace, viewerSpace, hitTestSource = null;

renderer.xr.addEventListener('sessionstart', async () => {
  const session = renderer.xr.getSession();
  localSpace    = await session.requestReferenceSpace('local');
  viewerSpace   = await session.requestReferenceSpace('viewer');
  hitTestSource = await session.requestHitTestSource({ space: viewerSpace });
  session.addEventListener('select', placeObject);
});

renderer.xr.addEventListener('sessionend', () => { hitTestSource = null; });

function placeObject() {
  if (!reticle.visible) return;
  const mesh = new THREE.Mesh(
    new THREE.SphereGeometry(0.05, 16, 16),
    new THREE.MeshStandardMaterial({ color: 0x00aaff })
  );
  mesh.position.setFromMatrixPosition(reticle.matrix);
  scene.add(mesh);
}

renderer.setAnimationLoop((_, frame) => {
  if (frame && hitTestSource) {
    const results = frame.getHitTestResults(hitTestSource);
    if (results.length > 0) {
      reticle.visible = true;
      reticle.matrix.fromArray(results[0].getPose(localSpace).transform.matrix);
    } else {
      reticle.visible = false;
    }
  }
  renderer.render(scene, camera);
});
```

---

## Sources

- [Google ARCore WebXR Hello WebXR tutorial](https://developers.google.com/ar/develop/webxr/hello-webxr) — accessed 2026-05-26, tier 1
- [MDN XRSession.requestHitTestSource](https://developer.mozilla.org/en-US/docs/Web/API/XRSession/requestHitTestSource) — accessed 2026-05-26, tier 1
- [W3C WebXR Hit Test Module Level 1](https://www.w3.org/TR/webxr-hit-test-1/) — accessed 2026-05-26, tier 1
- [Three.js AR hit test example](https://threejs.org/examples/webxr_ar_hittest.html) — accessed 2026-05-26, tier 1
