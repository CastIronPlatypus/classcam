# Implementation Brief: Draggable corner-handle overlay + seed-from-detection / drag-is-source-of-truth wiring

**Spec:** `dekspec/working-specs/WS-004-detect-corners-and-adjust.md`
**Intent:** `dekspec/intents/INT-002-detect-and-adjust-corners.md`
**Source AEs:** AE-001
**Depends on:** IB-005 (`detectQuad` + fallback box), IB-004 (Geometry conversions), IB-001 (`Quad`/`ProcessingState`)
**Production gate:** on-device manual attestation (INT-002 Verification — draggable handles on a real captured still)
**Status:** ACCEPTED

## Precedence

Reviewed for conflicts before writing. Resolve residual ambiguity by: (1) Constraints & Decisions, (2) Domain Constraints, (3) Quality Checklists. Do not implement from Spec Context. Stop and ask on any unresolved conflict.

## Goal

The processing screen overlays four **draggable corner handles** on the captured still. The overlay's handle positions are **seeded once** from `detectQuad` (or the default inset box on `nil`), thereafter the **user's dragged `Quad` is the source of truth** — detection is **never re-run over the user's edits**. Dragging updates normalized points on `@MainActor`; no pixels are touched on the main thread.

## Out of Scope

- Detection itself + the fallback box `Quad` — IB-005 (consumed here as the seed).
- Coordinate-space math — IB-004 (consumed to place/read handles; no inline flips).
- `Quad` ordering/validation — IB-001 (a dragged corner update reconstructs a validated `Quad`).
- Perspective warp / white balance / export / the Result screen — INT-003/004.

## Escalation Protocol

Stop and ask when a decision needs information not in this IB, when a file outside Files to Modify must change, or when a Done When criterion cannot be met without out-of-scope work. Do not guess. In particular: do **not** re-run `detectQuad` after the user has begun dragging (ADR-007), and do **not** run a pixel conversion on the main thread (ADR-005).

## Spec Context

**Traceability only.** WS-004 §What This Does (Adjustment), §Business Rules BR7–BR9, §Domain Constraints (seed-not-authority, drag isolation, cancellation), §Failure Behavior. WS-003 (the conversions consumed to position handles). WS-001 (`ProcessingState.quad`, `Quad` failable init). ADR-007 (seed-not-authority).

## Files to Modify

*Xcode-side paths (built on the Mac). Structure per council plan §4.*

| File | Change |
|------|--------|
| `ClassCam/Processing/ProcessingScreen.swift` | On entry to `processing`, kick `detectQuad` from `.task {}`; seed the overlay quad from the result (or `defaultInsetQuad` on `nil`) **once**; hold the adjustable quad in `ProcessingState`. Do not re-run detection over edits. |
| `ClassCam/Processing/CornerHandleOverlay.swift` | New: a SwiftUI overlay drawing the quad edges + four draggable handles; positions handles by converting the normalized `Quad` corners to view points via `ClassCamCore/Geometry` (IB-004); a `DragGesture` per handle updates the grabbed corner. Ephemeral gesture state (grabbed-corner index, drag offset) is local `@State`. |
| `ClassCam/Processing/CornerDragModel.swift` | New (or a small extension): applies a normalized drag delta to one corner and reconstructs a validated `Quad` (WS-001 init) for `ProcessingState`; all `@MainActor`, normalized points only. |

## Reuse Inventory

| Capability | Location | Use instead of reimplementing |
|------------|----------|-------------------------------|
| `ClassCamCore/Geometry` conversions | `ClassCamCore` (IB-004) | Convert the normalized `Quad` corners to view points to place handles, and the drag point back to normalized; never inline a flip. |
| `detectQuad(in:)` + `defaultInsetQuad` | `ClassCam/Imaging` + `ClassCamCore` (IB-005) | Seed the overlay from these; don't re-implement detection or invent a second fallback. |
| `Quad` failable init | `ClassCamCore` (IB-001) | Reconstruct a validated `Quad` after a drag; don't hold four loose points that could go degenerate. |
| `ProcessingState.quad` | `ClassCamCore` (IB-001/WS-001) | The adjusted quad lives in the existing state; don't add a parallel store. |
| SwiftUI `DragGesture`, `@State`, `.task {}` | SwiftUI / Swift concurrency | Use native gesture + view lifecycle; cancel detection on disappear via `.task {}`; no detached `Task`. |

## Domain Constraints

| Constraint | Value |
|------------|-------|
| Seed-not-authority | Detection seeds handles once; the dragged `Quad` is the source of truth; detection never re-run over edits (ADR-007) |
| Drag isolation | Drag updates normalized points on `@MainActor`; no pixel conversion on main (ADR-005) |
| Conversions | route through `ClassCamCore/Geometry` (IB-004) — no inline flips |
| Fallback | on `detectQuad == nil`, seed from `defaultInsetQuad` (IB-005) — never a dead screen |
| Cancellation | `detectQuad` kicked from `.task {}`; cancelled on leaving processing |
| Validity | a dragged corner reconstructs a validated `Quad` (WS-001 init) |
| Compute device / dtype | n/a — normalized-point UI math, not tensors |

## Environment Prerequisites

| Prerequisite | Probe command | Required |
|--------------|---------------|----------|
| A real iPad with a captured still (drag feel + handle placement have no automatable surface — INT-002 Verification) | manual on-device run | yes |

