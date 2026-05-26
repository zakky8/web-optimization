# WebGL2 Baseline Feature Matrix

What WebGL2 provides natively — no `getExtension()` call required. Understanding this boundary prevents two common mistakes: asking for extensions that are already built in (wasted code), and assuming WebGL1 extension habits carry over to WebGL2.

Sources verified: MDN WebGL2RenderingContext, MDN WebGL2 drawBuffers. Date: 2026-05-26.

---

## The boundary question

WebGL2 is based on OpenGL ES 3.0. Many capabilities that required extensions in WebGL1 (or in desktop OpenGL) are part of the ES 3.0 core specification and therefore part of the WebGL2 core specification. If a feature is in the core spec, no `getExtension()` call is needed — and `getSupportedExtensions()` will not list it, because there is no extension to enable.

---

## Features native to WebGL2 (no extension required)

### Multiple Render Targets (MRT)

**WebGL1 required:** `WEBGL_draw_buffers`  
**WebGL2:** Built into `gl.drawBuffers()`, part of the `WebGL2RenderingContext` interface.

The `gl.drawBuffers(buffers)` method specifies which color attachment points are used as draw buffers. `buffers` is an array of `gl.COLOR_ATTACHMENT0` through `gl.COLOR_ATTACHMENT15` (or `gl.NONE`).

```js
// In a WebGL2 context — no extension needed
gl.drawBuffers([
  gl.COLOR_ATTACHMENT0,  // fragment output 0 -> color texture
  gl.COLOR_ATTACHMENT1,  // fragment output 1 -> normal texture
  gl.COLOR_ATTACHMENT2,  // fragment output 2 -> depth/roughness texture
]);
```

In a WebGL2 fragment shader the outputs are declared as layout-qualified `out` variables:

```glsl
#version 300 es
precision highp float;

layout(location = 0) out vec4 fragColor;
layout(location = 1) out vec4 fragNormal;
layout(location = 2) out vec4 fragMaterial;

void main() {
  fragColor    = vec4(1.0, 0.0, 0.0, 1.0);
  fragNormal   = vec4(0.0, 1.0, 0.0, 1.0);
  fragMaterial = vec4(0.5, 0.5, 0.0, 1.0);
}
```

Maximum color attachments: `gl.MAX_DRAW_BUFFERS` (typically 8, guaranteed ≥ 4).

---

### Float and Integer Textures

**WebGL1 required:** `OES_texture_float`, `OES_texture_half_float`  
**WebGL2:** Sized internal formats are part of the core. The following are natively supported without extensions:

| Internal format | Type | Description |
|---|---|---|
| `gl.RGBA32F` | Float 32 | 4-channel full float |
| `gl.RG32F` | Float 32 | 2-channel |
| `gl.R32F` | Float 32 | 1-channel |
| `gl.RGBA16F` | Half float | 4-channel HDR |
| `gl.RG16F` | Half float | 2-channel |
| `gl.R16F` | Half float | 1-channel |
| `gl.RGBA8I` | Signed int 8 | Integer texture |
| `gl.RGBA8UI` | Unsigned int 8 | |
| `gl.RGBA16I` | Signed int 16 | |
| `gl.RGBA32I` | Signed int 32 | |
| `gl.RGBA32UI` | Unsigned int 32 | |

**Note:** While float textures are native, **rendering to float textures** and **linear filtering of float textures** still require extensions (`EXT_color_buffer_float` and `OES_texture_float_linear` respectively — see `essential-extensions.md`).

---

### Instanced Rendering

**WebGL1 required:** `ANGLE_instanced_arrays`  
**WebGL2:** Built-in core methods on `WebGL2RenderingContext`:

```js
// No extension needed in WebGL2
gl.drawArraysInstanced(mode, first, count, instanceCount);
gl.drawElementsInstanced(mode, count, type, offset, instanceCount);
gl.vertexAttribDivisor(index, divisor);
```

`vertexAttribDivisor(index, divisor)` sets how often a vertex attribute advances per instance. A `divisor` of 0 means per-vertex, 1 means per-instance, 2 means every 2 instances, etc.

---

### Vertex Array Objects (VAOs)

**WebGL1 required:** `OES_vertex_array_object`  
**WebGL2:** Built-in:

