# Caching and CDN

3D scenes often load megabytes of assets — GLB models, KTX2 textures, HDR environment maps, WASM physics engines. Without an intentional caching strategy, users re-download these assets on every visit. With the right setup, they download once and load instantly thereafter.

## Asset Categories and Cache Strategy

| Asset type | Change frequency | Cache strategy | Cache-Control header |
|---|---|---|---|
| Three.js bundle (hashed filename) | Rarely | Cache First forever | `max-age=31536000, immutable` |
| GLB models (hashed) | Per deploy | Cache First forever | `max-age=31536000, immutable` |
| KTX2 / WebP textures (hashed) | Per deploy | Cache First forever | `max-age=31536000, immutable` |
| HDR environments (hashed) | Per deploy | Cache First forever | `max-age=31536000, immutable` |
| WASM physics binary | Rarely | Cache First, check for update | `max-age=604800, stale-while-revalidate=86400` |
| index.html | Every deploy | Network First | `no-cache` |
| API/dynamic data | Frequent | Network First or SWR | `no-store` or `max-age=60` |

## Content-Hashed Filenames

Vite and other bundlers append a content hash to output filenames. This is the foundation of long-lived caching: filename changes when content changes, so cached files are safe to keep forever.

```
hero-model.glb            → hero-model.Bx3kR9pl.glb
three.min.js              → three.BdQ7YzNc.js
environment-city.hdr      → environment-city.Kw1pF8mA.hdr
```

```ts
// vite.config.ts — ensure hashed output
export default defineConfig({
  build: {
    rollupOptions: {
      output: {
        // JS chunks
        entryFileNames: "assets/[name].[hash].js",
        chunkFileNames: "assets/[name].[hash].js",
        // Static assets (GLB, images, HDR)
        assetFileNames: "assets/[name].[hash][extname]",
      },
    },
  },
});
```

## Service Worker Caching

A service worker intercepts network requests and serves them from cache when available. This enables offline support and near-instant repeat loads.

### Registration

```js
// main.js — register service worker
if ("serviceWorker" in navigator) {
  window.addEventListener("load", () => {
    navigator.serviceWorker
      .register("/sw.js")
      .then(reg => console.log("SW registered:", reg.scope))
      .catch(err => console.error("SW registration failed:", err));
  });
}
```

### Service Worker Strategies

```js
// public/sw.js

const CACHE_NAME  = "3d-assets-v2";
const STATIC_CACHE = "static-v2";

// Assets to precache at install time
const PRECACHE_URLS = [
  "/",
  "/assets/three.BdQ7YzNc.js",
  "/assets/hero-model.Bx3kR9pl.glb",
];

self.addEventListener("install", (event) => {
  event.waitUntil(
    caches.open(STATIC_CACHE)
      .then(cache => cache.addAll(PRECACHE_URLS))
      .then(() => self.skipWaiting()) // activate immediately
  );
});

self.addEventListener("activate", (event) => {
  // Delete old caches
  event.waitUntil(
    caches.keys().then(keys =>
      Promise.all(
        keys
          .filter(key => key !== CACHE_NAME && key !== STATIC_CACHE)
          .map(key => caches.delete(key))
      )
    ).then(() => self.clients.claim())
  );
});

self.addEventListener("fetch", (event) => {
  const url = new URL(event.request.url);

  // Cache First — hashed 3D assets (GLB, textures, WASM, JS chunks)
  if (isCacheFirstAsset(url)) {
    event.respondWith(cacheFirst(event.request));
    return;
  }

  // Network First — HTML documents
  if (event.request.mode === "navigate") {
    event.respondWith(networkFirst(event.request));
    return;
  }

  // Stale While Revalidate — frequently updated assets without hashes
  event.respondWith(staleWhileRevalidate(event.request));
});

function isCacheFirstAsset(url) {
  // Hashed filenames or known-stable extensions
  return (
    /\.[a-f0-9]{8,}\.(js|css|glb|ktx2|hdr|webp|wasm)$/.test(url.pathname) ||
    url.pathname.endsWith(".wasm")
  );
}

// Cache First: serve from cache, fall back to network, populate cache
async function cacheFirst(request) {
  const cached = await caches.match(request);
  if (cached) return cached;

  const response = await fetch(request);
  if (response.ok) {
    const cache = await caches.open(CACHE_NAME);
    cache.put(request, response.clone());
  }
  return response;
}

// Network First: try network, fall back to cache
async function networkFirst(request) {
  try {
    const response = await fetch(request);
    if (response.ok) {
      const cache = await caches.open(CACHE_NAME);
      cache.put(request, response.clone());
    }
    return response;
  } catch {
    const cached = await caches.match(request);
    return cached || new Response("Offline", { status: 503 });
  }
}

// Stale While Revalidate: serve cache immediately, update in background
async function staleWhileRevalidate(request) {
  const cache    = await caches.open(CACHE_NAME);
  const cached   = await cache.match(request);

  const fetchPromise = fetch(request).then(response => {
    if (response.ok) cache.put(request, response.clone());
    return response;
  });

  return cached || fetchPromise;
}
```

