# Working Spec: Render pipeline — perspective correction, 1920 long-edge scale, crop to aspect

## Status

ACCEPTED

## Created

2026-08-10

## Modified

2026-08-10

## Silent Failure Domain(s)

*None of the five Dektora domains applies. ClassCam's real silent-failure risks for **this** spec are (a) a **`CIContext` built per render** — no error, just a performance cliff that janks every correction (council R4) — and (b) a **wrong-size or wrong-colour-space slide shipped silently** because a green-looking filter chain was trusted over an explicit assertion. Both are pinned in ADR-008 and re-encoded here as Business Rules BR6–BR8 and Failure Behavior.*

- [ ] Transformer internals (position IDs, injection layer, KV cache)
- [ ] Numerical precision (quantization, tiered compression, serialization round-trips)
- [ ] GPU multi-process isolation (device assignment, process crash recovery)
- [ ] Graph consistency (shadow graph / Neo4j flush, phantom nodes)
- [ ] Timeline coherence (topic segmentation, tier assignment, decay, shadow timeline / PostgreSQL)

## Expertise Audit Record

*Native Swift/Core Image rendering; no Dektora role triggers. AE-001 is **Core**, so the audit is recorded for completeness. The relevant expertise (Core Image graph composition, colour space) is supplied by the ClassCam persona council, not the Dektora role set.*

| Role | Triggered | Trigger rule | Rationale |
|------|-----------|-------------|-----------|
| ML / Model Behavior Expert | No | injection / position IDs / KV cache | No model. |
| Quantization / Precision Expert | No | tensor dtype / bit depth / precision threshold | Pixels are camera stills processed by Core Image, not tensors; no quantization. The one numeric surface (output pixel dimensions) is exact integer sizing, specified in BR3–BR5, not a precision threshold. |
| CUDA Multi-Process Expert | No | multiple devices / process boundaries | Single process; Core Image renders on the one `CIContext`, an intra-process framework object (ADR-005/008), not a CUDA device. |
| Graph / Multi-Store Expert | No | shadow graph / Neo4j / timeline stores | The "graph" here is a lazy `CIImage` filter graph, not a datastore. |
| Embedding Space Geometer | No | similarity / distance | No embeddings. |
| Pipeline Sequencing Analyst | No | pipeline-stage reordering | The render graph's node order is authored here (BR1–BR2), not reordered from an existing pipeline. |

## Related Architecture Elements

- AE-001: ClassCam App — this spec measures the correction/crop stage: how a placed quad + a source still become a rectilinear, correctly-sized slide.

## Governing ADRs

- ADR-008: Compose one lazy `CIImage` graph, render once through the single reused `CIContext`, export sRGB at 1920 long edge — this spec is the behavioral contract for that decision.
- ADR-002: Apple-native Vision + Core Image — the render uses `CIPerspectiveCorrection`, `CILanczosScaleTransform`, and `CIContext`.
- ADR-006: ClassCamCore UI-free pure package — the output-sizing + crop-rect math lives here as unit-tested pure functions.
- ADR-007: coordinate-space conversions (INT-002) — the four corners are converted into Core-Image pixel space via these conversions before feeding `CIPerspectiveCorrection`; this spec consumes those conversions and does not re-derive them.

## Interface Contracts

**Consumed contracts:** IC-001 (`CapturedImage`) — the tagged, deep-copied still this render reads as its oriented source.
**Defined contracts:** none (the produced `Slide` is defined by WS-001; this spec fills its image payload and sizing).

## What This Does

This spec defines how `ImagingService.render(_:quad:whitePoint:aspect:)` turns a `CapturedImage` plus a validated `Quad` and a selected `AspectRatio` into a finished `Slide`. It composes **one lazy `CIImage` graph** — apply the sealed orientation as the **first** node, feed the four corners (converted into Core-Image pixel space via the ClassCamCore Geometry conversions, WS-003/ADR-007) as `CIVector`s to `CIPerspectiveCorrection`, leave a **slot** for INT-004's white-balance colour node (a `nil` white point means no node), scale to **exactly 1920 on the long edge** with `CILanczosScaleTransform`, and crop to the selected aspect — and renders it a **single time** through the **one app-lifetime `CIContext`** into an **sRGB `CGImage`**. The pure output-sizing and crop-rect math (long-edge scale factor, per-aspect crop rectangle) is owned by this spec but lives in `ClassCamCore`'s `Output` slice as unit-tested pure functions. After the render, the produced pixel dimensions and colour space are **asserted**.

**Mechanism:** `render` builds the whole `CIImage` recipe lazily (no intermediate bitmap), computes the target size + crop rect via the pure `Output` functions, and calls `context.createCGImage(_:from:colorSpace:)` **once** on the single reused `CIContext`; it then asserts the result is 1920×1080 (16:9) or 1920×1440 (4:3) and sRGB before wrapping it in a `Slide`.

## What This Does NOT Do

