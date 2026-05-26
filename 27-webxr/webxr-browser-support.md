# WebXR Browser & Device Support

**Last verified:** 2026-05-26  
**Sources:** immersiveweb.dev (accessed 2026-05-26), caniuse.com/webxr (accessed 2026-05-26), testmuai.com WebXR Safari article (accessed 2026-05-26), MDN WebXR compatibility data (accessed 2026-05-26)

---

## Global Usage

caniuse.com reports **75.38% global browser usage share** with partial WebXR support as of 2026-05-26. "Partial" means the base `WebXR Device API` is present; advanced modules (hit-test, hand-input, layers) have narrower support.

---

## Desktop Browsers

| Browser | WebXR Core | Notes |
|---------|------------|-------|
| Chrome 79+ | Partial | First stable desktop WebXR support |
| Edge 79+ | Partial | Chromium-based; mirrors Chrome support |
| Edge 91 (HoloLens 2) | Partial | HoloLens-specific build |
| Opera 66+ | Partial | Chromium-based |
| Firefox (all versions) | Disabled by default | Never enabled in stable; requires about:config flag `dom.vr.webxr.enabled` — treat as NOT supported in production |
| Safari (macOS, all versions) | Not supported | Apple has not implemented WebXR on macOS Safari as of 2026-05-26 |
| Internet Explorer | Not supported | EOL; irrelevant |

---

## Mobile Browsers

| Browser | WebXR Core | AR (`immersive-ar`) | Notes |
|---------|------------|---------------------|-------|
| Chrome Android 81+ | Supported | Supported | The primary target for handheld AR experiences |
| Samsung Internet 12.0+ | Partial | Partial (14.2+ for DOM Overlay) | Galaxy devices; good real-world coverage |
| Opera Mobile 80+ | Partial | — | Chromium-based |
| Firefox for Android | Disabled by default | Disabled | Same situation as desktop Firefox |
| Safari on iOS (all versions) | **Not supported** | **Not supported** | Confirmed absent on iOS/iPadOS as of 2026-05-26 |
| UC Browser, QQ Browser, Baidu Browser | Not supported | Not supported | — |

---

## Dedicated XR Headset Browsers

| Browser | Version | WebXR Core | Notes |
|---------|---------|------------|-------|
| **Meta Quest Browser** | 7.0+ (Dec 2019) | Supported | Most widely used WebXR VR/AR target; supports hit-test, hand-input, layers, anchors on Quest 2/3/Pro |
| **Wolvic** | 0.9.3+ (Feb 2022) | Supported | Open-source browser for head-mounted displays; Firefox Reality successor |
| **PICO Browser** | 3.3.32+ | Partial | ByteDance's PICO headsets |
| **Microsoft Edge (HoloLens 2)** | 91+ | Partial | Supports `immersive-ar` for HoloLens passthrough |

---

## Safari: Detailed Status (VERIFIED)

This is the most frequently misunderstood area. Status as of 2026-05-26:

| Platform | Safari WebXR status |
|----------|---------------------|
| macOS (all versions) | **Not implemented.** No flag exists. |
| iOS (iPhone/iPad, all versions) | **Not implemented.** |
| iPadOS | **Not implemented.** |
| visionOS 2.0 (Apple Vision Pro) | **Partial.** `immersive-vr` works by default. `immersive-ar` module is **not enabled** — AR sessions will fail. |

Source: testmuai.com confirmed "Safari does not implement WebXR on macOS, iOS, or iPadOS" (tier 3 aggregator, corroborated by caniuse.com tier 3 data showing all Safari versions as unsupported).

**Verdict:** Do not target Safari for any WebXR experience unless you are specifically building for Apple Vision Pro VR-only. iOS users require a native app (ARKit) or a third-party WebAR platform (8th Wall, Niantic) instead.

---

## Module-Level Support (Advanced Features)

Support for individual WebXR modules is narrower than base API support:

