# VS Code GLSL Development Setup

**Last verified:** 2026-05-26

---

## Extension Stack

### 1. glsl-literal — Syntax Highlighting in Template Literals

- **Marketplace ID:** `boyswan.glsl-literal`
- **Publisher:** boyswan
- **Installs:** ~36,700
- **Rating:** 5/5

Provides GLSL syntax highlighting inside JavaScript/TypeScript tagged template literals. Activates on the tag names `glsl`, `glslify`, `frag`, and `vert`.

```js
// Without a processing library, define an identity tag first
const glsl = x => x;

const fragmentShader = glsl`
  precision mediump float;
  uniform float uTime;
  void main() {
    gl_FragColor = vec4(sin(uTime), 0.5, 1.0, 1.0);
  }
`;
```

If you use `vite-plugin-glsl` with `.glsl` files, this extension handles the inline string case — useful for small utility shaders or test snippets kept directly in a `.js`/`.ts` file.

---

### 2. Shader Languages Support for VS Code

- **Marketplace ID:** `slevesque.shader`
- **Publisher:** slevesque
- **Version:** 1.1.5
- **Installs:** 1.2M+

Syntax highlighter and IntelliSense for standalone shader files. Supports:

| Format | Extension |
|--------|-----------|
| GLSL | `.glsl`, `.vert`, `.frag`, `.vs`, `.fs` |
| HLSL | `.hlsl`, `.fx` |
| Cg | `.cg` |

HLSL gets the most complete IntelliSense (completions, signatures, hover docs). GLSL gets syntax coloring and basic completions. Install this alongside `boyswan.glsl-literal` — they serve different surfaces (files vs. template literals).

---

### 3. GLSL Lint

- **Marketplace ID:** `dtoplak.vscode-glsllint`  
  *(The original `cadenas.vscode-glsllint` v1.5.2 is deprecated; install the DanielToplak fork instead.)*
- **Publisher:** DanielToplak
- **Version:** 1.9.1

Wraps `glslangValidator` (Khronos Reference Compiler) to give real-time error squiggles in the editor. Validates `.vert`, `.frag`, `.comp`, `.geom`, `.tesc`, `.tese`, and ray-tracing shader types.

**Configuration (`.vscode/settings.json`):**

```jsonc
{
  // Path to glslangValidator binary — omit if on system PATH
  "glsllint.glslangValidatorPath": "glslangValidator",

  // Extra flags, e.g. -V for Vulkan SPIR-V validation
  "glsllint.glslangValidatorArgs": "",

  // Map non-standard extensions to shader stages
  "glsllint.additionalStageAssociations": {
    "*.glsl": "frag"   // treat bare .glsl files as fragment shaders
  },

  // Enable glslify pragma support
  "glsllint.glslifyPattern": "glslify",

  // Also lint GLSL strings embedded in .js/.ts files
  "glsllint.languageSettings": {
    "javascript": { "start": "glsl`", "end": "`" },
    "typescript": { "start": "glsl`", "end": "`" }
  }
}
```

Requires `glslangValidator` to be installed separately — see the `glsl-linting-testing.md` file in this folder for installation instructions.

---

### 4. glsl-canvas — Live WebGL Preview

- **Marketplace ID:** `circledev.glsl-canvas`
- **Publisher:** circledev
- **Version:** 0.2.15
- **Installs:** ~170,800

Opens a side-panel WebGL canvas that renders the active fragment shader in real time. Supports WebGL and WebGL2.

**Key capabilities:**

| Feature | Detail |
|---------|--------|
| Preview modes | Flat, box, sphere, torus, OBJ mesh |
| Uniform inspector | Color picker for `vec3`/`vec4` uniforms |
| Texture channels | Map local or remote images to sampler uniforms |
| Performance | stats.js overlay |
| Export | WebM video, standalone HTML |
| `#include` | Resolves relative paths |

