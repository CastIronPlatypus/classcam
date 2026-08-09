# Implementation Brief: ClassCamCore foundation (model + Stage + quad math)

**Spec:** `dekspec/working-specs/WS-001-classcamcore-model-state-machine.md`
**Intent:** `dekspec/intents/INT-001-capture-presentation-still.md`
**Source AEs:** AE-001
**Depends on:** none
**Production gate:** none (spec-only repo; built in Xcode on a Mac)
**Status:** ACCEPTED

## Precedence

Reviewed for conflicts before writing. Resolve residual ambiguity by: (1) Constraints & Decisions (this IB), (2) Domain Constraints (this IB), (3) Quality Checklists. Do not implement from Spec Context. Stop and ask on any unresolved conflict.

## Goal

The `ClassCamCore` Swift package exists and compiles with no UI/capture-framework import, exporting the `Stage` state machine, all `Sendable` model value types, and the pure quad ordering + validation logic — with a unit-test suite that runs green with **zero simulator**.

## Out of Scope

- The `@Observable AppModel` and any SwiftUI view (IB-002) — this IB ships the *types* the shell consumes, not the shell.
- Real capture (IB-003).
- Coordinate-space conversions (Vision↔UIKit↔Core Image), white-balance gain math, and 1920-long-edge output sizing — those are later Intents' ClassCamCore additions; this IB owns only quad **ordering** and **validation** plus the type declarations.

## Escalation Protocol

Stop and ask when a decision needs information not in this IB, when a file outside Files to Modify must change, or when a Done When criterion cannot be met without touching out-of-scope work. Do not guess.

## Spec Context

**Do not implement from this section — traceability only.** WS-001 §Interfaces (the value-type table), §Business Rules BR1–BR9, §Failure Behavior. IC-001 §Interface Definition (the `CapturedImage` field set).

## Files to Modify

*Xcode-side paths (built on the Mac). Structure per council plan §4.*

| File | Change |
|------|--------|
| `ClassCamCore/Package.swift` | New Swift package; no UIKit/AVFoundation/SwiftUI dependency; Swift 6 tools + strict concurrency. |
| `ClassCamCore/Sources/ClassCamCore/Model/Stage.swift` | `enum Stage { case capture; case processing(ProcessingState); case result(Slide) }`. |
| `ClassCamCore/Sources/ClassCamCore/Model/ProcessingState.swift` | `struct ProcessingState { let image: CapturedImage; var quad: Quad; var whitePoint: WhitePoint?; var aspect: AspectRatio }`. |
| `ClassCamCore/Sources/ClassCamCore/Model/CapturedImage.swift` | Thin-envelope value per IC-001 (deep-copied pixels handle, orientation tag, pixel dims, colour-space tag); `Sendable`. |
| `ClassCamCore/Sources/ClassCamCore/Model/Quad.swift` | Four normalized `CGPoint`; failable init that **orders** (TL→TR→BR→BL) and **validates** (BR6–BR7). |
| `ClassCamCore/Sources/ClassCamCore/Model/AspectRatio.swift` | `enum AspectRatio { case sixteenNine, fourThree }`. |
| `ClassCamCore/Sources/ClassCamCore/Model/WhiteBalanceGains.swift` | `(r,g,b)` multipliers; `g == 1` convention. |
| `ClassCamCore/Sources/ClassCamCore/Model/WhitePoint.swift` | `struct WhitePoint { tappedPoint: CGPoint; gains: WhiteBalanceGains }`. |
| `ClassCamCore/Sources/ClassCamCore/Model/Slide.swift` | `struct Slide { image; aspect: AspectRatio }` (image payload finalized in INT-003). |
| `ClassCamCore/Sources/ClassCamCore/Geometry/QuadOrdering.swift` | Pure ordering + degenerate/self-intersection/range validation used by `Quad.init`. |
| `ClassCamCore/Tests/ClassCamCoreTests/QuadTests.swift` | Ordering + validation unit tests (BR6–BR7). |
| `ClassCamCore/Tests/ClassCamCoreTests/StageTransitionTests.swift` | State-shape tests (BR1–BR5, BR8–BR9) exercisable without the app. |

## Reuse Inventory

| Capability | Location | Use instead of reimplementing |
|------------|----------|-------------------------------|
| `CGPoint` / `CGSize` / `CGRect` | CoreGraphics (value types only) | Don't hand-roll a 2D point type; CoreGraphics values are `Sendable` and UI-free. |
| Swift Testing (`@Test`/`#expect`) | Swift toolchain | Use for the unit suite; don't build a bespoke assertion harness. |

