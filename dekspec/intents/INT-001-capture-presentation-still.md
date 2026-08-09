# INT-001: Capture a presentation still with live preview and digital zoom

## Status

IMPLEMENTING

## Intent type

feature

## Autonomy

manual

## Risk Tier

default

## Branch

`int/INT-001-capture-presentation-still`

## Mission

MSN-001

## Source

none

## Created

2026-08-09

## Modified

2026-08-10

## Linked Architecture Elements

- AE-001: ClassCam App — realises the capture stage: live preview, digital-zoom framing, and full-resolution still capture.

## Motivation

Someone sitting off to the side of a room wants a picture of the slide on the front-of-room display, but the built-in camera gives them no framing help tuned to this task and no bridge into the correction flow that follows. Without a dedicated capture surface, the user cannot frame the presentation comfortably from their seat or hand a full-resolution still to the rest of the app. This first slice gives the app its front door: a camera screen the user aims and shoots from.

## Desired Outcome

The user opens ClassCam to a live camera preview, frames the presentation using a digital-zoom control, and taps a capture button to take a full-resolution still, which the app carries forward to the processing stage.

## Type-specific required fields

### `feature` — Desired Outcome

The Desired Outcome above describes the new user-observable behaviour: a live preview with a zoom control and a capture action that yields a full-resolution still.

## Components affected

- `dekspec/architecture-elements/AE-001-classcam-app.md`

*Blast-radius note (spec-only repo).* ClassCam's runnable code is built in Xcode on a Mac, outside this repository; INT-001's diff here is confined to the AE-001 revision plus the spec artifacts this Intent authors (ADR-005/006, WS-001/002, IC-001, IB-001/002/003, index files). The Xcode-side blast radius — the `ClassCamCore` package and the `ClassCam` app target's App / Capture feature slices (council plan §4) — is realized off-repo and attested manually (see Verification).

## Coverage report

*Populated by `--analyze`. Both gaps are resolved inside this Intent (not deferred), so neither becomes a blocking `P1` Open Issue.*

| Gap | Source | Resolution | Status |
| --- | --- | --- | --- |
| **Walking-skeleton / app-shell prerequisite.** INT-001's Desired Outcome ("the app carries the still forward to the processing stage") presumes an app shell — the `Stage` state machine, the `ClassCamCore` Sendable value types, the `@Observable AppModel`, three screens, and a stub `ImagingService` — that no prior Intent stands up (council plan §5 Step 0, "walking skeleton before any feature"). | analyze — top-down coverage | **Resolve in this Intent.** Folded in as IU1 (`ClassCamCore` model foundation) + IU2 (app shell). INT-001 is MSN-001's First Intent and its own Motivation frames it as "the app's front door" — the doorframe lands with it. | closed |
| **Foundational architecture decisions unrecorded.** The council plan's spine — Swift 6 strict concurrency with actor-isolated services (one `@MainActor @Observable AppModel` + `CaptureService`/`ImagingService` actors, only `Sendable` value types across actor lines), and a UI-free `ClassCamCore` package for correctness-critical pure logic — is captured in no ADR. ADR-001…004 cover form / frameworks / on-device / SwiftUI, but not the concurrency + package architecture every child Intent depends on. | analyze — ADR gap (D-check "depends on an undocumented decision") | **Resolve in this Intent.** Author **ADR-005** (Swift 6 strict-concurrency actor-isolated services + `@Observable AppModel`/`Stage`) and **ADR-006** (`ClassCamCore` pure package); revise AE-001 to reference both. These govern all of MSN-001. | closed |

## Size assessment

*Populated by `--analyze`. ADRs are not size-capped; only new **AEs** count against the New-L1 cap.*

| Cap | Limit | Measured | Verdict |
| --- | --- | --- | --- |
| Implementation Units (IBs / direct beads) | ≤ 3 | 3 (IU1 `ClassCamCore` foundation · IU2 app shell · IU3 `CaptureService`) | PASS |
| Components affected | ≤ 3 | 1 (AE-001) | PASS |
| New L1 artifacts (AEs) | ≤ 1 | 0 (AE-001 revised, not new; ADR-005/006 are ADRs, uncapped) | PASS |
| New + revised L2 artifacts (WSes + ICs) | ≤ 3 | 3 (WS-001 new · WS-002 new · IC-001 new) | PASS |
| Coverage gaps | ≤ 2 | 2 (both resolved in-Intent) | PASS |

## Layer impact analysis

*Populated by `--analyze`. IU → WS fan-in footnote consumed by `--decompose`.*

