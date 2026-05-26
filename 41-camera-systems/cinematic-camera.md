# Cinematic Camera Systems in Three.js

## CatmullRomCurve3 — Camera Path Spline

`THREE.CatmullRomCurve3` creates a smooth spline through an array of control points. The curve is C1-continuous (smooth tangents) and passes through every control point, making it ideal for authored camera paths.

```js
import * as THREE from 'three';

const points = [
  new THREE.Vector3(-10,  2,  10),
  new THREE.Vector3(  0,  4,   5),
  new THREE.Vector3(  8,  2,   0),
  new THREE.Vector3(  5,  6,  -8),
  new THREE.Vector3( -5,  3,  -5),
];

const curve = new THREE.CatmullRomCurve3(
  points,
  false,         // closed loop?
  'catmullrom',  // type: 'centripetal' | 'chordal' | 'catmullrom'
  0.5            // tension (0 = sharp, 1 = loose; only for 'catmullrom' type)
);
```

**Type notes:**
- `'centripetal'` — avoids cusps and self-intersections; best for general camera paths
- `'chordal'` — distributes points by arc length, feels slower on tight curves
- `'catmullrom'` — classic; can overshoot on sharp angle changes

---

## getPointAt(t) and getTangentAt(t)

`t` is a normalized arc-length parameter in `[0, 1]`. `getPointAt` is arc-length parameterized (uniform speed along the curve); `getPoint` is parameterized by control point spacing (non-uniform speed).

**Always prefer `getPointAt` / `getTangentAt` over `getPoint` / `getTangent` for camera animation** — non-uniform parameterization creates apparent acceleration at dense control points.

```js
const t = 0.35; // 35% along the path

const position = curve.getPointAt(t);       // THREE.Vector3
const tangent  = curve.getTangentAt(t);     // THREE.Vector3, normalized direction of travel
const lookTarget = position.clone().add(tangent.multiplyScalar(5));

camera.position.copy(position);
camera.lookAt(lookTarget); // look along the direction of travel
```

### Generating a lookup table for performance

Computing arc-length parameterization on every frame is expensive. Pre-build a lookup table:

```js
// Build once
curve.updateArcLengths(); // must call before getPointAt works correctly

// Or cache all points at a fixed resolution
const cameraPathPoints = curve.getSpacedPoints(200); // 200 evenly spaced points
```

---

## Separate Look-At Spline

The camera position path and the point the camera looks at are often different splines. This gives the camera director independent control over framing.

```js
const lookAtCurve = new THREE.CatmullRomCurve3([
  new THREE.Vector3(0, 1, 0),
  new THREE.Vector3(3, 2, -2),
  new THREE.Vector3(6, 1, -6),
]);

function updateCinematicCamera(t) {
  camera.position.copy(positionCurve.getPointAt(t));
  camera.lookAt(lookAtCurve.getPointAt(t));
}
```

---

## Scroll-Driven Camera Animation

Map the user's scroll position to the `t` parameter. Use a smooth target with `MathUtils.damp` to prevent jitter from discrete scroll events.

```js
let scrollTarget = 0;
let scrollCurrent = 0;

window.addEventListener('wheel', (e) => {
  scrollTarget += e.deltaY * 0.0005; // scale to [0, 1] range
  scrollTarget = THREE.MathUtils.clamp(scrollTarget, 0, 1);
});

// In render loop:
function animate() {
  requestAnimationFrame(animate);
  const dt = clock.getDelta();

  // Smoothly approach scroll target
  scrollCurrent = THREE.MathUtils.damp(scrollCurrent, scrollTarget, 5, dt);

  camera.position.copy(positionCurve.getPointAt(scrollCurrent));
  camera.lookAt(lookAtCurve.getPointAt(scrollCurrent));

  renderer.render(scene, camera);
}
```

For touch/mobile, replace `wheel` with `touchmove` tracking vertical delta.

---

## Keyframe Camera Animation with THREE.AnimationMixer

For authored animations exported from Blender/Maya (glTF), the `AnimationMixer` drives camera transforms directly.

### glTF camera animation

```js
import { GLTFLoader } from 'three/examples/jsm/loaders/GLTFLoader.js';

const loader = new GLTFLoader();
loader.load('scene.glb', (gltf) => {
  // Find the camera track in animations
  const cameraClip = THREE.AnimationClip.findByName(gltf.animations, 'CameraFly');

  const mixer = new THREE.AnimationMixer(gltf.scene);
  const action = mixer.clipAction(cameraClip);
  action.play();

  // In render loop:
  // mixer.update(delta);
});
```