## Do Not Touch

| Function/File | Reason |
|---------------|--------|
| `ClassCamCore/Geometry` conversions | Owned by IB-004; consume, don't modify. |
| `ImagingService.detectQuad` / `defaultInsetQuad` | Owned by IB-005; consume as the seed. |
| `Quad` ordering/validation | Owned by IB-001. |
| Imaging render / result / export | Later Intents (INT-003/004). |

## Governing ADRs

| ADR | Title |
|-----|-------|
| ADR-007 | Coordinate-space conversion discipline (detection seeds; dragged `Quad` authoritative; never re-run over edits) |
| ADR-005 | Swift 6 strict-concurrency actor-isolated services (drag is `@MainActor` normalized math, no pixels on main) |
| ADR-004 | Build the UI in SwiftUI |
| ADR-002 | Apple-native Vision + Core Image |

## Constraints & Decisions

- **Seed once, then hands off:** on entry to `processing`, `detectQuad` seeds the overlay quad exactly once (or `defaultInsetQuad` on `nil`); after that the user's dragged `Quad` is authoritative and detection is never re-run over edits — a dragged corner is never snapped back (WS-004 BR8, ADR-007).
- **Drag stays on main, in normalized space:** a handle drag updates the grabbed corner's normalized point on `@MainActor` via the IB-004 conversions; no pixel conversion runs on main (WS-004 BR7, ADR-005).
- **Validated after every drag:** the updated four corners reconstruct a `Quad` through the WS-001 failable init so `ProcessingState.quad` is always valid (WS-004 BR7 / WS-001 BR6–BR7).
- **Cancellable seed:** `detectQuad` is kicked from the view's `.task {}` and cancelled on disappear; no detached `Task` outlives the view (WS-004 BR9; ADR-005 / council §2).
- **Ephemeral gesture state is local:** grabbed-corner index and in-flight drag offset are view-local `@State`, not app state (council §8).

## Interface Contracts

- `dekspec/interface-contracts/IC-001-captured-image.md` — the still the overlay sits on + the dims IB-004 converts against (traceability only).

## Quality Checklists

- SwiftUI gesture correctness; strict-concurrency cleanliness (no pixel work on main); on-device attestation checklist for handle placement + drag feel + no-re-detect-over-edits.

## Test Promotion Criteria

Promotion refs: WS-004 BR7 (drag updates normalized, no pixels on main), BR8 (seed-not-authority — dragged corner never overwritten), BR9 (cancellation) — all on-device attestation (drag/overlay have no automatable surface in this repo). The `Quad`-reconstruction validity leans on the IB-001 `Quad` init contract already unit-tested.

## Test Layout

- On-device attestation checklist (no automatable surface for the SwiftUI drag/overlay behaviors — INT-002 Verification). The underlying pure pieces (coordinate conversions, `Quad` validity, default box) are unit-tested in IB-004/IB-005.

## Done When

- [ ] On device: on entry to processing, four handles appear seeded on the detected quad (or the default inset box when detection returns `nil`) — verified by on-device attestation (WS-004 BR6, BR8).
- [ ] On device: dragging any handle refines that corner smoothly; the adjusted quad becomes what the (later) warp will consume — verified by on-device attestation (WS-004 BR7).
- [ ] On device: a dragged corner is **never** snapped back by a re-detection; detection is not re-run over the user's edits — verified by on-device attestation (WS-004 BR8, ADR-007).
- [ ] Dragging touches only normalized-point math on `@MainActor`; no pixel conversion runs on main — verified by code review + on-device attestation (main-thread checker clean; WS-004 BR7 / ADR-005).
- [ ] Handle positions are computed through `ClassCamCore/Geometry` (no inline flip); each drag reconstructs a validated `Quad` — verified by code review (ADR-007; WS-001 init).
- [ ] Leaving processing cancels an in-flight `detectQuad`; no detached `Task` leaks — verified by code review + on-device attestation (WS-004 BR9).
- [ ] No new unit-testable pure logic is introduced here that lacks a test; the pure pieces it relies on (conversions, `Quad`, default box) are already tested in IB-004/IB-005 — verified by review.
- [ ] All pre-existing tests continue to pass — verified by test run.

**Golden State Transitions**

| Input | Expected Output | Verified by |
|-------|----------------|-------------|
| enter processing, `detectQuad` returns a quad | four handles seeded on the detected corners | on-device attestation |
| enter processing, `detectQuad` returns `nil` | four handles seeded on the ~10% default inset box | on-device attestation |
| drag a handle, then a re-detection scenario occurs | the dragged corner stays put (never snapped back) | on-device attestation |
| drag a handle to a new normalized point | `ProcessingState.quad` updates to a validated `Quad`; no pixel work on main | on-device attestation + code review |

## Open Issues

- *None.*

## Amendment Log

| Date | Type | Change | Author |
|------|------|--------|--------|
| 2026-08-10 | Substantive | Initial authoring — IB-006 draggable corner-handle overlay + seed-from-detection / drag-is-source-of-truth wiring (INT-002 IU3, from WS-004; ADR-007 seed-not-authority). | noreply@anthropic.com |
| 2026-08-10 | Substantive | Accepted as INT-002 IB. | noreply@anthropic.com |
