# Implementation Brief: ImagingService sampleGains + white-balance render node

**Spec:** `dekspec/working-specs/WS-007-tap-white-balance.md`
**Intent:** `dekspec/intents/INT-004-tap-white-balance.md`
**Source AEs:** AE-001
**Depends on:** IB-010 (the pure gain math), and the `ImagingService` actor + shared `CIContext` + `render(_:quad:whitePoint:aspect:)` signature established by INT-003.
**Production gate:** on-device manual attestation (INT-004 Verification — a real iPad; tapped spot reads neutral, no residual cast)
**Status:** ACCEPTED

## Precedence

Reviewed for conflicts before writing. Resolve residual ambiguity by: (1) Constraints & Decisions, (2) Domain Constraints, (3) Quality Checklists. Do not implement from Spec Context. Stop and ask on any unresolved conflict.

## Goal

`ImagingService` gains a `sampleGains(at:in:)` that samples the tapped pixel by rendering a **1×1 region through the shared `CIContext` in the image's tagged colour space**, and the existing render graph gains a **white-balance `CIColorMatrix` node** that applies the gains in `extendedLinearSRGB` (linear light) and encodes back to sRGB — filling the `whitePoint` slot INT-003 reserved, with sampling and rendering sharing the one app-lifetime `CIContext`.

## Out of Scope

- The pure per-channel gain arithmetic (IB-010) — this IB calls it; it does not reimplement `g_R=G/R` etc.
- The white-balance control UI and non-destructive `ProcessingState` flow (IB-012).
- Capture, corner detection, perspective correction, output sizing, export (INT-001/002/003) — this IB adds one node into the graph INT-003 authored and adds one sampling method; it does not restructure the rest of the pipeline.

## Escalation Protocol

Stop and ask when a decision needs information not in this IB, when a file outside Files to Modify must change, or when a Done When criterion cannot be met without out-of-scope work. Do not guess. In particular: do **not** create a second `CIContext`, and do **not** read a framework image pixel of unknown/untagged encoding to "get a colour."

## Spec Context

**Traceability only.** WS-007 §What This Does, §Business Rules BR2–BR5, BR8, §Domain Constraints (sampling space, colour space, single `CIContext`, actor isolation). ADR-009 §Decision. IC-001 (the tagged colour space the sample is interpreted in).

## Files to Modify

*Xcode-side paths (built on the Mac). Structure per council plan §4.*

| File | Change |
|------|--------|
| `ClassCam/Imaging/ImagingService.swift` | Add `func sampleGains(at point: CGPoint, in image: CapturedImage) async -> WhiteBalanceGains` — render a 1×1 region at the tapped point through the shared `CIContext` in the image's tagged colour space, read the linear-light channels, and delegate to `ClassCamCore/Color` (IB-010) for the gains. |
| `ClassCam/Imaging/WhiteBalanceNode.swift` | The white-balance graph node: a diagonal `CIColorMatrix` built from `WhiteBalanceGains`, inserted in the `extendedLinearSRGB` (linear-light) stage of the render graph, with encode-back-to-sRGB downstream. |
| `ClassCam/Imaging/RenderGraph.swift` | Wire the white-balance node into the existing graph at the `whitePoint` slot: when `whitePoint` is non-nil, apply the node; when nil, omit it. (Edits the INT-003 graph assembly; does not restructure the other nodes.) |

## Reuse Inventory

| Capability | Location | Use instead of reimplementing |
|------------|----------|-------------------------------|
| `CIColorMatrix` (diagonal per-channel scale) | Core Image | Use the framework colour-matrix filter for the diagonal gain; do not hand-roll a per-pixel loop. |
| Shared `CIContext` (one, app-lifetime) | `ImagingService` (INT-003) | Reuse the existing context for both the 1×1 sample and the render; never construct a second (council R4). |
| `CIContext.render(_:toBitmap:...)` / 1×1 read | Core Image | Use the framework's tagged-colour-space render to read the sampled pixel; do not read a `UIImage`/`CGImage` pixel of unknown encoding (council R2). |
| `extendedLinearSRGB` working space | Core Image / CoreGraphics colour spaces | Apply the matrix in the linear-light working space; do not multiply gamma-encoded sRGB (ADR-009). |
| White-balance gain math | `ClassCamCore/Color` (IB-010) | Call the pure `WhiteBalanceGains` computation; the actor does not reimplement the arithmetic. |
| `render(_:quad:whitePoint:aspect:)` signature | `ImagingService` (INT-003) | Fill the existing `whitePoint` slot; do not change the signature. |

## Domain Constraints

| Constraint | Value |
|------------|-------|
| Sampling space | 1×1 region rendered through the shared `CIContext` in the image's tagged colour space |
| Application space | Diagonal `CIColorMatrix` in `extendedLinearSRGB`, then encode to sRGB |
| Single `CIContext` | Sample + render reuse the one app-lifetime context (council R4) |
| Actor isolation | `sampleGains`/`render` are `async` actor methods; no pixel work on main (ADR-005) |
| Compute device / dtype | n/a — Core Image float pixels, no tensors or device pinning |

## Environment Prerequisites

| Prerequisite | Probe command | Required |
|--------------|---------------|----------|
| A real iPad (the simulator proves nothing for the visual colour result — council R5) | manual on-device run | yes |

