# Three.js 3D Text Rendering

**Folder:** `33-typography-kinetic`
**Last updated:** 2026-05-26

Sources verified:
- troika-three-text: https://github.com/protectwise/troika/tree/main/packages/troika-three-text — accessed 2026-05-26
- drei Text component: http://drei.docs.pmnd.rs/abstractions/text — accessed 2026-05-26

---

## TextGeometry (Legacy Three.js Approach)

`TextGeometry` was the original Three.js solution. It extrudes text into a 3D mesh using a JSON-format font.

```js
import * as THREE from 'three';
import { FontLoader } from 'three/examples/jsm/loaders/FontLoader.js';
import { TextGeometry } from 'three/examples/jsm/geometries/TextGeometry.js';

const loader = new FontLoader();
loader.load('/fonts/helvetiker_regular.typeface.json', (font) => {
  const geometry = new TextGeometry('Hello', {
    font: font,
    size: 0.5,
    height: 0.1,        // extrusion depth
    curveSegments: 12,
    bevelEnabled: true,
    bevelThickness: 0.02,
    bevelSize: 0.01,
    bevelOffset: 0,
    bevelSegments: 5,
  });

  geometry.computeBoundingBox();
  geometry.center();

  const material = new THREE.MeshStandardMaterial({ color: 0xffffff });
  const mesh = new THREE.Mesh(geometry, material);
  scene.add(mesh);
});
```

### Font format for TextGeometry

Three.js uses a custom JSON "typeface" format. Convert a TTF/OTF to this format using the Facetype.js tool: https://gero3.github.io/facetype.js/

### Why TextGeometry is considered deprecated

| Problem                         | Detail                                                             |
|---------------------------------|--------------------------------------------------------------------|
| Rasterizes at geometry level    | Curved letters become blocky at small sizes or far distances       |
| No runtime text changes          | Changing the string requires rebuilding the entire geometry        |
| No multiline or wrapping        | Manual line-break calculations required                            |
| Font format is non-standard      | Requires conversion from TTF; subset is painful                   |
| Polygon-heavy                   | Each character is dozens of triangles; expensive for long strings  |

`TextGeometry` is still useful for extruded 3D title effects where you want physical depth and bevels. For flat readable text at any size, use troika-three-text.

---

## troika-three-text

troika-three-text is a standalone package from the Troika monorepo. It renders text via **signed distance fields (SDF)** — a technique where each glyph is encoded as a distance map rather than a pixel bitmap, allowing crisp rendering at any scale.

GitHub: https://github.com/protectwise/troika (monorepo)
Package: `troika-three-text`

```bash
npm install troika-three-text
```

### How SDF works (brief)

1. The font file (TTF, OTF, or WOFF — not WOFF2) is parsed at runtime.
2. For each glyph used, an SDF texture atlas entry is generated. This runs in a **Web Worker** to avoid main-thread blocking.
3. The Three.js material is patched with GLSL shader code that reads the SDF texture and reconstructs sharp edges at any zoom level.
4. GPU acceleration is used for SDF generation where available (via `OffscreenCanvas` + WebGL worker).

Key implication: text quality does not degrade when zoomed in. This is the fundamental advantage over bitmap-based approaches.

### Basic usage

```js
import { Text } from 'troika-three-text';

const myText = new Text();
scene.add(myText);

myText.text = 'Hello World';
myText.font = '/fonts/inter-var.ttf';
myText.fontSize = 0.2;
myText.color = 0xffffff;
myText.anchorX = 'center';
myText.anchorY = 'middle';

// Must call sync() after changing properties
myText.sync();
```

### Key properties

| Property          | Type                        | Default                    | Description                                       |
|-------------------|-----------------------------|----------------------------|---------------------------------------------------|
| `text`            | string                      | `''`                       | The text content                                  |
| `font`            | string (URL)                | Roboto from Google Fonts   | Path to TTF, OTF, or WOFF file                   |
| `fontSize`        | number                      | `0.1`                      | Em-height in local world units                    |
| `color`           | THREE.Color / hex / string  | inherited from material    | Shortcut for text color                           |
| `anchorX`         | number / '%' / keyword      | `0`                        | `'left'`, `'center'`, `'right'`, or numeric       |
| `anchorY`         | number / '%' / keyword      | `0`                        | `'top'`, `'middle'`, `'bottom'`, `'baseline'`     |
| `textAlign`       | string                      | `'left'`                   | `'left'`, `'right'`, `'center'`, `'justify'`      |
| `maxWidth`        | number                      | `Infinity`                 | Word-wrap threshold in world units                |
| `lineHeight`      | number / `'normal'`         | `'normal'`                 | Multiplier of fontSize                            |
| `letterSpacing`   | number                      | `0`                        | Em units added between characters                 |
| `whiteSpace`      | string                      | `'normal'`                 | `'normal'` or `'nowrap'`                          |
| `outlineWidth`    | number / '%'                | `0`                        | Halo/outline thickness around glyphs              |
| `outlineColor`    | color                       | `0x000000`                 | Outline color                                     |
| `outlineBlur`     | number / '%'                | `0`                        | Soften outline edges                              |
| `outlineOffsetX`  | number / '%'                | `0`                        | Horizontal shadow offset                          |
| `outlineOffsetY`  | number / '%'                | `0`                        | Vertical shadow offset                            |
| `strokeWidth`     | number / '%'                | `0`                        | Stroke drawn inside glyph edges                   |
| `strokeColor`     | color                       | `0x808080`                 | Stroke color                                      |
| `clipRect`        | `[minX, minY, maxX, maxY]`  | `null`                     | Clip text to a rectangle in local space           |
| `glyphGeometryDetail` | number              | `1`                        | Mesh subdivision for curved geometry              |

