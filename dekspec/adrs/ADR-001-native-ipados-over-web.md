# ADR-001: Build ClassCam as a native iPadOS app rather than a web app

## Status

PROPOSED

## Supersession

*Supersedes:* none
*Superseded by:* none

## Related Architecture Elements

- AE-001: ClassCam App — establishes the app's form as a single native Container rather than a browser-delivered surface.

## Created

2026-08-09

## Modified

2026-08-09

## Date

2026-08-09

## Deciders

Jeff Haskin

## Context and Decision Drivers

ClassCam began as a browser-based concept. Investigation of mobile Safari (WebKit) on iPad surfaced structural ceilings for the app's core loop: the browser cannot provide a live camera preview together with a full-resolution still from one pipeline, camera capture is capped well below the sensor, and camera permissions, clipboard image writes, and canvas memory all carry fragile, WebKit-specific constraints. These limits sit directly on the app's essential path — aim, capture at full quality, correct, and copy out.

**Decision drivers:**
- The core loop needs both a live aiming preview and a full-resolution still, which the browser pipeline cannot deliver together.
- The browser's camera, clipboard, and canvas constraints make the essential flow fragile.
- A native platform exposes the capture, vision, and image-processing capabilities the app depends on.

*Technical story:* none

## Decision

ClassCam is built as a native iPadOS application. The browser/PWA delivery path is rejected because its camera-capture, clipboard, and canvas ceilings prevent the app's core capture-and-correct loop from working reliably, whereas a native app removes those ceilings and exposes the platform capture and image frameworks the app needs.

## Options Considered (if applicable)

### Option A: Web app (browser / PWA)

Deliver ClassCam as a web page run in mobile Safari.

**Pros:** no install; single codebase reachable from any device.
**Cons:** no live preview plus full-resolution still from one pipeline; capture resolution capped below the sensor; fragile camera-permission, clipboard-image, and canvas-memory behaviour on iPad WebKit.

### Option B: Native iPadOS app

Deliver ClassCam as a native app.

**Pros:** live preview and full-resolution capture together; direct access to platform capture, vision, and image frameworks; reliable clipboard and share.
**Cons:** platform-specific; requires the Apple toolchain and distribution.

## Consequences

**Positive:**
- The core capture-and-correct loop is unblocked by the browser ceilings.
- The app can use native capture, vision, and image-processing capabilities.

**Negative:**
- The app is iPadOS-specific and must be built and distributed through the Apple toolchain.

## Validation

**Observable confirmation:**
The shipped app captures a full-resolution still while showing a live preview, and correction plus clipboard export work reliably on device.

**Reconsideration triggers:**
The browser platform later removes the capture/clipboard/canvas ceilings, or a cross-platform reach requirement outweighs the native capabilities.

## Links

- none

## Open Issues

- *None currently.*

## Amendment Log

| Date | Type | Change | Author |
|------|------|--------|--------|
| 2026-08-09 | Substantive | Initial authoring — records the native-iPadOS-over-web decision. | noreply@anthropic.com |
