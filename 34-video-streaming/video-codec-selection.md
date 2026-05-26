# Video Codec Selection for Web and Three.js

**Last verified:** 2026-05-26  
**Primary source:** https://developer.mozilla.org/en-US/docs/Web/Media/Guides/Formats/Video_codecs

---

## Overview

Choosing the wrong codec for a video texture kills performance in two ways: (1) the browser decodes every frame on the CPU rather than GPU, burning battery and stealing cycles from the render loop, and (2) a large bitstream stresses network and memory bandwidth shared with WebGL draw calls. The decision matrix below covers the four codecs with significant web presence.

---

## Browser Support Table

Data sourced from MDN and caniuse.com, accessed 2026-05-26.

| Codec | Chrome | Firefox | Safari | Edge | Containers | Notes |
|---|---|---|---|---|---|---|
| **H.264 (AVC)** | All versions | All versions* | All versions | All versions | MP4, 3GP | *Firefox depends on OS codecs to avoid patent fees |
| **VP9** | All versions | All versions | All versions† | All versions | WebM, MP4, Ogg | †Safari does not support VP9 alpha channel |
| **AV1** | 70+ | 67+ | 17+ (hardware only‡) | 121+ | WebM, MP4, ISOBMFF | ‡M3 MacBook+, iPhone 15 Pro+, iPhone 16+ only |
| **HEVC (H.265)** | 107+ (HW-gated§) | 120+ (partial¶) | 11+ (macOS High Sierra+) | 18+ (extension needed∥) | MP4, QuickTime, MPEG-TS | Most complex support story |

**Footnotes:**
- §HEVC Chrome 107+: hardware decode required on Windows 8+ and Linux; all devices on macOS Big Sur 11+ and Android 5.0+
- ¶HEVC Firefox: Windows with HW/SW decode; macOS from Firefox 136+; Linux from Firefox 137+; Android 137+ hardware only
- ∥HEVC Edge: requires installing the free "HEVC Video Extensions" from Microsoft Store on Windows 10 1709+

---

## File Size Comparison (Same Perceptual Quality)

Relative bitrate required to achieve equivalent visual quality at 1080p:

| Codec | Relative bitrate | Example at 1080p |
|---|---|---|
| H.264 | 1.0× (baseline) | ~5,000 kbps |
| VP9 | ~0.5–0.6× | ~2,500–3,000 kbps |
| HEVC (H.265) | ~0.5× | ~2,500 kbps |
| AV1 | ~0.4–0.5× | ~2,000–2,500 kbps |

AV1 achieves up to 50% better compression than H.264 and approximately 30% better than VP9, at the cost of significantly slower encoding. HEVC is roughly equivalent to VP9 in compression efficiency but has the most fragmented decode support.

Source: https://uploadcare.com/blog/navigating-codec-landscapes/ and https://antmedia.io/video-codecs-streaming-guide/

---

## Hardware Decode Support

Hardware decode matters for Three.js scenes because the browser and GPU share memory bandwidth. Software decode competes with the render loop for CPU time.

| Codec | Desktop HW decode | Mobile HW decode |
|---|---|---|
| H.264 | Near universal (Intel QSV, AMD VCE, NVDEC, Apple VT) | Near universal |
| VP9 | Common (Intel Gen 8+, NVDEC Pascal+, AMD GCN 5+) | Common (Snapdragon 800+, Exynos 7+) |
| AV1 | Growing (Intel Gen 12+, NVDEC Ampere+, AMD RDNA 2+, Apple M3+) | Limited (Snapdragon 8 Gen 1+, Tensor G2+, Samsung Exynos 2200+) |
| HEVC | Common on macOS/iOS (Apple VT), patchy on Windows | Common on Apple; patchy elsewhere |

**ScientiaMobile measured roughly 10% of global devices with hardware AV1 decode in 2024.** Software AV1 decode (dav1d) works but increases CPU temperature and drains battery on long video loops.

