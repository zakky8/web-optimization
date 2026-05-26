# Custom Per-Instance Attributes

> Source: Three.js r184 `src/core/InstancedBufferAttribute.js`, `src/core/InstancedBufferGeometry.js`, medium.com/@pailhead011 instancing series, three.js forum
> Accessed: 2026-05-26

---

## 1. Why You Need This

`InstancedMesh` gives you per-instance transform and one RGB color. Everything else — UV offsets, animation phase, material property overrides, team color, health bar progress, wind strength — requires adding your own `InstancedBufferAttribute` to the geometry and reading it in a custom or patched vertex shader.

---

## 2. `InstancedBufferAttribute`

### Source (r184, verbatim)

```js
// src/core/InstancedBufferAttribute.js
class InstancedBufferAttribute extends BufferAttribute {

  constructor(array, itemSize, normalized, meshPerAttribute = 1) {
    super(array, itemSize, normalized);
    this.isInstancedBufferAttribute = true;
    this.meshPerAttribute = meshPerAttribute;
  }

  copy(source) {
    super.copy(source);
    this.meshPerAttribute = source.meshPerAttribute;
    return this;
  }

  toJSON() {
    const data = super.toJSON();
    data.meshPerAttribute = this.meshPerAttribute;
    data.isInstancedBufferAttribute = true;
    return data;
  }
}
```

### Constructor Parameters

| Param | Type | Notes |
|-------|------|-------|
| `array` | `TypedArray` | `Float32Array`, `Uint8Array`, etc. Length = `instanceCount × itemSize` |
| `itemSize` | `integer` | Components per instance: 1 (float), 2 (vec2), 3 (vec3), 4 (vec4) |
| `normalized` | `boolean` | Map integer range to [0,1] / [-1,1]. Typical `false` for floats. |
| `meshPerAttribute` | `integer` | Default `1`. Set to `N` for N-to-1 sharing (see §5). |

### `meshPerAttribute`

Controls how many consecutive instances share the same attribute value. With the default `1`, instance `i` reads `array[i * itemSize]`. With `meshPerAttribute = 4`, every group of 4 instances shares the same value:

```
instance 0,1,2,3  → array[0]
instance 4,5,6,7  → array[1]
```

Use this to reduce buffer size when many instances share a discrete set of variants (e.g. 4 instances per "team", 16 teams → `meshPerAttribute = 4`, array length = 16).

---

## 3. `InstancedBufferGeometry`

### Source (r184, verbatim excerpt)

```js
class InstancedBufferGeometry extends BufferGeometry {
  constructor() {
    super();
    this.isInstancedBufferGeometry = true;
    this.type = 'InstancedBufferGeometry';
    this.instanceCount = Infinity;
  }

  copy(source) {
    super.copy(source);
    this.instanceCount = source.instanceCount;
    return this;
  }
}
```

`instanceCount` defaults to `Infinity`, meaning "draw all instances implied by the attribute arrays". Set it explicitly to cap draws:

```js
geo.instanceCount = 500; // render 500 of the 1000 allocated
```

`InstancedBufferGeometry` is the lower-level primitive. In modern practice, prefer attaching `InstancedBufferAttribute` directly to a `BufferGeometry` used by `InstancedMesh` — it achieves the same result with less boilerplate.

---

## 4. Full Pipeline: `InstancedMesh` + Custom Attribute

### Step 1 — Create the InstancedMesh normally

```js
const geo = new THREE.IcosahedronGeometry(1, 2);
const mat = new THREE.ShaderMaterial({ /* see step 3 */ });
const COUNT = 1000;
const mesh = new THREE.InstancedMesh(geo, mat, COUNT);
scene.add(mesh);
```

### Step 2 — Add per-instance attributes to the geometry

```js
// Float per instance: animation phase 0–2π
const phaseData = new Float32Array(COUNT);
const scaleData  = new Float32Array(COUNT * 3); // vec3: non-uniform scale per instance

for (let i = 0; i < COUNT; i++) {
  phaseData[i] = Math.random() * Math.PI * 2;
  scaleData[i * 3 + 0] = 0.5 + Math.random();   // x
  scaleData[i * 3 + 1] = 0.5 + Math.random();   // y
  scaleData[i * 3 + 2] = 0.5 + Math.random();   // z
}

// itemSize must match the vec type used in GLSL
geo.setAttribute('aPhase',      new THREE.InstancedBufferAttribute(phaseData, 1));
geo.setAttribute('aLocalScale', new THREE.InstancedBufferAttribute(scaleData,  3));
```