| Feature | Chrome Android | Quest Browser | Samsung Internet | Safari visionOS |
|---------|---------------|---------------|-----------------|-----------------|
| `immersive-vr` | Yes | Yes | Yes (12.0+) | VR only |
| `immersive-ar` | Yes | Yes | Yes | No |
| Hit Test | Yes | Yes | Yes | No |
| Hand Input | Chrome 131+ | Yes | Partial | Partial (15.1+) |
| DOM Overlays | Yes | Yes | 14.2+ | No |
| Layers API | Partial | Yes | Partial | Partial |
| Anchors | Partial | Yes | No | No |
| Depth Sensing | No | Yes (Quest 3) | No | No |
| Mesh Detection | No | Yes (Quest 3) | No | No |

---

## Firefox: Detailed Status

Firefox has never shipped WebXR in a stable release. The feature has been behind `dom.vr.webxr.enabled` in `about:config` for years but is never on by default. As of 2026-05-26:

- Firefox desktop: disabled
- Firefox for Android (150+): disabled
- Mozilla's standalone **WebXR Viewer** app for iOS (separate browser): provides AR via ARKit on iPhone, but this is a separate download — not the system browser.

**Verdict:** Do not target Firefox for WebXR without a polyfill or explicit user opt-in.

---

## Polyfills

### webxr-polyfill (immersive-web)

Maintained by the W3C Immersive Web Community Group.

```bash
npm install webxr-polyfill
# or CDN:
# https://cdn.jsdelivr.net/npm/webxr-polyfill@latest/build/webxr-polyfill.js
```

```js
import WebXRPolyfill from 'webxr-polyfill';
const polyfill = new WebXRPolyfill();
// Must be initialized before any navigator.xr usage
```

**What it covers:**
- Provides `navigator.xr` shim on browsers without it
- Falls back to WebVR 1.1 on legacy headset browsers
- Provides basic mobile AR via DeviceMotion/DeviceOrientation where WebXR is absent

**What it does NOT cover:**
- Cannot add `immersive-ar` to iOS Safari (no ARKit bridge from a web polyfill)
- Hit testing, hand input, mesh detection — these require hardware drivers; no JS polyfill can emulate them
- Latest version on npm: `2.0.3` (published approximately 2020; minimal recent maintenance activity)

**Repository:** `github.com/immersive-web/webxr-polyfill`

### webxr-layers-polyfill

Separate polyfill for the XR Layers API. Available at `github.com/immersive-web/webxr-layers-polyfill`.

---

## Checking Support at Runtime

```js
// Check if ANY XR is available
if (!navigator.xr) {
  // No WebXR support — show fallback
  return;
}

// Check specific session type before showing button
const vrSupported = await navigator.xr.isSessionSupported('immersive-vr');
const arSupported = await navigator.xr.isSessionSupported('immersive-ar');

if (arSupported) {
  // show AR button
} else if (vrSupported) {
  // show VR button
} else {
  // show inline / fallback experience
}
```

`VRButton.createButton()` and `ARButton.createButton()` call `isSessionSupported` internally and update their disabled/label state automatically — you do not need to do this manually when using those helpers.

---

## Sources

- [immersiveweb.dev device support table](https://immersiveweb.dev/) — accessed 2026-05-26, tier 2 (W3C Immersive Web Community Group)
- [caniuse.com/webxr](https://caniuse.com/webxr) — accessed 2026-05-26, tier 3 (75.38% global usage, all Safari versions listed as unsupported)
- [TestMu AI WebXR Safari browser compatibility](https://www.testmuai.com/web-technologies/webxr-safari/) — accessed 2026-05-26, tier 3 (corroborates no iOS/macOS support, partial visionOS)
- [MDN WebXR Device API browser compat](https://developer.mozilla.org/en-US/docs/Web/API/WebXR_Device_API) — accessed 2026-05-26, tier 1 ("not Baseline because it does not work in some widely-used browsers")
- [webxr-polyfill npm page](https://www.npmjs.com/package/webxr-polyfill) — accessed 2026-05-26, tier 2 (version 2.0.3 confirmed)