```js
const vao = gl.createVertexArray();
gl.bindVertexArray(vao);
// ... configure vertex attribute pointers ...
gl.bindVertexArray(null);  // unbind

// Later, restore entire attribute state in one call:
gl.bindVertexArray(vao);
gl.drawArrays(gl.TRIANGLES, 0, count);
```

---

### Uniform Buffer Objects (UBOs)

**WebGL1:** Not available at all.  
**WebGL2:** Core, via `bindBufferBase`, `bindBufferRange`, `getUniformBlockIndex`, `uniformBlockBinding`.

UBOs allow sharing a block of uniforms across multiple shader programs without re-uploading. Essential for large numbers of shader programs sharing per-frame matrices.

```js
// Shader source
// layout(std140) uniform PerFrame { mat4 viewProj; vec3 cameraPos; };

const ubo = gl.createBuffer();
gl.bindBuffer(gl.UNIFORM_BUFFER, ubo);
gl.bufferData(gl.UNIFORM_BUFFER, data, gl.DYNAMIC_DRAW);
gl.bindBufferBase(gl.UNIFORM_BUFFER, 0, ubo);  // bind to binding point 0

const blockIndex = gl.getUniformBlockIndex(program, 'PerFrame');
gl.uniformBlockBinding(program, blockIndex, 0);
```

---

### 3D Textures and 2D Array Textures

**WebGL1:** Not available.  
**WebGL2:** Core, via `gl.TEXTURE_3D` and `gl.TEXTURE_2D_ARRAY` targets:

```js
// 3D texture
const tex3d = gl.createTexture();
gl.bindTexture(gl.TEXTURE_3D, tex3d);
gl.texImage3D(gl.TEXTURE_3D, 0, gl.RGBA8, width, height, depth, 0, gl.RGBA, gl.UNSIGNED_BYTE, data);

// 2D array texture (array of same-size 2D slices)
const texArray = gl.createTexture();
gl.bindTexture(gl.TEXTURE_2D_ARRAY, texArray);
gl.texImage3D(gl.TEXTURE_2D_ARRAY, 0, gl.RGBA8, width, height, layers, 0, gl.RGBA, gl.UNSIGNED_BYTE, data);
```

2D array textures are useful for texture atlases accessed with a layer index in the shader, animated textures, and cascaded shadow maps.

---

### Non-Power-of-2 (NPOT) Textures

**WebGL1:** NPOT textures could not use mipmaps and required clamp-to-edge wrapping — a significant constraint.  
**WebGL2:** NPOT textures are fully supported with any filtering mode and any wrap mode (repeat, mirrored-repeat, clamp-to-edge). No restrictions.

```js
// WebGL2: this is fine with any width/height
gl.texStorage2D(gl.TEXTURE_2D, Math.log2(Math.max(w, h)) + 1, gl.RGBA8, w, h);
gl.texParameteri(gl.TEXTURE_2D, gl.TEXTURE_WRAP_S, gl.REPEAT);
gl.texParameteri(gl.TEXTURE_2D, gl.TEXTURE_MIN_FILTER, gl.LINEAR_MIPMAP_LINEAR);
gl.generateMipmap(gl.TEXTURE_2D);
```

---

### Transform Feedback

**WebGL1:** Not available.  
**WebGL2:** Core. Captures the output of the vertex shader stage into a buffer object, enabling GPU-side particle systems, physics integration, and other general-purpose vertex compute patterns.

```js
const tf = gl.createTransformFeedback();
gl.bindTransformFeedback(gl.TRANSFORM_FEEDBACK, tf);
gl.bindBufferBase(gl.TRANSFORM_FEEDBACK_BUFFER, 0, outputBuffer);

gl.beginTransformFeedback(gl.POINTS);
gl.drawArrays(gl.POINTS, 0, particleCount);
gl.endTransformFeedback();
```

Transform feedback varyings are declared at link time:
```js
gl.transformFeedbackVaryings(program, ['gl_Position', 'velocity'], gl.SEPARATE_ATTRIBS);
gl.linkProgram(program);
```

---

### Sampler Objects

**WebGL1:** Not available (texture sampling state was attached to the texture object itself).  
**WebGL2:** Core. Sampler objects decouple sampling parameters from texture objects, allowing the same texture to be sampled differently in different draw calls without state changes.

