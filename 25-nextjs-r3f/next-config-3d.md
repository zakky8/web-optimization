# next.config.js Configuration for 3D / Three.js Projects in Next.js 15

> Sources verified 2026-05-26 against Next.js 16.2.6 docs, pmndrs/react-three-next, and MDN.

---

## Overview

A standard Next.js 15 project needs several configuration changes to work well with Three.js, R3F, and related libraries:

1. `transpilePackages` — tell Next.js to compile ESM packages from `node_modules`
2. COOP / COEP headers — enable `SharedArrayBuffer` for WebAssembly multithreading
3. Webpack custom rules — support GLSL shader imports
4. Turbopack custom rules — the Turbopack equivalent of webpack loader rules
5. Bundle optimization — exclude server-only or native modules from client bundles

---

## Minimum Viable Config

For a basic R3F project without shaders or WASM:

```js
// next.config.js
/** @type {import('next').NextConfig} */
const nextConfig = {
  transpilePackages: ['three', '@react-three/fiber', '@react-three/drei'],
  reactStrictMode: true,
}

module.exports = nextConfig
```

That is the baseline. Everything below adds optional capabilities on top.

---

## `transpilePackages`

### What it does

`transpilePackages` tells Next.js to run the named `node_modules` packages through the same Babel/SWC transpilation pipeline used for your own source files. Without it, packages that ship untranspiled ESM or use advanced syntax can fail with parse errors.

Three.js and several related packages fall into this category. The three.js ecosystem distributes ESM-only packages with `"type": "module"` in their `package.json`. Next.js's default bundler does not always handle these correctly without transpilation.

### Which packages to include

```js
const nextConfig = {
  transpilePackages: [
    'three',                    // Core Three.js
    '@react-three/fiber',       // R3F reconciler
    '@react-three/drei',        // Drei helpers (most commonly needed)
    '@react-three/postprocessing', // If using postprocessing effects
    'maath',                    // Math utilities used by drei
    'meshline',                 // If using meshline
    'troika-three-text',        // If using 3D text
  ],
}
```

Add packages to this list as you encounter `SyntaxError: Cannot use import statement outside a module` or similar errors from a `node_modules` package.

> **Note**: As of Next.js 13+, the `next-transpile-modules` npm package is obsolete. Use `transpilePackages` directly in `next.config.js`.

---

## COOP and COEP Headers for WebAssembly Threads

### Why these headers are needed

Some Three.js use cases require `SharedArrayBuffer`:
- **Physijs / Rapier / Cannon-es** physics with multi-threading
- **Draco decoder** with `DracoWorker` parallelism
- **KTX2 / Basis texture compression** with worker threads
- **WebGPU** in certain implementations