**Usage:** Open a `.frag` or `.glsl` file, then run `Cmd/Ctrl+Shift+P` → "Show glslCanvas".

**Built-in uniforms injected by glsl-canvas:**

```glsl
uniform float     u_time;        // elapsed seconds
uniform vec2      u_resolution;  // canvas width/height in px
uniform vec2      u_mouse;       // mouse position in px
uniform sampler2D u_texture_N;   // texture channels (0-indexed)
```

---

## Recommended `.vscode/extensions.json`

```json
{
  "recommendations": [
    "boyswan.glsl-literal",
    "slevesque.shader",
    "dtoplak.vscode-glsllint",
    "circledev.glsl-canvas"
  ]
}
```

---

## Setup for vite-plugin-glsl Workflow

`vite-plugin-glsl` v1.6.0 lets you import `.glsl`, `.vert`, `.frag`, `.vs`, `.fs`, and `.wgsl` files directly as strings with HMR support. The VS Code extensions above pair with it as follows:

```
.glsl file in editor
  → slevesque.shader  (syntax color + completions)
  → dtoplak.vscode-glsllint  (live error squiggles via glslangValidator)
  → circledev.glsl-canvas  (live render preview in side panel)
  → vite-plugin-glsl  (imports the file as a string, resolves #include, HMR)
```

**`vite.config.js`:**

```js
import { defineConfig } from 'vite';
import glsl from 'vite-plugin-glsl';

export default defineConfig({
  plugins: [
    glsl({
      include: ['**/*.glsl', '**/*.vert', '**/*.frag'],
      watch: true,       // HMR on file change (default: true)
      minify: false,     // set true for production builds
      warnDuplicatedImports: true,
    })
  ]
});
```

**TypeScript declarations** — add to `vite-env.d.ts` or a local `glsl.d.ts`:

```ts
/// <reference types="vite-plugin-glsl/ext" />
```

This gives `import shader from './shader.glsl'` a `string` type without manual module declarations.

---

## Workspace Snippet: GLSL Fragment Boilerplate

Add to `.vscode/snippets/glsl.json`:

```json
{
  "Fragment shader boilerplate": {
    "prefix": "glsl-frag",
    "body": [
      "precision mediump float;",
      "",
      "uniform float uTime;",
      "uniform vec2  uResolution;",
      "",
      "void main() {",
      "  vec2 uv = gl_FragCoord.xy / uResolution;",
      "  $0",
      "  gl_FragColor = vec4(uv, 0.5 + 0.5 * sin(uTime), 1.0);",
      "}"
    ],
    "description": "Basic fragment shader with common uniforms"
  }
}
```

---

## Sources

| Source | URL | Tier | Accessed |
|--------|-----|------|----------|
| boyswan.glsl-literal marketplace page | https://marketplace.visualstudio.com/items?itemName=boyswan.glsl-literal | 1 | 2026-05-26 |
| slevesque.shader marketplace page | https://marketplace.visualstudio.com/items?itemName=slevesque.shader | 1 | 2026-05-26 |
| dtoplak.vscode-glsllint marketplace page | https://marketplace.visualstudio.com/items?itemName=dtoplak.vscode-glsllint | 1 | 2026-05-26 |
| cadenas.vscode-glsllint marketplace page (deprecated notice) | https://marketplace.visualstudio.com/items?itemName=cadenas.vscode-glsllint | 1 | 2026-05-26 |
| circledev.glsl-canvas marketplace page | https://marketplace.visualstudio.com/items?itemName=circledev.glsl-canvas | 1 | 2026-05-26 |
| vite-plugin-glsl GitHub (package.json v1.6.0) | https://github.com/UstymUkhman/vite-plugin-glsl | 1 | 2026-05-26 |
| vite-plugin-glsl README (watch/HMR options) | https://github.com/UstymUkhman/vite-plugin-glsl/blob/main/README.md | 1 | 2026-05-26 |
