# Contact Shadows — drei ContactShadows

**Knowledge base entry for:** `web-optimization / 29-shadows-advanced`
**Last verified:** 2026-05-26
**Sources:** drei source `src/core/ContactShadows.tsx` (raw GitHub, accessed 2026-05-26), drei docs at `drei.docs.pmnd.rs/staging/contact-shadows` (accessed 2026-05-26)

---

## What It Is

`ContactShadows` is a React Three Fiber component from [`@react-three/drei`](https://github.com/pmndrs/drei) that renders a ground-plane shadow without using Three.js's native shadow map pipeline. It is a fake shadow technique: no lights participate, no `castShadow`/`receiveShadow` flags are used.

The technique is ported from the Three.js contact shadow example (`examples/webgl_shadowmap_progressive.html`). The result looks like soft, blurred shadows pooling under objects — closest to what you'd see from a soft overhead area light.

---

## How It Works Internally

### Render pipeline (one frame)

1. **Depth capture pass**
   - An `OrthographicCamera` is positioned above the plane, pointing downward.
   - `gl.render(scene, shadowCamera)` is called with all scene materials temporarily replaced by a custom `MeshDepthMaterial`-derived material that renders depth as a dark color. The background is white (fully lit / no shadow).
   - The result is written to a `WebGLRenderTarget` (the shadow texture).

2. **Horizontal blur pass**
   - The shadow render target is blurred horizontally using `HorizontalBlurShader` from `three-stdlib` into a second render target.

3. **Vertical blur pass**
   - The horizontally-blurred target is blurred vertically using `VerticalBlurShader` back into the first target.

4. **Optional secondary blur (`smooth: true`)**
   - A second blur cycle is applied. This softens the shadow further and produces a more naturalistic, rounded appearance at the cost of one extra pair of blur passes.

5. **Display**
   - A `PlaneGeometry` aligned with the XZ plane (facing up) displays the final shadow texture using a `MeshBasicMaterial` with `transparent: true`, `opacity` set to the `opacity` prop, and `depthWrite: false`.

### Key design consequences
- The shadow is always projected onto a flat horizontal plane — there is no geometry awareness for non-planar receivers.
- The scene is rendered an extra time per frame from above (unless `frames={1}` is used).
- The camera captures everything within the `scale` extent and `far` distance. Objects outside this volume cast no shadow.

---

## Component API

```tsx
import { ContactShadows } from '@react-three/drei'
```

### Type signature

```ts
type ContactShadowsProps = Omit<ThreeElements['group'], 'ref' | 'scale'> & {
  opacity?: number                      // default: 1
  width?: number                        // default: 1 (world units, multiplied by scale)
  height?: number                       // default: 1 (world units, multiplied by scale)
  blur?: number                         // default: 1
  near?: number                         // default: 0 (shadow camera near, relative to the group)
  far?: number                          // default: 10 (how high above the plane to capture)
  smooth?: boolean                      // default: true (second blur pass for softer result)
  resolution?: number                   // default: 256 (render target pixel resolution)
  frames?: number                       // default: Infinity (renders per frame; set 1 for static)
  scale?: number | [x: number, y: number] // default: 10 (world-unit footprint of the shadow plane)
  color?: THREE.ColorRepresentation     // default: '#000000'
  depthWrite?: boolean                  // default: false
  renderOrder?: number
}
```

### Props in depth

#### `opacity` (default: `1`)
`MeshBasicMaterial.opacity` of the shadow plane. Range `0–1`. Lower values produce more transparent, ghostly shadows. Use `0.5–0.8` for realism.

#### `scale` (default: `10`)
World-unit size of the shadow-receiving plane. A single number = square; `[x, y]` = rectangular footprint. The orthographic shadow camera frustum is sized to match. If objects cast shadows outside this area, their shadow contribution is clipped.

#### `blur` (default: `1`)
Controls the blur kernel width passed to `HorizontalBlurShader` and `VerticalBlurShader`. Higher values = softer, more spread-out shadows. Values above `3–5` can produce visually disconnected shadows that look like glow rather than contact shadows.

Typical range for realistic look: `0.5–2.5`.

#### `resolution` (default: `256`)
Pixel resolution of the shadow render target (square). Higher values = sharper shadow edges before blur. Since the blur is applied post-render, you can often get good results at `256–512` without needing `1024+`.

Cost: `resolution × resolution × 4 bytes` for the render target texture. At `512`: 1MB; at `1024`: 4MB.

#### `far` (default: `10`)
Distance above the plane up to which objects are captured. The orthographic camera is placed at `far` world units above the group position and looks down toward `near`. Objects above this height are not captured.

Increase for tall objects, but be aware this also reduces the effective depth resolution of the shadow camera.

#### `near` (default: `0`)
The near plane of the shadow camera relative to the group. Usually left at `0`. Increase if objects very close to the plane cause artifacts (the plane itself may be captured).

#### `smooth` (default: `true`)
Runs a second pair of horizontal+vertical blur passes. Produces a noticeably softer result with more circular falloff. Disable for sharper contact shadows or to save the extra two draw calls.

#### `frames` (default: `Infinity`)
Number of frames the shadow is rendered. `Infinity` recomputes every frame (required for moving objects). `1` renders once and never updates — ideal for static scenes (furniture, architecture). Any positive integer N renders for the first N frames then stops.

```jsx
{/* Static architecture — bake once */}
<ContactShadows frames={1} />

{/* Animated character — update every frame */}
<ContactShadows frames={Infinity} />
```

#### `color` (default: `'#000000'`)
Color of the shadow. Black is standard. Can be set to a warm color (e.g., `'#200000'`) for stylized looks.

#### `depthWrite` (default: `false`)
Passed directly to `MeshBasicMaterial.depthWrite`. Keep `false` to prevent z-fighting between the shadow plane and the ground mesh. If the shadow plane IS the only ground, you may set this to `true`.

---

## Positioning

`ContactShadows` is a `group`. Position it at ground level. It renders the shadow texture looking downward from `far` units above the group origin.

```jsx
{/* Ground at y=0 */}
<ContactShadows
  position={[0, 0, 0]}   // group origin = shadow plane position
  scale={8}
  far={5}
  blur={2}
  opacity={0.6}
/>
```

If the ground is not at Y=0 in your scene, adjust position accordingly:
```jsx
<ContactShadows position={[0, -1.5, 0]} scale={6} far={3} />
```

---

## When to Use ContactShadows Instead of Native Shadow Maps

| Scenario | ContactShadows | Native shadow maps |
|---|---|---|
| Static object on a flat surface | Excellent — `frames={1}`, zero runtime cost | Works but overkill |
| Product viewer / configurator | Very good — object changes, scene is simple | Overkill |
| Multiple objects on a complex floor | Poor — only flat receiver | Use native maps |
| Moving objects | Works — `frames={Infinity}` | More accurate |
| Non-planar receivers (stairs, terrain) | Not supported | Required |
| Outdoor sun/sky with terrain | Not appropriate | Required |
| Shadow from multiple light directions | Not possible (single top-down capture) | Supported per light |

**Rule of thumb:** ContactShadows excels in product, portfolio, and hero-object scenes where a flat or near-flat ground plane is acceptable and shadow quality needs to look polished with minimal setup. For game-like scenes with complex geometry, use native shadow maps or CSM.

---

## Performance Characteristics

Every frame with `frames={Infinity}`:
- 1 extra full scene render (depth capture from above)
- 2–4 full-screen quad passes (blur)
- Additional GPU memory: `resolution × resolution × 4 bytes` × 2 render targets

At `resolution={512}`: roughly 2MB texture memory + the cost of the extra render pass.

For a static scene at `frames={1}`, runtime cost after the first frame is effectively zero — just a single textured plane draw.

---

## Minimal JSX Example

```jsx
import { ContactShadows } from '@react-three/drei'

function Scene() {
  return (
    <>
      <mesh position={[0, 1, 0]} castShadow={false}>
        <boxGeometry />
        <meshStandardMaterial color="orange" />
      </mesh>

      <ContactShadows
        position={[0, 0, 0]}
        opacity={0.7}
        scale={10}
        blur={2.5}
        far={4}
        resolution={512}
        color="#000000"
        frames={Infinity}
      />
    </>
  )
}
```

---

## Sources

```yaml
- claim: "ContactShadows renders from above using an OrthographicCamera into a WebGLRenderTarget, then applies HorizontalBlurShader + VerticalBlurShader"
  sources:
    - url: "https://raw.githubusercontent.com/pmndrs/drei/master/src/core/ContactShadows.tsx"
      tier: 1
      accessed_at_utc: "2026-05-26T00:00:00Z"
  status: verified

- claim: "Props and defaults: opacity=1, scale=10, blur=1, far=10, resolution=256, color='#000000', frames=Infinity, smooth=true, depthWrite=false"
  sources:
    - url: "https://drei.docs.pmnd.rs/staging/contact-shadows"
      tier: 1
      accessed_at_utc: "2026-05-26T00:00:00Z"
    - url: "https://raw.githubusercontent.com/pmndrs/drei/master/src/core/ContactShadows.tsx"
      tier: 1
      accessed_at_utc: "2026-05-26T00:00:00Z"
  status: verified

- claim: "ContactShadows is 'a rather expensive effect' per drei documentation"
  exact_quote: "a rather expensive effect"
  sources:
    - url: "https://drei.docs.pmnd.rs/staging/contact-shadows"
      tier: 1
      accessed_at_utc: "2026-05-26T00:00:00Z"
  status: verified
```
