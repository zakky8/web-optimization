# GPU Picking for Instanced Meshes

> Sources: Three.js r184, `webgl_instancing_raycast.html` example, GPU picking gist (duhaime), three.js forum "Best way to do Instanced Mesh picking in 2024", three-mesh-bvh README
> Accessed: 2026-05-26

---

## 1. The Three Approaches

| Approach | Complexity | Accuracy | Instance count sweet spot |
|----------|-----------|----------|--------------------------|
| Built-in `raycast` | Low | Exact geometry intersection | < 50k instances, geometry < ~10k triangles |
| BVH-accelerated raycast | Medium | Exact geometry intersection | 50k–500k instances with static/infrequent updates |
| GPU color picking (offscreen render) | Medium–High | Pixel-exact, GPU-side | > 100k instances, or shader-animated geometry |

Choose based on your instance count, whether the transforms are updated on the CPU or GPU, and whether you need per-pixel accuracy vs. bounding-box-level selection.

---

## 2. Built-in Raycasting (Baseline)

Three.js's `InstancedMesh.raycast` iterates all `mesh.count` instances, transforms the ray into each instance's local space, and runs a geometry-level triangle test. The result carries `.instanceId`.

```js
const raycaster = new THREE.Raycaster();
const mouse = new THREE.Vector2();

window.addEventListener('pointermove', (e) => {
  mouse.x =  (e.clientX / window.innerWidth)  * 2 - 1;
  mouse.y = -(e.clientY / window.innerHeight) * 2 + 1;
});

function tick() {
  raycaster.setFromCamera(mouse, camera);
  const hits = raycaster.intersectObject(mesh);

  if (hits.length > 0) {
    const { instanceId, point, distance } = hits[0];
    // instanceId is the integer index into the InstancedMesh
    highlightInstance(instanceId);
  }
}
```

Complexity: **O(N × triangles)**. Fine for small counts; degrades quickly above ~50k.

**Critical limitation:** If you animate positions in a vertex shader (GPU-side), the CPU raycast operates on the original, un-animated geometry positions. The result will be wrong. Switch to GPU color picking in that case.

---

## 3. GPU Color Picking (ID Encoding)

### Concept

Render the instanced mesh to a separate offscreen `WebGLRenderTarget` using a custom material that writes a unique color per instance instead of shading. On click/hover, read the pixel under the cursor — the RGB value decodes back to an instance ID. Supports up to 16,777,215 instances with RGB24 encoding.

### 3.1 — ID-to-Color Encoding

With three 8-bit channels you get 24 bits = 16M unique IDs. Index 0 is reserved for "no object" (black background).

```js
function idToColor(id) {
  // id is 1-based so background (0,0,0) means "miss"
  return new THREE.Color(
    ((id >> 16) & 0xff) / 255,
    ((id >>  8) & 0xff) / 255,
    ( id        & 0xff) / 255,
  );
}

function colorToId(r, g, b) {
  return (r << 16) | (g << 8) | b;
}
```

### 3.2 — Picking Material

Build one `ShaderMaterial` that writes the instance's encoded color. The instance ID is available in the vertex shader via the `gl_InstanceID` built-in (WebGL2) and passed to the fragment shader as a flat varying:

```js
const pickingMaterial = new THREE.ShaderMaterial({
  vertexShader: /* glsl */`
    flat out highp int vInstanceId;

    void main() {
      vInstanceId = gl_InstanceID + 1;  // 1-based; 0 = background
      gl_Position = projectionMatrix * modelViewMatrix * instanceMatrix * vec4(position, 1.0);
    }
  `,

  fragmentShader: /* glsl */`
    flat in highp int vInstanceId;
    out vec4 outColor;

    void main() {
      // Pack 24-bit id into RGB
      int id = vInstanceId;
      outColor = vec4(
        float((id >> 16) & 0xff) / 255.0,
        float((id >>  8) & 0xff) / 255.0,
        float( id        & 0xff) / 255.0,
        1.0
      );
    }
  `,

  // Disable features that affect depth testing
  depthWrite: true,
  depthTest:  true,
});
```

