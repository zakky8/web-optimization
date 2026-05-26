# Custom Material Patterns — ShaderMaterial, RawShaderMaterial, NodeMaterial

**Source revision:** Three.js r184
**Accessed:** 2026-05-26
**Primary sources:**
- `src/materials/ShaderMaterial.js` (r184) — defaults, property list
- `src/materials/RawShaderMaterial.js` (r184) — differences from ShaderMaterial
- `src/materials/Material.js` (r184) — needsUpdate setter, version counter, clone()
- `src/materials/nodes/NodeMaterial.js` (r184) — customProgramCacheKey, copy(), base inputs
- `src/materials/nodes/MeshStandardNodeMaterial.js` (r184) — TSL node override pattern

---

## ShaderMaterial vs RawShaderMaterial

### ShaderMaterial

`ShaderMaterial` provides a GLSL shader with Three.js built-in uniforms, attributes, and varyings automatically prepended. You write the core logic; Three.js supplies the scaffolding.

**Default property values:**

| Property | Default | Notes |
|----------|---------|-------|
| `uniforms` | `{}` | Object of `{ name: { value: ... } }` |
| `vertexShader` | `default_vertex` (built-in GLSL) | No-op passthrough |
| `fragmentShader` | `default_fragment` (built-in GLSL) | Outputs basic white |
| `defines` | `{}` | Preprocessor `#define` entries |
| `clipping` | `false` | Must be `true` to respect clipping planes |
| `fog` | `false` | Must be `true` to receive fog uniforms |
| `lights` | `false` | Must be `true` to receive light uniforms |
| `linewidth` | `1` | WebGL line width (most GPUs ignore >1) |
| `wireframe` | `false` | |
| `glslVersion` | `null` | Set to `THREE.GLSL3` for WebGL2 syntax |
| `uniformsNeedUpdate` | `false` | Per-frame uniform sync flag |

**Automatically prepended when relevant flags are true:**
- `projectionMatrix`, `viewMatrix`, `modelMatrix`, `modelViewMatrix`, `normalMatrix` — always
- `cameraPosition` — always
- `position`, `normal`, `uv` attributes — always
- Light structs (`directionalLights`, `pointLights`, etc.) — when `lights: true`
- Fog uniforms (`fogColor`, `fogDensity`, etc.) — when `fog: true`
- Clip planes — when `clipping: true`

### RawShaderMaterial

`RawShaderMaterial` extends `ShaderMaterial` and sets `isRawShaderMaterial = true`. The only material difference: **no built-in uniforms, attributes, or precision qualifiers are prepended**. You write the complete GLSL shader from the `#version` directive onward.

```js
// RawShaderMaterial requires all declarations explicitly
const raw = new THREE.RawShaderMaterial({
  glslVersion: THREE.GLSL3,
  vertexShader: `
    in vec3 position;
    uniform mat4 modelViewMatrix;
    uniform mat4 projectionMatrix;
    void main() {
      gl_Position = projectionMatrix * modelViewMatrix * vec4(position, 1.0);
    }
  `,
  fragmentShader: `
    precision highp float;
    out vec4 fragColor;
    void main() { fragColor = vec4(1.0, 0.0, 0.0, 1.0); }
  `,
});
```

### Side-by-side comparison

| Feature | ShaderMaterial | RawShaderMaterial |
|---------|---------------|-------------------|
| Built-in uniforms prepended | Yes | No |
| Built-in attributes prepended | Yes | No |
| Precision qualifier prepended | Yes | No |
| `#version` directive | Auto-inserted | Must be explicit |
| Use case | Custom effects on top of Three.js setup | Full GLSL control, minimal overhead |
| `isRawShaderMaterial` | `false` | `true` |
| Type string | `'ShaderMaterial'` | `'RawShaderMaterial'` |

---

## MeshStandardNodeMaterial with TSL

