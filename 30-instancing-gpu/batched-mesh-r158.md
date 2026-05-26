# THREE.BatchedMesh (r158+)

> Sources: Three.js r184 `src/objects/BatchedMesh.js`, official docs `threejs.org/docs/pages/BatchedMesh.html`, `webgl_mesh_batch.html` example, r184 release notes
> Accessed: 2026-05-26

---

## 1. What BatchedMesh Solves

`InstancedMesh` renders **one geometry** N times in a single draw call. The limitation: all instances must be the same shape.

`THREE.BatchedMesh` (added in r158, significantly expanded since) renders **multiple geometries** in a single draw call. It is the Three.js abstraction over the `WEBGL_multi_draw` extension (or its emulated fallback). Each instance can reference a different geometry while sharing one material.

Use `BatchedMesh` when:
- You have a large scene of objects with different shapes but the same material (terrain chunks, building components, vegetation variety).
- You want to avoid one draw call per unique object type without maintaining N separate `InstancedMesh` pools.

Do **not** use it when all instances are the same geometry — `InstancedMesh` is simpler and has lower CPU overhead for that case.

---

## 2. Constructor

```js
new THREE.BatchedMesh(
  maxInstanceCount,  // integer — maximum number of instances ever added
  maxVertexCount,    // integer — total vertex budget across all unique geometries
  maxIndexCount,     // integer — total index budget (default: maxVertexCount * 2)
  material           // Material | Material[]
)
```

All buffers are allocated up-front at construction. There is no dynamic growth; if you exceed `maxInstanceCount` or vertex/index budgets, Three.js throws. Size conservatively and call `optimize()` after bulk deletions to reclaim space.

### Sizing example

```js
// 3 geometries: box (24v/36i), sphere (~500v/~1500i), cone (~200v/~600i)
// 500 instances total
const bm = new THREE.BatchedMesh(
  500,                // maxInstanceCount
  500 * 500,          // maxVertexCount — generous estimate for all unique geos
  500 * 1500,         // maxIndexCount
  new THREE.MeshStandardMaterial({ vertexColors: true })
);
scene.add(bm);
```

---

## 3. Geometry Management

### `addGeometry(geometry, reservedVertexCount?, reservedIndexCount?)` → `number`

Uploads a geometry's position/normal/uv/index data into the shared buffer pool. Returns a **geometry ID** integer used to create instances.

```js
const boxId    = bm.addGeometry(new THREE.BoxGeometry(1, 1, 1));
const sphereId = bm.addGeometry(new THREE.SphereGeometry(0.6, 16, 8));
const coneId   = bm.addGeometry(new THREE.ConeGeometry(0.5, 1.5));
```

`reservedVertexCount` and `reservedIndexCount` let you pre-allocate extra space for future geometry replacement via `setGeometryAt`. If omitted, only the exact vertex/index count of `geometry` is reserved.

### `setGeometryAt(geometryId, geometry)` → `number`

Replaces geometry in-place. The replacement must fit within the reserved vertex/index count from `addGeometry`. All instances using `geometryId` switch to the new geometry immediately.

### `deleteGeometry(geometryId)` → `BatchedMesh`

Marks geometry slots as unused. Instances referencing this geometry are also deleted. The space is reclaimed only after calling `optimize()`.

### `getGeometryRangeAt(geometryId, target?)` → `{start, count, ...}`

Returns the draw range (start index and triangle count) for a geometry. Useful for debugging or custom render passes.

### `getBoundingBoxAt(geometryId, target)` / `getBoundingSphereAt(geometryId, target)`

Per-geometry bounds, not per-instance. Returns `null` if geometry not found.

---

## 4. Instance Management

### `addInstance(geometryId)` → `number`

Creates one instance referencing `geometryId`. Returns an **instance ID** integer used for all subsequent per-instance operations. Starts with an identity matrix and no color (white).

```js
const ids = [];
for (let i = 0; i < 200; i++) {
  const geoId = [boxId, sphereId, coneId][i % 3];
  ids.push(bm.addInstance(geoId));
}
```

### `deleteInstance(instanceId)` → `BatchedMesh`

Marks instance as deleted. Slot is reused by future `addInstance` calls. Call `optimize()` after bulk deletions to compact the buffer.

### `setInstanceCount(maxInstanceCount)` → `void`

Resizes the instance buffer. Use when you need more instances than originally allocated; this reallocates internal buffers.

### `getGeometryIdAt(instanceId)` / `setGeometryIdAt(instanceId, geometryId)`

Read or reassign which geometry an instance uses. Changing geometry at runtime is valid — the instance immediately draws the new shape.

---

## 5. Transforms and Colors

### `setMatrixAt(instanceId, matrix)` → `BatchedMesh`

Sets the local transformation matrix for instance `instanceId`. Internally writes to a texture (`batchingTexture`) sampled by the vertex shader, not a flat `Float32Array` like `InstancedMesh`.

