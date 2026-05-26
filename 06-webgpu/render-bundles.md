# WebGPU Render Bundles

Render bundles pre-record draw commands on the GPU, eliminating per-frame CPU overhead.
**Speedup: 2-5x** for scenes with many static objects.

## When to Use

- Static geometry (terrain, environment, furniture)
- Same draw calls repeated every frame
- High object count with no per-object JS updates

## API (Raw WebGPU)

```js
// 1. Create bundle encoder
const bundleEncoder = device.createRenderBundleEncoder({
  colorFormats: ['bgra8unorm'],
  depthStencilFormat: 'depth24plus-stencil8',
  sampleCount: 4  // if using MSAA
});

// 2. Record draw commands (exactly like a render pass)
bundleEncoder.setPipeline(pipeline);
bundleEncoder.setBindGroup(0, sceneBindGroup);
staticMeshes.forEach(mesh => {
  bundleEncoder.setBindGroup(1, mesh.bindGroup);
  bundleEncoder.setVertexBuffer(0, mesh.vertexBuffer);
  bundleEncoder.setIndexBuffer(mesh.indexBuffer, 'uint32');
  bundleEncoder.drawIndexed(mesh.indexCount);
});

// 3. Finish â€” creates immutable bundle
const bundle = bundleEncoder.finish();

// 4. Execute in render pass every frame â€” FAST
const pass = encoder.beginRenderPass(renderPassDesc);
pass.executeBundles([bundle]);  // replays all recorded commands
// Dynamic objects can still be drawn normally after
dynamicObjects.forEach(obj => drawDynamic(pass, obj));
pass.end();
```

## Three.js Integration (r184+)

Three.js WebGPURenderer does not expose render bundles directly via high-level API yet.
Access the underlying device via:

```js
const backend = renderer.backend;
const device  = backend.device;  // GPUDevice
```

Use custom render passes for bundle-eligible static meshes, then let Three.js handle dynamic objects.

## Bundle Invalidation

Bundles are immutable. Recreate when:
- Geometry changes (new buffers)
- Pipeline changes (material swap)
- Bind group layout changes

```js
// Pattern: dirty flag
let bundleDirty = true;
let cachedBundle = null;

function getBundle() {
  if (bundleDirty || !cachedBundle) {
    cachedBundle = buildBundle();
    bundleDirty = false;
  }
  return cachedBundle;
}
```

## Sources
- WebGPU spec â€” render bundles: https://www.w3.org/TR/webgpu/#render-bundle
- WebGPU samples: https://webgpu.github.io/webgpu-samples/
- Chrome blog: https://developer.chrome.com/docs/web-platform/webgpu/
