# INT-004: Correct the display colour cast by tapping a neutral spot

## Status

IMPLEMENTING

## Intent type

feature

## Autonomy

manual

## Risk Tier

default

## Branch

`int/INT-004-tap-white-balance`

## Mission

MSN-001

## Source

none

## Created

2026-08-09

## Modified

2026-08-10

## Linked Architecture Elements

- AE-001: ClassCam App — realises the white-point correction stage on the processing screen.

## Motivation

Photographs of a lit display come out with a colour cast — whites read blue — because the camera balances for room light while the screen emits its own cooler light, so a straightened slide still looks wrong. The user needs a direct way to tell the app which spot should be neutral and have the colour fixed to match. This slice lets the user correct the cast by tapping a should-be-white spot.

## Desired Outcome

On the processing screen the user taps a White Balance control and then taps a spot on the image that should be neutral; the app samples that spot and re-colours the image instantly so the tapped spot becomes neutral, non-destructively, allowing the user to re-tap a different spot or Reset to the original colours.

## Type-specific required fields

### `feature` — Desired Outcome

The Desired Outcome above describes the new user-observable behaviour: tap-to-set white point with an instant, non-destructive, resettable correction.

## Components affected

- `dekspec/architecture-elements/AE-001-classcam-app.md`

*Blast-radius note (spec-only repo).* ClassCam's runnable code is built in Xcode on a Mac, outside this repository; INT-004's diff here is confined to the spec artifacts this Intent authors (ADR-009, WS-007, IB-010/011/012, index files) — AE-001 is referenced, not edited (its backlinks are relinked centrally later). The Xcode-side blast radius — the `ClassCamCore` `Color/` module and the app target's `Imaging` + `Processing` feature slices (council plan §4) — is realized off-repo and attested manually (see Verification).

## Coverage report

*Populated by `--analyze`. The single gap is resolved inside this Intent (not deferred), so it does not become a blocking `P1` Open Issue.*

| Gap | Source | Resolution | Status |
| --- | --- | --- | --- |
| **Colour-math decision unrecorded.** INT-004's Desired Outcome ("the app re-colours the image so the tapped spot reads neutral") depends on an architectural choice the council flags as the hill its imaging seat will die on (council plan §3.4, risk R2): the correction must be computed in **linear light**, not gamma-encoded sRGB (which leaves a residual cast), and stored **non-destructively** (gains in state, no baked bitmap). ADR-001…006 cover form / frameworks / on-device / SwiftUI / concurrency / the pure package, but not the linear-vs-gamma colour-math + non-destructive-storage decision this slice hinges on. | analyze — ADR gap (D-check "depends on an undocumented decision") | **Resolve in this Intent.** Author **ADR-009** (compute tap-to-white-balance gains in linear light, apply as a diagonal colour matrix in `extendedLinearSRGB` then encode back to sRGB, store the tapped point + gains non-destructively in `ProcessingState.whitePoint`). It governs WS-007 and all three IBs. | closed |

## Size assessment

*Populated by `--analyze`. ADRs are not size-capped; only new **AEs** count against the New-L1 cap.*

| Cap | Limit | Measured | Verdict |
| --- | --- | --- | --- |
| Implementation Units (IBs / direct beads) | ≤ 3 | 3 (IU1 linear-light gain math · IU2 `sampleGains` + white-balance render node · IU3 white-balance control UI) | PASS |
| Components affected | ≤ 3 | 1 (AE-001, referenced) | PASS |
| New L1 artifacts (AEs) | ≤ 1 | 0 (no new AE; ADR-009 is an ADR, uncapped) | PASS |
| New + revised L2 artifacts (WSes + ICs) | ≤ 3 | 1 (WS-007 new; the shared `WhitePoint`/`WhiteBalanceGains` types + `ImagingService.render` signature already exist — IC-001, WS-001, INT-003 — so no new IC) | PASS |
| Coverage gaps | ≤ 2 | 1 (resolved in-Intent) | PASS |

## Layer impact analysis

*Populated by `--analyze`. IU → WS fan-in footnote consumed by `--decompose`.*

| Layer | Artifact | Action |
| --- | --- | --- |
| L1 (Architecture & Decisions) | AE-001 (ClassCam App) — realises the white-point correction stage; referenced, not edited (backlinks relinked centrally later) | reference |
| L1 (Architecture & Decisions) | ADR-009 — compute tap-to-white-balance in linear light and store it non-destructively | new |
| L2 (Specification) | WS-007 — tap-to-white-balance behavior (gain math + 1×1 sampling + non-destructive flow + control UI) | new |
| L3 (Implementation) | IB-010 (ClassCamCore Color: linear-light gain math + gamma-vs-linear divergence test) · IB-011 (`ImagingService.sampleGains` + white-balance render node) · IB-012 (white-balance control UI + non-destructive state) | new |
| L4 (Construction) | 5 beads (see decomposition trail) | new |

