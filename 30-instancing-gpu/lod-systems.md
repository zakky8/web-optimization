# LOD Systems in Three.js

> Sources: Three.js r184 `src/objects/LOD.js`, `webgl_lod.html` example, three.js forum "LOD + Instancing", three.js forum "InstancedMesh LOD — 1 million instances"
> Accessed: 2026-05-26

---

## 1. THREE.LOD — Core API

`THREE.LOD` is an `Object3D` subclass. It holds an array of meshes paired with distance thresholds and shows the appropriate mesh each frame based on camera distance.

### Constructor

```js
const lod = new THREE.LOD();
scene.add(lod);
lod.position.set(x, y, z);
```

No constructor parameters. All configuration is done via `addLevel`.

---

## 2. Properties

### `levels` — `Array<{object: Object3D, distance: number, hysteresis: number}>`

Direct reference to the internal levels array. Sorted by `distance` ascending after each `addLevel` call.

```js
console.log(lod.levels);
// [ {object: mesh0, distance: 0, hysteresis: 0},
//   {object: mesh1, distance: 300, hysteresis: 0}, ...]
```

### `autoUpdate` — `boolean` (default `true`)

When `true`, the renderer calls `update(camera)` automatically after each `renderer.render(scene, camera)`. When `false`, you must call it manually (useful for off-screen or deferred rendering).

---

## 3. `addLevel(object, distance, hysteresis)` → `this`

Registers a mesh at a camera-distance threshold. Three.js displays the mesh whose threshold is the largest value still ≤ the camera distance.

```js
const lod = new THREE.LOD();

// Level 0: highest detail — visible when camera ≤ 50 units away
lod.addLevel(new THREE.Mesh(new THREE.IcosahedronGeometry(100, 16), mat), 0);
// Level 1: medium detail
lod.addLevel(new THREE.Mesh(new THREE.IcosahedronGeometry(100, 8),  mat), 300);
// Level 2: low detail
lod.addLevel(new THREE.Mesh(new THREE.IcosahedronGeometry(100, 4),  mat), 1000);
// Level 3: very low
lod.addLevel(new THREE.Mesh(new THREE.IcosahedronGeometry(100, 2),  mat), 2000);
// Level 4: billboard (see §6)
lod.addLevel(billboardSprite, 8000);

scene.add(lod);
```

From the `webgl_lod.html` example (1000 LOD objects):
```js
const distances = [50, 300, 1000, 2000, 8000];
const subdivisions = [16, 8, 4, 2, 1];

for (let i = 0; i < distances.length; i++) {
  const geo = new THREE.IcosahedronGeometry(100, subdivisions[i]);
  const mesh = new THREE.Mesh(geo, mat);
  lod.addLevel(mesh, distances[i]);
}
```

### `hysteresis` parameter (r152+)

Optional float in range [0, 1). Prevents level-flickering when an object sits exactly on a distance threshold. When the current level `L` is already active, its effective distance threshold is reduced by `hysteresis * threshold`, making it "stickier":

```
effective_distance = threshold - (threshold * hysteresis)
```

```js
// Switch to low-poly at 500, but don't switch back to high-poly
// until camera is within 450 (10% hysteresis)
lod.addLevel(highPolyMesh, 0,   0);
lod.addLevel(lowPolyMesh,  500, 0.1);
```

---

## 4. `update(camera)` → `void`

Computes camera-to-LOD distance and sets visibility on all level objects. Called automatically each frame when `autoUpdate = true`.

```js
// Distance formula (from LOD.js source):
// distance = camera.position.distanceTo(lod.getWorldPosition(_v)) / camera.zoom
```

Division by `camera.zoom` ensures correct behaviour with orthographic cameras where zoom affects apparent size.

Manual update (when `autoUpdate = false`):
```js
lod.autoUpdate = false;

function animate() {
  requestAnimationFrame(animate);
  lod.update(camera);     // explicit call required
  renderer.render(scene, camera);
}
```

---

## 5. `getObjectForDistance(distance)` and `getCurrentLevel()`

```js
// What mesh would show at 750 units away?
const mesh = lod.getObjectForDistance(750);

// Current active level index (0-based)
const levelIndex = lod.getCurrentLevel();
console.log(lod.levels[levelIndex]);
```

`getObjectForDistance` is useful for pre-computing expected mesh before an LOD is in the scene.

---

## 6. Impostor Sprites for Distant Levels

