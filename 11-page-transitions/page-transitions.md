# Page Transitions

Seamless page transitions are the difference between a 3D website that feels alive and one that flashes between routes. The core challenge: keeping a persistent WebGL context across page changes while content fades in and out.

## The Core Architecture Problem

Browsers destroy and recreate the DOM on full page loads. Three.js scenes live in the DOM. Naive page transitions kill the WebGL context and lose GPU state (textures, shaders, render targets) on every navigation.

The solution: keep the canvas outside the content container, navigate only the content.

```html
<body>
  <!-- Canvas lives OUTSIDE Barba container — never gets swapped -->
  <canvas id="webgl-canvas"></canvas>

  <!-- Barba swaps only the content inside this container -->
  <div id="barba-wrapper">
    <div data-barba="container" data-barba-namespace="home">
      <main class="page-content">...</main>
    </div>
  </div>
</body>
```

## Barba.js

Barba.js is a 7 KB library that turns multi-page sites into SPA-like experiences by intercepting link clicks, fetching the next page via fetch API, and swapping only the content container while calling animation hooks.

### Installation

```bash
npm install @barba/core @barba/prefetch
```

### Basic Setup with Three.js

```js
import barba from "@barba/core";
import { prefetch } from "@barba/prefetch";
import gsap from "gsap";

barba.use(prefetch); // prefetch on hover

// Persistent Three.js scene — created once
const renderer = new THREE.WebGLRenderer({ canvas: document.querySelector("#webgl-canvas"), antialias: true });
const scene    = new THREE.Scene();
const camera   = new THREE.PerspectiveCamera(45, innerWidth / innerHeight, 0.1, 100);
camera.position.z = 5;

function startLoop() {
  renderer.setAnimationLoop(() => renderer.render(scene, camera));
}

barba.init({
  preventRunning: true,

  transitions: [{
    name: "default",

    // Called once on initial page load
    async once({ next }) {
      await animateIn(next.container);
      startLoop();
    },

    // Called before the current page leaves
    async leave({ current }) {
      await gsap.to(current.container, { opacity: 0, duration: 0.4 });
      current.container.remove();
    },

    // Called after new page is inserted but before it enters
    async enter({ next }) {
      gsap.set(next.container, { opacity: 0 });
    },

    // Called after leave completes
    async after({ next }) {
      await gsap.to(next.container, { opacity: 1, duration: 0.4 });
    },
  }],

  views: [
    {
      namespace: "home",
      afterEnter() {
        // Initialize page-specific Three.js objects
        addParticlesForHomePage();
      },
      beforeLeave() {
        // Remove page-specific objects, dispose resources
        removeParticlesForHomePage();
      },
    },
    {
      namespace: "about",
      afterEnter() { addGeometryForAboutPage(); },
      beforeLeave() { removeGeometryForAboutPage(); },
    },
  ],
});
```

### Sync Mode (Simultaneous Transitions)

```js
transitions: [{
  sync: true,  // leave and enter hooks run at the same time

  async leave({ current }) {
    return gsap.to(current.container, { xPercent: -100, duration: 0.6 });
  },
  async enter({ next }) {
    gsap.set(next.container, { xPercent: 100 });
    return gsap.to(next.container, { xPercent: 0, duration: 0.6 });
  },
}]
```

### GSAP and Three.js Memory Cleanup

ScrollTrigger, SplitText, and Three.js resources do not auto-clean on Barba transitions:

```js
import { ScrollTrigger } from "gsap/ScrollTrigger";

barba.hooks.beforeLeave(() => {
  // Kill all ScrollTrigger instances from the current page
  ScrollTrigger.getAll().forEach(t => t.kill());

  // Dispose Three.js objects created per-page
  pageGeometries.forEach(g => g.dispose());
  pageMaterials.forEach(m => m.dispose());
  pageGeometries.length = 0;
  pageMaterials.length  = 0;
});
```

## View Transitions API

The View Transitions API is a native browser API (no library) that cross-fades between DOM states. Chrome 111+, Safari 18+.

### Basic Page Transition

```js
// Wrap any DOM mutation in startViewTransition
document.startViewTransition(() => {
  // Mutate the DOM here — browser captures before/after snapshots
  document.querySelector("main").innerHTML = newPageHTML;
});
```

### CSS Transition Control

