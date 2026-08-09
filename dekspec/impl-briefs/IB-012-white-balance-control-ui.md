# Implementation Brief: White-balance control UI + non-destructive state

**Spec:** `dekspec/working-specs/WS-007-tap-white-balance.md`
**Intent:** `dekspec/intents/INT-004-tap-white-balance.md`
**Source AEs:** AE-001
**Depends on:** IB-011 (`sampleGains` + the white-balance render node), and the processing screen + `ProcessingState` established by INT-001/INT-002/INT-003.
**Production gate:** on-device manual attestation (INT-004 Verification — a real iPad; arm → tap → recolour, Reset, re-tap)
**Status:** ACCEPTED

## Precedence

Reviewed for conflicts before writing. Resolve residual ambiguity by: (1) Constraints & Decisions, (2) Domain Constraints, (3) Quality Checklists. Do not implement from Spec Context. Stop and ask on any unresolved conflict.

## Goal

The processing screen gains a **white-balance control**: the user arms it, taps a should-be-neutral spot, and the image recolours instantly; the `WhitePoint{tappedPoint, gains}` is stored in `ProcessingState` non-destructively — **Reset** drops it (restoring original colours) and **re-tap** overwrites it — with the tap on the `@MainActor` and all pixel work behind the `ImagingService` actor.

## Out of Scope

- The pure gain math (IB-010) and the `sampleGains`/render node (IB-011) — this IB wires the UI and state flow that call them.
- Capture, corner detection/drag, perspective correction, output sizing, export (INT-001/002/003) — this IB adds only the white-balance control and its non-destructive state handling.
- Any new `ProcessingState` field — `whitePoint` already exists (WS-001); this IB sets and clears it, it does not add to the type.

## Escalation Protocol

Stop and ask when a decision needs information not in this IB, when a file outside Files to Modify must change, or when a Done When criterion cannot be met without out-of-scope work. Do not guess. In particular: do **not** bake the corrected pixels into a stored bitmap to "make Reset work."

## Spec Context

**Traceability only.** WS-007 §What This Does (piece 3), §Business Rules BR6–BR8, §Domain Constraints (non-destructive storage, actor isolation). ADR-009 §Decision (non-destructive storage). WS-001 BR5 (`whitePoint` starts nil).

## Files to Modify

*Xcode-side paths (built on the Mac). Structure per council plan §4.*

| File | Change |
|------|--------|
| `ClassCam/Processing/WhiteBalanceControl.swift` | The arm toggle + tap-target affordance: a local `@State` "armed" flag; while armed, a tap on the image yields a normalized `CGPoint`. |
| `ClassCam/Processing/ProcessingScreen.swift` | Wire the control into the processing screen: on tap → `await imaging.sampleGains(...)` → store `WhitePoint` in `ProcessingState` → `await imaging.render(...)`; add the **Reset** action (set `whitePoint = nil`) and re-tap (overwrite); recolour from the render result. |

## Reuse Inventory

| Capability | Location | Use instead of reimplementing |
|------------|----------|-------------------------------|
| `TapGesture` / `spatialTapGesture` + `GeometryReader` | SwiftUI | Use SwiftUI's tap + geometry to get the normalized point; do not hand-roll hit-testing. |
| `@State` / `@Environment(AppModel.self)` | SwiftUI + Observation | Arm flag is ephemeral local `@State`; the `ProcessingState`/`whitePoint` lives on the injected `AppModel` (ADR-005). |
| `sampleGains` / `render` | `ImagingService` (IB-011) | Call the actor methods; the screen never touches a `CIContext` or pixels. |
| `WhitePoint` / `WhiteBalanceGains` / `ProcessingState` | `ClassCamCore` (IB-001) | Reuse the existing value types + the `whitePoint` slot; do not add fields. |

## Domain Constraints

| Constraint | Value |
|------------|-------|
| Non-destructive storage | `WhitePoint` in `ProcessingState.whitePoint`; Reset = nil, re-tap = overwrite; no baked bitmap |
| Tap point | Normalized `CGPoint` (0–1) on the `@MainActor` |
| Actor isolation | `sampleGains`/`render` are `await` calls; the screen touches no pixels (ADR-005) |
| Compute device / dtype | n/a — SwiftUI view + value-type state |

## Environment Prerequisites

| Prerequisite | Probe command | Required |
|--------------|---------------|----------|
| A real iPad (the visual recolour + interaction feel are on-device — council R5) | manual on-device run | yes |

