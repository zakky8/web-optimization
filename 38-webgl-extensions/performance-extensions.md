# Performance-Oriented WebGL2 Extensions

Four extensions that directly affect CPU/GPU throughput: reducing draw call overhead, parallelising shader compilation, and measuring GPU timing. None are universally available — all require availability checks with fallback paths.

Sources verified: MDN Web Docs, web3dsurvey.com/webgl2. Date: 2026-05-26.

---

## 1. WEBGL_multi_draw

### What it does

Collapses multiple `drawArrays` / `drawElements` / instanced variants into a **single GPU command**, passing all per-draw parameters (offset, count, instance count) in typed arrays. The primary benefit is CPU-side: it eliminates the JavaScript overhead and driver state validation of N individual draw calls, reducing both call stack cost and WebGL command buffer serialisation.

The extension also exposes `gl_DrawID` in vertex shaders — a built-in integer containing the zero-based index of the current draw call within the multi-draw batch. This enables per-draw data lookups in texture or UBO arrays without per-instance buffers.

Support rate: **93.41%** (web3dsurvey.com, accessed 2026-05-26). Good desktop coverage; check mobile.

Works on both WebGL1 and WebGL2 contexts. Implicitly enables `ANGLE_instanced_arrays`.

### Methods

| Method | Equivalent to |
|---|---|
| `ext.multiDrawArraysWEBGL(mode, firsts, firstsOffset, counts, countsOffset, drawCount)` | N × `gl.drawArrays()` |
| `ext.multiDrawElementsWEBGL(mode, counts, countsOffset, type, offsets, offsetsOffset, drawCount)` | N × `gl.drawElements()` |
| `ext.multiDrawArraysInstancedWEBGL(mode, firsts, firstsOffset, counts, countsOffset, instanceCounts, instanceCountsOffset, drawCount)` | N × `gl.drawArraysInstanced()` |
| `ext.multiDrawElementsInstancedWEBGL(mode, counts, countsOffset, type, offsets, offsetsOffset, instanceCounts, instanceCountsOffset, drawCount)` | N × `gl.drawElementsInstanced()` |

All typed-array parameters accept an offset (in elements) into the array, enabling reuse of larger pre-allocated buffers without slicing.

### How to use

```js
const ext = gl.getExtension('WEBGL_multi_draw');

// Pre-allocate parameter arrays once per frame
const firsts        = new Int32Array(objectCount);
const counts        = new Int32Array(objectCount);
const instanceCounts = new Int32Array(objectCount);

// Fill per-frame
for (let i = 0; i < objectCount; i++) {
  firsts[i]         = drawData[i].startVertex;
  counts[i]         = drawData[i].vertexCount;
  instanceCounts[i] = drawData[i].instances;
}

if (ext) {
  ext.multiDrawArraysInstancedWEBGL(
    gl.TRIANGLES,
    firsts, 0,
    counts, 0,
    instanceCounts, 0,
    objectCount
  );
} else {
  // Fallback: individual draw calls
  for (let i = 0; i < objectCount; i++) {
    gl.drawArraysInstanced(gl.TRIANGLES, firsts[i], counts[i], instanceCounts[i]);
  }
}
```

### Shader usage (gl_DrawID)

The `gl_DrawID` built-in requires an explicit `#extension` directive:

```glsl
#version 300 es
#extension GL_ANGLE_multi_draw : require

// Note: directive name is GL_ANGLE_multi_draw even though the JS extension
// is named WEBGL_multi_draw.

void main() {
  // Use gl_DrawID to index into a per-object data texture or UBO array
  int drawIndex = gl_DrawID;
  // ...
}
```

### Fallback

Loop over individual `drawArraysInstanced` / `drawElementsInstanced` calls. No functionality is lost — only CPU-side batching efficiency.

---

## 2. WEBGL_draw_instanced_base_vertex_base_instance

### What it does

Extends instanced drawing with two additional base parameters:
- **baseVertex** — an integer offset added to every vertex index fetched from the element buffer, enabling multiple meshes to share a single merged vertex buffer without re-binding.
- **baseInstance** — the first instance ID used when fetching instanced vertex attributes (and the starting value of `gl_InstanceID` / `gl_BaseInstance`).

