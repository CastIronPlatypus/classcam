# ADR-003: Process everything on-device with no backend, network, or accounts

## Status

ACCEPTED

## Supersession

*Supersedes:* none
*Superseded by:* none

## Related Architecture Elements

- AE-001: ClassCam App — fixes the app as a self-contained on-device Container with no server side.

## Created

2026-08-09

## Modified

2026-08-10

## Date

2026-08-09

## Deciders

Jeff Haskin

## Context and Decision Drivers

ClassCam captures and corrects a single image and hands it to a note-taking app. Nothing in the flow inherently requires a server: capture, detection, correction, and export can all run locally on the iPad. Introducing a backend, network calls, or accounts would add moving parts and send the user's captured images off the device.

**Decision drivers:**
- The whole capture-and-correct flow can run locally on the device.
- Keeping images on the device is preferred for speed and privacy.
- A backend or account system would add operational surface with no benefit to the core flow.

*Technical story:* none

## Decision

ClassCam performs all processing on-device and has no backend, network dependency, or account system. A server-assisted or account-based design is rejected because the capture-and-correct flow runs entirely locally, and adding remote processing would send the user's images off the device and add operational surface for no functional gain.

## Consequences

**Positive:**
- Captured images never leave the device.
- The app has no server to operate and works without connectivity.

**Negative:**
- Capabilities that would require server-side processing are unavailable by construction.

## Validation

**Observable confirmation:**
The app completes the full capture-to-export flow with no network access and stores no data off the device.

**Reconsideration triggers:**
A future capability genuinely requires processing that cannot run on-device.

## Links

- none

## Open Issues

- *None currently.*

## Amendment Log

| Date | Type | Change | Author |
|------|------|--------|--------|
| 2026-08-09 | Substantive | Initial authoring — records the fully-on-device decision. | noreply@anthropic.com |
| 2026-08-10 | Substantive | Status PROPOSED to ACCEPTED — auto-transition by `dekspec audit linkage --fix` (T-STATUS status-maturity coherence, ADR-020). | dekspec-audit-fix |