```js
const sampler = gl.createSampler();
gl.samplerParameteri(sampler, gl.TEXTURE_MIN_FILTER, gl.LINEAR_MIPMAP_LINEAR);
gl.samplerParameterf(sampler, gl.TEXTURE_MAX_ANISOTROPY_EXT, 8);  // still needs extension constant
gl.bindSampler(0, sampler);  // override texture unit 0's sampling state
```

---

### Sync Objects and Client-Wait

**WebGL1:** Not available.  
**WebGL2:** Core. `gl.fenceSync()`, `gl.clientWaitSync()`, `gl.waitSync()` allow GPU-CPU synchronisation without a full `gl.finish()` stall.

---

### Additional core WebGL2 capabilities (brief)

| Feature | Notes |
|---|---|
| `texStorage2D` / `texStorage3D` | Immutable texture allocation — preferred over `texImage2D` for perf |
| `gl.RED`, `gl.RG` internalformats | Single and two-channel textures native |
| Integer vertex attributes | `gl.vertexAttribIPointer()`, `gl.vertexAttribI4i()` |
| `gl.drawRangeElements()` | Hint to driver about vertex index range |
| `gl.blitFramebuffer()` | Fast FBO-to-FBO copies and MSAA resolve |
| `gl.renderbufferStorageMultisample()` | MSAA render buffers |
| `gl.readBuffer()` | Select which color attachment is the read source |
| `gl.PIXEL_PACK_BUFFER` / `gl.PIXEL_UNPACK_BUFFER` | Async pixel transfer via PBOs |
| `gl.copyBufferSubData()` | GPU-side buffer copies |
| `gl.getBufferSubData()` | Async GPU→CPU readback via PBO |
| Anisotropy constants in samplers | Extension constant still needed, but object is core |

---

## WebGL1 vs WebGL2: what moved from extension to core

| Capability | WebGL1 extension | WebGL2 status |
|---|---|---|
| Multiple render targets | `WEBGL_draw_buffers` | Core |
| Float textures (upload) | `OES_texture_float` | Core |
| Half-float textures (upload) | `OES_texture_half_float` | Core |
| Instanced drawing | `ANGLE_instanced_arrays` | Core |
| Vertex array objects | `OES_vertex_array_object` | Core |
| Non-power-of-2 textures | Restricted in WebGL1 | Unrestricted in WebGL2 |
| Depth textures | `WEBGL_depth_texture` | Core |
| Standard derivatives in shaders | `OES_standard_derivatives` | Core (built into GLSL ES 3.00) |
| Element index uint | `OES_element_index_uint` | Core |
| Unsigned integer types in shaders | Not available | Core (GLSL ES 3.00) |
| 3D textures | Not available | Core |
| 2D array textures | Not available | Core |
| Uniform buffer objects | Not available | Core |
| Transform feedback | Not available | Core |
| Sampler objects | Not available | Core |
| Sync objects | Not available | Core |
| MSAA renderbuffers | Not available | Core |
| `texStorage` immutable allocation | Not available | Core |
| Pixel buffer objects | Not available | Core |

---

## What WebGL2 still requires extensions for

These are NOT in the WebGL2 core and do require `getExtension()`:

| Capability | Extension |
|---|---|
| Rendering to float framebuffer attachments | `EXT_color_buffer_float` |
| Linear filtering of float textures | `OES_texture_float_linear` |
| Anisotropic filtering | `EXT_texture_filter_anisotropic` |
| GPU compressed texture formats (all families) | `WEBGL_compressed_texture_s3tc`, `WEBGL_compressed_texture_astc`, `WEBGL_compressed_texture_etc`, `EXT_texture_compression_bptc` |
| Multi-draw indirect | `WEBGL_multi_draw` |
| Base vertex / base instance draw | `WEBGL_draw_instanced_base_vertex_base_instance` |
| Async shader compile polling | `KHR_parallel_shader_compile` |
| GPU timer queries | `EXT_disjoint_timer_query_webgl2` |
| Clip control (depth range [0,1]) | `EXT_clip_control` |
| Polygon offset clamp | `EXT_polygon_offset_clamp` |
| Provoking vertex | `WEBGL_provoking_vertex` |
| Stencil texturing | `WEBGL_stencil_texturing` |
| Float blend | `EXT_float_blend` |
| Shader-explicit texture level | `EXT_shader_texture_lod` (WebGL1; in core GLSL 3.00 for WebGL2) |
