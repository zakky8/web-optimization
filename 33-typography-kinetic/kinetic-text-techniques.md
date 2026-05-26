# Kinetic Text Techniques

**Folder:** `33-typography-kinetic`
**Last updated:** 2026-05-26

---

## 1. Text on Path (SVG textPath)

SVG `<textPath>` renders text that follows an arbitrary path element.

### Basic usage

```html
<svg viewBox="0 0 500 200" xmlns="http://www.w3.org/2000/svg">
  <!-- Define the path (can be invisible) -->
  <defs>
    <path
      id="curve"
      d="M 50,150 Q 150,50 250,150 T 450,150"
    />
  </defs>

  <!-- Optionally show the path -->
  <use href="#curve" fill="none" stroke="#eee" stroke-width="1" />

  <!-- Text following the path -->
  <text font-size="18" fill="currentColor">
    <textPath href="#curve" startOffset="0%">
      Text flowing along a curve
    </textPath>
  </text>
</svg>
```

### Animating startOffset (circular text)

```html
<svg viewBox="0 0 200 200">
  <defs>
    <path id="circle-path" d="
      M 100,20
      a 80,80 0 1,1 -0.01,0
      Z
    "/>
  </defs>
  <text font-size="11" fill="white">
    <textPath href="#circle-path">
      <animate
        attributeName="startOffset"
        from="0%"
        to="100%"
        dur="8s"
        repeatCount="indefinite"
      />
      Circular kinetic text • Circular kinetic text •
    </textPath>
  </text>
</svg>
```

Or animate via CSS:

```css
textPath {
  animation: orbit 8s linear infinite;
}
@keyframes orbit {
  from { startOffset: 0%; }
  to   { startOffset: 100%; }
}
```

Note: CSS animation of SVG presentation attributes is inconsistently supported. Prefer SMIL `<animate>` or JavaScript for `startOffset`.

### JavaScript-driven path animation

```js
const textPath = document.querySelector('textPath');
let offset = 0;

function animate() {
  offset = (offset + 0.05) % 100;
  textPath.setAttribute('startOffset', `${offset}%`);
  requestAnimationFrame(animate);
}
animate();
```

### Inline SVG with dynamic path from JavaScript

```js
// Generate a sine wave path
function sinePath(width, height, frequency, amplitude) {
  const points = [];
  for (let x = 0; x <= width; x += 5) {
    const y = height / 2 + Math.sin((x / width) * Math.PI * 2 * frequency) * amplitude;
    points.push(`${x},${y}`);
  }
  return `M ${points.join(' L ')}`;
}

document.getElementById('wave-path').setAttribute('d', sinePath(500, 100, 2, 30));
```

---

## 2. Scroll-Linked Character Reveals

### Pattern: Splitting.js + IntersectionObserver

```js
import Splitting from 'splitting';

// Split all headline elements
document.querySelectorAll('.scroll-headline').forEach(el => {
  el.setAttribute('aria-label', el.textContent);
  Splitting({ target: el, by: 'chars' });
});

const observer = new IntersectionObserver((entries) => {
  entries.forEach(entry => {
    if (entry.isIntersecting) {
      entry.target.classList.add('revealed');
      observer.unobserve(entry.target);
    }
  });
}, {
  threshold: 0.15,
  rootMargin: '0px 0px -50px 0px'
});

document.querySelectorAll('.scroll-headline').forEach(el => observer.observe(el));
```

```css
.scroll-headline .char {
  display: inline-block;
  opacity: 0;
  transform: translateY(0.6em) rotate(8deg);
  transition:
    opacity 0.5s ease,
    transform 0.5s cubic-bezier(0.34, 1.56, 0.64, 1);
  transition-delay: calc(var(--char-index) * 0.03s);
}

.scroll-headline.revealed .char {
  opacity: 1;
  transform: translateY(0) rotate(0deg);
}
```

### Pattern: GSAP ScrollTrigger per character

```js
import gsap from 'gsap';
import ScrollTrigger from 'gsap/ScrollTrigger';
import Splitting from 'splitting';

gsap.registerPlugin(ScrollTrigger);

const results = Splitting({ target: '.pinned-text', by: 'chars' });
const chars = results[0].chars;

// Each character reveals as it enters the viewport
chars.forEach((char, i) => {
  gsap.from(char, {
    opacity: 0,
    y: 40,
    rotationX: -80,
    transformOrigin: '50% 0%',
    duration: 0.6,
    ease: 'back.out(1.4)',
    scrollTrigger: {
      trigger: char,
      start: 'top 90%',
      toggleActions: 'play none none reverse',
    },
    delay: i * 0.02,
  });
});
```

### Pattern: Scroll progress mapped to char visibility

Map scroll progress (0–1) to reveal characters one by one based on `--char-index`.

