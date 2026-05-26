# Compressed Texture Extensions for WebGL2

Compressed textures are decoded directly by GPU hardware, reducing VRAM bandwidth and allowing larger textures at the same memory budget. The catch: no single format works everywhere. A shipping application needs a fallback chain keyed on which extensions are available at runtime.

Sources verified: MDN Web Docs, web3dsurvey.com/webgl2. Date: 2026-05-26.

---

## The four main format families

### WEBGL_compressed_texture_s3tc (S3TC / DXT / BCn 1–5)

**GPU families:** Desktop GPUs on Windows/Linux/macOS — NVIDIA, AMD, Intel. Also available via Chrome's ANGLE on macOS over Metal. Not reliably available on Android or iOS.

**Support rate (WebGL2):** 81.45% (web3dsurvey.com, accessed 2026-05-26). Dominant on desktop; absent on most mobile.

**Formats provided:**

| Constant | Format | Compression | Notes |
|---|---|---|---|
| `ext.COMPRESSED_RGB_S3TC_DXT1_EXT` | DXT1 / BC1 | 8:1 | RGB, no alpha |
| `ext.COMPRESSED_RGBA_S3TC_DXT1_EXT` | DXT1 with alpha | 8:1 | RGB + 1-bit punch-through alpha |
| `ext.COMPRESSED_RGBA_S3TC_DXT3_EXT` | DXT3 / BC2 | 4:1 | RGBA, explicit 4-bit alpha |
| `ext.COMPRESSED_RGBA_S3TC_DXT5_EXT` | DXT5 / BC3 | 4:1 | RGBA, interpolated alpha — preferred for smooth alpha |

```js
const s3tc = gl.getExtension('WEBGL_compressed_texture_s3tc') ||
             gl.getExtension('MOZ_WEBGL_compressed_texture_s3tc') ||
             gl.getExtension('WEBKIT_WEBGL_compressed_texture_s3tc');

if (s3tc) {
  const tex = gl.createTexture();
  gl.bindTexture(gl.TEXTURE_2D, tex);
  gl.compressedTexImage2D(
    gl.TEXTURE_2D,
    0,                                     // mip level
    s3tc.COMPRESSED_RGBA_S3TC_DXT5_EXT,   // internal format
    512, 512,                              // width, height
    0,                                     // border (must be 0)
    dxtData                                // ArrayBufferView
  );
  gl.texParameteri(gl.TEXTURE_2D, gl.TEXTURE_MIN_FILTER, gl.LINEAR);
  gl.texParameteri(gl.TEXTURE_2D, gl.TEXTURE_MAG_FILTER, gl.LINEAR);
}
```

---

### WEBGL_compressed_texture_astc (ASTC)

**GPU families:** ARM Mali (most Android mid-range and up from ~2014), Qualcomm Adreno 400-series and up, Apple A8 and later (iOS 9+ in Safari 15+), NVIDIA Tegra. Broadly absent on desktop Intel/AMD/NVIDIA GPUs unless running on ANGLE with software decode (which defeats the purpose).

**Support rate (WebGL2):** 51.28% (web3dsurvey.com, accessed 2026-05-26). Dominant on modern mobile; unreliable on desktop.

**Profiles:** The extension exposes a `getSupportedProfiles()` method returning an array. `"ldr"` (low dynamic range) is the common case. `"hdr"` (high dynamic range) is rarer.

**Formats:** 28 format constants in two series (RGBA and sRGB8_ALPHA8), each with block sizes 4×4, 5×4, 5×5, 6×5, 6×6, 8×5, 8×6, 8×8, 10×5, 10×6, 10×8, 10×10, 12×10, 12×12. Smaller block = higher quality = larger file.

| Format (example) | Block | Bits/pixel | Use |
|---|---|---|---|
| `COMPRESSED_RGBA_ASTC_4x4_KHR` | 4×4 | 8.00 | Highest quality |
| `COMPRESSED_RGBA_ASTC_6x6_KHR` | 6×6 | 3.56 | Balanced |
| `COMPRESSED_RGBA_ASTC_8x8_KHR` | 8×8 | 2.00 | Aggressive compression |
| `COMPRESSED_RGBA_ASTC_12x12_KHR` | 12×12 | 0.89 | Smallest, lowest quality |
| `COMPRESSED_SRGB8_ALPHA8_ASTC_4x4_KHR` | 4×4 | 8.00 | sRGB variant |

