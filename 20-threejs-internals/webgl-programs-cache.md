# WebGLPrograms — Shader Compilation & Cache

## How the Cache Works

Three.js compiles a GLSL program for each unique combination of:
- Material type (MeshStandardMaterial, ShaderMaterial, etc.)
- Number/type of lights
- Shadow map type
- Enabled maps (normalMap, roughnessMap, etc.)
- Skinning/morphing
- Tone mapping setting
- Vertex colors, instancing flags
- `customProgramCacheKey()` return value

This is called a **program variant**. Too many variants = **shader explosion**.

## Monitoring Shader Count

```js
// Normal for a medium scene: 10-30 programs
// Warning zone: 50+
// Problem: 100+ (significant compile stall, memory waste)

console.log(`Shader programs: ${renderer.info.programs.length}`);

// Detailed breakdown
renderer.info.programs.forEach(prog => {
  console.log(prog.name, prog.usedTimes);
});
```

## Shader Explosion Causes

```js
// ❌ WRONG: unique material per object
objects.forEach((obj, i) => {
  obj.material = new THREE.MeshStandardMaterial({
    color: colors[i],  // Different color = potentially different program variant?
    // Actually color alone doesn't cause explosion...
  });
});
// ...but THIS does:
objects.forEach((obj, i) => {
  obj.material = new THREE.MeshStandardMaterial({
    map: textures[i],       // New material with different map = new program key
    onBeforeCompile: ...    // Custom code = new program
  });
});

// ✓ CORRECT: shared material, color via instancing or vertex colors
const mat = new THREE.MeshStandardMaterial({ vertexColors: true });
```

## customProgramCacheKey

When using `onBeforeCompile` with variants, ALWAYS provide a cache key:

```js
mat.onBeforeCompile = (shader) => {
  shader.fragmentShader = shader.fragmentShader.replace(
    '#include <dithering_fragment>',
    mat.userData.glow ? glowCode : normalCode
  );
};

// Without this: Three.js can't distinguish variants → wrong shader served
mat.customProgramCacheKey = () => {
  return `myMat-glow:${mat.userData.glow}`;
};
```

## Shader Compilation Stalls

First-time compilation of a program stalls the main thread (synchronous WebGL compile).

**Mitigation strategies:**

```js
// 1. Warm up all materials before first render
renderer.compile(scene, camera);
// Forces compile of all materials in scene during loading screen

// 2. KHR_parallel_shader_compile (Chrome 70+, Firefox, Safari 15+)
// Three.js uses this automatically when available
// Compiles shaders on a separate thread — no stall

// 3. Limit material variants by design
// One material per visual archetype, not per object

// 4. THREE.WebGLRenderer.info to find programs compiled per frame
```

## BatchedMesh (r158+)

Replaces InstancedMesh for heterogeneous geometry batching.

```js
// InstancedMesh: N identical meshes → 1 draw call
// BatchedMesh: N different meshes → 1 draw call (WebGL2 multi-draw)

const batched = new THREE.BatchedMesh(
  100,        // maxGeometryCount
  5000,       // maxVertexCount
  10000,      // maxIndexCount
  material
);

// Add geometries (returns ID)
const boxId    = batched.addGeometry(new THREE.BoxGeometry());
const sphereId = batched.addGeometry(new THREE.SphereGeometry(0.5));
const coneId   = batched.addGeometry(new THREE.ConeGeometry(0.5, 1));

// Add instances of each geometry
const inst0 = batched.addInstance(boxId);
const inst1 = batched.addInstance(sphereId);
const inst2 = batched.addInstance(boxId);

// Position each instance
const matrix = new THREE.Matrix4();
matrix.setPosition(0, 0, 0);
batched.setMatrixAt(inst0, matrix);

matrix.setPosition(2, 0, 0);
batched.setMatrixAt(inst1, matrix);

scene.add(batched);
```

## Sources
- Three.js WebGLPrograms: https://github.com/mrdoob/three.js/blob/master/src/renderers/webgl/WebGLPrograms.js
- BatchedMesh docs: https://threejs.org/docs/#api/en/objects/BatchedMesh
- Three.js r158 changelog: https://github.com/mrdoob/three.js/blob/master/CHANGELOG.md
