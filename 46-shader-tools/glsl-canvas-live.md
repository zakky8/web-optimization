# glslCanvas: Live GLSL Editing in the Browser

**Last verified:** 2026-05-26

---

## What Is glslCanvas?

glslCanvas is a JavaScript library by Patricio Gonzalez Vivo (author of *The Book of Shaders*) that attaches a live WebGL renderer to any `<canvas>` element. Drop in a fragment shader via a `data-` attribute and the canvas renders it immediately — no build step, no boilerplate WebGL setup.

**GitHub:** https://github.com/patriciogonzalezvivo/glslCanvas  
**Used by:** https://thebookofshaders.com/edit.php (the book's inline editor)

---

## Quick Start

```html
<!-- CDN -->
<script src="https://rawgit.com/patriciogonzalezvivo/glslCanvas/master/dist/GlslCanvas.js"></script>

<!-- The canvas becomes a live shader renderer -->
<canvas class="glslCanvas"
        data-fragment-url="/shaders/my-shader.frag"
        width="500" height="500">
</canvas>
```

Or inline the shader source directly:

```html
<canvas class="glslCanvas"
        data-fragment="
          precision mediump float;
          uniform float u_time;
          void main() {
            gl_FragColor = vec4(abs(sin(u_time)), 0.5, 1.0, 1.0);
          }
        "
        width="500" height="500">
</canvas>
```

The library finds every `<canvas class="glslCanvas">` on page load and instantiates a renderer for each.

---

## Built-In Uniforms

These uniforms are injected automatically — no declaration needed in your HTML, but you must declare them in your shader source.

| Uniform | GLSL type | Description |
|---------|-----------|-------------|
| `u_time` | `float` | Elapsed time in seconds since renderer start |
| `u_resolution` | `vec2` | Canvas width and height in pixels |
| `u_mouse` | `vec2` | Mouse position in pixels (updated on `mousemove`) |
| `u_tex0` … `u_tex7` | `sampler2D` | Texture channels loaded via `data-textures` |

```glsl
precision mediump float;

uniform float u_time;
uniform vec2  u_resolution;
uniform vec2  u_mouse;

void main() {
    vec2 uv = gl_FragCoord.xy / u_resolution;
    gl_FragColor = vec4(uv, 0.5 + 0.5 * sin(u_time), 1.0);
}
```

---

## Texture Loading

### Via HTML Attribute

```html
<canvas class="glslCanvas"
        data-fragment-url="/shaders/textures.frag"
        data-textures="/assets/noise.png,/assets/normal.png"
        width="800" height="600">
</canvas>
```

Textures are loaded in order and mapped to `u_tex0`, `u_tex1`, etc.

```glsl
uniform sampler2D u_tex0;  // noise.png
uniform sampler2D u_tex1;  // normal.png

void main() {
    vec2 uv = gl_FragCoord.xy / u_resolution;
    vec4 col = texture2D(u_tex0, uv);
    gl_FragColor = col;
}
```

### Via JavaScript API

```js
const canvas  = document.querySelector('.glslCanvas');
const sandbox = new GlslCanvas(canvas);

// Load a shader string
sandbox.load(`
  precision mediump float;
  uniform float u_time;
  void main() { gl_FragColor = vec4(sin(u_time), 0.0, 0.0, 1.0); }
`);

// Load a texture into a specific channel
sandbox.setUniform('u_tex0', '/assets/paper.jpg');

// Set a custom float uniform
sandbox.setUniform('u_speed', 2.5);

// Set a vec2 uniform
sandbox.setUniform('u_offset', 0.3, 0.7);
```

---

## JavaScript API Reference

```js
const sandbox = new GlslCanvas(canvasElement, [optionsObject]);

sandbox.load(fragmentSource, [vertexSource]);   // reload shader
sandbox.setUniform(name, ...values);           // set any uniform
sandbox.setMouse({ x: 100, y: 200 });          // programmatic mouse
sandbox.destroy();                             // stop render loop, release GL
```

**Options object:**

```js
new GlslCanvas(canvas, {
    premultipliedAlpha: false,
    preserveDrawingBuffer: true,  // needed for canvas.toDataURL()
})
```

---

## Limitations

- Fragment shaders only. Vertex shader customisation is possible via the JS API but not exposed via `data-` attributes.
- No `#include` preprocessing — split across multiple canvas elements or pre-concatenate.
- No multi-pass / buffer support (use Shadertoy or a custom Three.js setup for that).
- Single canvas per GlslCanvas instance.

---

## Alternatives for Browser-Based Prototyping

| Tool | URL | Best For |
|------|-----|----------|
| **glslCanvas** | https://github.com/patriciogonzalezvivo/glslCanvas | Embedding live shaders in web pages; Book of Shaders tutorials |
| **Shadertoy** | https://www.shadertoy.com | Multi-pass shaders, audio-reactive, community sharing, buffer A/B/C/D |
| **shdr.bkcore.com** | http://shdr.bkcore.com | Minimal editor, vertex + fragment side by side, 3D mesh preview |
| **glsl.app** | https://glsl.app | Modern alternative to Shadertoy UI, clean editor |
| **The Book of Shaders editor** | https://thebookofshaders.com/edit.php | Learning context, glslCanvas underneath |

### Recommended Workflow Before Three.js Integration

```
Prototype in Shadertoy (full tooling, community sharing)
    ↓
Verify in glslCanvas (test the exact GLSL ES 1.00 / 2.00 path)
    ↓
Integrate into Three.js ShaderMaterial (see shadertoy-to-threejs.md)
    ↓
Move shader to .frag files with vite-plugin-glsl (see shader-hot-reload.md)
```

Use Shadertoy for complex multi-pass or audio-reactive work; use glslCanvas when you want to embed a live example in a webpage or verify a shader outside any framework.

---

## Sources

| Source | URL | Tier | Accessed |
|--------|-----|------|----------|
| glslCanvas GitHub README (uniforms, data-textures, JS API) | https://github.com/patriciogonzalezvivo/glslCanvas | 1 | 2026-05-26 |
| thebookofshaders.com editor | https://thebookofshaders.com/edit.php | 1 | 2026-05-26 — page returned minimal HTML; uniform list confirmed from GitHub README |
