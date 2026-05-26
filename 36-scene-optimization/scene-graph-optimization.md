# Scene Graph Optimization

**Sources:**
- https://threejs.org/docs/index.html#api/en/core/Object3D (accessed 2026-05-26)
- https://threejs.org/docs/index.html#examples/en/utils/BufferGeometryUtils (accessed 2026-05-26)
- https://github.com/mrdoob/three.js/blob/master/examples/jsm/utils/BufferGeometryUtils.js (accessed 2026-05-26)

---

## The Hidden Cost of the Scene Graph

Three.js's scene graph is a tree of `Object3D` nodes. Each frame, `renderer.render(scene, camera)` traverses this tree to:

1. Update each node's `matrixWorld` (if `matrixWorldAutoUpdate` is true).
2. Collect renderable objects into a render list.
3. Sort the render list (opaque front-to-back, transparent back-to-front).
4. Issue draw calls.

For a deep scene graph with hundreds of nodes, steps 1–3 alone can consume several milliseconds per frame — before any GPU work starts. The optimizations below target the CPU side of this pipeline.

---

## Avoid Deep Hierarchies

### The Problem

Each level of nesting multiplies matrix multiplications. A node at depth 8 requires 8 matrix multiplications to compute its `matrixWorld` from the root. With many nodes at deep levels this accumulates:

```
depth 1: 1 multiply
depth 2: 2 multiplies
depth 8: 8 multiplies per node
100 nodes at depth 8: 800 matrix multiplies per frame
```

### Flat vs Deep

```js
// Deep (slow for static content)
scene
  └─ world
       └─ region
            └─ block
                 └─ building
                      └─ floor
                           └─ room
                                └─ wall  <- 7 levels deep

// Flat (preferred for static geometry)
scene
  └─ wall  <- 1 level deep, matrixWorld = its own matrix
```

### Practical Flattening

For static objects that never move, remove intermediate grouping nodes after the scene is built. Instead of grouping for the sake of code organization, group only when you need shared transforms or when visibility toggling on a group is required.

```js
// Before optimization: 3-level group used only for code organization
const group = new THREE.Group();
group.add(meshA, meshB, meshC);
scene.add(group);

// After: if group never moves or toggles, add children directly
scene.add(meshA, meshB, meshC);
```

---

## matrixWorldAutoUpdate = false for Static Objects

`Object3D.matrixWorldAutoUpdate` (default: `true`) tells the renderer to recompute `matrixWorld` every frame. For objects that never move, this is wasteful.

```ts
object.matrixWorldAutoUpdate: boolean  // default: true
```

```js
// Disable automatic updates for any object that won't move
staticMesh.matrixWorldAutoUpdate = false;
staticMesh.matrixAutoUpdate      = false;  // also disable local matrix rebuild

// Recursively disable for an entire static sub-tree
function freezeTransforms(object) {
  object.matrixAutoUpdate      = false;
  object.matrixWorldAutoUpdate = false;
  for (const child of object.children) freezeTransforms(child);
}

freezeTransforms(terrainGroup);
```

**Important:** After calling `freezeTransforms`, the object's current transform is baked in. If you later need to move the object, re-enable the flags and update:

```js
staticMesh.matrixWorldAutoUpdate = true;
staticMesh.matrixAutoUpdate      = true;
staticMesh.position.set(10, 0, 5);  // change will take effect next frame
```

---

## Object3D.updateMatrixWorld() — Manual Calls

When `matrixWorldAutoUpdate = false`, you are responsible for calling `updateMatrixWorld` whenever the transform changes.

```ts
object.updateMatrixWorld(force?: boolean): void
```

- Recomputes `matrixWorld` for the object and **all its children**.
- `force = true` propagates the update even if the object's `matrixWorldNeedsUpdate` flag is false.

```js
// Pattern: update once at scene build time, then never again
staticRoot.updateMatrixWorld(true);
freezeTransforms(staticRoot);

// Pattern: update only when the transform actually changes
function moveObject(object, newPosition) {
  object.position.copy(newPosition);
  object.updateMatrixWorld(true);
}
```

### updateWorldMatrix vs updateMatrixWorld

```ts
// updateWorldMatrix: more surgical — update only this node, optionally propagate up/down
object.updateWorldMatrix(
  updateParents: boolean,   // walk up to root first
  updateChildren: boolean   // propagate down after
): void
```

Use `updateWorldMatrix(false, false)` to update only a single node's `matrixWorld` when you know the parent is already current.

---

## Merging Static Geometries with BufferGeometryUtils

The most impactful scene graph optimization for static content: merge multiple geometries into one. One merged mesh = one draw call instead of N draw calls. Each draw call has 0.05–0.3ms of CPU overhead in WebGL; eliminating 500 draw calls can recover 25–150ms of frame budget.

### Import

```js
import {
  mergeGeometries,
  mergeVertices,
  mergeAttributes,
  interleaveAttributes,
  estimateBytesUsed,
  toTrianglesDrawMode,
} from 'three/examples/jsm/utils/BufferGeometryUtils.js';
```

### mergeGeometries

```ts
mergeGeometries(
  geometries: THREE.BufferGeometry[],
  useGroups: boolean = false
): THREE.BufferGeometry | null
```

Merges an array of `BufferGeometry` instances into one. All geometries must have the same set of attributes (same attribute names and item sizes). Returns `null` on attribute mismatch.

- `useGroups = false` — all triangles share the same material index 0.
- `useGroups = true` — each input geometry becomes a draw group, allowing a `MultiMaterial` (array of materials) on the merged mesh.

