# CSS Houdini — Paint & Layout Worklets

## Browser Support

| API | Chrome | Firefox | Safari |
|-----|--------|---------|--------|
| CSS Paint API (Houdini) | 65+ | ❌ | ❌ |
| CSS Layout API | Behind flag | ❌ | ❌ |
| CSS Properties & Values | 78+ (via @property) | 128+ | 16.4+ |
| Typed OM | 66+ | 114+ | 15.4+ |

**Reality check:** Paint Worklet is Chrome-only. Use as progressive enhancement with fallback.

## CSS Paint Worklet

```js
// my-painter.js — runs in worklet context
registerPaint('noise-background', class {
  // Declare custom properties this painter reads
  static get inputProperties() {
    return ['--noise-scale', '--noise-color', '--noise-opacity'];
  }

  paint(ctx, geometry, properties) {
    const scale   = parseFloat(properties.get('--noise-scale') || 100);
    const color   = properties.get('--noise-color').toString().trim() || '#000';
    const opacity = parseFloat(properties.get('--noise-opacity') || 0.1);

    const { width, height } = geometry;

    // Draw on ctx — same as Canvas 2D API
    ctx.fillStyle = color;
    for (let x = 0; x < width; x += scale * 0.1) {
      for (let y = 0; y < height; y += scale * 0.1) {
        if (Math.random() < opacity) {
          ctx.fillRect(x, y, scale * 0.1, scale * 0.1);
        }
      }
    }
  }
});
```

```html
<!-- Load worklet -->
<script>
if ('paintWorklet' in CSS) {
  CSS.paintWorklet.addModule('/my-painter.js');
}
</script>
```

```css
.noise-card {
  --noise-scale: 80;
  --noise-color: #333;
  --noise-opacity: 0.08;

  /* Fallback for non-Chrome */
  background: #1a1a2e;
  /* Houdini override */
  background: paint(noise-background);
}
```

## @property — Animatable Custom Properties

```css
/* Register with type — enables transitions/animations */
@property --gradient-angle {
  syntax: '<angle>';
  inherits: false;
  initial-value: 0deg;
}

@property --glow-opacity {
  syntax: '<number>';
  inherits: false;
  initial-value: 0;
}

.glowing-button {
  --gradient-angle: 0deg;
  --glow-opacity: 0;

  background: linear-gradient(var(--gradient-angle), #7c3aed, #2563eb);
  box-shadow: 0 0 20px rgba(124, 58, 237, var(--glow-opacity));

  transition: --gradient-angle 600ms ease, --glow-opacity 300ms ease;
}

.glowing-button:hover {
  --gradient-angle: 135deg;
  --glow-opacity: 0.7;
}
```

**Without @property:** `transition: --gradient-angle` does nothing (custom props are untyped).
**With @property + syntax:** Fully animatable — browser knows it's an angle.

## CSS Typed OM

```js
// Read typed value (not a string)
const el = document.querySelector('.box');
const width = el.computedStyleMap().get('width');
console.log(width.value);   // 200
console.log(width.unit);    // 'px'

// Write typed value
el.attributeStyleMap.set('opacity', CSS.number(0.5));
el.attributeStyleMap.set('width',   CSS.px(200));
el.attributeStyleMap.set('transform',
  new CSSTransformValue([
    new CSSTranslate(CSS.px(10), CSS.px(20))
  ])
);
```

## Sources
- Chrome Houdini: https://developer.chrome.com/docs/css-ui/houdini
- @property MDN: https://developer.mozilla.org/en-US/docs/Web/CSS/@property
- Houdini specs: https://drafts.css-houdini.org/
