# Implementation Brief: ImagingService.render — the one CIContext + the lazy CIImage graph

**Spec:** `dekspec/working-specs/WS-005-render-pipeline-output-sizing.md`
**Intent:** `dekspec/intents/INT-003-correct-crop-export.md`
**Source AEs:** AE-001
**Depends on:** IB-001 (model types), IB-007 (Output sizing/crop math), and the ClassCamCore Geometry conversions (INT-002, WS-003)
**Production gate:** on-device manual attestation (INT-003 Verification — render on a real iPad) + the post-render dimensions/colour-space assertion
**Status:** ACCEPTED

## Precedence

Reviewed for conflicts before writing. Resolve residual ambiguity by: (1) Constraints & Decisions, (2) Domain Constraints, (3) Quality Checklists. Do not implement from Spec Context. Stop and ask on any unresolved conflict.

## Goal

`ImagingService` owns **one** app-lifetime `CIContext` and implements `render(_:quad:whitePoint:aspect:) async throws -> Slide` as **one lazy `CIImage` graph** (oriented source → `CIPerspectiveCorrection` with pixel-space corners → white-balance colour-node slot → `CILanczosScaleTransform` to 1920 long edge → crop to aspect) rendered a **single time** through that context into an **sRGB `CGImage`**, whose dimensions (1920×1080 / 1920×1440) and colour space are **asserted** before it is wrapped in a `Slide`.

## Out of Scope

- The white-balance gain **math and colour node contents** (INT-004) — this IB leaves the optional colour-node **slot** (a `nil` `whitePoint` = no node) but does not compute gains.
- The output sizing/crop-rect math (IB-007) — this IB **calls** those pure functions.
- The coordinate-space conversions (INT-002, WS-003) — this IB **consumes** them to place corners in Core-Image pixel space; it does not re-derive them.
- Export, clipboard, share, aspect persistence, Result screen (IB-009).

## Escalation Protocol

Stop and ask when a decision needs information not in this IB, when a file outside Files to Modify must change, or when a Done When criterion cannot be met without out-of-scope work. Do not guess. In particular: do **not** construct a `CIContext` inside `render`, and do **not** feed `CIPerspectiveCorrection` normalized/view-space corners.

## Spec Context

**Traceability only.** WS-005 §What This Does, §Business Rules BR1–BR8, §Domain Constraints, §Failure Behavior. ADR-008 §Decision (one lazy graph, render once, one CIContext, sRGB, assert). IC-001 (the source `CapturedImage` — oriented, tagged).

## Files to Modify

*Xcode-side paths (built on the Mac). Structure per council plan §4.*

| File | Change |
|------|--------|
| `ClassCam/Imaging/ImagingService.swift` | The `actor` gains the **one** `CIContext` (created once, stored property) and the `render(_:quad:whitePoint:aspect:)` method; replaces any stub. |
| `ClassCam/Imaging/RenderGraph.swift` | Builds the lazy `CIImage` recipe: `.oriented(...)` first → `CIPerspectiveCorrection` (corners as `CIVector`s in CI pixel space) → optional white-balance node slot → `CILanczosScaleTransform` (via IB-007's scale) → crop (via IB-007's crop rect). |
| `ClassCam/Imaging/ImagingError.swift` | `enum ImagingError: Error { case renderFailed }` (extended by INT-004). |
| `ClassCam/Imaging/RenderOutput.swift` | Single `createCGImage` on the one context into sRGB; asserts produced dims + colour space, throws `ImagingError.renderFailed` on mismatch; wraps into `Slide`. |

## Reuse Inventory

