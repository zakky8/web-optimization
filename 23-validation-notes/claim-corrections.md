# Validated Claim Corrections

All corrections verified by A23 validation agent — 2026-05-26
These replace inaccurate claims that circulate in training data, blog posts, and docs.

## iOS Canvas Memory Limit

### Claim (Circulating)
> "iOS has a hard 256MB canvas memory limit"

### Corrected Finding
The 256MB figure originated from older WebKit documentation.
The actual limit is device-RAM-dependent, typically `RAM / 4`.
Recent WebKit (Safari 17+/iOS 17+) may have removed or raised this limit.

**Status:** UNVERIFIED — exact current limit requires device-specific testing.
**Safe practice:** Keep total VRAM under 256MB for broad iOS compatibility.
**Test with:** Safari Web Inspector → Memory profiler on target device.

Source: WebKit source + community reports — not official Apple documentation.

---

## Draco Geometry Compression

### Claim (Circulating)
> "Draco compresses geometry by 50-70%"

### Corrected Finding
Official gltf-transform documentation states **70-98%** reduction in mesh data size.
The lower bound 50% is incorrect; Draco is more effective than that.

**Exact quote from gltf-transform docs:**
> "Draco mesh compression, reducing geometry size 70-98%."

Source: https://gltf-transform.dev/cli — accessed 2026-05-26
Status: VERIFIED

---

## KTX2 VRAM Reduction

### Claim (Circulating)
> "KTX2 reduces VRAM usage by 8x"

### Corrected Finding
KTX2 texture compression reduces GPU-side texture memory by **4-8×** depending on format:
- ETC1S: ~6-8× vs uncompressed
- UASTC: ~4-5× vs uncompressed
- PNG (uncompressed on GPU): baseline

"8x" is a best-case figure. The correct range is **4-8x**.

Status: VERIFIED (range)
Source: Khronos KTX specification, gltf-transform docs

---

## gltfjsx Size Reduction

### Claim (Circulating)
> "gltfjsx reduces 3D scene size by 90%"

### Corrected Finding
Official pmndrs/gltfjsx README states:

> "gltfjsx reduces 3D scenes by **70-90%**"

The upper bound "90%" is technically accurate but misleading without the lower bound.
Actual reduction depends on scene complexity, geometry duplication, and instance count.

Source: https://github.com/pmndrs/gltfjsx — README — accessed 2026-05-26
Status: VERIFIED (range: 70-90%)

---

## DPR Cap Values

### Claim (Circulating)
> "Cap devicePixelRatio at 2.0 on mobile, 1.5 on mid-range devices"

### Corrected Finding
These values are **community convention**, not official specifications.
There is no W3C or browser spec mandating 2.0 or 1.5 as DPR caps.

Common practice comes from real-world benchmarks:
- DPR 2.0: fine for flagship phones (iPhone 15 Pro, Pixel 8)
- DPR 1.5: recommended for mid-range (most Android devices)
- DPR > 2: visible difference is negligible, GPU cost is not

**Status:** UNVERIFIED as official spec. Community-validated as best practice.

```js
// The Three.js/R3F convention
renderer.setPixelRatio(Math.min(window.devicePixelRatio, 2));
// "2" is a judgment call, not a spec
```

---

## Sources
- A23 validation agent research — 2026-05-26
- gltf-transform: https://gltf-transform.dev/cli
- gltfjsx README: https://github.com/pmndrs/gltfjsx
