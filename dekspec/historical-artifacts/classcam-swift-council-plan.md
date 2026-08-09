# ClassCam — Swift Implementation Plan (Persona Council synthesis)

**Status:** Advisory plan (not a LOCKED DekSpec artifact)
**Date:** 2026-08-09
**Produced by:** `/persona-council` — a 3-seat council debated to consensus over agent-mail.
**Council:** The Swift / SwiftUI App Architect · The AVFoundation Capture Engineer · The Core Image / Vision Imaging Engineer
**Assignment:** A plan for building ClassCam in Swift, deeply optimized for Swift best practices, architecture, organization, concurrency, and the correct Apple-framework approach for each feature — surfacing the pitfalls of each subsystem.
**Consensus:** Unanimous. No dissent recorded. One risk flagged and folded into the sequence (pixel-buffer lifetime).

> This is the *thinking* deliverable. It maps cleanly onto the existing spec graph
> (MSN-001, AE-001, INT-001…004) but does not itself modify any LOCKED artifact.
> Hand it to `/shotgun` to turn it into intents/beads/an implementation team, or
> use it directly as the build blueprint in Xcode.

---

## 0. The one-paragraph shape

ClassCam is three screens and one linear flow, so its architecture is deliberately
small: **one `@MainActor @Observable AppModel`** owns a single **`Stage` state-machine
enum**, and **two `actor`-backed services** (`CaptureService`, `ImagingService`) hide
all of AVFoundation and Core Image/Vision behind narrow `async` interfaces. Only a
handful of **`Sendable` value types** cross the actor boundaries — never a
`CIImage`, `CIContext`, `AVCaptureSession`, or `UIImage`. All the correctness-critical
math (coordinate conversions, quad ordering, white-balance gains, output sizing) lives
in a **pure `ClassCamCore` Swift package** that builds and unit-tests with no simulator.
The whole thing is **Swift 6 strict-concurrency-clean from the first commit**. This is
the smallest structure that expresses the app cleanly — and the plan actively resists
anything heavier.

---

## 1. Architecture

### 1.1 The state machine is the architecture

Model the app as one value type; illegal states become unrepresentable:

```
enum Stage {
    case capture
    case processing(ProcessingState)
    case result(Slide)
}

struct ProcessingState {          // all Sendable value types
    let image: CapturedImage      // thin envelope, immutable
    var quad: Quad                // four normalized CGPoints, validated
    var whitePoint: WhitePoint?   // nil until the user taps
    var aspect: AspectRatio       // .sixteenNine / .fourThree
}
```

You cannot be `processing` without a captured image, cannot hold a white point before
you have pixels, cannot reach `result` without a `Quad`. This kills the boolean soup
(`isProcessing` / `hasImage` / `didBalance`) before it is ever written.

### 1.2 The locked surface (ratified by all three seats)

**Sendable value types** (in `ClassCamCore` unless noted):

| Type | Shape |
|---|---|
| `CapturedImage` | **Thin envelope**: deep-copied `CVPixelBuffer` + sealed `CGImagePropertyOrientation` + pixel dimensions + a `CGColorSpace`/pixel-format tag. Never an eager 12 MP `CGImage`. |
| `Quad` | Four **normalized** `CGPoint`s (0–1), ordered + validated. |
| `AspectRatio` | `enum { case sixteenNine, fourThree }` |
| `WhiteBalanceGains` | Per-channel `(r, g, b)` multipliers. |
| `WhitePoint` | `{ tappedPoint: CGPoint; gains: WhiteBalanceGains }` |
| `ProcessingState` | see above |
| `Slide` | Finished image (`CGImage` / PNG `Data`) + `AspectRatio`. |
| `Stage` | the enum above |

**Service actors:**

```
actor CaptureService {
    func startPreview() async throws
    func capturePhoto() async throws -> CapturedImage
    func setZoom(_ factor: CGFloat)
    func stop()
}   // + a UIViewRepresentable preview surface over AVCaptureVideoPreviewLayer

actor ImagingService {
    func detectQuad(in image: CapturedImage) async -> Quad?
    func sampleGains(at point: CGPoint, in image: CapturedImage) async -> WhiteBalanceGains
    func render(_ image: CapturedImage, quad: Quad,
                whitePoint: WhitePoint?, aspect: AspectRatio) async throws -> Slide
}
```

**One `@MainActor @Observable AppModel`** owns `stage` plus both actors.

### 1.3 What the council explicitly talked the team OUT of

