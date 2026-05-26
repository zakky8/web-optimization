# GLSL Hot Reload in Vite

**Last verified:** 2026-05-26  
**vite-plugin-glsl version:** 1.6.0

---

## Overview

`vite-plugin-glsl` transforms `.glsl`, `.vert`, `.frag`, `.vs`, `.fs`, and `.wgsl` imports into JavaScript strings. When `watch: true` (default), Vite's HMR pipeline picks up changes to those files and re-runs the module graph update — meaning your Three.js materials can receive new shader source without a full page reload.

---

## Installation

```bash
npm install vite-plugin-glsl --save-dev
```

---

## Plugin Configuration

```js
// vite.config.js
import { defineConfig } from 'vite';
import glsl from 'vite-plugin-glsl';

export default defineConfig({
  plugins: [
    glsl({
      // File extensions to process
      include: [
        '**/*.glsl', '**/*.wgsl',
        '**/*.vert', '**/*.frag',
        '**/*.vs', '**/*.fs',
      ],

      // Recompile shader on change (enables HMR) — default: true
      // Was called `watch` before v1.4.0; same behavior
      watch: true,

      // Minify shader string in production — default: false
      minify: false,

      // Warn when the same #include chunk appears more than once — default: true
      warnDuplicatedImports: true,

      // Default extension added when #include path has no extension
      defaultExtension: 'glsl',
    })
  ]
});
```

**TypeScript** — add to `vite-env.d.ts` (or any `.d.ts` file included in your tsconfig):

```ts
/// <reference types="vite-plugin-glsl/ext" />
```

This declares `*.glsl` etc. as `string` modules, ending red squiggles on shader imports.

---

## How HMR Works with Shader Files

When you save a `.frag` or `.glsl` file:

1. Vite's file watcher detects the change.
2. `vite-plugin-glsl` re-processes the file (resolves `#include` directives, optionally minifies).
3. Vite invalidates the importing module and sends an HMR update.
4. Your HMR accept handler receives the new shader string.
5. You apply it to the Three.js material.

Without an explicit `import.meta.hot.accept` handler, Vite falls back to a **full page reload**. The handlers below prevent that.

---

## Pattern 1: Update Uniforms Only (No Material Rebuild)

Use when the shader structure is unchanged and only uniforms need refreshing. The fastest path — no GPU recompile.

```js
import fragSrc  from './shader.frag';
import vertSrc  from './shader.vert';
import * as THREE from 'three';

const uniforms = {
  uTime:       { value: 0 },
  uResolution: { value: new THREE.Vector2(800, 600) },
};

const material = new THREE.ShaderMaterial({
  vertexShader:   vertSrc,
  fragmentShader: fragSrc,
  uniforms,
});

// HMR handlers
if (import.meta.hot) {
  import.meta.hot.accept('./shader.frag', (newModule) => {
    material.fragmentShader = newModule.default;
    material.needsUpdate    = true;       // triggers GPU recompile
  });

  import.meta.hot.accept('./shader.vert', (newModule) => {
    material.vertexShader = newModule.default;
    material.needsUpdate  = true;
  });
}
```

`material.needsUpdate = true` is the critical flag. Without it, Three.js will not recompile the WebGL program even if `fragmentShader` was replaced.

---

## Pattern 2: Full Material Rebuild

Use when the uniform block changes (added/removed uniforms) between hot reloads, or when you want a clean slate. Slightly more expensive — the old material must be disposed and the mesh re-assigned.

```js
import fragSrc  from './shader.frag';
import vertSrc  from './shader.vert';
import * as THREE from 'three';

function createUniforms() {
  return {
    uTime:       { value: 0 },
    uResolution: { value: new THREE.Vector2(800, 600) },
  };
}

function buildMaterial(frag, vert) {
  return new THREE.ShaderMaterial({
    vertexShader:   vert,
    fragmentShader: frag,
    uniforms:       createUniforms(),
  });
}

const mesh = new THREE.Mesh(geometry, buildMaterial(fragSrc, vertSrc));
scene.add(mesh);

if (import.meta.hot) {
  // Accept both shaders in one handler by tracking current sources
  let currentFrag = fragSrc;
  let currentVert = vertSrc;

  const rebuild = () => {
    mesh.material.dispose();                      // free GPU program
    mesh.material = buildMaterial(currentFrag, currentVert);
  };

  import.meta.hot.accept('./shader.frag', (mod) => {
    currentFrag = mod.default;
    rebuild();
  });

  import.meta.hot.accept('./shader.vert', (mod) => {
    currentVert = mod.default;
    rebuild();
  });
}
```

**Always call `material.dispose()`** before replacing a material. Without it, the compiled WebGL program, textures, and uniform buffers remain allocated on the GPU.

---

## Pattern 3: Centralised Shader Registry (Larger Projects)

For scenes with many materials, maintain a registry that maps each material to its shader files:

```js
// shaderRegistry.js
const registry = new Map(); // Map<material, { frag: string, vert: string }>

export function register(material, fragSrc, vertSrc) {
  registry.set(material, { frag: fragSrc, vert: vertSrc });
}

export function updateFrag(fragPath, newSrc) {
  for (const [mat, srcs] of registry.entries()) {
    if (srcs.frag === fragPath) {      // compare by path if using a naming convention
      mat.fragmentShader = newSrc;
      mat.needsUpdate    = true;
    }
  }
}
```

---

## `needsUpdate` vs Full Rebuild: Decision Guide

| Situation | Recommended approach |
|-----------|---------------------|
| Only math/color logic changed | `needsUpdate = true` (keep material, no dispose) |
| Uniform count or type changed | Full rebuild + `material.dispose()` |
| `#define` constants changed | `needsUpdate = true` (recompile triggers re-evaluation) |
| Struct layout changed | Full rebuild to be safe |
| Switching between WebGL 1 / 2 shaders | Full rebuild |

---

## Using `#include` with vite-plugin-glsl

The plugin resolves `#include` at build/watch time:

```glsl
// shader.frag
#include './utils/noise.glsl'
#include './utils/lighting.glsl'

void main() {
  float n = noise(gl_FragCoord.xy);
  gl_FragColor = vec4(n, n, n, 1.0);
}
```

When `noise.glsl` changes, the plugin re-processes `shader.frag`, and Vite sends an HMR update to any module that imported `shader.frag`. The accept handler above fires with the re-bundled string automatically.

---

## Development vs Production

```js
glsl({
  watch:  true,            // dev only — Vite disables in production build
  minify: process.env.NODE_ENV === 'production',
})
```

In production, `minify: true` strips comments and compresses whitespace, reducing the shader string size in the final bundle. It does not affect runtime behavior.

---

## Sources

| Source | URL | Tier | Accessed |
|--------|-----|------|----------|
| vite-plugin-glsl GitHub (package.json) | https://github.com/UstymUkhman/vite-plugin-glsl/blob/main/package.json | 1 | 2026-05-26 — version 1.6.0 confirmed |
| vite-plugin-glsl README (watch, HMR, options) | https://github.com/UstymUkhman/vite-plugin-glsl/blob/main/README.md | 1 | 2026-05-26 |
| Three.js ShaderMaterial.needsUpdate | https://threejs.org/docs/#api/en/materials/Material.needsUpdate | 1 | 2026-05-26 |
