# Animation Formats

Not all web animations belong in a 3D scene. Understanding which animation format fits which use case — and the performance tradeoffs of each — prevents over-engineering and the performance problems that come with it.

## Format Overview

| Format | Renderer | Interactivity | File size | GPU | Best for |
|---|---|---|---|---|---|
| CSS animations | Browser layout engine | None | Zero | No | Simple transitions, loaders |
| SVG animations (SMIL/JS) | Browser SVG engine | Limited | Small | No | Logos, icons, illustrations |
| Canvas 2D | CPU rasterizer | Full JS | Medium | No | Charts, 2D games, custom drawing |
| Lottie (.json / .lottie) | lottie-web (CPU or Canvas) | Playback control | Medium | Optional (WebGL/WebGPU) |
| Rive (.riv) | Custom WASM (WebGL/Metal) | State machine | Small | Yes | UI icons, interactive characters |
| Three.js / WebGL | GPU | Full JS | Largest | Yes | 3D, complex visual effects |
| WebGL sprites | GPU | Full JS | Small | Yes | 2D particle effects in 3D |

## Lottie

Lottie is Adobe After Effects animation exported as JSON (or the newer compressed `.lottie` binary). The lottie-web runtime plays it back frame by frame.

### Installation

```bash
npm install lottie-web
# Or the dotLottie player (smaller, faster)
npm install @lottiefiles/dotlottie-web
```

### Basic Playback

```js
import lottie from "lottie-web";

const anim = lottie.loadAnimation({
  container: document.querySelector("#lottie-container"),
  renderer:  "svg",       // "svg" | "canvas" | "html"
  loop:      true,
  autoplay:  true,
  path:      "/animations/hero.json", // or use `animationData` for inline JSON
});

// Control playback
anim.pause();
anim.play();
anim.setSpeed(0.5);
anim.goToAndStop(30, true); // go to frame 30, stop
anim.goToAndPlay(30, true); // go to frame 30, play

// Events
anim.addEventListener("complete",   () => console.log("done"));
anim.addEventListener("loopComplete", () => console.log("loop"));
anim.addEventListener("enterFrame",   (e) => console.log("frame", e.currentTime));

// Cleanup
anim.destroy();
```

### dotLottie Format

`.lottie` is the ZIP-compressed binary format. 40–70% smaller than `.json`.

```js
import { DotLottie } from "@lottiefiles/dotlottie-web";

const dotLottie = new DotLottie({
  canvas:    document.querySelector("canvas"),
  src:       "/animations/hero.lottie",
  loop:      true,
  autoplay:  true,
});
```

### Renderer Comparison

| Renderer | Rendering | Memory | Scale | Use case |
|---|---|---|---|---|
| `svg` (default) | CPU via browser SVG | Low | Vector (sharp at any size) | Logos, illustrations |
| `canvas` | CPU via Canvas 2D | Lower DOM overhead | Raster | Better perf with many animations |
| `html` | DOM elements | Highest | Mixed | Complex text/masking effects |

**Performance note:** The `canvas` renderer is significantly faster than `svg` when multiple animations play simultaneously. Avoid the `svg` renderer for more than 3 concurrent animations.

### Scroll-Driven Lottie

```js
import lottie from "lottie-web";

const anim = lottie.loadAnimation({
  container: document.querySelector("#scroll-anim"),
  renderer:  "canvas",
  loop:      false,
  autoplay:  false,
  path:      "/animations/scroll-story.json",
});

anim.addEventListener("DOMLoaded", () => {
  const totalFrames = anim.totalFrames;

  window.addEventListener("scroll", () => {
    const section  = document.querySelector("#scroll-section");
    const rect     = section.getBoundingClientRect();
    const progress = Math.max(0, Math.min(1,
      -rect.top / (rect.height - window.innerHeight)
    ));
    anim.goToAndStop(progress * totalFrames, true);
  });
});
```

## Rive

Rive is a design tool + runtime for interactive, state-machine-driven animations. Its web runtime uses WebGL and ships as a ~200 KB WASM binary.

### Installation

```bash
npm install @rive-app/canvas   # Canvas 2D renderer (smaller)
npm install @rive-app/webgl    # WebGL renderer (better perf for complex animations)
```

