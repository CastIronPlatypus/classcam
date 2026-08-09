# INT-002: Detect and adjust the presentation's four corners

## Status

DRAFT

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

2026-08-09

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

- *None currently.*

## Amendment Log

| Date | Type | Change | Author |
|------|------|--------|--------|
| 2026-08-09 | Substantive | Initial authoring — Mission slice: corner detection and adjustment. | noreply@anthropic.com |
