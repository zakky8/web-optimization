# React 19 + React Three Fiber v9 Compatibility — Validation Record
**Validator:** Senior Research Analyst
**Date:** 2026-05-26
**Folder:** 47-validation-round2

---

## Summary

React Three Fiber v9 explicitly pairs with React 19. The peer dependency in
`package.json` is `"react": ">=19 <19.3"`. The R3F docs state the pairing rule
directly: "@react-three/fiber@9 pairs with react@19." No specific StrictMode
issues were found in the issue tracker matching this combination; no active
open issues flagging StrictMode breakage with v9.

---

## 1. React 19 + R3F v9 — Official Pairing

**Sources:**
- https://github.com/pmndrs/react-three-fiber/blob/master/docs/getting-started/introduction.mdx
  (tier 1, accessed 2026-05-26)
- https://github.com/pmndrs/react-three-fiber/blob/master/packages/fiber/package.json
  (tier 1, accessed 2026-05-26)

### From the introduction docs (exact quote):
> "@react-three/fiber@8 pairs with react@18, @react-three/fiber@9 pairs with react@19."

### From package.json (exact quote):
> "react": ">=19 <19.3"
> "react-dom": (optional peer, same constraint)
> "three": ">=0.156" (peer dependency)

**Finding:** VERIFIED — R3F v9 requires React 19. The upper bound `<19.3` is a hard
constraint as of the current package.json. React 18 users must stay on R3F v8.

```yaml
- claim: "R3F v9 requires React 19 (>=19 <19.3); R3F v8 is for React 18"
  exact_quote: "@react-three/fiber@8 pairs with react@18, @react-three/fiber@9 pairs with react@19."
  category: r3f.compatibility.react_version
  sources:
    - url: "https://github.com/pmndrs/react-three-fiber/blob/master/docs/getting-started/introduction.mdx"
      tier: 1
      accessed_at_utc: "2026-05-26T00:00:00Z"
      archive_url: null
      quote_location: "Compatibility section"
    - url: "https://github.com/pmndrs/react-three-fiber/blob/master/packages/fiber/package.json"
      tier: 1
      accessed_at_utc: "2026-05-26T00:00:00Z"
      archive_url: null
      quote_location: "peerDependencies.react"
  status: verified
  notes: "Upper bound <19.3 is current as of 2026-05-26; may change in future releases"
```

---

## 2. React 19 SubVersion Compatibility History

**Source:** https://github.com/pmndrs/react-three-fiber/releases (tier 1, accessed 2026-05-26)

Key releases affecting React 19 compatibility:

| Release | Date | Note |
|---|---|---|
| v9.0.0 | — | Initial React 19.0 pairing |
| v9.3.0 | Jul 28 (2025) | React Native 0.79+ support fixed; `flushSync` restored |
| v9.4.0 | Oct 13 (2025) | Improved dashed property name resolution |
| v9.4.1 | Nov 29 (2025) | React DevTools integration fix |
| v9.4.2 | Nov 29 (2025) | iOS simulator docs; Expo SDK 54 compatibility |
| v9.5.0 | Dec 30 (2025) | **React 19.2 compatibility added** |
| v9.6.0 | Apr 13 (2026) | Stable uniform references for ShaderMaterial |
| v9.6.1 | Apr 28 (2026) | Bug fix: interactivity state when swapping instances |

### v9.5.0 note (exact quote from releases page):
> "v9.5.0 added compatibility with React 19.2"
> "compatible with all versions of React between 19.0 and 19.2"

This indicates that React 19.2 required explicit reconciler-bundling changes in R3F
to maintain backward compatibility with 19.1.x.

```yaml
- claim: "R3F v9.5.0 added explicit React 19.2 compatibility"
  exact_quote: "v9.5.0 added compatibility with React 19.2"
  category: r3f.compatibility.react_19_2
  sources:
    - url: "https://github.com/pmndrs/react-three-fiber/releases"
      tier: 1
      accessed_at_utc: "2026-05-26T00:00:00Z"
      archive_url: null
      quote_location: "v9.5.0 release notes"
  status: verified
  notes: "React 19.2 required bundling the reconciler with R3F; this is a notable
          internal change but transparent to end users"
```

---

## 3. React 19 StrictMode Issues with R3F

**Sources:**
- https://github.com/pmndrs/react-three-fiber/issues?q=strict+mode+react+19
  (tier 1, accessed 2026-05-26)
- https://github.com/pmndrs/react-three-fiber/releases (tier 1, accessed 2026-05-26)

**Finding:** UNVERIFIED — No specific StrictMode breakage confirmed for the R3F v9 + React 19 combination.

The GitHub issue search for "strict mode react 19" returned three closed issues:
1. "Error when install dependencies" (closed Mar 31, 2025)
2. "Not working on nextjs projects" (closed Dec 10, 2025)
3. "Creating a basic shape and extruding" (closed Mar 12, 2019)

None of the issue titles directly indicate a StrictMode-specific regression. The release
notes for v9.0.0–v9.6.1 do not mention StrictMode fixes or known issues.

**Context:** React StrictMode double-invocation of effects (in development) is a known
general tension with custom renderers that manage their own lifecycle (R3F uses a custom
React reconciler). Historical issues existed with R3F v7/v8 + React 18 StrictMode around
double-mounting effects causing renderer state corruption. Whether this pattern recurs
in v9 + React 19 is not confirmed from available primary sources.

```yaml
- claim: "No confirmed StrictMode-specific breakage for R3F v9 + React 19 found in primary sources"
  exact_quote: null
  category: r3f.compatibility.strict_mode
  sources:
    - url: "https://github.com/pmndrs/react-three-fiber/issues?q=strict+mode+react+19"
      tier: 1
      accessed_at_utc: "2026-05-26T00:00:00Z"
      archive_url: null
      quote_location: "Issue search results — 3 closed, unrelated issues"
  status: unverified
  notes: "Absence of open issues is not confirmation of compatibility.
          Historical StrictMode tension with custom reconcilers is a risk factor.
          Recommend direct testing or checking R3F Discord for known workarounds."
```

---

## 4. Peer Dependency Summary

From `packages/fiber/package.json` (accessed 2026-05-26):

| Dependency | Constraint | Required/Optional |
|---|---|---|
| `react` | `>=19 <19.3` | Required |
| `react-dom` | `>=19 <19.3` | Optional |
| `three` | `>=0.156` | Required |
| `react-native` / Expo packages | version-specific | Optional |

---

## Overall Assessment

| Claim | Status |
|---|---|
| R3F v9 pairs with React 19 | VERIFIED |
| peer dep constraint is `>=19 <19.3` | VERIFIED |
| React 19.2 required explicit compatibility work in R3F | VERIFIED |
| React 19 StrictMode causes issues with R3F v9 | UNVERIFIED (no evidence found; not ruled out) |

**Sources used:**
- https://github.com/pmndrs/react-three-fiber/releases (tier 1, accessed 2026-05-26)
- https://github.com/pmndrs/react-three-fiber/blob/master/packages/fiber/package.json (tier 1, accessed 2026-05-26)
- https://github.com/pmndrs/react-three-fiber/blob/master/docs/getting-started/introduction.mdx (tier 1, accessed 2026-05-26)
- https://github.com/pmndrs/react-three-fiber/issues?q=strict+mode+react+19 (tier 1, accessed 2026-05-26)