**Decomposition trail (`--decompose`, 5 beads — plan authored to the Intent's bead file, emitted at construction):**

| Bead | IB | Unit | Blocks-on |
| --- | --- | --- | --- |
| `b1` | IB-010 | Linear-light per-channel gain math + gamma-vs-linear divergence test (pure `ClassCamCore/Color`) | — (ready) |
| `b2` | IB-011 | `sampleGains` — 1×1 sample through the shared `CIContext` in a tagged colour space | — (ready) |
| `b3` | IB-011 | Integrate the white-balance `CIColorMatrix` node into the render graph (`extendedLinearSRGB` → encode sRGB) | b1, b2 |
| `b4` | IB-012 | White-balance control UI: arm → tap a neutral spot → recolour instantly | — (ready) |
| `b5` | IB-012 | Non-destructive state: store `WhitePoint`, Reset drops it, re-tap overwrites | b4 |

*The INT-004 Outcome Verification (tap neutralises the white point non-destructively) is the acceptance of the terminal beads `b3` (visible correction) + `b5` (Reset/re-tap), attested on-device.*

**IU → WS fan-in (for `--decompose`):**
- IU1 linear-light Color math ← WS-007 (defines the gain formulas + divergence test) — packaged as IB-010.
- IU2 `sampleGains` + white-balance render node ← WS-007 (defines the sampling + node behavior) — packaged as IB-011.
- IU3 white-balance control UI + non-destructive state ← WS-007 (defines the arm→tap→recolour + Reset/re-tap flow) — packaged as IB-012.

*All three IUs draw from the single WS-007 but are packaged as separate Implementation Briefs (rather than taking the single-WS direct-bead shortcut), exactly as INT-001 packaged IB-001/IB-002 from the single WS-001: each IB's file list, sequencing, and Reuse Inventory is the load-bearing Mac-side build guide for a distinct slice (pure package math · imaging actor · SwiftUI screen).*

## Verification

```yaml
verification:
  - name: tap-neutralises-white-point-nondestructively
    cmd: echo MANUAL-VERIFY
    manual: true
    manual_rationale: ClassCam is built and run locally in Xcode; this specification repository has no executable app surface, so the white-point correction is verified by manual on-device attestation.
```

## Outcome Verification

After tapping the White Balance control and then a should-be-neutral spot, the image re-colours so that spot reads neutral; re-tapping a different spot re-applies the correction and Reset restores the original colours. Verified by manual on-device attestation.

## Open Issues

- *None blocking.* The single `--analyze` coverage finding (unrecorded linear-vs-gamma colour-math + non-destructive-storage decision) is resolved in-Intent by ADR-009 — see Coverage report. No `P1` issues remain; the Intent is clean for PROPOSED.

## Amendment Log

| Date | Type | Change | Author |
|------|------|--------|--------|
| 2026-08-09 | Substantive | Initial authoring — Mission slice: tap-to-white-balance. | noreply@anthropic.com |
| 2026-08-10 | Substantive | `--analyze`: populated Coverage report (1 gap — unrecorded linear-vs-gamma colour-math decision, resolved in-Intent via ADR-009), Size assessment (all 5 caps PASS), Layer impact analysis (3 IUs → IB-010/011/012 from the single WS-007). DRAFT → PROPOSED. Design input: council plan (`historical-artifacts/classcam-swift-council-plan.md` §3.4/§1.2/§7). | noreply@anthropic.com |
| 2026-08-10 | Substantive | Promoted PROPOSED to ACCEPTED via /write-intent --accept (engineer directive: drive INT-004 to beads for Mac build). | noreply@anthropic.com |
| 2026-08-10 | Substantive | --analyze complete: 1 coverage gap (colour-math ADR) resolved in-Intent, all size caps PASS | jeffhaskin1@gmail.com |
| 2026-08-10 | Substantive | Engineer directive: drive INT-004 to beads for Mac build | noreply@anthropic.com |
| 2026-08-10 | Substantive | Decomposed: ADR-009, WS-007, IB-010/011/012 (ACCEPTED) + 5 beads (br-d9t,bh6,daz,t36,v94). | noreply@anthropic.com |
