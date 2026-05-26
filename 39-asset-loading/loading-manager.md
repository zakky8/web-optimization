# THREE.LoadingManager

**Source:** https://threejs.org/docs/#api/en/loaders/managers/LoadingManager  
**Accessed:** 2026-05-26

---

## Overview

`LoadingManager` coordinates multiple loaders and fires lifecycle callbacks as a group of assets loads. It tracks which items are in-flight and notifies you when all are complete—useful for showing and hiding a loading screen.

---

## Import

```js
import { LoadingManager } from 'three';
```

---

## Constructor

```js
new LoadingManager(onLoad?, onProgress?, onError?)
```

All three arguments are optional. You can also assign callbacks on the instance after construction.

```js
// All callbacks in constructor:
const manager = new THREE.LoadingManager(
  () => console.log('all done'),
  (url, loaded, total) => console.log(url, loaded, total),
  (url) => console.error('failed:', url)
);

// Or assign after:
const manager = new THREE.LoadingManager();
manager.onLoad     = () => { /* ... */ };
manager.onProgress = (url, itemsLoaded, itemsTotal) => { /* ... */ };
manager.onError    = (url) => { /* ... */ };
manager.onStart    = (url, itemsLoaded, itemsTotal) => { /* ... */ };
```

---

## Callbacks

### onStart

```js
manager.onStart = function(url, itemsLoaded, itemsTotal) {
  // url          — URL of the first item that triggered loading
  // itemsLoaded  — items loaded so far (typically 1 here)
  // itemsTotal   — total items registered with this manager
};
```

Fires when the *first* item starts loading. Not available through the constructor; must be assigned on the instance.

### onProgress

```js
manager.onProgress = function(url, itemsLoaded, itemsTotal) {
  // url          — URL of the item that just finished loading
  // itemsLoaded  — cumulative count of completed items
  // itemsTotal   — total items registered with this manager
};
```

Fires each time *one* item finishes—not during a file download. For per-byte download progress, use the `onProgress` argument on the individual `loader.load()` call.

### onLoad

```js
manager.onLoad = function() {
  // No arguments. Fires when itemsLoaded === itemsTotal.
};
```

Fires once, when every registered item has loaded or errored. Errors do not prevent `onLoad` from firing—it fires when there is nothing left pending.

### onError

```js
manager.onError = function(url) {
  // url — the URL that failed
};
```

Fires for each failed item. `onLoad` still fires after all items (including failed ones) are settled.

---

## Methods

### addHandler(regex, loader)

Routes URLs matching a pattern to a specific loader, bypassing normal loader selection.

```js
manager.addHandler(/\.tga$/i, new TGALoader());
manager.addHandler(/\.exr$/i, new EXRLoader());
```

### removeHandler(regex)

Removes the handler registered for the given pattern.

```js
manager.removeHandler(/\.tga$/i);
```

### setURLModifier(callback)

Intercepts every URL before a request is made. Useful for adding tokens, swapping CDN origins, or redirecting to local blobs.

```js
manager.setURLModifier((url) => {
  // Rewrite to a local object URL from a pre-fetched cache:
  const blob = preloadedBlobs.get(url);
  return blob ? URL.createObjectURL(blob) : url;
});
```

Return the (possibly modified) URL string.

### itemStart(url) / itemEnd(url) / itemError(url)

Called internally by loaders. You can call them yourself to register custom async work under the manager's tracking:

```js
manager.itemStart('my-custom-resource');
doSomeWork().then(() => manager.itemEnd('my-custom-resource'))
             .catch(() => manager.itemError('my-custom-resource'));
```

---

## DEFAULT_LOADING_MANAGER

```js
import { DefaultLoadingManager } from 'three';
```

A globally shared `LoadingManager` instance. Any loader constructed without an explicit manager uses this. Attaching callbacks to it lets you monitor all load activity across the app without passing a manager instance around:

```js
THREE.DefaultLoadingManager.onProgress = (url, loaded, total) => {
  console.log(`${url}: ${loaded}/${total}`);
};
```

Avoid mutating `DefaultLoadingManager` in library code—it affects every other loader in the page.

---

## Loading Screen Pattern

A standard implementation that shows an overlay, tracks progress, and hides when complete.

### HTML

```html
<div id="loading-screen">
  <div id="loading-bar">
    <div id="loading-fill"></div>
  </div>
  <p id="loading-label">Loading…</p>
</div>
```

### CSS (minimal)

```css
#loading-screen {
  position: fixed; inset: 0;
  background: #000;
  display: flex; flex-direction: column;
  align-items: center; justify-content: center;
  z-index: 9999;
  transition: opacity 0.4s;
}
#loading-screen.hidden { opacity: 0; pointer-events: none; }
#loading-bar { width: 300px; height: 4px; background: #333; border-radius: 2px; }
#loading-fill { height: 100%; background: #fff; width: 0; transition: width 0.1s; }
```

### JavaScript

```js
const screen = document.getElementById('loading-screen');
const fill   = document.getElementById('loading-fill');
const label  = document.getElementById('loading-label');

const manager = new THREE.LoadingManager();

manager.onStart = (url, loaded, total) => {
  screen.classList.remove('hidden');
};

manager.onProgress = (url, loaded, total) => {
  const pct = Math.round((loaded / total) * 100);
  fill.style.width  = pct + '%';
  label.textContent = `${pct}%  —  ${url.split('/').pop()}`;
};

manager.onLoad = () => {
  // Small delay so the 100% state is visible
  setTimeout(() => screen.classList.add('hidden'), 300);
};

manager.onError = (url) => {
  label.textContent = `Failed to load: ${url}`;
  label.style.color = 'red';
};

// Pass the manager to every loader:
const gltfLoader    = new GLTFLoader(manager);
const textureLoader = new THREE.TextureLoader(manager);
```

---

## Gotchas

- `onProgress` counts *completed items*, not bytes. A 200 MB file loading is one item; ten 1 KB icons are ten items. Byte-level progress requires the `onProgress` callback on each individual `loader.load()` call.
- `onLoad` fires even when some items errored. Check `onError` separately if you need to gate on clean loads.
- If you add more items *after* `onLoad` fires (lazy loading), `onLoad` will fire again when those items complete.
- `THREE.Cache.enabled = true` caches raw file bytes. The second load of the same URL calls `onProgress` immediately (cache hit) but still counts as one item.
