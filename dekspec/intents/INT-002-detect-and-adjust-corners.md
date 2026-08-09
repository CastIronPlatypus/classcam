# INT-002: Detect and adjust the presentation's four corners

## Status

IMPLEMENTING

## Intent type

feature

## Autonomy

manual

## Risk Tier

default

## Branch

`int/INT-002-detect-and-adjust-corners`

## Mission

MSN-001

## Source

none

## Created

2026-08-09

## Modified

2026-08-10

## Linked Architecture Elements

- AE-001: ClassCam App — realises the corner-detection and corner-adjustment stages of the processing screen.

## Motivation

A photograph of a display taken from a seat off to the side is trapezoidal, and the app cannot straighten it without knowing where the presentation's four corners are. Asking the user to place all four corners by hand every time would be slow and tedious. This slice gives the processing screen an auto-detected quad the user only nudges — and a dependable fallback when detection finds nothing.

## Desired Outcome

On the processing screen, the app automatically detects the presentation surface and overlays a box with four draggable corner handles on the detected corners; the user drags any handle to refine the quad, and when no clear rectangle is found the app presents a default inset box to position manually.

## Type-specific required fields

### `feature` — Desired Outcome

The Desired Outcome above describes the new user-observable behaviour: an auto-detected, draggable four-corner overlay with a default-box fallback.

## Components affected

- `dekspec/architecture-elements/AE-001-classcam-app.md`

*Blast-radius note (spec-only repo).* ClassCam's runnable code is built in Xcode on a Mac, outside this repository; INT-002's diff here is confined to the spec artifacts this Intent authors (ADR-007, WS-003/004, IB-004/005/006, index files) — AE-001 is referenced, not revised (its capture/processing structure already stands from INT-001; backlinks are relinked centrally later). The Xcode-side blast radius — new pure functions in the `ClassCamCore` `Geometry` slice, `ImagingService.detectQuad` in the `ClassCam` `Imaging` slice, and the draggable-corner overlay in the `Processing` slice (council plan §3.2, §4) — is realized off-repo and attested manually (see Verification).

## Coverage report

*Populated by `--analyze`. This slice is **greenfield**: the council plan (`historical-artifacts/classcam-swift-council-plan.md` §3.2) is the design input, and INT-001 already stood up the app shell, the `ClassCamCore` package, the `Stage`/`ProcessingState`/`Quad` value types, and the stub `ImagingService`. There is no orphaned code surface to archaeologize. Both gaps below are resolved inside this Intent (not deferred), so neither becomes a blocking `P1` Open Issue.*

| Gap | Source | Resolution | Status |
| --- | --- | --- | --- |
| **Coordinate-space discipline unrecorded.** INT-002's whole detection→drag→(later)warp flow lives across three coordinate spaces — Vision's normalized bottom-left, SwiftUI/UIKit's top-left points, and Core Image's bottom-left pixels — whose conversions are the app's single most likely source of a silent, plausible-but-wrong image (council R3). No ADR fixes where those conversions live, how they are validated, or that detection only seeds handles while the dragged quad is the source of truth. | analyze — ADR gap (D-check "depends on an undocumented decision") | **Resolve in this Intent.** Author **ADR-007** (convert between the three coordinate spaces once, in `ClassCamCore` `Geometry`, unit-tested on a known rectangle before device use; detection seeds, the dragged `Quad` is the source of truth, detection is never re-run over edits). Governs INT-002 and INT-003's warp. | closed |
| **`detectQuad` behaviour unspecified.** INT-001 shipped `ImagingService` as a stub returning canned values; INT-002's Desired Outcome ("automatically detects the presentation surface … and a default inset box when none is found") presumes a real `detectQuad(in:) -> Quad?` tuned for a glowing angled screen plus a mandatory fallback, which no prior spec defines. | analyze — top-down coverage (stub → real behaviour) | **Resolve in this Intent.** WS-004 defines the `VNDetectRectanglesRequest` tuning + the default-inset-box fallback + the seed-not-authority rule; IB-005 implements `detectQuad`, IB-006 the draggable overlay. | closed |

## Size assessment

*Populated by `--analyze`. ADRs are not size-capped; only new **AEs** count against the New-L1 cap.*

| Cap | Limit | Measured | Verdict |
| --- | --- | --- | --- |
| Implementation Units (IBs / direct beads) | ≤ 3 | 3 (IU1 `Geometry` conversions · IU2 `detectQuad` + fallback · IU3 draggable-handle overlay) | PASS |
| Components affected | ≤ 3 | 1 (AE-001, referenced not revised) | PASS |
| New L1 artifacts (AEs) | ≤ 1 | 0 (no new AE; ADR-007 is an ADR, uncapped) | PASS |
| New + revised L2 artifacts (WSes + ICs) | ≤ 3 | 2 (WS-003 new · WS-004 new; no new IC — INT-002 consumes IC-001 and the fixed `ClassCamCore` types) | PASS |
| Coverage gaps | ≤ 2 | 2 (both resolved in-Intent) | PASS |

