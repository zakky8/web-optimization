# THREE.OrthographicCamera

## Constructor and Frustum Parameters

```js
const camera = new THREE.OrthographicCamera(
  left,   // left plane of the frustum
  right,  // right plane
  top,    // top plane
  bottom, // bottom plane
  near,   // near clipping plane (default 0.1)
  far     // far clipping plane (default 2000)
);
```

All six values are in **world units**. Unlike `PerspectiveCamera`, there is no FOV — the projection is purely parallel. Objects at any depth appear the same size.

### Standard setup: frustum driven by aspect ratio and a frustum size

```js
const frustumSize = 10; // half-height of the visible area in world units
const aspect = window.innerWidth / window.innerHeight;

const camera = new THREE.OrthographicCamera(
  (frustumSize * aspect) / -2,  // left
  (frustumSize * aspect) /  2,  // right
   frustumSize            /  2,  // top
   frustumSize            / -2,  // bottom
  0.1,                           // near
  1000                           // far
);
camera.position.set(0, 0, 10);
camera.lookAt(0, 0, 0);
```

`frustumSize` defines the total visible height. To show a 10-unit-tall slice of the world, set `frustumSize = 10`.

---

## Frustum Size Update on Resize

On window resize you must recalculate the frustum planes and call `camera.updateProjectionMatrix()`. Without `updateProjectionMatrix()` the change has no effect.

```js
function onWindowResize() {
  const aspect = window.innerWidth / window.innerHeight;

  camera.left   = (frustumSize * aspect) / -2;
  camera.right  = (frustumSize * aspect) /  2;
  camera.top    =  frustumSize /  2;
  camera.bottom =  frustumSize / -2;

  camera.updateProjectionMatrix();

  renderer.setSize(window.innerWidth, window.innerHeight);
  renderer.setPixelRatio(window.devicePixelRatio);
}

window.addEventListener('resize', onWindowResize);
```

**Common mistake:** updating the aspect on a `PerspectiveCamera` via `camera.aspect = newAspect` without calling `updateProjectionMatrix()` also silently breaks the projection. The call is required for both camera types whenever any frustum property changes.

---

## Pixel-Perfect 2D Overlay in a 3D Scene

For UI elements, HUD sprites, or 2D canvas text rendered as textures, you want orthographic "pixel coordinates" where 1 world unit = 1 screen pixel.

```js
const overlayCamera = new THREE.OrthographicCamera(
  0,                        // left  = screen left edge in pixels
  window.innerWidth,        // right = screen right edge
  0,                        // top   = screen top edge
  -window.innerHeight,      // bottom = screen bottom (negative for standard Y-down)
  -1,                       // near  (can be negative for flat 2D objects at z=0)
  1                         // far
);
// No position needed — orthographic, objects at z=0 are rendered directly

// Update on resize:
function onResizeOverlay() {
  overlayCamera.right  =  window.innerWidth;
  overlayCamera.bottom = -window.innerHeight;
  overlayCamera.updateProjectionMatrix();
}
```

Place 2D meshes at `z = 0` with pixel-coordinate `x`/`y` positions:

```js
const planeGeo = new THREE.PlaneGeometry(200, 50); // 200×50 px sprite
const planeMat = new THREE.MeshBasicMaterial({ map: uiTexture, transparent: true });
const uiMesh   = new THREE.Mesh(planeGeo, planeMat);

uiMesh.position.set(100, -25, 0); // top-left corner at (0, 0) with this setup
overlayScene.add(uiMesh);
```

---

## Mixing Perspective + Orthographic in the Same Frame

The canonical technique uses two `Scene` objects and renders them sequentially, clearing depth between passes.

```js
const mainScene    = new THREE.Scene();
const overlayScene = new THREE.Scene();

const perspCamera  = new THREE.PerspectiveCamera(60, aspect, 0.1, 1000);
const orthoCamera  = new THREE.OrthographicCamera(
  0, window.innerWidth, 0, -window.innerHeight, -1, 1
);

function render() {
  // 1. Render the 3D perspective scene normally
  renderer.clear(); // clears color + depth
  renderer.render(mainScene, perspCamera);

  // 2. Render the 2D overlay on top, WITHOUT clearing color buffer
  //    but clearing depth so 2D elements always appear on top
  renderer.clearDepth();
  renderer.render(overlayScene, orthoCamera);
}

// Enable manual clear control
renderer.autoClear = false;
```

