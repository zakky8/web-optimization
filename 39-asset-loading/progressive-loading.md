# Progressive Asset Loading in Three.js

**Sources:**  
- https://threejs.org/docs/#examples/en/loaders/GLTFLoader  
- https://threejs.org/docs/#api/en/objects/LOD  
**Accessed:** 2026-05-26

---

## Overview

Three.js has no built-in "progressive streaming" for geometry (unlike video). Progressive loading is achieved by composing the existing loader primitives in deliberate sequences. Four patterns cover most production use cases:

1. Low-res placeholder → full-res swap
2. LOD loading (closest level first)
3. Simulated streaming via chunked fetch
4. Lazy loading for off-screen assets

---

## 1. Low-Res Placeholder → Full-Res Swap

Load a small, fast placeholder GLB first. When the high-res version is ready, swap it into the same scene node.

```js
const loader = new GLTFLoader();

// Step 1: load the low-res stand-in immediately
const { scene: placeholder } = await loader.loadAsync('/models/hero-lo.glb');
placeholder.name = 'hero';
scene.add(placeholder);

// Step 2: load the full-res version in the background
loader.loadAsync('/models/hero-hi.glb').then(({ scene: hiRes }) => {
  hiRes.name = 'hero';

  // Swap: copy transform from placeholder
  hiRes.position.copy(placeholder.position);
  hiRes.rotation.copy(placeholder.rotation);
  hiRes.scale.copy(placeholder.scale);

  scene.remove(placeholder);
  scene.add(hiRes);

  // Dispose placeholder geometry to free GPU memory
  placeholder.traverse((obj) => {
    if (obj.isMesh) {
      obj.geometry.dispose();
      if (Array.isArray(obj.material)) obj.material.forEach(m => m.dispose());
      else obj.material.dispose();
    }
  });
});
```

**Texture-only variant:** Keep one geometry but swap texture resolution:

```js
const lowTexture  = await new THREE.TextureLoader().loadAsync('/tex/diffuse-512.jpg');
mesh.material.map = lowTexture;
mesh.material.needsUpdate = true;

new THREE.TextureLoader().loadAsync('/tex/diffuse-4k.jpg').then((hiTex) => {
  mesh.material.map = hiTex;
  mesh.material.needsUpdate = true;
  lowTexture.dispose();
});
```

---

## 2. LOD Loading — Load LOD0 First

`THREE.LOD` lets you attach multiple meshes at distance thresholds. Load the closest (highest-detail) level first so the object is interactive immediately, then attach lower-detail levels as they arrive.

```js
// THREE.LOD constructor
const lod = new THREE.LOD();
lod.autoUpdate = true; // default — updates which level is shown each frame
scene.add(lod);

// LOD.addLevel(object, distance, hysteresis?)
// distance: camera distance at which this level becomes active
// hysteresis: prevents flickering at boundaries (0–1, default 0)

// Load LOD0 (highest detail, shown at distance 0–50)
const { scene: lod0 } = await loader.loadAsync('/models/tree-lod0.glb');
lod.addLevel(lod0, 0);

// Load remaining levels in the background
loader.loadAsync('/models/tree-lod1.glb').then(({ scene: lod1 }) => lod.addLevel(lod1, 50));
loader.loadAsync('/models/tree-lod2.glb').then(({ scene: lod2 }) => lod.addLevel(lod2, 150));
loader.loadAsync('/models/tree-lod3.glb').then(({ scene: lod3 }) => lod.addLevel(lod3, 300));
```

In the render loop, `LOD.update(camera)` selects the appropriate level:

```js
// With lod.autoUpdate = true (default), call this in your render loop:
lod.update(camera);

// Or call it manually if autoUpdate is false:
lod.autoUpdate = false;
renderer.setAnimationLoop(() => {
  lod.update(camera);
  renderer.render(scene, camera);
});
```

The `levels` array structure:

```js
// Each entry after addLevel():
lod.levels // [{ object: THREE.Object3D, distance: number, hysteresis: number }, ...]
```

`getCurrentLevel()` returns the index of the currently active level.

### LOD with a LoadingManager

Wrap all LOD loads in a shared manager to know when the full set is ready:

```js
const manager = new THREE.LoadingManager();
manager.onLoad = () => console.log('All LOD levels loaded');

const loader = new GLTFLoader(manager);
// ... load each level with loader
```

---

## 3. Streaming Geometry — Not Natively Supported