An impostor is a billboard (camera-facing quad) textured with a pre-rendered image of the object. It replaces the geometry entirely at extreme distances where individual pixels are subpixel or a few pixels wide.

### Creating a Static Impostor

```js
// 1. Pre-render the object to a texture (offline or at startup)
const impostorRT = new THREE.WebGLRenderTarget(256, 256, {
  minFilter: THREE.LinearMipmapLinearFilter,
  magFilter: THREE.LinearFilter,
  format: THREE.RGBAFormat,
});
renderer.setRenderTarget(impostorRT);
renderer.render(impostorScene, impostorCamera);
renderer.setRenderTarget(null);

// 2. Create a billboard sprite using that texture
const spriteMat = new THREE.SpriteMaterial({
  map: impostorRT.texture,
  transparent: true,
  sizeAttenuation: true,   // scale with distance (perspective)
});
const sprite = new THREE.Sprite(spriteMat);
sprite.scale.set(objectDiameter, objectDiameter, 1);

// 3. Add as the farthest LOD level
lod.addLevel(sprite, 8000);
```

`THREE.Sprite` always faces the camera automatically — no manual billboarding required.

### Texture Atlas for Many Impostors

For hundreds of LOD objects sharing the same impostor, bake all angles into one texture atlas and use UV offsets per instance rather than unique render targets per object.

---

## 7. Combined LOD + Instancing

Three.js does not have a built-in class that merges `LOD` and `InstancedMesh`. The standard pattern is **multiple InstancedMesh objects, one per LOD level**, with CPU-side distance binning each frame.

### Pattern: Per-Frame LOD Bins

```js
const LOD_DISTANCES = [0, 100, 500, 2000];

// One InstancedMesh per LOD level
const lodMeshes = LOD_DISTANCES.map((_, i) =>
  new THREE.InstancedMesh(geometries[i], material, MAX_INSTANCES)
);
lodMeshes.forEach(m => scene.add(m));

// Per-instance data (positions stored CPU-side)
const positions = new Float32Array(TOTAL_INSTANCES * 3);
// ...fill positions...

const dummy  = new THREE.Object3D();
const counts = new Int32Array(LOD_DISTANCES.length);

function updateLOD() {
  // Reset counts
  counts.fill(0);

  const camPos = camera.position;

  for (let i = 0; i < TOTAL_INSTANCES; i++) {
    const dx = positions[i * 3]     - camPos.x;
    const dy = positions[i * 3 + 1] - camPos.y;
    const dz = positions[i * 3 + 2] - camPos.z;
    const dist = Math.sqrt(dx * dx + dy * dy + dz * dz);

    // Find the highest LOD level whose distance threshold is ≤ dist
    let level = LOD_DISTANCES.length - 1;
    for (let l = 0; l < LOD_DISTANCES.length - 1; l++) {
      if (dist < LOD_DISTANCES[l + 1]) {
        level = l;
        break;
      }
    }

    const idx = counts[level]++;
    dummy.position.set(
      positions[i * 3],
      positions[i * 3 + 1],
      positions[i * 3 + 2],
    );
    dummy.updateMatrix();
    lodMeshes[level].setMatrixAt(idx, dummy.matrix);
  }

  // Apply counts and flag updates
  for (let l = 0; l < lodMeshes.length; l++) {
    lodMeshes[l].count = counts[l];
    lodMeshes[l].instanceMatrix.needsUpdate = true;
  }
}
```

### Performance Notes on LOD Binning

From the three.js forum (verified, 2024): using spatial chunking reduces the per-frame work from O(N) to O(visible chunks):

> One implementation achieved a reduction from 4.5ms to 1.3ms per frame with 10,000 visible trees across four LOD levels by dividing the world into 16×16 grid cells and only re-binning instances in cells that crossed a LOD boundary.

For scenes with > 500k instances, use a BVH or octree over instance positions (see `three-mesh-bvh` or `BVH.js`) to turn distance queries into spatial lookups.

---

## 8. BVH-Indexed LOD at Scale (InstancedMesh2 Pattern)

The community library **`three-instanced-mesh2`** (agargaro, 2024) demonstrates the maximal approach:

```js
import { InstancedMeshLOD } from 'three-instanced-mesh2';

const lodMesh = new InstancedMeshLOD(renderer, TOTAL_COUNT);
lodMesh.addLevel(highGeo, mat, 0);    // 0–100 units
lodMesh.addLevel(medGeo,  mat, 100);  // 100–500 units
lodMesh.addLevel(lowGeo,  mat, 500);  // 500–1000 units
lodMesh.computeBVH();                 // build spatial index once

scene.add(lodMesh);
// LOD switching happens internally on update
```