### Basic Playback

```js
import Rive from "@rive-app/canvas";

const riveInstance = new Rive({
  src:       "/animations/button.riv",
  canvas:    document.querySelector("canvas"),
  autoplay:  true,
  stateMachines: "ButtonStateMachine",

  onLoad: () => {
    console.log("Rive loaded");
    riveInstance.resizeDrawingSurfaceToCanvas();
  },
});

// State machine inputs
const inputs = riveInstance.stateMachineInputs("ButtonStateMachine");
const hovered = inputs.find(i => i.name === "isHovered");
const clicked = inputs.find(i => i.name === "isClicked");

canvas.addEventListener("pointerenter", () => { hovered.value = true; });
canvas.addEventListener("pointerleave", () => { hovered.value = false; });
canvas.addEventListener("pointerdown",  () => { clicked.fire(); });

// Cleanup
riveInstance.cleanup();
```

### Rive in React

```jsx
import { useRive, useStateMachineInput } from "@rive-app/react-canvas";

function AnimatedButton({ onClick }) {
  const { rive, RiveComponent } = useRive({
    src:           "/animations/button.riv",
    stateMachines: "ButtonStateMachine",
    autoplay:      true,
  });

  const isHovered = useStateMachineInput(rive, "ButtonStateMachine", "isHovered");
  const isClicked = useStateMachineInput(rive, "ButtonStateMachine", "isClicked");

  return (
    <RiveComponent
      style={{ width: 200, height: 80 }}
      onPointerEnter={() => isHovered && (isHovered.value = true)}
      onPointerLeave={() => isHovered && (isHovered.value = false)}
      onClick={() => { isClicked?.fire(); onClick?.(); }}
    />
  );
}
```

### Rive vs Lottie Decision Guide

**Choose Rive when:**
- The animation is interactive (hover, click, drag triggers)
- You need state machines (idle → hover → pressed → loading → success)
- GPU-accelerated rendering matters (complex vector characters)
- Designers work in the Rive editor

**Choose Lottie when:**
- Animation is non-interactive (plays once, loops, scroll-driven)
- Content comes from After Effects (motion designers use AE)
- You need maximum format portability
- Bundle size is critical and the `@rive-app/canvas` 200 KB is too large

## Canvas 2D

The HTML Canvas 2D API is a CPU-based immediate mode drawing API. It is not GPU-accelerated beyond basic compositing, but it is universally supported, has zero runtime overhead, and integrates cleanly with the DOM.

### Animation Loop

```js
const canvas  = document.querySelector("canvas");
const ctx     = canvas.getContext("2d");

// High-DPI support
function setupCanvas() {
  const dpr = window.devicePixelRatio || 1;
  const rect = canvas.getBoundingClientRect();
  canvas.width  = rect.width  * dpr;
  canvas.height = rect.height * dpr;
  ctx.scale(dpr, dpr);
}
setupCanvas();

function drawFrame(time) {
  const t = time * 0.001; // seconds

  ctx.clearRect(0, 0, canvas.clientWidth, canvas.clientHeight);

  // Animated circle
  ctx.beginPath();
  ctx.arc(
    canvas.clientWidth  / 2 + Math.sin(t) * 50,
    canvas.clientHeight / 2 + Math.cos(t) * 50,
    30, 0, Math.PI * 2
  );
  ctx.fillStyle = `hsl(${t * 60 % 360}, 70%, 60%)`;
  ctx.fill();

  requestAnimationFrame(drawFrame);
}
requestAnimationFrame(drawFrame);
```

### Canvas 2D for Custom Charts

```js
function drawBarChart(data, labels) {
  const w = canvas.clientWidth;
  const h = canvas.clientHeight;
  const maxVal = Math.max(...data);
  const barW = (w / data.length) * 0.7;
  const gap  = (w / data.length) * 0.3;

  data.forEach((val, i) => {
    const x   = i * (barW + gap) + gap / 2;
    const barH = (val / maxVal) * (h - 40);
    const y   = h - barH - 20;

    // Animate with lerp
    ctx.fillStyle = "#4f88ff";
    ctx.fillRect(x, y, barW, barH);

    ctx.fillStyle = "#fff";
    ctx.font = "12px sans-serif";
    ctx.fillText(labels[i], x, h - 5);
  });
}
```

