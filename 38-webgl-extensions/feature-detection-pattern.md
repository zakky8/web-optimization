# WebGL Extension Feature Detection Patterns

Best practices for querying, enabling, and gracefully degrading around WebGL extensions.

Sources verified: MDN Web Docs (Using WebGL Extensions), web3dsurvey.com/webgl2, MDN WebGL Best Practices. Date: 2026-05-26.

---

## getExtension() vs getSupportedExtensions()

These two methods serve different purposes and should not be used interchangeably.

### gl.getSupportedExtensions()

Returns a `string[]` of all extension names the current context supports. It does **not** enable anything. Use it to:
- Enumerate available extensions at startup for capability logging
- Batch-check many extensions at once without the overhead of calling `getExtension` on each
- Build a capabilities map for your engine's feature flags

```js
const supported = new Set(gl.getSupportedExtensions() || []);

const caps = {
  anisotropy:          supported.has('EXT_texture_filter_anisotropic'),
  floatLinear:         supported.has('OES_texture_float_linear'),
  colorBufferFloat:    supported.has('EXT_color_buffer_float'),
  multiDraw:           supported.has('WEBGL_multi_draw'),
  parallelCompile:     supported.has('KHR_parallel_shader_compile'),
  s3tc:                supported.has('WEBGL_compressed_texture_s3tc'),
  astc:                supported.has('WEBGL_compressed_texture_astc'),
  etc2:                supported.has('WEBGL_compressed_texture_etc'),
  bptc:                supported.has('EXT_texture_compression_bptc'),
};

console.table(caps);  // useful for debugging
```

**Caveat:** `getSupportedExtensions()` may return `null` in some error states. Always guard with `|| []`.

### gl.getExtension(name)

**Enables** the extension and returns its object (or `null` if unavailable). You must call this before using any extension-defined tokens or methods — calling `getSupportedExtensions` alone does not activate anything.

Returns `null` if:
- The extension is not supported on this hardware/driver
- The extension name is misspelled (case-sensitive)
- The extension is a draft extension and draft extensions are not enabled in the browser

```js
// Returns the extension object with its constants and methods, or null
const ext = gl.getExtension('EXT_texture_filter_anisotropic');

if (ext) {
  // ext.TEXTURE_MAX_ANISOTROPY_EXT is now accessible
  const max = gl.getParameter(ext.MAX_TEXTURE_MAX_ANISOTROPY_EXT);
}
```

### Rule of thumb

Use `getSupportedExtensions()` for bulk capability introspection and logging. Use `getExtension()` when you need the extension object itself to access its constants and methods. For extensions you always want active, call `getExtension()` once at context init and cache the result.

---

## Vendor-prefix fallback pattern

Some extensions shipped under vendor-prefixed names in older browsers before standardisation. Always try the unprefixed name first; fall back to vendor variants only for extensions where legacy support matters.

```js
const aniso =
  gl.getExtension('EXT_texture_filter_anisotropic') ||
  gl.getExtension('MOZ_EXT_texture_filter_anisotropic') ||    // Firefox < 49
  gl.getExtension('WEBKIT_EXT_texture_filter_anisotropic');   // Safari < 9

const s3tc =
  gl.getExtension('WEBGL_compressed_texture_s3tc') ||
  gl.getExtension('MOZ_WEBGL_compressed_texture_s3tc') ||
  gl.getExtension('WEBKIT_WEBGL_compressed_texture_s3tc');
```

In 2026 the vendor-prefixed fallbacks are mainly legacy insurance; unprefixed names work in all modern browsers.

---

## Canonical extension detection pattern

The pattern that avoids both silent failures and defensive over-engineering:

