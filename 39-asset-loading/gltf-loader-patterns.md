# GLTFLoader Patterns

**Source:** https://threejs.org/docs/#examples/en/loaders/GLTFLoader  
**Source (raw loader):** https://github.com/mrdoob/three.js/blob/dev/examples/jsm/loaders/GLTFLoader.js  
**Accessed:** 2026-05-26

---

## Import

```js
import { GLTFLoader } from 'three/addons/loaders/GLTFLoader.js';
```

---

## Constructor

```js
GLTFLoader(manager?: LoadingManager)
```

Passing a `LoadingManager` is optional. Omitting it uses `THREE.DefaultLoadingManager`.

---

## Required Extensions

### DRACOLoader (Draco-compressed meshes)

Draco compresses geometry. If any mesh in the GLB uses `KHR_draco_mesh_compression`, the loader will silently fail to decode geometry without this.

```js
import { DRACOLoader } from 'three/addons/loaders/DRACOLoader.js';

const dracoLoader = new DRACOLoader();
// Path must contain: draco_decoder.js + draco_decoder.wasm
dracoLoader.setDecoderPath('/draco/');          // trailing slash required
dracoLoader.setDecoderConfig({ type: 'wasm' }); // 'wasm' (default) | 'js'
dracoLoader.preload();                          // warm-up decoder worker ahead of first load

const loader = new GLTFLoader();
loader.setDRACOLoader(dracoLoader);
```

Decoder files ship with three.js at `node_modules/three/examples/jsm/libs/draco/`. Copy them to your public directory. The CDN path `https://www.gstatic.com/draco/versioned/decoders/1.5.5/` works but introduces an external dependency.

### KTX2Loader (Basis Universal compressed textures)

Required for `KHR_texture_basisu` textures in glTF.

```js
import { KTX2Loader } from 'three/addons/loaders/KTX2Loader.js';

const ktx2Loader = new KTX2Loader();
// Path must contain the Basis Universal transcoder wasm + js wrapper
ktx2Loader.setTranscoderPath('/basis/');
ktx2Loader.detectSupport(renderer); // checks GPU compressed-format support; must run after renderer is created

const loader = new GLTFLoader();
loader.setKTX2Loader(ktx2Loader);
```

Transcoder files ship at `node_modules/three/examples/jsm/libs/basis/`.

### MeshoptDecoder (EXT_meshopt_compression)

```js
import { MeshoptDecoder } from 'three/addons/libs/meshopt_decoder.module.js';

loader.setMeshoptDecoder(MeshoptDecoder);
// If MeshoptDecoder is an async factory, call it first:
// loader.setMeshoptDecoder(await MeshoptDecoder());
```

---

## load() — Callback Pattern

```js
loader.load(
  'model.glb',
  function onLoad(gltf) {
    scene.add(gltf.scene);
  },
  function onProgress(event) {
    // event.loaded / event.total may both be 0 for streamed responses
    const pct = event.total ? (event.loaded / event.total * 100).toFixed(0) : '?';
    console.log(`${pct}% loaded`);
  },
  function onError(error) {
    console.error('GLTFLoader error:', error);
  }
);
```

`onProgress` fires with a `ProgressEvent`. `event.total` is 0 when the server sends no `Content-Length` header—guard against division-by-zero.

---

## loadAsync() — Async/Await Pattern

```js
async function loadModel(url) {
  const gltf = await loader.loadAsync(url, (event) => {
    if (event.total) console.log(event.loaded / event.total);
  });
  return gltf;
}
```

`loadAsync()` is a promise wrapper around `load()`. The second argument is the same `onProgress` callback.

---

## parse() and parseAsync() — In-Memory Data

Useful when you've already fetched the binary via `fetch()` or loaded it from a file picker.

```js
const response = await fetch('model.glb');
const arrayBuffer = await response.arrayBuffer();

// Callback version
loader.parse(arrayBuffer, '', (gltf) => {
  scene.add(gltf.scene);
}, (error) => {
  console.error(error);
});

// Async version
const gltf = await loader.parseAsync(arrayBuffer, '');
```

