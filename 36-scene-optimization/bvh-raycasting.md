# BVH Raycasting with three-mesh-bvh

**Library:** `three-mesh-bvh` by Garrett Johnson (gkjohnson)
**Version verified:** v0.9.10 (released 2026-05-13)
**Source:** https://github.com/gkjohnson/three-mesh-bvh
**API reference:** https://github.com/gkjohnson/three-mesh-bvh/blob/master/API.md
**Accessed:** 2026-05-26

---

## What It Is

`three-mesh-bvh` builds a Bounding Volume Hierarchy (BVH) over a `THREE.BufferGeometry`'s triangle data and replaces Three.js's default linear raycasting loop with an O(log n) tree traversal. The library also exposes that same spatial structure for a broad set of non-raycast queries (distance, overlap, custom shape casts).

The README's performance headline: **"Casting 500 rays against an 80,000 polygon model at 60fps!"** — the default Three.js approach cannot sustain this workload at real-time frame rates.

---

## Installation

```bash
npm install three-mesh-bvh
```

---

## One-Time Prototype Extension (Global Setup)

The recommended pattern is to extend Three.js prototypes once at application startup. This lets every `BufferGeometry` use `.computeBoundsTree()` and every `Mesh` automatically use accelerated raycasting.

```js
import * as THREE from 'three';
import {
  computeBoundsTree,
  disposeBoundsTree,
  acceleratedRaycast,
  computeBatchedBoundsTree,
} from 'three-mesh-bvh';

// Extend BufferGeometry for BVH build/dispose
THREE.BufferGeometry.prototype.computeBoundsTree  = computeBoundsTree;
THREE.BufferGeometry.prototype.disposeBoundsTree  = disposeBoundsTree;

// Replace default raycast with the accelerated version
THREE.Mesh.prototype.raycast = acceleratedRaycast;

// BatchedMesh support (r168+)
THREE.BatchedMesh.prototype.computeBoundsTree = computeBatchedBoundsTree;
```

After this setup, every `new THREE.Mesh(geometry, material)` raycasts through the BVH automatically—no per-mesh changes needed.

---

## Building and Destroying a BVH

```js
// Build — runs synchronously; stores the BVH on geometry.boundsTree
geometry.computeBoundsTree(options);

// Destroy — frees the typed-array memory held in geometry.boundsTree
geometry.disposeBoundsTree();
```

### Constructor Options

All options are passed to `computeBoundsTree()` or the `MeshBVH` constructor directly.

| Option | Type | Default | Notes |
|---|---|---|---|
| `strategy` | `CENTER \| AVERAGE \| SAH` | `CENTER` | Tree-split strategy (see below) |
| `maxDepth` | `number` | `40` | Maximum tree depth |
| `maxLeafSize` | `number` | `10` | Max triangles per leaf node |
| `setBoundingBox` | `boolean` | `true` | Auto-set `geometry.boundingBox` |
| `useSharedArrayBuffer` | `boolean` | `false` | Use SAB for worker transfer |
| `indirect` | `boolean` | `false` | Indirect index mode |
| `verbose` | `boolean` | `true` | Log build info to console |
| `onProgress` | `function \| null` | `null` | Progress callback during build |
| `range` | `Object \| null` | `null` | Restrict BVH to index sub-range |

### Split Strategy Tradeoffs

- **CENTER** — fastest build, good general performance. Default choice.
- **AVERAGE** — may produce tighter bounds than CENTER for irregular geometry.
- **SAH** (Surface Area Heuristic) — slowest build, tightest bounds, lowest memory. Best for geometry that is raycasted very frequently and built only once.

```js
import { SAH } from 'three-mesh-bvh';

geometry.computeBoundsTree({ strategy: SAH });
```

---

## MeshBVH Class — Direct API

When you need more control than the prototype extensions provide, use `MeshBVH` directly:

```js
import { MeshBVH } from 'three-mesh-bvh';

const bvh = new MeshBVH(geometry, { strategy: SAH });
```

### Raycast Methods

```ts
// All intersections, unsorted
bvh.raycast(
  ray: THREE.Ray,
  materialOrSide: number | Material | Material[] = FrontSide,
  near: number = 0,
  far: number = Infinity
): Intersection[]

// First intersection only — "typically much faster" than raycast()
bvh.raycastFirst(
  ray: THREE.Ray,
  materialOrSide: number | Material | Material[] = FrontSide,
  near: number = 0,
  far: number = Infinity
): Intersection | null
```

Use `raycastFirst` whenever you only need a hit/miss answer (e.g., mouse picking, shadow rays).

---

## Spatial Query Methods

All queries operate directly on the `MeshBVH` instance (`geometry.boundsTree`).

### closestPointToPoint

```ts
bvh.closestPointToPoint(
  point: THREE.Vector3,
  target: HitPointInfo = {},
  minThreshold: number = 0,
  maxThreshold: number = Infinity
): HitPointInfo | null
```

"Computes the closest distance from the point to the mesh and gives additional information in `target`."

`HitPointInfo` fields: `{ point: Vector3, distance: number, faceIndex: number }`.

```js
const result = geometry.boundsTree.closestPointToPoint(
  playerPosition,
  {},      // target
  0,       // minThreshold — ignore hits closer than this
  50       // maxThreshold — early-exit if we're already far away
);
if (result) console.log(result.distance, result.point);
```

### closestPointToGeometry