At three screens, these are pure liability — **do not add them**:

- MVVM-per-screen / a `ViewModel` class per view — SwiftUI views *are* the view layer.
- A Coordinator / router.
- A DI container.
- Combine / `ObservableObject` / `@Published` — the Observation framework (`@Observable`)
  plus `async`/`await` covers every stream this app has.

---

## 2. Concurrency model (Swift 6 strict, from commit one)

Turn on **Swift 6 language mode + complete strict concurrency + warnings-as-errors**
in the very first commit. It is far cheaper to stay clean than to retrofit. Three
isolation domains:

- **`@MainActor`** — `AppModel`, all SwiftUI views, all gesture/drag state. Corner
  dragging is 60 fps UI math on *normalized points*; no pixels are touched on main.
- **`actor CaptureService`** — owns the `AVCaptureSession`. **Key pattern:** the actor
  owns a *private serial* `DispatchQueue(label: "classcam.session")` and marshals every
  `AVCaptureSession` call onto it via `withCheckedThrowingContinuation`. The actor gives
  Sendable-safe `await` points; the serial queue gives AVFoundation the thread affinity
  it demands. (An actor's executor alone is a cooperative-pool thread — *not* a serial
  queue — so the private queue is non-negotiable.) Preview frames bypass the actor
  mailbox on their own layer.
- **`actor ImagingService`** — owns the **one** `CIContext` (created once, app-lifetime)
  and runs all Vision + Core Image work off main.

**Rule:** only the locked value types cross an actor line. If strict concurrency
complains, it is telling you exactly where data crosses a thread — fix the boundary;
**never** stamp `@MainActor` everywhere or reach for `@unchecked Sendable` to silence it
(that would drag image work onto the UI thread and jank the app).

**Structured concurrency:** kick service calls from a view's `.task {}` so they cancel
automatically on disappear; no detached `Task {}` that outlives its view. Check
`Task.isCancelled` around `detectQuad`/`render`.

---

## 3. The four features, done the Swift way

### 3.1 INT-001 — Capture (AVFoundation)

- **Session on a serial queue, off main.** `startRunning` is synchronous and burns
  hundreds of ms — on main it hangs the launch. All `beginConfiguration`/`commit`,
  `addInput`/`addOutput`, `start`/`stop`, zoom, and capture happen on the session queue.
- **Device:** `AVCaptureDevice.default(.builtInWideAngleCamera, for: .video, position: .back)`,
  **guarded** (never force-unwrapped). Preset `.photo`; still resolution is driven by
  `maxPhotoDimensions`, not the preset.
- **Preview:** a `UIViewRepresentable` whose backing `UIView` returns
  `AVCaptureVideoPreviewLayer` as its `layerClass` (the layer *is* the backing layer, so
  it resizes for free). **`videoGravity = .resizeAspect`** — the user must see the *whole*
  presentation surface to frame it near the screen edges; `.resizeAspectFill` would crop
  corners out of view and surprise them at detection. Build once in `makeUIView`;
  `updateUIView` stays inert (no per-render rebuild).
- **Full-resolution still:** `AVCapturePhotoOutput`; set `maxPhotoDimensions` to the
  active format's max `supportedMaxPhotoDimensions` and mirror it on each
  `AVCapturePhotoSettings`; `photoQualityPrioritization = .quality`. Detail *is* the
  product for a slide — accept the added shutter latency.
- **Delegate → async:** `capturePhoto(with:delegate:)` is delegate-based; bridge it with
  `withCheckedThrowingContinuation`, and **retain the delegate** (keyed by
  `settings.uniqueID`) until `didFinishProcessingPhoto` fires — the classic bug is the
  delegate deallocating mid-capture so the continuation never resumes.
- **Zoom:** `videoZoomFactor` on the locked device, clamped to
  `[1.0, min(maxAvailableVideoZoomFactor, ceiling)]`. It is *digital* — it crops the
  sensor, trading resolution for reach — so always capture the still at the full active
  format regardless of zoom.
- **Permissions:** `NSCameraUsageDescription` **must** be in Info.plist (missing key =
  instant crash). `authorizationStatus(.video)`; if `.notDetermined`, `await requestAccess`;
  `.denied`/`.restricted` → a real Settings deep-link, not a dead preview.
- **Orientation:** iOS 17+ `AVCaptureConnection.videoRotationAngle` (guard
  `isVideoRotationAngleSupported`), **not** the deprecated `videoOrientation`. Seal the
  orientation *value* into the immutable `CapturedImage` at the shutter; do **not**
  pre-rotate pixels (the imaging stage applies `.oriented()`).
