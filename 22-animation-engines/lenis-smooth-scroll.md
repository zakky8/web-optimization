# Lenis Smooth Scroll

Lenis provides buttery-smooth scrolling with easing + scroll-linked animations.
Repo: https://github.com/darkroomengineering/lenis
Version: v1.3.23 (2026-04-15) — Active

## Installation

```bash
npm install lenis
```

## Basic Setup

```js
import Lenis from 'lenis';

const lenis = new Lenis({
  // Core options
  duration: 1.2,           // Scroll animation duration (seconds)
  easing: (t) => Math.min(1, 1.001 - Math.pow(2, -10 * t)),
  orientation: 'vertical', // 'vertical' | 'horizontal'
  gestureOrientation: 'vertical',
  smoothWheel: true,       // Smooth mouse wheel
  wheelMultiplier: 1,      // Wheel sensitivity
  touchMultiplier: 2,      // Touch sensitivity
  infinite: false,         // Infinite scroll
});

// Connect to RAF (required)
function raf(time) {
  lenis.raf(time);
  requestAnimationFrame(raf);
}
requestAnimationFrame(raf);

// OR integrate with GSAP ticker (preferred with GSAP)
gsap.ticker.add((time) => {
  lenis.raf(time * 1000);  // GSAP time is in seconds, Lenis expects ms
});
gsap.ticker.lagSmoothing(0);  // Prevent catching up after tab switch
```

## GSAP ScrollTrigger Integration

```js
import Lenis from 'lenis';
import gsap from 'gsap';
import { ScrollTrigger } from 'gsap/ScrollTrigger';

const lenis = new Lenis();

// Sync Lenis scroll position to ScrollTrigger
lenis.on('scroll', ScrollTrigger.update);

// Use GSAP ticker for RAF
gsap.ticker.add((time) => {
  lenis.raf(time * 1000);
});
gsap.ticker.lagSmoothing(0);
```

## React Setup

```jsx
import Lenis from 'lenis';
import { useEffect, useRef } from 'react';

function App() {
  useEffect(() => {
    const lenis = new Lenis();

    function raf(time) {
      lenis.raf(time);
      requestAnimationFrame(raf);
    }
    requestAnimationFrame(raf);

    return () => lenis.destroy();
  }, []);

  return <main>{/* content */}</main>;
}
```

## Scroll Events

```js
// Listen to scroll updates
lenis.on('scroll', ({ scroll, limit, velocity, direction, progress }) => {
  // scroll: current scroll position (px)
  // limit: max scroll position
  // velocity: scroll speed
  // direction: 1 (down) | -1 (up)
  // progress: 0-1

  // Sync Three.js camera
  camera.position.y = scroll * 0.01;
});

// Programmatic scroll
lenis.scrollTo('#section-3', {
  offset: -100,          // px offset from target
  duration: 2,
  easing: (t) => t < 0.5 ? 2 * t * t : -1 + (4 - 2 * t) * t,
  immediate: false,      // true = instant jump
});

lenis.scrollTo('top');    // Scroll to top
lenis.scrollTo('bottom'); // Scroll to bottom
lenis.scrollTo(500);      // Scroll to px position
```

## Stop/Start

```js
// Pause scroll (e.g., modal open)
lenis.stop();

// Resume
lenis.start();

// Destroy completely
lenis.destroy();
```

## Sources
- Lenis GitHub: https://github.com/darkroomengineering/lenis
- Lenis docs: https://lenis.darkroom.engineering/
- v1.3.23 release: 2026-04-15 (verified GitHub)
