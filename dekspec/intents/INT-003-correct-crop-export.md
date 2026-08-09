# INT-003: Perspective-correct, crop to aspect ratio, and export

## Status

DRAFT

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

2026-08-09

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

- *None currently.*

## Amendment Log

| Date | Type | Change | Author |
|------|------|--------|--------|
| 2026-08-09 | Substantive | Initial authoring — Mission slice: perspective correction, aspect crop, and export. | noreply@anthropic.com |