The second argument is the resource path base used to resolve relative texture URIs. Pass `''` if all resources are embedded.

---

## The gltf Object

```js
gltf.scene        // THREE.Group  — the default scene (index 0)
gltf.scenes       // THREE.Group[] — all scenes defined in the file
gltf.animations   // THREE.AnimationClip[] — all animations
gltf.cameras      // THREE.Camera[] — all cameras defined in the file
gltf.asset        // { version, generator, copyright, minVersion, extensions, extras }
gltf.userData     // arbitrary extras from the glTF JSON
gltf.parser       // GLTFParser instance — access raw JSON via gltf.parser.json
```

Accessing `gltf.scene` is sufficient for most single-scene GLBs. For files with multiple scenes, iterate `gltf.scenes`.

Animations need an `AnimationMixer`:

```js
const mixer = new THREE.AnimationMixer(gltf.scene);
gltf.animations.forEach((clip) => mixer.clipAction(clip).play());

// In the render loop:
mixer.update(delta);
```

---

## Caching with THREE.Cache

`THREE.Cache` is a key-value store that `FileLoader` (used internally by `GLTFLoader`) checks before making network requests.

```js
import { Cache } from 'three';

Cache.enabled = true; // false by default — must be set explicitly

// After enabling, duplicate load() calls for the same URL hit memory, not the network.
// The cache key is the fully resolved URL string.

// Inspect cached entries:
console.log(Cache.files); // { 'https://example.com/model.glb': ArrayBuffer, ... }

// Manually add a pre-fetched resource:
const buffer = await fetch('model.glb').then(r => r.arrayBuffer());
Cache.add('model.glb', buffer);

// Remove one entry:
Cache.remove('model.glb');

// Flush everything:
Cache.clear();
```

`THREE.Cache` only caches the raw file bytes. Each `GLTFLoader.load()` call still parses the bytes into a new scene graph. If you need to share a *parsed* scene, clone it with `gltf.scene.clone(true)` or use drei's `useGLTF` which caches at the parsed level.

---

## loader.manager

Every loader holds a reference to a `LoadingManager` at `loader.manager`.

```js
const manager = new THREE.LoadingManager();
const loader = new GLTFLoader(manager);

// Access after construction:
console.log(loader.manager === manager); // true

// Swap the manager post-construction (rare but valid):
loader.manager = new THREE.LoadingManager();
```

---

## Custom Extension Plugins (register)

```js
loader.register((parser) => ({
  name: 'MY_CUSTOM_EXTENSION',
  // Called after the entire gltf is built:
  afterRoot: async (gltf) => {
    gltf.scene.traverse((obj) => { /* post-process */ });
  },
  // Other hooks: beforeRoot, loadMesh, loadMaterial, loadTexture,
  // getMaterialType, extendMaterialParams, createNodeMesh, etc.
}));

// Remove a registered plugin:
loader.unregister(pluginCallback);
```

---

## Complete Setup Example

```js
import * as THREE from 'three';
import { GLTFLoader } from 'three/addons/loaders/GLTFLoader.js';
import { DRACOLoader } from 'three/addons/loaders/DRACOLoader.js';
import { KTX2Loader } from 'three/addons/loaders/KTX2Loader.js';

THREE.Cache.enabled = true;

const manager = new THREE.LoadingManager();
const loader = new GLTFLoader(manager);

const draco = new DRACOLoader();
draco.setDecoderPath('/draco/');
draco.preload();
loader.setDRACOLoader(draco);

const ktx2 = new KTX2Loader();
ktx2.setTranscoderPath('/basis/');
ktx2.detectSupport(renderer);
loader.setKTX2Loader(ktx2);

try {
  const gltf = await loader.loadAsync('/models/scene.glb');
  scene.add(gltf.scene);
} catch (err) {
  console.error('Failed to load model:', err);
} finally {
  // Dispose loaders when no more models will be loaded
  draco.dispose();
  ktx2.dispose();
}
```