| Capability | Location | Use instead of reimplementing |
|------------|----------|-------------------------------|
| `CIPerspectiveCorrection`, `CILanczosScaleTransform` | Core Image | Use the framework filters; don't hand-roll a warp or a resampler. |
| `CIContext` (`createCGImage(_:from:colorSpace:)`) | Core Image | The **one** app-lifetime context; render once. Never build a context per call. |
| `CIImage.oriented(_:)` / `CIVector` | Core Image | Apply the sealed orientation as the first node; pass corners as `CIVector`s. |
| `CGColorSpace(name: CGColorSpace.sRGB)` | CoreGraphics | The output colour space; assert the produced image carries it. |
| `longEdgeScale` / `outputSize` / `cropRect` | `ClassCamCore/Output` (IB-007) | Call the pure sizing/crop math; don't inline it on the GPU path. |
| coordinate conversions (normalized → CI pixel space) | `ClassCamCore/Geometry` (INT-002, WS-003) | Convert the `Quad` corners before `CIPerspectiveCorrection`; don't re-derive the Y-flip/scale. |
| `CapturedImage` | `ClassCamCore` (IB-001 / IC-001) | Read the oriented, tagged source; don't redefine a capture-result type. |

## Domain Constraints

| Constraint | Value |
|------------|-------|
| Orientation node | `.oriented(...)` as the first graph node |
| Corner space | `CIVector`s in Core-Image pixel space (converted via ClassCamCore Geometry) |
| WB slot | one optional colour node between perspective and scale; `nil` = absent |
| Long-edge size | `CILanczosScaleTransform` to exactly 1920 long edge |
| Render count | one lazy graph, one `createCGImage` |
| Context lifetime | the one app-lifetime `CIContext`; never per render (council R4) |
| Colour space | output `CGImage` is sRGB |
| Output assertion | dims (1920×1080 / 1920×1440) + colour space asserted after render |
| Compute device / dtype | n/a — Core Image on the one CIContext, no tensors |

## Environment Prerequisites

| Prerequisite | Probe command | Required |
|--------------|---------------|----------|
| A real iPad (Core Image render + on-device attestation — the simulator proves nothing for imaging quality; council R5) | manual on-device run | yes |

## Do Not Touch

| Function/File | Reason |
|---------------|--------|
| `ClassCamCore` Output/Geometry functions | Owned by IB-007 / INT-002; call them, don't reimplement. |
| White-balance gain computation | INT-004 fills the colour-node slot. |
| Export / clipboard / share / Result screen | Owned by IB-009. |
| `CaptureService` / model types | Owned by INT-001. |

## Governing ADRs

| ADR | Title |
|-----|-------|
| ADR-008 | One lazy CIImage graph, render once through the single CIContext, sRGB at 1920 long edge |
| ADR-005 | Swift 6 strict-concurrency actor-isolated services |
| ADR-002 | Apple-native Vision + Core Image |
| ADR-006 | ClassCamCore UI-free pure package |
| ADR-007 | Coordinate-space conversions (INT-002) |

## Constraints & Decisions

- **One CIContext, app-lifetime:** `ImagingService` holds the single `CIContext` as a stored property, created once. `render` **never** constructs a context (council R4 / ADR-008). This same context is reused by INT-004's sampling render.
- **Oriented first:** the sealed `CGImagePropertyOrientation` from the `CapturedImage` is applied as the **first** node via `.oriented(...)`, so everything downstream is upright (WS-005 BR1).
- **Corners in pixel space:** the `Quad`'s four normalized corners are converted into Core-Image pixel space via the ClassCamCore Geometry conversions (WS-003) and fed to `CIPerspectiveCorrection` as `CIVector`s (topLeft/topRight/bottomRight/bottomLeft). Never feed normalized/view-space corners (WS-005 BR2).
- **White-balance slot:** between perspective correction and the Lanczos scale sits a **single optional** colour node; when `whitePoint == nil` the node is omitted entirely, and the graph is otherwise identical. INT-004 fills the node (WS-005 BR7).
- **Scale + crop from pure math:** `CILanczosScaleTransform` uses `longEdgeScale` (IB-007) to hit exactly 1920 on the long edge; the crop uses `cropRect` (IB-007) for the selected aspect (WS-005 BR3–BR5).
- **Render once:** compose the whole lazy `CIImage` graph, then call `createCGImage` **once** on the one context into an sRGB colour space; no intermediate stage is separately rendered (WS-005 BR6 / ADR-008).
- **Assert the output:** after render, assert the `CGImage` is 1920×1080 (16:9) or 1920×1440 (4:3) **and** sRGB; on mismatch throw `ImagingError.renderFailed` rather than returning a `Slide` (WS-005 BR8).

