# WebGPU

WebGPU is the successor to WebGL, exposing a low-level GPU API that maps closely to modern graphics APIs (Vulkan, Metal, Direct3D 12). It shipped in Chrome 113 (May 2023), Firefox 118 (Sept 2023), and Safari 26 (2025), giving roughly 95% global browser coverage as of 2026 with automatic WebGL 2 fallback for the remainder.

## Why WebGPU vs WebGL

| Concern | WebGL 2 | WebGPU |
|---|---|---|
| Draw call CPU overhead | High | Low (bind groups reduce per-draw state) |
| Compute shaders | No (fragment shader workarounds) | First-class compute pipeline |
| Multi-threading | None | GPUQueue on workers |
| Shader language | GLSL | WGSL |
| Error model | Driver-specific | Explicit validation layer |

## Basic Setup

### Adapter and Device

```js
const adapter = await navigator.gpu.requestAdapter({ powerPreference: "high-performance" });
if (!adapter) throw new Error("WebGPU not supported");
const device = await adapter.requestDevice();

const canvas = document.querySelector("canvas");
const context = canvas.getContext("webgpu");
const format = navigator.gpu.getPreferredCanvasFormat();
context.configure({ device, format, alphaMode: "premultiplied" });
```

### Render Pipeline (Triangle)

```js
const shader = device.createShaderModule({
  code: `
    @vertex fn vs(@builtin(vertex_index) vi: u32) -> @builtin(position) vec4f {
      var pos = array<vec2f, 3>(
        vec2f( 0.0,  0.5), vec2f(-0.5, -0.5), vec2f( 0.5, -0.5),
      );
      return vec4f(pos[vi], 0.0, 1.0);
    }
    @fragment fn fs() -> @location(0) vec4f { return vec4f(1.0, 0.4, 0.0, 1.0); }
  `
});

const pipeline = device.createRenderPipeline({
  layout: "auto",
  vertex:   { module: shader, entryPoint: "vs" },
  fragment: { module: shader, entryPoint: "fs", targets: [{ format }] },
  primitive: { topology: "triangle-list" }
});

function frame() {
  const encoder = device.createCommandEncoder();
  const pass = encoder.beginRenderPass({
    colorAttachments: [{
      view: context.getCurrentTexture().createView(),
      clearValue: [0, 0, 0, 1],
      loadOp: "clear", storeOp: "store"
    }]
  });
  pass.setPipeline(pipeline);
  pass.draw(3);
  pass.end();
  device.queue.submit([encoder.finish()]);
  requestAnimationFrame(frame);
}
requestAnimationFrame(frame);
```

## WGSL

WGSL is statically typed and Rust-flavored. Key differences from GLSL:

```wgsl
// Explicit types, no implicit conversions
var x: f32 = 1.0;

struct VertOut {
  @builtin(position) pos: vec4f,
  @location(0) uv: vec2f,
}

@vertex fn vs(@location(0) position: vec3f, @location(1) uv: vec2f) -> VertOut {
  var out: VertOut;
  out.pos = vec4f(position, 1.0);
  out.uv  = uv;
  return out;
}

// Uniforms via bind groups (not global uniforms)
@group(0) @binding(0) var<uniform> mvp: mat4x4f;
@group(0) @binding(1) var myTex: texture_2d<f32>;
@group(0) @binding(2) var mySampler: sampler;

@fragment fn fs(in: VertOut) -> @location(0) vec4f {
  return textureSample(myTex, mySampler, in.uv);
}
```

## Compute Shaders

Compute shaders run arbitrary parallel workloads without a render pipeline.

### Workgroup Model

```
dispatchWorkgroups(x, y, z)        <- grid of workgroups
  each workgroup @workgroup_size(wx, wy, wz) threads
    each thread exposes:
      global_invocation_id          <- unique across entire dispatch
      local_invocation_id           <- unique within workgroup
      workgroup_id
```

General advice: workgroup size of 64 unless profiling says otherwise. Occupancy peaks when total threads is a multiple of the warp/wave size (32 on NVIDIA, 64 on AMD).

### Sum Reduction

```wgsl
@group(0) @binding(0) var<storage, read>       input:  array<f32>;
@group(0) @binding(1) var<storage, read_write> output: array<f32>;

var<workgroup> shared: array<f32, 64>;

@compute @workgroup_size(64)
fn main(
  @builtin(global_invocation_id) gid: vec3u,
  @builtin(local_invocation_id)  lid: vec3u,
) {
  let i = gid.x;
  shared[lid.x] = select(0.0, input[i], i < arrayLength(&input));
  workgroupBarrier();

  for (var stride = 32u; stride > 0u; stride >>= 1u) {
    if (lid.x < stride) { shared[lid.x] += shared[lid.x + stride]; }
    workgroupBarrier();
  }
  if (lid.x == 0u) { output[gid.x / 64u] = shared[0]; }
}
```