**`renderer.autoClear = false`** is critical — it prevents the renderer from auto-clearing before each `render()` call.

### Alternative: autoClearColor / autoClearDepth

```js
renderer.autoClear      = false;
renderer.autoClearColor = false; // preserve 3D render in color buffer
renderer.autoClearDepth = true;  // clear depth before each render() call
                                  // This means you must call renderer.clear() yourself for pass 1
```

Most implementations prefer the explicit `renderer.clearDepth()` pattern above for clarity.

---

## OrthographicCamera for Isometric / Top-Down Views

For isometric views, rotate the camera — do not manipulate the frustum:

```js
// 45° around Y, then tilted down ~35.26° for true isometric
const ISO_ANGLE = Math.atan(1 / Math.sqrt(2)); // ≈ 35.26°

camera.position.set(10, 10, 10);
camera.lookAt(0, 0, 0);
// Three.js handles the rotation matrix automatically from lookAt
```

For tile-based isometric where you need exact pixel snapping, ensure the frustum width/height maps to an integer number of pixels and set `renderer.setPixelRatio(1)`.

---

## zoom Property vs Frustum Scaling

`OrthographicCamera` has a `zoom` property (default `1`) that scales the frustum without changing the frustum plane values:

```js
camera.zoom = 2;   // effectively halves the visible area (zoom in)
camera.zoom = 0.5; // doubles the visible area (zoom out)
camera.updateProjectionMatrix(); // required
```

This is useful when you want to animate zoom independently of frustum setup. Note that `camera-controls` (yomotsu) drives `camera.zoom` for orthographic cameras when `dolly`/`zoom` methods are called.

---

## Frustum Culling and OrthographicCamera

Frustum culling works correctly with `OrthographicCamera` — objects outside the parallel frustum volume are not rendered. The camera's `left`/`right`/`top`/`bottom` are the exact clip planes in view space, so ensure they are not set too tight if you have objects at the edges of screen space.

---

## Shadows with OrthographicCamera

`DirectionalLight` uses an internal `OrthographicCamera` to render its shadow map. The same frustum parameters apply:

```js
const dirLight = new THREE.DirectionalLight(0xffffff, 1);
dirLight.castShadow = true;

// Shadow camera frustum (these are world units, not NDC)
dirLight.shadow.camera.left   = -20;
dirLight.shadow.camera.right  =  20;
dirLight.shadow.camera.top    =  20;
dirLight.shadow.camera.bottom = -20;
dirLight.shadow.camera.near   = 0.5;
dirLight.shadow.camera.far    = 100;
dirLight.shadow.camera.updateProjectionMatrix();

// Shadow map resolution
dirLight.shadow.mapSize.set(2048, 2048);

// Visualize the shadow camera frustum (debug)
const helper = new THREE.CameraHelper(dirLight.shadow.camera);
scene.add(helper);
```

Tight frustum bounds improve shadow quality (more shadow map texels per world unit). Set the bounds to just encompass the scene, not the entire world.

---

## Quick Reference

| Task | Code |
|---|---|
| Create ortho camera | `new THREE.OrthographicCamera(L, R, T, B, near, far)` |
| Pixel-perfect overlay | L=0, R=width, T=0, B=-height, near=-1, far=1 |
| Update after resize | Set new L/R/T/B, then `camera.updateProjectionMatrix()` |
| Zoom in/out | `camera.zoom = v; camera.updateProjectionMatrix()` |
| Dual-scene overlay | `renderer.autoClear=false`, render main, `clearDepth()`, render overlay |
| Debug frustum | `new THREE.CameraHelper(camera)` |

Source: Three.js r165+ documentation and source. Accessed 2026-05-26.