```css
/* Default cross-fade — customize with ::view-transition-* pseudo-elements */
::view-transition-old(root) {
  animation: 400ms ease-out fade-and-scale-out;
}
::view-transition-new(root) {
  animation: 400ms ease-in fade-and-scale-in;
}

@keyframes fade-and-scale-out {
  from { opacity: 1; transform: scale(1); }
  to   { opacity: 0; transform: scale(0.95); }
}
@keyframes fade-and-scale-in {
  from { opacity: 0; transform: scale(1.05); }
  to   { opacity: 1; transform: scale(1); }
}

/* Per-element transitions (matched by view-transition-name) */
.hero-image {
  view-transition-name: hero;
}
::view-transition-old(hero),
::view-transition-new(hero) {
  animation-duration: 600ms;
  animation-timing-function: cubic-bezier(0.4, 0, 0.2, 1);
}
```

### View Transitions with Router (SPA)

```js
// Astro / React Router / Next.js — wrap navigation
async function navigate(url) {
  if (!document.startViewTransition) {
    // Fallback for unsupported browsers
    await loadPage(url);
    return;
  }

  await document.startViewTransition(async () => {
    await loadPage(url);
  }).ready;
}
```

### View Transitions + Three.js Canvas

The canvas element is typically excluded from view transitions (GPU content cannot be captured as a DOM snapshot). Keep it persistent and overlay the transition on top:

```css
#webgl-canvas {
  position: fixed;
  inset: 0;
  z-index: 0;
  /* Exclude canvas from view transition capture */
  view-transition-name: none;
}

.page-overlay {
  position: fixed;
  inset: 0;
  z-index: 1;
  pointer-events: none;
  view-transition-name: page-overlay;
}
```

## Three.js Scene Switching

When different pages have fundamentally different 3D scenes, dispose and rebuild cleanly:

```js
class SceneManager {
  constructor(renderer, camera) {
    this.renderer = renderer;
    this.camera   = camera;
    this.active   = null;
  }

  async switchTo(SceneClass, options = {}) {
    // 1. Animate out
    if (this.active) {
      await this.active.out();
      this.active.dispose();
    }

    // 2. Build new scene
    this.active = new SceneClass(this.renderer, this.camera);
    await this.active.init();

    // 3. Animate in
    await this.active.in();
  }
}

// Scene contract
class HomeScene {
  constructor(renderer, camera) {
    this.renderer = renderer;
    this.camera   = camera;
    this.scene    = new THREE.Scene();
    this.objects  = [];
  }

  async init() {
    const mesh = new THREE.Mesh(
      new THREE.IcosahedronGeometry(1),
      new THREE.MeshStandardMaterial({ color: 0xffffff })
    );
    this.scene.add(mesh);
    this.objects.push(mesh);
    this.renderer.setAnimationLoop(() => {
      this.renderer.render(this.scene, this.camera);
    });
  }

  async in() {
    return gsap.from(this.scene.position, { y: -2, duration: 0.8, ease: "power3.out" });
  }

  async out() {
    return gsap.to(this.scene.position, { y: 2, duration: 0.5, ease: "power3.in" });
  }

  dispose() {
    this.renderer.setAnimationLoop(null);
    this.objects.forEach(obj => {
      obj.geometry.dispose();
      obj.material.dispose();
      this.scene.remove(obj);
    });
    this.renderer.clear();
  }
}
```

## Performance Implications

- **Texture re-upload cost**: switching scenes that need different textures causes GPU texture uploads. Preload textures before the transition starts.
- **Context loss risk**: rapid transitions can trigger WebGL context loss on mobile. Add a `contextlost` event listener and re-init if needed.
- **Memory leaks**: failing to dispose geometry/material/texture on scene exit is the most common 3D site bug. Use `renderer.info.memory` to monitor object counts during development.
- **View Transitions + canvas**: the browser cannot capture WebGL canvas pixels for view transition snapshots. Design the transition to animate DOM overlays, not the canvas itself.
- **GSAP + Barba**: always kill GSAP tweens and ScrollTrigger instances in `beforeLeave`. Leaked animations continue running and fight the new page's animations.

## Tools and Libraries

| Tool | Purpose |
|---|---|
| [Barba.js](https://barba.js.org) | SPA-like page transitions |
| [@barba/prefetch](https://barba.js.org/docs/plugins/prefetch/) | Link hover prefetching |
| [GSAP](https://gsap.com) | Animation for transition hooks |
| [View Transitions MDN](https://developer.mozilla.org/en-US/docs/Web/API/View_Transitions_API) | Native API reference |
| [Codrops Barba + Three.js tutorial](https://tympanus.net/codrops/2026/03/18/building-seamless-3d-transitions-with-webflow-gsap-and-three-js/) | 2026 practical walkthrough |
