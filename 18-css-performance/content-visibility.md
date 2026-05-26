# content-visibility & Layout Containment

## content-visibility: auto

Tells the browser to skip rendering off-screen content entirely.
Measured impact: up to 7x faster initial render for long pages.

```css
/* Apply to sections/cards that might be off-screen */
.article-section {
  content-visibility: auto;

  /* REQUIRED: tell browser the expected size to reserve space */
  /* Prevents layout shift when content becomes visible */
  contain-intrinsic-size: auto 600px;
  /* auto = browser remembers actual size after first render */
  /* 600px = initial estimate before first render */
}

/* More specific */
.card {
  content-visibility: auto;
  contain-intrinsic-size: auto 300px;  /* height estimate */
}

/* With width + height */
.hero-section {
  content-visibility: auto;
  contain-intrinsic-size: auto 100vw auto 80vh;
  /* syntax: [<width>] <height> or <width> <height> */
}
```

## contain-intrinsic-size

```css
/* Single value = both width and height */
contain-intrinsic-size: 300px;

/* Two values = width height */
contain-intrinsic-size: 500px 300px;

/* auto keyword: use remembered size, fall back to estimate */
contain-intrinsic-size: auto 300px;

/* Per-axis */
contain-intrinsic-width: auto 500px;
contain-intrinsic-height: auto 300px;
```

## contain

Manual containment for performance:

```css
/* Layout containment: changes inside don't affect outside */
.isolated-widget {
  contain: layout;
}

/* Style containment: counters, quotes don't escape */
.counter-widget {
  contain: style;
}

/* Size containment: element's size independent of content */
.fixed-size-box {
  contain: size;
  width: 300px;
  height: 200px;
}

/* Strict: layout + style + size (most aggressive) */
.fully-isolated {
  contain: strict;
}

/* content: layout + style (no size) — most common */
.content-isolated {
  contain: content;
}
```

## Browser Support

| Property | Chrome | Firefox | Safari |
|----------|--------|---------|--------|
| content-visibility | 85+ | 109+ | 18+ |
| contain-intrinsic-size | 83+ | 109+ | 17+ |
| contain | 52+ | 69+ | 10.1+ |

Source: MDN — accessed 2026-05-26

## @starting-style (Entry Animations)

```css
/* Animate FROM when element first appears in DOM */
.dialog[open] {
  opacity: 1;
  transform: translateY(0);
  transition: opacity 200ms, transform 200ms;

  /* Starting state — applied before transition starts */
  @starting-style {
    opacity: 0;
    transform: translateY(-10px);
  }
}

/* With transition-behavior: allow-discrete for display changes */
.menu {
  display: none;
  opacity: 0;
  transition: opacity 200ms, display 200ms;
  transition-behavior: allow-discrete;

  &.open {
    display: block;
    opacity: 1;

    @starting-style { opacity: 0; }
  }
}
```

| Feature | Chrome | Firefox | Safari |
|---------|--------|---------|--------|
| @starting-style | 117+ | 129+ | 17.5+ |
| transition-behavior: allow-discrete | 117+ | 129+ | 17.4+ |

## Sources
- content-visibility MDN: https://developer.mozilla.org/en-US/docs/Web/CSS/content-visibility
- @starting-style MDN: https://developer.mozilla.org/en-US/docs/Web/CSS/@starting-style
- Una Kravets explainer: https://developer.chrome.com/docs/css-ui/css-animation-entry-exit