---

## WebM Container

WebM is an open container (royalty-free) that mandates only:
- **Video:** VP8, VP9, or AV1
- **Audio:** Vorbis or Opus

It is not a catch-all web format. H.264 and HEVC cannot be placed in a WebM file. For Three.js texture use, WebM + VP9 is a strong choice because:
1. VP9 is hardware-decoded on virtually all non-Apple devices sold since 2016
2. VP9 supports an alpha channel (transparency), unlike H.264 and HEVC
3. File sizes are 30–50% smaller than H.264 at the same quality

---

## Alpha Channel (Transparency in Video)

| Codec | Alpha support | Container |
|---|---|---|
| VP9 | Yes | WebM |
| AV1 | Yes | WebM, MP4 |
| H.264 | No | — |
| HEVC | Yes (HEVC with alpha, Apple only) | MOV/MP4 |

For chroma-key effects in Three.js (green screen removed in shader), alpha is not needed — you remove the color in GLSL. But for pre-composited video with transparency (e.g., a character walking over a 3D scene with no green screen), VP9 in WebM or AV1 in WebM is the practical web-compatible option.

---

## Codec Selection for 3D Scene Backgrounds

**Decision tree:**

1. **Widest possible reach + no alpha needed** → H.264 in MP4. Works everywhere including old Android, Smart TVs, and game consoles. Largest file size.

2. **Good compression + wide reach + no alpha** → VP9 in WebM with H.264/MP4 fallback. 30–50% smaller files, hardware-decoded on most modern devices.

3. **Best compression, okay with Safari gaps** → AV1 in WebM or MP4. Provide VP9 or H.264 fallback for Safari < 17 / older Apple hardware.

4. **Pre-composited alpha video (character overlay)** → VP9 in WebM (best compat) or AV1 in WebM. Safari requires HEVC-with-alpha in MOV for Apple hardware — impractical for general web delivery.

5. **Apple-primary audience** → HEVC in MP4 with H.264 fallback.

**Recommended production source element:**

```html
<video autoplay muted loop playsinline crossorigin="anonymous">
  <source type="video/webm; codecs=av01,opus" src="background.av1.webm">
  <source type="video/webm; codecs=vp9,opus" src="background.vp9.webm">
  <source type="video/mp4"                   src="background.h264.mp4">
</video>
```

Browsers take the first `<source>` they can play. AV1 lands on Chrome/Firefox/Edge; VP9 on browsers that skip AV1; H.264 as the universal fallback including Safari where native AV1 hardware support is not present.

---

## Practical Encoder Settings for Web Backgrounds (1080p loop)

| Codec | Tool | Target bitrate | Notes |
|---|---|---|---|
| H.264 | ffmpeg `-c:v libx264` | 2,500–4,000 kbps | `-preset slow -crf 23` |
| VP9 | ffmpeg `-c:v libvpx-vp9` | 1,500–2,500 kbps | `-b:v 0 -crf 33 -speed 2` |
| AV1 | ffmpeg `-c:v libsvtav1` | 1,000–2,000 kbps | `-crf 35 -preset 6` (SVT-AV1 is fastest open encoder) |

For short loops (< 30 s) used as Three.js backgrounds, constant-quality (`-crf`) encoding is preferred over ABR — you get the smallest file at a target quality level rather than a fixed bitrate.

---

## Sources

- https://developer.mozilla.org/en-US/docs/Web/Media/Guides/Formats/Video_codecs — accessed 2026-05-26
- https://caniuse.com/av1 — AV1 support table, accessed 2026-05-26 (93.77% global coverage)
- https://uploadcare.com/blog/navigating-codec-landscapes/ — codec comparison 2025
- https://antmedia.io/video-codecs-streaming-guide/ — bitrate comparison table
- https://visionular.ai/av1-decoding-and-hardware-ecosystem-the-future-of-video-delivery/ — hardware decode landscape