```js
// Collect all road tile geometries, apply their world transforms, merge
const roadGeometries = roadTiles.map(tile => {
  const geom = tile.geometry.clone();
  geom.applyMatrix4(tile.matrixWorld);  // bake transform into vertices
  return geom;
});

const merged = mergeGeometries(roadGeometries, false);
if (merged) {
  const mergedMesh = new THREE.Mesh(merged, sharedMaterial);
  scene.add(mergedMesh);
  // Remove original tiles
  for (const tile of roadTiles) scene.remove(tile);
}
```

**Limitations of mergeGeometries:**
- All merged triangles share the same draw call — you cannot frustum-cull sub-sections of the merged mesh.
- Merged geometry cannot be partially updated; if one tile changes, rebuild the entire merged mesh.
- Memory usage equals the sum of all input geometries.

### Merging with Groups (Multi-Material)

```js
const merged = mergeGeometries(geometries, true);  // useGroups = true
const multiMaterialMesh = new THREE.Mesh(merged, [materialA, materialB, materialC]);
// Each group corresponds to one input geometry in the original array
```

### mergeVertices

```ts
mergeVertices(
  geometry: THREE.BufferGeometry,
  tolerance: number = 1e-4
): THREE.BufferGeometry
```

Deduplicates vertices within `tolerance` distance. Reduces geometry size for meshes with shared vertices that were not indexed (e.g., after Boolean operations or import from formats without indices).

```js
let geom = someImportedGeometry.clone();
geom = mergeVertices(geom, 1e-4);
geom.computeVertexNormals();
```

### mergeAttributes

```ts
mergeAttributes(
  attributes: THREE.BufferAttribute[]
): THREE.BufferAttribute | null
```

Concatenates multiple `BufferAttribute` arrays into one. Used internally by `mergeGeometries` but useful standalone when you manage raw attributes.

### interleaveAttributes

```ts
interleaveAttributes(
  attributes: THREE.BufferAttribute[]
): THREE.InterleavedBufferAttribute[] | null
```

Interleaves multiple attributes (position, normal, uv, ...) into a single `InterleavedBuffer`. Improves GPU cache locality by ensuring position+normal+uv for each vertex are adjacent in memory. Can reduce vertex fetch stalls on cache-pressure-heavy scenes.

```js
const [iPos, iNorm, iUV] = interleaveAttributes([
  geometry.attributes.position,
  geometry.attributes.normal,
  geometry.attributes.uv,
]);
geometry.setAttribute('position', iPos);
geometry.setAttribute('normal',   iNorm);
geometry.setAttribute('uv',       iUV);
```

### estimateBytesUsed

```ts
estimateBytesUsed(geometry: THREE.BufferGeometry): number
```

Returns the estimated GPU memory usage in bytes for a geometry's buffer attributes. Useful for memory budgeting and leak detection.

```js
console.log(`${estimateBytesUsed(mergedGeometry) / 1024 / 1024} MB`);
```

### toTrianglesDrawMode

```ts
toTrianglesDrawMode(
  geometry: THREE.BufferGeometry,
  drawMode: number  // THREE.TriangleStripDrawMode | THREE.TriangleFanDrawMode
): THREE.BufferGeometry
```

Converts strip or fan geometries to standard triangle lists. Needed before merging with other triangle-list geometries.

### computeMikkTSpaceTangents

```ts
computeMikkTSpaceTangents(
  geometry: THREE.BufferGeometry,
  MikkTSpace: Object,
  negateSign: boolean = true
): THREE.BufferGeometry
```

Computes accurate tangent vectors using the MikkTSpace algorithm (the standard used by Blender, Maya, Substance). Required for correct normal mapping. `MikkTSpace` is an instance from the `three/examples/jsm/libs/mikktspace.module.js` WASM module.

---

## Geometry Instance vs Merge Decision

| Scenario | Use | Reason |
|---|---|---|
| Many identical meshes (trees, rocks, street lights) | `THREE.InstancedMesh` | Single geometry, single draw call, per-instance transforms on GPU |
| Many different static meshes that never move | `mergeGeometries` | One draw call, one vertex buffer |
| Different static meshes with different materials | `mergeGeometries` with `useGroups: true` + `MultiMaterial` | One draw call per material group |
| Objects that need individual frustum culling | Keep separate meshes | Merged mesh is culled as one unit |
| Objects that animate or move | Keep separate meshes | Cannot partially update merged vertex buffers |

---

## Object Disposal

Merged geometries grow large. Failing to dispose them causes GPU memory leaks.

```js
// When removing a merged mesh
scene.remove(mergedMesh);
mergedMesh.geometry.dispose();
mergedMesh.material.dispose();  // or iterate array for MultiMaterial

// Dispose original tiles too (don't forget them in your reference list)
for (const geom of roadGeometries) geom.dispose();
```

---

## Profiling the Scene Graph

Use `renderer.info` to count draw calls and triangles:

```js
console.log({
  calls:     renderer.info.render.calls,
  triangles: renderer.info.render.triangles,
  points:    renderer.info.render.points,
  lines:     renderer.info.render.lines,
  geometries: renderer.info.memory.geometries,
  textures:   renderer.info.memory.textures,
});

// Reset each frame (Three.js resets automatically in render() call)
```

A scene with fewer than 200 draw calls per frame is generally comfortable. Above 1000 calls, draw call overhead starts to dominate CPU frame time on mid-range hardware.
