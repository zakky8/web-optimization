# Debugging GLSL Shaders

GLSL has no `console.log`. The only output channel a shader has is the fragment color written to the framebuffer. All shader debugging is therefore a form of data visualization: map the value you want to inspect onto a color channel and render it.

---

## The Core Technique: Value → Color

```glsl
// Fragment shader: visualize any float in [0, 1] as a grayscale value
void main() {
  float myValue = someComputation();
  gl_FragColor = vec4(myValue, myValue, myValue, 1.0);
}
```

For values outside [0, 1], clamp or remap first:

```glsl
float remapped = (myValue - minExpected) / (maxExpected - minExpected);
gl_FragColor = vec4(remapped, remapped, remapped, 1.0);
```

---

## Visualizing Floats as Color

### Single float → grayscale
```glsl
float val = roughness;  // expected range 0..1
gl_FragColor = vec4(vec3(val), 1.0);
```

### Separate channels to see three values at once
```glsl
gl_FragColor = vec4(r, g, b, 1.0);  // R=roughness, G=metalness, B=AO
```

### Negative values detection
```glsl
float val = dot(N, L);  // can be negative
// Red = positive, blue = negative
gl_FragColor = vec4(
  max(0.0, val),
  0.0,
  max(0.0, -val),
  1.0
);
```

### Heat map for large ranges
```glsl
// Maps 0..1 to blue→cyan→green→yellow→red
vec3 heatmap(float t) {
  t = clamp(t, 0.0, 1.0);
  return clamp(
    vec3(
      1.5 - abs(t * 4.0 - 3.0),
      1.5 - abs(t * 4.0 - 2.0),
      1.5 - abs(t * 4.0 - 1.0)
    ),
    0.0, 1.0
  );
}
```

---

## step / smoothstep Visualization

`step` and `smoothstep` are common sources of invisible bugs when the threshold is wrong.

### Check where step fires
```glsl
float edge = 0.5;
float val  = someValue;

// White where val > edge, black below
gl_FragColor = vec4(vec3(step(edge, val)), 1.0);
```

### Visualize the smoothstep transition band
```glsl
float lo = 0.3;
float hi = 0.7;
float s  = smoothstep(lo, hi, someValue);

// Visualize: 0→1 gradient across the transition band
gl_FragColor = vec4(vec3(s), 1.0);

// To show only the transition band:
float inBand = step(lo, someValue) * (1.0 - step(hi, someValue));
gl_FragColor = vec4(inBand, s, 0.0, 1.0);  // green in band, red below, off above
```

### Common `step` trap
```glsl
// Bug: arguments reversed — step(edge, x) not step(x, edge)
float wrong = step(someValue, 0.5);   // fires when 0.5 >= someValue
float right = step(0.5, someValue);   // fires when someValue >= 0.5
```
Visualize both and compare to diagnose argument-order bugs.

---

## Normal Visualization

Normals live in [-1, 1] per component. Map to [0, 1] for display:

```glsl
// World-space or view-space normals
vec3 N = normalize(vNormal);
gl_FragColor = vec4(N * 0.5 + 0.5, 1.0);
// Result: neutral grey = (0.5, 0.5, 0.5)
// Pointing right (+X) = red
// Pointing up (+Y)    = green
// Pointing toward cam (+Z) = blue
```

### What correct normals look like
- Flat-shaded surface facing the camera: uniform medium blue
- Sphere: smooth gradient across all hues; poles are distinct colors
- Back faces: the +Z component flips to negative → dark blue becomes very dark

### Spotting broken normals
- **All one color (e.g., solid red):** normals are not being transformed — check that the normal matrix is applied (`transpose(inverse(modelMatrix))` not `modelMatrix`)
- **Sudden black patches:** normals are NaN or zero-length — a `normalize(vec3(0.0))` call upstream
- **Inverted appearance:** normal map Y channel needs flip; add `N.y = 1.0 - N.y` after sampling

---

## UV Checker Pattern

The UV checker is the fastest way to verify UV unwrapping, tiling, and texture coordinate transforms.

```glsl
varying vec2 vUv;

void main() {
  // Checker with N tiles per unit
  float N = 8.0;
  vec2 uv = vUv * N;
  float checker = mod(floor(uv.x) + floor(uv.y), 2.0);
  gl_FragColor = vec4(vec3(checker), 1.0);
}
```

### What to look for
- **Stretching:** tiles are rectangles, not squares → non-uniform UV scaling
- **Seam tears:** discontinuous tiles at mesh seams → UV island gaps; check seam handling in the modeler
- **Mirroring:** tiles reflect instead of continuing → UV uses mirroring wrap mode or mirrored UV island
- **Incorrect tiling count:** adjust `N` to match expected texture repetitions

### Augmented UV checker (shows U and V separately)
```glsl
vec2 uv = vUv * 8.0;
float checker = mod(floor(uv.x) + floor(uv.y), 2.0);
// U direction = red tint, V direction = green tint
gl_FragColor = vec4(fract(uv.x), fract(uv.y), checker * 0.5, 1.0);
```

