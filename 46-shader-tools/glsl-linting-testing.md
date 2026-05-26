# GLSL Static Analysis: Linting and CI Validation

**Last verified:** 2026-05-26

---

## Tools Overview

| Tool | Role | Maintains |
|------|------|-----------|
| `glslangValidator` / `glslang` | Reference GLSL/ESSL validator, produces SPIR-V | Khronos Group |
| `glsl-optimizer` (deprecated) | IR-level shader optimizer | Unity Technologies (archived) |
| `dtoplak.vscode-glsllint` | VS Code extension wrapping `glslangValidator` | DanielToplak |
| `glslang` npm package | Node.js bindings for glslang (WebAssembly) | Various community ports |

---

## glslangValidator (Khronos Reference Compiler)

**GitHub:** https://github.com/KhronosGroup/glslang  
**Binary releases:** https://github.com/KhronosGroup/glslang/releases (tag `main-tot`)

glslang is the authoritative GLSL front-end. It validates GLSL ES and GLSL against the specification, and can output SPIR-V for Vulkan. For web shader development, you use it purely as a validator — the SPIR-V output is irrelevant.

### What It Validates

- **Lexical errors** — invalid tokens, unterminated strings
- **Syntax errors** — missing semicolons, unmatched braces
- **Semantic errors** — undeclared variables, type mismatches, incompatible operations
- **Stage-specific rules** — `gl_FragColor` not valid in vertex shaders, `gl_Position` not valid in fragment shaders
- **Version and extension conflicts** — using features from GLSL ES 3.00 with a `#version 100` directive
- **Precision qualifier violations** — missing `precision` in GLSL ES 1.00
- **Implicit type conversion** — GLSL does not allow implicit `int` → `float`; `1` ≠ `1.0`

### Installation

**macOS (Homebrew):**
```bash
brew install glslang
```

**Linux (apt):**
```bash
sudo apt install glslang-tools
```

**Windows (Scoop):**
```bash
scoop install glslang
```

**Or download the binary directly:**
```bash
# From GitHub releases page (main-tot tag)
# https://github.com/KhronosGroup/glslang/releases/tag/main-tot
# Binaries: glslang-master-linux-Release.zip, glslang-master-windows-Release.zip, etc.
```

Verify install:
```bash
glslangValidator --version
# Khronos glslang Shader Validator ...
```

### Basic Usage

File extension determines shader stage automatically:

| Extension | Stage |
|-----------|-------|
| `.vert` | Vertex |
| `.frag` | Fragment |
| `.comp` | Compute |
| `.geom` | Geometry |
| `.tesc` | Tessellation control |
| `.tese` | Tessellation evaluation |

```bash
# Validate a fragment shader (OpenGL)
glslangValidator shader.frag

# Validate against GLSL ES 3.00 (WebGL 2)
glslangValidator -G shader.frag

# Validate against Vulkan (SPIR-V output — for WebGPU / offline tools)
glslangValidator -V shader.frag

# Validate with explicit stage when extension is non-standard
glslangValidator -S frag shader.glsl

# Validate multiple files
glslangValidator *.vert *.frag

# Quiet mode — only print errors (exit code 0 = clean)
glslangValidator -q shader.frag
```

**Exit codes:**
- `0` — no errors
- Non-zero — errors found (number of errors)

### Common Errors Caught

```
ERROR: 0:12: 'x' : undeclared identifier
```
Variable used before declaration or outside scope.

```
ERROR: 0:8: '=' : cannot convert from 'const int' to 'highp float'
```
Implicit int-to-float conversion. Fix: `1.0` instead of `1`.

```
ERROR: 0:15: 'sampler2D' : cannot apply to a matrix
```
Wrong type passed to a function expecting a different type.

```
ERROR: 0:3: '' : No precision specified for (float)
```
Missing `precision mediump float;` in GLSL ES 1.00 (WebGL 1) shaders.

```
WARNING: 0:7: 'uTime' : variable is not being used
```
Declared uniform never referenced. Not an error, but worth cleaning up.

---

## VS Code Integration: `dtoplak.vscode-glsllint`

**Marketplace ID:** `dtoplak.vscode-glsllint`  
**Version:** 1.9.1  
**Requires:** `glslangValidator` on system PATH (or configured via `glsllint.glslangValidatorPath`)

This extension runs `glslangValidator` on every file save and reports errors as inline squiggles. It also validates GLSL strings embedded in `.js`/`.ts` files when configured.

**`.vscode/settings.json`:**

```jsonc
{
  "glsllint.glslangValidatorPath": "glslangValidator",
  "glsllint.glslangValidatorArgs": "",

  // Map .glsl files (no stage in extension) to fragment stage
  "glsllint.additionalStageAssociations": {
    "*.glsl": "frag"
  },

  // Lint GLSL inside tagged template literals
  "glsllint.languageSettings": {
    "javascript": {
      "start": "/*glsl*/`",
      "end": "`"
    },
    "typescript": {
      "start": "/*glsl*/`",
      "end": "`"
    }
  }
}
```

---

## glsl-optimizer

**GitHub:** https://github.com/aras-p/glsl-optimizer (archived, last commit ~2016)  
**Status:** Deprecated / archived. Not recommended for new projects.

Was used by Unity to optimize GLSL for mobile. Performed dead-code elimination and constant folding at the IR level. No longer maintained. Modern alternatives:

- `glslangValidator` + SPIR-V → use `spirv-opt` (SPIRV-Tools) for optimization passes
- For web: the GPU driver optimizes at runtime; source-level minification via `vite-plugin-glsl` minify option is sufficient

---

## CI Pipeline Integration

### npm Script Approach

Add a validation script to `package.json`:

```json
{
  "scripts": {
    "lint:shaders": "node scripts/validate-shaders.js",
    "lint": "eslint . && npm run lint:shaders"
  }
}
```

**`scripts/validate-shaders.js`:**

```js
import { execSync } from 'child_process';
import { globSync } from 'glob';
import path from 'path';

