# Implementation Brief: ClassCamCore Output — 1920 long-edge sizing + crop-rect math

**Spec:** `dekspec/working-specs/WS-005-render-pipeline-output-sizing.md`
**Intent:** `dekspec/intents/INT-003-correct-crop-export.md`
**Source AEs:** AE-001
**Depends on:** IB-001 (ClassCamCore package + `AspectRatio`)
**Production gate:** none (pure Swift; unit-tested with zero simulator)
**Status:** ACCEPTED

## Precedence

Reviewed for conflicts before writing. Resolve residual ambiguity by: (1) Constraints & Decisions (this IB), (2) Domain Constraints (this IB), (3) Quality Checklists. Do not implement from Spec Context. Stop and ask on any unresolved conflict.

## Goal

The `ClassCamCore` `Output` slice exists and computes the output-image sizing and crop rectangle as **pure functions** — the long-edge scale factor to exactly 1920, and the per-aspect output size (1920×1080 for 16:9, 1920×1440 for 4:3) and centered crop rect — with a unit-test suite that runs green with **zero simulator**. `ImagingService.render` (IB-008) calls these functions rather than reimplementing the math on the GPU path.

## Out of Scope

- The `CIImage` graph, `CIPerspectiveCorrection`, `CILanczosScaleTransform`, and the `CIContext` render (IB-008) — this IB ships only the pure sizing/crop math those consume.
- Any UIKit/Core Image import — `Output` is pure `ClassCamCore` (CoreGraphics value types only).
- Export, clipboard, share, aspect persistence (IB-009).

## Escalation Protocol

Stop and ask when a decision needs information not in this IB, when a file outside Files to Modify must change, or when a Done When criterion cannot be met without touching out-of-scope work. Do not guess.

## Spec Context

**Do not implement from this section — traceability only.** WS-005 §Governing Formulas (long-edge scale, per-aspect output, crop rect), §Business Rules BR3–BR5, §Domain Constraints (long-edge size, crop, colour space contract). ADR-008 §Decision (the pure sizing/crop math lives in `ClassCamCore`).

## Files to Modify

*Xcode-side paths (built on the Mac). Structure per council plan §4.*

| File | Change |
|------|--------|
| `ClassCamCore/Sources/ClassCamCore/Output/OutputSizing.swift` | Pure `longEdgeScale(for:)` → scale factor s.t. the long edge becomes exactly 1920; `outputSize(for aspect:)` → `(1920,1080)` / `(1920,1440)`. |
| `ClassCamCore/Sources/ClassCamCore/Output/CropRect.swift` | Pure `cropRect(for aspect:in scaledExtent:)` → centered `CGRect` of the aspect's dimensions within the scaled extent (bottom-left origin, Core-Image convention). |
| `ClassCamCore/Tests/ClassCamCoreTests/OutputSizingTests.swift` | Unit tests: long-edge scale for landscape/portrait inputs; `outputSize` yields exactly 1920×1080 and 1920×1440 (WS-005 BR3–BR4). |
| `ClassCamCore/Tests/ClassCamCoreTests/CropRectTests.swift` | Unit tests: centered crop rect for both aspects; rect stays within the scaled extent (WS-005 BR5). |

## Reuse Inventory

| Capability | Location | Use instead of reimplementing |
|------------|----------|-------------------------------|
| `CGSize` / `CGRect` / `CGFloat` | CoreGraphics (value types only) | Don't hand-roll size/rect types; CoreGraphics values are `Sendable` and UI-free. |
| `AspectRatio` | `ClassCamCore` (IB-001) | Switch over the existing closed enum; don't redefine the aspect set. |
| Swift Testing (`@Test`/`#expect`) | Swift toolchain | Use for the unit suite; don't build a bespoke assertion harness. |

## Domain Constraints

| Constraint | Value |
|------------|-------|
| Long-edge size | Exactly 1920 on the long edge |
| Output size | 1920×1080 (16:9), 1920×1440 (4:3) |
| Package imports | No UIKit / AVFoundation / SwiftUI / Core Image |
| Coordinate convention | Crop rect in bottom-left pixel origin (Core-Image convention) |
| Compute device / dtype | n/a — pure Swift values |

