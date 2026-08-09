# Implementation Brief: CaptureService (real AVFoundation capture)

**Spec:** `dekspec/working-specs/WS-002-capture-service.md`
**Intent:** `dekspec/intents/INT-001-capture-presentation-still.md`
**Source AEs:** AE-001
**Depends on:** IB-001, IB-002
**Production gate:** on-device manual attestation (INT-001 Verification — capture on a real iPad)
**Status:** ACCEPTED

## Precedence

Reviewed for conflicts before writing. Resolve residual ambiguity by: (1) Constraints & Decisions, (2) Domain Constraints, (3) Quality Checklists. Do not implement from Spec Context. Stop and ask on any unresolved conflict.

## Goal

The stub `CaptureService` is replaced by a real AVFoundation `actor` that shows a live preview, drives digital zoom, and captures a full-resolution still — deep-copied, orientation-sealed, and colour-space-tagged — handed back as a `CapturedImage` (IC-001), with the whole session run off the main thread on a private serial queue.

## Out of Scope

- Corner detection, perspective correction, colour work, export (INT-002/003/004).
- The `ClassCamCore` model types (IB-001) and the shell/AppModel (IB-002) — this IB swaps the capture actor's body only.
- Pre-rotating pixels — orientation travels as metadata.

## Escalation Protocol

Stop and ask when a decision needs information not in this IB, when a file outside Files to Modify must change, or when a Done When criterion cannot be met without out-of-scope work. Do not guess. In particular: do **not** hand a pooled `CVPixelBuffer` across the actor boundary to "make it work."

## Spec Context

**Traceability only.** WS-002 §What This Does, §Business Rules BR1–BR11, §Domain Constraints, §Failure Behavior. IC-001 §Interface Definition + §Domain Constraints (the producer obligations: deep copy, tag, orientation, full resolution).

## Files to Modify

*Xcode-side paths (built on the Mac). Structure per council plan §4.*

| File | Change |
|------|--------|
| `ClassCam/Capture/CaptureService.swift` | Replace stub with a real `actor` owning `AVCaptureSession` + a private serial `DispatchQueue(label: "classcam.session")`; `startPreview`/`capturePhoto`/`setZoom`/`stop`. |
| `ClassCam/Capture/CameraPreview.swift` | `UIViewRepresentable` whose backing `UIView` returns `AVCaptureVideoPreviewLayer` as its `layerClass`; `videoGravity = .resizeAspect`; built once in `makeUIView`. |
| `ClassCam/Capture/PhotoCaptureDelegate.swift` | `AVCapturePhotoCaptureDelegate` bridged to `async` via `withCheckedThrowingContinuation`; retained by `settings.uniqueID`; deep-copies the buffer before resuming. |
| `ClassCam/Capture/CaptureError.swift` | `enum CaptureError: Error { case noCamera, permissionDenied, captureFailed }`. |
| `ClassCam/Capture/CameraAuthorization.swift` | `authorizationStatus`/`requestAccess` flow + a Settings-deeplink path for denied/restricted. |
| `ClassCamCore/Sources/ClassCamCore/Capture/ZoomClamp.swift` | Pure `clampZoom(_:max:ceiling:)` helper (unit-testable in the package). |
| `ClassCamCore/Tests/ClassCamCoreTests/ZoomClampTests.swift` | Unit tests for the clamp (WS-002 BR7 math). |
| `ClassCam/Capture/CaptureScreen.swift` | Wire the real preview + zoom control + shutter (replacing the IB-002 placeholder body). |

## Reuse Inventory

| Capability | Location | Use instead of reimplementing |
|------------|----------|-------------------------------|
| `AVCaptureSession`, `AVCapturePhotoOutput`, `AVCaptureVideoPreviewLayer`, `AVCaptureDevice` | AVFoundation | Use the framework capture pipeline; do not hand-roll camera access. |
| `withCheckedThrowingContinuation` | Swift concurrency | Bridge the delegate callback to `async`; don't invent a completion-handler scheme. |
| `CapturedImage` | `ClassCamCore` (IB-001) | Construct the existing envelope; don't define a new capture-result type. |
| `clampZoom` | `ClassCamCore/Capture` (this IB) | The zoom clamp is pure math in the package, reused by the actor; don't inline it in the actor. |

## Domain Constraints

| Constraint | Value |
|------------|-------|
| Session threading | private serial queue, never main |
| Still resolution | `maxPhotoDimensions` = active-format max; `.quality` |
| Zoom | clamped `[1.0, min(maxAvailableVideoZoomFactor, ceiling)]`; digital |
| Orientation | `videoRotationAngle`; sealed as metadata |
| Buffer ownership | deep-copied at the shutter (council R1) |
| Colour tag | sRGB unless deliberately wide (council R2) |
| Compute device / dtype | n/a |

## Environment Prerequisites

| Prerequisite | Probe command | Required |
|--------------|---------------|----------|
| A real iPad with a rear camera (the simulator camera proves nothing — council R5) | manual on-device run | yes |

## Do Not Touch

| Function/File | Reason |
|---------------|--------|
| `ClassCamCore` model types | Owned by IB-001. |
| `AppModel` transition logic | Owned by IB-002. |
| Imaging pipeline | Later Intents. |

## Governing ADRs

