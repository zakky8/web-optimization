# SVG Filter Effects for Animation

> Sources fetched 2026-05-26.
> Primary sources:
> - https://developer.mozilla.org/en-US/docs/Web/SVG/Reference/Element/feDisplacementMap
> - https://developer.mozilla.org/en-US/docs/Web/SVG/Reference/Element/feTurbulence
> - https://developer.mozilla.org/en-US/docs/Web/SVG/Reference/Element/feBlend
> All three: Baseline widely available since July 2015.

---

## SVG Filter Architecture

SVG filters are defined in `<defs>` and applied to elements via the `filter` attribute. Each filter contains a **primitive chain** where the output of one primitive feeds into the next via named `result` references.

```xml
<svg xmlns="http://www.w3.org/2000/svg">
  <defs>
    <filter id="myFilter">
      <!-- Primitives here, each outputs to a named result -->
      <feTurbulence result="noise" ... />
      <feDisplacementMap in="SourceGraphic" in2="noise" ... />
    </filter>
  </defs>

  <circle cx="100" cy="100" r="80" filter="url(#myFilter)" />
</svg>
```

Standard `in` values available to all primitives:

| Value | Meaning |
|-------|---------|
| `SourceGraphic` | The original, unfiltered element |
| `SourceAlpha` | Alpha channel of the original element |
| `BackgroundImage` | Content behind the element (rarely used) |
| `result` name | Output of a previous primitive in the chain |

---

## `<feTurbulence>` — Procedural Noise

`feTurbulence` generates Perlin noise or fractal noise textures that fill the filter region. It produces no visual output on its own — it is almost always used as the displacement map input for `feDisplacementMap` or as a texture input for other primitives.

### Attributes

| Attribute | Type | Description |
|-----------|------|-------------|
| `type` | `"turbulence"` \| `"fractalNoise"` | Noise algorithm. `turbulence` = cloud-like, higher contrast. `fractalNoise` = smoother, fog-like. |
| `baseFrequency` | Number or `x y` pair | Spatial frequency of noise. Lower values = larger features. Accepts `"0.05"` or `"0.05 0.08"` for anisotropic noise. |
| `numOctaves` | Integer | Number of noise layers stacked. More octaves = more detail, higher CPU cost. |
| `seed` | Integer | Random seed. Different values produce different noise patterns; same seed always produces the same pattern. |
| `stitchTiles` | `"stitch"` \| `"noStitch"` | Whether tile edges are blended to seamlessly tile. |

### Animating `feTurbulence` Attributes

Animating `baseFrequency` or `seed` creates continuously shifting noise — this drives animated wave, fire, water, and distortion effects.

**SMIL animation (no JS required):**
```xml
<feTurbulence type="turbulence" numOctaves="3" baseFrequency="0.02" result="noise">
  <animate
    attributeName="baseFrequency"
    values="0.02;0.06;0.02"
    dur="4s"
    repeatCount="indefinite" />
</feTurbulence>
```

**GSAP animation (JS):**
```js
const turb = document.querySelector("#waveTurb");
gsap.to(turb, {
  attr: { baseFrequency: 0.08 },
  duration: 2,
  repeat: -1,
  yoyo: true,
  ease: "sine.inOut"
});
```

**Seed cycling for random-feeling motion:**
```xml
<feTurbulence type="fractalNoise" baseFrequency="0.05" numOctaves="2">
  <animate
    attributeName="seed"
    values="0;10;20;30;0"
    dur="3s"
    repeatCount="indefinite"
    calcMode="discrete" />
</feTurbulence>
```

Note: Seed changes are discrete (instant), not interpolated. For smooth continuous motion, animate `baseFrequency` instead.

---

## `<feDisplacementMap>` — Wave Distortion

`feDisplacementMap` spatially displaces each pixel of the input image (`in`) by an amount derived from the corresponding pixel in a displacement map (`in2`). Pairing it with `feTurbulence` as the displacement map creates organic, wave-like distortion.

### Displacement Formula

```
output(x, y) = input(
  x + scale × (XChannel(x,y) - 0.5),
  y + scale × (YChannel(x,y) - 0.5)
)
```

