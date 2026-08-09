# ADR-008: Compose one lazy CIImage graph and render it once through a single reused CIContext; export sRGB at 1920 long edge

## Status

ACCEPTED

## Supersession

*Supersedes:* none
*Superseded by:* none

## Related Architecture Elements

- AE-001: ClassCam App — fixes how the correction/crop/export stage turns a placed quad into a finished slide: one lazy image recipe rendered a single time through the one app-lifetime image-processing context, into a verified-size sRGB image.

## Created

2026-08-10

## Modified

2026-08-10

## Date

2026-08-10

## Deciders

Jeff Haskin; ClassCam Swift persona council (Swift/SwiftUI App Architect · AVFoundation Capture Engineer · Core Image/Vision Imaging Engineer) — unanimous consensus.

## Context and Decision Drivers

ClassCam's payoff stage takes the user's four placed corners and turns the skewed presentation region into a clean, rectilinear slide (ADR-002 Apple-native Core Image; ADR-005 the imaging work lives on the `ImagingService` actor). Core Image offers two failure modes here that both ship a plausible-but-wrong or a janky result with no error signal: (1) building a fresh `CIContext` inside each render — a `CIContext` compiles and caches GPU state, so constructing one per render is a large, repeated cost that turns a snappy correction into a stutter (council R4, the "performance cliff"); and (2) treating the intermediate `CIImage`s as bitmaps and rendering the correction, the scale, and the crop as separate passes — three renders triple GPU cost and compound resampling. Core Image's own model is the opposite: a `CIImage` is a *recipe*, not pixels, so the whole correction can be composed as one lazy graph and materialized exactly once. Finally, the produced pixels must be verified: a filter chain that "looks green" is not proof it emitted the right dimensions or colour space, and a slide with the wrong size or an embedded wide-gamut profile pastes into GoodNotes/Notability with a cast or at the wrong resolution.

**Decision drivers:**
- A `CIContext` is expensive to build and cheap to reuse; the app renders repeatedly (every Done, every white-balance re-tap in INT-004), so the context must outlive any single render.
- `CIImage` is a deferred recipe; composing the whole correction as one graph and rendering once is both the cheapest and the most correct path (single resample, single GPU pass).
- The output is consumed by third-party note apps that assume sRGB; a wrong colour space or a wrong pixel size is a silent, shipped defect.
- The correctness-critical *sizing* math (1920 long edge, crop rect per aspect) is pure and belongs off the GPU, unit-tested, in `ClassCamCore` (ADR-006).

*Technical story:* Persona-council synthesis, `dekspec/historical-artifacts/classcam-swift-council-plan.md` §3.3 (and §1.2 for the locked `render` surface, §7 for the output-sizing test).

## Decision

The perspective correction is expressed as **one lazy `CIImage` graph** — `oriented source → CIPerspectiveCorrection (the four corners as `CIVector`s in Core-Image pixel space) → [white-balance colour node — a slot INT-004 fills; a nil white point means no node] → CILanczosScaleTransform to exactly 1920 on the long edge → crop to the selected aspect` — and rendered a **single time** through **one `CIContext` created once at app-lifetime and owned by `ImagingService`** (ADR-005), producing an **sRGB `CGImage`** that is then PNG-encoded for export. A `CIContext` is **never** constructed inside a render function; the one context is reused for every render and for INT-004's 1×1 sampling render. After the render, the code **asserts the produced pixel dimensions** (1920×1080 for 16:9, 1920×1440 for 4:3) **and the colour space** (sRGB) — a green-looking chain is not accepted as proof. The pure output-sizing and crop-rect math (long-edge scaling, per-aspect crop rectangle) lives in `ClassCamCore`'s `Output` slice as unit-tested pure functions (ADR-006), which the actor calls rather than reimplementing on the GPU path.

Rejected: a `CIContext` per render (the council-R4 performance cliff); rendering the correction, the scale, and the crop as three separate passes (triple GPU cost, compounded resampling); and shipping the render output unverified (trusting the visual result over an explicit dimensions-and-colour-space assertion).

