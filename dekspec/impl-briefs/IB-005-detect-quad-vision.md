# Implementation Brief: ImagingService.detectQuad — VNDetectRectangles tuning + default-inset-box fallback

**Spec:** `dekspec/working-specs/WS-004-detect-corners-and-adjust.md`
**Intent:** `dekspec/intents/INT-002-detect-and-adjust-corners.md`
**Source AEs:** AE-001
**Depends on:** IB-004 (Geometry conversions), IB-001 (`Quad`/`CapturedImage`)
**Production gate:** on-device manual attestation (INT-002 Verification — detection on a real glowing TV/whiteboard viewed off-axis)
**Status:** ACCEPTED

## Precedence

Reviewed for conflicts before writing. Resolve residual ambiguity by: (1) Constraints & Decisions, (2) Domain Constraints, (3) Quality Checklists. Do not implement from Spec Context. Stop and ask on any unresolved conflict.

## Goal

The stub `ImagingService.detectQuad(in:)` is replaced by a real `VNDetectRectanglesRequest` wrapper — tuned for a **glowing angled screen** — that picks the best observation by confidence, converts its corners to a normalized, ordered, validated `Quad` through the IB-004 `Geometry` conversions, and returns it, or returns `nil` when nothing clears the (deliberately permissive) bar. On `nil`, the processing screen has a **mandatory default inset box (~10%)** fallback the user can immediately position.

## Out of Scope

- The draggable-handle gesture + seed/source-of-truth wiring — IB-006 (this IB provides the `Quad?` the overlay seeds from, and the fallback box `Quad`).
- Coordinate-space math — IB-004 (consumed, never re-derived here).
- `Quad` ordering/validation — IB-001 (consumed via the failable init).
- Perspective warp / white balance / export — INT-003/004.

## Escalation Protocol

Stop and ask when a decision needs information not in this IB, when a file outside Files to Modify must change, or when a Done When criterion cannot be met without out-of-scope work. Do not guess. In particular: do **not** inline a coordinate flip here (route through IB-004), and do **not** re-run detection over the user's edits (ADR-007 — that belongs to nobody; the dragged quad is authoritative).

## Spec Context

**Traceability only.** WS-004 §What This Does (Detection + Fallback), §Business Rules BR1–BR6, §Domain Constraints (the request-parameter tuning), §Governing Formulas (best-observation, default inset box), §Failure Behavior. WS-003 (the conversions consumed). IC-001 (the `CapturedImage` detection runs on).

## Files to Modify

*Xcode-side paths (built on the Mac). Structure per council plan §4.*

| File | Change |
|------|--------|
| `ClassCam/Imaging/ImagingService.swift` | Replace the stub `detectQuad(in:)` with a real `VNDetectRectanglesRequest` run on the actor (off main): configure the tuned parameters, run via `VNImageRequestHandler`, select the highest-confidence observation, convert its corners through `ClassCamCore/Geometry` (IB-004), construct a `Quad` via the failable init, return `Quad?`. Honour `Task.isCancelled`. |
| `ClassCam/Imaging/RectangleDetection.swift` | New: the request configuration (`quadratureTolerance`, `minimumConfidence`, `minimumAspectRatio`, `minimumSize`, `maximumObservations`) + the best-by-confidence selection, factored out for readability/tuning. |
| `ClassCamCore/Sources/ClassCamCore/Geometry/DefaultQuad.swift` | New pure helper: `defaultInsetQuad(inset:) -> Quad` producing the ~10% inset box as a valid ordered `Quad` (used by the fallback; pure so it unit-tests with zero simulator). |
| `ClassCamCore/Tests/ClassCamCoreTests/DefaultQuadTests.swift` | New: unit tests that the default inset box is a valid, ordered `Quad` at the expected inset. |

## Reuse Inventory

| Capability | Location | Use instead of reimplementing |
|------------|----------|-------------------------------|
| `VNDetectRectanglesRequest`, `VNImageRequestHandler`, `VNRectangleObservation` | Vision | Use the framework rectangle detector; do not hand-roll edge/corner detection. |
| `ClassCamCore/Geometry` conversions | `ClassCamCore` (IB-004) | Convert Vision's normalized-BL corners to the `Quad`'s space through the tested functions; never inline a flip. |
| `Quad` failable init | `ClassCamCore` (IB-001) | Construct the quad through the existing ordering+validating init; a degenerate observation fails construction → treat as no detection. |
| `CapturedImage` (pixels + orientation) | `ClassCamCore` (IB-001) | Feed the still + its sealed orientation to `VNImageRequestHandler`; don't re-decode the image. |
| `Task.isCancelled` / structured concurrency | Swift concurrency | Cancel an in-flight detection when the view leaves; don't spin a detached `Task`. |
| Swift Testing (`@Test`/`#expect`) | Swift toolchain | Use for the `defaultInsetQuad` unit tests. |

## Domain Constraints

| Constraint | Value |
|------------|-------|
| Detection actor | `detectQuad` runs on `ImagingService` (off main); returns `Sendable` `Quad?` |
| `quadratureTolerance` | generous, ~30–45° |
| `minimumConfidence` | low, ~0.3 |
| `minimumAspectRatio` | permissive, ~0.3 |
| `maximumObservations` | a small handful; pick best by confidence |
| Fallback inset | default box ~10% inset (mandatory) |
| Conversions | route through `ClassCamCore/Geometry` (IB-004) — no inline flips |
| Cancellation | honour `Task.isCancelled`; called from the view's `.task {}` |
| Compute device / dtype | n/a — camera stills + Vision observations, not tensors |

## Environment Prerequisites

| Prerequisite | Probe command | Required |
|--------------|---------------|----------|
| A real iPad + a real glowing TV/whiteboard viewed off-axis (the simulator proves nothing for detection quality — council R5/§7) | manual on-device run | yes |

