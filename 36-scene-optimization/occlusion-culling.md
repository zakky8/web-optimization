# Occlusion Culling

**Sources:**
- https://threejs.org/docs/index.html#api/en/math/Frustum (accessed 2026-05-26)
- https://github.com/gkjohnson/three-mesh-bvh (accessed 2026-05-26)
- WebGL2 spec / MDN for EXT_occlusion_query (accessed 2026-05-26)

---

## What Occlusion Culling Solves

Frustum culling removes objects outside the camera's view volume. Occlusion culling goes further: it removes objects that *are* inside the frustum but are fully blocked by other geometry. In a scene where walls, terrain, or large opaque meshes cover significant portions of the screen, occlusion culling can eliminate 50–90% of draw calls in indoor or dense scenes.

There is no built-in occlusion culling in Three.js. Implementing it requires choosing one of the approaches below based on scene type and target platform.

---

## Approach 1: Distance-Based (Chunked) Culling

The simplest form of occlusion approximation. Objects beyond a threshold distance are hidden. Combined with LOD, this is the most common technique in open-world web scenes.

```js
const CULL_DISTANCE_SQ = 200 * 200;  // 200 units

function distanceCull(camera, objects) {
  const camPos = camera.position;

  for (const obj of objects) {
    const dx = obj.position.x - camPos.x;
    const dy = obj.position.y - camPos.y;
    const dz = obj.position.z - camPos.z;
    obj.visible = (dx * dx + dy * dy + dz * dz) < CULL_DISTANCE_SQ;
  }
}
```

### Chunked Distance Culling

For large worlds, divide the world into a spatial grid. Each chunk contains a list of objects. Only test chunks whose bounding box is within the active radius:

```js
class ChunkedWorld {
  constructor(chunkSize = 64) {
    this.chunkSize = chunkSize;
    this.chunks = new Map();  // "x,z" -> Object3D[]
  }

  add(object) {
    const key = this._key(object.position);
    if (!this.chunks.has(key)) this.chunks.set(key, []);
    this.chunks.get(key).push(object);
  }

  update(camera, activeRadius = 3) {
    const cx = Math.floor(camera.position.x / this.chunkSize);
    const cz = Math.floor(camera.position.z / this.chunkSize);

    for (const [key, objects] of this.chunks) {
      const [kx, kz] = key.split(',').map(Number);
      const dist = Math.max(Math.abs(kx - cx), Math.abs(kz - cz));
      const visible = dist <= activeRadius;
      for (const o of objects) o.visible = visible;
    }
  }

  _key(pos) {
    const x = Math.floor(pos.x / this.chunkSize);
    const z = Math.floor(pos.z / this.chunkSize);
    return `${x},${z}`;
  }
}
```

**Best for:** Open worlds, procedural terrain, city scenes where drawing distance is naturally limited.

---

## Approach 2: Hardware Occlusion Queries (WebGL2)

WebGL2 exposes `EXT_occlusion_query` / `EXT_occlusion_query_webgl2`. These allow you to ask the GPU: "How many fragments passed the depth test for this draw call?" If the answer is zero, the object is fully occluded.

### Support Status

**Verify support before relying on this technique.** Extension availability varies by driver and browser.

```js
const gl = renderer.getContext();
const ext = gl.getExtension('EXT_occlusion_query_webgl2')
          || gl.getExtension('EXT_occlusion_query');  // WebGL1 fallback

if (!ext) {
  console.warn('Occlusion queries not supported — falling back to CPU culling');
}
```

As of 2026, `EXT_occlusion_query_webgl2` is broadly supported on desktop (Chrome, Firefox, Edge) via ANGLE on Windows and Mesa on Linux. Mobile support (especially iOS Safari / WebKit) remains inconsistent — **do not assume it is available on all targets without a runtime check.**

### Basic Query Pattern

The fundamental challenge: GPU queries have a **one-to-two frame latency**. You issue a query on frame N and read the result on frame N+1 or N+2. The standard pattern is to use results from the previous frame to cull the current frame.

```js
const queries = new Map();   // Object3D -> WebGLQuery
const results = new Map();   // Object3D -> boolean (last known visibility)
const proxyGeometry = new THREE.BoxGeometry(1, 1, 1);  // cheap proxy mesh

function beginOcclusionQuery(gl, ext, object) {
  if (queries.has(object)) {
    const q = queries.get(object);
    const available = gl.getQueryParameter(q, gl.QUERY_RESULT_AVAILABLE);
    if (available) {
      const passed = gl.getQueryParameter(q, gl.QUERY_RESULT);
      results.set(object, passed > 0);
      gl.deleteQuery(q);
      queries.delete(object);
    }
  }

  if (!queries.has(object)) {
    const q = gl.createQuery();
    queries.set(object, q);
    ext.beginQueryEXT(ext.ANY_SAMPLES_PASSED_CONSERVATIVE_EXT, q);
    // Render a cheap proxy (bounding box) for the object here
    // ...
    ext.endQueryEXT(ext.ANY_SAMPLES_PASSED_CONSERVATIVE_EXT);
  }
}

function renderWithOcclusion(scene, camera, renderer) {
  for (const object of scene.children) {
    // Use last frame's result
    object.visible = results.get(object) ?? true;  // default visible on first frame
    // Issue this frame's query
    beginOcclusionQuery(gl, ext, object);
  }
  renderer.render(scene, camera);
}
```

### ANY_SAMPLES_PASSED vs ANY_SAMPLES_PASSED_CONSERVATIVE

- `ANY_SAMPLES_PASSED_EXT` — precise count; may cause GPU pipeline stalls on some hardware.
- `ANY_SAMPLES_PASSED_CONSERVATIVE_EXT` — the GPU may return "passed" even for fully-occluded objects (conservative = fewer false negatives). **Prefer this for culling to avoid pop-in.**

