# renderer.info â€” GPU Resource Ground Truth

## Properties

```js
const info = renderer.info;

// â”€â”€ Memory â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€
info.memory.geometries  // Number of geometries currently on GPU
info.memory.textures    // Number of textures currently on GPU

// â”€â”€ Render (per-frame, resets each frame automatically) â”€â”€â”€â”€â”€â”€â”€â”€
info.render.calls       // Draw calls this frame
info.render.triangles   // Triangles drawn this frame
info.render.points      // Points drawn
info.render.lines       // Lines drawn
info.render.frame       // Cumulative frame counter

// â”€â”€ Programs â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€
info.programs           // Array<WebGLProgram> â€” all compiled shaders
info.programs.length    // Shader variant count
```

## Monitoring for Leaks

```js
// Check before/after loading a scene to confirm proper cleanup
console.log('Before:', {
  geo: renderer.info.memory.geometries,
  tex: renderer.info.memory.textures,
  prg: renderer.info.programs.length,
});

loadScene();

console.log('After load:', {
  geo: renderer.info.memory.geometries,
  tex: renderer.info.memory.textures,
});

disposeScene();

console.log('After dispose:', {
  geo: renderer.info.memory.geometries,   // Should return to before-load value
  tex: renderer.info.memory.textures,     // Same
});
```

## Shader Explosion Detection

```js
// Shader explosion: geometry/texture count looks fine but programs keeps growing
// Cause: per-frame material creation, missing customProgramCacheKey

let lastProgramCount = 0;

renderer.setAnimationLoop(() => {
  const current = renderer.info.programs.length;
  if (current !== lastProgramCount) {
    console.warn(`Shader programs changed: ${lastProgramCount} â†’ ${current}`);
    lastProgramCount = current;
  }
  renderer.render(scene, camera);
});
```

## Draw Call Budget

```
Mobile (low-end): 50-100 draw calls target
Mobile (flagship): 100-200 draw calls
Desktop (integrated GPU): 200-500
Desktop (dedicated GPU): 1000+
```

Reduce with:
- `BatchedMesh` (r158+): merge static meshes, 1 draw call
- `InstancedMesh`: N identical objects, 1 draw call
- Frustum culling: mesh.frustumCulled = true (default)
- Occlusion culling: manual via GPU picking or spatial grid

## Automatic Reset

`renderer.info.reset()` is called automatically by `renderer.render()` each frame.
Do NOT call it manually unless you want per-subframe stats.

## Sources
- Three.js WebGLInfo: https://threejs.org/docs/#api/en/renderers/WebGLRenderer.info
- Three.js r183 GLTFLoader cache fix: https://github.com/mrdoob/three.js/blob/master/CHANGELOG.md
