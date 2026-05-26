# InstancedMesh Deep Dive

> Source: Three.js r184 source `src/objects/InstancedMesh.js`, official docs, example `webgl_instancing_dynamic.html`
> Accessed: 2026-05-26

---

## 1. What InstancedMesh Is

`THREE.InstancedMesh` renders `N` copies of one `BufferGeometry` in a **single draw call**, with per-instance transform (and optionally color) stored in GPU buffers. It extends `Mesh`, so it inherits scene-graph placement, frustum culling (whole-mesh AABB), and shadow support.

Use it when:
- You have 50–1,000,000+ copies of the same geometry.
- Each copy needs an independent transform (position, rotation, scale).
- Optionally each copy needs an independent base color multiplier.

Do **not** use it when copies need different geometries (use `BatchedMesh`) or different materials.

---

## 2. Constructor

```js
const mesh = new THREE.InstancedMesh(
  geometry,   // BufferGeometry — shared across all instances
  material,   // Material — shared across all instances
  count       // integer — number of instances to allocate
);
```

Internally the constructor allocates:
```js
this.instanceMatrix = new InstancedBufferAttribute(
  new Float32Array(count * 16), 16
);
```
and initialises every slot to the identity matrix via a `setMatrixAt` loop. `instanceColor` starts as `null` and is only allocated on the first `setColorAt` call.

---

## 3. Core Properties

### `instanceMatrix` — `InstancedBufferAttribute`

Flat `Float32Array` of length `count * 16`. Each 16-float block is a column-major `Matrix4`.

```
instance 0: indices  0–15
instance 1: indices 16–31
instance N: indices N*16 … N*16+15
```

After writing one or more matrices, set:
```js
mesh.instanceMatrix.needsUpdate = true;
```

For meshes updated every frame, mark the buffer as dynamic at creation time so the driver hints for streaming:
```js
mesh.instanceMatrix.setUsage(THREE.DynamicDrawUsage);
```

### `instanceColor` — `InstancedBufferAttribute | null`

`null` by default. Allocated automatically as `Float32Array(count * 3)` on the first call to `setColorAt`. Stores linear-space RGB per instance. After writes:
```js
mesh.instanceColor.needsUpdate = true;
```

Requires the material to have `vertexColors: true` **or** the shader to consume `instanceColor`. The built-in chunk `USE_INSTANCING_COLOR` multiplies `instanceColor.rgb` into `vColor`.

### `count` — `integer`

The number of instances **currently rendered**. You can reduce `count` at runtime to hide trailing instances without reallocating:
```js
mesh.count = 500; // only first 500 instances drawn
```

Setting `count` above the originally allocated size is illegal and produces undefined behaviour. To grow the pool, replace `instanceMatrix` with a new, larger `InstancedBufferAttribute`.

### `maxInstanceCount` — `integer | null`

In the r184 source `maxInstanceCount` is **not** a first-class property on `InstancedMesh`; the allocation ceiling is the initial `count` argument stored implicitly as `instanceMatrix.count`. Some community wrappers expose `maxInstanceCount` explicitly — do not rely on it in core.

### `previousInstanceMatrix` — `InstancedBufferAttribute | null`

Used by motion-blur post-processing passes (e.g. with `WebGPURenderer`). Set to `null` by default. If non-null, the renderer stores the previous frame's matrices here for velocity computation.

### `morphTexture` — `DataTexture | null`

Stores per-instance morph-target weights when using `setMorphAt`. Allocated lazily as a `Float32Array` texture of size `(numMorphTargets+1) × count`.

### `boundingBox` / `boundingSphere`

Not computed by default. Call `mesh.computeBoundingBox()` / `mesh.computeBoundingSphere()` explicitly. Both iterate all instances, apply their matrices to the geometry bounds, and union the results. Costly — cache the result if your instances don't change shape.

---

## 4. Methods

### `setMatrixAt(index, matrix)` → `this`

