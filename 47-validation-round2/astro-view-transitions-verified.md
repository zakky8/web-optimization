# Astro View Transitions — Validation Record
**Validator:** Senior Research Analyst
**Date:** 2026-05-26
**Folder:** 47-validation-round2

---

## Summary

Astro's current stable version as of 2026-05-26 is **6.3.8** (released same day). A 5.x
maintenance branch (5.18.2) also released same day. The view transitions feature uses
`<ClientRouter />` and `document.startViewTransition`. The `transition:persist` directive
does NOT reliably preserve canvas elements — a known Safari 18 bug caused WebGL context
loss on persisted canvas elements; a fix was merged March 2026 (PR #15728). The docs do
not document canvas-specific behavior, leaving this an area of caution. Known hard
limitations: CSS animations restart and iframes reload even with `transition:persist`.

---

## 1. Current Astro Version

**Source:** https://github.com/withastro/astro/releases (tier 1, accessed 2026-05-26)

| Branch | Latest release | Released |
|---|---|---|
| Astro 6.x (current stable) | **6.3.8** | May 26, 2026 |
| Astro 5.x (maintenance) | 5.18.2 | May 26, 2026 |

Recent 6.x releases leading up to current:
- 6.3.8 (May 26): fixes 404s with dynamically imported JS chunks, misleading error messages,
  scoped styles in MDX, missing CSS for Svelte components, false-positive cache warnings
- 6.3.7 (May 21): fixes request body handling in Node adapter, Vite build option forwarding,
  custom elements in MDX, dynamic route URL encoding, i18n import issues
- 6.3.6 (May 20): markdown image alt text, HMR stale content, SVG processing, content
  collection JSON schema generation

```yaml
- claim: "Current Astro stable version is 6.3.8 as of 2026-05-26"
  exact_quote: null
  category: astro.version.current
  sources:
    - url: "https://github.com/withastro/astro/releases"
      tier: 1
      accessed_at_utc: "2026-05-26T00:00:00Z"
      archive_url: null
      quote_location: "Latest releases list"
  status: verified
  notes: "Astro 5.18.2 is the current maintenance release for the v5 branch"
```

---

## 2. How View Transitions Work (Current Docs)

**Source:** https://docs.astro.build/en/guides/view-transitions/ (tier 1, accessed 2026-05-26)

Astro's view transitions operate via the `<ClientRouter />` component (client-side routing).
The navigation lifecycle:
1. **Preparation** — router fetches next page content
2. **DOM Swap** — old page content replaced with new content; `document.startViewTransition`
   called to trigger browser animations
3. **Completion** — scripts execute, loading finishes

The router adds `data-astro-transition` attribute to the HTML element with values
`"forward"` or `"back"`.

### Lifecycle events (fire in sequence):
1. `astro:before-preparation`
2. `astro:after-preparation`
3. `astro:before-swap`
4. `astro:after-swap`
5. `astro:page-load`

### Known limitation on scripts (exact quote from docs):
> "Bundled module scripts...are only ever executed once. After initial execution they
> will be ignored, even if the script exists on the new page after a transition."

```yaml
- claim: "Astro view transitions use ClientRouter + document.startViewTransition; lifecycle has 5 named events"
  exact_quote: "Bundled module scripts...are only ever executed once. After initial execution they will be ignored, even if the script exists on the new page after a transition."
  category: astro.view_transitions.mechanism
  sources:
    - url: "https://docs.astro.build/en/guides/view-transitions/"
      tier: 1
      accessed_at_utc: "2026-05-26T00:00:00Z"
      archive_url: null
      quote_location: "How transitions work section / Scripts section"
  status: verified
  notes: "ClientRouter replaced the deprecated <ViewTransitions /> component in Astro 4.x"
```

---

## 3. Does `transition:persist` Work on Canvas Elements?

**Claim under investigation:** "transition:persist works on canvas elements"
**Finding:** PARTIALLY CORRECTED — Documented to have a known Safari 18 bug causing
WebGL context loss; a fix was merged but the docs do not confirm reliable canvas support.

### From official docs (accessed 2026-05-26)
The Astro view transitions guide contains this limitation statement:
> "Not all state can be preserved in this way. The restart of CSS animations and the
> reload of iframes cannot be avoided during view transitions even when using
> `transition:persist`."

Canvas elements are **not mentioned in the documented limitations** and **not mentioned
as explicitly supported**. The docs are silent on canvas behavior.

### From GitHub issue tracker
**Issue #15727** (found in https://github.com/withastro/astro/issues?q=canvas+transition+persist,
accessed 2026-05-26):

- Title: "Safari 18: View Transitions break CSS rendering and cause WebGL context loss
  on transition:persist canvas"
- Status: **Closed March 5, 2026**
- Fix: **PR #15728** — "fix(transitions): prevent WebGL context loss on persist canvas
  during Safari swap"

This confirms:
1. A real bug existed where `transition:persist` on a canvas element caused **WebGL context loss** in Safari 18
2. The fix was merged as of early March 2026
3. The fix targets the DOM swap phase to prevent context loss during Safari's view transition handling

```yaml
- claim: "transition:persist on canvas elements caused WebGL context loss in Safari 18; fixed in PR #15728 (merged ~March 2026)"
  exact_quote: "Safari 18: View Transitions break CSS rendering and cause WebGL context loss on transition:persist canvas"
  category: astro.view_transitions.canvas_persist
  sources:
    - url: "https://github.com/withastro/astro/issues?q=canvas+transition+persist"
      tier: 1
      accessed_at_utc: "2026-05-26T00:00:00Z"
      archive_url: null
      quote_location: "Issue #15727 title"
    - url: "https://docs.astro.build/en/guides/view-transitions/"
      tier: 1
      accessed_at_utc: "2026-05-26T00:00:00Z"
      archive_url: null
      quote_location: "Known limitations — transition:persist section"
  status: corrected
  notes: "Bug was present, fix merged March 2026. Current Astro 6.3.8 should include the
          fix. Docs do not explicitly confirm canvas is reliably persistable.
          Recommend testing + archiving the closed issue for evidence chain.
          The claim 'transition:persist works on canvas elements' is overstated without
          qualification — cross-browser reliability is not confirmed in primary docs."
```

---

## 4. Known `transition:persist` Limitations (Documented)

From official docs (exact quote):
> "Not all state can be preserved in this way. The restart of CSS animations and the
> reload of iframes cannot be avoided during view transitions even when using
> `transition:persist`."

| Element type | Behavior with transition:persist |
|---|---|
| CSS animations | Restart (cannot be avoided) |
| iframes | Reload (cannot be avoided) |
| Canvas (WebGL) | Safari 18 bug fixed in PR #15728; behavior on other browsers unconfirmed in docs |
| Canvas (2D) | Not documented |

---

## Overall Assessment

| Claim | Status |
|---|---|
| Current Astro version is 6.3.8 (2026-05-26) | VERIFIED |
| View transitions use ClientRouter + document.startViewTransition | VERIFIED |
| transition:persist works reliably on canvas elements | CORRECTED — Safari 18 bug existed; fix merged March 2026; doc confirmation absent |
| CSS animations restart even with transition:persist | VERIFIED |
| iframes reload even with transition:persist | VERIFIED |

**Sources used:**
- https://docs.astro.build/en/guides/view-transitions/ (tier 1, accessed 2026-05-26)
- https://github.com/withastro/astro/releases (tier 1, accessed 2026-05-26)
- https://github.com/withastro/astro/issues?q=canvas+transition+persist (tier 1, accessed 2026-05-26)
- https://astro.build/blog/astro-5/ (tier 1, accessed 2026-05-26)
