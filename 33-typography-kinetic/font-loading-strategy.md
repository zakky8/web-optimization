# Font Loading Strategy

**Folder:** `33-typography-kinetic`
**Last updated:** 2026-05-26

---

## The Core Problem

Web fonts are render-blocking in the sense that the browser will not paint text until it has resolved how to render it. The window between "font request sent" and "font bytes received" is where text flicker, invisible text, and layout shift live.

---

## font-display Values

`font-display` is a descriptor inside `@font-face` that controls the swap timeline.

```css
@font-face {
  font-family: 'Inter';
  src: url('/fonts/inter-var.woff2') format('woff2');
  font-weight: 100 900;
  font-display: swap;  /* <-- this */
}
```

| Value      | Block period      | Swap period       | Practical effect                                      |
|------------|-------------------|-------------------|-------------------------------------------------------|
| `auto`     | Browser default   | Browser default   | Usually equivalent to `block` in most browsers       |
| `block`    | Short (~3 s)      | Infinite          | Text invisible then swaps. Avoids FOUT, causes FOIT  |
| `swap`     | Extremely short   | Infinite          | Fallback shown immediately, swaps when ready. FOUT.  |
| `fallback` | Extremely short   | Short (~3 s)      | Fallback shown; swaps if font loads within 3 s only  |
| `optional` | Extremely short   | None              | Font used only if immediately available (cached)     |

Guidance:
- **`swap`** — best for body text and headings where content is primary. Accepts a flash of fallback font.
- **`fallback`** — good middle ground for branding fonts that must load quickly or not at all.
- **`optional`** — ideal for decorative/display fonts that add style but whose absence is acceptable. Also the best choice for performance-first builds.
- **`block`** — only justified for icon fonts where fallback characters are meaningless.

---

## Preloading Fonts

Preload tells the browser to fetch the font at highest priority before it parses the stylesheet.

```html
<link
  rel="preload"
  href="/fonts/inter-var.woff2"
  as="font"
  type="font/woff2"
  crossorigin
>
```

Rules:
- `crossorigin` is required even for same-origin fonts. Font requests are always CORS.
- Only preload fonts you are certain will be used on the page. Unnecessary preloads waste bandwidth and compete with critical resources.
- Preload + `font-display: optional` is a common high-performance pattern: the font is available from cache on first visit, `optional` means it will be used immediately on return visits.

---

## FontFace Observer

FontFace Observer is a small (~1.3 kB) library that resolves a Promise when a font is loaded.

```js
import FontFaceObserver from 'fontfaceobserver';

const inter = new FontFaceObserver('Inter', { weight: 700 });

inter.load().then(() => {
  document.documentElement.classList.add('fonts-loaded');
}).catch(() => {
  document.documentElement.classList.add('fonts-failed');
});
```

Then in CSS:

```css
/* Fallback state — uses system font */
body {
  font-family: 'Arial', sans-serif;
}

/* Font loaded — switch to web font */
.fonts-loaded body {
  font-family: 'Inter', sans-serif;
}
```

This pattern gives you explicit control over when the swap happens, and lets you animate the transition.

FontFace Observer source: https://github.com/bramstein/fontfaceobserver

---

## Subsetting

Subsetting removes unused glyphs from a font file. A full Unicode typeface can be 500 kB+. A Latin subset for a marketing page can be under 20 kB.

### glyphhanger

glyphhanger crawls a URL or list of characters and generates a subset.

```bash
npm install -g glyphhanger

# Crawl a URL and subset automatically
glyphhanger https://example.com --subset=fonts/inter-var.ttf --formats=woff2

# Specify a character whitelist manually
glyphhanger --whitelist="ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz0123456789.,!?" \
  --subset=fonts/inter-var.ttf \
  --formats=woff2
```

glyphhanger repo: https://github.com/zachleat/glyphhanger

### pyftsubset (fonttools)

```bash
pip install fonttools brotli

# Subset to Latin Extended glyphs only
pyftsubset inter-var.ttf \
  --unicodes="U+0000-00FF,U+0100-017F" \
  --flavor=woff2 \
  --output-file=inter-var-latin.woff2

# Subset to a specific character string
pyftsubset inter-var.ttf \
  --text="Hello World" \
  --flavor=woff2 \
  --output-file=inter-hello.woff2
```

