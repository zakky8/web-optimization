# Astro + Three.js Setup

> Verified against: https://docs.astro.build/  
> Astro version: 6.x (current as of 2026-05-26)  
> Sources accessed: 2026-05-26

---

## 1. Project Initialization

```bash
npm create astro@latest my-3d-site
cd my-3d-site
npm install three
npm install --save-dev @types/three
```

Astro ships **zero client-side JavaScript by default**. Every Three.js scene must be explicitly activated with a client directive. That constraint is a feature — you pay only for what you use.

---

## 2. Vanilla Three.js in an `.astro` Component

### The `client:only` Pattern

Three.js manipulates the DOM directly and depends on `window`, `document`, and `canvas` APIs that do not exist in Astro's server-side rendering pass. Use `client:only` to skip SSR entirely for any component that owns a canvas.

**Option A — Inline `<script>` in an `.astro` file**

Astro processes `<script>` tags as ES modules by default: TypeScript is supported, imports are bundled, and the script is deduped when the component appears multiple times.

```astro
---
// src/components/ThreeCanvas.astro
---
<canvas id="three-canvas" style="width:100%;height:100vh;display:block;"></canvas>

<script>
  import * as THREE from 'three';

  const canvas = document.getElementById('three-canvas') as HTMLCanvasElement;
  const renderer = new THREE.WebGLRenderer({ canvas, antialias: true });
  renderer.setSize(window.innerWidth, window.innerHeight);
  renderer.setPixelRatio(Math.min(window.devicePixelRatio, 2));

  const scene = new THREE.Scene();
  const camera = new THREE.PerspectiveCamera(75, window.innerWidth / window.innerHeight, 0.1, 1000);
  camera.position.z = 5;

  const geometry = new THREE.BoxGeometry();
  const material = new THREE.MeshStandardMaterial({ color: 0x6366f1 });
  const mesh = new THREE.Mesh(geometry, material);
  scene.add(mesh);

  const light = new THREE.DirectionalLight(0xffffff, 1);
  light.position.set(1, 2, 3);
  scene.add(light);

  let animId: number;
  function animate() {
    animId = requestAnimationFrame(animate);
    mesh.rotation.x += 0.005;
    mesh.rotation.y += 0.01;
    renderer.render(scene, camera);
  }
  animate();

  // Clean up when Astro swaps the page (View Transitions)
  document.addEventListener('astro:before-swap', () => {
    cancelAnimationFrame(animId);
    renderer.dispose();
  });

  // Handle resize
  window.addEventListener('resize', () => {
    camera.aspect = window.innerWidth / window.innerHeight;
    camera.updateProjectionMatrix();
    renderer.setSize(window.innerWidth, window.innerHeight);
  });
</script>
```

Use this component in a page:

```astro
---
// src/pages/index.astro
import ThreeCanvas from '../components/ThreeCanvas.astro';
---
<html>
  <head><title>3D Scene</title></head>
  <body>
    <ThreeCanvas />
  </body>
</html>
```

No extra directive needed — the `<script>` tag runs client-side automatically. The component itself has no server-side JS footprint.

**Option B — Separate `.ts` entrypoint imported from `<script>`**

For larger scenes, keep the Three.js logic in a dedicated module so it can be tested and reused:

```
src/
  scripts/
    scene.ts        ← Three.js scene logic
  components/
    ThreeCanvas.astro
```

```astro
<!-- ThreeCanvas.astro -->
<canvas id="three-canvas"></canvas>
<script>
  import '../scripts/scene';   // Astro bundles this at build time
</script>
```

---

## 3. Vite Config for GLSL Shaders and WASM

Astro uses Vite internally. Extend the Vite config via the `vite` key in `astro.config.mjs`.

### GLSL / WGSL Shader Files

Install a Vite GLSL plugin:

```bash
npm install --save-dev vite-plugin-glsl
```

```js
// astro.config.mjs
import { defineConfig } from 'astro/config';
import glsl from 'vite-plugin-glsl';

export default defineConfig({
  vite: {
    plugins: [
      glsl({
        include: ['**/*.glsl', '**/*.vert', '**/*.frag', '**/*.wgsl'],
        compress: false,  // set true for production to strip whitespace
        watch: true,
      }),
    ],
  },
});
```

Import shaders directly in TypeScript:

```ts
import vertexShader from './shaders/vertex.vert';
import fragmentShader from './shaders/fragment.frag';

const material = new THREE.ShaderMaterial({ vertexShader, fragmentShader });
```

Add a `env.d.ts` declaration so TypeScript does not complain:

```ts
// src/env.d.ts  (Astro creates this file — append to it)
declare module '*.glsl' { const v: string; export default v; }
declare module '*.vert' { const v: string; export default v; }
declare module '*.frag' { const v: string; export default v; }
```

