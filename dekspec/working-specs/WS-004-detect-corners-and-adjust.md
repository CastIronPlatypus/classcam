# Working Spec: Corner detection, default-box fallback, and draggable-handle adjustment

## Status

ACCEPTED

## Created

2026-08-10

## Modified

2026-08-10

## Silent Failure Domain(s)

*The five domains below are the DekSpec host-project (Dektora) domains — none applies to ClassCam. ClassCam's real silent-failure risks for **this** spec are (a) **detection tuned too tight** for a glowing angled screen, so a real presentation shot off-axis returns nothing and the app looks broken (no crash, just a permanent fallback box), and (b) **detection re-running over the user's edits**, silently overwriting a corner the user deliberately dragged. Both are pinned in Business Rules BR1–BR3 and BR7–BR8 and Failure Behavior below. The coordinate-conversion silent-failure (council R3) is owned by WS-003, which this spec consumes.*

- [ ] Transformer internals (position IDs, injection layer, KV cache)
- [ ] Numerical precision (quantization, tiered compression, serialization round-trips)
- [ ] GPU multi-process isolation (device assignment, process crash recovery)
- [ ] Graph consistency (shadow graph / Neo4j flush, phantom nodes)
- [ ] Timeline coherence (topic segmentation, tier assignment, decay, shadow timeline / PostgreSQL)

## Expertise Audit Record

*Native Swift/Vision detection + SwiftUI gesture UI; no Dektora role triggers. AE-001 is **Core**, so the audit is recorded for completeness. The relevant expertise (Vision rectangle detection, SwiftUI drag gestures) is supplied by the ClassCam persona council, not the Dektora role set.*

| Role | Triggered | Trigger rule | Rationale |
|------|-----------|-------------|-----------|
| ML / Model Behavior Expert | No | injection / position IDs / KV cache | `VNDetectRectanglesRequest` is a framework CV request, not an LLM; no model internals are touched or tuned beyond documented request parameters. |
| Quantization / Precision Expert | No | tensor dtype / precision threshold | No tensors; detection confidence is a framework float, not a quantized value. |
| CUDA Multi-Process Expert | No | multiple devices / process boundaries | Single process; detection runs on the imaging actor (ADR-005), an intra-process boundary, not CUDA. |
| Graph / Multi-Store Expert | No | shadow graph / Neo4j / timeline stores | No datastore. |
| Embedding Space Geometer | No | similarity / distance | Rectangle detection is geometric, not embedding-based. |
| Pipeline Sequencing Analyst | No | pipeline-stage reordering | Detection is a new leaf behavior on the processing stage, not a reorder. |

## Related Architecture Elements

- AE-001: ClassCam App — this spec measures the processing stage's corner-detection behavior (`ImagingService.detectQuad`), the mandatory fallback box, and the draggable-handle adjustment overlay.

## Governing ADRs

- ADR-007: Coordinate-space conversion discipline — detection **seeds** the handles and the user's dragged `Quad` is the **source of truth**; detection is never re-run over edits; all point conversions route through the WS-003 `Geometry` functions.
- ADR-002: Apple-native Vision + Core Image — detection is `VNDetectRectanglesRequest` on the imaging actor.
- ADR-005: Swift 6 strict-concurrency actor-isolated services — `detectQuad(in:) async -> Quad?` runs on the `ImagingService` actor (off main); dragging is pure `@MainActor` normalized-point math, no pixels on main.
- ADR-004: SwiftUI UI layer — the overlay and its draggable handles are a SwiftUI view over the still.

## Interface Contracts

**Consumed contracts:** IC-001 (`CapturedImage`) — `detectQuad` takes it as input; and the WS-003 `Geometry` conversions (in-package, consumed to seed handles).
**Defined contracts:** none — `detectQuad` returns the existing `Quad` value type (WS-001); no new cross-component boundary type.

## What This Does

This spec defines how ClassCam finds the presentation's four corners and lets the user adjust them. It has three parts:

1. **Detection** — `ImagingService.detectQuad(in image: CapturedImage) async -> Quad?` runs a `VNDetectRectanglesRequest` **tuned for a glowing angled screen** (a bright TV/whiteboard shot off-axis is not a crisp square document), picks the best observation by confidence, converts its corners into a normalized `Quad` (via the WS-003 conversions + WS-001 ordering), and returns it — or `nil` when nothing clears the (deliberately permissive) bar.
2. **Fallback** — when `detectQuad` returns `nil`, the processing screen presents a **default inset box (~10% inset)** the user positions manually. Detection failing is a normal, expected outcome, not an error.
3. **Adjustment** — the processing screen overlays **four draggable corner handles** on the quad. Detection only **seeds** the initial handle positions; once on screen, the **user's dragged `Quad` is the source of truth**, and **detection is never re-run over the user's edits**. Dragging is pure `@MainActor` UI math on **normalized points** — no pixels are touched on the main thread.

**Mechanism:** on entry to `processing`, the screen awaits `detectQuad`; the returned `Quad` (or the default inset box on `nil`) seeds the overlay's handle positions in normalized space; a SwiftUI drag gesture updates the grabbed corner's normalized point through the WS-003 conversions; the updated `Quad` lives in `ProcessingState` (WS-001) and is what INT-003's warp consumes.

## What This Does NOT Do

- **Coordinate math:** does not define the space conversions — it consumes WS-003's `Geometry` functions (no inline flips).
- **Quad ordering/validation:** does not order or validate a `Quad` — it constructs one through WS-001's failable init (which orders + validates).
- **Warp/export:** does not perspective-correct, crop, colour-adjust, or export (INT-003/004) — it only produces the quad those consume.
- **Re-detection:** does not re-run detection after the user has begun adjusting; there is no "auto-snap" that overrides a dragged corner (ADR-007).
- **Persistence:** does not persist the quad across launches.

## Interfaces

### Data Interfaces

| Interface | Direction | Type / Shape | Source or Consumer | Guarantees |
|-----------|-----------|--------------|--------------------|-----------|
| `detectQuad(in:) async -> Quad?` | out | `CapturedImage → Quad?` | AppModel/processing screen | Returns a normalized, ordered, validated `Quad` (best observation) or `nil` when none clears the bar (BR1–BR5). |
| default inset box | out | `Quad` (~10% inset of the frame) | processing screen when `detectQuad` is `nil` | A valid, ordered `Quad` the user can immediately drag (BR6). |
| corner drag | in | grabbed-corner index + normalized `CGPoint` delta | SwiftUI gesture → `ProcessingState.quad` | Updates one corner in normalized space via WS-003; `@MainActor`, no pixels (BR7). |
| adjusted `Quad` | out | `Quad` in `ProcessingState` | INT-003 warp | The user's dragged quad is authoritative; detection never overrides it (BR8). |

### Process Interfaces

| Boundary | Transport | Device | Serialization | Failure mode |
|----------|-----------|--------|---------------|-------------|
| processing screen → `ImagingService` | `await detectQuad(in:)` (actor hop) | imaging actor (off main) | `CapturedImage` in, `Quad?` out (both `Sendable`) | Detection running on main would jank; ADR-005 keeps it on the actor. `Task.isCancelled` checked around `detectQuad` so leaving processing cancels it. |
| gesture → `ProcessingState` | in-memory `@MainActor` mutation | main | normalized `CGPoint` | A pixel conversion on main would violate ADR-005; only normalized-point math runs on main (BR7). |

### Dependencies