### Programmatic keyframe track

```js
// Position keyframes at times [0, 1, 2, 3] seconds
const times = [0, 1, 2, 3];
const positions = [
  0, 2, 10,   // t=0: (0, 2, 10)
  5, 4,  5,   // t=1
  8, 2,  0,   // t=2
  0, 1, -5,   // t=3
];

const posTrack = new THREE.VectorKeyframeTrack(
  '.position',      // property path on the target object
  times,
  positions,
  THREE.InterpolateSmooth  // CubicSpline-like | InterpolateLinear | InterpolateDiscrete
);

// Quaternion track for rotation
const rotTrack = new THREE.QuaternionKeyframeTrack(
  '.quaternion',
  [0, 1.5, 3],
  [
    ...new THREE.Quaternion().toArray(),       // identity
    ...new THREE.Quaternion().setFromEuler(
         new THREE.Euler(0, Math.PI / 4, 0)
       ).toArray(),
    ...new THREE.Quaternion().toArray(),
  ]
);

const clip = new THREE.AnimationClip('CameraMove', 3, [posTrack, rotTrack]);
const mixer = new THREE.AnimationMixer(camera);
const action = mixer.clipAction(clip);
action.play();
```

`THREE.InterpolateSmooth` uses a Catmull-Rom spline internally — matches curve-based paths.

### Render loop integration

```js
const clock = new THREE.Clock();

function animate() {
  requestAnimationFrame(animate);
  mixer.update(clock.getDelta());
  renderer.render(scene, camera);
}
```

---

## Depth of Field Linked to Camera Focus Distance

Three.js does not ship a built-in DoF post-process, but it integrates with `postprocessing` (vanruesc) or `three/addons BokehPass`.

### BokehPass (built-in addon)

```js
import { EffectComposer } from 'three/addons/postprocessing/EffectComposer.js';
import { RenderPass }     from 'three/addons/postprocessing/RenderPass.js';
import { BokehPass }      from 'three/addons/postprocessing/BokehPass.js';

const composer = new EffectComposer(renderer);
composer.addPass(new RenderPass(scene, camera));

const bokeh = new BokehPass(scene, camera, {
  focus:    5.0,   // world-space focus distance from camera
  aperture: 0.025, // controls depth of field range (smaller = more blurry)
  maxblur:  0.01,  // max blur kernel radius in NDC
});
composer.addPass(bokeh);
```

### Animate focus to follow a target (rack focus)

```js
let focusTarget = 5;
let focusCurrent = 5;

function rackFocusTo(worldPoint) {
  // Compute distance from camera to point
  focusTarget = camera.position.distanceTo(worldPoint);
}

// In render loop:
focusCurrent = THREE.MathUtils.damp(focusCurrent, focusTarget, 4, delta);
bokeh.uniforms['focus'].value = focusCurrent;

composer.render();
```

### Linking focus to CatmullRom look-at point

```js
function updateCinematicCamera(t) {
  const camPos  = positionCurve.getPointAt(t);
  const lookPos = lookAtCurve.getPointAt(t);

  camera.position.copy(camPos);
  camera.lookAt(lookPos);

  // Keep DoF focused on the look-at point
  rackFocusTo(lookPos);
}
```

---

## Visualizing the Curve (Debug)

```js
const points500 = curve.getPoints(500);
const geometry  = new THREE.BufferGeometry().setFromPoints(points500);
const material  = new THREE.LineBasicMaterial({ color: 0xff0000 });
const line      = new THREE.Line(geometry, material);
scene.add(line);
```

Remove or hide this object in production builds.

---

## Notes

- `curve.getLength()` returns the arc length in world units — useful for computing total duration at a constant world-speed.
- For loops, set `closed: true` in the constructor. The tangent at `t=0` and `t=1` will be matched automatically.
- `THREE.AnimationMixer` operates in object-local space. If the camera is a child of another object, its keyframe values must be in that parent's local space.
- The `BokehPass` is a screen-space approximation. For higher fidelity, `CoC` + depth-aware blur from the `postprocessing` library gives better results at higher cost.

Source: Three.js documentation and source (r165+). Accessed 2026-05-26.
