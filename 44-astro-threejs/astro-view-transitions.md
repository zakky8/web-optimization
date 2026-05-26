# Astro View Transitions + Three.js Canvas

> Verified against: https://docs.astro.build/en/guides/view-transitions/  
> Astro version: 6.x (current as of 2026-05-26)  
> Sources accessed: 2026-05-26

---

## Critical API Change (Astro 5 → 6)

| Astro version | Component to use | Import path |
|---|---|---|
| < 5.0 | `<ViewTransitions />` | `astro:transitions` |
| 5.0 – 5.x | `<ClientRouter />` (renamed) | `astro:transitions` |
| 6.0+ | `<ClientRouter />` | `astro:transitions` |

`<ViewTransitions />` was renamed to `<ClientRouter />` in Astro 5 to clarify its role in client-side routing (not just CSS transitions). The old name was completely removed in Astro 6. **Using `<ViewTransitions />` in Astro 6 will error at build time.**

---

## 1. Enabling the ClientRouter

Add `<ClientRouter />` once to your shared layout. It turns the MPA into a client-side router with animated page transitions.

```astro
---
// src/layouts/BaseLayout.astro
import { ClientRouter } from 'astro:transitions';
---
<html>
  <head>
    <meta charset="utf-8" />
    <title><slot name="title" /></title>
    <ClientRouter />
  </head>
  <body>
    <slot />
  </body>
</html>
```

With no further configuration, navigation between pages animates with a default crossfade. Astro intercepts `<a>` clicks and `<form>` submissions, fetches the new page, and swaps the DOM.

---

## 2. The Problem: Three.js Canvas Does Not Survive Page Swaps

Without `transition:persist`, every navigation destroys the old `<canvas>` element and creates a new one. This means:

- The WebGL context is torn down and rebuilt
- Any preloaded assets (textures, geometries) are lost
- The `requestAnimationFrame` loop is interrupted and must be restarted
- The user sees a visible flash or reload of the 3D scene

---

## 3. `transition:persist` — Keep the Canvas Alive

`transition:persist` tells `<ClientRouter />` to **transplant** the element from the old page into the new page's DOM rather than replacing it.

```astro
---
// src/components/PersistentCanvas.astro
---
<canvas
  id="main-canvas"
  transition:persist
  style="position:fixed;top:0;left:0;width:100%;height:100%;z-index:-1;"
></canvas>
```

With a named persist key for disambiguation:

```astro
<canvas id="main-canvas" transition:persist="webgl-canvas"></canvas>
```

The string argument becomes the transition name. Astro matches the element on the old page and the new page by this name and moves the DOM node rather than deleting and recreating it.

**Using `transition:persist` with a React/R3F island:**

```astro
<Scene client:only="react" transition:persist="r3f-canvas" />
```

Add `transition:persist-props` to prevent the island from re-rendering with new page's props during the transition:

```astro
<Scene client:only="react" transition:persist="r3f-canvas" transition:persist-props />
```

---

## 4. Background Canvas Pattern

A common 3D pattern is a full-screen WebGL background that persists across all pages:

```astro
---
// src/layouts/BaseLayout.astro
import { ClientRouter } from 'astro:transitions';
import BackgroundCanvas from '../components/BackgroundCanvas.astro';
---
<html>
  <head>
    <ClientRouter />
  </head>
  <body>
    <!-- Canvas is outside the page slot so it is present on all pages -->
    <BackgroundCanvas />

    <!-- Page content overlaid on top -->
    <main>
      <slot />
    </main>
  </body>
</html>
```

```astro
---
// src/components/BackgroundCanvas.astro
---
<canvas
  id="bg-canvas"
  transition:persist="bg-canvas"
  style="
    position: fixed;
    top: 0; left: 0;
    width: 100%; height: 100%;
    z-index: -1;
    pointer-events: none;
  "
></canvas>

<script>
  import * as THREE from 'three';

  const canvas = document.getElementById('bg-canvas') as HTMLCanvasElement;

  // Guard: only initialize once (transition:persist keeps the canvas alive,
  // but the script may run again if the layout is re-parsed)
  if (!(canvas as any).__threeInitialized) {
    (canvas as any).__threeInitialized = true;

    const renderer = new THREE.WebGLRenderer({ canvas, alpha: true, antialias: true });
    renderer.setPixelRatio(Math.min(window.devicePixelRatio, 2));
    renderer.setSize(window.innerWidth, window.innerHeight);

    const scene = new THREE.Scene();
    const camera = new THREE.PerspectiveCamera(60, window.innerWidth / window.innerHeight, 0.1, 100);
    camera.position.z = 3;

    // ... scene setup ...

    function animate() {
      requestAnimationFrame(animate);
      renderer.render(scene, camera);
    }
    animate();

    window.addEventListener('resize', () => {
      camera.aspect = window.innerWidth / window.innerHeight;
      camera.updateProjectionMatrix();
      renderer.setSize(window.innerWidth, window.innerHeight);
    });
  }
</script>
```

---

## 5. Lifecycle Hooks

