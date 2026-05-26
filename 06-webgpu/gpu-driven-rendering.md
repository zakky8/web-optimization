# GPU-Driven Rendering (WebGPU)

## Indirect Draws

CPU submits draw parameters from a GPU buffer â€” enables GPU culling with zero CPU readback.

```js
// Indirect draw buffer layout (per draw):
// [indexCount, instanceCount, firstIndex, baseVertex, firstInstance]
// = 5 Ã— Uint32 = 20 bytes per draw call

const DRAW_COUNT = 1000;
const indirectBuffer = device.createBuffer({
  size: DRAW_COUNT * 20,
  usage: GPUBufferUsage.INDIRECT | GPUBufferUsage.STORAGE,
});

// Compute shader writes the indirect args after culling
const cullComputeModule = device.createShaderModule({ code: `
  @group(0) @binding(0) var<storage, read>       meshBounds:   array<AABB>;
  @group(0) @binding(1) var<storage, read_write> indirectArgs: array<DrawArgs>;
  @group(0) @binding(2) var<uniform>             frustum:      FrustumPlanes;

  @compute @workgroup_size(64)
  fn main(@builtin(global_invocation_id) id: vec3u) {
    let i = id.x;
    if (frustumCull(meshBounds[i], frustum)) {
      // Write real draw args
      indirectArgs[i] = DrawArgs(meshBounds[i].indexCount, 1u, ...);
    } else {
      // Cull: write 0 instances
      indirectArgs[i].instanceCount = 0u;
    }
  }
` });

// Execute culling
const cullPass = encoder.beginComputePass();
cullPass.setPipeline(cullPipeline);
cullPass.setBindGroup(0, cullBindGroup);
cullPass.dispatchWorkgroups(Math.ceil(DRAW_COUNT / 64));
cullPass.end();

// Draw with GPU-written args
const renderPass = encoder.beginRenderPass(renderPassDesc);
renderPass.setPipeline(renderPipeline);
for (let i = 0; i < DRAW_COUNT; i++) {
  renderPass.drawIndexedIndirect(indirectBuffer, i * 20);
}
renderPass.end();
```

## Timestamp Queries

Measure GPU-side execution time (requires `timestamp-query` feature).

```js
// 1. Check feature
const adapter = await navigator.gpu.requestAdapter();
if (!adapter.features.has('timestamp-query')) {
  throw new Error('timestamp-query not available');
}

const device = await adapter.requestDevice({
  requiredFeatures: ['timestamp-query']
});

// 2. Create query set + resolve buffer
const querySet = device.createQuerySet({
  type: 'timestamp',
  count: 2  // begin + end
});

const resolveBuffer = device.createBuffer({
  size: 16,  // 2 Ã— 8 bytes (BigUint64)
  usage: GPUBufferUsage.QUERY_RESOLVE | GPUBufferUsage.COPY_SRC,
});

const readbackBuffer = device.createBuffer({
  size: 16,
  usage: GPUBufferUsage.COPY_DST | GPUBufferUsage.MAP_READ,
});

// 3. Use in render pass descriptor
const renderPassDesc = {
  // ...
  timestampWrites: {
    querySet,
    beginningOfPassWriteIndex: 0,
    endOfPassWriteIndex: 1,
  }
};

// 4. Resolve after encoder.finish()
encoder.resolveQuerySet(querySet, 0, 2, resolveBuffer, 0);
encoder.copyBufferToBuffer(resolveBuffer, 0, readbackBuffer, 0, 16);

device.queue.submit([encoder.finish()]);

// 5. Read result (async)
await readbackBuffer.mapAsync(GPUMapMode.READ);
const times = new BigInt64Array(readbackBuffer.getMappedRange());
const gpuMs = Number(times[1] - times[0]) / 1_000_000;
console.log(`GPU frame: ${gpuMs.toFixed(3)}ms`);
readbackBuffer.unmap();
```

## Sources
- WebGPU spec indirect draws: https://www.w3.org/TR/webgpu/#dom-gpurenderpassencoder-drawindexedindirect
- Timestamp queries: https://www.w3.org/TR/webgpu/#dom-gpufeaturename-timestamp-query
- WebGPU best practices: https://toji.dev/webgpu-best-practices/