Three.js has no progressive/incremental geometry parser (like how browsers progressively render JPEG). The entire file is parsed before any geometry appears.

**Approaches to approximate streaming:**

### 3a. Chunked GLB Pre-split at Build Time

Split a large scene into multiple GLB chunks at export time (by object, material, or spatial region). Load chunks independently:

```js
const chunks = [
  '/scene/environment.glb',
  '/scene/characters.glb',
  '/scene/props.glb',
];

// Load in priority order; add each to scene as it arrives
chunks.forEach((url, i) => {
  loader.loadAsync(url).then(({ scene: chunk }) => {
    scene.add(chunk);
    console.log(`Chunk ${i + 1}/${chunks.length} loaded`);
  });
});
```

### 3b. Fetch → ArrayBuffer → parse() for Cache-then-Parse

Pre-fetch the bytes while a loading screen is visible, then parse synchronously:

```js
// Fetch and cache the bytes (no CORS issues if same-origin or correct headers)
const [buf1, buf2] = await Promise.all([
  fetch('/models/a.glb').then(r => r.arrayBuffer()),
  fetch('/models/b.glb').then(r => r.arrayBuffer()),
]);

// Parse after all bytes are ready
const gltfA = await loader.parseAsync(buf1, '');
const gltfB = await loader.parseAsync(buf2, '');
scene.add(gltfA.scene, gltfB.scene);
```

### 3c. KTX2 Basis Universal Textures

KTX2 textures transcode to the GPU's native compressed format (BC, ASTC, ETC2) at runtime, significantly reducing texture upload time. This is the closest thing to true texture streaming available without a custom server.

```js
import { KTX2Loader } from 'three/addons/loaders/KTX2Loader.js';
const ktx2 = new KTX2Loader();
ktx2.setTranscoderPath('/basis/');
ktx2.detectSupport(renderer);
// Use with GLTFLoader.setKTX2Loader(ktx2) or stand-alone:
const texture = await ktx2.loadAsync('/textures/diffuse.ktx2');
```

---

## 4. Lazy Loading Off-Screen Assets

Load assets only when they are about to become visible. Use an `IntersectionObserver` on a proxy DOM element, or check camera frustum intersections.

### IntersectionObserver Trigger (HTML/Canvas overlay)

```js
const sentinel = document.createElement('div');
sentinel.style.cssText = 'position:absolute;width:1px;height:1px;top:800px;left:0;pointer-events:none;';
document.body.appendChild(sentinel);

const observer = new IntersectionObserver((entries) => {
  if (entries[0].isIntersecting) {
    loadSection2(); // only now fetch the assets
    observer.disconnect();
  }
});
observer.observe(sentinel);
```

### Camera Frustum Check (WebGL scene)

```js
const frustum = new THREE.Frustum();
const cameraMatrix = new THREE.Matrix4();

function checkFrustum() {
  cameraMatrix.multiplyMatrices(camera.projectionMatrix, camera.matrixWorldInverse);
  frustum.setFromProjectionMatrix(cameraMatrix);

  pendingObjects.forEach(({ boundingSphere, url, loaded }) => {
    if (!loaded && frustum.intersectsSphere(boundingSphere)) {
      loader.loadAsync(url).then(({ scene: obj }) => scene.add(obj));
      pendingObjects.get(url).loaded = true;
    }
  });
}

renderer.setAnimationLoop(() => {
  checkFrustum();
  renderer.render(scene, camera);
});
```

### Distance-Based Trigger

```js
const assetTriggers = [
  { position: new THREE.Vector3(100, 0, 0), url: '/models/distant-building.glb', radius: 80 }
];

renderer.setAnimationLoop(() => {
  assetTriggers.forEach((trigger) => {
    if (!trigger.loaded && camera.position.distanceTo(trigger.position) < trigger.radius) {
      trigger.loaded = true;
      loader.loadAsync(trigger.url).then(({ scene: obj }) => scene.add(obj));
    }
  });
  renderer.render(scene, camera);
});
```

---

## Comparison

| Approach | Complexity | When to Use |
|---|---|---|
| Placeholder swap | Low | Single hero asset with noticeable load time |
| LOD loading | Low-Medium | Outdoor scenes, many repeated assets |
| Chunked GLB | Medium | Large scenes split at build time |
| Lazy / frustum-based | Medium | Open-world or paginated content |
| KTX2 textures | Medium (toolchain) | Texture-heavy scenes; reduces VRAM pressure |
| fetch → parse | Low | Parallel pre-fetch before render loop starts |
