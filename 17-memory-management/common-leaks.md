# Common Three.js Memory Leaks

## 1. scene.background / scene.environment

**Problem:** `scene.traverse()` does NOT visit background or environment textures.
They must be disposed manually.

```js
// âŒ WRONG â€” misses background and environment
scene.traverse(obj => {
  obj.geometry?.dispose();
  obj.material?.dispose();
});

// âœ“ CORRECT
scene.traverse(obj => {
  if (!obj.isMesh) return;
  obj.geometry?.dispose();
  const mats = Array.isArray(obj.material) ? obj.material : [obj.material];
  mats.forEach(m => {
    Object.values(m).forEach(v => v?.isTexture && v.dispose());
    m.dispose();
  });
});

// Always dispose these separately
if (scene.background?.isTexture)  scene.background.dispose();
if (scene.environment?.isTexture) scene.environment.dispose();
```

## 2. EffectComposer Pass RenderTargets

**Problem:** `composer.dispose()` disposes the composer but NOT each pass's internal render targets.

```js
// âŒ WRONG
composer.dispose();  // Leaks each pass's renderTarget

// âœ“ CORRECT
composer.passes.forEach(pass => {
  pass.dispose?.();      // Some passes implement dispose()
  // For UnrealBloomPass, RenderPass, etc. that don't:
  if (pass.renderTargetBright) pass.renderTargetBright.dispose();
  if (pass.renderTargetsHorizontal) {
    pass.renderTargetsHorizontal.forEach(rt => rt.dispose());
    pass.renderTargetsVertical.forEach(rt => rt.dispose());
  }
});
composer.dispose();
```

## 3. CubeRenderTarget Leak (Fixed r180)

**Affected versions:** Three.js < r180
**Fix:** Update to r180+. PMREMGenerator.fromScene() leaked the intermediate CubeRenderTarget.

```js
// Pre-r180 workaround (no longer needed on r180+)
const cubeRenderTarget = pmremGenerator.fromScene(scene);
pmremGenerator.dispose(); // Did not free cubeRenderTarget in <r180
cubeRenderTarget.dispose(); // Had to dispose manually
```

## 4. PMREMGenerator Leak (Fixed r181)

**Affected versions:** Three.js < r181
**Fix:** Update to r181+.

```js
// r181+ â€” properly disposes internal textures
const envMap = pmremGenerator.fromEquirectangular(hdrTexture).texture;
pmremGenerator.dispose();  // Safe in r181+
hdrTexture.dispose();
```

## 5. GLTFLoader Cache (Fixed r183)

**Affected versions:** Three.js < r183
**Symptom:** Loading/unloading same GLTF repeatedly caused texture accumulation.

```js
// r183+ â€” loader properly evicts from cache on dispose
const loader = new GLTFLoader();
loader.load('model.glb', (gltf) => {
  // ... use model

  // On teardown â€” cache eviction now works correctly in r183+
  gltf.scene.traverse(obj => {
    obj.geometry?.dispose();
    obj.material?.dispose();
  });
});
```

## 6. Material Created Inside Render Loop

```js
// âŒ WRONG â€” creates new shader program every frame
renderer.setAnimationLoop(() => {
  mesh.material = new THREE.MeshStandardMaterial({ color: newColor });
  renderer.render(scene, camera);
});

// âœ“ CORRECT â€” create once, update uniform
const mat = new THREE.MeshStandardMaterial({ color: 0xff0000 });
mesh.material = mat;

renderer.setAnimationLoop(() => {
  mat.color.set(newColor);  // No new program
  renderer.render(scene, camera);
});
```

## 7. Event Listener Leaks (Outside Three.js)

```js
// Always clean up ResizeObserver, pointer events, etc. when dismounting
const handleResize = () => { /* ... */ };
window.addEventListener('resize', handleResize);

// React: return cleanup from useEffect
useEffect(() => {
  window.addEventListener('resize', handleResize);
  return () => window.removeEventListener('resize', handleResize);
}, []);
```

## Sources
- Three.js dispose guide: https://threejs.org/docs/#manual/en/introduction/How-to-dispose-of-objects
- Three.js changelog r180-183: https://github.com/mrdoob/three.js/blob/master/CHANGELOG.md
