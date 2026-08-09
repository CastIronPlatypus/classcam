# Working Spec: CaptureService — live preview, digital zoom, full-resolution still

## Status

ACCEPTED

## Created

2026-08-09

## Modified

2026-08-10

## Silent Failure Domain(s)

*None of the five Dektora domains applies. ClassCam's real silent-failure risks for **this** spec are (a) a **pixel buffer recycled** out from under a later imaging stage — torn/blank pixels, no error (council R1) — and (b) an **untagged colour space** breaking downstream white balance (council R2). Both are pinned in IC-001 and re-encoded here as Business Rules BR9–BR10 and Failure Behavior.*

- [ ] Transformer internals (position IDs, injection layer, KV cache)
- [ ] Numerical precision (quantization, tiered compression, serialization round-trips)
- [ ] GPU multi-process isolation (device assignment, process crash recovery)
- [ ] Graph consistency (shadow graph / Neo4j flush, phantom nodes)
- [ ] Timeline coherence (topic segmentation, tier assignment, decay, shadow timeline / PostgreSQL)

## Expertise Audit Record

*Native Swift/AVFoundation capture; no Dektora role triggers. AE-001 is **Core**, so the audit is recorded for completeness. The relevant expertise (AVFoundation capture) is supplied by the ClassCam persona council, not the Dektora role set.*

| Role | Triggered | Trigger rule | Rationale |
|------|-----------|-------------|-----------|
| ML / Model Behavior Expert | No | injection / position IDs / KV cache | No model. |
| Quantization / Precision Expert | No | tensor dtype / precision threshold | Pixels are camera stills, not tensors; no quantization. |
| CUDA Multi-Process Expert | No | multiple devices / process boundaries | Single process; the capture session runs on a serial `DispatchQueue`, an intra-process boundary handled by ADR-005, not CUDA. |
| Graph / Multi-Store Expert | No | shadow graph / Neo4j / timeline stores | No datastore. |
| Embedding Space Geometer | No | similarity / distance | No embeddings. |
| Pipeline Sequencing Analyst | No | pipeline-stage reordering | Capture is the source stage; nothing reordered. |

## Related Architecture Elements

- AE-001: ClassCam App — this spec measures the Capture stage: live preview, digital-zoom framing, and full-resolution still capture, and the value it hands downstream.

## Governing ADRs

- ADR-005: Swift 6 strict-concurrency actor-isolated services — `CaptureService` is an `actor` owning the `AVCaptureSession` on a **private serial queue**; only the `Sendable` `CapturedImage` crosses its boundary.
- ADR-002: Apple-native Vision + Core Image — capture uses AVFoundation (`AVCaptureSession`, `AVCapturePhotoOutput`, `AVCaptureVideoPreviewLayer`).
- ADR-004: SwiftUI UI layer — the preview is a `UIViewRepresentable` bridge (SwiftUI has no native camera-preview view).
- ADR-003: Fully on-device — capture and the still never leave the device.

## Interface Contracts

**Consumed contracts:** none.
**Defined contracts:** IC-001 (`CapturedImage`) — `CaptureService` is the **producer** side; every guarantee in IC-001 (deep-copied buffer, tagged colour space, orientation-as-metadata, full native resolution) is this spec's obligation.

## What This Does

`CaptureService` owns everything between the rear camera sensor and a captured still. It configures and runs an `AVCaptureSession` **off the main thread** on a private serial queue, exposes a live preview surface for SwiftUI to embed, drives digital zoom, and captures a full-resolution still that it deep-copies, tags, and hands back as a `CapturedImage` (IC-001). It also owns the camera-permission flow and the session lifecycle (interruptions, background, thermal). It performs **no** detection, correction, or colour work — it owns capture quality, latency, and lifecycle only, and hands the pixels plus orientation to the imaging stage.

**Mechanism:** `CaptureService.capturePhoto()` triggers `AVCapturePhotoOutput` on the session queue, and in the capture-completion callback **deep-copies** the pixel buffer and seals the orientation and colour-space tag **before** resuming the `async` continuation, so the returned `CapturedImage` owns memory the capture pool cannot reclaim.

## What This Does NOT Do

- **R1 (buffer lifetime):** does not hand a live/pooled `CVPixelBuffer` across the actor boundary — it deep-copies first; nothing downstream ever holds a pool-owned buffer.
- **R2 (colour):** does not perform white balance or any colour correction — it only *tags* the still's colour space so INT-004 can sample correctly.
- **Capture scope:** does not detect corners, warp, crop, or export (INT-002/003/004); does not pre-rotate pixels (orientation travels as metadata); does not upscale/downscale the still (delivers full native resolution).

