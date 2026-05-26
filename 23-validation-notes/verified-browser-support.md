# Verified Browser Support Tables

All data from MDN Web Docs and caniuse.com — accessed 2026-05-26

## WebGL

| Feature | Chrome | Firefox | Safari | iOS Safari |
|---------|--------|---------|--------|------------|
| WebGL 1.0 | 9+ | 4+ | 5.1+ | 8+ |
| WebGL 2.0 | 56+ | 51+ | 15+ | 15+ |
| OES_texture_float | 17+ | 22+ | 15+ | 15+ |
| WEBGL_draw_buffers (MRT) | 36+ | 28+ | 15+ | 15+ |
| EXT_color_buffer_float | 56+ | 51+ | 15+ | 15+ |

## WebGPU

| Browser | Version | Status |
|---------|---------|--------|
| Chrome / Edge | 113+ | Stable |
| Chrome Android | 121+ | Stable |
| Safari | 26+ (2026) | Stable |
| iOS Safari | 26+ | Stable |
| Firefox | 141+ | Stable |

Source: caniuse.com/webgpu — verified 2026-05-26

## CSS Animations & Layout

| Feature | Chrome | Firefox | Safari |
|---------|--------|---------|--------|
| View Transitions (SPA) | 111+ | 134+ | 18+ |
| View Transitions (MPA) | 126+ | ❌ | ❌ |
| scroll() timeline | 115+ | 110+ | 15.4+ |
| view() timeline | 115+ | 114+ | 15.4+ |
| @starting-style | 117+ | 129+ | 17.5+ |
| transition-behavior | 117+ | 129+ | 17.4+ |
| @property | 78+ | 128+ | 16.4+ |
| CSS Paint (Houdini) | 65+ | ❌ | ❌ |
| content-visibility: auto | 85+ | 109+ | 18+ |

## Workers & Concurrency

| Feature | Chrome | Firefox | Safari |
|---------|--------|---------|--------|
| Web Workers | 4+ | 3.5+ | 4+ |
| Module Workers | 80+ | 114+ | 15+ |
| SharedArrayBuffer | 68+ (COOP) | 79+ (COOP) | 15.2+ |
| Atomics | 68+ | 78+ | 15.2+ |
| OffscreenCanvas | 69+ | 105+ | 16.4+ |
| OffscreenCanvas + 3D ctx | 69+ | 105+ | 16.4+ |

## Media & XR

| Feature | Chrome | Firefox | Safari |
|---------|--------|---------|--------|
| WebXR (VR) | 79+ | 98+ | ❌ (WebXR Viewer app) |
| WebXR (AR) | 81+ | ❌ | ❌ |
| Web Audio API | 14+ | 25+ | 6+ |
| MediaStream API | 21+ | 17+ | 11+ |

## Performance APIs

| Feature | Chrome | Firefox | Safari |
|---------|--------|---------|--------|
| PerformanceObserver | 52+ | 57+ | 11+ |
| LayoutShift entry | 77+ | ❌ | ❌ |
| Navigation Timing L2 | 57+ | 58+ | 15+ |
| Long Tasks API | 58+ | ❌ | ❌ |
| Element Timing | 77+ | ❌ | ❌ |

## Notes

- "❌" = Not supported as of 2026-05-26
- COOP = Requires `Cross-Origin-Opener-Policy: same-origin` header
- iOS Safari typically matches desktop Safari version within same cycle
- Firefox WebXR: available in Firefox Reality (discontinued), not desktop Firefox

## Sources
- MDN Browser Compatibility: https://developer.mozilla.org/en-US/docs/MDN/Writing_guidelines/Page_structures/Compatibility_tables
- caniuse.com: https://caniuse.com/
- WebGPU status: https://chromestatus.com/feature/6213121689518080