If you need to keep the real material for rendering, swap materials before/after the picking render:

```js
const realMaterial = mesh.material;
```

### 3.3 — Offscreen Render Target

```js
// Create once
const pickingTarget = new THREE.WebGLRenderTarget(
  renderer.domElement.width,
  renderer.domElement.height,
  {
    minFilter: THREE.NearestFilter,  // no interpolation — exact ID reads
    magFilter: THREE.NearestFilter,
    format: THREE.RGBAFormat,
  }
);

// Pixel buffer reused every pick
const pixelBuffer = new Uint8Array(4);
```

### 3.4 — Picking on Click

```js
renderer.domElement.addEventListener('click', (e) => {
  // Swap to picking material
  mesh.material = pickingMaterial;

  renderer.setRenderTarget(pickingTarget);
  renderer.render(scene, camera);
  renderer.setRenderTarget(null);

  // Restore real material
  mesh.material = realMaterial;

  // Read single pixel at cursor
  const x = Math.floor(e.clientX * window.devicePixelRatio);
  // WebGL Y-axis is flipped vs. screen
  const y = Math.floor(
    pickingTarget.height - e.clientY * window.devicePixelRatio
  );

  renderer.readRenderTargetPixels(pickingTarget, x, y, 1, 1, pixelBuffer);

  const id = colorToId(pixelBuffer[0], pixelBuffer[1], pixelBuffer[2]);

  if (id > 0) {
    const instanceId = id - 1;   // back to 0-based
    console.log('Picked instance', instanceId);
  }
});
```

`readRenderTargetPixels` signature:
```
renderer.readRenderTargetPixels(
  renderTarget,  // WebGLRenderTarget
  x, y,          // pixel coords (origin = bottom-left in WebGL)
  width, height, // read region size
  buffer         // Uint8Array or Uint8ClampedArray
)
```

### 3.5 — Performance Notes

- Read **1×1 pixel** on demand (click) rather than the full buffer each frame. Full-buffer reads stall the GPU pipeline.
- For hover highlighting, consider reading once per frame but only at the mouse position.
- For WebGPU contexts use async `readRenderTargetPixelsAsync` to avoid pipeline stalls entirely.
- Keep `pickingTarget` the same size as the canvas, or scale mouse coordinates if you use a smaller target (e.g. half-resolution for speed).
- If multiple meshes need picking, assign ID ranges or use separate render passes per mesh.

### 3.6 — Resize Handling

```js
window.addEventListener('resize', () => {
  pickingTarget.setSize(
    renderer.domElement.width,
    renderer.domElement.height
  );
});
```

---

## 4. BVH-Accelerated Raycasting (`three-mesh-bvh`)

For scenarios where geometry is static (or infrequently updated) and you want exact triangle intersection without the full GPU picking pipeline.

### Installation

```bash
npm install three-mesh-bvh
```

### Setup

```js
import {
  computeBoundsTree,
  disposeBoundsTree,
  acceleratedRaycast,
  MeshBVHHelper,
} from 'three-mesh-bvh';

// Extend THREE prototypes once at app startup
THREE.BufferGeometry.prototype.computeBoundsTree = computeBoundsTree;
THREE.BufferGeometry.prototype.disposeBoundsTree = disposeBoundsTree;
THREE.Mesh.prototype.raycast = acceleratedRaycast;
```

### Building the BVH

```js
// After creating geometry, build the BVH
const geo = new THREE.IcosahedronGeometry(1, 4);
geo.computeBoundsTree({
  strategy: 0,           // SAH = 0 (default, best quality), CENTER = 1, AVERAGE = 2
  maxLeafTris: 5,        // triangles per leaf node
  maxDepth: 40,          // tree depth cap
});

const mesh = new THREE.InstancedMesh(geo, mat, COUNT);
```