fonttools repo: https://github.com/fonttools/fonttools

### Unicode ranges with @font-face

Combine subsetting with `unicode-range` to let the browser only download the subset it needs:

```css
@font-face {
  font-family: 'Inter';
  src: url('/fonts/inter-latin.woff2') format('woff2');
  font-weight: 100 900;
  unicode-range: U+0000-00FF, U+0100-017F;
  font-display: swap;
}

@font-face {
  font-family: 'Inter';
  src: url('/fonts/inter-greek.woff2') format('woff2');
  font-weight: 100 900;
  unicode-range: U+0370-03FF;
  font-display: swap;
}
```

The browser only downloads the Greek subset if the page contains Greek characters.

---

## WOFF2 vs TTF Size Comparison

WOFF2 uses Brotli compression; TTF is uncompressed. Approximate ratios on real fonts:

| Font file          | TTF size  | WOFF2 size | Reduction |
|--------------------|-----------|------------|-----------|
| Inter Regular      | ~310 kB   | ~98 kB     | ~68 %     |
| Inter Variable     | ~380 kB   | ~110 kB    | ~71 %     |
| Roboto Regular     | ~168 kB   | ~64 kB     | ~62 %     |
| Playfair Display   | ~230 kB   | ~85 kB     | ~63 %     |
| Noto Sans CJK      | ~15 MB    | ~7 MB      | ~53 %     |

WOFF2 browser support: all evergreen browsers (Chrome 36+, Firefox 39+, Safari 12+, Edge 14+). Serving TTF as a fallback is no longer necessary for any supported browser. The `format('woff2')` hint lets the browser skip the file entirely if it does not support the format.

---

## Self-Hosted vs CDN

| Factor                    | Self-Hosted                              | Google Fonts CDN                        |
|---------------------------|------------------------------------------|-----------------------------------------|
| Privacy                   | No third-party requests                  | IP logged by Google                     |
| GDPR risk                 | None                                     | Yes — German courts have issued fines   |
| Cache sharing             | No cross-site cache (post-2020 Chrome)   | No cross-site cache (post-2020)         |
| Control over subsetting   | Full                                     | Partial (via URL axis/range params)     |
| HTTP/2 push               | Possible                                 | Not under your control                  |
| Preload                   | Easy — you control the URL              | Possible but adds DNS lookup            |
| Font updates               | Manual                                   | Automatic (can be a footgun)            |
| Variable font support     | Any axis you include                     | Limited to axes Google exposes          |

Since Chrome 86 (October 2020) partitioned the HTTP cache by site origin, the "Google Fonts CDN cache hit" argument is no longer valid. Self-hosting is recommended for performance-critical applications.

Self-hosting workflow:
1. Download from Google Fonts or fontsource: `npm install @fontsource-variable/inter`
2. Subset with pyftsubset/glyphhanger
3. Serve from `/fonts/` with long `Cache-Control` headers (`max-age=31536000, immutable`)
4. Add preload hint in `<head>`
5. Use `font-display: swap` or `optional`

---

## Performance Checklist

- [ ] WOFF2 format only (no TTF/OTF served to modern browsers)
- [ ] Variable font replaces all weight/width variants
- [ ] Subset to actual characters used (unicode-range or manual)
- [ ] Self-hosted (avoids third-party DNS lookup)
- [ ] `<link rel="preload">` for above-the-fold fonts
- [ ] `font-display: swap` or `optional` set
- [ ] Long `Cache-Control` with content hash in filename
- [ ] `font-optical-sizing: auto` enabled where opsz axis exists

---

## References

- MDN font-display: https://developer.mozilla.org/en-US/docs/Web/CSS/@font-face/font-display
- web.dev font best practices: https://web.dev/font-best-practices/
- FontFace Observer: https://github.com/bramstein/fontfaceobserver
- glyphhanger: https://github.com/zachleat/glyphhanger
- fonttools / pyftsubset: https://github.com/fonttools/fonttools
- Fontsource (npm variable fonts): https://fontsource.org