`computeBVH()` builds a spatial index over all instance positions, allowing the library to find which instances need level re-assignment in sub-linear time per frame. This is distinct from the per-geometry BVH that `three-mesh-bvh` provides.

---

## 9. Dynamic LOD by Screen-Space Size

An alternative to distance-based LOD is **screen-space projected size**. This handles cases where a close but small object should use a low LOD and a far but zoomed object should use a high LOD.

```js
function getScreenSpaceSize(objectWorldPos, camera, renderer) {
  // Project object center to clip space
  const projected = objectWorldPos.clone().project(camera);

  // Screen-space Y extent from a point 1 unit above center
  const top = objectWorldPos.clone()
    .add(new THREE.Vector3(0, 1, 0))
    .project(camera);

  const screenHeight = renderer.domElement.clientHeight;
  return Math.abs(projected.y - top.y) * screenHeight * 0.5;
}

// Use in LOD update instead of world-space distance
const pxSize = getScreenSpaceSize(instanceWorldPos, camera, renderer);
const level  = pxSize > 200 ? 0 : pxSize > 50 ? 1 : pxSize > 10 ? 2 : 3;
```

---

## 10. LOD with Frustum Culling

THREE.LOD inherits `Object3D.frustumCulled`. When `true` (default), the entire LOD object — all levels — is culled if the LOD's bounding sphere is outside the frustum. Set `lod.frustumCulled = true` and ensure `lod.geometry.computeBoundingSphere()` has been called, or set an explicit `lod.boundingSphere`.

For large scenes with many LOD objects, a scene-level spatial structure (octree, grid) is more efficient than relying on per-object frustum tests.

---

## 11. Raycast Behaviour

`LOD.raycast` delegates to the **currently visible level object** only:

```js
const hits = raycaster.intersectObject(lod, true);
// hits[0].object is the active level's mesh, not the LOD parent
```

If you need consistent raycasting regardless of the active level, intersect the highest-detail mesh directly or use the BVH approach on a dedicated invisible high-poly mesh.

---

## 12. Quick Reference

| Task | Code |
|------|------|
| Add high-detail level | `lod.addLevel(mesh, 0)` |
| Add fallback billboard | `lod.addLevel(sprite, farDist)` |
| Hysteresis (anti-flicker) | `lod.addLevel(mesh, dist, 0.1)` |
| Manual update | `lod.autoUpdate = false; lod.update(camera)` |
| Current level index | `lod.getCurrentLevel()` |
| Object at distance | `lod.getObjectForDistance(d)` |
| Instance LOD pool | Multiple `InstancedMesh` + per-frame bin sort |
| Impostor sprite | `THREE.Sprite` + `SpriteMaterial` + `sizeAttenuation: true` |

---

## Sources

```yaml
- url: https://raw.githubusercontent.com/mrdoob/three.js/dev/src/objects/LOD.js
  tier: 1
  accessed_at_utc: "2026-05-26T00:00:00Z"
  notes: "Constructor, levels array structure {object, distance, hysteresis}, autoUpdate, distance/zoom formula"

- url: https://raw.githubusercontent.com/mrdoob/three.js/dev/examples/webgl_lod.html
  tier: 1
  accessed_at_utc: "2026-05-26T00:00:00Z"
  notes: "1000 LOD objects, IcosahedronGeometry at distances [50,300,1000,2000,8000], autoUpdate used"

- url: https://discourse.threejs.org/t/lod-instancing/20524
  tier: 4
  accessed_at_utc: "2026-05-26T00:00:00Z"
  notes: "Multiple InstancedMesh pattern, chunk-based spatial binning, 4.5ms→1.3ms improvement with grid cells"

- url: https://discourse.threejs.org/t/instancedmesh-lod-1-million-instances/70748
  tier: 4
  accessed_at_utc: "2026-05-26T00:00:00Z"
  notes: "InstancedMeshLOD + computeBVH pattern, geometry levels with BVH spatial index"

- url: https://discourse.threejs.org/t/about-dynamic-imposters/27330
  tier: 4
  accessed_at_utc: "2026-05-26T00:00:00Z"
  notes: "Dynamic impostors concept, billboard replacement for distant meshes"
```