const shaderFiles = globSync('src/**/*.{vert,frag,glsl}');

if (shaderFiles.length === 0) {
  console.log('No shader files found.');
  process.exit(0);
}

let hasErrors = false;

for (const file of shaderFiles) {
  // Determine stage from extension; treat .glsl as fragment
  const ext   = path.extname(file).slice(1);
  const stage = ext === 'glsl' ? 'frag' : ext;

  try {
    execSync(`glslangValidator -S ${stage} "${file}"`, { stdio: 'pipe' });
    console.log(`  PASS  ${file}`);
  } catch (err) {
    console.error(`  FAIL  ${file}`);
    console.error(err.stdout.toString());
    hasErrors = true;
  }
}

process.exit(hasErrors ? 1 : 0);
```

### GitHub Actions

```yaml
# .github/workflows/shader-lint.yml
name: Shader Lint

on:
  push:
    paths: ['src/**/*.glsl', 'src/**/*.vert', 'src/**/*.frag']
  pull_request:
    paths: ['src/**/*.glsl', 'src/**/*.vert', 'src/**/*.frag']

jobs:
  validate:
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v4

      - name: Install glslang
        run: sudo apt-get install -y glslang-tools

      - name: Validate shaders
        run: |
          EXIT=0
          for f in $(find src -name '*.vert' -o -name '*.frag'); do
            glslangValidator "$f" || EXIT=1
          done
          # .glsl files need explicit stage
          for f in $(find src -name '*.glsl'); do
            glslangValidator -S frag "$f" || EXIT=1
          done
          exit $EXIT
```

### Vite Plugin Hook (Build-Time Validation)

Validate shaders at Vite build time using a custom plugin that shells out to `glslangValidator`:

```js
// vite.config.js
import { defineConfig } from 'vite';
import glsl from 'vite-plugin-glsl';
import { execSync } from 'child_process';

const glslValidatorPlugin = {
  name: 'glsl-validator',
  transform(code, id) {
    if (!/\.(vert|frag|glsl)$/.test(id)) return null;
    const ext   = id.split('.').pop();
    const stage = ext === 'glsl' ? 'frag' : ext;
    try {
      execSync(`glslangValidator -S ${stage} "${id}"`, { stdio: 'pipe' });
    } catch (err) {
      this.error(`Shader validation failed: ${id}\n${err.stdout}`);
    }
    return null; // let vite-plugin-glsl handle the actual transform
  }
};

export default defineConfig({
  plugins: [glslValidatorPlugin, glsl()]
});
```

Place `glslValidatorPlugin` before `glsl()` so validation runs on the raw source before inlining.

---

## Common GLSL Errors and Fixes

| Error message | Root cause | Fix |
|---------------|------------|-----|
| `undeclared identifier 'uTime'` | Uniform declared in JS but not in shader | Add `uniform float uTime;` to shader |
| `cannot convert from 'const int' to 'highp float'` | Integer literal in float context | Use `1.0` not `1` |
| `No precision specified for (float)` | Missing precision qualifier (GLSL ES 1.00) | Add `precision mediump float;` |
| `'main' : function already has a body` | Two `void main()` definitions (e.g., after bad `#include`) | Check `#include` guard — add `#ifndef INCLUDED_X` |
| `'gl_FragColor' : undeclared identifier` | Using GLSL ES 1.00 built-in in a `#version 300 es` shader | Declare `out vec4 fragColor;` and use that |
| `wrong operand types — no operation '+'` | Adding `vec2` to `float` without explicit cast | Use `vec2(f)` or `vec2(f, f)` |
| `sampler2D' : cannot apply to a vec2` | Passing `uv` to a function expecting a `sampler2D` | Check argument order |
| `'texture' : no matching overloaded function` | GLSL ES 1.00 does not have `texture()` | Use `texture2D()` |

---

## Sources

| Source | URL | Tier | Accessed |
|--------|-----|------|----------|
| KhronosGroup/glslang GitHub (README, error categories) | https://github.com/KhronosGroup/glslang | 1 | 2026-05-26 |
| dtoplak.vscode-glsllint marketplace page | https://marketplace.visualstudio.com/items?itemName=dtoplak.vscode-glsllint | 1 | 2026-05-26 |
| cadenas.vscode-glsllint deprecation notice | https://marketplace.visualstudio.com/items?itemName=cadenas.vscode-glsllint | 1 | 2026-05-26 |
| aras-p/glsl-optimizer GitHub (archived) | https://github.com/aras-p/glsl-optimizer | 1 | 2026-05-26 — noted as archived |
| Khronos reference compiler page | https://www.khronos.org/opengles/sdk/tools/Reference-Compiler/ — returned 404 on 2026-05-26; glslang info sourced from GitHub README | 1 | 2026-05-26 |