```js
class WebGLCapabilities {
  constructor(gl) {
    this.gl = gl;

    // Extensions enabled once at init; null = not available
    this.EXT_texture_filter_anisotropic =
      gl.getExtension('EXT_texture_filter_anisotropic') ||
      gl.getExtension('MOZ_EXT_texture_filter_anisotropic') ||
      gl.getExtension('WEBKIT_EXT_texture_filter_anisotropic');

    this.OES_texture_float_linear =
      gl.getExtension('OES_texture_float_linear');

    this.EXT_color_buffer_float =
      gl.getExtension('EXT_color_buffer_float');

    this.KHR_parallel_shader_compile =
      gl.getExtension('KHR_parallel_shader_compile');

    this.WEBGL_multi_draw =
      gl.getExtension('WEBGL_multi_draw');

    this.EXT_disjoint_timer_query_webgl2 =
      gl.getExtension('EXT_disjoint_timer_query_webgl2');

    // Texture compression — cache on the capabilities object
    this.WEBGL_compressed_texture_s3tc =
      gl.getExtension('WEBGL_compressed_texture_s3tc') ||
      gl.getExtension('MOZ_WEBGL_compressed_texture_s3tc') ||
      gl.getExtension('WEBKIT_WEBGL_compressed_texture_s3tc');

    this.WEBGL_compressed_texture_astc =
      gl.getExtension('WEBGL_compressed_texture_astc');

    this.WEBGL_compressed_texture_etc =
      gl.getExtension('WEBGL_compressed_texture_etc');

    this.EXT_texture_compression_bptc =
      gl.getExtension('EXT_texture_compression_bptc');

    // Derived capability: safe to call getMaxAnisotropy() without null check
    this.maxAnisotropy = this.EXT_texture_filter_anisotropic
      ? gl.getParameter(
          this.EXT_texture_filter_anisotropic.MAX_TEXTURE_MAX_ANISOTROPY_EXT
        )
      : 1;
  }

  // Convenience boolean checks
  get supportsFloatRenderTargets() {
    return !!this.EXT_color_buffer_float;
  }

  get supportsFloatLinearFiltering() {
    return !!this.OES_texture_float_linear;
  }

  get supportsParallelCompile() {
    return !!this.KHR_parallel_shader_compile;
  }
}

// Usage at init:
const caps = new WebGLCapabilities(gl);
```

---

## Graceful degradation strategies

### Pattern 1: Feature flag with fallback code path

```js
// Anisotropic filtering — purely a quality enhancement, safe to skip
function setTextureAnisotropy(gl, target, caps) {
  if (caps.EXT_texture_filter_anisotropic) {
    gl.texParameterf(
      target,
      caps.EXT_texture_filter_anisotropic.TEXTURE_MAX_ANISOTROPY_EXT,
      caps.maxAnisotropy
    );
  }
  // No else needed — omitting the call just means lower quality, not a crash
}
```

### Pattern 2: Critical capability with hard fallback

```js
// Float render target — required for HDR, but LDR is a valid fallback
function createHDRFramebuffer(gl, caps, width, height) {
  const fbo = gl.createFramebuffer();
  const tex = gl.createTexture();
  gl.bindTexture(gl.TEXTURE_2D, tex);

  const internalFormat = caps.supportsFloatRenderTargets
    ? gl.RGBA16F
    : gl.RGBA8;   // LDR fallback — disable tone mapping in shader

  gl.texStorage2D(gl.TEXTURE_2D, 1, internalFormat, width, height);
  gl.bindFramebuffer(gl.FRAMEBUFFER, fbo);
  gl.framebufferTexture2D(gl.FRAMEBUFFER, gl.COLOR_ATTACHMENT0, gl.TEXTURE_2D, tex, 0);

  return { fbo, tex, isHDR: caps.supportsFloatRenderTargets };
}
```

### Pattern 3: Progressive enhancement with user feedback

```js
// Compressed textures — fall through to uncompressed only as last resort
function getBestTextureAsset(caps) {
  if (caps.EXT_texture_compression_bptc) return { url: 'diffuse.bc7.ktx', format: 'bc7' };
  if (caps.WEBGL_compressed_texture_s3tc) return { url: 'diffuse.dxt5.dds', format: 'dxt5' };
  if (caps.WEBGL_compressed_texture_etc)  return { url: 'diffuse.etc2.ktx', format: 'etc2' };
  if (caps.WEBGL_compressed_texture_astc) return { url: 'diffuse.astc.ktx', format: 'astc' };
  return { url: 'diffuse.png', format: 'rgba8' };   // uncompressed fallback
}
```

### Pattern 4: Async shader compilation with graceful sync fallback