```js
const dummy = new THREE.Object3D();
ids.forEach((id, i) => {
  dummy.position.set(
    (Math.random() - 0.5) * 20,
    (Math.random() - 0.5) * 20,
    (Math.random() - 0.5) * 20,
  );
  dummy.rotation.set(
    Math.random() * Math.PI,
    Math.random() * Math.PI,
    0,
  );
  dummy.updateMatrix();
  bm.setMatrixAt(id, dummy.matrix);
});
```

**Note from r184 docs:** Negatively scaled matrices are not supported.

### `getMatrixAt(instanceId, matrix)` → `Matrix4`

Reads the matrix back from the internal texture into an existing `Matrix4`.

```js
const m = new THREE.Matrix4();
bm.getMatrixAt(instanceId, m);
m.multiply(rotationDelta);
bm.setMatrixAt(instanceId, m);
```

No `needsUpdate` flag needed — `BatchedMesh` marks its textures dirty internally on each `set*` call.

### `setColorAt(instanceId, color)` → `BatchedMesh`

```js
bm.setColorAt(instanceId, new THREE.Color(0xff0000));
```

Accepts `THREE.Color` or `THREE.Vector4` (for alpha). Color is stored in a separate `batchingColorTexture` and read in the fragment shader via the `getBatchingColor` helper (see shader internals below).

### `getColorAt(instanceId, color)` → `Color | Vector4`

