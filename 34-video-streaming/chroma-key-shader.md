# Chroma Key (Green Screen) Shader for Three.js VideoTexture

**Last verified:** 2026-05-26  
**Reference implementations:**  
- https://github.com/drinkspiller/threejs_chromakey_video_material (Mugen87 / three.js forum)  
- https://jameshfisher.com/2020/08/11/production-ready-green-screen-in-the-browser/  
- https://discourse.threejs.org/t/chromakey-shader/64910

---

## What Chroma Keying Does in a Shader

A chroma key shader computes the perceptual distance between each video pixel's color and a target key color (typically `vec3(0.0, 1.0, 0.0)` for green). Pixels within the threshold radius are made transparent; pixels outside it are kept opaque. A `smoothstep` band between inner and outer radii softens the edge so you don't get a hard jagged silhouette.

The math runs entirely on the GPU — one fragment shader invocation per pixel per frame, no CPU round-trip. For a 1280×720 video at 30 fps, that is ~27.6 million fragment invocations per second, which modern GPUs handle comfortably in a fraction of a millisecond.

---

## Core GLSL Pattern

```glsl
// Fragment shader
uniform sampler2D videoTexture;
uniform vec3      keyColor;        // the color to remove, e.g. vec3(0.0,1.0,0.0)
uniform float     threshold;       // hard cut radius
uniform float     smoothing;       // soft edge width beyond threshold

varying vec2 vUv;

void main() {
  vec4 texColor = texture2D(videoTexture, vUv);

  // Euclidean distance in RGB space to the key color
  float dist = distance(texColor.rgb, keyColor);

  // smoothstep(edge0, edge1, x):
  //   x < edge0  → 0.0 (fully transparent, inside key color region)
  //   x > edge1  → 1.0 (fully opaque)
  //   between    → smooth 0→1 (the soft edge)
  float alpha = smoothstep(threshold, threshold + smoothing, dist);

  gl_FragColor = vec4(texColor.rgb, alpha);
}
```

**Minimum working values:**
- `threshold`: `0.3` – `0.4` for studio green screen with clean lighting
- `smoothing`: `0.05` – `0.15` for natural edge blur

Both values should be exposed as uniforms so they can be tuned at runtime without recompiling the shader.

---

## Full Three.js ShaderMaterial Setup

```js
const chromaKeyMaterial = new THREE.ShaderMaterial({
  uniforms: {
    videoTexture: { value: videoTexture },
    keyColor:     { value: new THREE.Color(0.0, 1.0, 0.0) },  // green
    threshold:    { value: 0.35 },
    smoothing:    { value: 0.1 },
  },
  vertexShader: `
    varying vec2 vUv;
    void main() {
      vUv = uv;
      gl_Position = projectionMatrix * modelViewMatrix * vec4(position, 1.0);
    }
  `,
  fragmentShader: `
    uniform sampler2D videoTexture;
    uniform vec3      keyColor;
    uniform float     threshold;
    uniform float     smoothing;
    varying vec2      vUv;

    void main() {
      vec4 texColor = texture2D(videoTexture, vUv);
      float dist    = distance(texColor.rgb, keyColor);
      float alpha   = smoothstep(threshold, threshold + smoothing, dist);
      gl_FragColor  = vec4(texColor.rgb, alpha);
    }
  `,
  transparent: true,   // required for alpha to composite correctly
  depthWrite: false,   // prevent z-fighting on transparent surfaces
  side: THREE.DoubleSide,
});
```

`transparent: true` tells Three.js to route this material through the transparent render pass, where it blends with whatever is behind it in the scene.

---

## YCbCr (YUV) Distance — Better Color Accuracy

