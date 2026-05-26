# Spatial Structures for Scene Optimization

**Sources:**
- https://raw.githubusercontent.com/mrdoob/three.js/master/examples/jsm/math/Octree.js (accessed 2026-05-26)
- https://threejs.org/docs/index.html#api/en/math/Frustum (accessed 2026-05-26)
- https://github.com/gkjohnson/three-mesh-bvh/blob/master/API.md (accessed 2026-05-26)

---

## Why Spatial Structures Matter

A flat list of N objects requires O(n) tests for any spatial query: frustum culling, collision detection, nearest-neighbor lookup, raycast. At 10,000 objects this is fast; at 1,000,000 it is not. Spatial structures reduce query cost to O(log n) or O(k) where k is the number of results.

Three.js has no general-purpose spatial acceleration built into its scene graph. The options are:

- **Three.js Octree** (`three/addons/math/Octree.js`) — triangle-level collision tree, built from scene geometry.
- **three-mesh-bvh** — BVH over a single geometry's triangles; see `bvh-raycasting.md` for full coverage.
- **Custom grid / quadtree / octree** — hand-rolled structures for object-level (not triangle-level) queries.
- **Chunk-based streaming** — spatial partitioning for world loading/unloading.

---

## Three.js Octree (`examples/jsm/math/Octree.js`)

### What It Is

The Three.js Octree is a triangle-level spatial structure designed primarily for **character-vs-world collision detection** in first-person or physics-driven scenes. It ingests a scene graph and stores all triangle data from mesh geometry in an octree.

**Import path:**

```js
import { Octree } from 'three/addons/math/Octree.js';
import { OctreeHelper } from 'three/addons/math/OctreeHelper.js';
```

Or with the full path:

```js
import { Octree }       from 'three/examples/jsm/math/Octree.js';
import { OctreeHelper } from 'three/examples/jsm/math/OctreeHelper.js';
```

### Key Methods

```ts
// Build the octree from a scene graph node (traverses all meshes)
octree.fromGraphNode(group: THREE.Object3D): void

// Test a sphere against the octree — returns intersection data or false
octree.sphereIntersect(sphere: THREE.Sphere): {
  normal: THREE.Vector3,
  depth: number,
  point: THREE.Vector3,
} | false

// Test a capsule (character controller shape) — returns intersection data or false
octree.capsuleIntersect(capsule: Capsule): {
  normal: THREE.Vector3,
  depth: number,
} | false

// Ray intersection — returns nearest hit or false
octree.rayIntersect(ray: THREE.Ray): {
  normal: THREE.Vector3,
  distance: number,
  point: THREE.Vector3,
  triangle: Triangle,
} | false

// Remove all data
octree.clear(): void
```

Lower-level methods (generally called internally):

```ts
octree.addTriangle(triangle: Triangle): void
octree.build(): void
octree.calcBox(): void
octree.getSphereTriangles(sphere: Sphere, triangles: Triangle[]): Triangle[]
octree.getBoxTriangles(box: Box3, triangles: Triangle[]): Triangle[]
octree.getCapsuleTriangles(capsule: Capsule, triangles: Triangle[]): Triangle[]
octree.getRayTriangles(ray: Ray, triangles: Triangle[]): Triangle[]
octree.boxIntersect(box: Box3): boolean
```

### Usage Pattern (First-Person Character Collision)

```js
import { Octree }   from 'three/addons/math/Octree.js';
import { Capsule }  from 'three/addons/math/Capsule.js';
import { OctreeHelper } from 'three/addons/math/OctreeHelper.js';

const worldOctree = new Octree();

// After loading the level geometry
const gltf = await loader.loadAsync('level.glb');
scene.add(gltf.scene);
worldOctree.fromGraphNode(gltf.scene);

// Debug visualization (development only)
const helper = new OctreeHelper(worldOctree, 0x88ccee);
helper.visible = false;
scene.add(helper);

// Player capsule: bottom point, top point, radius
const playerCapsule = new Capsule(
  new THREE.Vector3(0, 0.35, 0),
  new THREE.Vector3(0, 1.35, 0),
  0.35
);

// Each frame: test and resolve
function playerCollisions() {
  const result = worldOctree.capsuleIntersect(playerCapsule);
  if (result) {
    const isFloor = result.normal.y > 0;
    // Push capsule out of geometry by the penetration depth
    playerCapsule.translate(result.normal.multiplyScalar(result.depth));
    if (isFloor) playerOnFloor = true;
  }
}
```

