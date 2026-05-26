# Canvas 2D as a Three.js Texture

**Sources:** Three.js documentation, MDN Web Docs  
**Accessed:** 2026-05-26  
**Three.js version context:** r160+ (CanvasTexture API is stable across versions)

---

## 1. `CanvasTexture`

`THREE.CanvasTexture` wraps an `HTMLCanvasElement` (or `OffscreenCanvas`) as a Three.js texture. It is a subclass of `THREE.Texture` with one meaningful difference from the base class: its `needsUpdate` flag is set to `true` by default on construction so the first upload happens automatically.

### Constructor

```javascript
new THREE.CanvasTexture(
  canvas,           // HTMLCanvasElement | OffscreenCanvas
  mapping,          // optional, default THREE.UVMapping
  wrapS,            // optional, default THREE.ClampToEdgeWrapping
  wrapT,            // optional, default THREE.ClampToEdgeWrapping
  magFilter,        // optional, default THREE.LinearFilter
  minFilter,        // optional, default THREE.LinearMipmapLinearFilter
  format,           // optional, default THREE.RGBAFormat
  type,             // optional, default THREE.UnsignedByteType
  anisotropy        // optional, default 1
)
```

### Minimal usage

```javascript
const canvas = document.createElement('canvas');
canvas.width = 512;
canvas.height = 512;
const ctx = canvas.getContext('2d');

// Draw initial content
ctx.fillStyle = '#1a1a2e';
ctx.fillRect(0, 0, 512, 512);
ctx.fillStyle = '#ffffff';
ctx.font = '48px sans-serif';
ctx.fillText('Hello Three.js', 20, 256);

const texture = new THREE.CanvasTexture(canvas);

const material = new THREE.MeshBasicMaterial({ map: texture });
const mesh = new THREE.Mesh(new THREE.PlaneGeometry(2, 2), material);
scene.add(mesh);
```

> Source: Three.js documentation — CanvasTexture — accessed 2026-05-26  
> https://threejs.org/docs/#api/en/textures/CanvasTexture

---

## 2. `texture.needsUpdate = true` Each Frame

### Why it is required

Three.js does not watch the canvas for changes. The renderer caches the last uploaded texture on the GPU. To push new pixel data from the canvas to GPU memory, you must set the flag explicitly:

```javascript
function animate() {
  requestAnimationFrame(animate);

  // 1. Draw to the 2D canvas
  ctx.clearRect(0, 0, canvas.width, canvas.height);
  drawDynamicContent(ctx, Date.now());

  // 2. Signal Three.js to re-upload
  texture.needsUpdate = true;

  renderer.render(scene, camera);
}
```

Setting `needsUpdate = true` triggers a `gl.texImage2D` or `gl.texSubImage2D` call on the next render. The entire canvas is re-uploaded unless you use a partial update path — the base Three.js `CanvasTexture` has no built-in region-update API.

### The cost of `needsUpdate = true` every frame

Each `needsUpdate = true` causes Three.js to call `gl.texImage2D(GL_TEXTURE_2D, ...)` with the full canvas pixel data on the next render call. For a 512×512 RGBA canvas that is:

```
512 × 512 × 4 bytes = 1,048,576 bytes ≈ 1 MB upload per frame
```

At 60 fps, that is 60 MB/s of texture upload bandwidth, plus GPU driver overhead from the API call itself. For a 1024×1024 canvas the cost quadruples.

**Do not set `needsUpdate = true` on frames where the canvas did not change.**

```javascript
let canvasDirty = false;

function drawIfNeeded() {
  if (!needsRedraw()) return;
  ctx.clearRect(0, 0, canvas.width, canvas.height);
  drawContent(ctx);
  canvasDirty = true;
}

function animate() {
  requestAnimationFrame(animate);
  drawIfNeeded();
  if (canvasDirty) {
    texture.needsUpdate = true;
    canvasDirty = false;
  }
  renderer.render(scene, camera);
}
```

---

## 3. Performance of Dynamic Canvas Textures

### Filter settings for dynamic textures

The default `minFilter` is `THREE.LinearMipmapLinearFilter`, which requires a full mipmap chain. Every time `needsUpdate = true`, Three.js regenerates all mip levels — compounding the upload cost. For frequently updated textures, disable mipmapping:

```javascript
const texture = new THREE.CanvasTexture(canvas);
texture.minFilter = THREE.LinearFilter; // no mipmaps
texture.generateMipmaps = false;        // explicit guard
```

