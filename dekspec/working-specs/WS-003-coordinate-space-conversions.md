# Working Spec: ClassCamCore Geometry — the three-coordinate-space conversion functions

## Status

ACCEPTED

## Created

2026-08-10

## Modified

2026-08-10

## Silent Failure Domain(s)

*The five domains below are the DekSpec host-project (Dektora) domains — none applies to ClassCam. ClassCam's real silent-failure risk for **this** spec is the single most dangerous one in the app: a **coordinate-space conversion error** (a dropped Y-flip between the bottom-left and top-left origins, or a normalized-vs-pixel unit mix-up) ships a **plausible-but-wrong warp** — the corrected image looks reasonable but the corners are subtly off, and **nothing raises an error** (council R3). This spec's entire purpose is to make those conversions provable off-device before any device sees them, encoded in Business Rules BR1–BR8 and Failure Behavior below.*

- [ ] Transformer internals (position IDs, injection layer, KV cache)
- [ ] Numerical precision (quantization, tiered compression, serialization round-trips)
- [ ] GPU multi-process isolation (device assignment, process crash recovery)
- [ ] Graph consistency (shadow graph / Neo4j flush, phantom nodes)
- [ ] Timeline coherence (topic segmentation, tier assignment, decay, shadow timeline / PostgreSQL)

## Expertise Audit Record