### Limitations

- The Three.js Octree is built specifically for collision geometry, not for object-level culling.
- `fromGraphNode` traverses and copies all triangle data — do not call it every frame.
- Skinned/animated meshes must rebuild the octree when geometry changes (expensive); best for static collision meshes.
- No partial update support; rebuild entirely after level changes.

---

## Grid-Based Culling

A spatial grid is the simplest O(1) lookup structure for object-level visibility. Divide the world into a 2D (XZ) or 3D grid of fixed-size cells. Each cell holds a list of objects whose origin falls within it.

### Flat Grid Implementation

```js
class SpatialGrid {
  constructor(cellSize = 50) {
    this.cellSize = cellSize;
    this.cells    = new Map();
  }

  _key(x, z) {
    return `${Math.floor(x / this.cellSize)},${Math.floor(z / this.cellSize)}`;
  }

  insert(object) {
    const key = this._key(object.position.x, object.position.z);
    if (!this.cells.has(key)) this.cells.set(key, []);
    this.cells.get(key).push(object);
  }

  remove(object) {
    const key = this._key(object.position.x, object.position.z);
    const cell = this.cells.get(key);
    if (cell) {
      const idx = cell.indexOf(object);
      if (idx !== -1) cell.splice(idx, 1);
    }
  }

  // Return all objects within a radius (in cells)
  query(x, z, cellRadius = 1) {
    const results = [];
    const cx = Math.floor(x / this.cellSize);
    const cz = Math.floor(z / this.cellSize);

    for (let dx = -cellRadius; dx <= cellRadius; dx++) {
      for (let dz = -cellRadius; dz <= cellRadius; dz++) {
        const cell = this.cells.get(`${cx + dx},${cz + dz}`);
        if (cell) results.push(...cell);
      }
    }
    return results;
  }
}

// Usage
const grid = new SpatialGrid(50);
for (const tree of trees) grid.insert(tree);

function updateVisibility(camera) {
  // Hide everything, then show only nearby objects
  for (const [, cell] of grid.cells) {
    for (const obj of cell) obj.visible = false;
  }
  const nearby = grid.query(camera.position.x, camera.position.z, 3);
  for (const obj of nearby) obj.visible = true;
}
```

**Cell size choice:** Too small = more cells to query, overhead from the loop. Too large = more objects per cell, less culling benefit. Rule of thumb: cell size ≈ 2–4× the average object's footprint. Profile to find the sweet spot for your scene density.

---

## Quadtree (2D Hierarchical Grid)

A quadtree recursively subdivides a 2D area into four quadrants. Better than a flat grid when object density is highly non-uniform (dense in cities, sparse in countryside).

```js
class QuadNode {
  constructor(bounds, depth = 0, maxDepth = 6, maxObjects = 10) {
    this.bounds     = bounds;         // THREE.Box2
    this.depth      = depth;
    this.maxDepth   = maxDepth;
    this.maxObjects = maxObjects;
    this.objects    = [];
    this.children   = null;  // null until split
  }

  insert(object) {
    const pos = new THREE.Vector2(object.position.x, object.position.z);
    if (!this.bounds.containsPoint(pos)) return false;

    if (this.children) {
      for (const child of this.children) {
        if (child.insert(object)) return true;
      }
      return false;
    }

    this.objects.push(object);
    if (this.objects.length > this.maxObjects && this.depth < this.maxDepth) {
      this._split();
    }
    return true;
  }

  query(range /* THREE.Box2 */, results = []) {
    if (!this.bounds.intersectsBox(range)) return results;

    for (const obj of this.objects) {
      const p = new THREE.Vector2(obj.position.x, obj.position.z);
      if (range.containsPoint(p)) results.push(obj);
    }

    if (this.children) {
      for (const child of this.children) child.query(range, results);
    }
    return results;
  }

  _split() {
    const mid = this.bounds.getCenter(new THREE.Vector2());
    const { min, max } = this.bounds;

    this.children = [
      new QuadNode(new THREE.Box2(min, mid),                                this.depth + 1, this.maxDepth, this.maxObjects),
      new QuadNode(new THREE.Box2(new THREE.Vector2(mid.x, min.y), new THREE.Vector2(max.x, mid.y)), this.depth + 1, this.maxDepth, this.maxObjects),
      new QuadNode(new THREE.Box2(mid, max),                                this.depth + 1, this.maxDepth, this.maxObjects),
      new QuadNode(new THREE.Box2(new THREE.Vector2(min.x, mid.y), new THREE.Vector2(mid.x, max.y)), this.depth + 1, this.maxDepth, this.maxObjects),
    ];

    const oldObjects = this.objects;
    this.objects = [];
    for (const obj of oldObjects) {
      for (const child of this.children) {
        if (child.insert(obj)) break;
      }
    }
  }
}
```