This eliminates mip generation overhead at the cost of quality at minified scales.

### Canvas resolution vs mesh size

Match the canvas resolution to the texel density you actually need. A 2048×2048 canvas texture on a 200px screen-space quad wastes 16 MB of VRAM and 16 MB/frame of upload bandwidth. Calculate the required texel density:

```
canvas_px = mesh_screen_size_px × devicePixelRatio × quality_factor
```

A typical UI label on a 3D object needs 256×64 at most.

### CPU drawing cost

The canvas 2D draw calls themselves (text rendering, `fillRect`, gradients) happen on the CPU. Complex draw routines that take more than ~4ms will reduce available frame budget for everything else. Pre-render static elements once; re-draw only the parts that change.

### Benchmarks (indicative, not from MDN or Three.js docs)

| Canvas size | RGBA upload cost (approx.) | Suitable for |
|-------------|---------------------------|--------------|
| 128×128 | ~64 KB/frame | Frequent updates, many textures |
| 256×256 | ~256 KB/frame | HUD elements, labels |
| 512×512 | ~1 MB/frame | Occasional updates only |
| 1024×1024 | ~4 MB/frame | Static or near-static content |
| 2048×2048 | ~16 MB/frame | Should use static texture instead |

---

## 4. When to Use Canvas Texture vs Shader-Based Text/UI

### Use `CanvasTexture` when

- Text or UI content is **driven by JavaScript data** (scores, labels, tooltips) and changes infrequently (< 10 fps equivalent change rate).
- You need **rich 2D layout** — multi-line text with wrapping, emoji, mixed fonts, CSS-style gradients — that would require a full SDF font atlas or complex shader logic to replicate.
- The texture updates are **event-driven** rather than every frame (player presses a button, label text changes).
- **Prototype speed** matters more than runtime performance.

```javascript
// Event-driven update — efficient
function updateScoreLabel(score) {
  ctx.clearRect(0, 0, canvas.width, canvas.height);
  ctx.fillText(`Score: ${score}`, 10, 50);
  texture.needsUpdate = true; // fires once on score change, not every frame
}
```

### Use shader-based text/UI when

- Text updates **every frame** (timers, live data feeds, particle labels).
- You are rendering **hundreds of labels simultaneously** — one canvas texture per label is impractical.
- You need **resolution independence** at arbitrary zoom levels (SDF fonts scale without pixelation).
- You want to keep the render pipeline entirely on the GPU with no CPU→GPU texture uploads.

Libraries like [troika-three-text](https://github.com/protectwise/troika/tree/main/packages/troika-three-text) implement SDF-based text rendering in Three.js and are the standard recommendation for text that changes frequently or at high density.

### Decision matrix

| Criterion | CanvasTexture | Shader / SDF |
|-----------|---------------|--------------|
| Update frequency | Low–medium (event-driven) | Any |
| Number of instances | Small (< 20) | Any |
| Rich layout (wrap, emoji) | Yes | Limited |
| Resolution independence | No (rasterized) | Yes |
| CPU cost | Drawing: yes | No |
| GPU upload cost | Yes, per update | No |
| Setup complexity | Low | High |

---

## 5. Practical Patterns

### Shared canvas, multiple materials

One canvas can back multiple `CanvasTexture` instances pointing to the same DOM element. Setting `needsUpdate = true` on one does not automatically update the others — you must flag each one.

If multiple meshes should share identical dynamic content, point them all at the same texture object (not separate `new CanvasTexture(sameCanvas)` calls):

```javascript
const texture = new THREE.CanvasTexture(canvas);
const mat1 = new THREE.MeshBasicMaterial({ map: texture });
const mat2 = new THREE.MeshBasicMaterial({ map: texture });
// One needsUpdate = true update is enough — both materials reference same texture object
```

### Power-of-two canvas dimensions

WebGL1 contexts require power-of-two (PoT) textures for repeat wrapping and full mipmap support. PoT dimensions also allow the GPU to optimize memory layout. Three.js will warn and resize non-PoT textures in certain configurations:

```javascript
// Prefer: 128, 256, 512, 1024, 2048
canvas.width  = 512;
canvas.height = 256;
```

WebGL2 (Three.js default when available) lifts the PoT requirement, but PoT sizes are still recommended for mipmap efficiency.

### Disposing textures

`CanvasTexture` holds a GPU texture handle. Dispose when the mesh is removed:

```javascript
texture.dispose();
```

Failure to dispose dynamic textures in scenes with frequent mesh creation/destruction causes VRAM leaks.
