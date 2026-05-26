# three-mesh-bvh — Validation Record
**Validator:** Senior Research Analyst
**Date:** 2026-05-26
**Folder:** 47-validation-round2

---

## Summary

three-mesh-bvh is an active, well-maintained library with 3,400+ stars and a release
as recently as May 13, 2026. The API (computeBoundsTree / disposeBoundsTree) is
confirmed. The "100x speedup" claim stated in the research batch is NOT present in the
README or repository landing page — the library's own performance framing is
"Casting 500 rays against an 80,000 polygon model at 60fps!" No numerical speedup
multiplier is quoted by the library itself.

---

## 1. Repository Vitals

**Source:** https://github.com/gkjohnson/three-mesh-bvh (tier 1, accessed 2026-05-26)

| Metric | Value |
|---|---|
| Stars | 3,400+ (shown as "3.4k") |
| Current version | v0.9.10 |
| Release date of v0.9.10 | May 13, 2026 |
| Total commits (master) | 3,453 |
| Maintenance status | Active |

```yaml
- claim: "three-mesh-bvh is at v0.9.10, released May 13 2026, with 3.4k GitHub stars"
  exact_quote: null
  category: three_mesh_bvh.repository.vitals
  sources:
    - url: "https://github.com/gkjohnson/three-mesh-bvh"
      tier: 1
      accessed_at_utc: "2026-05-26T00:00:00Z"
      archive_url: null
      quote_location: "repository header / releases"
  status: verified
  notes: "Star count is approximate; GitHub truncates to '3.4k'"
```

---

## 2. API Confirmation

**Source:** https://github.com/gkjohnson/three-mesh-bvh/blob/master/README.md
(tier 1, accessed 2026-05-26)

The following methods are confirmed present in the library:

### Core geometry methods (prototype extensions on THREE.BufferGeometry)
- `computeBoundsTree()` — generates the BVH structure for the geometry
- `disposeBoundsTree()` — cleans up BVH resources

### Usage pattern (from README):
```javascript
import { computeBoundsTree, disposeBoundsTree, acceleratedRaycast } from 'three-mesh-bvh';

THREE.BufferGeometry.prototype.computeBoundsTree = computeBoundsTree;
THREE.BufferGeometry.prototype.disposeBoundsTree = disposeBoundsTree;
THREE.Mesh.prototype.raycast = acceleratedRaycast;

// Then on any geometry:
geom.computeBoundsTree();
```

### Additional API (confirmed):
- `computeBatchedBoundsTree()` / `disposeBatchedBoundsTree()` — for `BatchedMesh` objects
- `acceleratedRaycast()` — replacement raycast function attached to `THREE.Mesh.prototype`

```yaml
- claim: "computeBoundsTree() and disposeBoundsTree() are prototype extensions on THREE.BufferGeometry"
  exact_quote: null
  category: three_mesh_bvh.api.core_methods
  sources:
    - url: "https://github.com/gkjohnson/three-mesh-bvh/blob/master/README.md"
      tier: 1
      accessed_at_utc: "2026-05-26T00:00:00Z"
      archive_url: null
      quote_location: "Quick Start / API section"
  status: verified
  notes: "Methods are added via prototype assignment, not built into Three.js core"
```

---

## 3. Raycast Speedup Claim — "100x"

**Claim under investigation:** The research batch states a "100x" raycast speedup.

**Finding:** CORRECTED — The 100x figure does not appear in the README or repository
landing page. The library's own performance claim is:

> "Casting 500 rays against an 80,000 polygon model at 60fps!"

This is a capability statement, not a comparative speedup multiplier. The README
does not present a benchmark table with baseline vs. BVH numbers or a stated
multiplier.

### What the library does claim
- The animated demonstration in the README header shows real-time raycasting against
  complex geometry as the primary value proposition
- The library description frames itself as providing "hierarchical spatial acceleration
  structures" — the speedup is implicit in the use case (complex geometry at interactive
  framerates) but not quantified as "100x" anywhere in primary sources

### Where "100x" may originate
The 100x figure appears in community discussions, blog posts, and tutorial content
(Tier 4–5 sources). It is plausible as a rough ballpark for simple scene geometries
versus naive O(n) raycast — BVH reduces raycast complexity from O(n) to O(log n) —
but the specific multiplier is not sourced from the library authors.

```yaml
- claim: "The '100x raycast speedup' figure is NOT found in the three-mesh-bvh README or official sources"
  exact_quote: "Casting 500 rays against an 80,000 polygon model at 60fps!"
  category: three_mesh_bvh.performance.speedup_claim
  sources:
    - url: "https://github.com/gkjohnson/three-mesh-bvh/blob/master/README.md"
      tier: 1
      accessed_at_utc: "2026-05-26T00:00:00Z"
      archive_url: null
      quote_location: "README header / performance demo description"
    - url: "https://github.com/gkjohnson/three-mesh-bvh"
      tier: 1
      accessed_at_utc: "2026-05-26T00:00:00Z"
      archive_url: null
      quote_location: "Repository landing page animated demo caption"
  status: corrected
  notes: "The '100x' figure is UNVERIFIED at Tier 1. The library's own claim is
          descriptive ('500 rays at 60fps on 80k polygon model'), not a multiplier.
          Research batch should remove or source this figure to Tier 4-5 with
          appropriate UNVERIFIED tag."
```

---

## Overall Assessment

| Claim | Status |
|---|---|
| v0.9.10 released May 13, 2026 | VERIFIED |
| 3.4k GitHub stars | VERIFIED |
| `computeBoundsTree()` on geometry — confirmed API | VERIFIED |
| `disposeBoundsTree()` on geometry — confirmed API | VERIFIED |
| `acceleratedRaycast()` — confirmed API | VERIFIED |
| 100x raycast speedup claim | CORRECTED — not found in primary source; figure is UNVERIFIED |

**Sources used:**
- https://github.com/gkjohnson/three-mesh-bvh (tier 1, accessed 2026-05-26)
- https://github.com/gkjohnson/three-mesh-bvh/blob/master/README.md (tier 1, accessed 2026-05-26)
