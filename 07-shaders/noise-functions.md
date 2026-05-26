# Noise Functions — Simplex, Worley, FBM

## Simplex 2D Noise (Stefan Gustavson / Ian McEwan, MIT)

```glsl
// ---- Required helpers ----
vec3 mod289(vec3 x) { return x - floor(x * (1.0/289.0)) * 289.0; }
vec4 mod289(vec4 x) { return x - floor(x * (1.0/289.0)) * 289.0; }
vec4 permute(vec4 x) { return mod289(((x*34.0)+10.0)*x); }
vec4 taylorInvSqrt(vec4 r) { return 1.79284291400159 - 0.85373472095314*r; }

// ---- 2D Simplex ----
float snoise(vec2 v) {
  const vec4 C = vec4(0.211324865405187, 0.366025403784439,
                     -0.577350269189626, 0.024390243902439);
  vec2 i  = floor(v + dot(v, C.yy));
  vec2 x0 = v - i + dot(i, C.xx);
  vec2 i1 = (x0.x > x0.y) ? vec2(1.0,0.0) : vec2(0.0,1.0);
  vec4 x12 = x0.xyxy + C.xxzz;
  x12.xy -= i1;
  i = mod289(i);
  vec3 p = permute(permute(i.y + vec3(0.0,i1.y,1.0)) + i.x + vec3(0.0,i1.x,1.0));
  vec3 m = max(0.5 - vec3(dot(x0,x0), dot(x12.xy,x12.xy), dot(x12.zw,x12.zw)), 0.0);
  m = m*m; m = m*m;
  vec3 x = 2.0 * fract(p * C.www) - 1.0;
  vec3 h = abs(x) - 0.5;
  vec3 ox = floor(x + 0.5);
  vec3 a0 = x - ox;
  m *= taylorInvSqrt(a0*a0 + h*h);
  vec3 g;
  g.x  = a0.x  * x0.x  + h.x  * x0.y;
  g.yz = a0.yz * x12.xz + h.yz * x12.yw;
  return 130.0 * dot(m, g);
}
```

## Simplex 3D Noise

```glsl
float snoise(vec3 v) {
  const vec2 C = vec2(1.0/6.0, 1.0/3.0);
  const vec4 D = vec4(0.0, 0.5, 1.0, 2.0);
  vec3 i  = floor(v + dot(v, C.yyy));
  vec3 x0 = v - i + dot(i, C.xxx);
  vec3 g = step(x0.yzx, x0.xyz);
  vec3 l = 1.0 - g;
  vec3 i1 = min(g.xyz, l.zxy);
  vec3 i2 = max(g.xyz, l.zxy);
  vec3 x1 = x0 - i1 + C.xxx;
  vec3 x2 = x0 - i2 + C.yyy;
  vec3 x3 = x0 - D.yyy;
  i = mod289(i);
  vec4 p = permute(permute(permute(
    i.z + vec4(0.0, i1.z, i2.z, 1.0))
    + i.y + vec4(0.0, i1.y, i2.y, 1.0))
    + i.x + vec4(0.0, i1.x, i2.x, 1.0));
  float n_ = 0.142857142857;
  vec3 ns = n_ * D.wyz - D.xzx;
  vec4 j = p - 49.0 * floor(p * ns.z * ns.z);
  vec4 x_ = floor(j * ns.z);
  vec4 y_ = floor(j - 7.0 * x_);
  vec4 x = x_ *ns.x + ns.yyyy;
  vec4 y = y_ *ns.x + ns.yyyy;
  vec4 h = 1.0 - abs(x) - abs(y);
  vec4 b0 = vec4(x.xy, y.xy);
  vec4 b1 = vec4(x.zw, y.zw);
  vec4 s0 = floor(b0)*2.0 + 1.0;
  vec4 s1 = floor(b1)*2.0 + 1.0;
  vec4 sh = -step(h, vec4(0.0));
  vec4 a0 = b0.xzyw + s0.xzyw*sh.xxyy;
  vec4 a1 = b1.xzyw + s1.xzyw*sh.zzww;
  vec3 p0 = vec3(a0.xy, h.x);
  vec3 p1 = vec3(a0.zw, h.y);
  vec3 p2 = vec3(a1.xy, h.z);
  vec3 p3 = vec3(a1.zw, h.w);
  vec4 norm = taylorInvSqrt(vec4(dot(p0,p0), dot(p1,p1), dot(p2,p2), dot(p3,p3)));
  p0 *= norm.x; p1 *= norm.y; p2 *= norm.z; p3 *= norm.w;
  vec4 m = max(0.6 - vec4(dot(x0,x0), dot(x1,x1), dot(x2,x2), dot(x3,x3)), 0.0);
  m = m * m;
  return 42.0 * dot(m*m, vec4(dot(p0,x0), dot(p1,x1), dot(p2,x2), dot(p3,x3)));
}
```

