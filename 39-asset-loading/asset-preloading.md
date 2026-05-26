# Asset Preloading Strategies

**Sources:**  
- https://html.spec.whatwg.org/multipage/links.html#link-type-preload  
- https://developer.mozilla.org/en-US/docs/Web/HTML/Attributes/rel/preload  
- https://developer.mozilla.org/en-US/docs/Web/HTML/Attributes/rel/prefetch  
- https://developer.mozilla.org/en-US/docs/Web/HTML/Attributes/rel/modulepreload  
**Accessed:** 2026-05-26

---

## Overview

Preloading tells the browser to fetch resources before they are requested by JavaScript, using the browser's fetch priority infrastructure. The choice of mechanism depends on timing: same-navigation vs. future navigation, and resource type.

| Mechanism | When resource is needed | Browser fetch priority |
|---|---|---|
| `<link rel="preload">` | Current page, soon | High (same navigation) |
| `<link rel="prefetch">` | Future page / next scene | Idle / Low |
| `<link rel="modulepreload">` | JS modules (same page) | High |
| `fetch()` + Cache API | Full control over timing | Programmatic |

---

## `<link rel="preload">` for GLB Files

`preload` instructs the browser to fetch the resource with high priority and store it in the HTTP cache. When Three.js later `fetch()`-es the same URL (which `FileLoader` does internally), the response comes from cache with no network round-trip.

```html
<!-- In <head> -->
<link rel="preload" href="/models/hero.glb" as="fetch" crossorigin="anonymous">
<link rel="preload" href="/textures/env.hdr" as="fetch" crossorigin="anonymous">
```

- `as="fetch"` is the correct value for GLB, HDR, and other binary files the browser doesn't natively understand.
- `crossorigin="anonymous"` is required whenever `crossorigin` would be sent on the eventual `fetch()` call. Three.js's `FileLoader` sends `credentials: 'same-origin'` (default), which maps to `crossorigin="anonymous"`. Mismatched CORS attributes cause a double-fetch.
- The URL must match exactly (including query string) what the loader will request.

### Verifying a preload hit

Open DevTools → Network. The GLB request should show `(disk cache)` or `(memory cache)` if the preload was used. If it shows a second network request, the URL or `crossorigin` attribute is mismatched.

---

## `<link rel="prefetch">` for Next-Scene Assets

`prefetch` requests the resource at *idle* priority for use in a *future* navigation or user action. It does not affect current page load performance.

```html
<!-- Prefetch assets for the next scene while the user is on scene 1 -->
<link rel="prefetch" href="/models/scene2-environment.glb" as="fetch" crossorigin="anonymous">
<link rel="prefetch" href="/models/scene2-characters.glb" as="fetch" crossorigin="anonymous">
```

Trigger prefetch dynamically when you predict the user is about to transition:

```js
function prefetchNextScene(urls) {
  urls.forEach((url) => {
    const link = document.createElement('link');
    link.rel  = 'prefetch';
    link.as   = 'fetch';
    link.crossOrigin = 'anonymous';
    link.href = url;
    document.head.appendChild(link);
  });
}

// Example: prefetch on mouse-enter of a navigation button
nextButton.addEventListener('mouseenter', () => {
  prefetchNextScene(['/models/scene2-environment.glb']);
}, { once: true });
```

Note: `prefetch` is a hint only. The browser may ignore it under memory pressure or on slow connections. Do not rely on it for critical-path assets.

---

## `<link rel="modulepreload">` for JS Chunks

`modulepreload` is specifically for ES module scripts (`.js`). It parses and compiles the module in addition to fetching it, unlike plain `preload`.

```html
<!-- Preload the Three.js chunk and the GLTFLoader chunk -->
<link rel="modulepreload" href="/assets/three-abc123.js">
<link rel="modulepreload" href="/assets/GLTFLoader-xyz789.js">
```

With Vite or Webpack, these tags are often injected automatically via the `modulepreload` polyfill or the build output. Check your bundler's output HTML before adding them manually.

```js
// Vite vite.config.js — modulepreload is on by default:
export default {
  build: {
    modulePreload: { polyfill: true } // injects polyfill for Safari < 17
  }
}
```

For dynamically imported chunks:

```js
// Trigger preload of an async chunk before it's needed
const link = document.createElement('link');
link.rel  = 'modulepreload';
link.href = '/assets/SceneB-abc123.js';
document.head.appendChild(link);

// Later, the dynamic import hits the preload cache:
const { SceneB } = await import('/assets/SceneB-abc123.js');
```

---

## Preload with fetch() + Cache API

For maximum control—especially when you want to pre-parse or store assets across sessions—use the Cache API directly. Unlike `<link rel="preload">`, the Cache API persists across pages (similar to a service worker cache).

### Fetch into the HTTP Cache (simplest)

```js
// Fire-and-forget fetch — browser caches the response
function backgroundFetch(url) {
  fetch(url, { priority: 'low' }).catch(() => {}); // ignore errors; this is opportunistic
}

// Call during idle time
if ('requestIdleCallback' in window) {
  requestIdleCallback(() => backgroundFetch('/models/level2.glb'));
} else {
  setTimeout(() => backgroundFetch('/models/level2.glb'), 2000);
}
```

### Fetch + Cache API for Offline / Repeatable Access

```js
const CACHE_NAME = 'three-assets-v1';

async function preloadToCache(urls) {
  const cache = await caches.open(CACHE_NAME);
  await cache.addAll(urls); // fetches and stores all
}

async function loadFromCache(url) {
  const cache  = await caches.open(CACHE_NAME);
  const cached = await cache.match(url);
  if (cached) return cached.arrayBuffer();
  const fresh = await fetch(url);
  await cache.put(url, fresh.clone());
  return fresh.arrayBuffer();
}

// Pre-fetch during a loading screen:
await preloadToCache([
  '/models/hero.glb',
  '/textures/env.ktx2',
]);

// Later, in GLTFLoader.parse() flow:
const buffer = await loadFromCache('/models/hero.glb');
const gltf   = await loader.parseAsync(buffer, '');
```

### Fetch + THREE.Cache (Three.js-level caching)

```js
THREE.Cache.enabled = true;

async function preloadIntoThreeCache(url) {
  const buffer = await fetch(url).then(r => r.arrayBuffer());
  THREE.Cache.add(url, buffer); // the key must match exactly what GLTFLoader will request
}

await preloadIntoThreeCache('/models/hero.glb');

// GLTFLoader.load('/models/hero.glb', ...) now hits THREE.Cache, no network request
const gltf = await loader.loadAsync('/models/hero.glb');
```

---

## Priority Summary and When to Use Each

```
Page load    ←———————————————————————————→  Future session
high priority                               low / persisted

<link modulepreload>  <link preload>  fetch+idle  <link prefetch>  Cache API
JS modules only       current page    current pg  next page        cross-session
```

### Recommended combinations

**Single-page Three.js app (Vite/bundler):**
- `<link rel="modulepreload">` for JS chunks (often automatic)
- `<link rel="preload" as="fetch">` for the primary GLB in `<head>`
- `prefetch` or `requestIdleCallback + fetch` for secondary scenes

**Multi-page / scene router:**
- `<link rel="prefetch">` on route transition hover/intent
- Cache API (or service worker) for repeat visitors

**Offline / installable PWA:**
- Service worker with `cache.addAll()` on install
- Cache API lookback before every `GLTFLoader` call