- **Lifecycle:** start on appear, stop on disappear/background; observe
  `wasInterrupted`/`interruptionEnded` (calls, Control Center, another app grabbing the
  camera) and restart cleanly; watch `systemPressureState` for thermal throttling.

### 3.2 INT-002 — Detect & adjust corners (Vision + coordinate spaces)

- **Pin the three coordinate spaces first.** Vision = **normalized, bottom-left** (0–1);
  SwiftUI/UIKit = **top-left points**; Core Image = **bottom-left pixels**. Write the
  conversions **once**, in one `Geometry` file in `ClassCamCore`, as pure functions
  (`VNImageRectForNormalizedRect` / explicit Y-flip against image height; points↔pixels
  via scale), and **unit-test them on a known rectangle** before any device sees them.
  This is where the app most easily ships a subtly-wrong image.
- **Detection tuned for a glowing angled screen:** `VNDetectRectanglesRequest` with a
  **generous `quadratureTolerance` (~30–45°)** (angled shots are *not* square; tightening
  this means angled quads never detect), **low `minimumConfidence` (~0.3)** (a bright TV
  off-axis is not a crisp document), permissive `minimumAspectRatio` (~0.3) and
  `minimumSize`, a handful of `maximumObservations`; pick the best by confidence.
- **Mandatory fallback:** when nothing returns, present a **default inset box (~10%)** the
  user positions. Detection only *seeds* the initial handle positions; **the user's
  dragged `Quad` is the source of truth**, and detection is **never re-run over their
  edits**.
- **Dragging is pure `@MainActor` UI** on normalized points — no pixels touched.

### 3.3 INT-003 — Perspective-correct, crop, export (Core Image)

- **`.oriented()` first.** Apply the sealed orientation as the *first* node of the graph
  so detection, handles, and warp all live in one upright frame — one owner, no
  "rotated result" bugs smeared across the pipeline.
- **One lazy `CIImage` graph, rendered once:**
  `oriented source → CIPerspectiveCorrection (corners as CIVectors in CI pixel space)
  → [white-balance CIColorMatrix] → CILanczosScaleTransform to exactly 1920 long edge
  → crop to aspect`. `CIImage` is a *recipe*, not a bitmap — compose the whole graph and
  render it a **single time** through the **one reused `CIContext`** into an **sRGB
  `CGImage`**, then PNG-encode. Three separate renders would triple GPU cost and compound
  resampling.
- **Verify the output.** Assert the produced pixel dimensions (1920×1080 or 1920×1440)
  **and** color space after render — a green filter chain is not proof.
- **Export:** **sRGB** (so GoodNotes/Notability render no cast); PNG to `UIPasteboard`
  automatically on Done; `UIActivityViewController` for share. Persist the aspect choice
  (`@AppStorage`) for the next photo.

### 3.4 INT-004 — Tap-to-white-balance (Core Image color math)

- **Gains go in LINEAR light, not gamma-encoded sRGB.** This is the hill the imaging seat
  will die on: sRGB is a nonlinear curve, so multiplying encoded values doesn't scale the
  actual light and leaves a residual cast.
- **The exact math:** tap a should-be-neutral spot → `sampleGains(at:in:)` samples that
  pixel by rendering a 1×1 region **through the same `CIContext` in a tagged color space**
  (never a `UIImage` pixel of unknown encoding) → compute per-channel gains normalized to
  green: **`g_R = G/R`, `g_G = 1`, `g_B = G/B`** → apply as a diagonal `CIColorMatrix` in
  `extendedLinearSRGB` → encode back to sRGB.
- **Non-destructive:** store `{ tappedPoint, gains }` in `ProcessingState`. **Reset** =
  set `whitePoint` to `nil`; **re-tap** = overwrite. Both are cheap graph rebuilds, no
  baked bitmap to undo.
- **The flow stays on the right actors:** tap gesture on `@MainActor` gives a normalized
  point → `await imaging.sampleGains(...)` → store in state → `await imaging.render(...)`.
  Sampling and rendering are separate actor calls sharing one `CIContext`. No pixel work
  ever runs on main.

---

## 4. Project organization