## Do Not Touch

| Function/File | Reason |
|---------------|--------|
| `ClassCamCore/Geometry` conversion functions | Owned by IB-004; consume, don't modify. |
| `Quad` ordering/validation | Owned by IB-001. |
| The draggable overlay / `ProcessingScreen` drag wiring | Owned by IB-006. |
| Imaging render / white-balance path | Later Intents (INT-003/004). |

## Governing ADRs

| ADR | Title |
|-----|-------|
| ADR-007 | Coordinate-space conversion discipline (detection seeds; conversions via Geometry; never re-run over edits) |
| ADR-002 | Apple-native Vision + Core Image |
| ADR-005 | Swift 6 strict-concurrency actor-isolated services |
| ADR-004 | Build the UI in SwiftUI |

## Constraints & Decisions

- **Tuned for a glowing angled screen:** generous `quadratureTolerance` (~30–45°), low `minimumConfidence` (~0.3), permissive `minimumAspectRatio` (~0.3) + `minimumSize`, a small `maximumObservations` — a bright TV shot off-axis is not a crisp square document; tightening any of these means real presentations never detect (WS-004 BR1).
- **Best by confidence:** when several observations return, select the single highest-confidence one to seed the handles (WS-004 BR2).
- **Convert through Geometry:** the observation's normalized-BL corners become the `Quad`'s corners via IB-004's conversions + the WS-001 failable init; no inline flip (WS-004 BR3, ADR-007).
- **Degenerate → no detection:** if `Quad` construction fails on the observation's corners, return `nil` (fall back) rather than a bad quad (WS-004 BR4).
- **`nil` is normal:** no observation clearing the bar returns `nil` — not a throw, not an error; the screen shows the fallback box (WS-004 BR5).
- **Mandatory fallback:** `defaultInsetQuad(inset: ~0.10)` yields a valid ordered `Quad` for the fallback; it is pure `ClassCamCore` math, unit-tested with zero simulator (WS-004 BR6).
- **Off main + cancellable:** detection runs on the actor; `Task.isCancelled` is checked around the request so leaving processing cancels it (ADR-005; WS-004 BR9-related).

## Interface Contracts

- `dekspec/interface-contracts/IC-001-captured-image.md` — the `CapturedImage` detection consumes (traceability only).

## Quality Checklists

- Vision request-configuration correctness; strict-concurrency cleanliness (detection off main, `Sendable` `Quad?` out); on-device attestation checklist for detection quality; pure-function purity for `defaultInsetQuad`.

## Test Promotion Criteria

Promotion refs: WS-004 BR6 (`defaultInsetQuad` validity/inset — unit). BR1–BR5 (detection tuning, selection, conversion, degenerate→nil, nil-is-normal) are on-device attestation (Vision detection has no automatable surface in this repo). BR4 leans on the IB-001 `Quad` init contract.

## Test Layout

- `ClassCamCore/Tests/ClassCamCoreTests/DefaultQuadTests.swift` (pure default-inset-box unit tests)
- On-device attestation checklist for detection quality/tuning (no automatable surface for the Vision behaviors — INT-002 Verification).

## Done When

- [ ] `defaultInsetQuad(inset: 0.10)` produces a valid, ordered `Quad` with corners at ~10% inset of the frame — verified by unit test (WS-004 BR6).
- [ ] On device: `detectQuad` finds the presentation on a real glowing TV/whiteboard viewed off-axis with the tuned parameters — verified by on-device attestation (WS-004 BR1).
- [ ] On device: when several rectangles are present, the highest-confidence one seeds the handles — verified by on-device attestation (WS-004 BR2).
- [ ] On device: an unclear/degenerate frame returns `nil` and the screen shows the default inset box (no crash, no hang) — verified by on-device attestation (WS-004 BR4–BR6).
- [ ] Detection runs off the main thread and honours cancellation on leaving processing — verified by code review + on-device attestation (WS-004 BR9-related; ADR-005).
- [ ] The observation's corners route through `ClassCamCore/Geometry` (no inline flip) — verified by code review (ADR-007).
- [ ] outcome test landed first (red), implementation made it green, no other test files modified to make it pass — strong-TDD per ADR-029 (applies to the `defaultInsetQuad` unit test); verified by git-blame.
- [ ] All new unit tests pass; no pre-existing tests break — verified by test run.

**Golden State Transitions**

| Input | Expected Output | Verified by |
|-------|----------------|-------------|
| `defaultInsetQuad(inset: 0.10)` | ordered `Quad` with corners `(0.1,0.1),(0.9,0.1),(0.9,0.9),(0.1,0.9)` (TL→TR→BR→BL after ordering) | unit test |
| several observations returned | the highest-confidence quad seeds the handles | on-device attestation |
| no observation clears the bar | `detectQuad` returns `nil` → default inset box shown | on-device attestation |
| observation with degenerate corners | `Quad` init fails → `nil` → fallback box | on-device attestation |

## Open Issues

- [ ] The exact request parameter values are finalized during on-device tuning against real glowing-screen shots (WS-004 Open Issue). — **Source:** initial draft — **Severity:** `P3`
- [ ] The default fallback inset fraction (~10%) is confirmed during on-device tuning (WS-004 Open Issue). — **Source:** initial draft — **Severity:** `P3`

## Amendment Log

| Date | Type | Change | Author |
|------|------|--------|--------|
| 2026-08-10 | Substantive | Initial authoring — IB-005 `ImagingService.detectQuad` `VNDetectRectangles` tuning + `defaultInsetQuad` fallback (INT-002 IU2, from WS-004; council §3.2). | noreply@anthropic.com |
| 2026-08-10 | Substantive | Accepted as INT-002 IB. | noreply@anthropic.com |