```ts
bvh.closestPointToGeometry(
  otherGeometry: THREE.BufferGeometry,
  geometryToBvh: THREE.Matrix4,
  target1: HitPointInfo = {},   // closest point on this mesh
  target2: HitPointInfo = {},   // closest point on otherGeometry
  minThreshold: number = 0,
  maxThreshold: number = Infinity
): HitPointInfo | null
```

"Computes the closest distance from the geometry to the mesh and puts the closest point on the mesh in `target1`."

### intersectsGeometry

```ts
bvh.intersectsGeometry(
  otherGeometry: THREE.BufferGeometry,
  geometryToBvh: THREE.Matrix4
): boolean
```

"Returns whether or not the mesh intersects the given geometry." Used for exact mesh-vs-mesh overlap tests (not just AABB).

### intersectsSphere

```ts
bvh.intersectsSphere(sphere: THREE.Sphere): boolean
```

"Returns whether or not the mesh intersects the given sphere."

### intersectsBox

```ts
bvh.intersectsBox(box: THREE.Box3, boxToBvh: THREE.Matrix4): boolean
```

"Returns whether or not the mesh intersects the given box."

### shapecast — Custom Shape Queries

`shapecast` is the generic traversal primitive. All the above methods are implemented on top of it.

```ts
bvh.shapecast({
  // Called for each BVH node's bounding box
  // Return value: NOT_INTERSECTED | INTERSECTED | CONTAINED (or numeric)
  intersectsBounds: (
    box: THREE.Box3,
    isLeaf: boolean,
    score: number | undefined,
    depth: number,
    nodeIndex: number
  ) => number,

  // Called for each triangle in intersected leaves
  // Return true to stop traversal early
  intersectsTriangle?: (
    triangle: ExtendedTriangle,
    index: number,
    contained: boolean,
    depth: number
  ) => boolean,

  // Alternative to intersectsTriangle — operates on index ranges
  intersectsRange?: (
    offset: number,
    count: number,
    contained: boolean,
    depth: number,
    nodeIndex: number,
    box: THREE.Box3
  ) => boolean,

  // Optional: controls traversal order by returning a sort key per node
  boundsTraverseOrder?: (box: THREE.Box3) => number,
}): boolean
```

---

## Serialization (Pre-baking BVHs)

Build the BVH offline (Node.js build step) and deserialize at runtime to avoid the build cost:

```js
// Build once, serialize to transferable buffer
const serialized = MeshBVH.serialize(bvh, { cloneBuffers: true });
// { roots: ArrayBuffer[], index: ArrayBuffer }

// At runtime: deserialize — O(1), just sets typed-array pointers
const bvh = MeshBVH.deserialize(serialized, geometry, { setIndex: true });
geometry.boundsTree = bvh;
```

---

## Asynchronous BVH Generation (Workers)

For large geometries that would block the main thread:

```js
import { GenerateMeshBVHWorker } from 'three-mesh-bvh/src/workers/GenerateMeshBVHWorker.js';

const worker = new GenerateMeshBVHWorker();
const bvh = await worker.generate(geometry, { strategy: SAH });
geometry.boundsTree = bvh;
worker.dispose();
```

`ParallelMeshBVHWorker` uses `SharedArrayBuffer` for multi-threaded construction (requires COOP/COEP headers).

---

## BVH Debug Visualization

Use `BVHHelper` (formerly `MeshBVHVisualizer`) to render the BVH node hierarchy as wireframe boxes:

```js
import { BVHHelper } from 'three-mesh-bvh';

// depth: how many BVH levels to visualize (0 = root only)
const helper = new BVHHelper(mesh, /* depth */ 10);
scene.add(helper);

// After BVH changes
helper.update();

// Remove
scene.remove(helper);
helper.dispose();
```

The helper is for development/profiling only. Remove it in production builds.

---

## Specialized BVH Types

The library supports non-mesh primitives:

| Class | Geometry type |
|---|---|
| `MeshBVH` | `THREE.Mesh` triangles |
| `PointsBVH` | `THREE.Points` |
| `LineBVH` | `THREE.Line` |
| `LineLoopBVH` | `THREE.LineLoop` |
| `LineSegmentsBVH` | `THREE.LineSegments` |
| `SkinnedMeshBVH` | `THREE.SkinnedMesh` (with skinning) |

---

## Utility: getTriangleHitPointInfo

Retrieve UV coordinates, face normal, and material group index from a raw hit point (useful after `raycastFirst`):

```js
import { getTriangleHitPointInfo } from 'three-mesh-bvh';

const hit = geometry.boundsTree.raycastFirst(ray);
if (hit) {
  const info = getTriangleHitPointInfo(hit.point, geometry, hit.faceIndex, {});
  // info.uv, info.normal, info.materialIndex
}
```

---

## Integration Checklist

- [ ] Call `geometry.computeBoundsTree()` after geometry is finalized (before first render).
- [ ] Call `geometry.disposeBoundsTree()` when the geometry is removed from the scene.
- [ ] For geometry that changes every frame (morph targets, skinning), rebuild the BVH or use `SkinnedMeshBVH`.
- [ ] For mouse picking, prefer `raycastFirst` over `raycast`.
- [ ] Set `verbose: false` in production to suppress console output.
- [ ] Pre-serialize BVHs for large static world geometry in your build pipeline.

---

## Related Libraries

- `@pmndrs/bvh` — React Three Fiber wrapper around three-mesh-bvh
- `three-pathfinding` — nav-mesh pathfinding, uses BVH internally
- `three-gpu-pathtracer` — GPU path tracer by the same author, uses `MeshBVHUniformStruct` for GLSL BVH traversal