### WASM (e.g. Rapier physics, Bullet.js)

Vite 7 (bundled with Astro 6) supports WASM natively via the `?init` query:

```ts
// No plugin needed in Vite 7+
import init from '@dimforge/rapier3d/rapier_wasm3d_bg.wasm?init';

await init();
```

For older WASM packages that ship a JS wrapper:

```js
// astro.config.mjs
export default defineConfig({
  vite: {
    optimizeDeps: {
      exclude: ['@dimforge/rapier3d'],  // skip pre-bundling for WASM
    },
  },
});
```

---

## 4. `astro.config.mjs` for 3D Assets

```js
// astro.config.mjs
import { defineConfig } from 'astro/config';
import react from '@astrojs/react';    // only if using R3F
import glsl from 'vite-plugin-glsl';

export default defineConfig({
  // output: 'static' is the default — correct for purely static 3D sites
  output: 'static',

  integrations: [
    react(),  // remove if not using React/R3F
  ],

  vite: {
    plugins: [glsl()],

    // Ensure large WASM / draco decoder files are not inlined
    build: {
      assetsInlineLimit: 0,
    },

    // Allow Draco and Basis WASM workers to be served correctly
    server: {
      fs: {
        allow: ['..'],  // allow serving files outside root (e.g. node_modules draco)
      },
    },
  },

  // Static assets (HDR, models, LUTs) go in public/ — no config needed;
  // anything in public/ is copied to dist/ verbatim.
});
```

**Tip:** Store large binary assets (`.glb`, `.hdr`, `.ktx2`, `.exr`) in `public/assets/`. They bypass Vite's asset pipeline and are served with their original filenames, which is required for the `DRACOLoader` and `KTX2Loader` file path APIs.

---

## 5. COOP / COEP Headers for SharedArrayBuffer

`SharedArrayBuffer` (required by Rapier, Bullet.js, and WebAssembly threads) is gated behind cross-origin isolation. You must set two response headers:

```
Cross-Origin-Opener-Policy: same-origin
Cross-Origin-Embedder-Policy: require-corp
```

### Dev server (astro dev / astro preview)

```js
// astro.config.mjs
export default defineConfig({
  server: {
    headers: {
      'Cross-Origin-Opener-Policy': 'same-origin',
      'Cross-Origin-Embedder-Policy': 'require-corp',
    },
  },
});
```

The `server.headers` option (type `OutgoingHttpHeaders`, added in `astro@1.7.0`) applies to both `astro dev` and `astro preview`.

### Production (Netlify, Vercel, Cloudflare Pages)

Headers must be set at the edge/CDN layer because `astro.config.mjs` does not emit HTTP headers to static hosting.

**Netlify** — `public/_headers`:
```
/*
  Cross-Origin-Opener-Policy: same-origin
  Cross-Origin-Embedder-Policy: require-corp
```

**Vercel** — `vercel.json`:
```json
{
  "headers": [
    {
      "source": "/(.*)",
      "headers": [
        { "key": "Cross-Origin-Opener-Policy", "value": "same-origin" },
        { "key": "Cross-Origin-Embedder-Policy", "value": "require-corp" }
      ]
    }
  ]
}
```

**Cloudflare Pages** — `public/_headers` (same format as Netlify).

> **Warning:** `COEP: require-corp` blocks any cross-origin resource that does not send `Cross-Origin-Resource-Policy: cross-origin`. CDN-hosted fonts, analytics pixels, and some ad scripts will break. Test thoroughly before deploying.

---

## 6. TypeScript / IDE Setup

```json
// tsconfig.json
{
  "extends": "astro/tsconfigs/strict",
  "compilerOptions": {
    "target": "ES2022",
    "lib": ["ES2022", "DOM", "DOM.Iterable"],
    "moduleResolution": "bundler"
  }
}
```

`three` ships its own types in `@types/three` (installed above). The `strict` Astro tsconfig preset enables `noImplicitAny` and `strictNullChecks`.

---

## Key Points

- Astro 6 requires Node 22.12.0+.
- `<ViewTransitions />` was renamed to `<ClientRouter />` in Astro 5 and removed in Astro 6. Use `<ClientRouter />`.
- Inline `<script>` tags in `.astro` files are automatically treated as ES modules and bundled by Vite. No `type="module"` attribute is needed.
- Scripts defined in `<script is:inline>` are NOT bundled — use this only for scripts that must run before hydration or that reference runtime globals not available at build time.
- Vite 7 (bundled with Astro 6) ships with the new Environment API; Three.js and most WebGL libs are unaffected.
