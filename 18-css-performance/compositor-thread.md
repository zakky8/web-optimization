# CSS Compositor Thread & RenderingNG

## The RenderingNG Pipeline (Chrome)

Chrome's rendering engine processes frames in 12 stages:

```
1.  Parse HTML/CSS → DOM + CSSOM
2.  Style recalc → Computed styles
3.  Layout → Box model, positions
4.  Layer tree construction → Which elements get own layers?
5.  Paint (rasterize) → Draw commands per layer
6.  Compositing → Combine layers on GPU thread
7.  Display → Screen output
```

The **compositor thread** runs stages 6-7 independently of the main thread.
Properties that only affect compositing skip stages 2-5 entirely → **no jank**.

## Compositor-Safe Properties

These properties animate on the compositor thread (main thread blocked = no jank):

| Property | Notes |
|----------|-------|
| `transform` | translate, rotate, scale, matrix3d |
| `opacity` | 0-1 fade |
| `filter` | blur, brightness, contrast, drop-shadow |
| `backdrop-filter` | frosted glass effect |
| `will-change: transform` | Promotes to own layer pre-emptively |

**Critical rule:** `transform: translateX()` is compositor-safe. `left: Xpx` is NOT.

## Non-Compositor Properties (Avoid Animating)

| Property | Triggers | Cost |
|----------|----------|------|
| `width`, `height` | Layout + Paint + Composite | Expensive |
| `top`, `left` | Layout + Paint + Composite | Expensive |
| `background-color` | Paint + Composite | Medium |
| `border-radius` | Paint + Composite (sometimes) | Medium |
| `box-shadow` | Paint + Composite | Medium |
| `clip-path` | Layout* + Paint + Composite | Medium-High |

*clip-path triggers layout in some browsers.

## will-change

```css
/* Promotes element to own compositor layer BEFORE animation starts */
/* Use sparingly — each layer costs VRAM */
.animated-card {
  will-change: transform, opacity;
}

/* Set via JS only for the animation duration */
element.addEventListener('mouseenter', () => {
  element.style.willChange = 'transform';
});
element.addEventListener('animationend', () => {
  element.style.willChange = 'auto';  /* Release layer */
});
```

**Warning:** `will-change: transform` on too many elements fragments the GPU layer tree
and causes worse performance than not using it. Target only the animating element.

## Chrome DevTools: Layers Panel

1. DevTools → More tools → Layers
2. Yellow highlights = composited layer
3. Hover layer to see why it was promoted
4. Check "Memory" column — each layer costs VRAM

## Sources
- RenderingNG: https://developer.chrome.com/blog/renderingng
- CSS triggers: https://csstriggers.com/
- Google: https://web.dev/articles/rendering-performance
