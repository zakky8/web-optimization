# Service Workers + PWA for 3D Apps

## Workbox with vite-plugin-pwa

```bash
npm install -D vite-plugin-pwa
```

```js
// vite.config.js
import { VitePWA } from 'vite-plugin-pwa';

export default defineConfig({
  plugins: [
    VitePWA({
      registerType: 'autoUpdate',
      workbox: {
        // 3D assets: cache-first (large, rarely change)
        runtimeCaching: [
          {
            urlPattern: /\.(?:glb|gltf|ktx2|bin|wasm)$/i,
            handler: 'CacheFirst',
            options: {
              cacheName: '3d-assets-v1',
              expiration: {
                maxEntries: 50,
                maxAgeSeconds: 30 * 24 * 60 * 60,  // 30 days
              },
              cacheableResponse: { statuses: [0, 200] }
            }
          },
          {
            // HDRI/large textures
            urlPattern: /\.(?:hdr|exr|png|jpg|webp)$/i,
            handler: 'StaleWhileRevalidate',
            options: {
              cacheName: 'textures-v1',
              expiration: { maxEntries: 100, maxAgeSeconds: 7 * 24 * 60 * 60 }
            }
          },
          {
            // JS/CSS bundles
            urlPattern: /\.(?:js|css)$/i,
            handler: 'StaleWhileRevalidate',
            options: { cacheName: 'static-v1' }
          }
        ],
        // Precache the app shell
        globPatterns: ['**/*.{js,css,html,ico,png,svg}'],
        // Increase default size limit for 3D asset chunks
        maximumFileSizeToCacheInBytes: 50 * 1024 * 1024  // 50MB
      },
      manifest: {
        name: '3D Experience',
        short_name: '3D',
        theme_color: '#000000',
        background_color: '#000000',
        display: 'standalone',
        icons: [
          { src: '/icon-192.png', sizes: '192x192', type: 'image/png' },
          { src: '/icon-512.png', sizes: '512x512', type: 'image/png' }
        ]
      }
    })
  ]
});
```

## Cache Strategies

| Strategy | Use For | Behavior |
|----------|---------|----------|
| `CacheFirst` | GLB, KTX2, WASM | Serve cache, update in background |
| `StaleWhileRevalidate` | Textures, HDRI | Serve cache, fetch update simultaneously |
| `NetworkFirst` | API calls, dynamic data | Try network, fall back to cache |
| `CacheOnly` | Offline-critical assets | Always serve from cache |
| `NetworkOnly` | Auth endpoints | Never cache |

## Manual Service Worker for 3D (Advanced)

```js
// sw.js
const CACHE_NAME = 'three-app-v1';
const PRECACHE = [
  '/',
  '/index.html',
  '/assets/scene.glb',
  '/assets/env.hdr',
];

self.addEventListener('install', (e) => {
  e.waitUntil(
    caches.open(CACHE_NAME).then(cache => cache.addAll(PRECACHE))
  );
});

self.addEventListener('fetch', (e) => {
  const url = new URL(e.request.url);

  // Cache-first for 3D assets
  if (/\.(glb|gltf|ktx2|bin|wasm|hdr)$/.test(url.pathname)) {
    e.respondWith(
      caches.match(e.request).then(cached => {
        if (cached) return cached;
        return fetch(e.request).then(response => {
          const clone = response.clone();
          caches.open(CACHE_NAME).then(c => c.put(e.request, clone));
          return response;
        });
      })
    );
    return;
  }

  // Network-first for everything else
  e.respondWith(
    fetch(e.request).catch(() => caches.match(e.request))
  );
});
```

## Sources
- Workbox: https://developer.chrome.com/docs/workbox/
- vite-plugin-pwa: https://vite-pwa-org.netlify.app/
- MDN Service Workers: https://developer.mozilla.org/en-US/docs/Web/API/Service_Worker_API