## Interfaces

### Data Interfaces

| Interface | Direction | Type / Shape | Source or Consumer | Guarantees |
|-----------|-----------|--------------|--------------------|-----------|
| `capturePhoto() async throws` | out | `-> CapturedImage` | AppModel → ImagingService | Returns a fully-valid `CapturedImage` per IC-001, or throws a typed `CaptureError`. |
| `startPreview() async throws` | in | starts session + preview feed | Capture screen `.task {}` | Throws on no-camera / denied permission; otherwise preview is live. |
| `setZoom(_:)` | in | `CGFloat` zoom factor | zoom control (UI) | Clamped to `[1.0, min(maxAvailableVideoZoomFactor, ceiling)]` (BR7). |
| `stop()` | in | stops the session | Capture screen `.onDisappear` / background | Idempotent; releases the camera. |
| preview surface | out | `UIViewRepresentable` over `AVCaptureVideoPreviewLayer` | Capture screen | `videoGravity = .resizeAspect` (BR6); built once, not rebuilt per render. |

### Process Interfaces

| Boundary | Transport | Device | Serialization | Failure mode |
|----------|-----------|--------|---------------|-------------|
| Actor ↔ capture session | private serial `DispatchQueue` inside the actor | on-device camera | in-memory Swift values | A session call on the wrong queue is a threading defect; the actor marshals every call onto the serial queue (BR1). |
| Capture callback → `async` caller | `withCheckedThrowingContinuation` | — | `CapturedImage` value | Delegate deallocating before callback → continuation never resumes (prevented by BR8). |

### Dependencies

| Dependency | Interface | Failure behavior |
|------------|-----------|-----------------|
| `AVCaptureDevice` (rear wide-angle) | `default(.builtInWideAngleCamera, .video, .back)` | May be `nil` (no camera / simulator); guarded, never force-unwrapped → throws `CaptureError.noCamera` (BR2). |
| Camera authorization | `AVCaptureDevice.authorizationStatus` / `requestAccess` | `.denied`/`.restricted` → surfaces a Settings-deeplink state, not a dead preview (BR3). |
| `Info.plist` `NSCameraUsageDescription` | required key | Missing key = OS-level crash on first camera access → build/config check (BR4). |

## Domain Constraints

| Constraint | Value | Scope | Rationale |
|------------|-------|-------|-----------|
| Session threading | All config/`startRunning`/`stopRunning`/zoom/capture on the actor's private serial queue; never main | all-IBs | `startRunning` blocks for hundreds of ms; on main it hangs the launch. An actor executor is a cooperative-pool thread, not a serial queue — the private queue is required (ADR-005). |
| Still resolution | `maxPhotoDimensions` = active format max; `photoQualityPrioritization = .quality` | all-IBs | Detail *is* the product for a slide; accept the added shutter latency. |
| Zoom | `videoZoomFactor` clamped to `[1.0, min(maxAvailableVideoZoomFactor, ceiling)]`; digital only | all-IBs | Digital zoom crops the sensor (trades resolution for reach); the still is always the full active format regardless of zoom. |
| Orientation | `AVCaptureConnection.videoRotationAngle` (guard `isVideoRotationAngleSupported`); **not** the deprecated `videoOrientation` | all-IBs | Modern API; orientation sealed as `CapturedImage` metadata, pixels not pre-rotated. |
| Pixel-buffer ownership | Deep-copied at the shutter before the continuation resumes | all-IBs | Council R1 — a pooled buffer held across imaging calls can be recycled (IC-001). |
| Colour-space tag | Tagged (sRGB unless deliberately wide) on the `CapturedImage` | all-IBs | Council R2 — enables linear-light sampling in INT-004 (IC-001). |
| Preview gravity | `.resizeAspect` | all-IBs | The user must see the whole presentation surface to frame near its edges; `.resizeAspectFill` would crop corners out of view. |
| Compute device / dtype | n/a — camera stills, not tensors; no device pinning | all-IBs | The Dektora device/dtype rows do not apply. |

## Governing Formulas

| Formula | Expression | Variables | Units / Scale | Valid range | Validated by |
|---------|-----------|-----------|---------------|-------------|-------------|
| Zoom clamp | `clamp(f, 1.0, min(maxAvailableVideoZoomFactor, ceiling))` | `f` requested factor; `ceiling` a quality-preserving cap | zoom factor (×) | `[1.0, deviceMax]` | this component (BR7 asserts the clamp) |

## Business Rules