```
ClassCam.xcodeproj / Package
├── ClassCamCore  (pure Swift package — NO UIKit / AVFoundation / SwiftUI)
│   ├── Geometry        // coordinate conversions + Y-flips, quad ordering/validation
│   ├── Color           // per-channel white-balance gain math (linear light)
│   ├── Output          // 1920 long-edge sizing + crop-rect math per aspect
│   └── Model           // Stage, ProcessingState, Quad, WhitePoint, AspectRatio, Slide, CapturedImage
└── ClassCam (app target)
    ├── App             // AppModel (@MainActor @Observable), entry point, @Environment wiring
    ├── Capture         // CaptureService actor, CameraPreview (UIViewRepresentable), Capture screen
    ├── Processing      // Processing screen, draggable-corner overlay, white-balance control
    ├── Result          // Result screen, share/copy/start-over
    └── Imaging         // ImagingService actor (owns the one CIContext), the CIImage graph
```

**Organize by feature slice, not by `Views/Models/Controllers`.** The load-bearing move
is the separate **`ClassCamCore`** package: it imports no Apple UI/capture frameworks, so
it builds and unit-tests on any machine with **zero simulator**. All app correctness that
*can* be pure, *is* pure and lives here; the actors call these functions and never
reimplement the math.

---

## 5. Build sequence

### Step 0 — Walking skeleton (before any feature)
1. Xcode project + `iPadOS` target + the `ClassCamCore` package.
2. Build settings: **Swift 6 language mode, complete strict concurrency, warnings-as-errors.**
3. Info.plist `NSCameraUsageDescription`.
4. Define `Stage` + all Sendable value types in `ClassCamCore`.
5. `AppModel` with hand-wired stage transitions; three empty SwiftUI screens switching on
   `stage`; **stub** `CaptureService`/`ImagingService` actors returning canned values.

> **Milestone:** the app launches and navigates Capture → Processing → Result on fake
> data, compiling strict-concurrency-clean. Everything after this fills stubs.

### Then, per Intent, in order
- **INT-001 Capture** → real `CaptureService`; preview; permissions; zoom; full-res still;
  **the CVPixelBuffer deep-copy fix lands here** (see §6).
- **INT-002 Detect & adjust** → `Geometry` conversions unit-tested first, then
  `detectQuad` + fallback box + draggable handles.
- **INT-003 Correct/crop/export** → single-pass `render` → `Slide`; clipboard + share;
  aspect persisted.
- **INT-004 White balance** → `sampleGains` + non-destructive gains-in-state + Reset.

---

## 6. Risk register (flagged by the council)

| # | Risk | Mitigation (in the sequence) |
|---|---|---|
| R1 | **`CVPixelBuffer` recycled across the actor hop.** AVFoundation stills come from a finite pool; a buffer can be reclaimed out from under a `CapturedImage` held across detect→sample→render. | In `didFinishProcessingPhoto`, **deep-copy** the buffer (`CVPixelBufferCreate` + per-plane `memcpy`, or take `photo.fileDataRepresentation` and let Imaging decode) **before** resuming the continuation, so `CapturedImage` owns memory the pool can't reclaim. **Never** hand a pooled buffer across the actor line. Verified held-buffer integrity is an **INT-001 acceptance check**. |
| R2 | **Untagged pixel buffer breaks linear-light sampling.** If the buffer has no known color space, white-balance sampling is the "sampled a pixel and hoped" trap. | Capture **tags** the buffer's `CGColorSpace`/pixel-format in `CapturedImage` (sRGB unless deliberately wide). Non-negotiable contract between Capture and Imaging. |
| R3 | **Coordinate-space errors ship a plausible-but-wrong warp.** | Conversions written once in `ClassCamCore`, unit-tested corner-for-corner on a known rectangle before any device use. |
| R4 | **`CIContext` per render = performance cliff.** | Exactly one `CIContext`, app-lifetime, owned by `ImagingService`. Code review rejects any context built inside a render function. |
| R5 | **Simulator proves nothing for capture.** | INT-001 verification is on-device manual attestation (see §7). |

---

## 7. Test strategy

**Off-device unit tests (Swift Testing against `ClassCamCore`, no simulator):**
- Quad ordering handles rotated/reversed corner input; validation rejects
  degenerate/self-intersecting quads.
- Coordinate round-trips (normalized-BL ↔ pixel-BL ↔ view-points-TL) are lossless,
  corner-for-corner.
- Linear-light gain math neutralizes a known cast pixel to expected neutral — **plus an
  explicit gamma-vs-linear divergence test** proving the linear path is the correct one.
- Output sizing yields exactly 1920×1080 and 1920×1440.
- `AppModel` state-transition tests with **faked actors** assert illegal states never
  arise (capture→processing→result, Retake, Reset, re-tap).

