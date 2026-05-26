# Frustum Culling — Deep Dive

**Sources:**
- https://threejs.org/docs/index.html#api/en/math/Frustum (accessed 2026-05-26)
- https://threejs.org/docs/index.html#api/en/core/Object3D (accessed 2026-05-26)
- https://threejs.org/docs/index.html#api/en/objects/InstancedMesh (accessed 2026-05-26)

---

## How Three.js Frustum Culling Works Internally

Every frame, `WebGLRenderer.render(scene, camera)` runs a visibility pass before issuing draw calls. The renderer:

1. Extracts the camera's combined view-projection matrix: `camera.projectionMatrix * camera.matrixWorldInverse`.
2. Calls `frustum.setFromProjectionMatrix(projScreenMatrix)` to derive the six frustum planes.
3. For each renderable `Object3D` in the scene graph (depth-first traversal):
   a. Checks `object.visible === false` — skip immediately.
   b. Checks `object.frustumCulled === true` — if so, tests the object's bounding sphere against the six planes.
   c. If the bounding sphere is outside any plane, the object is skipped for that frame (no draw call, no vertex shader invocation).
4. Bounding sphere comes from `geometry.boundingSphere`. If it is `null`, Three.js calls `geometry.computeBoundingSphere()` on demand.

**Key implication:** Frustum culling operates on the geometry's *world-space* bounding sphere. A geometry with a correct `boundingSphere` will be culled accurately. A geometry with an inflated or incorrect `boundingSphere` will over-render (safe but wasteful).

---

## THREE.Frustum API

```ts
// Derive frustum from camera matrices — call each frame
frustum.setFromProjectionMatrix(projectionMatrix: THREE.Matrix4): this

// Test a scene object (uses its geometry.boundingSphere in world space)
frustum.intersectsObject(object: THREE.Object3D): boolean

// Test an axis-aligned bounding box
frustum.intersectsBox(box: THREE.Box3): boolean

// Test a sphere
frustum.intersectsSphere(sphere: THREE.Sphere): boolean

// Test a single point
frustum.containsPoint(point: THREE.Vector3): boolean
```

### Manual Frustum Setup Pattern

```js
import * as THREE from 'three';

const frustum = new THREE.Frustum();
const projScreenMatrix = new THREE.Matrix4();

function cullObjects(camera, objects) {
  camera.updateMatrixWorld();
  projScreenMatrix.multiplyMatrices(
    camera.projectionMatrix,
    camera.matrixWorldInverse
  );
  frustum.setFromProjectionMatrix(projScreenMatrix);

  for (const obj of objects) {
    obj.visible = frustum.intersectsObject(obj);
  }
}
```

Call `camera.updateMatrixWorld()` before building the frustum, especially when the camera has been repositioned outside of the normal render loop.

---

## Object3D.frustumCulled

```ts
object.frustumCulled: boolean  // default: true
```

When `true`, Three.js tests the object's bounding sphere each frame. When `false`, the object always renders regardless of camera position.

**Set to `false` for:**
- Objects that are always on screen (skyboxes, fullscreen quads, UI planes).
- Objects whose bounding sphere is known to always intersect the frustum (e.g., very large terrain meshes).
- Objects managed by a custom culling system that sets `object.visible` directly — disabling the built-in check avoids the redundant sphere test.

```js
skyboxMesh.frustumCulled = false;
postProcessQuad.frustumCulled = false;
```

---

## Custom Frustum Culling with intersectsBox and intersectsSphere

`intersectsObject` is convenient but uses the geometry's sphere from its local-space `boundingSphere` transformed to world space. For tighter bounds or when you manage bounds yourself, use `intersectsBox` or `intersectsSphere` directly:

```js
const worldBox = new THREE.Box3();
const worldSphere = new THREE.Sphere();

function isVisible(mesh, frustum) {
  // Box test — tighter than sphere for axis-aligned geometry
  if (!mesh.geometry.boundingBox) mesh.geometry.computeBoundingBox();
  worldBox.copy(mesh.geometry.boundingBox).applyMatrix4(mesh.matrixWorld);
  return frustum.intersectsBox(worldBox);
}
```

Box tests are slightly more expensive per test than sphere tests but produce fewer false positives for elongated or flat geometry (terrain tiles, corridors).

---

## Instanced Mesh Culling Problem

`THREE.InstancedMesh` exposes a critical limitation: **`frustumCulled` applies to the entire instanced mesh as a single unit, not to individual instances.** The engine tests only the InstancedMesh's own bounding sphere. If any instance is in view, all instances are submitted to the GPU.

This means:
- Setting `instancedMesh.frustumCulled = true` culls the entire batch but not per-instance.
- 10,000-instance meshes spread across a large world will always render all 10,000 instances once the bounding sphere (which wraps all instances) enters the frustum.

### Manual Per-Instance Culling