| Layer | Artifact | Action |
| --- | --- | --- |
| L1 (Architecture & Decisions) | AE-001 (ClassCam App) — detail the capture stage's internal structure + reference new ADRs | revise |
| L1 (Architecture & Decisions) | ADR-005 — Swift 6 strict-concurrency actor-isolated services (`@Observable AppModel` + `Stage`; `CaptureService`/`ImagingService` actors) | new |
| L1 (Architecture & Decisions) | ADR-006 — `ClassCamCore` UI-free pure Swift package for correctness-critical logic | new |
| L2 (Specification) | WS-001 — `ClassCamCore` model + app-shell `Stage` state-machine behavior | new |
| L2 (Specification) | WS-002 — `CaptureService` capture behavior (session, preview, permissions, zoom, full-res still) | new |
| L2 (Specification) | IC-001 — `CapturedImage` contract (Capture → Imaging boundary value type) | new |
| L3 (Implementation) | IB-001 (`ClassCamCore` foundation) · IB-002 (app shell) · IB-003 (`CaptureService`) | new |
| L4 (Construction) | 7 beads (see decomposition trail) | new |

**Decomposition trail (`--decompose`, 7 beads in `.beads/`):**

| Bead | IB | Unit | Blocks-on |
| --- | --- | --- | --- |
| `br-4dq` | IB-001 | ClassCamCore package + Sendable model value types | — (ready) |
| `br-hvd` | IB-001 | Quad ordering + validation (failable init) + tests | br-4dq |
| `br-6u6` | IB-002 | App skeleton: build config + AppModel + ContentView + stub services | br-4dq |
| `br-vv9` | IB-002 | Three screen placeholders + AppModel transition tests | br-6u6 |
| `br-s2z` | IB-003 | Pure `clampZoom` helper + tests (ClassCamCore) | br-4dq |
| `br-ph0` | IB-003 | `CaptureService` actor: session/serial-queue/permissions/zoom + no-camera test | br-4dq, br-s2z, br-6u6 |
| `br-8px` | IB-003 | Still capture: delegate deep-copy (R1) + colour tag (R2) + orientation + preview wiring | br-ph0, br-vv9 |

*The INT-001 Outcome Verification (capture → advance to processing) is the acceptance of the terminal bead `br-8px`, attested on-device.*

**IU → WS fan-in (for `--decompose`):**
- IU1 `ClassCamCore` foundation ← WS-001 (defines) — packaged as IB-001.
- IU2 app shell ← WS-001 (consumes the state machine) — packaged as IB-002.
- IU3 `CaptureService` ← WS-002 (defines) + consumes IC-001 (`CapturedImage`) — packaged as IB-003.

*Each IU is packaged as an Implementation Brief (rather than taking the single-WS direct-bead shortcut) because INT-001's "coding session" is a human building in Xcode: the IB's file list, sequencing, and Reuse Inventory are the load-bearing Mac-side build guide.*

## Verification

```yaml
verification:
  - name: capture-produces-full-resolution-still
    cmd: echo MANUAL-VERIFY
    manual: true
    manual_rationale: ClassCam is built and run locally in Xcode; this specification repository has no executable app surface, so capture behaviour is verified by manual on-device attestation.
```

## Outcome Verification

On the capture screen, framing the presentation with the zoom control and tapping the capture button produces a full-resolution still that advances to the processing screen. Verified by manual on-device attestation (ClassCam is built and run locally; no automatable surface exists in this repository).

## Open Issues

- *None blocking.* Both `--analyze` coverage findings (walking-skeleton prerequisite; unrecorded foundational ADRs) are resolved in-Intent — see Coverage report. No `P1` issues remain; the Intent is clean for PROPOSED.

## Amendment Log

| Date | Type | Change | Author |
|------|------|--------|--------|
| 2026-08-09 | Substantive | Initial authoring — first Mission slice: the capture screen. | noreply@anthropic.com |
| 2026-08-09 | Substantive | `--analyze`: populated Coverage report (2 gaps, both resolved in-Intent — app-shell folds in as IU1/IU2; ADR-005/006 to record the concurrency + `ClassCamCore` decisions), Size assessment (all 5 caps PASS), Layer impact analysis (3 IUs → IB-001/002/003). DRAFT → PROPOSED. Design input: council plan (`historical-artifacts/classcam-swift-council-plan.md`). | noreply@anthropic.com |
| 2026-08-09 | Substantive | Promoted PROPOSED to ACCEPTED via /write-intent --accept (engineer directive: drive INT-001 to beads for Mac build) | noreply@anthropic.com |
| 2026-08-10 | Substantive | Decomposed into IB-001/002/003 (ACCEPTED) + 7 beads (br-4dq,hvd,6u6,vv9,s2z,ph0,8px). WS-001/002 + IC-001 ACCEPTED; ADR-005/006 authored. Design input: council plan. | noreply@anthropic.com |