| Dependency | Interface | Failure behavior |
|------------|-----------|-----------------|
| `VNDetectRectanglesRequest` (Vision) | rectangle observations with confidence + corners | Returns zero observations for an unclear/too-angled frame → `detectQuad` returns `nil` → fallback box (BR5, BR6). |
| WS-003 `Geometry` conversions | normalized-BL ↔ pixel-BL ↔ points-TL | If a conversion were wrong the quad would be plausibly-wrong (council R3) — mitigated by WS-003's round-trip tests, not re-derived here. |
| `Quad` failable init (WS-001) | orders + validates the four corners | A degenerate detection result fails `Quad` construction → treated as no detection → fallback box (BR4). |
| `CapturedImage` (IC-001) | the still + pixel dims + orientation | Supplies the image detection runs on and the dims WS-003 scales against. |

## Domain Constraints

| Constraint | Value | Scope | Rationale |
|------------|-------|-------|-----------|
| Detection actor | `detectQuad` runs on `ImagingService` (off main), returns `Sendable` `Quad?` | IB-005 | Vision work is heavy; it must not run on the UI thread (ADR-005). |
| `quadratureTolerance` | Generous, ~30–45° | IB-005 | Angled shots are **not** square; a tight tolerance means an off-axis presentation never detects — the exact failure that makes the app look broken. |
| `minimumConfidence` | Low, ~0.3 | IB-005 | A bright TV off-axis is not a crisp document; demanding high confidence yields constant fallbacks. |
| `minimumAspectRatio` | Permissive, ~0.3 | IB-005 | Presentations viewed at an angle are far from their nominal aspect; a strict ratio rejects them. |
| `maximumObservations` | A small handful | IB-005 | Consider a few candidates, then pick the best by confidence (BR2); not one-and-done, not unbounded. |
| Selection | Pick the single best observation by confidence | IB-005 | One quad seeds the handles; ties/ambiguity resolve to highest confidence. |
| Fallback inset | Default box ~10% inset of the frame | IB-006 | Gives the user an immediately-draggable starting quad when detection returns nothing — mandatory, never a dead screen. |
| Seed-not-authority | Detection seeds handles once; the dragged `Quad` is the source of truth; detection is never re-run over edits | IB-006 | ADR-007 — the user's deliberate correction must never be overwritten by a fresh guess. |
| Drag isolation | Dragging updates normalized points on `@MainActor`; no pixel conversion on main | IB-006 | 60 fps corner math stays cheap and on the UI thread; pixels are touched only by the imaging actor at warp time (ADR-005/007). |
| Cancellation | `detectQuad` is called from the view's `.task {}`; `Task.isCancelled` checked | IB-005/006 | Leaving processing cancels an in-flight detection; no detached `Task` outlives the view (ADR-005 / council §2). |
| Compute device / dtype | n/a — camera stills + Vision observations, not tensors; no device pinning | all-IBs | The Dektora device/dtype rows do not apply. |

## Governing Formulas

| Formula | Expression | Variables | Units / Scale | Valid range | Validated by |
|---------|-----------|-----------|---------------|-------------|-------------|
| Default inset box | corners at `(i, i), (1−i, i), (1−i, 1−i), (i, 1−i)` with `i ≈ 0.10` | `i` inset fraction | normalized (0–1) | `i ∈ (0, 0.5)` | IB-006 (produces a valid orderable `Quad`) |
| Best observation | `argmax_o confidence(o)` over returned observations | `o` observation | confidence (0–1) | — | IB-005 (selection) |

## Business Rules

