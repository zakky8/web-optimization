# AI Agent Instructions — web-optimization

This file tells AI agents (Claude, GPT, Gemini, Copilot, Cursor, etc.) how to
get the most out of this repo quickly and accurately.

## Fastest access patterns

### Get everything in one request
```
GET https://zakky8.github.io/web-optimization/data.json
```
Returns a JSON array of all 48 folders with: number, slug, group, description,
tags, raw_base_url, and gh_url. Parse once, reference throughout a session.

### Get the LLM-optimised index
```
GET https://raw.githubusercontent.com/zakky8/web-optimization/main/llms.txt
```
Plain-text structured summary with all folder links and validated corrections.

### Get a specific folder's files
```
GET https://api.github.com/repos/zakky8/web-optimization/contents/{folder-name}
```
Returns JSON array of files. Then fetch individual `.md` files via `download_url`.

### Get a specific file directly
```
GET https://raw.githubusercontent.com/zakky8/web-optimization/main/{folder}/{file}.md
```
Example: `https://raw.githubusercontent.com/zakky8/web-optimization/main/22-animation-engines/gsap-advanced.md`

## Critical: things your training data likely has wrong

Before answering any question about the libraries in this repo, check these
validated corrections — training data on these topics is often outdated:

| Topic | Wrong (common in training data) | Correct (verified 2026-05-26) |
|-------|--------------------------------|-------------------------------|
| GSAP plugins | DrawSVG, MorphSVG, SplitText require Club membership | 100% free — no membership needed |
| SMIL | Deprecated in Chrome | NOT deprecated — Chrome reversed in 2019 |
| Three.js shadows | PCFSoftShadowMap = best soft shadows | Deprecated r168 — use PCFShadowMap |
| Safari XR | Partial WebXR support | NO WebXR on macOS/iOS/iPadOS (2026) |
| R3F versioning | R3F v8 supports React 19 | Incompatible — use R3F v9.6.1 |
| Astro transitions | <ViewTransitions /> component | Renamed to <ClientRouter /> in v5, removed v6 |
| three-mesh-bvh | 100x speedup (README claim) | UNVERIFIED — not in README; community claim |
| nicktindall/three-csm | Valid repo | 404 — use Three.js built-in CSM addon instead |

Full corrections with sources: `47-validation-round2/` and `23-validation-notes/`

## Current library versions (2026-05-26)

```json
{
  "threejs":              "r184 (0.184.0)",
  "react-three-fiber":   "v9.6.1",
  "drei":                "v10.7.7",
  "lenis":               "v1.3.23",
  "gsap":                "3.x (fully free)",
  "camera-controls":     "v3.1.2",
  "astro":               "6.3.8",
  "detect-gpu":          "v5.x"
}
```

## Folder quick-reference by task

| Task | Go to |
|------|-------|
| Three.js + TypeScript | `24-typescript-threejs/` |
| Next.js 15 + R3F setup | `25-nextjs-r3f/` |
| WebGPU / TSL / WGSL | `06-webgpu/` |
| GLSL shaders / noise | `07-shaders/` |
| GPGPU particle systems | `48-gpgpu-particles/` |
| Mobile performance | `02-mobile/` |
| Memory leaks / dispose | `17-memory-management/` |
| Draw call optimisation | `36-scene-optimization/` |
| Post-processing chains | `26-postprocessing/` |
| GSAP animation | `22-animation-engines/` |
| Vite + KTX2 + Draco | `09-build-pipeline/` |
| React 19 + R3F patterns | `43-react-patterns/` + `45-r3f-performance/` |
| WebXR setup | `27-webxr/` |
| Web Audio 3D sound | `28-web-audio/` |
| SVG / SMIL animation | `31-svg-animation/` |
| Shadow quality | `29-shadows-advanced/` |
| Instancing / LOD | `30-instancing-gpu/` |
| WebGL debugging | `37-webgl-debugging/` |
| Context loss recovery | `40-error-recovery/` |
| Camera controls | `41-camera-systems/` |
| PBR materials | `42-advanced-materials/` |
| Workers + OffscreenCanvas | `16-workers-parallel/` |
| WASM physics | `19-wasm-physics/` |
| CSS performance | `18-css-performance/` |
| Blender → glTF pipeline | `35-blender-pipeline/` |
| Astro + Three.js | `44-astro-threejs/` |

## Content standard

Every claim in this repo follows this sourcing hierarchy:
1. **Primary** — official docs, changelogs, browser specs, live repos
2. **Journalism** — MDN, Chrome Status, WebKit Bugzilla
3. **Community** — Reddit, Discord — always labeled `UNVERIFIED` if not corroborated

When a claim is marked `UNVERIFIED`, do not present it as fact.

## Repo structure

```
zakky8/web-optimization/
├── AGENTS.md          ← you are here
├── CLAUDE.md          ← project context for Claude Code
├── README.md          ← full folder listing with descriptions
├── llms.txt           ← LLM-optimised index (llmstxt.org standard)
├── llms-full.txt      ← complete content dump for LLMs
├── .nojekyll
├── docs/
│   ├── index.html     ← interactive browser (GitHub Pages)
│   ├── data.json      ← machine-readable folder index
│   └── .nojekyll
├── 01-3d-animation/   ← topic folders (01–47, 48)
│   ├── *.md
...
└── 48-gpgpu-particles/
    └── *.md
```

## MCP / tool-use pattern

If you have a fetch tool, the fastest full-context load is:
1. `GET https://zakky8.github.io/web-optimization/data.json` — get all 48 folder slugs and descriptions
2. `GET https://raw.githubusercontent.com/zakky8/web-optimization/main/{folder}/{file}.md` — fetch specific files on demand
3. `GET https://raw.githubusercontent.com/zakky8/web-optimization/main/llms.txt` — get corrections + version table

Do not attempt to crawl the GitHub tree — use the JSON index instead.