---

## Common GLSL Errors and Their Visual Signatures

### 1. Precision mismatch / mediump overflow
**Symptom:** Banding artifacts on smooth gradients, especially at distance; values clamp at ±65504 on mediump float.  
**Diagnosis:**
```glsl
// mediump has ~3 decimal digits of precision
// If using mediump and seeing banding:
precision highp float;  // try switching to highp
```
Visualize by outputting the raw value range with the heatmap function — if it shows sharp steps instead of a gradient, precision is lost.

### 2. Missing `normalize` on interpolated normals
**Symptom:** Lighting looks slightly wrong; hard to see on small triangles, obvious on large flat quads; Mach banding appears at grazing angles.  
**Diagnosis:** Visualize `length(vNormal)` — should be 1.0 everywhere. Values diverging from 1.0 = not normalized after rasterizer interpolation.
```glsl
float len = length(vNormal);
gl_FragColor = vec4(vec3(abs(len - 1.0) * 10.0), 1.0);  // bright = bad
```

### 3. NaN / Infinity propagation
**Symptom:** Solid black or solid magenta patches; flickering geometry; depends on driver — some GPUs replace NaN with 0, others with max float.  
**Diagnosis:** GLSL has no `isnan()` in WebGL 1. Workaround:
```glsl
float v = suspectedValue;
float isNaN = float(v != v);          // NaN != NaN is true by IEEE 754
gl_FragColor = vec4(isNaN, 0.0, 0.0, 1.0);  // Red = NaN present
```
WebGL 2 / GLSL ES 3.00 has `isnan()` and `isinf()` built-in.

### 4. Divide by zero in shader
**Symptom:** Same as NaN propagation.  
**Diagnosis:**
```glsl
float denom = computeDenominator();
float safe = max(denom, 0.0001);  // floor to avoid zero
float result = numerator / safe;
// To debug: visualize (denom < 0.001) as red flag
gl_FragColor = vec4(step(denom, 0.001), result, 0.0, 1.0);
```

### 5. Wrong coordinate space (world vs view vs clip)
**Symptom:** Lighting moves with the camera instead of staying fixed to geometry; normal maps appear to slide.  
**Diagnosis:** Visualize position vectors as color. World-space positions do not change as you orbit; view-space positions rotate with the camera.
```glsl
// World space — should be static relative to geometry
gl_FragColor = vec4(vWorldPos * 0.1 + 0.5, 1.0);

// View space — changes as camera moves
vec4 viewPos = viewMatrix * vec4(vWorldPos, 1.0);
gl_FragColor = vec4(viewPos.xyz * 0.1 + 0.5, 1.0);
```

### 6. Texture sampling returning black
**Symptoms:** Solid black where texture should appear.  
**Common causes and visual tests:**

```glsl
// Test 1: Is the UV in expected range?
gl_FragColor = vec4(vUv, 0.0, 1.0);

// Test 2: Is the sampler actually receiving the texture?
vec4 sample = texture2D(uTexture, vec2(0.5, 0.5));  // center pixel
gl_FragColor = sample;

// Test 3: Mipmapping on NPOT texture
// In WebGL 1, non-power-of-two textures don't generate mipmaps.
// Force nearest/linear filtering and CLAMP_TO_EDGE wrap.
```

### 7. Clipping / backface culling
**Symptom:** Geometry disappears when viewed from certain angles.  
**Diagnosis:**
```glsl
// Visualize gl_FrontFacing
gl_FragColor = vec4(
  float(gl_FrontFacing),    // R=1 front, R=0 back
  float(!gl_FrontFacing),   // G=1 back
  0.0,
  1.0
);
// Front faces = green, back faces = red
// If all red, winding order is inverted or culling mode is wrong
```

---

## Workflow Recommendation

1. **Isolate.** Comment out everything in the fragment shader except `gl_FragColor = vec4(1.0, 0.0, 1.0, 1.0)` (magenta). If you see magenta, the shader compiles and the geometry draws. Work back in.
2. **Visualize inputs before computing outputs.** Confirm UV, normals, and world position look correct before debugging lighting math.
3. **Binary search.** Split the shader in half. If the first half looks correct, the bug is in the second. Repeat.
4. **Use `#ifdef DEBUG` guards** so visualization code ships in dev builds and gets stripped in production:
```glsl
#ifdef DEBUG_NORMALS
  gl_FragColor = vec4(N * 0.5 + 0.5, 1.0);
  return;
#endif
```
Inject `#define DEBUG_NORMALS` via Three.js shader defines:
```js
material.defines = { DEBUG_NORMALS: "" };
material.needsUpdate = true;
```

---

*Note: All techniques above apply to GLSL ES 1.00 (WebGL 1) and GLSL ES 3.00 (WebGL 2). WebGL 2 / GLSL ES 3.00 adds `isnan()`, `isinf()`, and integer types that simplify some of the workarounds above.*