## Domain Constraints

| Constraint | Value |
|------------|-------|
| Sendability | Every exported type is a `Sendable` value type |
| Package imports | No UIKit / AVFoundation / SwiftUI |
| Coordinate units | Normalized 0–1 for `Quad`/`WhitePoint` |
| Compute device / dtype | n/a — pure Swift values |

## Environment Prerequisites

| Prerequisite | Probe command | Required |
|--------------|---------------|----------|
| n/a — no live services | — | no |

## Do Not Touch

| Function/File | Reason |
|---------------|--------|
| App target / SwiftUI views | Belong to IB-002; this IB is package-only. |

## Governing ADRs

| ADR | Title |
|-----|-------|
| ADR-005 | Swift 6 strict-concurrency actor-isolated services |
| ADR-006 | ClassCamCore UI-free pure package |

## Constraints & Decisions

- **Package purity:** `ClassCamCore` links no UIKit/AVFoundation/SwiftUI. If a type seems to need one, it belongs in the app target (IB-002), not here.
- **Sendable value types:** every model type is a `struct`/`enum` conforming to `Sendable`; no classes, no shared mutable state.
- **Quad is validated-by-construction:** `Quad.init` is failable — it orders arbitrary corners to TL→TR→BR→BL and returns `nil` for degenerate (zero-area, coincident, self-intersecting) or out-of-range (outside 0–1) input. There is no way to hold an invalid `Quad`.
- **CapturedImage is a thin envelope:** it carries a deep-copied pixel handle + orientation tag + pixel dims + colour-space tag (IC-001). It does **not** eagerly decode a full-resolution `CGImage`. The actual deep-copy happens in IB-003 at the shutter; this IB defines the type and its `Sendable` conformance.
- **Green with zero simulator:** the test target runs on the host toolchain; no test imports a platform-visual framework.

## Interface Contracts

- `dekspec/interface-contracts/IC-001-captured-image.md` — defines `CapturedImage`'s field set (traceability only).

## Quality Checklists

- Swift API design + strict-concurrency cleanliness (no `@unchecked Sendable`).

## Test Promotion Criteria

Promotion refs: WS-001 BR1, BR4, BR6, BR7, BR8, BR9; IC-001 field set.

## Test Layout

- `ClassCamCore/Tests/ClassCamCoreTests/QuadTests.swift`
- `ClassCamCore/Tests/ClassCamCoreTests/StageTransitionTests.swift`

## Done When

- [ ] `ClassCamCore` compiles under Swift 6 strict concurrency with no UIKit/AVFoundation/SwiftUI import — verified by build + a grep/lint that the package manifest declares no such dependency.
- [ ] All eight model types exist and conform to `Sendable` — verified by unit test (compile-time conformance check).
- [ ] `Quad.init` orders permuted corners of a known rectangle to identical TL→TR→BR→BL output — verified by unit test (WS-001 BR6).
- [ ] `Quad.init` returns `nil` for zero-area, coincident-corner, self-intersecting, and out-of-range inputs — verified by unit test (WS-001 BR7).
- [ ] `ProcessingState.whitePoint` defaults to `nil`; `AspectRatio` has exactly two cases — verified by unit test (WS-001 BR5, BR8).
- [ ] The test suite runs green with **no simulator** — verified by running the package tests on the host toolchain.
- [ ] All new tests pass; no pre-existing tests break — verified by test run.

**Golden State Transitions**

| Input | Expected Output | Verified by |
|-------|----------------|-------------|
| `Quad` from corners `[BR, BL, TL, TR]` of the unit square | ordered `[TL, TR, BR, BL]` | unit test |
| `Quad` from `[(0,0),(0,0),(1,1),(1,0)]` (coincident) | `nil` | unit test |
| `Quad` from `[(0,0),(1,1),(1,0),(0,1)]` (crossed) | `nil` | unit test |
| `Quad` from `[(-0.1,0),(1,0),(1,1),(0,1)]` (out of range) | `nil` | unit test |

## Open Issues

- *None.*

## Amendment Log

| Date | Type | Change | Author |
|------|------|--------|--------|
| 2026-08-10 | Substantive | Initial authoring — IB-001 ClassCamCore foundation (INT-001 IU1, from WS-001). | noreply@anthropic.com |
| 2026-08-10 | Substantive | Accepted as INT-001 IB | noreply@anthropic.com |
