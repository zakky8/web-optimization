# Scroll-Driven Animations

## Browser Support

| Feature | Chrome | Firefox | Safari |
|---------|--------|---------|--------|
| scroll() timeline | 115+ | 110+ | 15.4+ |
| view() timeline | 115+ | 114+ | 15.4+ |
| animation-timeline | 115+ | 110+ | 15.4+ |

Source: MDN — accessed 2026-05-26

## Scroll Progress Timeline

```css
@keyframes fadeInUp {
  from { opacity: 0; transform: translateY(30px); }
  to   { opacity: 1; transform: translateY(0); }
}

.hero-text {
  animation: fadeInUp linear both;

  /* Link animation to scroll position */
  animation-timeline: scroll(root block);
  /* scroll(scroller axis) */
  /* scroller: root | nearest | self | <custom-ident> */
  /* axis: block (vertical) | inline (horizontal) | x | y */

  animation-range: entry 0% entry 30%;
  /* Animate during the first 30% of entry into viewport */
}
```

## View Progress Timeline (Most Common)

```css
/* Animate as element enters/exits viewport */
.card {
  animation: fadeInUp linear both;
  animation-timeline: view();
  animation-range: entry 10% entry 50%;
  /*
    entry:        element entering viewport
    exit:         element leaving viewport
    contain:      element fully within viewport
    entry-crossing: crossing entry boundary
    cover:        element covers any part of viewport
  */
}
```

## Named Timelines (Scroll Container Control)

```css
.scroll-container {
  /* Define named timeline */
  scroll-timeline: --my-scroll block;
  overflow-y: scroll;
}

.progress-bar {
  animation: expand-width linear both;
  /* Reference named timeline from a child */
  animation-timeline: --my-scroll;
}

@keyframes expand-width {
  from { width: 0%; }
  to   { width: 100%; }
}
```

## Parallax Effect

```css
.parallax-bg {
  animation: parallax-move linear both;
  animation-timeline: scroll(root block);
}

@keyframes parallax-move {
  from { transform: translateY(0); }
  to   { transform: translateY(-20%); }  /* Moves slower than scroll */
}
```

## Sticky Progress Indicator

```css
.sticky-header {
  position: sticky;
  top: 0;
}

.reading-progress {
  height: 3px;
  background: linear-gradient(90deg, #7c3aed, #2563eb);
  transform-origin: left;
  animation: reading-progress-animation linear both;
  animation-timeline: scroll(root block);
}

@keyframes reading-progress-animation {
  from { transform: scaleX(0); }
  to   { transform: scaleX(1); }
}
```

## JavaScript API

```js
// Create scroll timeline in JS
const timeline = new ScrollTimeline({
  source: document.documentElement,
  axis: 'block',
});

el.animate(
  [{ opacity: 0, transform: 'translateY(20px)' },
   { opacity: 1, transform: 'translateY(0)' }],
  {
    timeline,
    rangeStart: 'entry 0%',
    rangeEnd:   'entry 50%',
    fill: 'both',
  }
);
```

## Sources
- MDN animation-timeline: https://developer.mozilla.org/en-US/docs/Web/CSS/animation-timeline
- Chrome explainer: https://developer.chrome.com/docs/css-ui/scroll-driven-animations
- Polyfill: https://github.com/flackr/scroll-timeline