| ADR | Title |
|-----|-------|
| ADR-005 | Swift 6 strict-concurrency actor-isolated services |
| ADR-002 | Apple-native Vision + Core Image |
| ADR-004 | Build the UI in SwiftUI |
| ADR-003 | Fully on-device |

## Constraints & Decisions

- **Session off main, on a private serial queue:** every `beginConfiguration`/`commit`, `addInput`/`addOutput`, `startRunning`/`stopRunning`, zoom, and capture runs on `classcam.session`. The actor marshals calls onto it — an actor executor alone is not a serial queue (WS-002 BR1 / ADR-005).
- **Guard the device:** `AVCaptureDevice.default(.builtInWideAngleCamera, .video, .back)` may be `nil`; never force-unwrap → throw `CaptureError.noCamera` (BR2).
- **Permissions before preview:** check `authorizationStatus(.video)`; `.notDetermined` → `await requestAccess`; `.denied`/`.restricted` → Settings-deeplink state (BR3). `NSCameraUsageDescription` must be present (BR4, added in IB-002).
- **Full-resolution still:** `maxPhotoDimensions` = the active format's max; `photoQualityPrioritization = .quality` (BR5). The still is the full active format regardless of zoom.
- **Preview:** `UIViewRepresentable` over `AVCaptureVideoPreviewLayer` as `layerClass`, `videoGravity = .resizeAspect`, built once in `makeUIView`; `updateUIView` inert (BR6).
- **Zoom:** `setZoom` sets `videoZoomFactor` (within `lockForConfiguration`) to `clampZoom(...)` (BR7); digital zoom trades resolution for reach.
- **Delegate → async:** bridge `capturePhoto(with:delegate:)` via `withCheckedThrowingContinuation`; retain the delegate keyed by `settings.uniqueID` until `didFinishProcessingPhoto` (BR8).
- **R1 deep copy:** in the completion callback, **deep-copy** the pixel buffer (or take `fileDataRepresentation`) **before** resuming the continuation; never pass a pooled buffer across the actor line (BR9 / IC-001).
- **R2 colour tag:** seal a known colour space (sRGB unless deliberately wide) onto the `CapturedImage` (BR10 / IC-001).
- **Orientation:** `videoRotationAngle` (guard `isVideoRotationAngleSupported`), not the deprecated `videoOrientation`; seal as metadata, don't pre-rotate.
- **Lifecycle:** observe `wasInterrupted`/`interruptionEnded`; stop on disappear/background; restart on return (BR11).

## Interface Contracts

- `dekspec/interface-contracts/IC-001-captured-image.md` — the producer obligations this IB fulfills.

## Quality Checklists

- AVFoundation capture correctness; strict-concurrency cleanliness; on-device attestation checklist.

## Test Promotion Criteria

Promotion refs: WS-002 BR7 (zoom clamp — unit); BR2 (no-camera throw — unit via faked device seam). BR1, BR5, BR6, BR8–BR11 are on-device attestation.

## Test Layout

- `ClassCamCore/Tests/ClassCamCoreTests/ZoomClampTests.swift` (pure zoom-clamp unit tests)
- On-device attestation checklist (no automatable surface for the AVFoundation behaviors — INT-001 Verification).

## Done When

- [ ] Zoom clamp: `clampZoom(f, max:, ceiling:)` clamps below 1.0 up to 1.0 and above the cap down to the cap — verified by unit test (WS-002 BR7).
- [ ] No-camera path: with the device seam returning `nil`, `startPreview`/`capturePhoto` throw `CaptureError.noCamera` — verified by unit test (WS-002 BR2).
- [ ] On device: live preview shows the whole surface (`.resizeAspect`); shutter yields a full-active-format still; sub-second at `.quality` — verified by on-device attestation (WS-002 BR5, BR6).
- [ ] On device: the returned `CapturedImage` survives a held detect→sample→render sequence intact (deep copy holds) and carries a non-nil colour-space tag — verified by on-device attestation (WS-002 BR9–BR10 / IC-001 acceptance check, council R1/R2).
- [ ] On device: a call/Control-Center interruption mid-preview recovers and the session restarts — verified by on-device attestation (WS-002 BR11).
- [ ] The session never runs config/start/stop/capture on the main thread — verified by on-device attestation (main-thread checker clean; WS-002 BR1).
- [ ] All new unit tests pass; no pre-existing tests break — verified by test run.

**Golden State Transitions**

| Input | Expected Output | Verified by |
|-------|----------------|-------------|
| `clampZoom(0.5, max: 8, ceiling: 6)` | `1.0` | unit test |
| `clampZoom(10, max: 8, ceiling: 6)` | `6.0` | unit test |
| `clampZoom(3, max: 8, ceiling: 6)` | `3.0` | unit test |
| device seam → `nil`, then `capturePhoto()` | throws `CaptureError.noCamera` | unit test |

## Open Issues

- [ ] The quality-preserving zoom `ceiling` value is fixed during on-device tuning (WS-002 Open Issue). — **Source:** initial draft — **Severity:** `P3`

## Amendment Log

| Date | Type | Change | Author |
|------|------|--------|--------|
| 2026-08-10 | Substantive | Initial authoring — IB-003 CaptureService real AVFoundation capture (INT-001 IU3, from WS-002 + IC-001). | noreply@anthropic.com |
| 2026-08-10 | Substantive | Accepted as INT-001 IB | noreply@anthropic.com |
