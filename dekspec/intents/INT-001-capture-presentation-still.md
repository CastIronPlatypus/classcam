# INT-001: Capture a presentation still with live preview and digital zoom

## Status

DRAFT

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

2026-08-09

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

- *None currently.*

## Amendment Log

| Date | Type | Change | Author |
|------|------|--------|--------|
| 2026-08-09 | Substantive | Initial authoring — first Mission slice: the capture screen. | noreply@anthropic.com |