## Do Not Touch

| Function/File | Reason |
|---------------|--------|
| `ClassCamCore/Color` gain math | Owned by IB-010. |
| `ClassCam/Imaging` (`sampleGains` + node) | Owned by IB-011; call it, don't modify. |
| `ProcessingState` type definition | Owned by IB-001; set/clear `whitePoint`, do not add fields. |
| Corner-drag / detection UI | Owned by INT-002. |

## Governing ADRs

| ADR | Title |
|-----|-------|
| ADR-009 | Compute tap-to-white-balance in linear light and store it non-destructively |
| ADR-004 | Build the UI in SwiftUI |
| ADR-005 | Swift 6 strict-concurrency actor-isolated services |

## Constraints & Decisions

- **Non-destructive, no baked bitmap:** the correction is stored as `WhitePoint{tappedPoint, gains}` in `ProcessingState.whitePoint`. **Reset** sets it to `nil`; **re-tap** overwrites it. Both are cheap graph rebuilds via a re-`render` — never bake corrected pixels into a stored image (ADR-009 / WS-007 BR6, BR7).
- **Flow stays on the right actors:** the tap runs on the `@MainActor` and yields a normalized `CGPoint`; the screen then `await`s `imaging.sampleGains(...)`, stores the `WhitePoint`, and `await`s `imaging.render(...)`. The screen never sees a `CIContext` or a pixel (ADR-005 / WS-007 BR8).
- **Arm is ephemeral UI state:** the "armed" flag is local `@State` on the control, not part of `ProcessingState` — it is transient interaction state (council §8: ephemeral per-view state is local `@State`).
- **`whitePoint` starts nil:** on entry to `processing` there is no correction (WS-001 BR5); it appears only after an armed tap.

## Interface Contracts

- `dekspec/interface-contracts/IC-001-captured-image.md` — traceability only (the image whose spot is sampled).

## Quality Checklists

- SwiftUI interaction + strict-concurrency cleanliness (no pixel work on main; `.task`-scoped async); on-device attestation checklist.

## Test Promotion Criteria

Promotion refs: WS-007 BR6–BR8 (store `WhitePoint`, Reset, re-tap, actor isolation) are on-device attestation — no automatable UI/visual surface in this repo. The underlying gain math is covered off-device by IB-010.

## Test Layout

- On-device attestation checklist (no automatable SwiftUI/visual surface — INT-004 Verification).

## Done When

- [ ] Arming the control and tapping a should-be-neutral spot recolours the image so that spot reads neutral, instantly — verified by on-device attestation (WS-007 BR6, and the correction from IB-011).
- [ ] The tap stores `WhitePoint{tappedPoint, gains}` in `ProcessingState.whitePoint` — verified by on-device attestation (correction persists across a re-render) (WS-007 BR6).
- [ ] **Reset** sets `whitePoint = nil` and restores the original colours instantly, with no baked bitmap — verified by on-device attestation (WS-007 BR7).
- [ ] **Re-tap** on a different spot overwrites `whitePoint` and re-applies the correction to the new neutral — verified by on-device attestation (WS-007 BR6).
- [ ] The tap runs on the `@MainActor`, yields a normalized point, and all pixel work is behind `await imaging.*` calls — verified by code review + on-device main-thread-checker cleanliness (WS-007 BR8).
- [ ] All pre-existing tests continue to pass — verified by test run.

**Golden State Transitions**

| Input | Expected Output | Verified by |
|-------|----------------|-------------|
| arm control, tap a blue-cast neutral spot | image recolours; that spot reads neutral; `whitePoint` set | on-device attestation |
| tap **Reset** | `whitePoint = nil`; original colours restored instantly | on-device attestation |
| re-tap a different neutral spot | `whitePoint` overwritten; recoloured to the new neutral | on-device attestation |
| enter `processing` (no tap yet) | `whitePoint == nil`; original colours | on-device attestation |

## Open Issues

- *None.*

## Amendment Log

| Date | Type | Change | Author |
|------|------|--------|--------|
| 2026-08-10 | Substantive | Initial authoring — IB-012 white-balance control UI (arm→tap→recolour) + non-destructive `ProcessingState.whitePoint` flow (store / Reset / re-tap) (INT-004 IU3, from WS-007). | noreply@anthropic.com |
| 2026-08-10 | Substantive | Accepted as INT-004 IB | noreply@anthropic.com |