Pixel values in the displacement map range 0–1; subtracting 0.5 centres the displacement so a mid-grey pixel causes no movement.

### Attributes

| Attribute | Values | Description |
|-----------|--------|-------------|
| `in` | `SourceGraphic`, result name | The image to distort |
| `in2` | Result name (usually noise) | The displacement map |
| `scale` | Number | Displacement strength in user units. `0` = no effect. Higher = stronger distortion. |
| `xChannelSelector` | `R`, `G`, `B`, `A` | Which channel of `in2` drives X-axis displacement |
| `yChannelSelector` | `R`, `G`, `B`, `A` | Which channel of `in2` drives Y-axis displacement |

### Classic Wave Distortion Filter

```xml
<filter id="waveDistort">
  <feTurbulence
    type="turbulence"
    baseFrequency="0.05"
    numOctaves="2"
    result="noise" />
  <feDisplacementMap
    in="SourceGraphic"
    in2="noise"
    scale="20"
    xChannelSelector="R"
    yChannelSelector="G" />
</filter>
```

Apply: `<image href="photo.jpg" filter="url(#waveDistort)" />`

### Animated Wave Effect

Animate `scale` on `feDisplacementMap` for a pulsing distortion, or animate `baseFrequency` on `feTurbulence` for a flowing wave:

```xml
<filter id="animatedWave">
  <feTurbulence
    id="waveTurb"
    type="turbulence"
    baseFrequency="0.03"
    numOctaves="3"
    result="noise">
    <animate
      attributeName="baseFrequency"
      values="0.03;0.07;0.03"
      dur="5s"
      repeatCount="indefinite" />
  </feTurbulence>
  <feDisplacementMap
    in="SourceGraphic"
    in2="noise"
    scale="30"
    xChannelSelector="R"
    yChannelSelector="G" />
</filter>
```

**GSAP control over `scale` (animate distortion intensity):**
```js
const dispMap = document.querySelector("#waveDisp");

gsap.fromTo(dispMap,
  { attr: { scale: 0 } },
  {
    attr: { scale: 40 },
    duration: 1.5,
    ease: "power2.inOut",
    yoyo: true,
    repeat: -1
  }
);
```

### Glitch Effect Using `feDisplacementMap`

```xml
<filter id="glitch">
  <feTurbulence
    type="fractalNoise"
    baseFrequency="0 0.8"
    numOctaves="1"
    seed="2"
    result="noise" />
  <feDisplacementMap
    in="SourceGraphic"
    in2="noise"
    scale="8"
    xChannelSelector="R"
    yChannelSelector="G" />
</filter>
```

`baseFrequency="0 0.8"` creates horizontal-stripe noise (high frequency on y axis, zero on x) — this produces the characteristic horizontal-shift glitch look.

---

## `<feBlend>` — Layer Compositing

`feBlend` composites two inputs using a blend mode, equivalent to CSS `mix-blend-mode`. It is commonly used in filter chains to combine distorted and undistorted versions of an image, or to add colour grading effects.

### Attributes

| Attribute | Description |
|-----------|-------------|
| `in` | First input (usually `SourceGraphic`) |
| `in2` | Second input |
| `mode` | Blend mode |

### `mode` Values

`normal` | `multiply` | `screen` | `overlay` | `darken` | `lighten` | `color-dodge` | `color-burn` | `hard-light` | `soft-light` | `difference` | `exclusion` | `hue` | `saturation` | `color` | `luminosity`

### Example — Double-Exposure Effect with Animation

```xml
<filter id="doubleExposure">
  <feTurbulence
    type="fractalNoise"
    baseFrequency="0.6"
    numOctaves="3"
    result="grain" />
  <feBlend
    in="SourceGraphic"
    in2="grain"
    mode="screen"
    result="blended" />
</filter>
```

Animate `feFlood` colour into a `feBlend` for a colour-wash effect:

```js
const flood = document.querySelector("#colorFlood");
gsap.to(flood, {
  attr: { "flood-color": "#ff4400" },
  duration: 2,
  ease: "sine.inOut",
  yoyo: true,
  repeat: -1
});
```

