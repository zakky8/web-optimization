# CSS Tricks for Mobile 3D

## will-change - Use Sparingly

Forces browser to promote element to its own compositor layer BEFORE animation starts.
Eliminates the one-frame composite tear at animation start.
OVERUSE causes memory pressure from too many layers.

```css
/* GOOD: only set on elements you KNOW will animate, remove after */
.animated {
  will-change: transform, opacity;
}

/* GOOD: remove after animation completes */
```
```javascript
element.addEventListener('animationend', () => {
  element.style.willChange = 'auto'
})
```

```css
/* BAD: will-change on everything */
* { will-change: transform; }  /* destroys mobile GPU memory */

/* BAD: will-change: contents - almost never what you want */
.container { will-change: contents; }
```

## Force GPU Layer (Classic Hack, Still Works)

```css
.gpu-layer {
  transform: translateZ(0);
  /* or */
  transform: translate3d(0, 0, 0);
  /* Both promote element to GPU compositor layer */
}
```

## backface-visibility

Prevents rendering of back face of 3D-transformed elements.
Also acts as GPU layer promotion hint.

```css
.flip-card {
  transform-style: preserve-3d;
  backface-visibility: hidden;
  -webkit-backface-visibility: hidden;  /* still required for Safari */
}
```

Chrome bug: overflow: scroll | auto on an element IGNORES backface-visibility.
Also: overflow with any non-visible value forces transform-style to flat,
overriding preserve-3d even if explicitly set.

## -webkit-font-smoothing - iOS Has No Effect

-webkit-font-smoothing: antialiased has NO effect on iOS/iPadOS.
Only affects macOS. All iOS browsers use WebKit (App Store policy).
```css
/* This only works on macOS Safari */
-webkit-font-smoothing: antialiased;
/* Safe to include (harmless on iOS), but don't rely on it */
```

## Safari Bold Flash Fix

When element with text undergoes 3D transform, Safari switches antialiasing,
making text appear to change weight (flash) on animation start/end.

```css
.text-element {
  -webkit-font-smoothing: antialiased;
  transform: translateZ(0);           /* locks antialiasing mode on GPU layer */
  backface-visibility: hidden;
  -webkit-backface-visibility: hidden;
  perspective: 1000px;
}
```

## Overscroll Behavior

```css
/* Prevent iOS rubber-band bounce on scroll container */
.scroll-container {
  overscroll-behavior: none;          /* iOS 13.1+, Chrome 63+ */
  overscroll-behavior-y: contain;     /* contain = allow but no chain */
}

/* For body-level prevention */
body {
  overscroll-behavior: none;
}
```

## Canvas Element Best Practices

```css
canvas {
  display: block;                    /* removes 4px bottom gap from inline */
  touch-action: none;                /* no browser touch handling on canvas */
  user-select: none;
  -webkit-user-select: none;
  -webkit-tap-highlight-color: transparent;  /* no blue flash on tap */
  outline: none;
}

/* Prevent canvas from being draggable (affects pointer events) */
canvas[draggable="false"] {
  -webkit-user-drag: none;
}
```

## Performance CSS for Animated Elements

```css
/* Compositor-only properties - can animate without triggering layout or paint */
/* SAFE to animate: transform, opacity, filter (sometimes) */
/* AVOID animating: width, height, top, left, padding, margin, border */

.safe-to-animate {
  /* Use transform instead of top/left */
  transform: translateX(100px);  /* GOOD */
  /* top: 100px;  BAD - triggers layout */

  /* Use opacity instead of visibility/display */
  opacity: 0;                    /* GOOD */
  /* visibility: hidden; - triggers paint */
}
```