### Workbox (Production-Grade SW Management)

```bash
npm install workbox-window
npm install -D workbox-webpack-plugin  # or vite-plugin-pwa
```

```ts
// vite.config.ts with VitePWA
import { VitePWA } from "vite-plugin-pwa";

export default defineConfig({
  plugins: [
    VitePWA({
      strategies: "injectManifest",
      srcDir: "src",
      filename: "sw.ts",
      registerType: "autoUpdate",
      workbox: {
        globPatterns: ["**/*.{js,css,html,glb,ktx2,hdr,wasm,webp}"],
        runtimeCaching: [
          {
            // Cache First for hashed 3D assets
            urlPattern: /\.(?:glb|ktx2|hdr|wasm)$/,
            handler: "CacheFirst",
            options: {
              cacheName: "3d-assets",
              expiration: { maxAgeSeconds: 31536000 }, // 1 year
            },
          },
          {
            // Stale While Revalidate for fonts
            urlPattern: /\.(?:woff2|woff|ttf)$/,
            handler: "StaleWhileRevalidate",
            options: { cacheName: "fonts" },
          },
        ],
      },
    }),
  ],
});
```

## CDN Configuration

### Cloudflare (most common for 3D sites)

```toml
# cloudflare.toml or via dashboard Page Rules

# Long cache for hashed assets
[[headers]]
  for = "/assets/*"
  [headers.values]
    Cache-Control = "public, max-age=31536000, immutable"
    Vary = "Accept-Encoding"

# Never cache HTML
[[headers]]
  for = "/*.html"
  [headers.values]
    Cache-Control = "no-cache, no-store, must-revalidate"
    Pragma = "no-cache"

# CORS for cross-origin model loading
[[headers]]
  for = "/models/*"
  [headers.values]
    Access-Control-Allow-Origin = "*"
    Cache-Control = "public, max-age=31536000, immutable"
```

### Cache Invalidation on Deploy

```bash
# Cloudflare — purge cache on deploy
curl -X POST "https://api.cloudflare.com/client/v4/zones/$CF_ZONE_ID/purge_cache"   -H "Authorization: Bearer $CF_API_TOKEN"   -H "Content-Type: application/json"   --data '{"purge_everything":true}'
```

Better: use hashed filenames so you never need to purge. Only `index.html` needs purging on each deploy.

### Preloading Critical 3D Assets

Tell the browser to start downloading before the script requests it:

```html
<!-- In <head> -->
<!-- Preload critical model -->
<link rel="preload" href="/assets/hero-model.Bx3kR9pl.glb" as="fetch" crossorigin>

<!-- Preload environment map -->
<link rel="preload" href="/assets/environment.Kw1pF8mA.hdr" as="fetch" crossorigin>

<!-- Preconnect to CDN -->
<link rel="preconnect" href="https://cdn.example.com">
<link rel="dns-prefetch" href="https://cdn.example.com">
```

