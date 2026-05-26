# Performance Monitoring Tools

## renderer.info (Three.js)

The ground-truth source for GPU resource usage.

```js
const info = renderer.info;

// Memory
info.memory.geometries   // Number of uploaded geometries
info.memory.textures     // Number of uploaded textures

// Render stats (per frame â€” reset with renderer.info.reset())
info.render.calls        // Draw calls this frame
info.render.triangles    // Triangles rendered
info.render.points       // Points rendered
info.render.lines        // Lines rendered
info.render.frame        // Frame count

// Programs
info.programs            // Array of compiled WebGLProgram objects
info.programs.length     // Shader variant count (watch for shader explosion)
```

## Custom HUD Overlay

```js
function createInfoHUD() {
  const el = document.createElement('div');
  Object.assign(el.style, {
    position: 'fixed', top: '10px', left: '10px',
    fontFamily: 'monospace', fontSize: '12px',
    color: '#0f0', background: 'rgba(0,0,0,0.6)',
    padding: '8px', borderRadius: '4px',
    pointerEvents: 'none', zIndex: '9999'
  });
  document.body.appendChild(el);

  return function update(info) {
    el.textContent = [
      `FPS: ${Math.round(1 / deltaTime)}`,
      `Draw calls: ${info.render.calls}`,
      `Triangles: ${(info.render.triangles / 1000).toFixed(1)}K`,
      `Geometries: ${info.memory.geometries}`,
      `Textures: ${info.memory.textures}`,
      `Programs: ${info.programs.length}`,
    ].join('\n');
  };
}
```

## stats.js

```bash
npm install stats.js
```

```js
import Stats from 'stats.js';

const stats = new Stats();
stats.showPanel(0); // 0: FPS, 1: MS, 2: MB
document.body.appendChild(stats.dom);

renderer.setAnimationLoop(() => {
  stats.begin();
  renderer.render(scene, camera);
  stats.end();
});
```

## r3f-perf (React Three Fiber)

```bash
npm install r3f-perf
```

```jsx
import { Perf } from 'r3f-perf';

function App() {
  return (
    <Canvas>
      <Perf position="top-left" />
      {/* scene */}
    </Canvas>
  );
}
```

Shows: FPS, GPU, CPU, memory, draw calls, triangle count.
âš ï¸ Last significant push: December 2024 (marginal â€” verify compatibility with current r3f)

## Lighthouse + Web Vitals

```bash
npm install -g lighthouse
lighthouse https://localhost:3000 --view --output html
```

Key metrics for 3D sites:
- **LCP** â€” Largest Contentful Paint: hide canvas until model loaded
- **CLS** â€” Cumulative Layout Shift: reserve canvas dimensions in CSS
- **INP** â€” Interaction to Next Paint: keep main thread free (use workers)
- **TBT** â€” Total Blocking Time: code-split heavy Three.js imports

## Sources
- Three.js WebGLInfo: https://threejs.org/docs/#api/en/renderers/WebGLRenderer.info
- stats.js: https://github.com/mrdoob/stats.js
- r3f-perf: https://github.com/RenaudRohlinger/r3f-perf (last push Dec 2024)
- Lighthouse: https://developer.chrome.com/docs/lighthouse/