- **Colour correction:** does not compute or apply white-balance gains — it only leaves the colour-node slot in the graph for INT-004 (a `nil` white point = no node); the gain math is INT-004's spec.
- **Coordinate conversion:** does not re-derive the Vision↔UIKit↔Core Image conversions — it consumes the ClassCamCore Geometry conversions (WS-003/ADR-007) to place the corners in Core-Image pixel space.
- **Capture / detection / drag:** does not capture, detect corners, or handle the drag UI (INT-001/002); it starts from an already-placed, validated `Quad`.
- **Export / result UI:** does not place anything on the clipboard, present a share sheet, or persist the aspect choice — that is WS-006.
- **Context ownership plumbing beyond render:** does not create a `CIContext` per call — it uses the one context ADR-008 establishes on the actor.

## Interfaces

### Data Interfaces

| Interface | Direction | Type / Shape | Source or Consumer | Guarantees |
|-----------|-----------|--------------|--------------------|-----------|
| `render(_:quad:whitePoint:aspect:) async throws` | out | `-> Slide` | AppModel (Done) → result stage | Returns a `Slide` whose image is a verified 1920-long-edge sRGB `CGImage` cropped to `aspect`, or throws `ImagingError`. |
| `CapturedImage` | in | thin envelope (IC-001) | source still | Read-only source; oriented via its sealed orientation as the first graph node. |
| `Quad` | in | four normalized `CGPoint` (0–1), validated | placed corners (INT-002) | Converted to Core-Image pixel-space `CIVector`s before `CIPerspectiveCorrection`. |
| `AspectRatio` | in | `{ sixteenNine, fourThree }` | selected aspect | Selects the crop rect + the asserted output dimensions. |
| `WhitePoint?` | in | `{ tappedPoint; gains }?` | INT-004 | `nil` → no colour node; non-nil → INT-004's colour node fills the slot. |

### Process Interfaces

*Omitted — single-process, in-memory only. The only boundary is the `ImagingService` actor hop (ADR-005), across which the source `CapturedImage` (in) and the produced `Slide` (out) are `Sendable`; the `CIImage`/`CIContext`/`CGImage` never leave the actor.*

### Dependencies