*ClassCam is a native Swift/Apple app; the Dektora expertise roles (ML/Model, Quantization, CUDA, Graph, Embedding Geometer, Pipeline Analyst) are all non-triggered — this spec is pure 2D coordinate geometry (origin flips + scale), touching none of injection, tensors, CUDA, graph stores, embeddings, or ML pipeline ordering. AE-001 is classified **Core** (the app's competitive correctness lives here, and this spec is its most correctness-critical pure surface), so the audit is recorded for completeness rather than omitted.*

| Role | Triggered | Trigger rule | Rationale |
|------|-----------|-------------|-----------|
| ML / Model Behavior Expert | No | injection layer, position IDs, KV cache | No model; ClassCam runs no LLM. |
| Quantization / Precision Expert | No | tensor dtype / bit depth / precision threshold | No tensors; the only numeric concern is floating-point round-trip losslessness, addressed in BR1–BR3. |
| CUDA Multi-Process Expert | No | more than one device or process boundary | Single-process on-device app (ADR-003); pure value functions, no device. |
| Graph / Multi-Store Expert | No | shadow graph / Neo4j / shadow timeline / PostgreSQL | No datastore. |
| Embedding Space Geometer | No | similarity / distance / centroid | 2D image geometry, not embedding-space geometry; no similarity metric. |
| Pipeline Sequencing Analyst | No | pipeline-stage ordering change | These are leaf conversion functions, not a pipeline reorder. |

## Related Architecture Elements

- AE-001: ClassCam App — this spec measures the `ClassCamCore` `Geometry` surface: the pure functions that move the presentation's four corner points between Vision, SwiftUI/UIKit, and Core Image coordinate spaces.

## Governing ADRs

- ADR-007: Coordinate-space conversion discipline — this spec **is** the "written once, unit-tested on a known rectangle" surface that ADR-007 mandates; detection seeds and the dragged `Quad` is the source of truth (WS-004 consumes this).
- ADR-006: ClassCamCore UI-free pure package — every function here lives in `ClassCamCore/Geometry` and imports no UI/capture framework, so it unit-tests with zero simulator.
- ADR-002: Apple-native Vision + Core Image — the normalized-bottom-left space is Vision's convention; the pixel-bottom-left space is Core Image's; conversions may use `VNImageRectForNormalizedRect`.

## Interface Contracts

**Consumed contracts:** IC-001 (`CapturedImage`) — supplies the pixel dimensions and sealed orientation the conversions scale/flip against.
**Defined contracts:** none (these are pure `ClassCamCore` functions consumed in-package by WS-004; no new cross-component boundary type).

## What This Does

This spec defines the **pure coordinate-space conversion functions** that live in one `Geometry` file in the `ClassCamCore` package (ADR-006, ADR-007). The app moves the same four corner points across three spaces that disagree on origin and units:

- **Vision** — normalized (0–1), **bottom-left** origin.
- **Core Image** — **bottom-left pixels**.
- **SwiftUI/UIKit** — **top-left points**.

These functions are the single audited home for every origin flip and unit scale between those spaces. They are pure functions of the point(s) plus the image pixel dimensions (and the view size / scale for the point conversions); they hold no state and touch no framework image object.

**Mechanism:** normalized-BL→pixel-BL multiplies by the image pixel dimensions (optionally via `VNImageRectForNormalizedRect`); pixel-BL→view-points-TL flips Y against the image height and scales by the view/image ratio; every conversion has an inverse, and the round-trip is asserted lossless corner-for-corner on a known rectangle before any device use.

## What This Does NOT Do

- **Detection:** does not run `VNDetectRectanglesRequest` or produce a `Quad` from an image — it only converts points once detection (WS-004) has them.
- **Rendering:** does not apply the perspective warp, crop, or export (INT-003) — it supplies the pixel-space corners that warp will consume.
- **Quad ordering/validation:** does not order or validate a `Quad` — that is WS-001's surface (BR6–BR7 there); this spec converts already-ordered points between spaces.
- **Drag/UI:** does not own the draggable-handle gesture or the fallback box (WS-004) — it is the math those consume.

## Interfaces

### Data Interfaces

| Interface | Direction | Type / Shape | Source or Consumer | Guarantees |
|-----------|-----------|--------------|--------------------|-----------|
| normalized-BL → pixel-BL | in→out | `CGPoint` (0–1) + image `CGSize` (px) → `CGPoint` (px) | detection (WS-004) → warp (INT-003) | Multiplies by pixel dims; no origin flip (both bottom-left). |
| pixel-BL → normalized-BL | in→out | `CGPoint` (px) + image `CGSize` (px) → `CGPoint` (0–1) | inverse of the above | Exact inverse; round-trip lossless (BR1). |
| pixel-BL → view-points-TL | in→out | `CGPoint` (px) + image `CGSize` + view `CGSize` → `CGPoint` (pt) | overlay seeding (WS-004) | Y-flip against image height + scale by view/image ratio (BR2). |
| view-points-TL → pixel-BL | in→out | `CGPoint` (pt) + view `CGSize` + image `CGSize` → `CGPoint` (px) | inverse of the above | Exact inverse; round-trip lossless (BR2). |
| normalized-BL ↔ view-points-TL | in→out | composed of the two above | seed handles from a detected quad | Composition is associative and lossless corner-for-corner (BR3). |

### Process Interfaces

*Omitted — single-process, in-memory pure functions. No boundary crosses a process or an actor; these functions are called from both the `@MainActor` overlay (WS-004) and the imaging actor (INT-003), and being pure `Sendable`-safe value functions they are safe on either (ADR-005).*

### Dependencies

| Dependency | Interface | Failure behavior |
|------------|-----------|-----------------|
| `CapturedImage` (IC-001) | supplies pixel dimensions + sealed orientation | If dimensions are wrong the conversion is silently wrong — this is exactly why BR1–BR3 unit-test the round-trip against known dimensions before device use. |
| CoreGraphics `CGPoint`/`CGSize`/`CGAffineTransform` | value primitives | Pure value types; no failure surface. |

## Domain Constraints

| Constraint | Value | Scope | Rationale |
|------------|-------|-------|-----------|
| Package purity | Functions live in `ClassCamCore/Geometry`; import no UIKit/AVFoundation/SwiftUI/Vision-UI | all-IBs | Keeps them unit-testable with zero simulator (ADR-006); `VNImageRectForNormalizedRect` is a pure geometry helper (no image object) if used. |
| Single home | Every origin flip / unit scale between the three spaces exists **only** here | all-IBs | ADR-007 — no call site re-derives a flip; one place for a coordinate bug to hide. |
| Purity | Functions are pure (output depends only on inputs); no stored state | all-IBs | Deterministic + testable; safe to call from `@MainActor` or the imaging actor (ADR-005). |
| Coordinate units | Inputs/outputs carry an explicit space (normalized-BL / pixel-BL / points-TL) in the function name/signature | all-IBs | The space is unmistakable at the call site; a mix-up is a type/name error, not a silent flip. |
| Orientation | Conversions honour the sealed `CGImagePropertyOrientation` from `CapturedImage` (IC-001) when mapping to pixels | all-IBs | The upright frame is established by `.oriented()` in INT-003; conversions must agree on which axis is height. |
| Compute device / dtype | n/a — pure `CGFloat` geometry, no tensors or device pinning | all-IBs | ClassCam has no tensor/device surface; the Dektora rows do not apply. |

## Governing Formulas

| Formula | Expression | Variables | Units / Scale | Valid range | Validated by |
|---------|-----------|-----------|---------------|-------------|-------------|
| Normalized→pixel | `px = (n.x · W, n.y · H)` | `n` normalized point; `W,H` image pixel dims | px | `n ∈ [0,1]²` | this spec (BR1, round-trip test) |
| Pixel-BL→points-TL | `pt = (px.x · s, (H − px.y) · s)` | `s` view/image scale; `H` image height (px) | pt | in-bounds points | this spec (BR2, round-trip test) |
| Compose | `norm→pt = (norm→px) ∘ (px→pt)` | as above | — | — | this spec (BR3, corner-for-corner test) |

## Business Rules

1. **general** normalized-BL ↔ pixel-BL is an exact inverse pair: converting a point to pixels and back to normalized returns the original within floating-point tolerance — a unit test feeds the four corners of a known rectangle and asserts corner-for-corner losslessness (council R3).
2. **general** pixel-BL ↔ view-points-TL is an exact inverse pair including the Y-flip against image height and the view/image scale: converting to points and back returns the original within tolerance — a unit test asserts this on a known rectangle with a known view size.
3. **general** The composed normalized-BL ↔ view-points-TL round-trip (seed a handle from a detected corner, read the dragged corner back) is lossless **corner-for-corner** on a known rectangle — the canonical council-§7 test; it fails the build if any corner drifts.
4. **general** The Y-axis is flipped exactly once between a bottom-left space (Vision/Core Image) and the top-left space (SwiftUI/UIKit); a test that flips normalized-BL→points-TL asserts the top corner in one space maps to the bottom corner's row in the other (catches a missing or double flip).
5. **general** normalized-BL→pixel-BL carries **no** origin flip (both are bottom-left) — only a scale by pixel dimensions; a test asserts a bottom-left normalized corner maps to a small-Y pixel coordinate, not a flipped one.
6. **general** The conversions honour the image's sealed orientation (IC-001): the dimension used as "height" for the Y-flip is the oriented height, so a portrait-vs-landscape capture flips against the correct axis — a test parameterized on orientation asserts the corner mapping stays correct.
7. **general** Every function is pure — same inputs always yield the same output, no shared mutable state — so it is safe to call from the `@MainActor` overlay (WS-004) and the imaging actor (INT-003) alike; verified by the functions taking all dimensions as parameters (no capture of external state).
8. **general** A degenerate scale input (zero image dimension) is not silently divided-by — the conversion either is precondition-guarded or the caller guarantees non-zero dimensions from a valid `CapturedImage`; a test feeds a zero dimension and asserts the guarded behavior (no NaN corner leaks into a `Quad`).

## Failure Behavior

*ClassCam has no server/exception-telemetry surface; the observable signal for these pure functions is a **unit-test assertion** on a known rectangle (compile-and-test observable). A coordinate error does not throw — it produces a wrong number — so the "detection" of the failure IS the round-trip test asserting losslessness before any device run (council R3).*

| Failure | Detection | Assertion type | Behavior | Recovery |
|---------|-----------|---------------|----------|----------|
| Dropped / doubled Y-flip | Round-trip unit test on a known rectangle (BR1–BR4) | assert | Test fails the build; the wrong warp never ships | Fix the single conversion function; one place to fix |
| Normalized-vs-pixel unit mix-up | Corner-for-corner composition test (BR3) | assert | Build fails | Fix the conversion; the space is named in the signature |
| Wrong axis used as height (orientation) | Orientation-parameterized test (BR6) | assert | Build fails | Honour the sealed IC-001 orientation |
| Zero image dimension → NaN corner | Guard / precondition test (BR8) | assert | Guarded; no NaN corner reaches a `Quad` | Caller supplies valid `CapturedImage` dims |

## Open Issues

- *None. The conversion surface is fully specified for INT-002's scope; INT-003's warp consumes the same pixel-space conversions unchanged.*

## Amendment Log

| Date | Type | Change | Author |
|------|------|--------|--------|
| 2026-08-10 | Substantive | Initial authoring — `ClassCamCore` `Geometry` three-space coordinate-conversion contract; the "written once, unit-tested" surface ADR-007 mandates (INT-002 IU1). | noreply@anthropic.com |
| 2026-08-10 | Substantive | Accepted as INT-002 child spec. | noreply@anthropic.com |
