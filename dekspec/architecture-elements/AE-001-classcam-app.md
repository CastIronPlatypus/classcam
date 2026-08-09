# AE-001: ClassCam App

## Status

ACCEPTED

## Subtype

Container

## Classification

Core

## Created

2026-08-09

## Modified

2026-08-10

## Linked Artifacts

- **Related ADRs:** ADR-001, ADR-002, ADR-003, ADR-004, ADR-005, ADR-006, ADR-007, ADR-008, ADR-009
- **Related WSs:** WS-001, WS-002, WS-003, WS-004, WS-005, WS-006, WS-007
- **Related ICs:** IC-001
- **Related IBs:** IB-001, IB-002, IB-003, IB-004, IB-005, IB-006, IB-007, IB-008, IB-009, IB-010, IB-011, IB-012
- **Related Intents:** INT-001, INT-002, INT-003, INT-004
- **Owners:** Jeff Haskin

## Implements

- none

*ClassCam is described at the architectural level here; the implementing Swift sources live in the local Xcode project, outside this specification repository.*

## Purpose and Scope

ClassCam is the single deployable iPadOS application that realises the System Vision: it lets someone photograph a presentation shown on a TV, monitor, or whiteboard and obtain a clean, straightened, colour-corrected slide image for a note-taking app. It is one runnable unit — a native app with no server side — that carries the whole flow from camera to clipboard. As the system's only Container, it owns the capture surface, the on-device image-correction pipeline, and the export surface, and it is the slice every downstream Working Spec, Interface Contract, and Implementation Brief targets.

## Responsibilities

- Present a live camera preview with a digital-zoom framing control and capture a full-resolution still on the user's shutter action.
- Detect the presentation surface's four corners on the captured still and present them as draggable handles, with a default draggable box when detection finds no clear quad.
- Perspective-correct and crop the still to the user-selected aspect ratio (16:9 or 4:3).
- Apply a user-directed white-point correction sampled from a tapped neutral spot, re-applied non-destructively.
- Export the finished slide by placing it on the clipboard automatically and offering it to the system share sheet, so the user can paste or share it into a note-taking app.

## Boundaries and Non-Goals

**Inside the boundary:**
- The capture, corner-adjustment, perspective-correction, white-balance, and export stages of a single presentation image.
- All processing, performed locally on the device.

**Outside the boundary (non-goals):**
- Server-side or networked processing — excluded because ClassCam runs entirely on-device (ADR-003).
- Text extraction / OCR — the app produces an image, not text, so recognising or extracting slide text is out of scope.
- General photo editing — filters, drawing, and freeform adjustment are excluded to keep the app to one straighten-and-colour-fix purpose.

## Three-tier Boundaries

**Always do:**
- Process each image locally on the device.
- Give the user direct manual control over the corners and the white point.

**Ask first:**
- none

**Never do:**
- Send captured images or derived data off the device.

## Relationships and Dependencies

**Consumes:** the device rear-camera stream and full-resolution stills, plus the user's touch input for framing, corner placement, aspect choice, and white-point selection.

**Produces:** a rectilinear, colour-corrected slide image in the selected aspect ratio, delivered to the system clipboard and share sheet.

**Depends on:** the platform capture, vision, and image-processing frameworks selected in ADR-002; the native-application form selected in ADR-001; the on-device constraint of ADR-003; and the UI layer selected in ADR-004.

**Consumed by:** external note-taking apps (e.g. GoodNotes, Notability) that receive the exported image via clipboard paste or the share sheet — outside this system's boundary.

## Views

### component view — internal stages of the ClassCam app

```mermaid
flowchart LR
    Cam[Rear camera] --> Capture[Capture stage]
    Capture --> Detect[Corner detection]
    Detect --> Adjust[Corner adjustment]
    Adjust --> Correct[Perspective correction]
    Correct --> Balance[White-point correction]
    Balance --> Export[Export stage]
    Export --> Clip[Clipboard / share sheet]
```

## Runtime Behavior

A typical run: the user opens the app to the capture screen, frames the presentation with the zoom control, and taps the shutter to take a full-resolution still. The app moves to the processing screen, detects the surface's corners, and lets the user drag them and tap a neutral spot to correct colour. On completion the app perspective-corrects and crops to the chosen aspect ratio, applies the white-point correction, exports to the clipboard, and shows the result, from which the user may share, re-edit, or start over.

## Internal Structure

*Per ADR-005 and ADR-006. The implementing Swift sources live in the local Xcode project; this section describes the architectural shape they realise.*

The app is one `@MainActor`, `@Observable` application-state owner holding a single `Stage` state machine (`capture` → `processing` → `result`), plus two isolated service actors reached through narrow `async` interfaces:

- a **capture service** (owns the AVFoundation session on a private serial queue) that realises the *Capture stage* — live preview, digital-zoom framing, and full-resolution still capture (INT-001);
- an **imaging service** (owns the single Core Image context) that realises the *Corner detection*, *Perspective correction*, *White-point correction*, and *Export* stages (INT-002 / INT-003 / INT-004).

Only `Sendable` value types cross an actor boundary; the correctness-critical pure logic and those value types live in the UI-free `ClassCamCore` package (ADR-006). The first such boundary value — the captured still handed from the Capture stage to the imaging pipeline — is defined by IC-001 (`CapturedImage`).

## Constraints and Quality Notes

- Runs on iPadOS as a native application.
- Performs all image processing on-device, with no network dependency.

## Open Questions / Planned Follow-ons

- *None currently contemplated.*

## Amendment Log

| Date | Type | Change | Author |
|------|------|--------|--------|
| 2026-08-09 | Substantive | Initial authoring — ClassCam App Container AE describing the single on-device iPadOS app slice. | noreply@anthropic.com |
| 2026-08-09 | Substantive | INT-001 decomposition: added Internal Structure section (one @Observable AppModel + two service actors; ClassCamCore pure package) and linked ADR-005/006, WS-001/002, IC-001, IB-001/002/003. | noreply@anthropic.com |
| 2026-08-10 | Substantive | Status PROPOSED to ACCEPTED — auto-transition by `dekspec audit linkage --fix` (T-STATUS status-maturity coherence, ADR-020). | dekspec-audit-fix |
| 2026-08-10 | Substantive | Full MSN-001 decomposition: linked ADR-007/008/009, WS-003…007, IB-004…012 (INT-002/003/004 orchestrated to beads). | noreply@anthropic.com |
