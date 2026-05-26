# Essential WebGL2 Extensions

Every production 3D web application should check for and leverage these three extensions. They address near-universal gaps between what WebGL2 guarantees and what real-world rendering pipelines need.

Sources verified: MDN Web Docs, web3dsurvey.com/webgl2. Date: 2026-05-26.

---

## 1. EXT_texture_filter_anisotropic

### What it does

Without this extension, mipmapped textures degrade to a blurry grey at steep viewing angles — a visible artefact on any floor, road, or terrain surface. Anisotropic filtering (AF) samples the texture along the elongated footprint of the pixel on the surface, producing sharp results at oblique angles with minimal performance cost on any GPU built after 2010.

Support rate among WebGL2 contexts in the wild: **99.05%** (web3dsurvey.com, accessed 2026-05-26). Treat it as universally available.

### Constants

| Constant | Usage |
|---|---|
| `ext.TEXTURE_MAX_ANISOTROPY_EXT` | Pass to `gl.texParameterf()` to set the AF level for the bound texture |
| `ext.MAX_TEXTURE_MAX_ANISOTROPY_EXT` | Pass to `gl.getParameter()` to query the device ceiling (typically 4–16) |

### How to enable and use

```js
// Enable once at init time. Returns null if unavailable (rare).
const ext =
  gl.getExtension('EXT_texture_filter_anisotropic') ||
  gl.getExtension('MOZ_EXT_texture_filter_anisotropic') ||    // legacy Firefox
  gl.getExtension('WEBKIT_EXT_texture_filter_anisotropic');   // legacy Safari

// Query device maximum (clamp to this — exceeding it has no effect)
const maxAnisotropy = ext
  ? gl.getParameter(ext.MAX_TEXTURE_MAX_ANISOTROPY_EXT)
  : 1;

// Apply to a texture after binding it
const tex = gl.createTexture();
gl.bindTexture(gl.TEXTURE_2D, tex);
// ... upload texture data with gl.texImage2D / gl.texStorage2D ...

if (ext) {
  gl.texParameterf(gl.TEXTURE_2D, ext.TEXTURE_MAX_ANISOTROPY_EXT, maxAnisotropy);
}

// Recommended to also set mip-map filtering, which AF requires:
gl.texParameteri(gl.TEXTURE_2D, gl.TEXTURE_MIN_FILTER, gl.LINEAR_MIPMAP_LINEAR);
gl.texParameteri(gl.TEXTURE_2D, gl.TEXTURE_MAG_FILTER, gl.LINEAR);
gl.generateMipmap(gl.TEXTURE_2D);
```

### Fallback if missing

Use `gl.LINEAR_MIPMAP_LINEAR` (trilinear filtering). Textures will look blurry at grazing angles but won't break.

### Three.js

Three.js queries this extension through `renderer.capabilities.getMaxAnisotropy()`. Set `texture.anisotropy = renderer.capabilities.getMaxAnisotropy()` on any `THREE.Texture` instance; the renderer applies the `texParameterf` call automatically.

```js
const tex = textureLoader.load('diffuse.jpg');
tex.anisotropy = renderer.capabilities.getMaxAnisotropy();
```

---

## 2. OES_texture_float_linear

### What it does

WebGL2 natively supports 32-bit float textures (via `gl.FLOAT` type and sized internal formats like `gl.RGBA32F`). However, the core spec **does not guarantee** that these textures support linear filtering — hardware may only allow nearest-neighbour sampling of float textures. `OES_texture_float_linear` unlocks `gl.LINEAR`, `gl.LINEAR_MIPMAP_LINEAR`, and the other bilinear/trilinear filter modes for float-typed textures.

This matters for: HDR environment maps, bloom inputs, shadow maps stored as float depth, deferred G-buffers that are read with interpolation.

Support rate: **87.06%** (web3dsurvey.com, accessed 2026-05-26). Not safe to assume — always check.

### No new constants

The extension adds no new tokens. Its presence is the signal: if `getExtension` returns non-null, float linear filtering is supported.

### How to enable and use