Reads color back. Returns white if colors were never set. (A bug where this threw when colors were unset was fixed in r184 — issue #33079.)

### `setVisibleAt(instanceId, visible)` / `getVisibleAt(instanceId)` → `boolean`

Toggle per-instance visibility. Hidden instances are skipped in the draw lists without needing deletion.

---

## 6. Sorting and Culling

```js
bm.perObjectFrustumCulled = true;  // default — cull invisible instances
bm.sortObjects = true;             // default — sort back-to-front (opaque: front-to-back)

// Custom sort (e.g. radix sort for transparent objects):
bm.setCustomSort((instances, camera) => {
  instances.sort((a, b) => b.z - a.z); // sort by depth descending
});
```

For fully dynamic objects or when manual depth control is required, set `sortObjects = false` and handle ordering yourself.

---

## 7. Shader Internals — `batching_pars_vertex.glsl` (r184 source, verbatim)

```glsl
#ifdef USE_BATCHING
  #if ! defined( GL_ANGLE_multi_draw )
  #define gl_DrawID _gl_DrawID
  uniform int _gl_DrawID;
  #endif

  uniform highp sampler2D batchingTexture;
  uniform highp usampler2D batchingIdTexture;

  mat4 getBatchingMatrix( const in float i ) {
    int size = textureSize( batchingTexture, 0 ).x;
    int j = int( i ) * 4;
    int x = j % size;
    int y = j / size;
    vec4 v1 = texelFetch( batchingTexture, ivec2( x,     y ), 0 );
    vec4 v2 = texelFetch( batchingTexture, ivec2( x + 1, y ), 0 );
    vec4 v3 = texelFetch( batchingTexture, ivec2( x + 2, y ), 0 );
    vec4 v4 = texelFetch( batchingTexture, ivec2( x + 3, y ), 0 );
    return mat4( v1, v2, v3, v4 );
  }

  float getIndirectIndex( const in int i ) {
    int size = textureSize( batchingIdTexture, 0 ).x;
    int x = i % size;
    int y = i / size;
    return float( texelFetch( batchingIdTexture, ivec2( x, y ), 0 ).r );
  }
#endif

#ifdef USE_BATCHING_COLOR
  uniform sampler2D batchingColorTexture;
  vec4 getBatchingColor( const in float i ) {
    int size = textureSize( batchingColorTexture, 0 ).x;
    int j = int( i );
    int x = j % size;
    int y = j / size;
    return texelFetch( batchingColorTexture, ivec2( x, y ), 0 );
  }
#endif
```

Key observations:
- Transforms are stored in a float texture, not a per-vertex attribute. This is the fundamental difference from `InstancedMesh`.
- `gl_DrawID` (from `WEBGL_multi_draw` or its emulated fallback via a `_gl_DrawID` uniform) identifies which draw call is active, used to look up the indirect index.
- Color is a separate texture, enabling alpha (RGBA) per instance.

### WEBGL_multi_draw Extension

If the browser supports `WEBGL_multi_draw`, Three.js uses `gl.multiDrawElementsWEBGL` or `gl.multiDrawArraysWEBGL` — a single WebGL call that issues multiple draw commands with a parameter array, reducing CPU overhead further.

From r184 release notes (issue #33238):
> "Implement WEBGL_multi_draw fallback" — when the extension is absent, Three.js emulates it by looping over draw calls and injecting `_gl_DrawID` as a uniform per call.

The application code is identical either way; Three.js picks the path automatically.

---

## 8. Optimization and Maintenance

### `optimize()` → `BatchedMesh`

After deletions, call `optimize()` to compact the vertex/index pool and reclaim fragmented space. This is a CPU-side operation that repacks the buffer data and is not free — call it between scenes or after bulk removals, not every frame.

```js
// Bulk-delete 100 instances, then compact
for (const id of removedIds) bm.deleteInstance(id);
bm.optimize();
```

### `setGeometrySize(maxVertexCount, maxIndexCount)` / `setInstanceCount(maxInstanceCount)`

Resize the internal buffers at runtime. Both cause a buffer reallocation. Use sparingly.

---

## 9. BatchedMesh vs. InstancedMesh Decision Table

| Factor | InstancedMesh | BatchedMesh |
|--------|--------------|-------------|
| Geometry variety | 1 geometry | Many geometries |
| Transform storage | `Float32Array` attribute | Float texture + `gl_DrawID` |
| `needsUpdate` flag | Manual (`instanceMatrix.needsUpdate`) | Automatic on `set*` calls |
| Visibility per instance | Not built-in (reduce `count`) | `setVisibleAt` |
| Custom per-instance attributes | `InstancedBufferAttribute` on geometry | Not directly supported (use custom shaders) |
| Raycasting | `hits[0].instanceId` | `hits[0].instanceId` |
| Shader transparency sort | Manual | `sortObjects = true` handles it |
| Memory overhead | Lower (flat arrays) | Higher (textures + indirect tables) |
| Available since | r118 | r158 |
| Negative scale support | Yes (with normal caveat) | No |

---

## 10. Complete Working Example

```js
import * as THREE from 'three';

// --- Setup ---
const renderer = new THREE.WebGLRenderer({ antialias: true });
const scene    = new THREE.Scene();
const camera   = new THREE.PerspectiveCamera(60, innerWidth / innerHeight, 0.1, 1000);
camera.position.z = 30;

// --- BatchedMesh ---
const material = new THREE.MeshStandardMaterial({ vertexColors: true });
const bm = new THREE.BatchedMesh(300, 50_000, 100_000, material);
scene.add(bm);

// Register geometries
const geoIds = [
  bm.addGeometry(new THREE.ConeGeometry(1, 2, 8)),
  bm.addGeometry(new THREE.BoxGeometry(2, 2, 2)),
  bm.addGeometry(new THREE.SphereGeometry(1, 16, 8)),
];

// Create instances
const dummy   = new THREE.Object3D();
const color   = new THREE.Color();
const rotSpeeds = [];

for (let i = 0; i < 300; i++) {
  const geoId = geoIds[i % 3];
  const iid   = bm.addInstance(geoId);

  dummy.position.set(
    (Math.random() - 0.5) * 40,
    (Math.random() - 0.5) * 40,
    (Math.random() - 0.5) * 40,
  );
  dummy.rotation.set(
    Math.random() * Math.PI,
    Math.random() * Math.PI,
    0,
  );
  dummy.updateMatrix();
  bm.setMatrixAt(iid, dummy.matrix);

  color.setHSL(i / 300, 1, 0.5);
  bm.setColorAt(iid, color);

  rotSpeeds.push(new THREE.Matrix4().makeRotationY(0.01 * (Math.random() - 0.5)));
}

// --- Animation ---
const m = new THREE.Matrix4();

function animate() {
  requestAnimationFrame(animate);

  for (let i = 0; i < 300; i++) {
    bm.getMatrixAt(i, m);
    m.multiply(rotSpeeds[i]);
    bm.setMatrixAt(i, m);
  }

  renderer.render(scene, camera);
}
animate();
```

---

## Sources

```yaml
- url: https://raw.githubusercontent.com/mrdoob/three.js/dev/src/objects/BatchedMesh.js
  tier: 1
  accessed_at_utc: "2026-05-26T00:00:00Z"
  notes: "Full method signatures, property list, sorting/culling implementation"

- url: https://threejs.org/docs/pages/BatchedMesh.html
  tier: 1
  accessed_at_utc: "2026-05-26T00:00:00Z"
  notes: "Constructor params, all method signatures with descriptions, usage example"

- url: https://raw.githubusercontent.com/mrdoob/three.js/dev/examples/webgl_mesh_batch.html
  tier: 1
  accessed_at_utc: "2026-05-26T00:00:00Z"
  notes: "Heterogeneous geometry batching pattern, radix sort, frustum culling toggle"

- url: https://github.com/mrdoob/three.js/releases/tag/r184
  tier: 1
  accessed_at_utc: "2026-05-26T00:00:00Z"
  notes: "WEBGL_multi_draw fallback (#33238), getColorAt fix (#33079), deprecated paths removed (#33234)"

- url: https://github.com/mrdoob/three.js/releases/tag/r158
  tier: 1
  notes: "BatchedMesh first introduced"

- url: https://tympanus.net/codrops/2024/10/30/interactive-3d-with-three-js-batchedmesh-and-webgpurenderer/
  tier: 3
  accessed_at_utc: "2026-05-26T00:00:00Z"
  notes: "Real-world usage: sortObjects=false for performance, perObjectFrustumCulled flag"
```