This is the WebGL equivalent of OpenGL's `glDrawElementsInstancedBaseVertexBaseInstance`. It is essential for GPU-driven rendering pipelines where all geometry lives in one large buffer and draw parameters are generated on the GPU.

MDN does not have a standalone page for this extension (as of 2026-05-26). It is defined in the Khronos WebGL extension registry; check `getSupportedExtensions()` output at runtime.

### How to use

```js
const ext = gl.getExtension('WEBGL_draw_instanced_base_vertex_base_instance');

if (ext) {
  // drawArraysInstancedBaseInstanceWEBGL
  ext.drawArraysInstancedBaseInstanceWEBGL(
    gl.TRIANGLES,
    first,          // starting vertex index
    count,          // vertices per instance
    instanceCount,
    baseInstance    // first instance ID
  );

  // drawElementsInstancedBaseVertexBaseInstanceWEBGL
  ext.drawElementsInstancedBaseVertexBaseInstanceWEBGL(
    gl.TRIANGLES,
    count,
    gl.UNSIGNED_SHORT,
    offset,         // byte offset into element buffer
    instanceCount,
    baseVertex,     // added to each index
    baseInstance
  );
} else {
  // Fallback: split merged buffers or use separate bind-and-draw per mesh
}
```

### Fallback

Without `baseVertex` support, merged geometry buffers require either separate VBOs per mesh (more binds) or a manual index rebasing step on the CPU when building index data. Without `baseInstance`, GPU-driven indirect workflows degrade to per-mesh uniform updates.

---

## 3. KHR_parallel_shader_compile

### What it does

By default, calling `gl.getProgramParameter(prog, gl.LINK_STATUS)` — or any draw call that implicitly stalls on a program — forces the driver to **block the main thread** until shader compilation completes. On first load or when many shaders are compiled in sequence, this causes visible frame drops (the "shader stutter" problem).

`KHR_parallel_shader_compile` introduces a single constant, `COMPLETION_STATUS_KHR`, that can be polled **without blocking**. You can call `gl.getProgramParameter(prog, ext.COMPLETION_STATUS_KHR)` any number of times; it returns `false` while compilation is in progress and `true` when done, never stalling the thread.

Support rate: **72.04%** (web3dsurvey.com, accessed 2026-05-26). Not universally available — the fallback is mandatory.

Works on both WebGL1 and WebGL2 contexts.

### Constants

| Constant | Purpose |
|---|---|
| `ext.COMPLETION_STATUS_KHR` | Query whether a program or shader has finished compilation/linking asynchronously |

### How to use

```js
const ext = gl.getExtension('KHR_parallel_shader_compile');

// Kick off compilation for all programs first (do not interleave with status checks)
for (const shader of vertexShaders)   gl.compileShader(shader);
for (const shader of fragmentShaders) gl.compileShader(shader);
for (const program of programs)       gl.linkProgram(program);

// Poll on a per-frame basis — do not spin-wait
function checkCompilation(pendingPrograms) {
  if (!pendingPrograms.length) return;

  if (ext) {
    // Non-blocking: filter out programs that are done
    const remaining = pendingPrograms.filter(
      (prog) => !gl.getProgramParameter(prog, ext.COMPLETION_STATUS_KHR)
    );

    if (remaining.length < pendingPrograms.length) {
      // Some programs just finished — check LINK_STATUS only for newly done ones
      const done = pendingPrograms.filter(
        (prog) => !remaining.includes(prog)
      );
      for (const prog of done) {
        if (!gl.getProgramParameter(prog, gl.LINK_STATUS)) {
          console.error(gl.getProgramInfoLog(prog));
        }
      }
    }

    requestAnimationFrame(() => checkCompilation(remaining));
  } else {
    // Fallback: synchronous — check one program per frame to spread stalls
    const prog = pendingPrograms[0];
    // Querying LINK_STATUS forces a synchronous stall here
    if (!gl.getProgramParameter(prog, gl.LINK_STATUS)) {
      console.error(gl.getProgramInfoLog(prog));
    }
    requestAnimationFrame(() => checkCompilation(pendingPrograms.slice(1)));
  }
}

checkCompilation(programs);
```

