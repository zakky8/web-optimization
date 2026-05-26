# Variable Fonts

**Folder:** `33-typography-kinetic`
**Last updated:** 2026-05-26

---

## What is a Variable Font?

A variable font is a single font file that encodes a continuous design space across one or more axes. Instead of shipping four separate files for Regular, Bold, Italic, and Bold-Italic, one variable font file covers the entire range between those extremes — and every point in between.

The spec is defined by OpenType 1.8 (2016). All major browsers support it as of 2018.

---

## Registered Axes

Registered axes have four-letter lowercase tags and defined value ranges.

| Axis tag | Full name         | Typical range        | CSS property alias         |
|----------|-------------------|----------------------|----------------------------|
| `wght`   | Weight            | 100–900              | `font-weight`              |
| `wdth`   | Width             | 75–125 (%)           | `font-stretch`             |
| `ital`   | Italic            | 0 or 1 (binary)      | `font-style: italic`       |
| `slnt`   | Slant             | -90–90 (degrees)     | `font-style: oblique Xdeg` |
| `opsz`   | Optical size      | 8–144 (pt)           | `font-optical-sizing`      |

Notes:
- `ital` is binary on most faces; some fonts expose an `slnt` axis instead for a continuous oblique effect.
- `opsz` adjusts stroke contrast and spacing for the intended render size. Setting `font-optical-sizing: none` lets you override it manually.

---

## Custom (Private) Axes

Custom axes use four-letter UPPERCASE tags. Examples seen in the wild:

| Tag    | Description                          | Font example               |
|--------|--------------------------------------|----------------------------|
| `CASL` | Casual — straight to curved strokes  | Recursive                  |
| `CRSV` | Cursive on/off                       | Recursive                  |
| `MONO` | Monospace interpolation              | Recursive                  |
| `SOFT` | Softness / rounded corners           | Nunito (experimental)      |
| `FILL` | Fill amount (for icons)              | Material Symbols           |
| `GRAD` | Grade (weight without layout shift)  | Roboto Flex, Google fonts  |
| `wght` | (also used as custom on icon fonts)  | Material Symbols           |

`GRAD` is particularly useful for dark-mode switching: it changes apparent weight visually without altering advance widths, so text reflow does not occur.

---

## font-variation-settings Syntax

```css
/* Single axis */
font-variation-settings: 'wght' 650;

/* Multiple axes */
font-variation-settings: 'wght' 700, 'wdth' 85, 'opsz' 24;

/* Custom axis */
font-variation-settings: 'wght' 400, 'CASL' 1, 'MONO' 0.5;
```

Important caveats:
- `font-variation-settings` is a low-level override. Prefer high-level properties (`font-weight`, `font-stretch`) when they map to the axis — they compose better with inheritance.
- Specifying *any* axis in `font-variation-settings` forces you to also specify *all other axes* you care about, or they reset to their default. This is the "axis reset" footgun.
- Values are unitless numbers. The valid range depends on the font's axis definition — check `wght min/max` via `Font Gauntlet` or `Wakamai Fondue`.

---

## Animating Variable Fonts with @property

CSS custom properties are not animatable by default (the browser treats them as discrete). `@property` declares a typed custom property that the browser knows how to interpolate.

```css
/* 1. Register the custom property */
@property --font-weight {
  syntax: '<number>';
  inherits: true;
  initial-value: 400;
}

@property --font-width {
  syntax: '<number>';
  inherits: false;
  initial-value: 100;
}

/* 2. Wire it to font-variation-settings */
.text {
  font-variation-settings: 'wght' var(--font-weight), 'wdth' var(--font-width);
  transition: --font-weight 0.3s ease, --font-width 0.3s ease;
}

.text:hover {
  --font-weight: 900;
  --font-width: 75;
}
```

Why this works:
- Without `@property`, `var(--font-weight)` inside `font-variation-settings` is a string substitution; two strings cannot be interpolated.
- With `@property` type `<number>`, the browser interpolates the number at each animation frame, then substitutes.

Keyframe animation example:

```css
@keyframes weight-pulse {
  0%   { --font-weight: 100; }
  50%  { --font-weight: 900; }
  100% { --font-weight: 100; }
}

.text {
  animation: weight-pulse 2s ease-in-out infinite;
}
```

GSAP can also tween `--font-weight` when combined with `@property`:

```js
gsap.to('.text', {
  '--font-weight': 900,
  duration: 1,
  ease: 'power2.inOut',
});
```

---

## Google Fonts Variable Fonts

Google Fonts exposes variable font axes via URL parameters.

```html
<!-- wght range 100–900 -->
<link rel="preconnect" href="https://fonts.googleapis.com">
<link rel="preconnect" href="https://fonts.gstatic.com" crossorigin>
<link href="https://fonts.googleapis.com/css2?family=Inter:wght@100..900&display=swap" rel="stylesheet">
```

The `..` notation requests the full axis range rather than discrete weights, delivering a single variable font file instead of multiple static ones.

Multi-axis example (Roboto Flex — wght + wdth):

```
https://fonts.googleapis.com/css2?family=Roboto+Flex:opsz,wdth,wght@8..144,75..125,100..900&display=swap
```

Recursive (wght + CASL + MONO + CRSV):

```
https://fonts.googleapis.com/css2?family=Recursive:slnt,wght,CASL,CRSV,MONO@-15..0,300..1000,0..1,0..1,0..1&display=swap
```

Notes:
- Not all fonts on Google Fonts are variable. Filter by "Variable" in the catalog.
- Google Fonts automatically negotiates WOFF2 format for supported browsers.
- The CSS API adds a `@font-face` rule with the axis ranges embedded in `font-weight` / `font-stretch` descriptors.

---

## Checking Available Axes

Tools to inspect a font file:
- **Wakamai Fondue** — https://wakamaifondue.com (drag-and-drop, shows all axes with min/default/max)
- **Axis-Praxis** — https://www.axis-praxis.org
- `fonttools varLib.instancer` CLI — inspect and extract axis info from a `.ttf`/`.woff2`

---

## References

- OpenType Spec — Font Variations: https://docs.microsoft.com/en-us/typography/opentype/spec/fvar
- MDN font-variation-settings: https://developer.mozilla.org/en-US/docs/Web/CSS/font-variation-settings
- MDN @property: https://developer.mozilla.org/en-US/docs/Web/CSS/@property
- Google Fonts variable fonts guide: https://fonts.google.com/knowledge/using_type/using_variable_fonts_on_the_web
- v-fonts.com: https://v-fonts.com (variable font specimen catalog)