### Integration with Three.js WebGLRenderer

Three.js does not expose the raw `gl` query state machine through its public API. To use occlusion queries you must:

1. Access the raw WebGL context via `renderer.getContext()`.
2. Either bypass Three.js's render calls for occluder proxy meshes, or use `renderer.renderBufferDirect()` with a minimal render state.
3. This is low-level and fragile across Three.js versions — encapsulate it in a dedicated `OcclusionCuller` class.

---

## Approach 3: Scene Portals

Portal culling divides the scene into *cells* (rooms, corridors) connected by *portals* (doorways, windows). Visibility is computed by projecting portal rectangles through the view frustum:

1. Determine which cell the camera occupies.
2. The camera's frustum is the initial visibility volume.
3. For each portal visible from the current cell, clip the frustum to the portal's screen-space rectangle.
4. Recurse into the connected cell with the clipped frustum.
5. Only objects in visible cells, within the clipped frustum, are rendered.

```js
class Portal {
  constructor(fromCell, toCell, corners) {
    this.fromCell = fromCell;
    this.toCell   = toCell;
    this.corners  = corners;  // 4 x Vector3
    this.plane    = new THREE.Plane().setFromCoplanarPoints(...corners);
  }
}

class Cell {
  constructor(id) {
    this.id      = id;
    this.objects = [];   // Object3D[] in this cell
    this.portals = [];   // Portal[] leading out
  }
}

function gatherVisibleObjects(camera, startCell, frustum) {
  const visible = [];
  const visited = new Set();

  function traverse(cell, activeFrustum) {
    if (visited.has(cell.id)) return;
    visited.add(cell.id);

    for (const obj of cell.objects) {
      if (activeFrustum.intersectsObject(obj)) visible.push(obj);
    }

    for (const portal of cell.portals) {
      // Project portal corners into screen space, build clipped frustum
      // (full implementation requires custom frustum clipping)
      if (isPortalVisible(portal, camera, activeFrustum)) {
        traverse(portal.toCell, clipFrustumToPortal(activeFrustum, portal, camera));
      }
    }
  }

  traverse(startCell, frustum);
  return visible;
}
```

**Best for:** Indoor scenes with discrete rooms (game levels, architectural visualization). Requires manually authored cell/portal data or automatic extraction from BSP-style geometry.

---

## Approach 4: Pre-Computed Visibility Sets (PVS)

A PVS is a lookup table: for each cell in the world, it stores exactly which other cells (and their objects) are potentially visible from anywhere in that cell. The table is computed offline (baked) and streamed at runtime.

### Offline Bake

The bake process typically:
1. Samples the world from a dense grid of viewpoints within each cell.
2. Runs software rasterization or raycasting at each sample point.
3. Records which cells had any visible surface.
4. Takes the union of visible cells across all samples in a cell.
5. Outputs a compressed bitfield per cell.

```
cell_0 can see: [cell_0, cell_1, cell_3, cell_7]
cell_1 can see: [cell_0, cell_1, cell_2, cell_4, cell_5]
...
```

### Runtime Lookup

```js
class PVSManager {
  constructor(pvsTable) {
    // pvsTable: { cellId: Set<cellId> }
    this.table = pvsTable;
  }

  setVisibleFromCell(currentCellId, allCells) {
    const visibleCells = this.table[currentCellId];
    if (!visibleCells) return;

    for (const [id, cell] of allCells) {
      const isVisible = visibleCells.has(id);
      for (const obj of cell.objects) {
        obj.visible = isVisible;
      }
    }
  }
}

// Each frame: determine camera cell, update visibility
const cameraCell = worldGrid.getCellAtPosition(camera.position);
pvsManager.setVisibleFromCell(cameraCell.id, worldGrid.cells);
```

### Tradeoffs

| Approach | Build cost | Runtime cost | Scene type |
|---|---|---|---|
| Distance-based | None | O(n objects) | Open worlds |
| Hardware queries | None | GPU + CPU sync (1–2 frame lag) | Any |
| Portals | Manual authoring | O(portals traversed) | Indoor |
| PVS | High (offline bake) | O(1) lookup + O(objects in visible cells) | Indoor + outdoor |

**PVS** gives the lowest runtime cost but requires a build pipeline. It is what id Software's Quake engine and Unreal Engine's legacy BSP vis system used. For web projects, it is most practical when the world is authored (not procedural) and is large enough that runtime culling costs are measurable.

---

## Software Rasterization Occlusion (CPU-Side)

An alternative to hardware queries is a software depth buffer on the CPU:

1. Rasterize occluder geometry (large opaque meshes: terrain, building walls) at low resolution (e.g., 128×72) using a software rasterizer.
2. Build a hierarchical Z-buffer (depth pyramid) from the result.
3. Project each potential occludee's AABB into screen space and test against the HZB mip.

Libraries like [oc-cull](https://github.com/nicktindall/oc-cull) implement this pattern in JavaScript. The main cost is the software rasterization step — typically 0.5–2ms for a simplified occluder set.

---

## Combining Techniques (Recommended for Production)

For a typical Three.js web scene:

```
1. Frustum culling (built-in, always on)
         |
2. Distance / chunk culling (CPU, free-list chunks)
         |
3. PVS or Portal culling (if indoor / structured world)
         |
4. Hardware occlusion queries (WebGL2 desktop, with fallback)
         |
5. LOD (three-mesh-lod or THREE.LOD) — reduces triangle count, not draw count
```

Apply them in order of increasing complexity. Measure with browser GPU profiling tools before adding the next layer.
