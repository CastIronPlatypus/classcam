# ADR-002: Use Apple-native Vision and Core Image for detection and correction

## Status

ACCEPTED

## Supersession

*Supersedes:* none
*Superseded by:* none

## Related Architecture Elements

- AE-001: ClassCam App — shapes the corner-detection and perspective-correction stages of the app's pipeline.

## Created

2026-08-09

## Modified

2026-08-10

## Date

2026-08-09

## Deciders

Jeff Haskin

## Context and Decision Drivers

Having chosen a native iPadOS app (ADR-001), ClassCam needs a way to detect the presentation surface's corners and to perspective-correct and colour-adjust the captured still. The platform provides first-party frameworks for exactly these tasks. The main alternative is bundling a third-party computer-vision library that had been considered during the earlier browser exploration.

**Decision drivers:**
- The app needs rectangle/quad detection and a perspective warp plus colour adjustment.
- The platform offers first-party frameworks that cover these without a third-party runtime dependency.
- Keeping to on-device, dependency-free processing is preferred (ADR-003).

*Technical story:* none

## Decision

ClassCam uses Apple's Vision framework for presentation-surface corner detection and Core Image for perspective correction and the white-point colour adjustment. A bundled third-party computer-vision library is rejected because the first-party frameworks cover the app's detection and correction needs on-device without adding a runtime dependency.

## Options Considered (if applicable)

### Option A: Apple-native (Vision + Core Image)

Use platform frameworks for detection and correction.

**Pros:** first-party, on-device, no third-party runtime dependency; integrates with the native capture and image types.
**Cons:** ties the pipeline to platform frameworks.

### Option B: Third-party computer-vision library

Bundle a general CV library for detection and warping.

**Pros:** portable, familiar algorithms.
**Cons:** added runtime dependency and binary weight; memory-management hazards; redundant with capable first-party frameworks.

## Consequences

**Positive:**
- Detection and correction run on-device with no third-party runtime dependency.
- The pipeline uses native image types end to end.

**Negative:**
- The detection and correction stages depend on platform frameworks.

## Validation

**Observable confirmation:**
Corner detection and perspective/colour correction run on the device using the platform frameworks and produce the corrected slide image.

**Reconsideration triggers:**
The first-party frameworks prove insufficient for the app's detection or correction quality, requiring a different approach.

## Links

- none

## Open Issues

- *None currently.*

## Amendment Log

| Date | Type | Change | Author |
|------|------|--------|--------|
| 2026-08-09 | Substantive | Initial authoring — records the Apple-native Vision + Core Image decision. | noreply@anthropic.com |
| 2026-08-10 | Substantive | Status PROPOSED to ACCEPTED — auto-transition by `dekspec audit linkage --fix` (T-STATUS status-maturity coherence, ADR-020). | dekspec-audit-fix |