### Progressive Loading with Three.js

```js
// Show placeholder mesh while full model loads
function loadModelProgressive(url) {
  // Immediate: show low-poly placeholder
  const placeholder = createPlaceholderBox();
  scene.add(placeholder);

  const loader = new THREE.GLTFLoader();

  // Monitor progress
  loader.load(
    url,
    (gltf) => {
      scene.remove(placeholder);
      scene.add(gltf.scene);
    },
    (progress) => {
      const percent = (progress.loaded / progress.total * 100).toFixed(0);
      updateLoadingUI(percent);
    },
    (error) => console.error("Model load error:", error)
  );
}
```

### Prefetching with Intersection Observer

Start loading the next section's assets before they scroll into view:

```js
const observer = new IntersectionObserver((entries) => {
  entries.forEach(entry => {
    if (entry.isIntersecting) {
      const modelUrl = entry.target.dataset.model;
      if (modelUrl && !prefetchCache.has(modelUrl)) {
        prefetchCache.add(modelUrl);
        // Prefetch GLB (stores in browser HTTP cache)
        fetch(modelUrl, { priority: "low" });
      }
    }
  });
}, { rootMargin: "200px" }); // start 200px before visible

document.querySelectorAll("[data-model]").forEach(el => observer.observe(el));
```

## Cache Size Budgets

Browser HTTP cache limits vary by browser and disk space, but a practical budget for cached 3D assets:

| Category | Target size | Notes |
|---|---|---|
| Critical models (above fold) | < 500 KB each | Precached at SW install |
| Secondary models | < 2 MB each | Lazy loaded on demand |
| Total precache | < 5 MB | SW install budget |
| Runtime cache | < 50 MB | Browser enforces this eventually |

## Monitoring Cache Hit Rate

```js
// Log whether resources hit cache or network
// Requires PerformanceObserver API

const observer = new PerformanceObserver((list) => {
  for (const entry of list.getEntriesByType("resource")) {
    if (entry.name.match(/\.(glb|ktx2|hdr|wasm)$/)) {
      const cacheHit = entry.transferSize === 0; // 0 bytes transferred = cache hit
      console.log(`${entry.name.split("/").pop()}: ${cacheHit ? "CACHE" : "NETWORK"} (${entry.duration.toFixed(0)}ms)`);
    }
  }
});
observer.observe({ type: "resource", buffered: true });
```

## Performance Implications

- **Content-hashed filenames + `Cache-Control: immutable`** is the most impactful single change. Returning users load zero bytes for any asset that has not changed.
- **Service worker precaching** eliminates even the cache validation round-trip on repeat visits. Measured improvement: 3-12s load time → <0.5s on repeat visit for typical 3D scenes.
- **Cloudflare cache** reduces origin server load and halves latency for global users. TTFB drops from 300ms (origin) to 20ms (edge).
- **Avoid large precache lists**. Service worker install fails if any precache URL 404s. Only include assets that definitely exist at build time.
- **Cache partitioning** (browsers since 2021) prevents cross-site cache sharing. You cannot rely on users having Three.js cached from another site.

## Tools and Libraries

| Tool | Purpose |
|---|---|
| [Workbox](https://developer.chrome.com/docs/workbox/) | Production service worker toolkit |
| [vite-plugin-pwa](https://vite-pwa-org.netlify.app) | Vite + Workbox integration |
| [Cloudflare Workers](https://workers.cloudflare.com) | Edge caching with custom logic |
| [PerformanceObserver API](https://developer.mozilla.org/en-US/docs/Web/API/PerformanceObserver) | Cache hit monitoring |
| [Lighthouse](https://developer.chrome.com/docs/lighthouse/overview/) | Cache policy audit |
