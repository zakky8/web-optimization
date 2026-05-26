# Scroll and Animation Architecture

## Lenis - The Industry Standard Scroll Library (3KB)

Replaced Locomotive Scroll. Locomotive v5 is now a thin wrapper around Lenis.

Why Lenis over native scroll:
- Normalizes input: trackpad, mouse wheel, touch all behave identically
- Fixes de-sync between browser scroll position and WebGL frame
- Built on scrollTo, not CSS transforms (avoids transform pitfalls)
- 3KB gzipped - negligible cost

```javascript
import Lenis from 'lenis'

const lenis = new Lenis({
  lerp: 0.1,           // interpolation factor (lower = smoother)
  smoothWheel: true,
  touchMultiplier: 2,
})

// CRITICAL: sync Lenis with Three.js rAF loop
// They must run in the same requestAnimationFrame tick
function raf(time) {
  lenis.raf(time)               // update scroll position
  renderer.render(scene, camera) // render frame
  requestAnimationFrame(raf)
}
requestAnimationFrame(raf)

// If using GSAP ticker instead:
gsap.ticker.add((time) => {
  lenis.raf(time * 1000)
})
gsap.ticker.lagSmoothing(0)
```

## GSAP ScrollTrigger

Industry standard for scroll-driven animation. Syncs with rAF automatically.

```javascript
import gsap from 'gsap'
import { ScrollTrigger } from 'gsap/ScrollTrigger'
gsap.registerPlugin(ScrollTrigger)

// Basic scroll-driven animation
gsap.to(mesh.position, {
  x: 5,
  scrollTrigger: {
    trigger: '#section',
    start: 'top center',
    end: 'bottom center',
    scrub: true,  // maps scroll 0-100% to animation 0-100%
  }
})

// Pin a section during scroll
ScrollTrigger.create({
  trigger: '#pinned',
  start: 'top top',
  end: '+=500',
  pin: true,
  scrub: 1
})

// Refresh after all triggers initialized
ScrollTrigger.refresh()

// Mobile: disable heavy animations
const mm = gsap.matchMedia()
mm.add('(max-width: 768px)', () => {
  // simplified mobile animations here
})
```

## GSAP quickTo - Mouse Tracking Without GC Pressure

For continuous mouse-tracking animations (runs every frame).

```javascript
// BAD - creates new tween object every mousemove event
window.addEventListener('mousemove', (e) => {
  gsap.to(mesh.position, { x: e.clientX * 0.01, duration: 0.4 })
})
// Creates hundreds of objects per second -> GC spikes -> frame drops

// GOOD - reusable animation function, zero allocation per call
const moveX = gsap.quickTo(mesh.position, 'x', { duration: 0.4, ease: 'power2' })
const moveY = gsap.quickTo(mesh.position, 'y', { duration: 0.4, ease: 'power2' })

window.addEventListener('mousemove', (e) => {
  moveX(e.clientX * 0.01)
  moveY(e.clientY * 0.01)
})
// Zero new object creation per call
```

Stas Bondar '25: quickTo is the key performance technique for his continuous animations.

## CSS Scroll-Driven Animations (Native Browser)

Runs on compositor thread - completely immune to main thread JS jank.

```css
/* Progress bar */
@keyframes progress {
  from { transform: scaleX(0); }
  to   { transform: scaleX(1); }
}
.progress-bar {
  animation: progress linear both;
  animation-timeline: scroll();
}

/* Reveal on enter */
@keyframes reveal {
  from { opacity: 0; transform: translateY(20px); }
  to   { opacity: 1; transform: translateY(0); }
}
.element {
  animation: reveal linear both;
  animation-timeline: scroll();
  animation-range: entry 0% entry 40%;
}

/* Horizontal scroll progress linked to element */
.card {
  animation: fadeIn linear both;
  animation-timeline: view();
  animation-range: entry 20% cover 60%;
}
```

Use CSS scroll animations for:
- Progress bars
- Simple reveal effects
- Basic parallax
- Sticky element transitions

Use GSAP when:
- Complex multi-step sequencing needed
- WebGL sync required
- Physics-based easing
- Cross-browser consistency critical (CSS still has gaps)

## Velocity-Based Scroll Distortion (Stas Bondar Technique)

```javascript
let scrollY = 0
let scrollVelocity = 0
let lastScrollY = 0

function updateScroll() {
  scrollVelocity = scrollY - lastScrollY
  lastScrollY = scrollY

  // Pass velocity to shader as distortion amount
  material.uniforms.uDistortion.value = scrollVelocity * 0.01
}

// Use Lenis onScroll callback
lenis.on('scroll', ({ scroll, velocity }) => {
  material.uniforms.uDistortion.value = velocity * 0.005
})
```