1. **general** `detectQuad(in:)` runs a `VNDetectRectanglesRequest` on the imaging actor with a **generous `quadratureTolerance` (~30–45°)**, **low `minimumConfidence` (~0.3)**, **permissive `minimumAspectRatio` (~0.3)**, and a small `maximumObservations` — verified on-device by attesting detection succeeds on a real glowing TV/whiteboard viewed off-axis (council §7).
2. **general** When multiple observations return, `detectQuad` selects the single **highest-confidence** one — verified by code review of the selection + on-device attestation that the seeded quad tracks the actual presentation.
3. **general** The selected observation's corners are converted to a normalized `Quad` **through the WS-003 `Geometry` conversions** (no inline flip) and constructed via the WS-001 failable init (ordered + validated) — verified by code review that the conversion routes through `ClassCamCore/Geometry`.
4. **general** If the selected observation's corners fail `Quad` construction (degenerate/out-of-range), `detectQuad` treats it as **no detection** and returns `nil` rather than a bad quad — verified by the WS-001 `Quad` init contract + on-device attestation (a marginal frame falls back cleanly).
5. **general** When no observation clears the bar, `detectQuad` returns `nil` (not an error, not a throw) — verified on-device by attesting an unclear frame produces the fallback box, not a crash or hang.
6. **general (fallback)** On `detectQuad == nil`, the processing screen presents a **default inset box (~10% inset)** as a valid, immediately-draggable `Quad` — verified on-device by attesting a no-detection frame shows a positionable box, never a dead screen.
7. **general (drag)** Dragging a corner handle updates that corner's **normalized point on `@MainActor`** through the WS-003 conversions; **no pixel conversion runs on the main thread** — verified by code review (main-thread path touches only normalized math) + on-device attestation (smooth 60 fps drag).
8. **general (seed-not-authority)** Detection **seeds** the handle positions exactly once on entry to processing; thereafter the **user's dragged `Quad` is the source of truth** and **detection is never re-run over the user's edits** — verified on-device by attesting that a dragged corner is never snapped back by a re-detection (council §3.2, ADR-007).
9. **general** `detectQuad` is invoked from the processing view's `.task {}` and honours cancellation (`Task.isCancelled`), so leaving processing cancels an in-flight detection and no detached `Task` outlives the view — verified by code review + on-device attestation (rapid enter/leave does not leak work).

## Failure Behavior

*Observable signal for the pure/selection logic is code review + on-device attestation; ClassCam has no server/exception-telemetry surface, and detection returning `nil` is a **normal expected outcome** (the fallback), not a raised failure. On-device behaviors are attested manually per the Intent's Verification (no automatable surface in this repo).*

| Failure | Detection | Assertion type | Behavior | Recovery |
|---------|-----------|---------------|----------|----------|
| No rectangle clears the bar | Zero qualifying observations | assert (state) | `detectQuad` returns `nil`; screen shows the default inset box | User positions the box manually (BR6) |
| Detected corners degenerate | `Quad` failable init returns `nil` | assert | Treated as no detection → fallback box | User positions the box (BR4) |
| Detection tuned too tight (regression) | On-device: a real off-axis presentation returns nothing | attestation | App looks broken (permanent fallback) — caught by the on-device tuning checklist | Loosen `quadratureTolerance`/`minimumConfidence` per BR1 |
| Re-detection overwrites a dragged corner (regression) | On-device: dragged corner snaps back | attestation | Violates ADR-007 seed-not-authority — must not ship | Remove any re-detection over edits (BR8) |
| Pixel conversion on main thread (regression) | Code review / main-thread checker | assert | Violates ADR-005 — janks drag | Route pixel work to the imaging actor (BR7) |

## Open Issues

- [ ] The exact `VNDetectRectanglesRequest` parameter values (`quadratureTolerance`, `minimumConfidence`, `minimumAspectRatio`, `maximumObservations`) are seeded from the council ranges (§3.2) and finalized during on-device tuning against real glowing-screen shots. — **Source:** initial draft — **Severity:** `P3`
- [ ] The default fallback inset fraction (~10%) is a starting value to be confirmed during on-device tuning. — **Source:** initial draft — **Severity:** `P3`

## Amendment Log

| Date | Type | Change | Author |
|------|------|--------|--------|
| 2026-08-10 | Substantive | Initial authoring — corner detection (`VNDetectRectanglesRequest` tuning) + mandatory default-inset-box fallback + draggable-handle adjustment with seed-not-authority (INT-002 IU2/IU3; council §3.2, ADR-007). | noreply@anthropic.com |
| 2026-08-10 | Substantive | Accepted as INT-002 child spec. | noreply@anthropic.com |