Writes the 16 floats of `matrix` into `instanceMatrix.array` at offset `index * 16`.

```js
const dummy = new THREE.Object3D();
dummy.position.set(x, y, z);
dummy.rotation.y = angle;
dummy.scale.setScalar(s);
dummy.updateMatrix();
mesh.setMatrixAt(i, dummy.matrix);
```

`dummy.matrix` is a `Matrix4`. Calling `dummy.updateMatrix()` is required first because `Object3D` doesn't auto-update its matrix when properties change.

After the loop:
```js
mesh.instanceMatrix.needsUpdate = true;
```

### `getMatrixAt(index, matrix)` → `matrix`

Reads 16 floats back into an existing `Matrix4`. Non-destructive if `matrix` is pre-allocated:

```js
const m = new THREE.Matrix4();
mesh.getMatrixAt(i, m);
// m now holds instance i's transform
```

### `setColorAt(index, color)` → `this`

Allocates `instanceColor` on first call (fills with ones — white). Writes `color.r/g/b` at `index * 3`.

```js
const c = new THREE.Color(0xff4400);
mesh.setColorAt(i, c);
mesh.instanceColor.needsUpdate = true;
```

### `getColorAt(index, color)` → `color`

If `instanceColor` is null (never set), returns white. Otherwise reads from the buffer:

```js
const c = new THREE.Color();
mesh.getColorAt(i, c); // populates c
```

### `setMorphAt(index, object)` / `getMorphAt(index, object)`

Copies `object.morphTargetInfluences` into `morphTexture` for instance `index`. Inverse is `getMorphAt`. Requires the geometry to have morph targets.

### `raycast(raycaster, intersects)`

Built-in raycast iterates all `count` instances. For each:
1. Retrieves the instance matrix.
2. Constructs an inverse world matrix.
3. Transforms the ray into local space.
4. Delegates to geometry-level intersection tests.

Intersection objects carry `.instanceId` (integer index):
```js
const hits = raycaster.intersectObject(mesh);
if (hits.length) {
  console.log(hits[0].instanceId); // which instance was hit
}
```

This is O(N×triangles). For > ~50,000 instances use a BVH (see `gpu-picking.md`).

### `computeBoundingBox()` / `computeBoundingSphere()`

See note under properties above. Call once after static placement; call again only when instances move.

### `dispose()`

Dispatches `dispose` event, disposes `morphTexture` if present. Does **not** dispose `geometry` or `material` — you must do that separately.

### `copy(source, recursive)`

Deep-copies all `InstancedMesh`-specific properties including `instanceMatrix`, `instanceColor`, `morphTexture`, and bounding volumes. Returns `this`.

---

## 5. Batch Update Pattern (Animation Loop)

```js
const dummy = new THREE.Object3D();
const color  = new THREE.Color();
let colorNeedsUpdate = false;

function tick(t) {
  for (let i = 0; i < mesh.count; i++) {
    // reuse dummy to avoid per-frame allocation
    dummy.position.x = Math.sin(t + i) * 5;
    dummy.updateMatrix();
    mesh.setMatrixAt(i, dummy.matrix);

    if (someCondition) {
      color.setHSL(i / mesh.count, 1, 0.5);
      mesh.setColorAt(i, color);
      colorNeedsUpdate = true;
    }
  }

  mesh.instanceMatrix.needsUpdate = true;
  if (colorNeedsUpdate) {
    mesh.instanceColor.needsUpdate = true;
    colorNeedsUpdate = false;
  }
}
```

Key rules:
- Set `needsUpdate = true` **after** all writes, not inside the loop.
- Only set `instanceColor.needsUpdate` when colors actually changed; uploading an unchanged buffer wastes bandwidth.
- `setUsage(THREE.DynamicDrawUsage)` before the first render if the buffer changes every frame.

---

## 6. `count` Manipulation for Pool-Style Rendering