```js
async function compileAllPrograms(gl, programDefs, caps) {
  // Step 1: kick off all compilations before polling
  const programs = programDefs.map(def => {
    const vs = gl.createShader(gl.VERTEX_SHADER);
    const fs = gl.createShader(gl.FRAGMENT_SHADER);
    gl.shaderSource(vs, def.vert);
    gl.shaderSource(fs, def.frag);
    gl.compileShader(vs);
    gl.compileShader(fs);

    const prog = gl.createProgram();
    gl.attachShader(prog, vs);
    gl.attachShader(prog, fs);
    gl.linkProgram(prog);
    return { prog, vs, fs };
  });

  if (caps.KHR_parallel_shader_compile) {
    // Step 2a: non-blocking poll until all done
    const ext = caps.KHR_parallel_shader_compile;
    await new Promise(resolve => {
      function poll() {
        const pending = programs.filter(
          ({ prog }) => !gl.getProgramParameter(prog, ext.COMPLETION_STATUS_KHR)
        );
        if (!pending.length) return resolve();
        requestAnimationFrame(poll);
      }
      poll();
    });
  }
  // Step 2b: if no extension, the next getProgramParameter(LINK_STATUS) stalls synchronously
  // — no explicit action needed, it just happens during program use

  // Step 3: check link status for all programs
  for (const { prog } of programs) {
    if (!gl.getProgramParameter(prog, gl.LINK_STATUS)) {
      throw new Error(gl.getProgramInfoLog(prog));
    }
  }

  return programs.map(p => p.prog);
}
```

---

## Three.js capabilities object

Three.js wraps extension detection in `renderer.capabilities`, populated during `WebGLRenderer` construction. You do not need to call `gl.getExtension` yourself when using Three.js — the renderer does it internally and exposes the results here.

### Key properties

```js
const caps = renderer.capabilities;

caps.isWebGL2             // boolean — true if context is WebGL2
caps.precision            // 'highp' | 'mediump' | 'lowp' — best available shader precision
caps.maxTextures          // gl.MAX_TEXTURE_IMAGE_UNITS
caps.maxVertexTextures    // gl.MAX_VERTEX_TEXTURE_IMAGE_UNITS
caps.maxTextureSize       // gl.MAX_TEXTURE_SIZE
caps.maxCubemapSize       // gl.MAX_CUBE_MAP_TEXTURE_SIZE
caps.maxAttributes        // gl.MAX_VERTEX_ATTRIBS
caps.maxVertexUniforms    // gl.MAX_VERTEX_UNIFORM_VECTORS
caps.maxVaryings          // gl.MAX_VARYING_VECTORS
caps.maxFragmentUniforms  // gl.MAX_FRAGMENT_UNIFORM_VECTORS
caps.vertexTextures       // boolean — maxVertexTextures > 0
caps.floatFragmentTextures // boolean — OES_texture_float is available
caps.floatVertexTextures  // boolean — both vertex and fragment float textures available

// Method — returns max anisotropy or 1 if EXT_texture_filter_anisotropic is absent
caps.getMaxAnisotropy()
```

### Reading Three.js internal extension state

For extensions Three.js manages internally (but does not expose through `capabilities`), you can access the raw WebGL context:

```js
const gl = renderer.getContext();
const ext = gl.getExtension('EXT_disjoint_timer_query_webgl2');  // not exposed by Three.js
```

Or check Three.js's extension manager (internal, not public API — may change between versions):

```js
// Semi-public: Three.js manages extensions in WebGLExtensions
// Available via renderer.extensions if the version exposes it
const exts = renderer.extensions;
const hasMultiDraw = exts.has('WEBGL_multi_draw');
```

### Compressed texture support in Three.js

Three.js checks for all four compressed format extensions internally and exposes them via `renderer.extensions`. The `KTX2Loader` consults `renderer.capabilities` to decide which Basis Universal transcode target to use:

```js
const ktx2Loader = new KTX2Loader()
  .setTranscoderPath('/basis/')
  .detectSupport(renderer);   // internally queries ASTC, ETC2, S3TC, BPTC support
```

When `detectSupport(renderer)` is called, `KTX2Loader` calls `renderer.getContext()` and runs the same `getExtension` checks described throughout this document. You do not need to duplicate them.

---

## What to never do

- Do not call `gl.getExtension()` every frame. Call it once at init and cache the result. The call is not free — it goes through the browser's WebGL command buffer.
- Do not use `getSupportedExtensions().includes(name)` as a substitute for `getExtension()`. An extension in the supported list is not enabled until `getExtension` is called; its constants are not accessible without the returned object.
- Do not `throw` or crash when an extension is absent unless the extension is truly non-negotiable for your application's minimum feature set. Prefer silent fallback with quality degradation.
- Do not assume an extension's presence in desktop Chrome implies its presence in mobile Chrome — GPU and driver vary wildly.