---

## 3D Octree for Object-Level Queries

For fully 3D scenes (flying objects, multi-floor buildings), extend the grid concept to three dimensions. The pattern mirrors the quadtree but with eight children:

```js
class OctreeNode {
  constructor(bounds, depth = 0, maxDepth = 5) {
    this.bounds   = bounds;   // THREE.Box3
    this.depth    = depth;
    this.maxDepth = maxDepth;
    this.objects  = [];
    this.children = null;
  }

  insert(object) {
    if (!this.bounds.containsPoint(object.position)) return false;

    if (this.children) {
      for (const child of this.children) {
        if (child.insert(object)) return true;
      }
      return false;
    }

    this.objects.push(object);
    if (this.objects.length > 8 && this.depth < this.maxDepth) this._split();
    return true;
  }

  queryFrustum(frustum, results = []) {
    if (!frustum.intersectsBox(this.bounds)) return results;

    for (const obj of this.objects) {
      if (frustum.intersectsObject(obj)) results.push(obj);
    }

    if (this.children) {
      for (const child of this.children) child.queryFrustum(frustum, results);
    }
    return results;
  }

  _split() {
    const center = this.bounds.getCenter(new THREE.Vector3());
    const { min, max } = this.bounds;

    // 8 octants
    this.children = [
      [min, center],
      [new THREE.Vector3(center.x, min.y, min.z), new THREE.Vector3(max.x, center.y, center.z)],
      [new THREE.Vector3(min.x, center.y, min.z), new THREE.Vector3(center.x, max.y, center.z)],
      [new THREE.Vector3(center.x, center.y, min.z), new THREE.Vector3(max.x, max.y, center.z)],
      [new THREE.Vector3(min.x, min.y, center.z), new THREE.Vector3(center.x, center.y, max.z)],
      [new THREE.Vector3(center.x, min.y, center.z), new THREE.Vector3(max.x, center.y, max.z)],
      [new THREE.Vector3(min.x, center.y, center.z), new THREE.Vector3(center.x, max.y, max.z)],
      [center, max],
    ].map(([lo, hi]) => new OctreeNode(new THREE.Box3(lo, hi), this.depth + 1, this.maxDepth));

    const old = this.objects;
    this.objects = [];
    for (const obj of old) {
      for (const child of this.children) {
        if (child.insert(obj)) break;
      }
    }
  }
}
```

**Usage with frustum culling:**

```js
const root = new OctreeNode(new THREE.Box3(
  new THREE.Vector3(-1000, -100, -1000),
  new THREE.Vector3( 1000,  500,  1000)
));

for (const obj of allObjects) root.insert(obj);

function renderFrame(camera, renderer, scene) {
  // Build frustum
  const frustum = new THREE.Frustum();
  const m = new THREE.Matrix4().multiplyMatrices(
    camera.projectionMatrix,
    camera.matrixWorldInverse
  );
  frustum.setFromProjectionMatrix(m);

  // Query only visible objects
  const visible = root.queryFrustum(frustum);

  // Hide all, show visible
  for (const obj of allObjects) obj.visible = false;
  for (const obj of visible)    obj.visible = true;

  renderer.render(scene, camera);
}
```

---

## Chunk-Based Streaming for Large Worlds

When worlds are too large to hold in GPU memory at once, divide them into discrete chunks. Load and unload chunks based on camera proximity.