`MeshStandardNodeMaterial` (type string: `'MeshStandardNodeMaterial'`) extends `NodeMaterial` and exposes Three.js Shading Language (TSL) node inputs that override specific BRDF terms.

### Available node inputs

| Property | TSL type | Fallback |
|----------|----------|---------|
| `colorNode` | `vec3` | `materialColor` (from `color` + `map`) |
| `emissiveNode` | `vec3` | `materialEmissive` (from `emissive` × `emissiveIntensity`) |
| `roughnessNode` | `float` | `materialRoughness` (from `roughness` + `roughnessMap`) |
| `metalnessNode` | `float` | `materialMetalness` (from `metalness` + `metalnessMap`) |
| `normalNode` | `vec3` | `materialNormal` (from `normalMap` or geometry normal) |
| `positionNode` | `vec3` | `positionLocal` (geometry vertex position) |

All default to `null` — if `null`, the renderer uses the corresponding scalar/texture uniform. Setting a node property replaces the entire uniform chain for that term.

### TSL override pattern

```js
import { MeshStandardNodeMaterial } from 'three/webgpu';
import { float, vec3, sin, time, uv, color } from 'three/tsl';

const material = new MeshStandardNodeMaterial({
  roughness: 0.5,
  metalness: 0.0,
});

// Animate roughness over time using a TSL expression
material.roughnessNode = sin(time).mul(0.5).add(0.5);

// Drive colour from UV coordinates
material.colorNode = vec3(uv().x, uv().y, float(0.5));
```

When a node input is assigned, `NodeMaterial` recalculates the `customProgramCacheKey()`. This key is a hash of all active node property names and their child cache keys; a change forces shader recompilation automatically, without manually setting `needsUpdate = true`.

---

## Cloning Custom Materials

### `Material.clone()` (base implementation)

```js
clone() {
  return new this.constructor().copy(this);
}
```

Creates a new instance of the same class and copies all property values via `copy()`. For `ShaderMaterial`, this copies `uniforms` (shallow — `{ value }` references are shared), `vertexShader`, `fragmentShader`, `defines`, and flags.

**Critical caveat — uniform references are shared:** After `clone()`, `original.uniforms.time === cloned.uniforms.time` (same object). Mutations to the value affect both. Deep-clone uniforms when you need per-instance values:

```js
function cloneCustomMaterial(source) {
  const clone = source.clone();
  // Deep-clone each uniform's value object
  clone.uniforms = THREE.UniformsUtils.clone(source.uniforms);
  return clone;
}
```

`THREE.UniformsUtils.clone(uniforms)` performs a deep copy of the uniforms object, allocating new `{ value }` wrappers (but not cloning `THREE.Texture` objects inside them — texture references are still shared).

### Pattern for per-mesh instances

```js
const baseMaterial = new THREE.ShaderMaterial({ /* ... */ });

// For each mesh that needs unique uniform values:
const instanceMaterial = cloneCustomMaterial(baseMaterial);
instanceMaterial.uniforms.color.value = new THREE.Color(Math.random(), 0, 0);
mesh.material = instanceMaterial;
```

---

## `material.needsUpdate`

### What it does

`Material.needsUpdate` is a write-only setter. Setting it to `true` increments the material's internal `version` counter:

```js
set needsUpdate(value) {
  if (value === true) this.version++;
}
```

The renderer compares `material.version` against its cached program version. A mismatch triggers full shader recompilation on the next render frame. This is expensive (100–500 ms on large shaders) — treat it as a one-time setup cost, not a per-frame operation.

### When you must set `needsUpdate = true`

| Change | Requires needsUpdate |
|--------|---------------------|
| Modifying `vertexShader` or `fragmentShader` string | Yes |
| Adding or removing a key in `defines` | Yes |
| Changing a `defines` value | Yes |
| Changing `lights`, `fog`, `clipping` flags | Yes |
| Adding/removing a texture where presence drives a `#define` (e.g. `USE_MAP`) | Yes (Three.js handles this automatically for built-in materials) |
| Changing `uniforms[x].value` | No — uniforms are hot-updated without recompilation |
| Changing `visible`, `opacity`, `side` | No |