## Options Considered (if applicable)

### Option A: One lazy CIImage graph rendered once through a single app-lifetime CIContext, output asserted

**Pros:** the `CIContext`'s compiled/cached state is paid for once and amortized across every render; a single lazy graph means one resample and one GPU pass; the render-once path is the shape Core Image is designed for; the post-render dimensions+colour-space assertion catches a wrong-size or wrong-profile slide before it ships; the sizing math is pure and unit-tested off-device.
**Cons:** requires the discipline that no render function ever builds a context, and that the corner vectors are converted into Core-Image pixel space (via the ClassCamCore Geometry conversions) before feeding `CIPerspectiveCorrection`.

### Option B: Build a CIContext per render and materialize each stage separately

**Pros:** each render is self-contained; no shared long-lived object to manage.
**Cons:** rebuilding the `CIContext` every render is the exact performance cliff the council flagged (R4); materializing the correction, scale, and crop separately triples GPU cost and compounds resampling artifacts; nothing verifies the produced size or colour space, so a wrong-size/wrong-profile slide ships silently.

## Consequences

**Positive:**
- Rendering is fast and stays fast: the expensive context is built once and reused, so correction and re-tap feel immediate.
- One lazy graph rendered once gives a single, clean resample into a verified sRGB slide that pastes correctly into note apps.
- The output-size/crop math is pure, unit-tested (exactly 1920×1080 and 1920×1440), and shared with no GPU dependency.

**Negative:**
- The single `CIContext` is shared mutable framework state living on the `ImagingService` actor; every render path (INT-003 correction, INT-004 sampling + re-render) must route through it and never build its own.
- The corner coordinates must be converted into Core-Image pixel space before the graph is built (delegated to the ClassCamCore Geometry conversions authored under INT-002), adding a conversion step the render depends on.

## Validation

**Observable confirmation:**
On device, tapping Done produces a rectilinear slide with no per-render stutter; the produced `CGImage` measures exactly 1920×1080 (16:9) or 1920×1440 (4:3) and carries an sRGB colour space (both asserted in code after render); the exported PNG pastes into GoodNotes/Notability at 1920 on the long edge with no colour cast; a code review finds no `CIContext` constructed inside any render function.

**Reconsideration triggers:**
The single shared `CIContext` becomes a contention or memory-pressure problem under the app's render frequency (indicating a pool of contexts is warranted); or a future output target needs a colour space other than sRGB or a long edge other than 1920, requiring the fixed output contract to become configurable.

## Links

- ADR-002: Apple-native Vision + Core Image — the framework this render graph is built on.
- ADR-005: Swift 6 strict-concurrency actor-isolated services — `ImagingService` is the actor that owns the one `CIContext`; only `Sendable` value types (`CapturedImage`, `Slide`) cross its boundary.
- ADR-006: ClassCamCore UI-free pure package — the output-sizing and crop-rect math lives here as unit-tested pure functions.
- ADR-007: coordinate-space conversions (INT-002) — the corners are converted into Core-Image pixel space via these conversions before feeding `CIPerspectiveCorrection`.
- IC-001: CapturedImage contract — the tagged, deep-copied still this render consumes as its source.

## Open Issues

- *None currently.*

## Amendment Log

| Date | Type | Change | Author |
|------|------|--------|--------|
| 2026-08-10 | Substantive | Initial authoring — records the one-lazy-`CIImage`-graph, render-once, single-app-lifetime-`CIContext`, sRGB-at-1920-long-edge, assert-dims-and-colour decision (INT-003 coverage finding; council plan §3.3/§1.2). | noreply@anthropic.com |
| 2026-08-10 | Substantive | authored under INT-003 --decompose | jeffhaskin1@gmail.com |
| 2026-08-10 | Substantive | Accepted with INT-003 (council-ratified §3.3) | noreply@anthropic.com |