### Raycasting With BVH

```js
// After patching Mesh.prototype.raycast, intersectObject uses BVH automatically
const hits = raycaster.intersectObject(mesh);
// hits[0].instanceId is still populated correctly
```

### Important Limitation (Verified from three.js forum, 2024)

> "three-mesh-bvh works for instanced meshes in that it will perform BVH-based raycasting on **each individual geometry** in the instanced mesh — but it does **not** build a BVH over the set of instanced objects."

This means BVH accelerates the **per-instance triangle test**, but still iterates all `N` instances to check which ones the ray hits. For very large instance counts (> 500k), GPU color picking is still superior.

For a BVH over the instance set itself, the `InstancedMesh2` library (agargaro) builds a spatial index over instance positions and is designed for exactly this use case.

### Cleanup

```js
geo.disposeBoundsTree();
```

Call before disposing or replacing geometry to free the BVH heap.

---

## 5. Comparison Matrix

| Factor | Built-in raycast | BVH raycast | GPU color pick |
|--------|-----------------|-------------|----------------|
| Exact triangle hit | Yes | Yes | No (pixel raster) |
| Works with shader-animated positions | No | No | Yes |
| 10k instances @ 30fps | Yes | Yes | Yes |
| 500k instances | No (slow) | Marginal | Yes |
| Multiple meshes | Extra intersectObjects call | Same | Single render pass |
| instanceId in result | `hits[0].instanceId` | `hits[0].instanceId` | Decoded from pixel |
| Setup cost | None | Build BVH once | Render target + material |
| GPU stall risk | None | None | Yes (synchronous readPixels) |

---

## 6. Hover with Minimal Stall

Read one pixel per `requestAnimationFrame` rather than per `mousemove` event to limit GPU roundtrips:

```js
let pendingPickX = -1, pendingPickY = -1;

window.addEventListener('mousemove', (e) => {
  pendingPickX = Math.floor(e.clientX * dpr);
  pendingPickY = Math.floor(pickingTarget.height - e.clientY * dpr);
});

function tick() {
  renderer.render(scene, camera);

  if (pendingPickX >= 0) {
    // Picking render happens after main render — no double-clear needed
    mesh.material = pickingMaterial;
    renderer.setRenderTarget(pickingTarget);
    renderer.render(scene, camera);
    renderer.setRenderTarget(null);
    mesh.material = realMaterial;

    renderer.readRenderTargetPixels(
      pickingTarget, pendingPickX, pendingPickY, 1, 1, pixelBuffer
    );

    const hoveredId = colorToId(pixelBuffer[0], pixelBuffer[1], pixelBuffer[2]) - 1;
    updateHighlight(hoveredId);

    pendingPickX = -1;
  }
}
```

---

## Sources

```yaml
- url: https://raw.githubusercontent.com/mrdoob/three.js/dev/examples/webgl_instancing_raycast.html
  tier: 1
  accessed_at_utc: "2026-05-26T00:00:00Z"
  notes: "Official example: raycaster.intersectObject, hits[0].instanceId, setColorAt highlight"

- url: https://gist.github.com/duhaime/1eafa293e7ce16b074a6d55cac67badc
  tier: 3
  accessed_at_utc: "2026-05-26T00:00:00Z"
  notes: "RGB24 encoding pattern, readRenderTargetPixels with devicePixelRatio, Y-flip"

- url: https://discourse.threejs.org/t/best-way-to-do-instanced-mesh-picking-in-2024/59917
  tier: 4
  accessed_at_utc: "2026-05-26T00:00:00Z"
  notes: "BVH limitation quote: geometry-level not instance-set-level; async pixel read recommendation"

- url: https://github.com/gkjohnson/three-mesh-bvh
  tier: 3
  accessed_at_utc: "2026-05-26T00:00:00Z"
  notes: "computeBoundsTree, acceleratedRaycast, strategy options"
```