## Environment Prerequisites

| Prerequisite | Probe command | Required |
|--------------|---------------|----------|
| n/a — no live services | — | no |

## Do Not Touch

| Function/File | Reason |
|---------------|--------|
| `ImagingService` / any Core Image code | Owned by IB-008; this IB is package-only pure math. |
| Existing `ClassCamCore` model types | Owned by IB-001; add the `Output` slice without restructuring them. |

## Governing ADRs

| ADR | Title |
|-----|-------|
| ADR-008 | One lazy CIImage graph, render once through the single CIContext, sRGB at 1920 long edge |
| ADR-006 | ClassCamCore UI-free pure package |

## Constraints & Decisions

- **Pure math only:** `Output` imports no UIKit/Core Image; it computes sizes and rects from `CGSize`/`AspectRatio` inputs. The GPU render (IB-008) consumes these values.
- **Exact output sizes:** `outputSize(.sixteenNine) == (1920, 1080)` and `outputSize(.fourThree) == (1920, 1440)` — exact integers, no rounding drift.
- **Long-edge scale:** `longEdgeScale(for: size)` returns `1920 / max(size.width, size.height)` so the long edge lands on exactly 1920 (landscape and portrait both handled).
- **Centered crop:** `cropRect` is the aspect's `(w,h)` centered within the scaled extent, in bottom-left pixel origin (Core-Image convention) so IB-008 can feed it straight to a crop.
- **Green with zero simulator:** the test target runs on the host toolchain; no test imports a platform-visual framework.

## Interface Contracts

- `dekspec/interface-contracts/IC-001-captured-image.md` — traceability only (IB-008 consumes the source still; this IB is downstream sizing math).

## Quality Checklists

- Swift API design + strict-concurrency cleanliness (no `@unchecked Sendable`); exact-integer output-size assertions.

## Test Promotion Criteria

Promotion refs: WS-005 BR3 (16:9 = 1920×1080), BR4 (4:3 = 1920×1440), BR5 (centered crop rect); Governing Formulas (long-edge scale).

## Test Layout

- `ClassCamCore/Tests/ClassCamCoreTests/OutputSizingTests.swift`
- `ClassCamCore/Tests/ClassCamCoreTests/CropRectTests.swift`

## Done When

- [ ] `outputSize(.sixteenNine)` is exactly `1920×1080` and `outputSize(.fourThree)` is exactly `1920×1440` — verified by unit test (WS-005 BR3–BR4).
- [ ] `longEdgeScale(for:)` puts the long edge on exactly 1920 for both a landscape and a portrait input — verified by unit test.
- [ ] `cropRect(for:in:)` returns the aspect's dimensions centered within the scaled extent, fully inside it — verified by unit test (WS-005 BR5).
- [ ] The `Output` slice imports no UIKit/AVFoundation/SwiftUI/Core Image — verified by build + a manifest/import grep.
- [ ] The test suite runs green with **no simulator** — verified by running the package tests on the host toolchain.
- [ ] outcome test landed first (red), implementation made it green, no other test files modified to make it pass — strong-TDD per ADR-029; verified by git-blame.
- [ ] All new tests pass; no pre-existing tests break — verified by test run.

**Golden State Transitions**

| Input | Expected Output | Verified by |
|-------|----------------|-------------|
| `outputSize(.sixteenNine)` | `(1920, 1080)` | unit test |
| `outputSize(.fourThree)` | `(1920, 1440)` | unit test |
| `longEdgeScale(for: CGSize(4000, 2000))` | `0.48` (long edge → 1920) | unit test |
| `longEdgeScale(for: CGSize(2000, 4000))` | `0.48` (long edge → 1920) | unit test |
| `cropRect(.sixteenNine, in: 1920×1200 extent)` | centered `1920×1080` rect | unit test |

## Open Issues

- *None.*

## Amendment Log

| Date | Type | Change | Author |
|------|------|--------|--------|
| 2026-08-10 | Substantive | Initial authoring — IB-007 ClassCamCore Output sizing/crop math (INT-003 IU1, from WS-005). | noreply@anthropic.com |
| 2026-08-10 | Substantive | Accepted as INT-003 IB | noreply@anthropic.com |