### Chunk Manager Pattern

```js
class ChunkManager {
  constructor({ chunkSize = 128, loadRadius = 3, unloadRadius = 5, loader }) {
    this.chunkSize    = chunkSize;
    this.loadRadius   = loadRadius;
    this.unloadRadius = unloadRadius;
    this.loader       = loader;     // async (cx, cz) => THREE.Object3D
    this.loaded       = new Map();  // "cx,cz" -> { object, lastSeen }
    this.loading      = new Set();  // "cx,cz" being fetched
  }

  async update(camera, scene) {
    const cx = Math.round(camera.position.x / this.chunkSize);
    const cz = Math.round(camera.position.z / this.chunkSize);

    // Load new chunks
    for (let dx = -this.loadRadius; dx <= this.loadRadius; dx++) {
      for (let dz = -this.loadRadius; dz <= this.loadRadius; dz++) {
        const key = `${cx + dx},${cz + dz}`;
        if (!this.loaded.has(key) && !this.loading.has(key)) {
          this.loading.add(key);
          this.loader(cx + dx, cz + dz).then(obj => {
            scene.add(obj);
            this.loaded.set(key, { object: obj, lastSeen: Date.now() });
            this.loading.delete(key);
          });
        } else if (this.loaded.has(key)) {
          this.loaded.get(key).lastSeen = Date.now();
        }
      }
    }

    // Unload distant chunks
    for (const [key, { object }] of this.loaded) {
      const [kx, kz] = key.split(',').map(Number);
      const dist = Math.max(Math.abs(kx - cx), Math.abs(kz - cz));
      if (dist > this.unloadRadius) {
        scene.remove(object);
        object.traverse(child => {
          if (child.isMesh) {
            child.geometry.dispose();
            if (Array.isArray(child.material)) {
              child.material.forEach(m => m.dispose());
            } else {
              child.material.dispose();
            }
          }
        });
        this.loaded.delete(key);
      }
    }
  }
}
```

### Async Chunk Loading with Worker

For procedurally generated chunks, offload generation to a Worker:

```js
// chunk-worker.js
self.onmessage = function({ data: { cx, cz, seed } }) {
  const geometry = generateChunkGeometry(cx, cz, seed);
  const posArray  = geometry.attributes.position.array;
  const normArray = geometry.attributes.normal.array;
  // Transfer ownership of typed arrays (zero-copy)
  self.postMessage({ cx, cz, posArray, normArray }, [posArray.buffer, normArray.buffer]);
};

// Main thread
const worker = new Worker('chunk-worker.js');
worker.onmessage = function({ data: { cx, cz, posArray, normArray } }) {
  const geom = new THREE.BufferGeometry();
  geom.setAttribute('position', new THREE.BufferAttribute(posArray, 3));
  geom.setAttribute('normal',   new THREE.BufferAttribute(normArray, 3));
  geom.computeBoundingSphere();

  const mesh = new THREE.Mesh(geom, terrainMaterial);
  mesh.position.set(cx * CHUNK_SIZE, 0, cz * CHUNK_SIZE);
  scene.add(mesh);
};
```

---

## Choosing the Right Structure

| Scene type | Objects | Recommended structure |
|---|---|---|
| FPS / physics world | Static collision mesh | Three.js Octree (`fromGraphNode`) |
| Open world, flat | 1k–1M visible objects | Flat spatial grid or quadtree |
| Open world, 3D | 1k–1M visible objects | Custom octree |
| Triangle-level queries (raycasting, distance) | Any mesh | three-mesh-bvh (see `bvh-raycasting.md`) |
| Infinite procedural world | N/A | Chunk manager with async streaming |
| Mostly static, occasional camera pan | Any | Per-frame BVH not needed — frustum culling + LOD sufficient |

---

## Performance Notes

- Re-insert objects into a spatial structure only when they move. For static worlds, build once at load time.
- Spatial structures accelerate query time but their own traversal has a minimum cost. For fewer than ~200 objects, a flat array is often faster than tree traversal.
- Combine structures: use a quadtree for coarse chunk selection, then Three.js Octree for triangle-level collision within the active chunk.
- Profile with `performance.now()` brackets around your query calls before deciding a structure is necessary — premature optimization is common here.
