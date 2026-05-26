# Three.js WebGL Render Pipeline Internals

## renderer.render(scene, camera) — What Happens

```
renderer.render(scene, camera)
│
├─ 1. scene.autoUpdate → scene.updateMatrixWorld()
│     (traverses all objects, updates world matrices)
│
├─ 2. camera.updateMatrixWorld() + updateProjectionMatrix()
│
├─ 3. renderer.shadowMap.render() [if shadows enabled]
│     (renders depth maps for each shadow-casting light)
│
├─ 4. projectObject(scene, camera, ...)
│     (frustum culling + sort into render lists)
│     → renderListOpaque[]  (front-to-back by z)
│     → renderListTransparent[]  (back-to-front by z)
│
├─ 5. renderScene(renderList, scene, camera)
│     ├─ For each object in opaque list:
│     │   ├─ setRenderState(object)  [blend, depth, stencil]
│     │   ├─ setProgram(camera, scene, material, object)
│     │   │   ├─ WebGLPrograms.acquireProgram() [cache lookup or compile]
│     │   │   └─ setUniforms() [upload all uniforms]
│     │   └─ drawElements() / drawArrays()
│     └─ For each object in transparent list (same, reverse order):
│         └─ ...
│
└─ 6. EffectComposer / postprocessing passes (if active)
```

## The Render List Sort

```js
// Opaque: front-to-back (z ordering)
// Reason: early z-rejection — GPU discards hidden pixels BEFORE shading
// Massive savings when many objects overlap

// Transparent: back-to-front (painter's algorithm)
// Reason: correct alpha blending requires back objects rendered first

// Object.renderOrder overrides the sort
mesh.renderOrder = 999;  // Render last (on top)
mesh.renderOrder = -1;   // Render first (behind everything)
```

## Frustum Culling

```js
// Default: enabled
mesh.frustumCulled = true;

// The camera's frustum is computed once per frame
// Any mesh whose bounding sphere doesn't intersect the frustum is SKIPPED
// No draw call submitted — pure CPU-side test

// Geometry needs up-to-date bounding sphere
geometry.computeBoundingSphere();
geometry.computeBoundingBox();  // For Box3-based tests

// Dynamic geometry: must call after every vertex update
geometry.attributes.position.needsUpdate = true;
geometry.computeBoundingSphere();
```

## renderOrder vs depthTest vs depthWrite

```js
// renderOrder: CPU-side, controls submission order
mesh.renderOrder = 1;

// depthTest: GPU-side, compare fragment depth to depth buffer
material.depthTest = true;   // Default — discard if behind existing geometry
material.depthTest = false;  // Always visible (UI overlays, debug gizmos)

// depthWrite: GPU-side, write to depth buffer
material.depthWrite = true;  // Default — opaque objects write
material.depthWrite = false; // Don't write depth (particles, transparent)

// Typical transparent setup:
material.transparent = true;
material.depthWrite = false;   // Don't occlude other transparents
material.depthTest = true;     // Do occlude by opaque geometry
material.blending = THREE.AdditiveBlending;  // or NormalBlending
```

## Sources
- Three.js WebGLRenderer source: https://github.com/mrdoob/three.js/blob/master/src/renderers/WebGLRenderer.js
- Three.js materials guide: https://threejs.org/docs/#manual/en/introduction/Material-types
