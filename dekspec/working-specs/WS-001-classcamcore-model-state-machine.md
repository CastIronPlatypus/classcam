# Working Spec: ClassCamCore model & app-shell Stage state machine

## Status

ACCEPTED

## Created

2026-08-09

## Modified

2026-08-10

## Silent Failure Domain(s)

*The five domains below are the DekSpec host-project (Dektora) domains — none applies to ClassCam. ClassCam's analogous silent-failure risk for **this** spec is an **illegal application state** (e.g. "processing with no image", "result with no quad") arising with no error signal; this spec's entire purpose is to make those states unrepresentable, encoded in Business Rules BR1–BR9 and Failure Behavior below.*

- [ ] Transformer internals (position IDs, injection layer, KV cache)
- [ ] Numerical precision (quantization, tiered compression, serialization round-trips)
- [ ] GPU multi-process isolation (device assignment, process crash recovery)
- [ ] Graph consistency (shadow graph / Neo4j flush, phantom nodes)
- [ ] Timeline coherence (topic segmentation, tier assignment, decay, shadow timeline / PostgreSQL)

## Expertise Audit Record

*ClassCam is a native Swift/Apple app; the Dektora expertise roles (ML/Model, Quantization, CUDA, Graph, Embedding Geometer, Pipeline Analyst) are all non-triggered — this spec touches none of injection, tensors, CUDA, graph stores, embeddings, or ML pipeline ordering. AE-001 is classified **Core** (the app's competitive correctness lives here), so the audit is recorded for completeness rather than omitted.*

| Role | Triggered | Trigger rule | Rationale |
|------|-----------|-------------|-----------|
| ML / Model Behavior Expert | No | injection layer, position IDs, KV cache | No model internals; ClassCam runs no LLM. |
| Quantization / Precision Expert | No | tensor dtype / bit depth / precision threshold | No tensors; image pixels handled by Core Image in later specs. |
| CUDA Multi-Process Expert | No | more than one device or process boundary | Single-process on-device app (ADR-003); the only boundaries are Swift actors. |
| Graph / Multi-Store Expert | No | shadow graph / Neo4j / shadow timeline / PostgreSQL | No datastore; fully on-device, no persistence beyond a user default. |
| Embedding Space Geometer | No | similarity / distance / centroid | No embeddings. |
| Pipeline Sequencing Analyst | No | pipeline-stage ordering change | The `Stage` machine is authored here, not reordered from an existing pipeline. |

## Related Architecture Elements

- AE-001: ClassCam App — this spec measures the app's internal state model: the `Stage` state machine and the `Sendable` value types that flow through it.

## Governing ADRs

- ADR-005: Swift 6 strict-concurrency actor-isolated services — all model types here are `Sendable` value types; the `@Observable AppModel` that owns `Stage` is the `@MainActor` state owner.
- ADR-006: ClassCamCore UI-free pure package — every type and function in this spec lives in `ClassCamCore` and imports no UI/capture framework.
- ADR-004: SwiftUI UI layer — the three screens are SwiftUI views that switch on `Stage`.

## Interface Contracts

**Consumed contracts:** IC-001 (`CapturedImage`) — the value type that seeds `ProcessingState`.
**Defined contracts:** none (IC-001 is defined alongside this spec; this spec consumes it as `ProcessingState.image`).

## What This Does

This spec defines ClassCam's **application state model**: a single `Stage` enum that expresses the app's whole flow as one value, the `Sendable` value types that populate it, and the transition rules the `@MainActor @Observable AppModel` enforces. It is the shape the three SwiftUI screens render (`capture` → `processing` → `result`) and the shape every later Intent's data flows through. All of it lives in the UI-free `ClassCamCore` package (except `AppModel` itself, which is the app-target `@MainActor` owner of a `Stage` value).

**Mechanism:** `AppModel` holds one `Stage` value and mutates it only through named transition methods that each require the data the next stage needs, so an invalid state (processing without an image, result without a quad) is unconstructable rather than merely unreached.

The value types defined here: `Stage`, `ProcessingState`, `CapturedImage` (per IC-001), `Quad`, `AspectRatio`, `WhiteBalanceGains`, `WhitePoint`, `Slide`. Their correctness-critical pure logic that this spec owns is **quad ordering** and **quad validation** (the rest — coordinate conversion, colour math, output sizing — is owned by later specs but lives in the same package).

## What This Does NOT Do

- **State model:** does not perform any capture, detection, rendering, or export — it only defines the state those operations read and write. Those behaviors live in WS-002 (capture) and the INT-002/003/004 specs (imaging).
- **State model:** does not persist state across app launches beyond the single remembered `AspectRatio` default (INT-003); there is no session restore.
- **Coordinate/colour math:** does not define the Vision↔UIKit↔Core Image coordinate conversions or the white-balance gain computation — those are later ClassCamCore specs; this spec owns only quad **ordering** and **validation**.

## Interfaces

### Data Interfaces

| Interface | Direction | Type / Shape | Source or Consumer | Guarantees |
|-----------|-----------|--------------|--------------------|-----------|
| `Stage` | in/out | enum `{ capture; processing(ProcessingState); result(Slide) }` | AppModel ↔ SwiftUI views | Exactly one case active; associated value present for `processing`/`result`. |
| `ProcessingState` | in/out | struct `{ image: CapturedImage; quad: Quad; whitePoint: WhitePoint?; aspect: AspectRatio }` | AppModel; imaging Intents | `image` immutable; `quad` always validated; `whitePoint` nil until user taps. |
| `Quad` | in/out | four **normalized** `CGPoint` (0–1), ordered TL/TR/BR/BL | detection (INT-002), correction (INT-003) | Ordered + validated on construction (BR6–BR7); rejects degenerate input. |
| `AspectRatio` | in/out | enum `{ sixteenNine, fourThree }` | processing/result, export | Closed set of two (BR8). |
| `WhiteBalanceGains` | in/out | `(r, g, b)` multipliers | white balance (INT-004) | Per-channel positive multipliers; `g == 1` by convention (INT-004 defines math). |
| `WhitePoint` | in/out | struct `{ tappedPoint: CGPoint; gains: WhiteBalanceGains }` | white balance (INT-004) | `tappedPoint` normalized (0–1). |
| `Slide` | in/out | struct `{ image data/CGImage; aspect: AspectRatio }` | result stage, export (INT-003) | Finished 1920-long-edge image (INT-003 owns sizing). |

### Process Interfaces

*Omitted — single-process, in-memory only. The only boundaries are Swift actor hops (ADR-005), across which every type above is `Sendable`.*

### Dependencies

| Dependency | Interface | Failure behavior |
|------------|-----------|-----------------|
| CaptureService (WS-002) | produces `CapturedImage` to seed `processing` | If capture fails it throws (IC-001 Error Semantics); AppModel stays in `capture`. |
| ImagingService (INT-002/003/004) | consumes `ProcessingState`, produces `Slide` | Out of scope here; a render failure keeps the app in `processing`. |

## Domain Constraints

| Constraint | Value | Scope | Rationale |
|------------|-------|-------|-----------|
| Sendability | Every model type is a `Sendable` value type | all-IBs | They cross actor boundaries (ADR-005); the compiler enforces race-freedom. |
| Reference types | None in this spec | all-IBs | `AppModel` (app target) is the only reference type touching `Stage`; ClassCamCore holds pure values only (ADR-006). |
| Package imports | ClassCamCore imports no UIKit/AVFoundation/SwiftUI | all-IBs | Keeps the model unit-testable with zero simulator (ADR-006). |
| Coordinate units | `Quad`/`WhitePoint` points are normalized 0–1 | all-IBs | Resolution-independent; conversion to pixels is a later-spec concern. |
| Compute device / dtype | n/a — pure value types, no tensors or device pinning | all-IBs | ClassCam has no tensor/device surface; the Dektora rows do not apply. |

## Governing Formulas

*None. Quad ordering/validation are algorithmic (specified as Business Rules BR6–BR7), not configurable string-expression formulas.*

## Business Rules

1. **general** `Stage` has exactly three cases — `capture`, `processing(ProcessingState)`, `result(Slide)` — and carries its associated value in the latter two; a test enumerates the cases and asserts the associated types.
2. **general** `AppModel` can enter `processing` **only** by supplying a `CapturedImage`; the transition method takes the image as a required parameter, so "processing with no image" does not compile (asserted by the transition method's signature + a state-transition test).
3. **general** `AppModel` can enter `result` **only** by supplying a `Slide` (which carries a validated `Quad` provenance + `AspectRatio`); "result with no quad/aspect" is unconstructable.
4. **general** From `processing` or `result`, **Retake/Start-Over** returns `AppModel` to `capture` and discards the prior `ProcessingState`/`Slide`; a transition test asserts the resulting stage is `capture` and no stale associated value survives.
5. **general** `ProcessingState.whitePoint` begins `nil` on entry to `processing` and is set only by an explicit white-balance action (INT-004); a test asserts the initial value is `nil`.
6. **general** `Quad` construction **normalizes arbitrary corner input to canonical TL→TR→BR→BL order** — reversed or rotated corner inputs yield the same ordered quad; a unit test feeds permuted corners of a known rectangle and asserts identical ordered output.
7. **general** `Quad` construction **rejects a degenerate or self-intersecting quad** (zero/near-zero area, coincident corners, crossed edges, any coordinate outside 0–1) by failing construction (returns `nil` / throws) rather than yielding an invalid quad; unit tests feed each degenerate case and assert construction fails.
8. **general** `AspectRatio` is a closed enum of exactly `{ sixteenNine, fourThree }`; no third value exists (compile-time + test enumeration).
9. **general** `WhiteBalanceGains` normalizes to green (`g == 1`); a test constructing gains asserts the green channel is unity (the gain math itself is INT-004's spec).

## Failure Behavior

*ClassCam has no server/exception-telemetry surface; the observable "raise" signal is a Swift `throws`/failable initializer (compile-and-test observable), and illegal states are prevented at the type level (a non-signal because they cannot be expressed).*

| Failure | Detection | Assertion type | Behavior | Recovery |
|---------|-----------|---------------|----------|----------|
| Attempt to build `Quad` from degenerate/out-of-range corners | Failable initializer returns `nil` (or `throws`) | assert | Construction fails; no invalid `Quad` exists | Caller keeps prior valid quad / falls back to the default inset box (INT-002) |
| Attempt to enter `processing` without a `CapturedImage` | Does not compile (required parameter) | assert (compile-time) | Unrepresentable | n/a — cannot occur |
| Attempt to enter `result` without a `Slide` | Does not compile (required parameter) | assert (compile-time) | Unrepresentable | n/a — cannot occur |
| Stale associated value survives a Retake | State-transition unit test asserts `capture` has no residue | assert | Retake fully resets to `capture` | Test fails the build if residue is observable |

## Open Issues

- *None. The model is fully specified for INT-001's scope; later Intents extend the value types' behavior (colour math, coordinate conversion) without changing these transition rules.*

## Amendment Log

| Date | Type | Change | Author |
|------|------|--------|--------|
| 2026-08-09 | Substantive | Initial authoring — ClassCamCore model + Stage state machine behavioral contract (INT-001 IU1/IU2). | noreply@anthropic.com |
| 2026-08-10 | Substantive | Accepted as INT-001 child spec | noreply@anthropic.com |
