# WebXR Browser Support — Validation Record
**Validator:** Senior Research Analyst
**Date:** 2026-05-26
**Folder:** 47-validation-round2

---

## Summary

WebXR Device API support is fragmented. Chromium-based browsers lead; Apple/Safari has
selective module-level positions but no shipped general WebXR support in mainstream Safari;
Firefox desktop has support disabled by default; Meta Quest Browser is the most
feature-complete implementation.

---

## 1. Chrome Android

### Finding
- **Status:** VERIFIED — Partial support
- **First supported version:** Chrome for Android 79 (desktop Chrome 79 was the initial
  WebXR launch). AR Module support on Android arrived at Chrome for Android 81.
- **Current tested version:** Chrome for Android 148 (as of caniuse data fetched 2026-05-26)
- **Support type:** Marked "partial" — core VR/AR Device API supported; specific optional
  modules (Depth Sensing, Light Estimation, Anchors) require later versions

### Version ladder (from immersiveweb.dev, accessed 2026-05-26)
| Feature | Chrome Android minimum |
|---|---|
| WebXR Core | 79 |
| AR Module | 81 |
| Anchors | 85 |
| Depth Sensing / Light Estimation | 90 |

### Device notes
- Requires ARCore-compatible device for AR sessions on Android
- VR (inline) sessions work on more devices without ARCore
- No device whitelist published by Google; ARCore device list is the practical gate

```yaml
- claim: "Chrome Android supports WebXR Core from v79, AR Module from v81"
  exact_quote: null
  category: webxr.browser_support.chrome_android
  sources:
    - url: "https://immersiveweb.dev/"
      tier: 2
      accessed_at_utc: "2026-05-26T00:00:00Z"
      archive_url: null
      quote_location: "compatibility table"
    - url: "https://caniuse.com/webxr"
      tier: 2
      accessed_at_utc: "2026-05-26T00:00:00Z"
      archive_url: null
      quote_location: "browser support table"
  status: verified
  notes: "caniuse shows latest tested as v148 with partial support; immersiveweb.dev gives per-feature version numbers"
```

---

## 2. Apple / Safari WebXR Status

### Finding
- **Status:** VERIFIED — No general WebXR support in shipping Safari; selective module
  positions at the standards level
- **webkit.org/status/:** Page retired; redirects to MDN/caniuse for support data
- **WebKit Standards Positions (github.com/WebKit/standards-positions, accessed 2026-05-26):**

| Module | WebKit Position | Issue |
|---|---|---|
| WebXR Device API (core) | Under consideration | #155 (open) |
| WebXR Hand Input Module | Supported (shipped) | #395 (closed) |
| WebXR Layers API | Support | #601 (closed) |
| WebXR Depth Sensing Module | Under consideration with concerns | #503 (open) |
| WebXR Raw Camera Access API | **Oppose** (privacy) | #37 (closed) |
| WebXR Plane Detection Module | Under consideration | #608 (open) |

- **caniuse.com/webxr** records: Safari desktop — "Disabled by default" across versions
  3.1–26.5+. Safari on iOS — "Not supported" across all versions through 26.5.
- **immersiveweb.dev** notes: "Hand Input support available in Safari 15.1+" and references
  an "iOS WebXR Viewer app" as an alternative path on iPhone 6s and newer.

### Key Contradiction
caniuse shows Safari as "not supported" / "disabled" while WebKit standards positions show
some modules as having a "support" position. The reconciliation: WebKit has shipped hand
input and layers APIs within the visionOS/Vision Pro context, not in mainstream iOS/macOS
Safari for web developers. These positions do not translate to a usable `navigator.xr` API
in shipping Safari 18.x on iOS/macOS.

```yaml
- claim: "Safari does not ship WebXR Device API for general web use as of 2026-05-26"
  exact_quote: null
  category: webxr.browser_support.safari
  sources:
    - url: "https://caniuse.com/webxr"
      tier: 2
      accessed_at_utc: "2026-05-26T00:00:00Z"
      archive_url: null
      quote_location: "Safari row — 'Disabled by default' / iOS Safari 'Not supported'"
    - url: "https://github.com/WebKit/standards-positions/issues?q=webxr"
      tier: 1
      accessed_at_utc: "2026-05-26T00:00:00Z"
      archive_url: null
      quote_location: "issue #155 (core) open; issue #37 (camera) opposed"
  status: verified
  notes: "Contradiction logged: some modules have support position but do not ship in
          mainstream Safari. Hand Input shipped in Apple platforms but not general web Safari."
```

---

## 3. Firefox Desktop WebXR Status

### Finding
- **Status:** VERIFIED — Disabled by default in all desktop versions

caniuse data (accessed 2026-05-26): Firefox desktop versions 2–153 listed as
"Disabled by default". Firefox for Android v150 also "Disabled by default".

Firefox implemented WebXR behind the `dom.vr.webxr.enabled` flag for testing purposes
but has not shipped it as on-by-default in any release as of the data fetch date.

```yaml
- claim: "Firefox desktop WebXR is disabled by default in all versions through v153"
  exact_quote: null
  category: webxr.browser_support.firefox_desktop
  sources:
    - url: "https://caniuse.com/webxr"
      tier: 2
      accessed_at_utc: "2026-05-26T00:00:00Z"
      archive_url: null
      quote_location: "Firefox row — 'Disabled by default' versions 2–153"
  status: verified
  notes: "Can be enabled via about:config flag for development/testing only"
```

---

## 4. Meta Quest Browser

### Finding
- **Status:** VERIFIED — Most feature-complete shipping WebXR implementation

Source: immersiveweb.dev compatibility table (accessed 2026-05-26)

| Feature | Meta Quest Browser minimum version |
|---|---|
| WebXR Core | 7.0 (December 2019) |
| AR Module | 24.0 (October 2022) |
| Gamepads Module | Supported |
| Anchors | Supported |
| Hand Input | Supported |

```yaml
- claim: "Meta Quest Browser supports WebXR Core from v7.0 and AR Module from v24.0"
  exact_quote: null
  category: webxr.browser_support.meta_quest
  sources:
    - url: "https://immersiveweb.dev/"
      tier: 2
      accessed_at_utc: "2026-05-26T00:00:00Z"
      archive_url: null
      quote_location: "compatibility table — Meta Quest Browser row"
  status: verified
  notes: "Chromium-based; closely tracks Chrome WebXR support"
```

---

## Overall Assessment

| Browser | Status | Notes |
|---|---|---|
| Chrome Android | Partial (VERIFIED) | Core from v79, AR from v81, full feature ladder to v90 |
| Safari iOS/macOS | Not supported (VERIFIED) | Disabled by default; standards positions mixed |
| Firefox desktop | Disabled by default (VERIFIED) | Flag available for testing only |
| Meta Quest Browser | Full support (VERIFIED) | Most complete shipping implementation |

**Sources used:**
- https://caniuse.com/webxr (tier 2, accessed 2026-05-26)
- https://immersiveweb.dev/ (tier 2, accessed 2026-05-26)
- https://developer.mozilla.org/en-US/docs/Web/API/WebXR_Device_API (tier 2, accessed 2026-05-26)
- https://github.com/WebKit/standards-positions/issues?q=webxr (tier 1, accessed 2026-05-26)
