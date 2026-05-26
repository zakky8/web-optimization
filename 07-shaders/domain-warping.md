# Domain Warping — Inigo Quilez Technique

Source: https://iquilezles.org/articles/warp/

Domain warping distorts input coordinates using FBM before evaluating FBM again.
Creates lava, marble, fire, biological textures.

## Two-Layer Warp (Full IQ Technique)

```glsl
// Requires fbm(vec2) from noise-functions.md

float pattern(vec2 p) {
  // Layer 1: compute displacement q
  vec2 q = vec2(
    fbm(p + vec2(0.0, 0.0)),
    fbm(p + vec2(5.2, 1.3))
  );

  // Layer 2: displace q again with r
  vec2 r = vec2(
    fbm(p + 4.0*q + vec2(1.7, 9.2)),
    fbm(p + 4.0*q + vec2(8.3, 2.8))
  );

  // Final evaluation in warped domain
  return fbm(p + 4.0*r);
}
```

**Effect:** Layering 3 FBM evaluations with cross-offset inputs creates extremely complex,
organic-looking patterns from a simple mathematical structure.

## Single-Layer Warp (Simpler)

```glsl
float warpedNoise(vec2 uv, float strength) {
  vec2 warp = vec2(
    fbm(uv + vec2(0.0, 0.0)),
    fbm(uv + vec2(3.7, 1.9))
  );
  return fbm(uv + strength * warp);
}

// Usage: warpedNoise(vUv * 2.0, 1.5)
```

## 3D Domain Warping (for Volumetric Effects)

```glsl
float warp3D(vec3 p, float time) {
  vec3 q = vec3(
    fbm(p + vec3(0.0, 0.0, time * 0.1)),
    fbm(p + vec3(5.2, 1.3, time * 0.15)),
    fbm(p + vec3(9.7, 4.1, time * 0.12))
  );
  return fbm(p + 4.0 * q);
}
```

## Marble Texture

```glsl
float marble(vec2 uv, float time) {
  float n = fbm(uv * 3.0 + time * 0.05);
  // Warp horizontal stripes with noise
  float stripe = sin(uv.x * 10.0 + n * 15.0);
  return smoothstep(-1.0, 1.0, stripe);
}

void main() {
  float m = marble(vUv, uTime);
  vec3 color = mix(vec3(0.15, 0.15, 0.2), vec3(0.9, 0.85, 0.8), m);
  gl_FragColor = vec4(color, 1.0);
}
```

## Fire / Lava Effect

```glsl
float fire(vec2 uv, float time) {
  // Upward motion + warp
  vec2 p = uv + vec2(0.0, -time * 0.3);
  float n = pattern(p * 2.0);
  // Fade toward top
  float fade = smoothstep(0.0, 0.5, uv.y);
  return n * (1.0 - fade);
}

void main() {
  float f = fire(vUv, uTime);
  vec3 fireColor = mix(
    vec3(0.0),                    // black
    mix(vec3(1.0, 0.2, 0.0),     // red
        vec3(1.0, 0.9, 0.0),     // yellow
        f),
    f
  );
  gl_FragColor = vec4(fireColor, f);
}
```

## Performance Notes
- Each FBM = N octave evaluations of simplex noise
- Two-layer warp = 3 FBM calls = ~18 simplex evaluations per pixel
- On mobile: reduce OCTAVES to 3-4, single-layer warp only
- Can bake warp into a texture for static scenes
