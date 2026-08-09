# Implementation Brief: ClassCamCore Geometry — coordinate-space conversions + round-trip tests

**Spec:** `dekspec/working-specs/WS-003-coordinate-space-conversions.md`
**Intent:** `dekspec/intents/INT-002-detect-and-adjust-corners.md`
**Source AEs:** AE-001
**Depends on:** IB-001 (ClassCamCore package + `Quad`/`CapturedImage` types)
**Production gate:** none (spec-only repo; built in Xcode on a Mac) — but the round-trip unit tests are the hard gate before any device use (ADR-007)
**Status:** ACCEPTED

## Precedence

Reviewed for conflicts before writing. Resolve residual ambiguity by: (1) Constraints & Decisions (this IB), (2) Domain Constraints (this IB), (3) Quality Checklists. Do not implement from Spec Context. Stop and ask on any unresolved conflict.

## Goal

The `ClassCamCore` `Geometry` slice gains the pure functions that convert the presentation's four corner points between the three coordinate spaces — Vision normalized-bottom-left ↔ Core Image pixel-bottom-left ↔ SwiftUI/UIKit view-points-top-left — written **once**, with a unit-test suite proving the round-trips lossless corner-for-corner on a known rectangle, running green with **zero simulator**. This is the "written once, unit-tested before device use" surface ADR-007 mandates.

## Out of Scope

- Corner **detection** (`VNDetectRectanglesRequest`) — IB-005; this IB is the pure math detection will consume.
- The draggable-handle overlay and the fallback box — IB-006.
- The perspective **warp** that consumes the pixel-space corners — INT-003.
- `Quad` ordering/validation — already shipped by IB-001 (WS-001 BR6–BR7); this IB converts already-ordered points between spaces, it does not re-order.

## Escalation Protocol

Stop and ask when a decision needs information not in this IB, when a file outside Files to Modify must change, or when a Done When criterion cannot be met without touching out-of-scope work. Do not guess. In particular: do **not** add an inline coordinate flip at a call site outside `Geometry` to "make it work" (ADR-007 — one audited home only).

## Spec Context

**Do not implement from this section — traceability only.** WS-003 §Interfaces (the conversion table), §Business Rules BR1–BR8, §Governing Formulas, §Failure Behavior. IC-001 §Interface Definition (the `CapturedImage` pixel dimensions + sealed orientation the conversions scale/flip against).

## Files to Modify

*Xcode-side paths (built on the Mac). Structure per council plan §4.*

| File | Change |
|------|--------|
| `ClassCamCore/Sources/ClassCamCore/Geometry/CoordinateSpaces.swift` | New: the pure conversion functions — `normalizedBLToPixelBL`, `pixelBLToNormalizedBL`, `pixelBLToViewPointsTL`, `viewPointsTLToPixelBL`, and the composed `normalizedBLToViewPointsTL` / inverse. Each takes the point + image `CGSize` (px) (+ view `CGSize` for the point conversions) and the sealed orientation; no stored state. |
| `ClassCamCore/Tests/ClassCamCoreTests/CoordinateSpaceTests.swift` | New: round-trip losslessness tests on a known rectangle (BR1–BR8), including the single-Y-flip assertion and an orientation-parameterized case. |

## Reuse Inventory

| Capability | Location | Use instead of reimplementing |
|------------|----------|-------------------------------|
| `CGPoint` / `CGSize` / `CGRect` / `CGAffineTransform` | CoreGraphics (value types only) | Don't hand-roll a 2D point/scale type; CoreGraphics values are `Sendable` and UI-free. |
| `VNImageRectForNormalizedRect` / `VNImagePointForNormalizedPoint` | Vision (pure geometry helpers — no image object) | May be used for the normalized→pixel step; if imported, it must stay a pure helper so the package still tests with zero simulator. If it drags a UI/framework import, do the explicit multiply instead. |
| `CapturedImage` (pixel dims + orientation) | `ClassCamCore` (IB-001) | Read the pixel dimensions + sealed orientation from the existing envelope; don't re-derive image size. |
| `Quad` | `ClassCamCore` (IB-001) | The corners being converted come from / feed a `Quad`; don't define a new corner type. |
| Swift Testing (`@Test`/`#expect`) | Swift toolchain | Use for the round-trip suite; don't build a bespoke assertion harness. |

## Domain Constraints

| Constraint | Value |
|------------|-------|
| Package purity | Lives in `ClassCamCore/Geometry`; no UIKit/AVFoundation/SwiftUI import (Vision only if it stays a pure helper) |
| Single home | Every origin flip / unit scale between the three spaces exists only here (ADR-007) |
| Purity | Functions are pure; all dimensions passed as parameters; no shared mutable state |
| Coordinate units | The space (normalized-BL / pixel-BL / points-TL) is explicit in each function name |
| Orientation | Y-flip uses the oriented height from the sealed `CGImagePropertyOrientation` (IC-001) |
| Compute device / dtype | n/a — pure `CGFloat` geometry |

## Environment Prerequisites

| Prerequisite | Probe command | Required |
|--------------|---------------|----------|
| n/a — no live services; host toolchain only | — | no |