### Common mistake

```js
// Wrong — shader string was replaced but GPU program not recompiled
material.fragmentShader = newGLSL;

// Correct
material.fragmentShader = newGLSL;
material.needsUpdate = true;
```

For `ShaderMaterial.uniformsNeedUpdate`, this is a lighter flag that forces uniform re-upload without recompilation — useful inside `Object3D.onBeforeRender` when you modify uniforms on objects that share a material.

---

## NodeMaterial: Automatic Recompilation

In node materials (`NodeMaterial`, `MeshStandardNodeMaterial`, `MeshPhysicalNodeMaterial`), the `customProgramCacheKey()` mechanism replaces manual `needsUpdate`:

```js
// From NodeMaterial source (r184)
customProgramCacheKey() {
  // hashes all "*Node" property names and their child cache keys
  return `${this.type}${hashArray(values)}`;
}
```

When you assign a TSL node to `material.roughnessNode`, the cache key changes automatically. The renderer detects the mismatch and recompiles. You do not call `material.needsUpdate = true`.

However, if you **mutate internal state within an existing node** (rather than replacing the node reference), the cache key does not change and the shader is not recompiled. Assign a new node object in that case.

---

## Quick Decision Guide

| Situation | Use |
|-----------|-----|
| Custom GLSL with Three.js lighting/matrices available | `ShaderMaterial` |
| Full GLSL from scratch, no scaffolding | `RawShaderMaterial` |
| Procedural material using TSL, stays in node graph | `MeshStandardNodeMaterial` |
| PBR base + custom transmission/iridescence nodes | `MeshPhysicalNodeMaterial` |
| Per-mesh unique uniforms from a shared material | `cloneCustomMaterial` + `UniformsUtils.clone` |
| Changed shader code at runtime | Set `needsUpdate = true` after the change |

---

## Sources

```yaml
- claim: "ShaderMaterial defaults: uniforms {}, defines {}, clipping false, fog false, lights false, glslVersion null, uniformsNeedUpdate false"
  source: src/materials/ShaderMaterial.js (r184)
  tier: 1
  accessed_at_utc: "2026-05-26T00:00:00Z"
  status: verified

- claim: "RawShaderMaterial: no built-in uniforms/attributes prepended, isRawShaderMaterial = true, type 'RawShaderMaterial'"
  source: src/materials/RawShaderMaterial.js (r184)
  tier: 1
  accessed_at_utc: "2026-05-26T00:00:00Z"
  status: verified

- claim: "Material.needsUpdate setter increments this.version; renderer recompiles on version mismatch"
  exact_quote: "set needsUpdate( value ) { if ( value === true ) this.version ++; }"
  source: src/materials/Material.js (r184)
  tier: 1
  accessed_at_utc: "2026-05-26T00:00:00Z"
  status: verified

- claim: "Material.clone() = new this.constructor().copy(this)"
  exact_quote: "clone() { return new this.constructor().copy( this ); }"
  source: src/materials/Material.js (r184)
  tier: 1
  accessed_at_utc: "2026-05-26T00:00:00Z"
  status: verified

- claim: "MeshStandardNodeMaterial type 'MeshStandardNodeMaterial', extends NodeMaterial, exposes emissiveNode, roughnessNode, metalnessNode"
  source: src/materials/nodes/MeshStandardNodeMaterial.js (r184)
  tier: 1
  accessed_at_utc: "2026-05-26T00:00:00Z"
  status: verified

- claim: "NodeMaterial customProgramCacheKey hashes all *Node properties; changing a node reference triggers automatic recompile"
  source: src/materials/nodes/NodeMaterial.js (r184)
  tier: 1
  accessed_at_utc: "2026-05-26T00:00:00Z"
  status: verified
```
