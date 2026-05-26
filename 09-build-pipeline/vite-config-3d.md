# Vite Config for 3D/WebGL Projects

## Essential Plugins

```bash
npm install -D vite-plugin-glsl vite-plugin-wasm vite-plugin-top-level-await
```

## Full Config

```js
// vite.config.js
import { defineConfig } from 'vite';
import glsl from 'vite-plugin-glsl';
import wasm from 'vite-plugin-wasm';
import topLevelAwait from 'vite-plugin-top-level-await';

export default defineConfig({
  plugins: [
    // GLSL shader imports: import vert from './shader.vert'
    glsl({
      include: ['**/*.glsl', '**/*.vert', '**/*.frag', '**/*.wgsl'],
      compress: false,  // true in production for smaller bundles
      watch: true,      // HMR for shader files
    }),

    // WebAssembly support (Rapier, etc.)
    wasm(),

    // Required companion for wasm() â€” handles top-level await in WASM modules
    topLevelAwait(),
  ],

  server: {
    headers: {
      // Required for SharedArrayBuffer + OffscreenCanvas
      'Cross-Origin-Opener-Policy': 'same-origin',
      'Cross-Origin-Embedder-Policy': 'require-corp',
    }
  },

  build: {
    target: 'esnext',  // required for top-level await, BigInt
    assetsInlineLimit: 0,  // never inline 3D assets as base64
    rollupOptions: {
      output: {
        // Manual chunks â€” keep three.js separate from app code
        manualChunks: {
          'three': ['three'],
          'three-addons': ['three/addons'],
          'physics': ['@dimforge/rapier3d'],
        }
      }
    }
  },

  optimizeDeps: {
    // Exclude WASM packages from pre-bundling
    exclude: ['@dimforge/rapier3d', '@dimforge/rapier2d'],
  },

  worker: {
    // Use ES modules in workers (required for Three.js in worker)
    format: 'es',
    plugins: () => [glsl(), wasm(), topLevelAwait()],
  }
});
```

## glslify Integration

For npm-based GLSL modules (glsl-noise, etc.):

```bash
npm install -D vite-plugin-glslify glsl-noise
```

```js
import glslify from 'vite-plugin-glslify';

export default defineConfig({
  plugins: [
    glslify(),  // enables #pragma glslify import()
  ]
});
```

```glsl
// shader.vert â€” uses glslify imports
#pragma glslify: noise = require(glsl-noise/simplex/3d)

void main() {
  float n = noise(position + time);
  // ...
}
```

## Environment Variables in Shaders

```js
// vite.config.js
define: {
  'PARTICLE_COUNT': JSON.stringify(262144),
  'USE_HDR': JSON.stringify(true),
}
```

```glsl
// accessible in GLSL as a constant
#define COUNT PARTICLE_COUNT
```

## Sources
- vite-plugin-glsl: https://github.com/UstymUkhman/vite-plugin-glsl
- vite-plugin-wasm: https://github.com/Menci/vite-plugin-wasm
- Vite guide: https://vitejs.dev/guide/
