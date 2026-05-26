# Mobile Optimization Checklist

Use this specifically for mobile testing and deployment.

## GPU Detection & Tiering
- [ ] @pmndrs/detect-gpu implemented
- [ ] Tier 0 shows static fallback image (no WebGL)
- [ ] Tier 1 uses stripped settings (no shadows, no post-FX, low poly)
- [ ] Tier 2 uses reduced quality
- [ ] Tier 3 uses near-full quality
- [ ] isMobile flag used for all mobile-specific code paths

## Renderer Settings
- [ ] DPR capped at Math.min(devicePixelRatio, 1.5) - NEVER native 3x
- [ ] antialias: true on renderer (hardware MSAA - free on tile GPUs)
- [ ] No EffectComposer on mobile (direct renderer.render)
- [ ] THREE.Object3D.DEFAULT_MATRIX_AUTO_UPDATE = false

## Textures (iOS Critical)
- [ ] ALL textures in KTX2 format (mandatory for iOS 256MB budget)
- [ ] Max 1K textures on iOS (2K risky, 4K crashes)
- [ ] Total texture VRAM estimated to stay under 200MB for iOS
- [ ] KTX2Loader.detectSupport(renderer) called

## Geometry
- [ ] Lower-poly models served on mobile (1K verts max for LOD2)
- [ ] DRACO compression on all models
- [ ] Mobile-specific GLB files built (not same as desktop)

## Shaders
- [ ] precision highp float declared explicitly (not assumed)
- [ ] mediump used for color calculations only
- [ ] Varying variables under 3 on mobile (pack into vec4 if needed)
- [ ] No SSAO/DOF (requires OES_texture_float_linear, absent on iOS)

## Context Loss
- [ ] webglcontextlost event handler installed
- [ ] event.preventDefault() called in contextlost handler
- [ ] webglcontextrestored handler re-inits renderer and re-flags textures
- [ ] Chrome reload prompt for non-recoverable context loss

## Thermal & Battery
- [ ] 30fps cap implemented on mobile (TARGET_MS = 33.33)
- [ ] Rolling FPS monitor for thermal detection
- [ ] Low quality mode triggered at sustained < 25fps
- [ ] Render loop stops on document.hidden
- [ ] prefers-reduced-motion reduces quality

## Touch & Input
- [ ] touch-action: none on canvas
- [ ] Passive event listeners used (passive: true) where not calling preventDefault
- [ ] iOS bounce scroll prevented (overscroll-behavior: none or position:fixed body)
- [ ] Gyroscope gated behind user gesture (iOS 13+ requires requestPermission)
- [ ] -webkit-tap-highlight-color: transparent on canvas

## Viewport & Layout
- [ ] canvas uses height: 100svh (not 100vh)
- [ ] Resize handler only triggers on width changes (not address bar height changes)
- [ ] visualViewport used for accurate viewport size where supported
- [ ] No CLS on canvas load (explicit CSS dimensions)

## Network
- [ ] Network tier detected (navigator.connection) for Android
- [ ] saveData: true honored (low quality)
- [ ] Mobile serves smaller GLB files than desktop
- [ ] KTX2 transcoder (basis/) hosted and accessible

## Post-Processing
- [ ] ALL post-processing disabled on mobile: SSAO, DOF, bloom, SSR, SMAA
- [ ] Only tone mapping remains (single pass, renderer-level)

## Core Web Vitals
- [ ] LCP element is HTML (not canvas)
- [ ] INP < 200ms (OffscreenCanvas if heavy scene)
- [ ] CLS < 0.1 (canvas has explicit dimensions)

## Testing Checklist
- [ ] Tested on actual iPhone (not just Chrome DevTools mobile emulation)
- [ ] Tested on mid-range Android (not just flagship)
- [ ] Verified no crash after 60 seconds sustained (thermal test)
- [ ] Verified no tab crash on iOS (256MB texture budget check)
- [ ] Tested orientation change (portrait <-> landscape)
- [ ] Tested with address bar showing and hidden
- [ ] Tested on slow 3G connection (throttle in DevTools)