### Step 3 — Declare and use the attributes in GLSL

```js
const mat = new THREE.ShaderMaterial({
  uniforms: {
    uTime: { value: 0 },
  },

  vertexShader: /* glsl */`
    // Three.js built-ins available automatically:
    //   attribute vec4 instanceMatrix (actually 4×vec4, assembled to mat4)
    //   uniform mat4 modelMatrix, viewMatrix, projectionMatrix, etc.

    // Your custom per-instance attributes:
    attribute float aPhase;
    attribute vec3  aLocalScale;

    uniform float uTime;

    varying vec3 vNormal;

    void main() {
      // Apply per-instance non-uniform scale before the instance transform
      vec3 scaledPos = position * aLocalScale;

      // instanceMatrix is injected as mat4 by Three.js when USE_INSTANCING is set.
      // With ShaderMaterial, you must read it yourself:
      vec4 worldPos = instanceMatrix * vec4(scaledPos, 1.0);

      // Animate along normal
      float wave = sin(uTime + aPhase) * 0.15;
      worldPos.xyz += normal * wave;

      gl_Position = projectionMatrix * viewMatrix * modelMatrix * worldPos;
      vNormal = normalMatrix * normal;
    }
  `,

  fragmentShader: /* glsl */`
    varying vec3 vNormal;

    void main() {
      gl_FragColor = vec4(vNormal * 0.5 + 0.5, 1.0);
    }
  `,
});
```

> **Important:** When you use `ShaderMaterial`, Three.js does **not** automatically inject the instancing chunks. You must declare `instanceMatrix` yourself as four `vec4` attributes and reconstruct the `mat4`:

```glsl
// Manual declaration (required for ShaderMaterial)
attribute vec4 instanceMatrix0;
attribute vec4 instanceMatrix1;
attribute vec4 instanceMatrix2;
attribute vec4 instanceMatrix3;

// Reconstruct inside main():
mat4 instanceMatrix = mat4(
  instanceMatrix0,
  instanceMatrix1,
  instanceMatrix2,
  instanceMatrix3
);
```

Or use `onBeforeCompile` to patch a built-in material (see §5).

### Step 4 — Update each frame

```js
function tick(t) {
  mat.uniforms.uTime.value = t;
  // instanceMatrix.needsUpdate only if transforms changed
  // No needsUpdate needed for uTime (it's a uniform, not a buffer)
}
```

---

## 5. `onBeforeCompile` Pattern (Patch Built-in Materials)

Use this when you want per-instance data on `MeshStandardMaterial`, `MeshPhongMaterial`, etc., without rewriting the full shader.

```js
const mat = new THREE.MeshStandardMaterial({ color: 0xffffff });

mat.onBeforeCompile = (shader) => {
  shader.uniforms.uTime = { value: 0 };

  // Inject attribute declaration before the main vertex block
  shader.vertexShader = shader.vertexShader.replace(
    '#include <common>',
    /* glsl */`
      #include <common>
      attribute float aPhase;
      uniform float uTime;
    `
  );

  // Inject usage after the built-in transform
  shader.vertexShader = shader.vertexShader.replace(
    '#include <begin_vertex>',
    /* glsl */`
      #include <begin_vertex>
      transformed += normal * sin(uTime + aPhase) * 0.2;
    `
  );

  // Keep a reference to update the uniform each frame
  mat.userData.shader = shader;
};

// In tick():
if (mat.userData.shader) {
  mat.userData.shader.uniforms.uTime.value = elapsed;
}
```

This works because:
- `#include <begin_vertex>` is the chunk where `transformed = position;` is first set.
- Custom attributes are automatically picked up by `WebGLProgram` if they are attached to the geometry as `InstancedBufferAttribute`.
- The `instanceMatrix` chunks (`USE_INSTANCING` guards) are already present in the built-in vertex shader.

---

## 6. Multi-Component Attributes

### `vec2` — UV offset per instance

```js
const offsets = new Float32Array(COUNT * 2);
for (let i = 0; i < COUNT; i++) {
  offsets[i * 2]     = Math.floor(i % 4) * 0.25;  // atlas column
  offsets[i * 2 + 1] = Math.floor(i / 4) * 0.25;  // atlas row
}
geo.setAttribute('aUVOffset', new THREE.InstancedBufferAttribute(offsets, 2));
```

