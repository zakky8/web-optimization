# Astro Performance for 3D Sites

> Verified against: https://docs.astro.build/  
> Astro version: 6.x (current as of 2026-05-26)  
> Sources accessed: 2026-05-26

---

## The Core Advantage: Zero JS by Default

Astro's fundamental performance guarantee is that **no JavaScript ships to the browser unless you explicitly opt in**. A page that uses `<ThreeCanvas />` without a `client:*` directive renders as static HTML with an empty canvas tag. The 600+ KB Three.js bundle never reaches the client until you attach a hydration directive.

This inverts the Next.js / Vite SPA model: the default is zero JS, not "tree-shaken JS". For 3D sites, the implications are significant:

- Pages without 3D (About, Blog, FAQ) ship zero 3D JavaScript
- A marketing site with one 3D hero and ten static pages loads instantly for users who land on any non-hero page
- Lighthouse scores for non-3D pages are unaffected by Three.js being in the dependency tree

---

## 1. Only Load Three.js When Needed

### `client:only` — Load on arrival (for above-the-fold 3D)

```astro
<HeroScene client:only="react" />
```

JavaScript for `HeroScene` (including Three.js, R3F, and all its transitive deps) is sent to the browser as a separate chunk. Pages that do not render this component in their HTML receive none of this JavaScript.

### `client:visible` — Defer until in viewport

```astro
<ProductViewer client:visible />
```

Astro inserts an `IntersectionObserver`. The JavaScript chunk is **not requested** until the element enters the viewport. For users who never scroll to the product section, Three.js is never downloaded.

With `rootMargin` to pre-load slightly early:

```astro
<!-- Trigger hydration 300px before element enters view -->
<ProductViewer client:visible={{ rootMargin: "300px" }} />
```

### `client:idle` — Defer until browser idle

```astro
<BackgroundParticles client:idle />
```

Hydrates during `requestIdleCallback`, after page paint and critical scripts complete. Good for non-interactive ambient effects.

```astro
<!-- With a maximum wait timeout (Astro 4.15.0+) -->
<BackgroundParticles client:idle={{ timeout: 2000 }} />
```

---

## 2. Partial Hydration Architecture

The following page sends **zero Three.js** to browsers that do not scroll past the hero:

```astro
---
import HeroCanvas from '../components/HeroCanvas.astro';  // vanilla Three.js
import ProductViewer from '../components/ProductViewer';   // R3F
import BlogGrid from '../components/BlogGrid.astro';       // static HTML
---
<html>
  <body>
    <!-- Loaded immediately — visible above fold -->
    <HeroCanvas />                        <!--  0 kB extra: inline script only -->

    <!-- Static HTML section — zero JS -->
    <section>
      <BlogGrid />
    </section>

    <!-- 3D product viewer — JS deferred until scroll -->
    <div style="height:600px;">
      <ProductViewer client:visible />    <!-- ~600 kB Three.js loaded on scroll -->
    </div>

    <!-- Static footer — zero JS -->
    <footer>...</footer>
  </body>
</html>
```

The build output for this page:
- HTML: ~20 kB (full page content)
- Critical JS: 0 kB (hero uses an inline `<script>` that Astro inlines if small)
- Deferred JS: ~600 kB (ProductViewer chunk, downloaded only on scroll)

---

## 3. Bundle Analysis

Use Rollup's visualizer (works with Vite) to inspect what is going into the Three.js chunk:

```bash
npm install --save-dev rollup-plugin-visualizer
```

```js
// astro.config.mjs
import { defineConfig } from 'astro/config';
import { visualizer } from 'rollup-plugin-visualizer';

export default defineConfig({
  vite: {
    plugins: [
      visualizer({
        filename: 'dist/stats.html',
        open: true,             // auto-open after build
        gzipSize: true,
        brotliSize: true,
        template: 'treemap',    // 'treemap' | 'sunburst' | 'network'
      }),
    ],
    build: {
      rollupOptions: {
        output: {
          // Split Three.js into its own chunk for better caching
          manualChunks: {
            'three': ['three'],
            'r3f': ['@react-three/fiber', '@react-three/drei'],
          },
        },
      },
    },
  },
});
```

After `astro build`, open `dist/stats.html` to see a treemap of every module. Look for:
- Duplicate Three.js copies (should be exactly one)
- Unexpected large modules (e.g., Lodash, moment.js pulled in transitively)
- Examples or demos from Three.js that got imported accidentally

### Tree-shaking Three.js

Three.js is fully tree-shakeable when imported as named exports:

```ts
// Good — only imports what is used
import { WebGLRenderer, Scene, PerspectiveCamera, Mesh } from 'three';

// Bad — imports entire library (still tree-shakeable by Rollup but harder to analyze)
import * as THREE from 'three';
```

However, `import * as THREE` is still tree-shaken by Vite/Rollup in practice; the bundle size difference is usually negligible. The more important wins are avoiding unnecessary `drei` imports:

```ts
// Bad: imports entire drei library
import * as Drei from '@react-three/drei';

// Good: named imports — only ships what you use
import { OrbitControls, useGLTF, Environment } from '@react-three/drei';
```

---

## 4. Image Optimization with `astro:assets`

For 3D sites, optimized images matter for:
- Texture fallback images (WebP thumbnails before 3D loads)
- Preview images in model selection grids
- Environment map preview thumbnails
- OG images for social sharing