## SVG Animation

SVG animations use CSS animations, SMIL (deprecated in Chrome but alive in others), or JavaScript manipulation of SVG attributes.

### CSS-Driven SVG Animation

```html
<svg viewBox="0 0 100 100" xmlns="http://www.w3.org/2000/svg">
  <circle class="loading-ring" cx="50" cy="50" r="40"
    fill="none" stroke="#4f88ff" stroke-width="8"
    stroke-dasharray="251.2" stroke-dashoffset="0"
    stroke-linecap="round"
  />
</svg>
```

```css
.loading-ring {
  transform-origin: center;
  animation: spin 1.2s linear infinite, dash 1.5s ease-in-out infinite;
}

@keyframes spin {
  to { transform: rotate(360deg); }
}

@keyframes dash {
  0%   { stroke-dashoffset: 251.2; }
  50%  { stroke-dashoffset: 62.8; transform: rotate(135deg); }
  100% { stroke-dashoffset: 251.2; transform: rotate(450deg); }
}
```

### JavaScript SVG Morphing

```js
// GreenSock MorphSVG (GSAP plugin)
import { MorphSVGPlugin } from "gsap/MorphSVGPlugin";
gsap.registerPlugin(MorphSVGPlugin);

gsap.to("#play-icon", {
  duration: 0.4,
  morphSVG: "#pause-icon",
  ease: "power2.inOut",
});
```

## Performance Comparison

### Runtime Cost per Frame

| Format | CPU/frame | GPU/frame | Notes |
|---|---|---|---|
| CSS animation | Near zero | Near zero | Compositor thread, no JS |
| SVG animation (CSS) | Low | Near zero | Layout recalc if animating geometry |
| Canvas 2D (simple) | Low (1-2ms) | None | All raster, CPU-bound |
| Lottie SVG renderer | Medium (3-10ms) | None | DOM mutations per frame |
| Lottie canvas renderer | Low (1-3ms) | None | Better than SVG renderer |
| Rive (canvas) | Low WASM | None | WASM overhead at start |
| Rive (WebGL) | Minimal | Low | GPU-accelerated vector |
| Three.js | Low JS | High GPU | Most capable, most overhead |

### When Performance Matters

```js
// Detect device capability before choosing renderer
const isHighEnd = navigator.hardwareConcurrency >= 4
  && !navigator.connection?.saveData
  && window.devicePixelRatio >= 2;

if (isHighEnd) {
  loadRiveWebGL(); // Full GPU animation
} else {
  loadLottieCanvas(); // CPU fallback
}
```

### Pausing Off-Screen Animations

```js
// Pause Lottie when not visible
const io = new IntersectionObserver((entries) => {
  entries.forEach(e => {
    if (e.isIntersecting) anim.play();
    else                  anim.pause();
  });
});
io.observe(document.querySelector("#lottie-container"));

// Rive pause/play
const ioRive = new IntersectionObserver((entries) => {
  entries.forEach(e => {
    if (e.isIntersecting) riveInstance.play();
    else                  riveInstance.pause();
  });
});
ioRive.observe(canvas);
```

## Tools and Libraries

| Tool | Purpose |
|---|---|
| [lottie-web](https://github.com/airbnb/lottie-web) | Lottie JSON playback |
| [LottieFiles](https://lottiefiles.com) | Lottie animation marketplace |
| [@lottiefiles/dotlottie-web](https://github.com/LottieFiles/dotlottie-web) | dotLottie format player |
| [Rive](https://rive.app) | Interactive animation runtime and editor |
| [@rive-app/react-canvas](https://github.com/rive-app/rive-react) | Rive React integration |
| [GSAP](https://gsap.com) | High-performance JS animation |
| [GreenSock MorphSVG](https://gsap.com/docs/v3/Plugins/MorphSVGPlugin/) | SVG path morphing |
| [SVGator](https://www.svgator.com) | Visual SVG animation editor |
