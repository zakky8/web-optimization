# LYGIA Shader Library — Complete Module Inventory

GitHub: https://github.com/patriciogonzalezvivo/lygia

LYGIA supports GLSL, HLSL, WGSL, and Metal. Import via URL or copy individual files.

```glsl
// URL import in compatible environments (ShaderToy, GLSL sandbox)
#include "https://lygia.xyz/generative/fbm.glsl"

// Local copy (production recommended)
// Copy from: raw.githubusercontent.com/patriciogonzalezvivo/lygia/main/...
```

## Module Categories

### generative/ — Noise and Procedural
| Module | Description |
|---|---|
| `random.glsl` | Pseudo-random float/vec2/vec3 |
| `srandom.glsl` | Signed random (-1 to 1) |
| `snoise.glsl` | Simplex noise 1D/2D/3D/4D |
| `cnoise.glsl` | Classic (Perlin) noise |
| `pnoise.glsl` | Periodic Perlin noise |
| `gnoise.glsl` | Gradient noise |
| `fbm.glsl` | Fractal Brownian Motion |
| `worley.glsl` | Cellular/Voronoi noise (configurable distance) |
| `voronoi.glsl` | Voronoi with smooth edges |
| `voronoise.glsl` | Voronoi + noise blend |
| `psrdnoise.glsl` | Tiling simplex noise with derivatives |
| `noised.glsl` | Noise with analytic derivatives |
| `curl.glsl` | Curl noise (divergence-free) |
| `gerstnerWave.glsl` | Ocean wave simulation |
| `wavelet.glsl` | Wavelet noise |

### lighting/ — PBR and Illumination
| Module | Description |
|---|---|
| `pbr.glsl` | Full PBR pipeline |
| `pbrClearCoat.glsl` | PBR with clearcoat layer |
| `pbrGlass.glsl` | PBR glass/transmission |
| `fresnel.glsl` | Fresnel/Schlick |
| `fresnelReflection.glsl` | Reflection with Fresnel |
| `ssao.glsl` | Screen-space ambient occlusion |
| `ssr.glsl` | Screen-space reflections |
| `atmosphere.glsl` | Atmospheric scattering |
| `blackbody.glsl` | Blackbody radiation color |
| `iridescence.glsl` | Thin-film interference |
| `gooch.glsl` | Gooch toon shading |
| `sphericalHarmonics.glsl` | SH irradiance |

### color/ — Color Operations
| Module | Description |
|---|---|
| `luma.glsl` | Luminance |
| `saturation.glsl` | HSL saturation adjust |
| `blend.glsl` | Photoshop blend modes (multiply, screen, overlay, etc.) |
| `palette.glsl` | IQ cosine palettes |
| `tonemap.glsl` | ACES, Reinhard, Uncharted2 |
| Colorspace: `rgb2hsl`, `hsl2rgb`, `rgb2hsv`, `rgb2lab`, `rgb2xyz` |

### filter/ — Image Filters
| Module | Description |
|---|---|
| `gaussianBlur.glsl` | Gaussian blur (separable) |
| `bilateralBlur.glsl` | Edge-preserving blur |
| `boxBlur.glsl` | Fast box blur |
| `radialBlur.glsl` | Motion blur from center |
| `edge.glsl` | Sobel/Prewitt edge detection |
| `laplacian.glsl` | Laplacian edge |
| `kuwahara.glsl` | Painterly filter |
| `sharpen.glsl` | Unsharp mask |
| `smartDeNoise.glsl` | AI-inspired denoiser |
| `fibonacciBokeh.glsl` | Bokeh blur |

### sdf/ — Signed Distance Functions
| Module | Description |
|---|---|
| `circle.glsl` | 2D circle SDF |
| `box.glsl` | 2D/3D box SDF |
| `sphere.glsl` | 3D sphere SDF |
| `torus.glsl` | 3D torus SDF |
| `capsule.glsl` | Capsule SDF |
| `ops.glsl` | Union, subtract, intersect, smooth union |

### math/ — Math Utilities
- PI constants, smooth min/max, mix variants, SqrtLength

### space/ — Coordinate Transforms
- scale, rotate, lookAt, TBN matrix

### animation/ — Easing
- All CSS easing curves: `easeIn/Out/InOut`, elastic, bounce, cubic

## Worley Distance Options (lygia/generative/worley.glsl)

Configurable via defines before include:
```glsl
#define WORLEY_DIST_FNC worleyEuclidean  // default
// or: worleyManhattan, worleyChebyshev, worleyMinkowski
#define WORLEY_JITTER 1.0               // 0-1 cell irregularity
#include "lygia/generative/worley.glsl"
```

## Fresnel from LYGIA (confirmed from source)

```glsl
#include "lygia/lighting/fresnel.glsl"
// Provides: vec3 fresnel(vec3 f0, vec3 normal, vec3 view)
// Uses Schlick approximation:
// vec3 fresnel(const in vec3 f0, vec3 normal, vec3 view) {
//   return schlick(f0, 1.0, dot(view, normal));
// }
```

## Installation

```bash
# Via npm (for bundler projects)
npm install lygia
```

Or copy individual .glsl files directly — no runtime dependency, just include in your shader bundle.