```js
import Splitting from 'splitting';

const el = document.querySelector('.progress-text');
Splitting({ target: el, by: 'chars' });
const chars = el.querySelectorAll('.char');
const total = chars.length;

window.addEventListener('scroll', () => {
  const rect = el.getBoundingClientRect();
  const windowH = window.innerHeight;
  // Progress: 0 when el enters viewport top, 1 when el exits
  const progress = 1 - (rect.top / windowH);
  const revealCount = Math.round(progress * total);

  chars.forEach((char, i) => {
    char.classList.toggle('visible', i < revealCount);
  });
});
```

```css
.progress-text .char {
  display: inline-block;
  opacity: 0;
  transition: opacity 0.2s ease;
}
.progress-text .char.visible {
  opacity: 1;
}
```

---

## 3. Per-Character 3D Transforms

CSS `transform-style: preserve-3d` enables true 3D rotation on individual characters.

```css
.splitting .char {
  display: inline-block;
  transform-style: preserve-3d;
  backface-visibility: hidden;  /* prevent back-side flicker */
}

.scene {
  perspective: 800px;
}
```

### Flip-in animation (Y axis)

```css
.splitting .char {
  display: inline-block;
  animation: flip-in 0.6s cubic-bezier(0.36, 1.56, 0.64, 1) both;
  animation-delay: calc(var(--char-index) * 0.06s);
}

@keyframes flip-in {
  from {
    opacity: 0;
    transform: rotateY(-90deg) translateX(-0.5em);
  }
  to {
    opacity: 1;
    transform: rotateY(0deg) translateX(0);
  }
}
```

### Mouse-reactive per-character tilt (JavaScript)

```js
import Splitting from 'splitting';

const el = document.querySelector('.tilt-text');
Splitting({ target: el, by: 'chars' });

document.addEventListener('mousemove', (e) => {
  const cx = window.innerWidth / 2;
  const cy = window.innerHeight / 2;
  const dx = (e.clientX - cx) / cx;  // -1 to 1
  const dy = (e.clientY - cy) / cy;  // -1 to 1

  el.querySelectorAll('.char').forEach((char, i) => {
    const phase = i / el.querySelectorAll('.char').length;
    const rotX = dy * 25 * (1 - phase * 0.5);
    const rotY = dx * 30 * (0.5 + phase * 0.5);
    char.style.transform = `rotateX(${-rotX}deg) rotateY(${rotY}deg)`;
    char.style.transition = 'transform 0.15s ease-out';
  });
});
```

### GSAP per-character 3D scramble effect

```js
import gsap from 'gsap';
import { TextPlugin } from 'gsap/TextPlugin';
import Splitting from 'splitting';

// Manual character scramble using 3D transforms
const el = document.querySelector('.scramble');
const original = el.textContent;
Splitting({ target: el, by: 'chars' });
const chars = el.querySelectorAll('.char');

function scramble() {
  const tl = gsap.timeline();

  chars.forEach((char, i) => {
    tl.to(char, {
      rotationY: 360,
      duration: 0.5,
      ease: 'power2.inOut',
    }, i * 0.05);
  });
}

el.addEventListener('click', scramble);
```

---

## 4. Mixing GSAP SplitText with Three.js Font Rendering

GSAP's `SplitText` (Club GreenSock) and Splitting.js both operate on DOM elements. Three.js operates on WebGL meshes. Mixing them requires a bridge strategy.

### Strategy A: DOM overlay on canvas

Keep text in DOM (GSAP SplitText), position a transparent `<canvas>` behind or in front for Three.js background effects.

```html
<div class="scene-wrapper">
  <canvas id="bg-canvas"></canvas>        <!-- Three.js scene -->
  <div class="text-layer">                <!-- DOM text on top -->
    <h1 class="hero-text">Hello World</h1>
  </div>
</div>
```

```css
.scene-wrapper { position: relative; }
#bg-canvas {
  position: absolute;
  inset: 0;
  z-index: 0;
}
.text-layer {
  position: relative;
  z-index: 1;
  pointer-events: none;
}
```

Then animate the DOM text with GSAP SplitText normally, and animate the Three.js scene independently for background effects (particles, meshes, shaders).

### Strategy B: CSS3DRenderer + Three.js scene

`CSS3DRenderer` from Three.js projects DOM elements into 3D space alongside WebGL objects.

```js
import * as THREE from 'three';
import { CSS3DRenderer, CSS3DObject } from 'three/examples/jsm/renderers/CSS3DRenderer.js';

// WebGL renderer (for 3D objects)
const renderer = new THREE.WebGLRenderer({ alpha: true });
renderer.setSize(window.innerWidth, window.innerHeight);
document.body.appendChild(renderer.domElement);

// CSS3D renderer (for DOM elements in 3D space)
const cssRenderer = new CSS3DRenderer();
cssRenderer.setSize(window.innerWidth, window.innerHeight);
cssRenderer.domElement.style.position = 'absolute';
cssRenderer.domElement.style.top = '0';
document.body.appendChild(cssRenderer.domElement);

// Create a DOM element and wrap it
const div = document.createElement('div');
div.textContent = 'Hello 3D';
div.style.fontSize = '48px';

// GSAP SplitText on this div
gsap.from(div.querySelectorAll('.char'), { ... });

const cssObject = new CSS3DObject(div);
cssObject.position.set(0, 0, 0);
scene.add(cssObject);
```