```js
const astc = gl.getExtension('WEBGL_compressed_texture_astc');

if (astc) {
  // Optional: check HDR profile support
  const profiles = astc.getSupportedProfiles();
  const hasHDR = profiles.includes('hdr');

  const tex = gl.createTexture();
  gl.bindTexture(gl.TEXTURE_2D, tex);
  gl.compressedTexImage2D(
    gl.TEXTURE_2D, 0,
    astc.COMPRESSED_RGBA_ASTC_6x6_KHR,
    512, 512, 0,
    astcData
  );
}
```

---

### WEBGL_compressed_texture_etc (ETC2 / EAC)

**GPU families:** All OpenGL ES 3.0-capable hardware — meaning essentially every Android device with a GPU released after 2012 (Adreno 3xx, Mali-T6xx, PowerVR Series6 and later). ETC2 is mandated by the OpenGL ES 3.0 spec. iOS Safari also supports it. Desktop support via ANGLE is available in Chrome.

**Support rate (WebGL2):** 52.09% (web3dsurvey.com, accessed 2026-05-26). Note: Firefox historically exposed this extension as `WEBGL_compressed_texture_es3`; it now uses the standard name.

**Formats provided (10 constants):**

| Constant | Notes |
|---|---|
| `ext.COMPRESSED_RGB8_ETC2` | RGB, no alpha |
| `ext.COMPRESSED_RGBA8_ETC2_EAC` | RGBA with full 8-bit alpha |
| `ext.COMPRESSED_RGB8_PUNCHTHROUGH_ALPHA1_ETC2` | RGB with 1-bit punch-through alpha |
| `ext.COMPRESSED_SRGB8_ETC2` | sRGB, no alpha |
| `ext.COMPRESSED_SRGB8_ALPHA8_ETC2_EAC` | sRGBA |
| `ext.COMPRESSED_SRGB8_PUNCHTHROUGH_ALPHA1_ETC2` | sRGB + 1-bit alpha |
| `ext.COMPRESSED_R11_EAC` | Single red channel |
| `ext.COMPRESSED_RG11_EAC` | Red + green channels |
| `ext.COMPRESSED_SIGNED_R11_EAC` | Signed red |
| `ext.COMPRESSED_SIGNED_RG11_EAC` | Signed red + green |

```js
const etc = gl.getExtension('WEBGL_compressed_texture_etc');

if (etc) {
  gl.compressedTexImage2D(
    gl.TEXTURE_2D, 0,
    etc.COMPRESSED_RGBA8_ETC2_EAC,
    512, 512, 0,
    etcData
  );
}
```

---

### EXT_texture_compression_bptc (BPTC / BC6H + BC7)

**GPU families:** Desktop NVIDIA (Fermi+), AMD (GCN+), and Intel (Iris/HD 4000+). Not available on mobile. MDN notes "no support on Windows" in some configurations — this is a driver/OS dependency, not a hardware limitation per se; actual support varies.

**Support rate (WebGL2):** 79.17% (web3dsurvey.com, accessed 2026-05-26). Reasonable desktop coverage; avoid for mobile targets.

**Why use it:** BC7 produces dramatically better quality than DXT5 at the same 4:1 compression ratio, particularly for textures with subtle gradients and sharp colour transitions. BC6H is the only GPU-native format for HDR (float) colour data.

**Formats provided:**

| Constant | DirectX name | Description |
|---|---|---|
| `ext.COMPRESSED_RGBA_BPTC_UNORM_EXT` | BC7 | High-quality 8-bit/channel RGBA (128 bits per 4×4 block) |
| `ext.COMPRESSED_SRGB_ALPHA_BPTC_UNORM_EXT` | BC7 sRGB | sRGB variant of BC7 |
| `ext.COMPRESSED_RGB_BPTC_SIGNED_FLOAT_EXT` | BC6H signed | HDR signed float RGB |
| `ext.COMPRESSED_RGB_BPTC_UNSIGNED_FLOAT_EXT` | BC6H unsigned | HDR unsigned float RGB |