```glsl
attribute vec2 aUVOffset;
varying vec2 vUv;

void main() {
  vUv = uv + aUVOffset;
  // ...
}
```

### `vec4` — RGBA tint per instance

```js
const tints = new Float32Array(COUNT * 4);
// fill with RGBA values...
geo.setAttribute('aTint', new THREE.InstancedBufferAttribute(tints, 4));
```

```glsl
attribute vec4 aTint;
varying vec4 vTint;
// pass to fragment: vTint = aTint;
```

### Packed integers — flags and IDs

```js
// Use Uint8Array + normalized=true to pack 0–255 → 0.0–1.0
const flags = new Uint8Array(COUNT);
geo.setAttribute('aFlags', new THREE.InstancedBufferAttribute(flags, 1, true));
```

---

## 7. `InstancedBufferGeometry` Direct Usage (Lower-Level)

For cases where you are **not** using `InstancedMesh` — e.g. a custom draw loop or Points/Lines instancing:

```js
const iGeo = new THREE.InstancedBufferGeometry();

// Copy base geometry attributes
iGeo.setAttribute('position', baseGeo.getAttribute('position'));
iGeo.setAttribute('normal',   baseGeo.getAttribute('normal'));
iGeo.index = baseGeo.index;

// Per-instance attributes
iGeo.setAttribute('aOffset', new THREE.InstancedBufferAttribute(offsetArray, 3));

// Cap render count
iGeo.instanceCount = COUNT;

// Pair with a raw Mesh (not InstancedMesh)
const mesh = new THREE.Mesh(iGeo, material);
```

Caveat: Without `InstancedMesh`, `setMatrixAt` / `setColorAt` are unavailable. You handle all transforms via attributes.

---

## 8. Updating Attribute Data at Runtime

```js
// Direct array write (fastest — no allocation)
const attr = mesh.geometry.getAttribute('aPhase');
const arr  = attr.array;           // the underlying Float32Array

for (let i = 0; i < COUNT; i++) {
  arr[i] = newPhases[i];
}

attr.needsUpdate = true;           // upload to GPU on next render
```

Partial updates are not exposed in Three.js core (no `updateRange` shortcut in the public API), but you can set `attr.updateRanges` manually in r152+:

```js
attr.updateRanges = [{ start: startIndex, count: numItems }];
attr.needsUpdate = true;
```

---

## 9. Common Errors

| Error | Cause | Fix |
|-------|-------|-----|
| Attribute ignored silently | `itemSize` mismatch between JS and GLSL | Ensure `new InstancedBufferAttribute(arr, 3)` → `attribute vec3` |
| All instances show same value | Forgot `InstancedBufferAttribute`, used plain `BufferAttribute` | Only `InstancedBufferAttribute` advances per-instance |
| `needsUpdate` not working | Set it on the attribute object, not on the geometry | `attr.needsUpdate = true`, not `geo.needsUpdate` |
| `onBeforeCompile` not triggered | Material was cloned or shared; clone returns a new material | Call `mat.needsUpdate = true` after modifying |

---

## Sources

```yaml
- url: https://raw.githubusercontent.com/mrdoob/three.js/dev/src/core/InstancedBufferAttribute.js
  tier: 1
  accessed_at_utc: "2026-05-26T00:00:00Z"
  notes: "Verbatim source: constructor, meshPerAttribute, copy, toJSON"

- url: https://raw.githubusercontent.com/mrdoob/three.js/dev/src/core/InstancedBufferGeometry.js
  tier: 1
  accessed_at_utc: "2026-05-26T00:00:00Z"
  notes: "Verbatim source: instanceCount default Infinity"

- url: https://medium.com/@pailhead011/instancing-with-three-js-part-2-3be34ae83c57
  tier: 3
  accessed_at_utc: "2026-05-26T00:00:00Z"
  notes: "Manual mat4 from 4×vec4 attribute pattern; per-instance color varying"

- url: https://discourse.threejs.org/t/instanceduniformsmesh-set-shader-uniform-values-per-instance/22814
  tier: 4
  accessed_at_utc: "2026-05-26T00:00:00Z"
  notes: "Community discussion on onBeforeCompile for per-instance attributes"
```
