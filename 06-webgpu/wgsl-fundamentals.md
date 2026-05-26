# WGSL Fundamentals

WGSL (WebGPU Shading Language) â€” W3C spec: https://www.w3.org/TR/WGSL/

## Key Syntax Differences vs GLSL

| Feature | GLSL | WGSL |
|---------|------|------|
| Compound assign | `x += 1.0;` | `x = x + 1.0;` (no `+=` in older spec) |
| Ternary | `a ? b : c` | `select(c, b, a)` |
| Types | `float`, `vec3` | `f32`, `vec3f` |
| Functions | `void main()` | `@vertex fn main() -> @builtin(position) vec4f` |
| Bindings | `uniform Block { }` | `@group(0) @binding(0) var<uniform> ...` |
| Loops | `for(int i=0;...)` | `for (var i: i32 = 0; i < 10; i++)` |
| Discard | `discard;` | `discard;` (same) |
| Modulo | `mod(x, y)` | `x % y` |

## Binding Groups

```wgsl
// Group 0: per-frame uniforms
@group(0) @binding(0) var<uniform> camera: CameraUniforms;
@group(0) @binding(1) var<uniform> time: f32;

// Group 1: material
@group(1) @binding(0) var baseColorTexture: texture_2d<f32>;
@group(1) @binding(1) var baseColorSampler: sampler;

// Group 2: per-object
@group(2) @binding(0) var<uniform> model: mat4x4f;

// Group 3: storage (compute/read-write)
@group(3) @binding(0) var<storage, read_write> positions: array<vec3f>;
```

## Vertex + Fragment Shader

```wgsl
struct VertexInput {
  @location(0) position: vec3f,
  @location(1) normal:   vec3f,
  @location(2) uv:       vec2f,
}

struct VertexOutput {
  @builtin(position) clipPosition: vec4f,
  @location(0)       worldNormal:  vec3f,
  @location(1)       texCoord:     vec2f,
}

@vertex
fn vsMain(in: VertexInput) -> VertexOutput {
  var out: VertexOutput;
  out.clipPosition = camera.viewProj * model * vec4f(in.position, 1.0);
  out.worldNormal  = normalize((model * vec4f(in.normal, 0.0)).xyz);
  out.texCoord     = in.uv;
  return out;
}

@fragment
fn fsMain(in: VertexOutput) -> @location(0) vec4f {
  let color = textureSample(baseColorTexture, baseColorSampler, in.texCoord);
  let NdotL = max(dot(in.worldNormal, vec3f(0.0, 1.0, 0.0)), 0.0);
  return vec4f(color.rgb * NdotL, color.a);
}
```

## Compute Shader

```wgsl
@group(0) @binding(0) var<storage, read_write> data: array<f32>;

@compute @workgroup_size(64)
fn csMain(@builtin(global_invocation_id) id: vec3u) {
  let i = id.x;
  if (i >= arrayLength(&data)) { return; }
  data[i] = data[i] * 2.0;
}
```

## select() â€” WGSL Ternary

```wgsl
// select(false_val, true_val, condition)
let result = select(0.0, 1.0, x > 0.5);  // = x > 0.5 ? 1.0 : 0.0
let clamped = select(select(x, min_val, x < min_val), max_val, x > max_val);
```

## Sources
- W3C WGSL spec: https://www.w3.org/TR/WGSL/
- WebGPU fundamentals: https://webgpufundamentals.org/
- Three.js WGSL backend: three/src/renderers/webgpu/utils/WebGPUPipelineUtils.js