| Dependency | Interface | Failure behavior |
|------------|-----------|-----------------|
| The one `CIContext` (ADR-008) | app-lifetime, owned by `ImagingService` | Reused every render; never built per call. A render-time GPU failure throws `ImagingError.renderFailed`. |
| ClassCamCore Geometry (WS-003) | corner → Core-Image pixel-space conversion | Consumed as pure functions; a conversion defect would misplace the warp (guarded by INT-002's conversion tests). |
| ClassCamCore Output (this spec, IB-007) | 1920 long-edge scale + per-aspect crop rect | Pure functions, unit-tested to yield exactly 1920×1080 / 1920×1440. |
| `CIPerspectiveCorrection` / `CILanczosScaleTransform` | Core Image filters | Fed the pixel-space corners / the target scale; produce lazy `CIImage`s (no eager bitmap). |

## Domain Constraints

| Constraint | Value | Scope | Rationale |
|------------|-------|-------|-----------|
| Orientation node | `.oriented(...)` applied as the **first** graph node | all-IBs | One upright frame downstream; no "rotated result" bugs smeared across the pipeline (council §3.3). |
| Corner space | Four corners fed to `CIPerspectiveCorrection` as `CIVector`s in **Core-Image pixel space** | all-IBs | Core Image is bottom-left pixels; the corners arrive normalized and must be converted (WS-003/ADR-007) — never eyeballed. |
| White-balance slot | A single optional colour node between perspective and scale; `nil` white point = node absent | all-IBs | Keeps the graph one recipe; INT-004 fills the slot without restructuring (ADR-008). |
| Long-edge size | `CILanczosScaleTransform` to **exactly 1920** on the long edge | all-IBs | Fixed output contract (ADR-008); pastes into note apps at a predictable resolution. |
| Crop | Crop to the selected aspect → **1920×1080** (16:9) or **1920×1440** (4:3) | all-IBs | The two closed aspects (WS-001 BR8); the crop rect is pure `Output` math. |
| Render count | Rendered **once** through the one `CIContext`; the graph stays lazy until that single `createCGImage` | all-IBs | `CIImage` is a recipe; three passes triple GPU cost + compound resampling (ADR-008). |
| Context lifetime | The `CIContext` is the one app-lifetime instance; **never** built inside `render` | all-IBs | Council R4 performance cliff (ADR-008). |
| Colour space | Output `CGImage` is **sRGB** | all-IBs | Note apps assume sRGB; a wide-gamut profile reads as a cast (ADR-008). |
| Output assertion | Produced dimensions **and** colour space asserted after render | all-IBs | A green-looking chain is not proof (ADR-008); a wrong-size/wrong-profile slide must not ship. |
| Compute device / dtype | n/a — Core Image on the one `CIContext`, no tensors or device pinning | all-IBs | The Dektora device/dtype rows do not apply. |

## Governing Formulas

| Formula | Expression | Variables | Units / Scale | Valid range | Validated by |
|---------|-----------|-----------|---------------|-------------|-------------|
| Long-edge scale | `s = 1920 / max(correctedWidth, correctedHeight)` | `correctedWidth/Height` = perspective-corrected extent | scale factor (×) | `s > 0` | ClassCamCore Output unit tests (IB-007) |
| 16:9 output | `(w, h) = (1920, 1080)` | — | pixels | exact | ClassCamCore Output unit test (BR3) |
| 4:3 output | `(w, h) = (1920, 1440)` | — | pixels | exact | ClassCamCore Output unit test (BR4) |
| Crop rect | centered rect of the aspect's `(w, h)` within the scaled image | scaled extent, aspect | pixels (bottom-left origin) | within scaled extent | ClassCamCore Output unit test (BR5) |

## Business Rules

1. **general** The sealed orientation is applied as the **first** node of the `CIImage` graph (before perspective correction), so detection, corners, and warp all live in one upright frame — verified by code review + on-device attestation (result is upright regardless of capture orientation).
2. **general** The four `Quad` corners are converted into Core-Image pixel space (via ClassCamCore Geometry, WS-003) and fed to `CIPerspectiveCorrection` as `CIVector`s; the render never consumes normalized or view-space corners directly — verified by code review (conversion correctness is INT-002's unit test).
3. **general** The pure `Output` sizing function scales the corrected image to **exactly 1920 on the long edge** via `CILanczosScaleTransform` — verified by unit test on the pure scale-factor math (IB-007).
4. **general** Cropping to `.sixteenNine` yields exactly **1920×1080**; cropping to `.fourThree` yields exactly **1920×1440** — verified by unit test on the pure crop-rect/size math (IB-007) and re-asserted on-device after render.
5. **general** The crop rect is computed by the pure `Output` function (centered rect of the aspect's dimensions within the scaled extent), not hand-inlined on the GPU path — verified by unit test (IB-007).
6. **general (R4)** The render composes one lazy `CIImage` graph and materializes it with a **single** `createCGImage` call on the **one** app-lifetime `CIContext`; no `CIContext` is constructed inside `render`, and no intermediate stage is separately rendered — verified by code review + on-device attestation (no per-render stutter).
7. **general** The white-balance colour node is a single optional slot between perspective correction and the Lanczos scale; a `nil` `whitePoint` omits the node entirely — verified by code review (the graph is the same recipe with/without the slot; the gains are INT-004's).
8. **general** After render, the code asserts the produced `CGImage` is 1920×1080 (16:9) or 1920×1440 (4:3) **and** carries an sRGB colour space; a mismatch throws `ImagingError.renderFailed` rather than returning a `Slide` — verified on-device by the assertion firing on a deliberately-wrong size in test scaffolding, and by attestation that a correct render passes.

## Failure Behavior

*Observable signal is a Swift `throws` of a typed `ImagingError`; the pure sizing/crop math is unit-test observable; the on-device render behaviors are attested manually per the Intent's Verification (no automatable app surface in this repo).*

| Failure | Detection | Assertion type | Behavior | Recovery |
|---------|-----------|---------------|----------|----------|
| Output dimensions ≠ 1920×1080 / 1920×1440 | post-render size assertion | assert / raise (`ImagingError.renderFailed`) | Throw; construct no `Slide` | AppModel stays in `processing`; user retries / re-edits |
| Output colour space ≠ sRGB | post-render colour-space assertion | assert / raise (`ImagingError.renderFailed`) | Throw; no `Slide` | Stays in `processing` |
| `createCGImage` returns nil / GPU render failure | Core Image render returns nil | raise (`ImagingError.renderFailed`) | Throw from `render` | Stays in `processing`; user retries |
| A `CIContext` built inside `render` | code review | assert (review) | Rejected in review (council R4) | Route through the one app-lifetime context |
| Corners consumed in the wrong coordinate space | code review + INT-002 conversion tests | assert (review/test) | Prevented by using the ClassCamCore Geometry conversions | Fix the boundary, not the warp |

## Open Issues

- *None. The render pipeline is fully specified for INT-003's scope; INT-004 fills the white-balance colour-node slot without changing the graph shape, sizing, or output contract.*

## Amendment Log

| Date | Type | Change | Author |
|------|------|--------|--------|
| 2026-08-10 | Substantive | Initial authoring — render-pipeline + output-sizing behavioral contract (INT-003 IU1/IU2); encodes ADR-008 (one lazy graph, render once through the single `CIContext`, sRGB at 1920 long edge, assert dims + colour). | noreply@anthropic.com |
| 2026-08-10 | Substantive | authored under INT-003 --decompose | jeffhaskin1@gmail.com |
| 2026-08-10 | Substantive | Accepted as INT-003 child spec | noreply@anthropic.com |
