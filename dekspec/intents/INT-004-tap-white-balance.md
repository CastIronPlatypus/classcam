# INT-004: Correct the display colour cast by tapping a neutral spot

## Status

DRAFT

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

2026-08-09

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

- *None currently.*

## Amendment Log

| Date | Type | Change | Author |
|------|------|--------|--------|
| 2026-08-09 | Substantive | Initial authoring — Mission slice: tap-to-white-balance. | noreply@anthropic.com |