**Usage by dimension:**
- 2D: UV distortion, simple organic patterns
- 3D: Volumetric clouds, animated noise `snoise(vec3(uv, time))`
- 4D: Seamlessly looping animated noise `snoise(vec4(uv, time, 0.0))`

## Worley / Cellular Noise

```glsl
// Returns vec2(F1, F2) — F1 = distance to nearest cell center
vec2 worley(vec2 p) {
  vec2 n = floor(p);
  vec2 f = fract(p);
  float F1 = 8.0, F2 = 8.0;
  for (int j = -1; j <= 1; j++) {
    for (int i = -1; i <= 1; i++) {
      vec2 g = vec2(float(i), float(j));
      vec2 o = fract(sin(vec2(
        dot(n+g, vec2(127.1, 311.7)),
        dot(n+g, vec2(269.5, 183.3))
      )) * 43758.5453);
      vec2 r = g + o - f;
      float d = dot(r, r);
      if (d < F1) { F2 = F1; F1 = d; }
      else if (d < F2) { F2 = d; }
    }
  }
  return vec2(sqrt(F1), sqrt(F2));
}

// Usage:
// F2 - F1 = ridge pattern
// 1.0 - F1 = filled cell pattern
// (F2 - F1) * F1 = cracks
```

## FBM — Fractal Brownian Motion

Stacks octaves of noise at increasing frequency and decreasing amplitude. Each octave adds fine detail. Natural phenomena (clouds, terrain, fire) self-resemble across scales — FBM models this.

```glsl
#define OCTAVES 6

float fbm(vec2 st) {
  float value     = 0.0;
  float amplitude = 0.5;  // halves each octave
  float frequency = 1.0;  // doubles each octave (lacunarity = 2)

  for (int i = 0; i < OCTAVES; i++) {
    value     += amplitude * snoise(st * frequency);
    frequency *= 2.0;
    amplitude *= 0.5;
  }
  return value;
}

// 3D animated clouds:
float fbm(vec3 p) {
  float v = 0.0, a = 0.5;
  for (int i = 0; i < OCTAVES; i++) {
    v += a * snoise(p);
    p *= 2.0;
    a *= 0.5;
  }
  return v;
}
```

**Key parameters:**
- `lacunarity` — frequency multiplier per octave (typically 2.0)
- `gain` / persistence — amplitude multiplier per octave (typically 0.5)
- More octaves = more detail, more ALU cost; 4–8 is typical

## Cosine Palette (IQ Technique)

```glsl
// color(t) = a + b * cos(2*pi*(c*t + d))
vec3 palette(float t, vec3 a, vec3 b, vec3 c, vec3 d) {
  return a + b * cos(6.283185 * (c * t + d));
}

// Examples:
// Rainbow:   palette(t, vec3(0.5), vec3(0.5), vec3(1.0), vec3(0.0, 0.33, 0.67))
// Warm:      palette(t, vec3(0.5), vec3(0.5), vec3(1.0), vec3(0.3, 0.2, 0.2))
// Cool teal: palette(t, vec3(0.5), vec3(0.5), vec3(1.0,1.0,0.5), vec3(0.8,0.9,0.3))
```

## glslify npm Packages

```bash
npm install glslify glsl-noise glsl-fog glsl-easings
```

```glsl
// In .glsl files:
#pragma glslify: snoise3 = require(glsl-noise/simplex/3d)
#pragma glslify: cnoise2 = require(glsl-noise/classic/2d)
```

Available paths:
- `glsl-noise/simplex/2d`, `/simplex/3d`, `/simplex/4d`
- `glsl-noise/classic/2d`, `/classic/3d`, `/classic/4d`
- `glsl-noise/periodic/2d`, `/periodic/3d`, `/periodic/4d`

## Sources
- https://github.com/stegu/webgl-noise (canonical implementation, MIT)
- https://github.com/patriciogonzalezvivo/lygia (lygia library)
- https://github.com/hughsk/glsl-noise (npm glslify-compatible)