These events fire on `document` during every `<ClientRouter />` navigation. Use them to synchronize Three.js state with page changes.

### `astro:before-preparation`
Fires when navigation starts, before the new page is fetched. Use to show a loading indicator or pause animations.

### `astro:after-preparation`
Fires after the new page's content has been loaded but before the DOM swap. Remove loading indicators here.

### `astro:before-swap`
Fires just before the old DOM is replaced. The event has an `event.newDocument` property. Use to transfer state, theme, or context into the incoming document.

```astro
<script is:inline>
  document.addEventListener('astro:before-swap', (event) => {
    // Preserve dark-mode class across navigations
    event.newDocument.documentElement.classList.toggle(
      'dark',
      document.documentElement.classList.contains('dark')
    );
  });
</script>
```

For Three.js: this is the right moment to capture camera position or scene state before the swap:

```ts
document.addEventListener('astro:before-swap', () => {
  // Save camera state to sessionStorage
  sessionStorage.setItem('cam', JSON.stringify(camera.position.toArray()));
});
```

### `astro:after-swap`
Fires immediately after the DOM has been replaced, before scripts from the new page run. Use to restore scroll position or re-attach DOM-dependent Three.js elements.

```ts
document.addEventListener('astro:after-swap', () => {
  // Restore camera position
  const saved = sessionStorage.getItem('cam');
  if (saved) camera.position.fromArray(JSON.parse(saved));
});
```

### `astro:page-load`
Fires at the end of every navigation after all content is visible and scripts have loaded. This is the equivalent of `DOMContentLoaded` for client-side navigations.

```ts
document.addEventListener('astro:page-load', () => {
  // Re-initialize any Three.js elements that are page-specific
  // (i.e. NOT using transition:persist)
  initPageSpecificCanvas();
});
```

**Important:** Standard module `<script>` blocks in Astro only execute once per page load (they are deduped). If you need code to run on every navigation, wrap it in `astro:page-load`:

```astro
<script>
  // This runs only once even with ClientRouter navigations:
  // import './setup';

  // Do this instead:
  document.addEventListener('astro:page-load', () => {
    import('./setup').then(m => m.init());
  });
</script>
```

For inline scripts (`is:inline`), add `data-astro-rerun` to force re-execution on every navigation:

```astro
<script is:inline data-astro-rerun>
  // Runs on every ClientRouter navigation
  document.querySelector('.page-canvas')?.dispatchEvent(new Event('reinit'));
</script>
```

---

## 6. Page-Specific Canvas vs. Persistent Canvas

| Use case | Approach |
|---|---|
| Background / ambient scene shared across all pages | `transition:persist` in layout |
| Product viewer only on `/product` pages | No persist; initialize in `astro:page-load`, clean up in `astro:before-swap` |
| Scene that changes between pages but reuses assets | `transition:persist` + update scene on `astro:page-load` |

### Cleanup Pattern for Non-Persistent Canvases

```astro
<canvas id="page-canvas"></canvas>

<script>
  import * as THREE from 'three';

  let renderer: THREE.WebGLRenderer | null = null;
  let animId: number;

  function init() {
    const canvas = document.getElementById('page-canvas') as HTMLCanvasElement;
    if (!canvas) return;

    renderer = new THREE.WebGLRenderer({ canvas });
    // ... scene setup ...

    function loop() {
      animId = requestAnimationFrame(loop);
      renderer!.render(scene, camera);
    }
    loop();
  }

  function cleanup() {
    cancelAnimationFrame(animId);
    renderer?.dispose();
    renderer = null;
  }

  // Initialize on every page arrival
  document.addEventListener('astro:page-load', init);

  // Clean up before DOM swap to avoid WebGL context leaks
  document.addEventListener('astro:before-swap', cleanup);
</script>
```

---

## 7. Fallback Behavior

```astro
<ClientRouter fallback="swap" />
```

| Value | Behavior |
|---|---|
| `animate` (default) | Simulates view transitions with custom attributes in unsupported browsers |
| `swap` | Immediately replaces old page with new (no animation) |
| `none` | Full-page refresh, no JS navigation |

Three.js notes on fallback: with `fallback="none"`, the page reloads fully on every navigation, which re-initializes all WebGL contexts. If you rely on `transition:persist` to maintain renderer state, `fallback="none"` defeats that purpose.

---

## 8. Disabling ClientRouter Navigation for Specific Links

Force a full-page reload for links that navigate to pages with incompatible canvas state:

```astro
<a href="/other-page" data-astro-reload>Hard reload to this page</a>
```

---

## Key Points

- `<ViewTransitions />` is gone in Astro 6. Import `<ClientRouter />` from `astro:transitions`.
- `transition:persist` transplants DOM nodes (including the `<canvas>`) across navigations — the WebGL context survives.
- Module scripts run once; use `astro:page-load` for code that must run on every navigation.
- Always dispose `renderer`, cancel `requestAnimationFrame`, and clean up `EventListener` references in `astro:before-swap` for non-persistent canvases to prevent WebGL context leaks.
- Test with `fallback="swap"` to verify graceful degradation on browsers that do not support the native View Transitions API.