```astro
---
import { Image, Picture } from 'astro:assets';
import modelThumb from '../assets/images/product-thumb.png';
---

<!-- Image: Single format, automatic dimension inference (prevents CLS) -->
<Image
  src={modelThumb}
  alt="Product preview"
  width={400}
  height={300}
  format="webp"
  quality={80}
/>

<!-- Picture: Multiple formats with AVIF priority -->
<Picture
  src={modelThumb}
  alt="Product preview"
  formats={['avif', 'webp']}
  width={800}
  height={600}
  loading="lazy"
/>
```

Key behaviors (verified against Astro image docs, 2026-05-26):
- Images in `src/` are processed and optimized at build time
- Automatic dimension inference prevents Cumulative Layout Shift (CLS)
- Default output: WebP; use `format` prop to specify AVIF, PNG, JPEG
- Lazy loading is the default (`loading="lazy"`, `decoding="async"`)
- `getImage()` (server-only) generates optimized image URLs for use in CSS or canvas contexts
- Astro 6: images are never upscaled; crop is applied by default without requiring `fit` option
- Remote images require authorization via `image.domains` or `image.remotePatterns` in `astro.config.mjs`

```js
// astro.config.mjs — authorize remote texture thumbnails
export default defineConfig({
  image: {
    domains: ['cdn.yourassets.com'],
    remotePatterns: [{ protocol: 'https', hostname: '**.cloudfront.net' }],
  },
});
```

### Using `getImage()` for Texture Fallbacks

When you need an optimized image URL to pass as a prop (not inline HTML):

```astro
---
import { getImage } from 'astro:assets';
import rawThumb from '../assets/product-thumb.png';

// getImage() is server-only — runs at build time for static output
const thumb = await getImage({ src: rawThumb, format: 'webp', width: 400, height: 300 });
---
<!-- Pass the optimized URL to the R3F island as a fallback texture -->
<ProductViewer
  client:only="react"
  fallbackTextureUrl={thumb.src}
/>
```

---

## 5. Static Asset Strategy for 3D Files

Files in `public/` bypass Vite entirely — they are copied to `dist/` verbatim, served at the same path. This is correct for:

- `.glb` / `.gltf` model files
- `.hdr` / `.exr` environment maps
- `.ktx2` compressed textures
- Draco decoder WASM files
- Basis Universal transcoder files

```
public/
  assets/
    models/
      product.glb          → served at /assets/models/product.glb
      environment.glb
    env/
      studio.hdr
      warehouse.hdr
    textures/
      diffuse.ktx2
    draco/                 ← DRACOLoader.setDecoderPath('/assets/draco/')
      draco_decoder.js
      draco_decoder.wasm
      draco_wasm_wrapper.js
```

Do NOT import `.glb` files as Vite modules — Vite will base64-inline small files and add cache-busting hashes that break path-based loaders like `DRACOLoader`.

---

## 6. Preloading Critical 3D Assets

For above-the-fold 3D, preload the model to prevent a loading flash:

```astro
---
// src/layouts/BaseLayout.astro
---
<html>
  <head>
    <!-- Preload the hero model so it is fetched in parallel with JS -->
    <link rel="preload" as="fetch" href="/assets/models/hero.glb" crossorigin="anonymous" />
    <link rel="preload" as="fetch" href="/assets/env/studio.hdr" crossorigin="anonymous" />
  </head>
  <body>
    <slot />
  </body>
</html>
```

For Draco-compressed models, also preload the decoder:

```astro
<link rel="preload" as="script" href="/assets/draco/draco_decoder.js" />
```

---

## 7. Performance Checklist

| Item | Why it matters |
|---|---|
| Use `client:visible` for below-fold 3D | Eliminates Three.js download for users who don't scroll |
| Use `client:only` (not `client:load`) for R3F | Prevents failed SSR attempt and eliminates placeholder HTML |
| Store models in `public/` not `src/` | Avoids Vite hash mangling that breaks DRACOLoader paths |
| Set `renderer.setPixelRatio(Math.min(devicePixelRatio, 2))` | Halves GPU load on 3× screens with negligible quality loss |
| Use `manualChunks` to split `three` from app code | Lets browsers cache Three.js separately from application logic |
| Use Draco compression on all GLB models | Typically 80–90% size reduction vs. uncompressed geometry |
| Use KTX2 / Basis for textures | 4–8× GPU memory savings vs. PNG; requires KTX2Loader + BasisWorker |
| Preload hero model with `<link rel="preload">` | Removes model-loading gap from FID / LCP |
| Optimize fallback images with `astro:assets` | Prevents CLS from thumbnail placeholders |
| Run `rollup-plugin-visualizer` before each release | Catches accidental bundle bloat from transitive imports |

---

## 8. What Astro Does NOT Handle

These optimizations must be applied outside Astro:

- **DRACO / Basis compression** — use `gltf-pipeline` or Blender GLTF exporter settings
- **LOD (Level of Detail)** — implement in Three.js (`THREE.LOD`) or drei's `<Detailed>`
- **WebGL context loss recovery** — handle `webglcontextlost` / `webglcontextrestored` events manually
- **WASM thread count tuning** — set via `rapier.init(workerCount)` or equivalent
- **CDN caching headers for `public/` assets** — configure at the CDN layer (Netlify, Cloudflare, Vercel)

---

## Key Points

- Astro's zero-JS default means Three.js only ships to the browsers that need it, on the pages that use it.
- `client:visible` with `IntersectionObserver` is the most impactful single change for pages with below-fold 3D — it makes the performance cost of Three.js proportional to user behavior.
- Use `rollup-plugin-visualizer` to verify bundle composition; Three.js should appear as one deduplicated chunk.
- Store binary 3D assets in `public/` to keep file paths predictable for loader APIs.
- `astro:assets` `Image` and `Picture` components handle fallback thumbnails and preview images with automatic WebP conversion and CLS prevention.
