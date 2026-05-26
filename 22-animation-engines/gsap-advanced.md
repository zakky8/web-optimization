# GSAP Advanced Patterns

GSAP (GreenSock Animation Platform) — https://gsap.com/
Version 3.x — current as of 2026-05-26

## Installation

```bash
npm install gsap
# Club plugins (paid): ScrollTrigger, MorphSVG, SplitText, DrawSVG, MotionPath
# ScrollTrigger is now FREE in v3.11+
```

## ScrollTrigger — Three.js Integration

```js
import gsap from 'gsap';
import { ScrollTrigger } from 'gsap/ScrollTrigger';
gsap.registerPlugin(ScrollTrigger);

// Scroll-linked Three.js object
gsap.to(mesh.rotation, {
  y: Math.PI * 2,
  scrollTrigger: {
    trigger: '#section-2',
    start: 'top center',
    end: 'bottom center',
    scrub: 1,       // 1 = 1s lag behind scroll (smooth)
    // scrub: true  = tight coupling (instant)
    markers: false, // Debug markers
    onEnter: () => console.log('section entered'),
    onLeave: () => console.log('section left'),
    onUpdate: (self) => {
      // self.progress = 0-1
      mesh.material.opacity = self.progress;
    },
  }
});

// Pin a section while 3D animation plays
ScrollTrigger.create({
  trigger: '#hero',
  start: 'top top',
  end: '+=200%',  // Pin for 200vh
  pin: true,
  pinSpacing: true,
  scrub: 1,
  animation: gsap.timeline()
    .to(mesh.position, { y: 2, duration: 1 })
    .to(mesh.rotation, { y: Math.PI, duration: 1 }),
});
```

## Timeline — Sequence with Labels

```js
const tl = gsap.timeline({
  defaults: { ease: 'power2.out', duration: 0.6 },
  paused: true,  // Don't autoplay
});

tl
  .from('.hero-title', { opacity: 0, y: 40 })
  .from('.hero-subtitle', { opacity: 0, y: 20 }, '-=0.3')  // 0.3s overlap
  .to(mesh.material, { opacity: 1, duration: 1 }, 'sceneIn')  // label
  .to(camera.position, { z: 5, duration: 1.5 }, 'sceneIn')    // same label = simultaneous
  .from('.cta', { opacity: 0, scale: 0.8 }, '+=0.2');         // 0.2s after previous ends

// Play on trigger
document.querySelector('.enter-btn').addEventListener('click', () => tl.play());
```

## SplitText (Club Plugin)

```js
import { SplitText } from 'gsap/SplitText';
gsap.registerPlugin(SplitText);

const split = new SplitText('.hero-title', {
  type: 'chars,words,lines',
  charsClass: 'char',
  wordsClass: 'word',
});

// Animate characters individually
gsap.from(split.chars, {
  opacity: 0,
  y: 50,
  rotationX: -90,
  stagger: 0.02,
  duration: 0.6,
  ease: 'back.out(1.7)',
  onComplete: () => split.revert(),  // Restore original DOM
});
```

## MotionPath (Club Plugin)

```js
import { MotionPathPlugin } from 'gsap/MotionPathPlugin';
gsap.registerPlugin(MotionPathPlugin);

// Animate along SVG path
gsap.to('.spaceship', {
  duration: 4,
  ease: 'none',  // Constant speed along path
  motionPath: {
    path: '#orbit-path',  // SVG <path> selector
    align: '#orbit-path',
    autoRotate: 90,       // Align rotation to path tangent
    alignOrigin: [0.5, 0.5],
  }
});

// 3D path in Three.js (via GSAP ticker)
const curve = new THREE.CatmullRomCurve3([...]);
const progress = { t: 0 };

gsap.to(progress, {
  t: 1,
  duration: 4,
  ease: 'none',
  onUpdate: () => {
    const point = curve.getPointAt(progress.t);
    const tangent = curve.getTangentAt(progress.t);
    mesh.position.copy(point);
    mesh.lookAt(point.clone().add(tangent));
  }
});
```

## Context — React Cleanup

```js
import { useGSAP } from '@gsap/react';

function Component() {
  const container = useRef();

  // Auto-kills animations when component unmounts
  useGSAP(() => {
    gsap.to('.box', { x: 100, duration: 1 });
    ScrollTrigger.create({ /* ... */ });
    // No cleanup needed — useGSAP handles it
  }, { scope: container });

  return <div ref={container}><div className="box" /></div>;
}
```

## Sources
- GSAP docs: https://gsap.com/docs/v3/
- ScrollTrigger: https://gsap.com/docs/v3/Plugins/ScrollTrigger/
- useGSAP hook: https://gsap.com/docs/v3/React/
