# INT-003: Perspective-correct, crop to aspect ratio, and export

## Status

IMPLEMENTING

## Intent type

feature

## Autonomy

manual

## Risk Tier

default

## Branch

`int/INT-003-correct-crop-export`

## Mission

MSN-001

## Source

none

## Created

2026-08-09

## Modified

2026-08-10

## Linked Architecture Elements

- AE-001: ClassCam App — realises the perspective-correction, aspect-ratio, and export stages that produce and deliver the finished slide.

## Motivation

Once the user has placed the four corners, the point of the app is to turn that skewed region into a clean, square slide they can drop into their notes — which requires straightening it to a chosen aspect ratio and getting it out of the app. Without this slice the corner work produces nothing usable. This delivers the payoff: a rectilinear slide on the clipboard and in the share sheet.

## Desired Outcome

When the user taps Done, the app perspective-corrects and crops the image to the selected aspect ratio — 16:9 by default or 4:3 when toggled, with the choice remembered for the next photo — renders the finished slide, automatically copies it to the clipboard, and offers a Share action; the user can then re-edit or start over.

## Type-specific required fields

### `feature` — Desired Outcome

The Desired Outcome above describes the new user-observable behaviour: perspective correction, a remembered 16:9/4:3 aspect toggle, automatic clipboard copy, and a share action.

## Components affected

- `dekspec/architecture-elements/AE-001-classcam-app.md`

*Blast-radius note (spec-only repo).* ClassCam's runnable code is built in Xcode on a Mac, outside this repository; INT-003's diff here is confined to the AE-001 revision (relinked centrally) plus the spec artifacts this Intent authors (ADR-008, WS-005/006, IB-007/008/009, index files). The Xcode-side blast radius — the `ClassCamCore` package's `Output` slice and the `ClassCam` app target's `Imaging` / `Result` / `Processing` feature slices (council plan §4) — is realized off-repo and attested manually (see Verification).

## Coverage report

*Populated by `--analyze`. Greenfield slice: the council plan (`historical-artifacts/classcam-swift-council-plan.md` §3.1/§3.3) is the design input; both gaps below are resolved inside this Intent (not deferred), so neither becomes a blocking `P1` Open Issue.*

| Gap | Source | Resolution | Status |
| --- | --- | --- | --- |
| **The single app-lifetime `CIContext` is unestablished.** ADR-005 fixes that `ImagingService` owns *the one* `CIContext`, but no artifact yet establishes it or the render-once discipline. INT-003 is the first Intent to render, so it must stand up the one context and the lazy-graph-rendered-once rule (council R4 — a `CIContext` per render is the performance cliff). | analyze — top-down coverage | **Resolve in this Intent.** Folded in as IU2 (`ImagingService.render`) and recorded as **ADR-008**; the context is created once, app-lifetime, on the actor. | closed |
| **The compose-once-render-once + output-verification decision is unrecorded.** The council's §3.3 spine — one lazy `CIImage` recipe (oriented → perspective-correct → white-balance slot → Lanczos-to-1920 → crop) rendered a *single time* through the one context into an sRGB `CGImage`, with produced dimensions and colour space asserted after render — is captured in no ADR. | analyze — ADR gap (D-check "depends on an undocumented decision") | **Resolve in this Intent.** Author **ADR-008** (compose one lazy `CIImage` graph, render once through the single reused `CIContext`, export sRGB at 1920 long edge, assert dims + colour space). Governs the render path for INT-003 and INT-004. | closed |

## Size assessment

*Populated by `--analyze`. ADRs are not size-capped; only new **AEs** count against the New-L1 cap.*

| Cap | Limit | Measured | Verdict |
| --- | --- | --- | --- |
| Implementation Units (IBs / direct beads) | ≤ 3 | 3 (IU1 `ClassCamCore` Output sizing/crop math · IU2 `ImagingService.render` · IU3 export + result + aspect persistence) | PASS |
| Components affected | ≤ 3 | 1 (AE-001) | PASS |
| New L1 artifacts (AEs) | ≤ 1 | 0 (AE-001 revised centrally, not new; ADR-008 is an ADR, uncapped) | PASS |
| New + revised L2 artifacts (WSes + ICs) | ≤ 3 | 2 (WS-005 new · WS-006 new; consumes existing IC-001, no new IC) | PASS |
| Coverage gaps | ≤ 2 | 2 (both resolved in-Intent) | PASS |

## Layer impact analysis

*Populated by `--analyze`. IU → WS fan-in footnote consumed by `--decompose`.*