1. **general** Every `AVCaptureSession` configuration/`startRunning`/`stopRunning`/zoom/capture call executes on the actor's private serial queue, never the main thread — verified by on-device attestation (main-thread checker clean during a capture run) and by code review of call sites.
2. **general** When no rear camera device resolves, `startPreview()`/`capturePhoto()` throw `CaptureError.noCamera` rather than force-unwrapping — unit-testable via a device-provider seam faked to return `nil`.
3. **general** When camera authorization is `.denied`/`.restricted`, the capture screen shows a Settings-deeplink path (not a frozen preview); `.notDetermined` triggers `requestAccess` before first use — verified by on-device attestation across all authorization branches.
4. **general** `NSCameraUsageDescription` is present in the app `Info.plist` — verified by a config/build check (its absence is an immediate OS crash on camera access).
5. **general** The still is captured at the active format's maximum photo dimensions with `photoQualityPrioritization = .quality` — verified on-device by asserting the produced still's pixel dimensions equal the active format max.
6. **general** The preview layer uses `videoGravity = .resizeAspect` and is built once in `makeUIView` (not rebuilt in `updateUIView`) — verified by code review + on-device attestation (whole surface visible, no per-render layer rebuild).
7. **general** `setZoom(f)` sets `videoZoomFactor` to `clamp(f, 1.0, min(maxAvailableVideoZoomFactor, ceiling))`; the returned still is the full active format regardless of `f` — the clamp is unit-testable as pure math (ClassCamCore helper); the full-resolution guarantee is on-device attested.
8. **general** The `AVCapturePhotoCaptureDelegate` is retained (keyed by `settings.uniqueID`) until `didFinishProcessingPhoto` fires, and the delegate callback is bridged to `async` via `withCheckedThrowingContinuation` — verified by on-device attestation (capture never hangs; continuation always resumes).
9. **general (R1)** In the capture-completion callback the pixel buffer is **deep-copied** (or the encoded photo data taken) **before** the continuation resumes, so the returned `CapturedImage` owns non-pooled memory — verified on-device by holding the `CapturedImage` across a simulated detect→sample→render sequence and asserting the pixels remain intact (IC-001 acceptance check).
10. **general (R2)** The returned `CapturedImage` carries a known, tagged colour space (sRGB unless deliberately wide-gamut) — verified by asserting the tag is non-nil and equals the configured space (IC-001).
11. **general** Session interruptions (`wasInterrupted`/`interruptionEnded`) are observed and the session restarts cleanly on return; the session stops on view-disappear/background — verified by on-device attestation (call/Control-Center interruption mid-preview recovers).

## Failure Behavior

*Observable signal is a Swift `throws` of a typed `CaptureError`; on-device behaviors are attested manually per the Intent's Verification (no automatable surface in this repo).*

| Failure | Detection | Assertion type | Behavior | Recovery |
|---------|-----------|---------------|----------|----------|
| No rear camera device | `AVCaptureDevice.default(...)` returns `nil` | raise (`CaptureError.noCamera`) | Throw before starting; no force-unwrap | UI surfaces a "camera unavailable" state |
| Camera permission denied/restricted | `authorizationStatus` branch | assert (state) | Show Settings-deeplink state, not a dead preview | User grants in Settings, returns |
| `NSCameraUsageDescription` missing | config/build check | assert (build) | Caught before ship (OS would crash otherwise) | Add the plist key |
| Still capture errored / no buffer | photo callback error / nil buffer | raise (`CaptureError.captureFailed`) | Throw from `capturePhoto()`; construct no `CapturedImage` | AppModel stays in `capture`; user retries |
| Pixel-buffer deep-copy allocation failure | copy returns failure | raise (`CaptureError.captureFailed`) | Throw; never substitute a pooled buffer | User retries |
| Session interrupted mid-capture | interruption notification | assert (state) | Capture completes or fails cleanly; session restarts | Preview resumes on `interruptionEnded` |

## Open Issues

- [ ] The quality-preserving zoom `ceiling` (upper clamp beyond which digital zoom degrades the slide too far) is a tunable to be fixed during on-device tuning; until then the clamp uses `maxAvailableVideoZoomFactor`. — **Source:** initial draft — **Severity:** `P3`

## Amendment Log

| Date | Type | Change | Author |
|------|------|--------|--------|
| 2026-08-09 | Substantive | Initial authoring — CaptureService capture behavior contract; producer side of IC-001; encodes R1/R2 (INT-001 IU3). | noreply@anthropic.com |
| 2026-08-10 | Substantive | Accepted as INT-001 child spec | noreply@anthropic.com |
