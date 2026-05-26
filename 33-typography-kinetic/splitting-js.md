# Splitting.js

**Folder:** `33-typography-kinetic`
**Last updated:** 2026-05-26

Source verified: https://splitting.js.org/guide — accessed 2026-05-26

---

## What Splitting.js Does

Splitting.js is a utility that wraps text nodes (and other elements) in `<span>` elements and injects CSS custom properties onto each span. The library does no animation itself — it hands off numbered indices and counts to CSS or JavaScript animation code.

- Size: 1.5 kB minified and gzipped
- License: MIT
- Author: Stephen Shaw

Install:

```bash
npm install splitting
```

Or via CDN:

```html
<script src="https://unpkg.com/splitting/dist/splitting.min.js"></script>
<link rel="stylesheet" href="https://unpkg.com/splitting/dist/splitting.css">
```

The CSS file is required for the `lines` plugin (it uses inline-flex layout to measure line breaks).

---

## Plugins and Generated CSS Custom Properties

Each plugin targets a different DOM structure and generates a different set of CSS variables.

| Plugin  | Target               | CSS variables generated                                                  |
|---------|----------------------|--------------------------------------------------------------------------|
| `chars` | Text nodes           | `--char-index`, `--char-total`                                           |
| `words` | Text nodes           | `--word-index`, `--word-total`                                           |
| `lines` | Word spans (post-words split) | `--line-total` (on wrapper), `--line-index` (on line group)     |
| `items` | Direct child elements | `--item-index`, `--item-total`                                          |
| `grid`  | Child elements with position | `--row-index`, `--col-index`, `--row-total`, `--col-total`     |
| `cols`  | Child elements       | `--col-index`                                                            |
| `rows`  | Child elements       | `--row-index`                                                            |
| `cells` | Grid cell spans      | `--cell-index`, `--row-index`, `--col-index`, `--cell-total`, `--row-total`, `--col-total` |

The `chars` plugin also inherits `--word-index` and `--word-total` from a parent `words` split when both are run together (see `by: ['words', 'chars']`).

---

## API

### Basic usage

```js
import Splitting from 'splitting';

// Default: splits all [data-splitting] elements by chars
Splitting();

// Target specific elements
Splitting({ target: '.hero-text', by: 'chars' });

// Multiple plugins at once
Splitting({ target: '.headline', by: ['words', 'chars'] });
```

HTML attribute shorthand:

```html
<h1 data-splitting>Hello World</h1>
<script>Splitting();</script>
```

### Return value

`Splitting()` returns an array of result objects, one per matched element.

```js
const results = Splitting({ target: '#title', by: 'chars' });

// results[0].chars  → array of <span> elements, one per character
// results[0].words  → array of <span> elements, one per word
// results[0].lines  → array of arrays (if lines plugin ran)
```

### Server-side rendering

```js
const html = Splitting.html({ content: 'Hello', by: 'chars' });
// Returns HTML string: <span class="word" ...><span class="char" style="--char-index:0">H</span>...
```

### Custom plugins

```js
Splitting.add({
  by: 'syllables',
  key: 'syllable',
  depends: ['words'],
  split(el) {
    // Return array of split nodes
  }
});
```

---

## DOM Output

For `Splitting({ target: 'h1', by: 'chars' })` on `<h1>Hello</h1>`:

```html
<h1 class="splitting chars words" style="--word-total:1; --char-total:5">
  <span class="word" data-word="Hello" style="--word-index:0">
    <span class="char" data-char="H" style="--char-index:0">H</span>
    <span class="char" data-char="e" style="--char-index:1">e</span>
    <span class="char" data-char="l" style="--char-index:2">l</span>
    <span class="char" data-char="l" style="--char-index:3">l</span>
    <span class="char" data-char="o" style="--char-index:4">o</span>
  </span>
</h1>
```

---

## CSS Animation Patterns

### Stagger via --char-index

```css
.splitting .char {
  display: inline-block;
  opacity: 0;
  transform: translateY(1em);
  animation: char-reveal 0.4s ease forwards;
  animation-delay: calc(var(--char-index) * 0.04s);
}

@keyframes char-reveal {
  to {
    opacity: 1;
    transform: translateY(0);
  }
}
```

### Reversed stagger (last character first)

```css
.splitting .char {
  animation-delay: calc((var(--char-total) - var(--char-index)) * 0.03s);
}
```

### Clip-path character reveal

```css
.splitting .char {
  display: inline-block;
  clip-path: inset(0 100% 0 0);
  animation: clip-reveal 0.5s cubic-bezier(0.76, 0, 0.24, 1) forwards;
  animation-delay: calc(var(--char-index) * 0.05s);
}

@keyframes clip-reveal {
  to { clip-path: inset(0 0% 0 0); }
}
```

### Wave animation using index

```css
.splitting .char {
  display: inline-block;
  animation: wave 1.5s ease-in-out infinite;
  animation-delay: calc(var(--char-index) * 0.08s);
}

@keyframes wave {
  0%, 100% { transform: translateY(0); }
  50%       { transform: translateY(-0.4em); }
}
```

### Color gradient across characters

```css
.splitting .char {
  color: hsl(calc(var(--char-index) / var(--char-total) * 360deg), 80%, 55%);
}
```

### Word-level stagger

```css
.splitting .word {
  display: inline-block;
  opacity: 0;
  transform: translateX(-1em);
  animation: word-in 0.5s ease forwards;
  animation-delay: calc(var(--word-index) * 0.12s);
}

@keyframes word-in {
  to { opacity: 1; transform: translateX(0); }
}
```

---

## Combining with Intersection Observer for Scroll Reveals

```js
import Splitting from 'splitting';

Splitting({ target: '.reveal-text', by: 'chars' });

const observer = new IntersectionObserver((entries) => {
  entries.forEach(entry => {
    if (entry.isIntersecting) {
      entry.target.classList.add('in-view');
      observer.unobserve(entry.target);
    }
  });
}, { threshold: 0.2 });

document.querySelectorAll('.reveal-text').forEach(el => observer.observe(el));
```

```css
.splitting.reveal-text .char {
  display: inline-block;
  opacity: 0;
  transform: translateY(0.5em);
  transition:
    opacity 0.4s ease,
    transform 0.4s ease;
  transition-delay: calc(var(--char-index) * 0.025s);
}

.splitting.reveal-text.in-view .char {
  opacity: 1;
  transform: translateY(0);
}
```

---

## Combining with GSAP

```js
import Splitting from 'splitting';
import gsap from 'gsap';

const results = Splitting({ target: '.headline', by: 'chars' });
const chars = results[0].chars;

gsap.from(chars, {
  opacity: 0,
  y: 60,
  rotateX: -90,
  transformOrigin: '50% 0%',
  stagger: 0.04,
  duration: 0.7,
  ease: 'back.out(1.7)',
});
```

---

## Accessibility Notes

- Splitting wraps characters in `<span>` elements, which breaks the text for screen readers that read character-by-character.
- Mitigate by adding `aria-label` to the parent element with the full text, and `aria-hidden="true"` on the split container.

```js
const el = document.querySelector('.headline');
el.setAttribute('aria-label', el.textContent);

const wrapper = document.createElement('span');
wrapper.setAttribute('aria-hidden', 'true');
wrapper.innerHTML = el.innerHTML;
el.innerHTML = '';
el.appendChild(wrapper);

Splitting({ target: wrapper, by: 'chars' });
```

---

## References

- Splitting.js site: https://splitting.js.org — accessed 2026-05-26
- Splitting.js guide: https://splitting.js.org/guide — accessed 2026-05-26
- GitHub: https://github.com/shshaw/Splitting