```js
const pass = encoder.beginComputePass();
pass.setPipeline(computePipeline);
pass.setBindGroup(0, bindGroup);
pass.dispatchWorkgroups(Math.ceil(N / 64));
pass.end();
```

## TSL — Three Shading Language

TSL is Three.js's JavaScript-native shader authoring system. It compiles to both WGSL (WebGPU) and GLSL (WebGL), so one codebase covers both renderers. Stabilized in r170+.

```js
import { WebGPURenderer }            from "three/webgpu";
import { MeshStandardNodeMaterial }  from "three/webgpu";
import { normalWorld, vec3, Fn, uniform } from "three/tsl";

const renderer = new WebGPURenderer({ antialias: true });
await renderer.init(); // required for WebGPU

const uTime = uniform(0);
const mat   = new MeshStandardNodeMaterial();

mat.colorNode = Fn(() => {
  const n = normalWorld.normalize();
  return vec3(
    n.x.mul(0.5).add(0.5),
    n.y.mul(0.5).add(0.5),
    n.z.mul(0.5).add(0.5)
  );
})();

renderer.setAnimationLoop(() => {
  uTime.value += 0.016;
  renderer.render(scene, camera);
});
```

### GLSL ShaderMaterial to TSL

```js
// Before (WebGL only)
const mat1 = new THREE.ShaderMaterial({
  uniforms: { uTime: { value: 0 } },
  vertexShader: `
    uniform float uTime;
    void main() {
      vec3 pos = position;
      pos.y += sin(pos.x * 3.0 + uTime) * 0.2;
      gl_Position = projectionMatrix * modelViewMatrix * vec4(pos, 1.0);
    }`,
  fragmentShader: `void main() { gl_FragColor = vec4(1.0, 0.5, 0.0, 1.0); }`
});

// After (works on WebGPU and WebGL)
import { MeshBasicNodeMaterial }            from "three/webgpu";
import { positionLocal, sin, vec4, Fn, uniform } from "three/tsl";

const uTime2 = uniform(0);
const mat2   = new MeshBasicNodeMaterial();
mat2.positionNode = Fn(() => {
  const pos = positionLocal.toVar();
  pos.y = pos.y.add(sin(pos.x.mul(3.0).add(uTime2)).mul(0.2));
  return pos;
})();
mat2.colorNode = vec4(1.0, 0.5, 0.0, 1.0);
```

## Bind Group Strategy

Organize uniforms by update frequency:

| Group | Contents | Updates |
|---|---|---|
| 0 | Camera matrices, time | Once per frame |
| 1 | Model matrix | Once per draw |
| 2 | Material textures | Once per material |

Sort draw calls: pipeline then group 0 then group 1 then group 2.

## Performance Implications

| Workload | WebGL | WebGPU |
|---|---|---|
| 1K draw calls/frame | Baseline | ~2x via lower CPU overhead |
| 100K particle sim | FBO ping-pong hack | Native compute, 5-10x faster |
| Complex post-processing | Fragment shader chains | Compute passes |
| Mobile Apple A-series | Good | Metal backend, measurable gain |

- CPU-to-GPU buffer copies are frequently the bottleneck. Keep data on the GPU longer.
- Use `GPUBuffer.getMappedRange()` for async readback. Never block the main thread with sync reads.
- Timestamp queries via `device.createQuerySet({ type: "timestamp" })` for GPU-side profiling.
- Cache all pipelines at startup. Never recreate pipelines per frame.

## Feature Detection

```js
async function createRenderer(canvas) {
  if (navigator.gpu) {
    const adapter = await navigator.gpu.requestAdapter();
    if (adapter) {
      // WebGPURenderer auto-falls back to WebGL 2 if needed
      const renderer = new WebGPURenderer({ canvas, antialias: true });
      await renderer.init();
      return renderer;
    }
  }
  return new THREE.WebGLRenderer({ canvas, antialias: true });
}
```

## Tools and Libraries

| Tool | Purpose |
|---|---|
| [webgpufundamentals.org](https://webgpufundamentals.org) | Ground-up tutorials |
| [Three.js WebGPURenderer](https://threejs.org/docs/#api/en/renderers/webgpu/WebGPURenderer) | Drop-in renderer |
| [TSL wiki](https://github.com/mrdoob/three.js/wiki/Three.js-Shading-Language) | Official TSL reference |
| [Maxime Heckel TSL guide](https://blog.maximeheckel.com/posts/field-guide-to-tsl-and-webgpu/) | Deep TSL walkthrough |
| [WebGPU Report](https://webgpureport.org) | Inspect adapter limits |
| [wgpu-matrix](https://github.com/greggman/wgpu-matrix) | Math lib for WebGPU |
| Chrome DevTools GPU tab | Frame capture, pipeline inspection |