### sync() and the callback

Property changes do not take effect until `sync()` is called. This is by design — batch property changes then sync once.

```js
myText.text = 'Updated text';
myText.fontSize = 0.3;
myText.sync(() => {
  console.log('Text geometry ready, textRenderInfo available');
  console.log(myText.textRenderInfo); // bounding boxes, glyph positions, etc.
});
```

### WOFF2 not supported

troika-three-text parses font files using `opentype.js`, which does not support WOFF2 (Brotli-compressed). Use TTF, OTF, or WOFF (zlib-compressed) for the font URL.

### textRenderInfo — per-glyph data

After `sync()`, `myText.textRenderInfo` exposes:

```
{
  glyphBounds: Float32Array,  // [minX, minY, maxX, maxY] per glyph
  glyphAtlasIndices: Float32Array,
  caretPositions: Float32Array, // cursor positions between chars
  blockBounds: [minX, minY, maxX, maxY], // total text bounds
  visibleBounds: [...],
  chunkedBounds: [...],
  parameters: { ... }
}
```

This enables per-character hitboxes, cursor positioning, and custom geometry overlays.

---

## drei Text Component (React Three Fiber)

drei wraps troika-three-text into a declarative React Three Fiber component.

Source: http://drei.docs.pmnd.rs/abstractions/text — accessed 2026-05-26

```bash
npm install @react-three/drei troika-three-text
```

```jsx
import { Text } from '@react-three/drei';

function Scene() {
  return (
    <Text
      font="/fonts/inter-var.ttf"
      fontSize={0.5}
      color="white"
      anchorX="center"
      anchorY="middle"
    >
      Hello World
    </Text>
  );
}
```

### All troika props are valid

The drei `Text` component passes all props through to troika-three-text directly. The documented statement is: "All of troika's props are valid!"

### Suspense integration

The component suspends while loading font data. Wrap it in `<Suspense>` to show a fallback:

```jsx
import { Suspense } from 'react';
import { Text } from '@react-three/drei';

function Scene() {
  return (
    <Suspense fallback={null}>
      <Text font="/fonts/inter-var.ttf">Loading...</Text>
    </Suspense>
  );
}
```

### Prespecifying characters to avoid FOUC

Pass the `characters` prop with all characters the text might use. This pre-warms the SDF atlas before the text content is needed.

```jsx
<Text
  font="/fonts/inter-var.ttf"
  characters="abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ0123456789!.,?"
>
  Dynamic content here
</Text>
```

### Animating with useFrame + useRef

```jsx
import { useRef } from 'react';
import { useFrame } from '@react-three/fiber';
import { Text } from '@react-three/drei';

function AnimatedText() {
  const textRef = useRef();

  useFrame(({ clock }) => {
    if (textRef.current) {
      textRef.current.fontSize = 0.3 + Math.sin(clock.elapsedTime) * 0.1;
    }
  });

  return (
    <Text ref={textRef} font="/fonts/inter.ttf">
      Breathing text
    </Text>
  );
}
```

---

## Performance Comparison

| Approach             | Quality at scale | Dynamic text | Memory      | Render cost       | Notes                              |
|----------------------|------------------|--------------|-------------|-------------------|------------------------------------|
| TextGeometry (Three) | Low (polygon)    | No (rebuild) | High (mesh) | High (many tris)  | Good for extruded 3D titles only   |
| troika-three-text    | High (SDF)       | Yes          | Medium (tex)| Low (quad + SDF)  | Web Worker; no WOFF2               |
| drei `<Text>`        | High (SDF)       | Yes          | Medium      | Low               | Suspense-native; RTF only          |
| Canvas texture       | Medium (bitmap)  | Yes          | Medium      | Medium            | Blurs when zoomed; full CSS layout |
| CSS3DObject          | High (DOM)       | Yes          | Low         | DOM reflow cost   | No depth intersection with WebGL   |

For flat, readable text in a Three.js scene: use troika-three-text or drei `<Text>`.
For physical extruded titles: use TextGeometry with the understanding of its limitations.

---

## References

- troika-three-text GitHub: https://github.com/protectwise/troika/tree/main/packages/troika-three-text — accessed 2026-05-26
- drei Text docs: http://drei.docs.pmnd.rs/abstractions/text — accessed 2026-05-26
- drei GitHub: https://github.com/pmndrs/drei
- Three.js TextGeometry docs: https://threejs.org/docs/#examples/en/geometries/TextGeometry
- Facetype.js font converter: https://gero3.github.io/facetype.js/
- opentype.js (used by troika): https://opentype.js.org/