## Do Not Touch

| Function/File | Reason |
|---------------|--------|
| `Quad` ordering/validation (`QuadOrdering.swift`) | Owned by IB-001; this IB converts already-ordered points, it does not re-order. |
| App target / SwiftUI views / imaging actor | IB-005/IB-006 and later Intents. |

## Governing ADRs

| ADR | Title |
|-----|-------|
| ADR-007 | Coordinate-space conversion discipline (conversions once, unit-tested; detection seeds, dragged `Quad` authoritative) |
| ADR-006 | ClassCamCore UI-free pure package |
| ADR-002 | Apple-native Vision + Core Image |

## Constraints & Decisions

- **Written once:** every conversion between the three spaces lives in `CoordinateSpaces.swift`; no call site (detection, overlay, warp) re-derives a flip or a scale (ADR-007).
- **Pure functions:** output depends only on the inputs (point + image dims + view size + orientation); no captured state, so the same function is safe on `@MainActor` (overlay) and on the imaging actor (warp).
- **Single Y-flip:** exactly one Y-flip separates a bottom-left space (Vision/Core Image) from the top-left space (SwiftUI/UIKit); normalized-BL→pixel-BL carries **no** flip (both bottom-left), only a scale by pixel dims (WS-003 BR4–BR5).
- **Orientation-honest:** the dimension used as "height" for the flip is the oriented height from the sealed `CGImagePropertyOrientation` (IC-001), so a portrait-vs-landscape capture flips against the correct axis (WS-003 BR6).
- **Guard degenerate scale:** a zero image dimension must not silently divide-by / produce a NaN corner; precondition-guard or require the caller to supply valid `CapturedImage` dims (WS-003 BR8).
- **Green with zero simulator:** the test target runs on the host toolchain; no test imports a platform-visual framework.

## Interface Contracts

- `dekspec/interface-contracts/IC-001-captured-image.md` — supplies the pixel dimensions + sealed orientation the conversions use (traceability only).

## Quality Checklists

- Swift API design + strict-concurrency cleanliness (no `@unchecked Sendable`); pure-function discipline; float-tolerance choice in the round-trip assertions is explicit and justified.

## Test Promotion Criteria

Promotion refs: WS-003 BR1 (normalized↔pixel round-trip), BR2 (pixel↔points round-trip incl. Y-flip+scale), BR3 (composed corner-for-corner), BR4 (single Y-flip), BR5 (no flip normalized→pixel), BR6 (orientation), BR8 (zero-dimension guard).

## Test Layout

- `ClassCamCore/Tests/ClassCamCoreTests/CoordinateSpaceTests.swift` (pure round-trip unit tests — the ADR-007 hard gate)

## Done When

- [ ] normalized-BL ↔ pixel-BL round-trips the four corners of a known rectangle lossless corner-for-corner (within an explicit float tolerance) — verified by unit test (WS-003 BR1).
- [ ] pixel-BL ↔ view-points-TL round-trips lossless including the Y-flip and view/image scale — verified by unit test (WS-003 BR2).
- [ ] The composed normalized-BL ↔ view-points-TL round-trip is lossless corner-for-corner on a known rectangle — verified by unit test (WS-003 BR3, the canonical council-§7 test).
- [ ] Exactly one Y-flip separates bottom-left from top-left: a top normalized-BL corner maps to the top-left row in points-TL, and normalized→pixel carries no flip — verified by unit test (WS-003 BR4–BR5).
- [ ] The conversion honours the sealed orientation: an orientation-parameterized test maps corners correctly for a portrait vs landscape capture — verified by unit test (WS-003 BR6).
- [ ] A zero image dimension is guarded (no NaN corner leaks) — verified by unit test (WS-003 BR8).
- [ ] The suite runs green with **no simulator** — verified by running the package tests on the host toolchain.
- [ ] outcome test landed first (red), implementation made it green, no other test files modified to make it pass — strong-TDD per ADR-029; verified by git-blame.
- [ ] All new tests pass; no pre-existing tests break — verified by test run.

**Golden State Transitions**

| Input | Expected Output | Verified by |
|-------|----------------|-------------|
| corner `(0,0)` normalized-BL, image `4000×3000`, via normalized→pixel→normalized | `(0,0)` within tolerance | unit test |
| corner `(0,1)` normalized-BL (top-left in BL terms), image `4000×3000`, view `400×300`, via normalized→points-TL | a **top** (small-Y) point in TL space | unit test |
| corner `(0,0)` normalized-BL → pixel-BL | `(0,0)` pixel (no flip; both bottom-left) | unit test |
| four corners of the unit square, composed round-trip normalized↔points | identical four corners within tolerance | unit test |

## Open Issues

- *None.*

## Amendment Log

| Date | Type | Change | Author |
|------|------|--------|--------|
| 2026-08-10 | Substantive | Initial authoring — IB-004 `ClassCamCore` `Geometry` coordinate-space conversions + round-trip tests (INT-002 IU1, from WS-003; ADR-007 hard gate). | noreply@anthropic.com |
| 2026-08-10 | Substantive | Accepted as INT-002 IB. | noreply@anthropic.com |