Pure RGB distance treats hue and brightness equally, which produces poor keys when the subject has green tints in shadows. Professional implementations (including OBS Studio's chroma key filter) convert to YCbCr space and weight the chrominance channels.

```glsl
vec3 rgbToYCbCr(vec3 col) {
  float y  =  0.299 * col.r + 0.587 * col.g + 0.114 * col.b;
  float cb = -0.169 * col.r - 0.331 * col.g + 0.500 * col.b + 0.5;
  float cr =  0.500 * col.r - 0.419 * col.g - 0.081 * col.b + 0.5;
  return vec3(y, cb, cr);
}

void main() {
  vec4 texColor   = texture2D(videoTexture, vUv);
  vec3 yuv        = rgbToYCbCr(texColor.rgb);
  vec3 keyYuv     = rgbToYCbCr(keyColor);

  // Only compare chrominance (Cb, Cr) — ignore brightness (Y)
  float dist  = distance(yuv.yz, keyYuv.yz);
  float alpha = smoothstep(threshold, threshold + smoothing, dist);

  gl_FragColor = vec4(texColor.rgb, alpha);
}
```

This eliminates false keys in dark shadows that have the same luminance as the background but different chroma.

---

## Spill Suppression

"Spill" is green light bouncing off the key background and tinting the subject's edges. After keying, these semi-transparent green pixels look bad. Spill suppression desaturates the green channel in the edge zone:

```glsl
// After computing alpha, suppress green spill in soft-edge pixels
// alpha near 0 = inside key, alpha near 1 = outside key
// spill is most visible where 0 < alpha < 1
float spillStrength = 1.0 - alpha;  // how much spill to remove
vec3 noSpill = texColor.rgb;
noSpill.g = min(noSpill.g, max(noSpill.r, noSpill.b));  // cap green channel

vec3 finalRgb = mix(noSpill, texColor.rgb, alpha);
gl_FragColor = vec4(finalRgb, alpha);
```

---

## Updating Uniforms at Runtime

```js
// In the render loop or a GUI callback:
chromaKeyMaterial.uniforms.threshold.value  = 0.4;
chromaKeyMaterial.uniforms.smoothing.value  = 0.12;
chromaKeyMaterial.uniforms.keyColor.value.setRGB(0.0, 1.0, 0.0);
```

No recompilation needed. Uniforms are GPU registers updated per draw call.

---

## Performance vs. CSS mix-blend-mode

| Approach | GPU cost | CPU cost | Precision | Alpha | Use case |
|---|---|---|---|---|---|
| GLSL shader (above) | Low — one extra texture sample per fragment | Negligible | Full float per pixel | Exact, tuneable | Three.js scenes, WebGL, custom blending |
| CSS `mix-blend-mode: screen` | Very low — compositor layer | Negligible | Fixed screen blend | Not a real key | Simple HTML overlays with pre-lit footage |
| CSS `mix-blend-mode: multiply` | Very low | Negligible | Fixed multiply blend | Not a real key | Dark background removal only |
| Canvas 2D `getImageData` + CPU key | Extremely high | Very high — ~100 ms/frame at 1080p | Per-pixel control | Exact | Almost never appropriate for real-time |

**When CSS blending is enough:** If your video was shot on a truly black or white background (limbo footage), `mix-blend-mode: screen` or `multiply` removes it without any shader. The limitation is that you cannot tune threshold or smoothing, and it only works for pure black/white keys, not green.

**When the GLSL shader is necessary:**
- The video is in a Three.js scene on a mesh (not an HTML layer)
- You need the keyed video to cast light or have a material applied
- You need per-pixel transparency composited with 3D geometry
- You need spill suppression or YUV-space accuracy

---

## Integration Checklist

- [ ] `transparent: true` on the `ShaderMaterial`
- [ ] `depthWrite: false` to avoid z-sorting artifacts
- [ ] Video element has `crossOrigin = "anonymous"` if served from a different origin
- [ ] Both `threshold` and `smoothing` exposed as uniforms for real-time tuning
- [ ] Test with `THREE.SRGBColorSpace` on the VideoTexture — gamma mismatch shifts the key color in sRGB space

---

## Sources

- https://github.com/drinkspiller/threejs_chromakey_video_material — accessed 2026-05-26
- https://discourse.threejs.org/t/chromakey-shader/64910 — three.js forum, May 2024
- https://jameshfisher.com/2020/08/11/production-ready-green-screen-in-the-browser/ — OBS-derived algorithm
- https://byte-explorer.medium.com/webgl-chromakey-real-time-green-screen-matting-5c1a58c6df0a — YUV space chroma key