## Do Not Touch

| Function/File | Reason |
|---------------|--------|
| `ClassCamCore/Color` gain math | Owned by IB-010; call it, don't reimplement. |
| `ClassCam/Processing` (UI + state) | Owned by IB-012. |
| The other render-graph nodes (orientation, perspective, scale, crop) | Owned by INT-003; this IB adds only the white-balance node. |
| The `render` signature | Fill the `whitePoint` slot; do not change the signature (INT-003). |

## Governing ADRs

| ADR | Title |
|-----|-------|
| ADR-009 | Compute tap-to-white-balance in linear light and store it non-destructively |
| ADR-002 | Apple-native Vision + Core Image |
| ADR-005 | Swift 6 strict-concurrency actor-isolated services |
| ADR-006 | ClassCamCore UI-free pure package |

## Constraints & Decisions

- **Sample in the tagged space, through the shared context:** `sampleGains` renders a 1×1 region at the tapped point through the one `CIContext` in the `CapturedImage`'s tagged colour space (IC-001), reads the linear-light channels, and hands them to `ClassCamCore/Color` (IB-010). Never read a `UIImage`/`CGImage` pixel of unknown encoding (council R2).
- **Apply gains in linear light:** the white-balance node is a diagonal `CIColorMatrix` applied in `extendedLinearSRGB`, with the graph encoding back to sRGB downstream (ADR-009). Applying the matrix in gamma-encoded sRGB is the residual-cast bug the divergence test (IB-010) exists to prevent.
- **One context, always:** both the 1×1 sample and the full render use the single app-lifetime `CIContext` owned by `ImagingService` (INT-003 / council R4). Never construct a second.
- **Fill the reserved slot:** when `render` is called with a non-nil `whitePoint`, insert the node; with nil, omit it entirely so the image renders original colours (WS-007 BR5). The signature is unchanged.
- **Off the main actor:** `sampleGains` and `render` are `async` `ImagingService` methods; only `Sendable` value types (`CGPoint`, `CapturedImage`, `WhitePoint`, `WhiteBalanceGains`, `Slide`) cross the actor line; no pixel work runs on main (ADR-005 / WS-007 BR8).

## Interface Contracts

- `dekspec/interface-contracts/IC-001-captured-image.md` — the tagged colour space the 1×1 sample is interpreted in (the producer obligation this IB relies on).

## Quality Checklists

- Core Image colour-management correctness (linear-light application); strict-concurrency cleanliness; single-`CIContext` code review; on-device attestation checklist.

## Test Promotion Criteria

Promotion refs: the gain arithmetic is covered off-device by IB-010 (WS-007 BR1). WS-007 BR2–BR5, BR8 (sampling, linear-light application, nil-omits-node, actor isolation) are on-device attestation + code review — no automatable surface for the Core Image visual result in this repo.

## Test Layout

- Gain arithmetic: covered by IB-010's `WhiteBalanceTests` (host toolchain).
- On-device attestation checklist (no automatable surface for the Core Image sampling/render — INT-004 Verification).

## Done When

- [ ] `sampleGains(at:in:)` renders a 1×1 region through the shared `CIContext` in the image's tagged colour space and returns green-normalized gains — verified by on-device attestation (tapped spot reads neutral after correction) + code review (tagged-space 1×1 render; no `UIImage` pixel read) (WS-007 BR2, BR3).
- [ ] The white-balance node applies the gains as a diagonal `CIColorMatrix` in `extendedLinearSRGB`, encoded back to sRGB — verified by on-device attestation (no residual cast) + code review that the matrix sits in the linear-light stage (WS-007 BR4).
- [ ] With `whitePoint == nil`, the render graph omits the white-balance node (original colours) — verified by on-device attestation (WS-007 BR5).
- [ ] Sampling and rendering reuse the single app-lifetime `CIContext`; no second context is constructed — verified by code review (WS-007 BR2/BR4, council R4).
- [ ] `sampleGains`/`render` run off the main actor; only value types cross the actor line — verified by code review + on-device main-thread-checker cleanliness (WS-007 BR8).
- [ ] All pre-existing tests continue to pass — verified by test run.

**Golden State Transitions**

| Input | Expected Output | Verified by |
|-------|----------------|-------------|
| tap a should-be-neutral (blue-cast) spot → `sampleGains` | green-normalized `WhiteBalanceGains` that neutralize it | on-device attestation |
| `render(..., whitePoint: nonNil, ...)` | white-balance `CIColorMatrix` node applied in `extendedLinearSRGB`, tapped spot reads neutral | on-device attestation |
| `render(..., whitePoint: nil, ...)` | node omitted; original colours | on-device attestation |
| both `sampleGains` and `render` | share the one `CIContext` (no second context) | code review |

## Open Issues

- *None.*

## Amendment Log

| Date | Type | Change | Author |
|------|------|--------|--------|
| 2026-08-10 | Substantive | Initial authoring — IB-011 ImagingService `sampleGains` (1×1 tagged-space sample) + white-balance `CIColorMatrix` render node (`extendedLinearSRGB`→sRGB), filling the INT-003 `whitePoint` slot (INT-004 IU2, from WS-007). | noreply@anthropic.com |
| 2026-08-10 | Substantive | Accepted as INT-004 IB | noreply@anthropic.com |
