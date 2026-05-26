# Repository Status — Verified 2026-05-26

Activity classification:
- ✅ Active: commit within last 3 months
- ⚠️ Marginal: last commit 3-12 months ago
- 🔴 Dead: no commits > 12 months

All data verified against GitHub — 2026-05-26

## Core Three.js Stack

| Repo | Version | Last Release | Status |
|------|---------|-------------|--------|
| mrdoob/three.js | r184 (0.184.0) | 2026-04-16 | ✅ Active (daily commits) |
| pmndrs/react-three-fiber | v9.6.1 | 2026-04-28 | ✅ Active |
| pmndrs/drei | v10.7.7 | Nov 2025 (formal) | ✅ Active (code pushes) |
| pmndrs/detect-gpu | v5.x | 2026-05-24 | ✅ Active |
| darkroomengineering/lenis | v1.3.23 | 2026-04-15 | ✅ Active |

## Marginal (Verify Before Using)

| Repo | Last Push | Status | Note |
|------|----------|--------|------|
| pmndrs/gltfjsx | Nov 2024 | ⚠️ Marginal | CLI tool, may still work fine |
| RenaudRohlinger/r3f-perf | Dec 2024 | ⚠️ Marginal | Check R3F v9 compatibility |
| pmndrs/react-three-rapier | Nov 2025 | ⚠️ Marginal | 6 months inactive |
| pmndrs/leva | 2023 | ⚠️ Marginal | Replaced by lil-gui or custom UI |

## Dead Repos (Do Not Reference)

| Repo | Last Commit | Replacement |
|------|-------------|-------------|
| nicoptere/FBO | May 2021 | Use GPUComputationRenderer (built into Three.js) |
| luruke/awesome-casestudy | Sep 2022 | No direct replacement — curate manually |

## Stale Articles (URL Validity Issues)

| Source | Date | Issue |
|--------|------|-------|
| Evil Martians OffscreenCanvas article | 2019 | Browser compat claims outdated |
| Various "WebGPU coming soon" articles | 2020-2021 | WebGPU shipped; articles pre-date release |

**Rule:** Any article about WebGPU or OffscreenCanvas pre-2023 should be re-verified
against MDN and caniuse.com for current browser support.

## How to Verify Repo Status

```bash
# Check recent commits
gh api repos/pmndrs/drei/commits --jq '.[0].commit.author.date'

# Check latest release
gh api repos/pmndrs/drei/releases/latest --jq '.tag_name + " " + .published_at'

# Check open issues trend (proxy for activity)
gh api repos/pmndrs/drei/issues --jq 'length'
```

## Version Pinning Recommendation

For production projects, pin exact versions and verify compatibility:

```json
{
  "dependencies": {
    "three": "0.184.0",
    "@react-three/fiber": "9.6.1",
    "@react-three/drei": "10.7.7",
    "lenis": "1.3.23"
  }
}
```

Do NOT use `"three": "latest"` — breaking changes appear between minor versions.

## Sources
- All repos checked via GitHub API — 2026-05-26
- A24 validation agent — repo status scan