Implement per-instance culling by manipulating `InstancedMesh.count` and rearranging the instance matrix buffer each frame (or using a visibility mask):

```js
const tempMatrix = new THREE.Matrix4();
const tempSphere = new THREE.Sphere();

function culledInstancedRender(instancedMesh, frustum, allMatrices, boundRadius) {
  let visibleCount = 0;

  for (let i = 0; i < allMatrices.length; i++) {
    // Get the instance's world position from its matrix
    tempMatrix.fromArray(allMatrices[i]);
    const pos = new THREE.Vector3().setFromMatrixPosition(tempMatrix);
    tempSphere.set(pos, boundRadius);

    if (frustum.intersectsSphere(tempSphere)) {
      // Pack visible instance into the front of the buffer
      instancedMesh.setMatrixAt(visibleCount, tempMatrix);
      visibleCount++;
    }
  }

  instancedMesh.count = visibleCount;
  instancedMesh.instanceMatrix.needsUpdate = true;
}
```

**Notes:**
- Sorting instances each frame has O(n) CPU cost. For 10k–100k instances this can matter — consider only updating when the camera moves significantly.
- Alternatively, use a `THREE.Float32BufferAttribute` as a per-instance visibility flag and filter in a vertex shader, avoiding CPU matrix repacking.

---

## GPU-Side Culling

CPU frustum culling eliminates draw calls for invisible objects but still submits all visible geometry to the GPU. For scenes with millions of triangles per object or extremely high draw counts, GPU-side culling is necessary.

### Approach 1: Compute Shader Culling (WebGPU)

With `THREE.WebGPURenderer` (Three.js r168+, still in transition), you can run a compute pass that:
1. Reads per-instance world positions and bounding radii from a storage buffer.
2. Tests each against the frustum planes (passed as uniforms).
3. Writes visible instance indices into an indirect draw argument buffer.
4. The main draw call uses `drawIndexedIndirect`, never touching the CPU.

This approach scales to millions of instances with near-zero CPU overhead.

### Approach 2: Occlusion + Frustum in One Pass (WebGL2)

The `EXT_occlusion_query` extension (see `occlusion-culling.md`) can be combined with frustum culling: frustum-culled objects skip the occlusion query entirely. Only frustum-visible objects go through the hardware occlusion test.

### Approach 3: Hierarchical Z-Buffer (HZB) Culling

Pre-render a depth pyramid from the previous frame. In a compute shader, test each object's projected bounding box against the corresponding HZB mip level. Objects whose projected AABB is fully occluded in the previous frame's depth buffer are skipped. This handles both frustum and occlusion in one GPU pass.

HZB culling is standard in game engines (Unreal, Unity DOTS) but requires WebGPU compute for a full implementation in the browser.

---

## Frustum Plane Access

Sometimes you need the raw planes for custom GPU uniforms:

```js
// frustum.planes is an array of 6 THREE.Plane instances
// Each plane: { normal: Vector3, constant: number }
// Plane equation: dot(normal, point) + constant >= 0 means inside

const planes = frustum.planes;  // [left, right, bottom, top, near, far]

// Pack for GLSL (8 floats per plane = normal.xyz + constant)
const planeData = new Float32Array(6 * 4);
for (let i = 0; i < 6; i++) {
  planeData[i * 4 + 0] = planes[i].normal.x;
  planeData[i * 4 + 1] = planes[i].normal.y;
  planeData[i * 4 + 2] = planes[i].normal.z;
  planeData[i * 4 + 3] = planes[i].constant;
}

// Pass as uniform array to a compute or vertex shader
```

---

## Practical Culling Decision Tree

```
Object has frustumCulled = false?
  └─ Always render (skybox, UI)

Object is InstancedMesh?
  └─ Built-in cull tests entire batch bounding sphere only
  └─ Large worlds: implement per-instance CPU or GPU cull

Object is static with tight bounding sphere?
  └─ Default frustumCulled = true is sufficient

Object is elongated / flat (terrain tile)?
  └─ Use intersectsBox with geometry.boundingBox for tighter test

Object is LOD group?
  └─ THREE.LOD handles cull per-level automatically
```

---

## Common Mistakes

| Mistake | Consequence | Fix |
|---|---|---|
| `geometry.boundingSphere = null` and never calling `computeBoundingSphere()` | Three.js computes it on first render — stalls the frame | Call `computeBoundingSphere()` after geometry is finalized |
| Infinite `boundingSphere.radius` (e.g., procedural geometry with NaN) | Object never culled | Validate geometry before adding to scene |
| `frustumCulled = false` on all objects "for safety" | All objects always render | Only disable for objects that genuinely need it |
| Not calling `camera.updateMatrixWorld()` before manual frustum tests | Stale view matrix, incorrect culling | Always update before sampling `camera.matrixWorldInverse` |
| Per-instance culling without `instanceMatrix.needsUpdate = true` | GPU sees old matrices | Set flag after every `setMatrixAt` batch |
