# Desktop Optimization Checklist

Use this before every production deploy.

## Renderer Setup
- [ ] DPR capped at Math.min(devicePixelRatio, 2) - never above 2x
- [ ] alpha: false (unless transparency needed)
- [ ] stencil: false (unless stencil effects needed)
- [ ] powerPreference: 'high-performance'
- [ ] antialias: false (use FXAA post-pass instead)
- [ ] THREE.Object3D.DEFAULT_MATRIX_AUTO_UPDATE = false
- [ ] renderer.toneMapping = THREE.ACESFilmicToneMapping
- [ ] renderer.outputColorSpace = THREE.SRGBColorSpace

## Assets
- [ ] All textures converted to KTX2 (ETC1S for color, UASTC for normals)
- [ ] All geometry compressed with DRACO
- [ ] Textures are power-of-2 dimensions
- [ ] Mipmaps pre-generated
- [ ] No raw PNG/JPEG used as 3D textures in GPU

## Rendering Pipeline
- [ ] Draw calls under 100 per frame (verify in Spector.js)
- [ ] InstancedMesh used for repeated geometry (not individual meshes)
- [ ] Static geometries merged where possible
- [ ] LOD implemented for complex/large scenes
- [ ] Frustum culling enabled on all objects

## Shaders
- [ ] renderer.compileAsync() called behind loading screen
- [ ] if/else replaced with mix()/step() in fragment shaders
- [ ] No OES_texture_float_linear assumed - check extension availability
- [ ] Shader precision declared explicitly

## Lighting
- [ ] Static shadows baked to textures where possible
- [ ] Matcap textures used instead of real-time lights where possible
- [ ] Dynamic shadow map size appropriate (1024 standard, 2048 for hero)
- [ ] No more than 2-3 dynamic lights in scene

## Scroll & Animation
- [ ] Lenis initialized and synced with rAF loop
- [ ] GSAP quickTo used for mouse-tracking (not gsap.to per event)
- [ ] ScrollTrigger.refresh() called once after all triggers initialized
- [ ] gsap.matchMedia() used for responsive animation logic
- [ ] CSS scroll-driven animations used for simple reveals (not JS)

## Loading
- [ ] Draco decoder preloaded (dracoLoader.preload())
- [ ] Loading progress UI implemented
- [ ] Scene not revealed until compileAsync() resolves
- [ ] LCP element is real HTML (heading/image), not canvas
- [ ] Canvas has explicit CSS dimensions (no CLS on load)

## Post-Processing
- [ ] Effect composer used for post-processing
- [ ] FXAA pass is last pass in composer
- [ ] Bloom renders at 0.5x resolution minimum
- [ ] Post-processing disabled on mobile (separate code path)

## Performance Monitoring
- [ ] PerformanceMonitor implemented (R3F) or manual FPS tracking (Three.js)
- [ ] Render loop stops on document.hidden
- [ ] No memory leaks: renderer.dispose(), material.dispose(), geometry.dispose() on cleanup