| Layer | Artifact | Action |
| --- | --- | --- |
| L1 (Architecture & Decisions) | AE-001 (ClassCam App) — detail the correction/crop/export stages + reference ADR-008 (relinked centrally later) | revise |
| L1 (Architecture & Decisions) | ADR-008 — compose one lazy `CIImage` graph, render once through the single reused `CIContext`; export sRGB at 1920 long edge | new |
| L2 (Specification) | WS-005 — render pipeline + output sizing (oriented → perspective-correct → Lanczos-to-1920 → crop; render once; assert dims + colour) | new |
| L2 (Specification) | WS-006 — export (clipboard PNG + share sheet) + result screen + remembered aspect toggle | new |
| L3 (Implementation) | IB-007 (`ClassCamCore` Output sizing/crop math) · IB-008 (`ImagingService.render`) · IB-009 (export + result + aspect persistence) | new |
| L4 (Construction) | 5 beads (see decomposition trail) | new |

**Decomposition trail (`--decompose`, 5 beads in `.beads/`):**

| Bead | IB | Unit | Blocks-on |
| --- | --- | --- | --- |
| `b1` | IB-007 | Output sizing + crop-rect math (1920 long-edge for both aspects) + tests | — (ready) |
| `b2` | IB-008 | `ImagingService` owns the ONE `CIContext` + oriented→perspective-correct graph | — (ready) |
| `b3` | IB-008 | Lanczos→1920 + crop-to-aspect + render-once + assert dims/colour | b2 |
| `b4` | IB-009 | Export sRGB PNG to `UIPasteboard` + share sheet | b3 |
| `b5` | IB-009 | Aspect toggle `@AppStorage` + Result screen | b3 |

*The INT-003 Outcome Verification (Done → corrected, cropped, exported slide) is the acceptance of the terminal beads `b4`/`b5`, attested on-device; the output-dimension assertion (1920×1080 / 1920×1440) is a concrete checkable criterion carried by `b1` (unit) and `b3` (on-device render assertion).*

**IU → WS fan-in (for `--decompose`):**
- IU1 `ClassCamCore` Output sizing/crop math ← WS-005 (defines the pure sizing/crop-rect rules) — packaged as IB-007.
- IU2 `ImagingService.render` ← WS-005 (defines the render pipeline) + consumes IC-001 (`CapturedImage`) — packaged as IB-008.
- IU3 export + result + aspect persistence ← WS-006 (defines) — packaged as IB-009.

*Each IU is packaged as an Implementation Brief (rather than the single-WS direct-bead shortcut) because INT-003's "coding session" is a human building in Xcode: the IB's file list, sequencing, and Reuse Inventory are the load-bearing Mac-side build guide.*

## Verification

```yaml
verification:
  - name: corrected-slide-exported-in-selected-aspect
    cmd: echo MANUAL-VERIFY
    manual: true
    manual_rationale: ClassCam is built and run locally in Xcode; this specification repository has no executable app surface, so correction, cropping, and export are verified by manual on-device attestation.
```

## Outcome Verification

Given a placed four-corner quad and a selected aspect ratio, tapping Done yields a rectilinear cropped slide in that ratio, copied to the clipboard automatically and available via the share sheet, with the aspect choice retained for the next photo. Verified by manual on-device attestation.

## Open Issues

- *None blocking.* Both `--analyze` coverage findings (unestablished single `CIContext`; unrecorded compose-once/render-once + output-verification decision) are resolved in-Intent — see Coverage report. No `P1` issues remain; the Intent is clean for PROPOSED.

## Amendment Log

| Date | Type | Change | Author |
|------|------|--------|--------|
| 2026-08-09 | Substantive | Initial authoring — Mission slice: perspective correction, aspect crop, and export. | noreply@anthropic.com |
| 2026-08-10 | Substantive | `--analyze`: populated Coverage report (2 gaps, both resolved in-Intent — single `CIContext` established here; ADR-008 records the compose-once/render-once + output-verification decision), Size assessment (all 5 caps PASS), Layer impact analysis (3 IUs → IB-007/008/009, 5 beads). DRAFT → PROPOSED. Design input: council plan §3.1/§3.3/§1.2/§7. | noreply@anthropic.com |
| 2026-08-10 | Substantive | `--decompose`: authored ADR-008 (ACCEPTED), WS-005/006 (ACCEPTED), IB-007/008/009 (ACCEPTED), 5 beads (b1–b5). PROPOSED → ACCEPTED. Design input: council plan. | noreply@anthropic.com |
| 2026-08-10 | Substantive | --analyze complete: coverage/size/layer populated, all caps PASS | jeffhaskin1@gmail.com |
| 2026-08-10 | Substantive | Accepted to drive INT-003 to beads for Mac build | noreply@anthropic.com |
| 2026-08-10 | Substantive | Decomposed: ADR-008, WS-005/006, IB-007/008/009 (ACCEPTED) + 5 beads (br-qls,3ip,0yv,kmo,vc8). | noreply@anthropic.com |
