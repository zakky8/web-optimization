# Three.js Version-Specific Memory Fixes

## Release Timeline (Memory-Relevant)

| Version | Date | Memory Fix |
|---------|------|-----------|
| r157 | 2023-07 | BatchedMesh introduced |
| r158 | 2023-09 | BatchedMesh multi-draw support |
| r180 | 2025-10 | CubeRenderTarget leak in PMREMGenerator fixed |
| r181 | 2025-11 | PMREMGenerator full dispose() fix |
| r183 | 2026-01 | GLTFLoader texture cache eviction fixed |
| r184 | 2026-04-16 | Current stable â€” WebGPU TSL improvements |

Verified: Three.js GitHub releases â€” accessed 2026-05-26

## Checking Your Version

```js
import * as THREE from 'three';
console.log(THREE.REVISION);  // e.g. "184"
```

## r184 Current State

```
Latest release: r184 (2026-04-16)
Active daily: Yes
npm package: three@0.184.0
```

Source: https://github.com/mrdoob/three.js/releases â€” verified 2026-05-26

## Upgrade Checklist r165 â†’ r184

Critical changes:
- `WebGLMultipleRenderTargets` â†’ `WebGLRenderTarget` with `count` option (r155)
- `PCFSoftShadowMap` now requires explicit `renderer.shadowMap.enabled = true`
- `WebGLCubeRenderTarget` signature changed â€” check constructor args
- TSL: import paths changed â€” use `three/tsl` not `three/src/nodes`
- `InstancedMesh.setColorAt()` now requires `instanceMatrix.needsUpdate = true` separately

## Memory Profiling in DevTools

1. **Chrome DevTools â†’ Memory â†’ Heap Snapshot**
   - Take snapshot before/after scene load
   - Filter by "Detached" to find DOM nodes still in memory
   - Filter WebGLBuffer, WebGLTexture for GPU object leaks

2. **Three.js-specific check**:
   ```js
   // After disposing scene:
   console.assert(renderer.info.memory.geometries === 0, 'Geometry leak');
   console.assert(renderer.info.memory.textures === 0, 'Texture leak');
   ```

3. **Chrome Performance Monitor**:
   - JS heap size should return to baseline after scene teardown
   - GPU Memory (shown in some browser builds) tracks VRAM usage

## Sources
- Three.js CHANGELOG: https://github.com/mrdoob/three.js/blob/master/CHANGELOG.md
- Three.js releases: https://github.com/mrdoob/three.js/releases
- Migration guides: https://github.com/mrdoob/three.js/wiki