**On-device manual attestation (per the Intents' verification):**
- Capture: sub-second shutter-to-still at `.quality`; full native resolution; usable still
  in dim, blue-cast classroom light; interruption mid-capture completes or fails cleanly
  and the session restarts; deep-copied buffer survives to Imaging; zoom-then-capture
  delivers a full-format frame.
- Imaging: detection quality on a **real glowing TV/whiteboard viewed off-axis**; handle
  drag feel; visual white-balance correctness (tapped white reads neutral); the exported
  PNG pastes clean into GoodNotes/Notability at 1920 on the long edge.

---

## 8. Consolidated best-practices & pitfalls checklist

**Swift / architecture (Architect)**
- Value types by default; the only reference types are `AppModel` + the two actors. No
  bare mutable classes, no singletons, no global mutable state.
- `@Observable` `AppModel` at root, injected via `@Environment`; ephemeral per-view state
  (drag offset, grabbed-corner index, WB-arm toggle) is local `@State`; `@Binding` only
  for the aspect toggle.
- Structured concurrency from `.task {}`; cancel on disappear; no detached `Task {}`.
- Only the locked value types cross an actor line — never `CIImage`/`CIContext`/
  `AVCaptureSession`/`UIImage`. Views never see a `CIContext`.
- Typed-`throws` error enums (`CaptureError`, `ImagingError`); **no `try!` / force-unwraps**
  on camera-unavailable or encode-failure paths — surface a state, not a crash.
- Don't over-architect: three screens = three views + one model + two actors.

**Capture (AVFoundation)**
- Session config/start/stop **only** on the serial session queue, never main.
- `NSCameraUsageDescription` present; handle `.denied`/`.restricted`, not just `.authorized`.
- `maxPhotoDimensions` = format max; `photoQualityPrioritization = .quality`.
- Retain the photo delegate until callback; bridge to `async` via a continuation.
- `videoZoomFactor` clamped; digital zoom trades resolution for reach; still is always the
  full active format.
- `videoRotationAngle`, **not** the deprecated `videoOrientation`; orientation sealed as
  metadata at the shutter, pixels never pre-rotated.
- **Deep-copy the pixel buffer at the shutter**; tag its color space; never pass a pooled
  buffer across the actor line.
- Handle interruptions + thermal; stop on background; preview gravity `.resizeAspect`.
- **Test on a real iPad — the simulator camera proves nothing.**

**Imaging (Core Image / Vision)**
- Three coordinate spaces: conversions written **once**, tested, never eyeballed.
- `.oriented()` as the first graph node → one upright frame downstream.
- `CIImage` is a recipe — compose the whole graph, render **once**.
- **One** `CIContext`, app-lifetime, on the actor — never per render.
- White balance in **linear light** (`extendedLinearSRGB`): `g_R=G/R, g_G=1, g_B=G/B` via
  diagonal `CIColorMatrix`, then encode to sRGB. Gamma-space gains leave a residual cast.
- Sample the tapped pixel through the `CIContext` in a **tagged** color space — never a
  `UIImage` pixel of unknown encoding.
- Gains + tapped point live in state (non-destructive); no baked bitmap.
- Generous `quadratureTolerance` for angled screens; **mandatory fallback quad** — never
  trust detection blindly.
- Lanczos to exactly 1920; **assert** produced dims **and** color space.
- sRGB export for note-app compatibility; don't read `.extent` on an infinite/generator
  image (it triggers phantom work).

---

## 9. Out of scope (held firm)

Per the Constitution/AE-001/MSN-001 and unchanged by this plan: no OCR/text extraction,
no cloud/backend/accounts, no general photo editing (only the perspective crop + the
single tap-to-white-balance), no web/PWA delivery. Fully on-device, Apple frameworks only.

---

## 10. Handoff

- To build: follow §5 (walking skeleton → INT-001…004), honoring §2 (concurrency), §3
  (per-feature Swift), and §8 (checklist). §6 risks are acceptance gates.
- To industrialize: hand this plan to **`/shotgun`** to generate the intention test,
  refine INT-001…004, decompose into IBs/beads, and design a git-worktree implementation
  team.
- Personas authored for this council live in the personas DB and should be committed
  (see the run's close-out note): `the_swift_app_architect.md`,
  `the_avfoundation_capture_engineer.md`, `the_core_image_vision_engineer.md`, plus
  `personas.db`.