---

## Animating Filter Primitive Attributes — Methods Compared

| Method | Syntax | Best for |
|--------|--------|----------|
| SMIL `<animate>` | Inline in SVG | Self-contained SVGs, no-JS contexts |
| GSAP `attr:{}` | `gsap.to(el, { attr: { scale: 40 } })` | Timeline control, sequencing, easing library |
| anime.js attribute tween | `animate(el, { scale: 40 })` on the filter primitive | anime.js projects |
| Direct DOM | `el.setAttribute("scale", val)` in `requestAnimationFrame` | Raw control, no library overhead |
| CSS custom property + `@keyframes` | Not possible — filter primitive attributes are not CSS properties | N/A |

**Key constraint:** Filter primitive attributes (`scale`, `baseFrequency`, `seed`, etc.) are **SVG attributes, not CSS properties**. They cannot be animated with CSS `@keyframes` or CSS transitions. You must use SMIL, JS attribute manipulation, or a JS animation library targeting `attr`.

---

## Performance of SVG Filters

SVG filters are **CPU-rendered** (software rasterisation) in most browsers. They are not GPU-accelerated in the way CSS `transform` and `opacity` are.

### Cost hierarchy (low to high)

1. `feFlood`, `feColorMatrix` — cheap, mathematical operations
2. `feBlend`, `feComposite` — moderate
3. `feDisplacementMap` — moderate-to-expensive, depends on scale and element size
4. `feTurbulence` — expensive, especially with high `numOctaves`
5. `feGaussianBlur` with large `stdDeviation` — very expensive
6. Combinations (`feTurbulence` + `feDisplacementMap` animated every frame) — expensive

### Practical guidelines

- **Apply filters only to small elements** when animating. A full-viewport filtered element repaints the entire screen every frame.
- **`feBlend` and `feColorMatrix` are safer** to animate frequently than noise-based filters.
- **Pre-generate noise** with a static `seed` and animate `scale` or `x`/`y` offset rather than animating `baseFrequency` if you can achieve a similar look.
- **`filter` property triggers repaint**, not just composite. Toggling a filter on/off on a large element is expensive.
- **Test on mobile.** Filter performance on low-end Android devices is significantly worse than desktop. A smooth 60fps desktop animation can drop to 20fps on mobile.
- **CSS `filter: blur()` is faster** than SVG `feGaussianBlur` because it is GPU-composited in modern browsers. Prefer CSS `filter` for simple blur effects on DOM elements.
- Use Chrome DevTools **Layers panel** and **Rendering > Paint flashing** to identify which elements are causing excessive repaints due to filter animations.

---

## Complete Animated Distortion Example

```html
<svg width="400" height="400" viewBox="0 0 400 400">
  <defs>
    <filter id="liquidFilter" x="-20%" y="-20%" width="140%" height="140%">
      <feTurbulence
        id="liqTurb"
        type="turbulence"
        baseFrequency="0.02"
        numOctaves="3"
        seed="5"
        result="noise" />
      <feDisplacementMap
        id="liqDisp"
        in="SourceGraphic"
        in2="noise"
        scale="0"
        xChannelSelector="R"
        yChannelSelector="G" />
    </filter>
  </defs>

  <image
    href="subject.jpg"
    x="0" y="0"
    width="400" height="400"
    filter="url(#liquidFilter)" />
</svg>
```

```js
// GSAP: animate in the distortion, hold, then release
const tl = gsap.timeline({ repeat: -1 });
tl
  .to("#liqDisp", { attr: { scale: 35 }, duration: 1.5, ease: "power2.out" })
  .to("#liqTurb", { attr: { baseFrequency: 0.06 }, duration: 2, ease: "sine.inOut" }, "<")
  .to("#liqDisp", { attr: { scale: 0 }, duration: 1.5, ease: "power2.in" })
  .to("#liqTurb", { attr: { baseFrequency: 0.02 }, duration: 1.5, ease: "sine.inOut" }, "<");
```