`SharedArrayBuffer` requires the page to be in a [cross-origin isolated](https://web.dev/articles/coop-coep) state. Two response headers enable this:

| Header | Value | Effect |
|---|---|---|
| `Cross-Origin-Opener-Policy` | `same-origin` | Isolates the browsing context from other origins |
| `Cross-Origin-Embedder-Policy` | `require-corp` | Only loads cross-origin resources that opt in via CORP header |

After setting these headers, `self.crossOriginIsolated` is `true` and `SharedArrayBuffer` is available.

### Adding headers in `next.config.js`

```js
// next.config.js
/** @type {import('next').NextConfig} */
const nextConfig = {
  transpilePackages: ['three', '@react-three/fiber', '@react-three/drei'],

  async headers() {
    return [
      {
        // Apply to all routes — adjust the source if you only need it on specific paths
        source: '/:path*',
        headers: [
          {
            key: 'Cross-Origin-Opener-Policy',
            value: 'same-origin',
          },
          {
            key: 'Cross-Origin-Embedder-Policy',
            value: 'require-corp',
          },
        ],
      },
    ]
  },
}

module.exports = nextConfig
```

### Side effects of COOP/COEP — read this before enabling

These headers have real trade-offs:

| Feature broken by COOP/COEP | Workaround |
|---|---|
| OAuth popups (Google, GitHub login) | Use redirect flow instead of popup |
| `window.open()` cross-origin communication | Use `postMessage` with explicit origin |
| Third-party payment widgets (Stripe, PayPal iframes) | They need `COEP: credentialless` or `crossOrigin="anonymous"` attribute |
| Analytics that load cross-origin scripts without CORP | Add `Cross-Origin-Resource-Policy: cross-origin` to the analytics response |

If you only need `SharedArrayBuffer` on specific routes (e.g., a dedicated 3D experience page), scope the headers:

```js
async headers() {
  return [
    {
      source: '/experience/:path*', // Only on /experience/* routes
      headers: [
        { key: 'Cross-Origin-Opener-Policy', value: 'same-origin' },
        { key: 'Cross-Origin-Embedder-Policy', value: 'require-corp' },
      ],
    },
  ]
},
```

### Verifying cross-origin isolation in the browser

```js
// In browser DevTools console
console.log(self.crossOriginIsolated) // true if headers are correctly set
```

---

## Webpack Configuration for GLSL Shaders

By default, webpack does not know how to handle `.glsl`, `.vert`, or `.frag` files. Two approaches work in Next.js 15 (webpack build mode):

### Option A: `raw-loader` + `glslify-loader` (with glslify preprocessing)

Use this when your shaders use `#pragma glslify` to import shader chunks:

```js
// next.config.js (webpack mode)
/** @type {import('next').NextConfig} */
const nextConfig = {
  transpilePackages: ['three', '@react-three/fiber', '@react-three/drei'],

  webpack(config) {
    config.module.rules.push({
      test: /\.(glsl|vs|fs|vert|frag)$/,
      exclude: /node_modules/,
      use: [
        'raw-loader',       // Returns file contents as a JS string
        'glslify-loader',   // Processes #pragma glslify imports
      ],
    })

    return config
  },
}

module.exports = nextConfig
```

Install the loaders:

```bash
npm install --save-dev raw-loader glslify glslify-loader
```

### Option B: webpack 5 `asset/source` (no extra packages)

Webpack 5's built-in `asset/source` module type returns file contents as a string, equivalent to `raw-loader`. No extra packages required:

```js
// next.config.js (webpack mode)
/** @type {import('next').NextConfig} */
const nextConfig = {
  webpack(config) {
    config.module.rules.push({
      test: /\.(glsl|vs|fs|vert|frag)$/,
      exclude: /node_modules/,
      type: 'asset/source',
    })

    return config
  },
}

module.exports = nextConfig
```

### TypeScript declarations for shader imports

Create this file in your project root (or `src/` if using the `src` directory):

```ts
// shader.d.ts  (or src/shader.d.ts)
declare module '*.glsl' {
  const value: string
  export default value
}

declare module '*.vert' {
  const value: string
  export default value
}

declare module '*.frag' {
  const value: string
  export default value
}

declare module '*.vs' {
  const value: string
  export default value
}

declare module '*.fs' {
  const value: string
  export default value
}
```

### Using imported shaders in R3F

```tsx
'use client'

import { Canvas } from '@react-three/fiber'
import { useRef } from 'react'
import vertexShader from './shaders/custom.vert'
import fragmentShader from './shaders/custom.frag'
import * as THREE from 'three'

function ShaderMesh() {
  const uniforms = useRef({
    uTime: { value: 0 },
    uColor: { value: new THREE.Color('#ff6030') },
  })

  return (
    <mesh>
      <planeGeometry args={[2, 2, 64, 64]} />
      <shaderMaterial
        vertexShader={vertexShader}
        fragmentShader={fragmentShader}
        uniforms={uniforms.current}
      />
    </mesh>
  )
}
```

---

## Turbopack Configuration for GLSL Shaders

Next.js 15 uses Turbopack as its default dev server bundler (`next dev --turbopack`). The webpack config above does not apply to Turbopack. Use `turbopack.rules` instead:

```js
// next.config.js
/** @type {import('next').NextConfig} */
const nextConfig = {
  transpilePackages: ['three', '@react-three/fiber', '@react-three/drei'],

  turbopack: {
    rules: {
      // GLSL files — raw-loader is on Turbopack's tested-compatible list
      '*.glsl': {
        loaders: ['raw-loader'],
        as: '*.js',
      },
      '*.vert': {
        loaders: ['raw-loader'],
        as: '*.js',
      },
      '*.frag': {
        loaders: ['raw-loader'],
        as: '*.js',
      },
    },
  },
}

module.exports = nextConfig
```

> **Turbopack loaders constraint**: Only loaders that return JavaScript code are supported. The `raw-loader` returns the file content as a JS module exporting a string — this is valid. Loaders that transform stylesheets or binary assets are not currently supported.

> **Turbopack + GLSL alternative**: Turbopack supports a built-in `raw` module type that does the same thing as `raw-loader`. Use `type: 'raw'` to avoid the extra dependency:

```js
turbopack: {
  rules: {
    '*.glsl': { type: 'raw' },
    '*.vert': { type: 'raw' },
    '*.frag': { type: 'raw' },
  },
},
```

### Combining webpack and Turbopack config in the same file

Both `turbopack` and `webpack` keys can coexist. Turbopack uses the `turbopack` key; classic webpack builds use the `webpack()` function:

```js
// next.config.js
/** @type {import('next').NextConfig} */
const nextConfig = {
  transpilePackages: ['three', '@react-three/fiber', '@react-three/drei'],

  // Turbopack (next dev default in Next.js 15)
  turbopack: {
    rules: {
      '*.glsl': { type: 'raw' },
      '*.vert': { type: 'raw' },
      '*.frag': { type: 'raw' },
    },
  },

  // Webpack (next build, and next dev --webpack)
  webpack(config, { isServer }) {
    // GLSL support
    config.module.rules.push({
      test: /\.(glsl|vs|fs|vert|frag)$/,
      exclude: /node_modules/,
      type: 'asset/source',
    })

    // Prevent 'sharp' and native modules from being bundled for the browser
    if (!isServer) {
      config.resolve.fallback = {
        ...config.resolve.fallback,
        fs: false,
        path: false,
        sharp: false,
      }
    }

    return config
  },
}

module.exports = nextConfig
```

---

## WebGPU Notes

WebGPU support in Three.js is experimental (via `three/src/renderers/webgpu/WebGPURenderer.js`). No special `next.config.js` changes are required beyond `transpilePackages`. However:

- WebGPU is client-side only — same rules as WebGL apply
- The `WebGPURenderer` import tree pulls in additional modules; verify they are transpiled if you see parse errors
- Cross-origin isolation headers (COOP/COEP) do not gate WebGPU; they gate `SharedArrayBuffer` specifically

---

## Excluding `sharp` from Client Bundles

If your project uses `sharp` for server-side image processing, Next.js may try to include it in client bundles when it appears in shared code. Exclude it explicitly:

```js
webpack(config, { isServer }) {
  if (!isServer) {
    config.resolve.alias = {
      ...config.resolve.alias,
      sharp: false,
    }
  }
  return config
},
```

---

## Complete Production-Ready Config

```js
// next.config.js
/** @type {import('next').NextConfig} */
const nextConfig = {
  reactStrictMode: true,

  // Transpile ESM packages that Next.js can't handle natively
  transpilePackages: [
    'three',
    '@react-three/fiber',
    '@react-three/drei',
    '@react-three/postprocessing',
    'maath',
  ],

  // COOP/COEP for SharedArrayBuffer (physics, compressed textures, WASM threads)
  // WARNING: Read the side-effects section before enabling globally.
  async headers() {
    return [
      {
        source: '/:path*',
        headers: [
          {
            key: 'Cross-Origin-Opener-Policy',
            value: 'same-origin',
          },
          {
            key: 'Cross-Origin-Embedder-Policy',
            value: 'require-corp',
          },
        ],
      },
    ]
  },

  // Turbopack loader rules (used by `next dev` in Next.js 15)
  turbopack: {
    rules: {
      '*.glsl': { type: 'raw' },
      '*.vert': { type: 'raw' },
      '*.frag': { type: 'raw' },
      '*.vs': { type: 'raw' },
      '*.fs': { type: 'raw' },
    },
  },

  // Webpack rules (used by `next build` and `next dev --webpack`)
  webpack(config, { isServer }) {
    // GLSL shader imports
    config.module.rules.push({
      test: /\.(glsl|vs|fs|vert|frag)$/,
      exclude: /node_modules/,
      type: 'asset/source',
    })

    // Exclude native/server-only modules from client bundle
    if (!isServer) {
      config.resolve.fallback = {
        ...config.resolve.fallback,
        fs: false,
        path: false,
        sharp: false,
      }
    }

    return config
  },
}

module.exports = nextConfig
```

---

## Config Key Version Reference

| Config key | Added in Next.js version | Notes |
|---|---|---|
| `transpilePackages` | 13.0.0 | Replaces `next-transpile-modules` package |
| `headers()` | 9.5.0 | Async function returning header rules |
| `turbopack` | 15.3.0 | Previously `experimental.turbo` (13.0–15.2) |
| `turbopack.rules[].type: 'raw'` | 16.2.0 | Built-in module type, no loader needed |
| `webpack()` | All versions | Standard webpack customization callback |

---

## References

- [Next.js — `transpilePackages`](https://nextjs.org/docs/app/api-reference/config/next-config-js/transpilePackages) — accessed 2026-05-26
- [Next.js — `headers()`](https://nextjs.org/docs/app/api-reference/config/next-config-js/headers) — accessed 2026-05-26
- [Next.js — `turbopack`](https://nextjs.org/docs/app/api-reference/config/next-config-js/turbopack) — accessed 2026-05-26
- [Next.js — Configuration reference](https://nextjs.org/docs/app/api-reference/config/next-config-js) — accessed 2026-05-26
- [pmndrs/react-three-next — `next.config.js`](https://github.com/pmndrs/react-three-next) — accessed 2026-05-26
- [Loopspeed — GLSL in Next.js + R3F](https://blog.loopspeed.co.uk/nextjs-setup-glsl-shaders) — accessed 2026-05-26
- [web.dev — COOP and COEP](https://web.dev/articles/coop-coep) — accessed 2026-05-26
- [MDN — SharedArrayBuffer](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/SharedArrayBuffer) — accessed 2026-05-26
