# ADR-007: Convert between the three coordinate spaces once, in ClassCamCore, unit-tested before device use; detection seeds handles, the dragged Quad is the source of truth

## Status

ACCEPTED

## Supersession

*Supersedes:* none
*Superseded by:* none

## Related Architecture Elements

- AE-001: ClassCam App — fixes how the processing stage moves corner points between Vision, SwiftUI/UIKit, and Core Image coordinate spaces, and who owns the authoritative quad during corner adjustment.

## Created

2026-08-10

## Modified

2026-08-10

## Date

2026-08-10

## Deciders

Jeff Haskin; ClassCam Swift persona council (Swift/SwiftUI App Architect · AVFoundation Capture Engineer · Core Image/Vision Imaging Engineer) — unanimous consensus.

## Context and Decision Drivers

ClassCam's corner-detection and correction flow moves the same four points across three coordinate spaces that disagree on both origin and units: **Vision** returns normalized coordinates (0–1) with a **bottom-left** origin; **SwiftUI/UIKit** works in **top-left** points; **Core Image** works in **bottom-left pixels**. A single missed Y-flip or a normalized-vs-pixel mix-up produces a warp that looks plausible on screen but is subtly wrong — the corners land near, but not on, the real presentation edges, and nothing raises an error (council risk R3). This is the app's single most likely path to shipping a silently incorrect image.

Compounding the risk, detection and manual adjustment can fight each other: if detection re-runs while the user is dragging a handle, the user's deliberate correction is overwritten by a fresh guess, and the two coordinate paths (detected vs dragged) must agree exactly or the handle visibly jumps.

**Decision drivers:**
- Coordinate errors are silent — they produce a wrong-but-believable result with no exception, so they must be caught by construction and by test, not by inspection on device.
- The same conversion is needed in several places (seed the handles, feed the warp), so it must exist once, not be re-derived per call site where a flip could be dropped.
- The conversions are pure geometry with no framework dependency, so they can live in the UI-free `ClassCamCore` package (ADR-006) and be unit-tested with zero simulator (council §7) — the cheapest place to prove them correct.
- The user's manual adjustment must be authoritative; detection is a convenience that seeds the starting position, not a process that keeps second-guessing the user.

*Technical story:* Persona-council synthesis, `dekspec/historical-artifacts/classcam-swift-council-plan.md` §3.2 (Detect & adjust corners), §6 risk R3, §7 test strategy.

## Decision

The conversions between the three coordinate spaces — **Vision normalized bottom-left ↔ Core Image pixel bottom-left ↔ SwiftUI/UIKit view-point top-left** — are written **once**, as **pure functions in a single `Geometry` file in the `ClassCamCore` package** (ADR-006), and are **unit-tested on a known rectangle for corner-for-corner round-trip losslessness before any device sees them**. The Vision-normalized→pixel step uses the framework normalization helper (`VNImageRectForNormalizedRect`) or an explicit Y-flip against the image height; the point↔pixel step is a scale by the view/image ratio. No call site re-implements a flip or a scale inline; every coordinate movement in the app routes through these functions.

Detection **only seeds** the initial handle positions. Once the overlay is on screen, **the user's dragged `Quad` is the single source of truth**, and **detection is never re-run over the user's edits** — a fresh detection cannot overwrite a corner the user has moved. Corner dragging is pure `@MainActor` UI math on **normalized points**; no pixels are touched on the main thread (ADR-005). Pixel-space coordinates are produced only at the moment the warp needs them (INT-003), by calling the same tested conversions.

## Options Considered (if applicable)

### Option A: Conversions written once as pure functions in ClassCamCore, unit-tested on a known rectangle; detection seeds, dragged Quad is authoritative

**Pros:** the silent-warp risk is caught at build/test time in the cheapest environment (no simulator); the flip/scale exists in exactly one audited place; the user's correction is never clobbered by re-detection; corner math stays off the pixel path and off the main thread.
**Cons:** requires the discipline of routing every coordinate movement through the package rather than doing a "quick" inline flip at a call site.

### Option B: Convert coordinates ad hoc at each call site in the view/imaging layers

**Pros:** no package indirection; each site does exactly the flip it needs where it needs it.
**Cons:** the same flip is re-derived several times, so one dropped Y-flip ships a plausible-but-wrong warp with no error (council R3); the logic is entangled with UIKit/Core Image and cannot be unit-tested without a simulator; detection and drag paths can disagree and make handles jump.

## Consequences

**Positive:**
- The most dangerous silent failure in the app (a wrong-but-believable warp) is provable by a fast off-device unit test on a known rectangle before any device run.
- There is one audited home for every origin/unit conversion; a coordinate bug has exactly one place to hide.
- The user's manual corner adjustment is authoritative and stable — detection cannot overwrite it, and the seeded and dragged paths share the same conversions so handles never jump.

**Negative:**
- Every coordinate movement must route through the `ClassCamCore` `Geometry` functions rather than an inline flip, which is a discipline the reviewer must enforce.
- The `Geometry` surface must be kept in sync with the actual image dimensions/orientation it converts against (the sealed orientation from IC-001 / INT-001).

## Validation

**Observable confirmation:**
Off-device unit tests round-trip the four corners of a known rectangle through normalized-BL ↔ pixel-BL ↔ view-points-TL and back, asserting corner-for-corner losslessness; on device, the detected quad's handles land on the real presentation edges, and dragging a handle then triggering a re-detection scenario never moves the user's corner.

**Reconsideration triggers:**
A coordinate conversion is needed that cannot be expressed as a pure function of image dimensions/orientation (e.g. a nonlinear lens correction enters the pipeline), or a future feature genuinely requires detection to update the quad after the user has begun editing.

## Links

- ADR-006: Isolate correctness-critical pure logic in a UI-free ClassCamCore package — the `Geometry` conversions live in that package and unit-test with zero simulator.
- ADR-002: Use Apple-native Vision and Core Image — the coordinate spaces reconciled here are Vision's and Core Image's; detection uses `VNDetectRectanglesRequest`.
- ADR-005: Swift 6 strict-concurrency actor-isolated services — corner dragging is `@MainActor` normalized-point math; pixel conversion runs on the imaging actor, never main.

## Open Issues

- *None currently.*

## Amendment Log

| Date | Type | Change | Author |
|------|------|--------|--------|
| 2026-08-10 | Substantive | Initial authoring — records the coordinate-space conversion discipline (conversions once in `ClassCamCore`, unit-tested; detection seeds, dragged `Quad` authoritative) for INT-002 (coverage finding; council plan §3.2, R3). | noreply@anthropic.com |
| 2026-08-10 | Substantive | Accepted with INT-002 (council-ratified coordinate-space discipline). | noreply@anthropic.com |