```js
// Allocate maximum at creation
const MAX = 10_000;
const mesh = new THREE.InstancedMesh(geo, mat, MAX);

// Use only active portion
mesh.count = activeCount; // 0 … MAX

// "Kill" instance i without recompacting: swap with last active
mesh.getMatrixAt(activeCount - 1, tmpMatrix);
mesh.setMatrixAt(i, tmpMatrix);
activeCount--;
mesh.count = activeCount;
mesh.instanceMatrix.needsUpdate = true;
```

---

## 7. Custom Per-Instance Attributes

`InstancedMesh` stores matrices and colors, but nothing else. For arbitrary per-instance data (scale variant, UV offset, health, animation phase) you add `InstancedBufferAttribute` to the underlying geometry:

```js
// 1 float per instance (e.g. animation phase)
const phases = new Float32Array(COUNT);
for (let i = 0; i < COUNT; i++) phases[i] = Math.random() * Math.PI * 2;

const phaseAttr = new THREE.InstancedBufferAttribute(phases, 1);
mesh.geometry.setAttribute('aPhase', phaseAttr);
```

Access in the vertex shader (with `ShaderMaterial` or `onBeforeCompile`):

```glsl
attribute float aPhase;   // declared once per instance by the instancing mechanism

void main() {
  float wave = sin(uTime + aPhase) * 0.5;
  vec3 pos = position + normal * wave;
  gl_Position = projectionMatrix * modelViewMatrix * vec4(pos, 1.0);
}
```

See `custom-instance-attributes.md` for the full pipeline.

---

## 8. What the Built-in Shaders Inject

Three.js's `WebGLProgram` builder conditionally inserts GLSL when it detects `isInstancedMesh`:

- `#define USE_INSTANCING` — gates the `instanceMatrix` attribute injection.
- `#define USE_INSTANCING_COLOR` — gates `instanceColor` attribute injection (set when `instanceColor !== null`).

From `color_vertex.glsl`:
```glsl
#ifdef USE_INSTANCING_COLOR
  vColor.rgb *= instanceColor.rgb;
#endif
```

The `instanceMatrix` attribute is declared as four `vec4` rows internally, assembled into a `mat4` within the vertex shader.

---

## 9. Known Gotchas

| Gotcha | Detail |
|--------|--------|
| Frustum culling is whole-mesh | The single AABB covers all instances; individual culling requires a BVH or manual solution. |
| Shadow map also iterates all instances | `count` reduction helps, but each shadow cascade re-traverses. |
| `raycast` is O(N×triangles) | Unusable past ~50k instances without BVH; see `gpu-picking.md`. |
| Negative scale in matrix | Technically works but may flip normals; no automatic correction. |
| `instanceColor` is RGB-only | Alpha per-instance is not built in; add a custom attribute for transparency masks. |
| `dispose()` does not free geometry/material | Call `mesh.geometry.dispose()` and `mesh.material.dispose()` separately. |

---

## Sources

```yaml
- url: https://raw.githubusercontent.com/mrdoob/three.js/dev/src/objects/InstancedMesh.js
  tier: 1
  accessed_at_utc: "2026-05-26T00:00:00Z"
  notes: "r184 dev branch source — constructor, all methods, exact property names"

- url: https://threejs.org/docs/#api/en/objects/InstancedMesh
  tier: 1
  accessed_at_utc: "2026-05-26T00:00:00Z"

- url: https://raw.githubusercontent.com/mrdoob/three.js/dev/examples/webgl_instancing_dynamic.html
  tier: 1
  accessed_at_utc: "2026-05-26T00:00:00Z"
  notes: "DynamicDrawUsage pattern, needsUpdate pattern, setColorAt in loop"

- url: https://github.com/mrdoob/three.js/releases/tag/r184
  tier: 1
  accessed_at_utc: "2026-05-26T00:00:00Z"
  notes: "r184 release notes: consistent return values on InstancedMesh methods"
```