Limitation: CSS3D objects do not occlude WebGL geometry and vice versa (z-fighting between the two renderers is not solvable in general).

### Strategy C: troika-three-text + per-glyph animation via textRenderInfo

Use troika-three-text's `textRenderInfo` to get per-glyph bounding boxes, then create individual meshes or instanced geometry driven by GSAP.

```js
import { Text } from 'troika-three-text';
import gsap from 'gsap';

const text = new Text();
text.text = 'Hello';
text.font = '/fonts/inter.ttf';
text.fontSize = 0.2;
scene.add(text);

text.sync(() => {
  const { glyphBounds, caretPositions } = text.textRenderInfo;
  const numGlyphs = glyphBounds.length / 4;

  // Create a plane mesh per glyph for individual animation
  for (let i = 0; i < numGlyphs; i++) {
    const minX = glyphBounds[i * 4];
    const minY = glyphBounds[i * 4 + 1];
    const maxX = glyphBounds[i * 4 + 2];
    const maxY = glyphBounds[i * 4 + 3];

    const w = maxX - minX;
    const h = maxY - minY;
    const geo = new THREE.PlaneGeometry(w, h);
    const mat = new THREE.MeshBasicMaterial({ color: 0xff0000, wireframe: true });
    const mesh = new THREE.Mesh(geo, mat);
    mesh.position.set(minX + w / 2, minY + h / 2, 0.01);
    scene.add(mesh);

    // Animate each glyph overlay independently
    gsap.from(mesh.position, {
      z: -2,
      duration: 0.6,
      delay: i * 0.08,
      ease: 'power3.out',
    });
  }
});
```

### Strategy D: Render text to canvas, use as Three.js texture

For full CSS layout control with Three.js material:

```js
function textToTexture(str, fontSize = 64, color = '#fff') {
  const canvas = document.createElement('canvas');
  canvas.width = 512;
  canvas.height = 128;
  const ctx = canvas.getContext('2d');
  ctx.clearRect(0, 0, canvas.width, canvas.height);
  ctx.font = `${fontSize}px Inter, sans-serif`;
  ctx.fillStyle = color;
  ctx.textAlign = 'center';
  ctx.textBaseline = 'middle';
  ctx.fillText(str, canvas.width / 2, canvas.height / 2);
  return new THREE.CanvasTexture(canvas);
}

const texture = textToTexture('Kinetic');
const mat = new THREE.MeshBasicMaterial({ map: texture, transparent: true });
const mesh = new THREE.Mesh(new THREE.PlaneGeometry(4, 1), mat);
scene.add(mesh);
```

Limitation: bitmap texture blurs when zoomed. Use troika-three-text (SDF) for sharp text at any distance.

---

## 5. Technique Comparison

| Technique                     | Best for                                   | Limitations                                       |
|-------------------------------|--------------------------------------------|---------------------------------------------------|
| SVG textPath                  | Curved text, logos, circular labels        | No depth; 2D only                                 |
| Splitting.js + CSS            | Scroll reveals, stagger, hover effects     | DOM only; 3D via CSS transform-style              |
| Splitting.js + GSAP           | Complex sequences, scrub, timeline control | DOM only; GSAP Club for SplitText                 |
| GSAP SplitText                | Production-grade text animation            | Paid (Club GreenSock)                             |
| troika-three-text + GSAP      | Sharp 3D text with per-glyph animation     | No WOFF2; font loading overhead                   |
| CSS3DRenderer                 | DOM text in 3D space                       | No depth occlusion against WebGL                  |
| Canvas texture                | Quick text on a mesh                       | Blurs at scale; requires canvas redraw on change  |
| TextGeometry                  | Extruded physical 3D letters               | Static; polygon-heavy; small sizes look rough     |

---

## References

- MDN SVG textPath: https://developer.mozilla.org/en-US/docs/Web/SVG/Element/textPath
- Splitting.js: https://splitting.js.org — accessed 2026-05-26
- troika-three-text: https://github.com/protectwise/troika/tree/main/packages/troika-three-text — accessed 2026-05-26
- drei Text: http://drei.docs.pmnd.rs/abstractions/text — accessed 2026-05-26
- GSAP ScrollTrigger: https://gsap.com/docs/v3/Plugins/ScrollTrigger/
- GSAP SplitText: https://gsap.com/docs/v3/Plugins/SplitText/ (Club GreenSock)
- Three.js CSS3DRenderer: https://threejs.org/docs/#examples/en/renderers/CSS3DRenderer
