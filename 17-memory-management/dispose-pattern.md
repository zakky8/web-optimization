# Three.js Dispose Pattern

## The Event-Driven Dispose Model

When you call `.dispose()` on a Three.js resource, it:
1. Fires a `dispose` event on the object
2. The renderer's `WebGLObjects` listener catches it
3. Calls `gl.deleteBuffer()`, `gl.deleteTexture()`, etc. internally
4. Removes the object from renderer's internal caches

You never call WebGL delete functions directly.

## What Needs Disposing

| Resource | dispose() call | Notes |
|----------|---------------|-------|
| Geometry | `geometry.dispose()` | Deletes VBO/IBO from GPU |
| Texture | `texture.dispose()` | Deletes from VRAM |
| Material | `material.dispose()` | Removes shader program ref |
| RenderTarget | `rt.dispose()` | Frees framebuffer + texture |
| EffectComposer | `composer.dispose()` then each pass | See warning below |
| CubeRenderTarget | `crt.dispose()` | Fixed in r180 |
| PMREMGenerator | `pmrem.dispose()` | Fixed in r181 |
| GLTFLoader cache | Manual clear | Fixed in r183 |

## DisposeManager Class

```js
export class DisposeManager {
  constructor() {
    this._resources = new Set();
  }

  track(...resources) {
    resources.forEach(r => this._resources.add(r));
    return resources[0];
  }

  disposeAll() {
    this._resources.forEach(resource => {
      try {
        resource.dispose?.();
        // Recursively dispose material arrays
        if (Array.isArray(resource.material)) {
          resource.material.forEach(m => m.dispose?.());
        } else {
          resource.material?.dispose?.();
        }
        resource.geometry?.dispose?.();
      } catch (e) {
        console.warn('Dispose error:', e);
      }
    });
    this._resources.clear();
  }
}

// Usage
const mgr = new DisposeManager();

const geo  = mgr.track(new THREE.BufferGeometry());
const mat  = mgr.track(new THREE.MeshStandardMaterial());
const mesh = new THREE.Mesh(geo, mat);

// On scene teardown
mgr.disposeAll();
```

## Traverse Dispose (Full Scene)

```js
function disposeScene(scene, renderer) {
  scene.traverse((obj) => {
    if (!obj.isMesh) return;

    obj.geometry?.dispose();

    const mats = Array.isArray(obj.material) ? obj.material : [obj.material];
    mats.forEach(mat => {
      if (!mat) return;
      // Dispose all texture maps on the material
      Object.values(mat).forEach(value => {
        if (value?.isTexture) value.dispose();
      });
      mat.dispose();
    });
  });

  // IMPORTANT: scene.background and scene.environment are NOT
  // reachable via traverse() â€” dispose manually
  if (scene.background?.isTexture) scene.background.dispose();
  if (scene.environment?.isTexture) scene.environment.dispose();

  scene.clear();
}
```

## Sources
- Three.js How to dispose: https://threejs.org/docs/#manual/en/introduction/How-to-dispose-of-objects
- Three.js Material.dispose: https://threejs.org/docs/#api/en/materials/Material.dispose