```js
const bptc = gl.getExtension('EXT_texture_compression_bptc');

if (bptc) {
  // BC7 for high-quality diffuse maps
  gl.compressedTexImage2D(
    gl.TEXTURE_2D, 0,
    bptc.COMPRESSED_RGBA_BPTC_UNORM_EXT,
    512, 512, 0,
    bc7Data
  );
}
```

---

## GPU support matrix

| Extension | NVIDIA desktop | AMD desktop | Intel desktop | ARM Mali (Android) | Qualcomm Adreno | Apple (iOS/macOS) |
|---|---|---|---|---|---|---|
| S3TC (DXT) | Yes | Yes | Yes | No | No | Via ANGLE (Chrome macOS) |
| ASTC | No (most) | No (most) | No (most) | Yes (Mali-T6xx+) | Yes (Adreno 4xx+) | Yes (A8+, Safari 15+) |
| ETC2 | Via ANGLE | Via ANGLE | Via ANGLE | Yes (ES3.0+) | Yes (ES3.0+) | Yes |
| BPTC (BC6H/BC7) | Yes (Fermi+) | Yes (GCN+) | Yes (HD 4000+) | No | No | Partial |

---

## Recommended fallback chain

Query extensions in this order and use the first available format for your texture assets:

```js
function selectCompressedFormat(gl) {
  const bptc  = gl.getExtension('EXT_texture_compression_bptc');
  const s3tc  = gl.getExtension('WEBGL_compressed_texture_s3tc');
  const astc  = gl.getExtension('WEBGL_compressed_texture_astc');
  const etc   = gl.getExtension('WEBGL_compressed_texture_etc');

  if (bptc)  return { ext: bptc,  format: bptc.COMPRESSED_RGBA_BPTC_UNORM_EXT,    name: 'bc7'   };
  if (s3tc)  return { ext: s3tc,  format: s3tc.COMPRESSED_RGBA_S3TC_DXT5_EXT,     name: 'dxt5'  };
  if (etc)   return { ext: etc,   format: etc.COMPRESSED_RGBA8_ETC2_EAC,           name: 'etc2'  };
  if (astc)  return { ext: astc,  format: astc.COMPRESSED_RGBA_ASTC_6x6_KHR,      name: 'astc'  };

  return null; // fall through to uncompressed RGBA8
}

const selected = selectCompressedFormat(gl);
if (selected) {
  loadCompressedTexture(selected.name);    // fetch the correct .ktx2 / .dds / .astc file
} else {
  loadUncompressedTexture();               // fetch .png / .jpg and upload as gl.RGBA8
}
```

**Practical note:** In production, serve KTX2 container files (which can hold multiple format variants) and use a KTX2 loader (see Three.js KTX2Loader below) to select the right format at decode time rather than fetching separate assets per format.

---

## Three.js compressed texture handling

Three.js wraps compressed texture loading through dedicated loaders rather than raw extension queries.

### KTX2Loader (recommended)

`THREE.KTX2Loader` decodes KTX2 container files (based on Basis Universal transcoding), selecting the best available GPU format at runtime. It internally checks `renderer.capabilities` which wraps the same extension queries described above.

```js
import { KTX2Loader } from 'three/addons/loaders/KTX2Loader.js';

const ktx2Loader = new KTX2Loader()
  .setTranscoderPath('/libs/basis/')       // path to basis_transcoder.js + .wasm
  .detectSupport(renderer);               // queries gl extensions internally

ktx2Loader.load('/textures/diffuse.ktx2', (texture) => {
  mesh.material.map = texture;
  mesh.material.needsUpdate = true;
});
```

### CompressedTextureLoader (direct KTX/DDS)

For pre-compressed DDS or raw KTX files where you manage format selection yourself:

```js
import { CompressedTextureLoader } from 'three/examples/jsm/loaders/CompressedTextureLoader.js';
```

### renderer.capabilities.isWebGL2

Three.js exposes `renderer.capabilities.isWebGL2` (boolean) which you can use to gate WebGL2-only extension queries before calling `gl.getExtension`.
