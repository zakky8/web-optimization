# Project Context: web-optimization

## What This Repo Is

A structured knowledge base for building top-1% 3D and animation websites with Three.js, WebGL,
WebGPU, and related tools. 48 topic folders, each containing 3-7 markdown files with validated
technical data, code patterns, and sourced benchmarks.

## Folder Naming Convention

```
NN-topic-name/
  NN = zero-padded number (01, 02, ... 48)
  Files inside: descriptive-kebab-case.md
```

Folders 01-23: original reference set (foundations through animation QA)
Folders 24-47: advanced topics (TypeScript, Next.js 15, WebXR, post-processing, etc.)
Folder 48: GPGPU particles (renamed from 06 to resolve prefix conflict)

## Content Standard

Every claim in this repo follows this hierarchy:
1. Primary source first (official docs, changelogs, browser specs, live repos)
2. Independent journalism second (FinanceMagnates, MDN, Chrome Status)
3. Community signal last (Reddit, Discord) — always labeled UNVERIFIED if not corroborated

**Never paraphrase a rule — quote it verbatim and cite the URL.**
**Never use training-data memory for version numbers — always re-fetch.**

## Key Validated Facts (2026-05-26)

- three.js: r184 (0.184.0) — current stable
- react-three-fiber: v9.6.1 — requires React 19; v8 is incompatible with React 19
- GSAP: 3.x — ALL plugins now free (DrawSVG, MorphSVG, SplitText, MotionPath)
- Lenis: v1.3.23 — darkroomengineering/lenis
- PCFSoftShadowMap: DEPRECATED in r168 — silently redirects to PCFShadowMap
- SMIL: NOT deprecated — Chrome reversed 2015 intent; fully supported 2026
- Safari WebXR: NO support on macOS, iOS, iPadOS
- Astro 6.3.8: <ViewTransitions /> removed; use <ClientRouter /> from astro:transitions
- camera-controls: v3.1.2 (yomotsu/camera-controls)
- nicktindall/three-csm: 404 (does not exist) — use Three.js built-in CSM addon

## When Adding New Content

1. Pick the right folder — check existing folder list before creating a new one
2. One file per sub-topic (3-7 files per folder is ideal)
3. Include: verified version numbers, direct quotes from docs, archive.org URLs where helpful
4. Flag any UNVERIFIED claims explicitly
5. Add an entry to 47-validation-round2/ if correcting a widespread mistake

## File Template

```markdown
# Topic Name

## Overview
One paragraph summary with the primary use case.

## Key API / Pattern
Code snippet with comments.

## Performance Notes
Numbers with sources.

## Gotchas / Corrections
Common mistakes and the verified truth.

## Sources
- URL — what it confirms — accessed 2026-05-26
```

## Repo Structure at a Glance

| Range | Theme |
|-------|-------|
| 01-05 | Foundations (3D pipeline, mobile, metrics, resources, checklists) |
| 06-07 | GPU & Shaders (WebGPU, GLSL) |
| 08-09 | Build & Pipeline (physics, Vite, compression) |
| 10-15 | Frameworks & Rendering (R3F, transitions, a11y, caching, formats) |
| 16-19 | Concurrency, Memory, CSS, WASM |
| 20-23 | Three.js internals, OSS repos, animation, validation |
| 24-47 | Advanced topics (TS, Next.js, post-FX, XR, audio, typography, tooling, etc.) |
| 48 | GPGPU particles (compute shaders, ping-pong buffers) |