```js
// Both must be checked in WebGL2 for full float texture capability
const floatLinear = gl.getExtension('OES_texture_float_linear');
// Note: OES_texture_float itself is not required in WebGL2 — float formats are core.
// OES_texture_float_linear is the only extension needed here.

// Create a float texture
const hdrTex = gl.createTexture();
gl.bindTexture(gl.TEXTURE_2D, hdrTex);
gl.texStorage2D(gl.TEXTURE_2D, 1, gl.RGBA32F, width, height);
gl.texSubImage2D(gl.TEXTURE_2D, 0, 0, 0, width, height, gl.RGBA, gl.FLOAT, hdrData);

if (floatLinear) {
  // Safe to use linear filtering on this float texture
  gl.texParameteri(gl.TEXTURE_2D, gl.TEXTURE_MIN_FILTER, gl.LINEAR);
  gl.texParameteri(gl.TEXTURE_2D, gl.TEXTURE_MAG_FILTER, gl.LINEAR);
} else {
  // Fallback: only nearest is guaranteed
  gl.texParameteri(gl.TEXTURE_2D, gl.TEXTURE_MIN_FILTER, gl.NEAREST);
  gl.texParameteri(gl.TEXTURE_2D, gl.TEXTURE_MAG_FILTER, gl.NEAREST);
  // Alternatively: store HDR data as half-float (gl.HALF_FLOAT) and check
  // OES_texture_half_float_linear, which has broader hardware support
}
```

### Fallback strategy

Fall back to `gl.RGBA16F` (half-float) combined with `OES_texture_half_float_linear`. Half-float linear filtering has substantially wider hardware support, particularly on older mobile GPUs.

---

## 3. EXT_color_buffer_float

### What it does

WebGL2 natively includes float texture formats, but **rendering to a float framebuffer attachment is not guaranteed** without this extension. `EXT_color_buffer_float` makes the following internal formats **color-renderable** — meaning they can be used as `gl.FRAMEBUFFER` color attachments and achieve `FRAMEBUFFER_COMPLETE` status:

| Format | Bits/channel | Notes |
|---|---|---|
| `gl.R16F` | 16 (red only) | |
| `gl.RG16F` | 16 (two channel) | |
| `gl.RGBA16F` | 16 per channel | Most common for HDR render targets |
| `gl.R32F` | 32 (red only) | |
| `gl.RG32F` | 32 (two channel) | |
| `gl.RGBA32F` | 32 per channel | Highest precision, largest bandwidth |
| `gl.R11F_G11F_B10F` | 11/11/10 packed | Good quality/size tradeoff, no alpha |

**WebGL2 only.** There is no WebGL1 version of this extension. For WebGL1, use `WEBGL_color_buffer_float` (float32 only) or `EXT_color_buffer_half_float`.

Support rate: **99.95%** (web3dsurvey.com, accessed 2026-05-26). Effectively universal in WebGL2.

### How to enable and use

```js
// Extension object has no methods or constants of its own.
// Its presence alone unlocks the formats listed above.
const ext = gl.getExtension('EXT_color_buffer_float');

if (!ext) {
  console.warn('Float render targets not available — falling back to UNSIGNED_BYTE');
}

// Create an HDR render target (requires ext)
const fbo = gl.createFramebuffer();
const colorTex = gl.createTexture();

gl.bindTexture(gl.TEXTURE_2D, colorTex);
gl.texStorage2D(gl.TEXTURE_2D, 1, gl.RGBA16F, width, height);

gl.bindFramebuffer(gl.FRAMEBUFFER, fbo);
gl.framebufferTexture2D(
  gl.FRAMEBUFFER,
  gl.COLOR_ATTACHMENT0,
  gl.TEXTURE_2D,
  colorTex,
  0
);

const status = gl.checkFramebufferStatus(gl.FRAMEBUFFER);
if (status !== gl.FRAMEBUFFER_COMPLETE) {
  // Fallback: use gl.RGBA8 for the color attachment
}

// Alternatively use a renderbuffer:
const rb = gl.createRenderbuffer();
gl.bindRenderbuffer(gl.RENDERBUFFER, rb);
gl.renderbufferStorage(gl.RENDERBUFFER, gl.RGBA16F, 256, 256);  // requires ext
```

### Float blending note

Even with `EXT_color_buffer_float`, **float blending is not enabled**. If you need `gl.blendFunc` operations on float render targets, you must additionally check `EXT_float_blend`.

### Fallback if missing

Render to `gl.RGBA8` (8-bit per channel, LDR). This breaks HDR pipelines and tone mapping fidelity but is the only universally safe format.

---

## Summary table

| Extension | Support rate (WebGL2, 2026) | Context | Key use |
|---|---|---|---|
| `EXT_texture_filter_anisotropic` | 99.05% | WebGL1 + WebGL2 | Sharp textures at oblique angles |
| `OES_texture_float_linear` | 87.06% | WebGL1 + WebGL2 | Linear filtering on float textures |
| `EXT_color_buffer_float` | 99.95% | WebGL2 only | Float/HDR framebuffer render targets |