### Key caveats

- Kick off **all** `compileShader` and `linkProgram` calls before polling. Each call submits work to the driver's background compilation queue; polling before submitting everything under-utilises parallelism.
- Do not use `COMPLETION_STATUS_KHR` as a busy-wait in a synchronous loop — this defeats the purpose and can lock up the tab.
- When the extension is absent, querying `gl.LINK_STATUS` still works; it just stalls the thread.

---

## 4. EXT_disjoint_timer_query_webgl2

### What it does

Provides GPU-side timing queries for WebGL2 contexts. Unlike CPU `performance.now()` measurements, timer queries record when the GPU **begins and finishes** executing a command range, giving true GPU execution time unaffected by driver call batching or CPU synchronisation points.

**WebGL2 context only.** For WebGL1, use the older `EXT_disjoint_timer_query` which has a different API (separate `createQueryEXT` / `beginQueryEXT` methods on the extension object rather than the context).

Support rate: **64.1%** (web3dsurvey.com, accessed 2026-05-26). Limited — check before using.

**Security note:** This extension has historically been disabled in browsers following Spectre disclosures (high-resolution timers can be used for cache timing attacks). Some browsers require a flag or restrict it to secure contexts with appropriate COOP/COEP headers. Do not rely on it in production user-facing code; use it for developer tooling and profiling builds only.

### How to use

```js
const ext = gl.getExtension('EXT_disjoint_timer_query_webgl2');

if (ext) {
  // Create a query object using the standard WebGL2 API
  const query = gl.createQuery();

  // Begin a GPU timer for a range of draw calls
  gl.beginQuery(ext.TIME_ELAPSED_EXT, query);

  // ... issue draw calls to profile here ...
  drawScene();

  // End the timer
  gl.endQuery(ext.TIME_ELAPSED_EXT);

  // Do NOT check the result in the same frame — GPU is still executing
  // Poll in a subsequent frame
  function pollQueryResult() {
    // First check for GPU disjoint events (context switches, power state changes)
    // that would invalidate timing results
    const disjoint = gl.getParameter(ext.GPU_DISJOINT_EXT);
    if (disjoint) {
      console.warn('GPU disjoint — timing result invalid, discarding');
      gl.deleteQuery(query);
      return;
    }

    // Check if result is available (non-blocking)
    const available = gl.getQueryParameter(query, gl.QUERY_RESULT_AVAILABLE);
    if (!available) {
      requestAnimationFrame(pollQueryResult);
      return;
    }

    // Result ready — read elapsed nanoseconds
    const elapsedNs = gl.getQueryParameter(query, gl.QUERY_RESULT);
    console.log(`GPU frame time: ${(elapsedNs / 1e6).toFixed(3)} ms`);
    gl.deleteQuery(query);
  }

  requestAnimationFrame(pollQueryResult);
}
```

### Constants

| Constant | Purpose |
|---|---|
| `ext.TIME_ELAPSED_EXT` | Target for `beginQuery` / `endQuery` — measures elapsed GPU time for a command range |
| `ext.TIMESTAMP_EXT` | Target for `gl.queryCounterEXT` (if available) — records current GPU timestamp |
| `ext.GPU_DISJOINT_EXT` | Pass to `gl.getParameter()` — returns `true` if a disjoint event invalidated recent timers |

### Fallback

No equivalent for measuring true GPU time without this extension. Use CPU-side `performance.now()` timestamps around `gl.finish()` calls as a very rough proxy, accepting that `gl.finish()` serialises the pipeline and introduces its own overhead.

---

## Summary table

| Extension | Support rate | Context | Risk to ship without |
|---|---|---|---|
| `WEBGL_multi_draw` | 93.41% | WebGL1 + WebGL2 | Performance regression only — safe to degrade |
| `WEBGL_draw_instanced_base_vertex_base_instance` | Variable | WebGL2 | Required for GPU-driven pipelines; fallback is per-mesh binds |
| `KHR_parallel_shader_compile` | 72.04% | WebGL1 + WebGL2 | Shader stutter on first load — fallback is sync stall |
| `EXT_disjoint_timer_query_webgl2` | 64.1% | WebGL2 only | Dev tooling only — never block shipping on this |