## Interface Contracts

- `dekspec/interface-contracts/IC-001-captured-image.md` — the source still this render consumes (oriented, tagged colour space).

## Quality Checklists

- Core Image graph correctness; strict-concurrency cleanliness (only `CapturedImage`/`Slide` cross the actor line; `CIImage`/`CIContext`/`CGImage` never leave it); on-device attestation checklist; the render-once / one-context code-review gate (council R4).

## Test Promotion Criteria

Promotion refs: WS-005 BR3–BR5 are unit-tested in IB-007; this IB's render-time contract (BR1, BR2, BR6, BR7, BR8) is on-device attestation, with the **post-render dimensions/colour-space assertion** (BR8) a concrete checkable criterion (1920×1080 / 1920×1440 + sRGB).

## Test Layout

- Pure sizing/crop unit tests live in IB-007 (`ClassCamCore`).
- On-device attestation checklist for the render pipeline (no automatable app surface for the Core Image behaviors — INT-003 Verification), plus the in-code post-render assertion.

## Done When

- [ ] `ImagingService` holds exactly **one** `CIContext` (a stored property, created once); `render` constructs no context — verified by code review (council R4 gate).
- [ ] The graph applies `.oriented(...)` first, feeds pixel-space `CIVector` corners to `CIPerspectiveCorrection`, and omits the colour node when `whitePoint == nil` — verified by code review + on-device attestation (upright, correctly-warped result).
- [ ] The image is scaled to exactly 1920 on the long edge and cropped to the selected aspect using the IB-007 pure functions — verified by on-device attestation + IB-007 unit tests.
- [ ] The graph is materialized with a **single** `createCGImage` on the one context into sRGB — verified by code review (no intermediate render) + on-device attestation (no per-render stutter).
- [ ] After render, the produced `CGImage` is asserted to be 1920×1080 (16:9) or 1920×1440 (4:3) **and** sRGB; a mismatch throws `ImagingError.renderFailed` — verified on-device (correct render passes; a forced-wrong size in test scaffolding fires the assertion).
- [ ] Only `CapturedImage` (in) and `Slide` (out) cross the actor boundary; no `CIImage`/`CIContext`/`CGImage` leaks — verified by strict-concurrency build + code review.
- [ ] All new unit tests (IB-007) pass; no pre-existing tests break — verified by test run.

**Golden State Transitions**

| Input | Expected Output | Verified by |
|-------|----------------|-------------|
| render(quad, `nil` whitePoint, `.sixteenNine`) | sRGB `CGImage` of exactly 1920×1080; no colour node in graph | on-device attestation + post-render assertion |
| render(quad, `nil` whitePoint, `.fourThree`) | sRGB `CGImage` of exactly 1920×1440 | on-device attestation + post-render assertion |
| a render forced to produce ≠ target size (test scaffold) | throws `ImagingError.renderFailed` | on-device (assertion fires) |
| two consecutive renders | one shared `CIContext`, no context rebuilt | code review |

## Open Issues

- *None.*

## Amendment Log

| Date | Type | Change | Author |
|------|------|--------|--------|
| 2026-08-10 | Substantive | Initial authoring — IB-008 ImagingService.render: one CIContext + lazy CIImage graph, render once, assert dims/colour (INT-003 IU2, from WS-005 + ADR-008). | noreply@anthropic.com |
| 2026-08-10 | Substantive | Accepted as INT-003 IB | noreply@anthropic.com |