## Layer impact analysis

*Populated by `--analyze`. IU → WS fan-in footnote consumed by `--decompose`.*

| Layer | Artifact | Action |
| --- | --- | --- |
| L1 (Architecture & Decisions) | AE-001 (ClassCam App) — corner detection + adjustment realise the processing stage AE-001 already frames | reference |
| L1 (Architecture & Decisions) | ADR-007 — convert between the three coordinate spaces once, in `ClassCamCore` `Geometry`, unit-tested; detection seeds, the dragged `Quad` is source of truth | new |
| L2 (Specification) | WS-003 — `ClassCamCore` `Geometry` coordinate-space conversion functions (round-trip-lossless pure surface) | new |
| L2 (Specification) | WS-004 — corner detection (`VNDetectRectanglesRequest` tuning) + default-inset-box fallback + draggable handles | new |
| L3 (Implementation) | IB-004 (`Geometry` conversions + round-trip tests) · IB-005 (`detectQuad` + fallback) · IB-006 (draggable-handle overlay) | new |
| L4 (Construction) | ~5 beads (see decomposition trail) | new |

**Decomposition trail (`--decompose`, ~5 beads in `.beads/`):**

| Bead | IB | Unit | Blocks-on |
| --- | --- | --- | --- |
| b1 | IB-004 | `Geometry` coordinate conversions (normalized-BL ↔ pixel-BL ↔ view-points-TL) + round-trip unit tests | — (ready) |
| b2 | IB-005 | `detectQuad` `VNDetectRectanglesRequest` wrapper tuned for a glowing angled screen | b1 |
| b3 | IB-005 | Default-inset-box (~10%) fallback when detection returns nothing | b2 |
| b4 | IB-006 | Draggable corner-handle overlay on normalized points (`@MainActor`) | b1 |
| b5 | IB-006 | Seed handles from detection + user-drag-is-source-of-truth wiring (detection never re-run over edits) | b4, b2 |

*IB-004's coordinate round-trips are pure-logic beads with real off-device unit-test acceptance; IB-005 (Vision) and IB-006 (drag UI) have no automatable surface in this repo and are attested on-device (council §7).*

**IU → WS fan-in (for `--decompose`):**
- IU1 `Geometry` conversions ← WS-003 (defines) — packaged as IB-004.
- IU2 `detectQuad` + fallback ← WS-004 (defines) — packaged as IB-005.
- IU3 draggable-handle overlay ← WS-004 (defines the seed/source-of-truth + drag contract) — packaged as IB-006.

*Each IU is packaged as an Implementation Brief (rather than the single-WS direct-bead shortcut) because INT-002's "coding session" is a human building in Xcode: the IB's file list, sequencing, and Reuse Inventory are the load-bearing Mac-side build guide.*

## Verification

```yaml
verification:
  - name: corners-detected-and-draggable-with-fallback
    cmd: echo MANUAL-VERIFY
    manual: true
    manual_rationale: ClassCam is built and run locally in Xcode; this specification repository has no executable app surface, so corner detection and adjustment are verified by manual on-device attestation.
```

## Outcome Verification

On a captured still of a presentation surface, the app overlays four draggable corner handles on the detected corners; each handle can be dragged to refine the quad, and when detection finds no rectangle a default inset box appears instead. Verified by manual on-device attestation.

## Open Issues

- *None blocking.* Both `--analyze` coverage findings (unrecorded coordinate-space discipline; unspecified `detectQuad` behaviour) are resolved in-Intent — see Coverage report. No `P1` issues remain; the Intent is clean for PROPOSED.

## Amendment Log

| Date | Type | Change | Author |
|------|------|--------|--------|
| 2026-08-09 | Substantive | Initial authoring — Mission slice: corner detection and adjustment. | noreply@anthropic.com |
| 2026-08-10 | Substantive | `--analyze`: greenfield slice (council plan §3.2 is the design input; INT-001 stood up the shell + `ClassCamCore` types + stub `ImagingService` — no archaeology surface). Populated Coverage report (2 gaps, both resolved in-Intent — ADR-007 records the coordinate-space discipline; WS-004 specifies `detectQuad` + fallback), Size assessment (all 5 caps PASS), Layer impact analysis (3 IUs → IB-004/005/006). DRAFT → PROPOSED. | noreply@anthropic.com |
| 2026-08-10 | Substantive | Promoted PROPOSED → ACCEPTED (engineer directive: drive INT-002 to beads for the Mac build). Design input: council plan `historical-artifacts/classcam-swift-council-plan.md` §3.2/§1.2/§7. | noreply@anthropic.com |
| 2026-08-10 | Substantive | Accepted to drive INT-002 to beads for the Mac build (council plan §3.2). | noreply@anthropic.com |
| 2026-08-10 | Substantive | Decomposed: ADR-007, WS-003/004, IB-004/005/006 (ACCEPTED) + 5 beads (br-gg7,lhf,kgu,es4,s0m). | noreply@anthropic.com |
