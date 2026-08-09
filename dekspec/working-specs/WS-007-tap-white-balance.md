# Working Spec: Tap-to-white-balance — linear-light gains, 1×1 sampling, non-destructive flow

## Status

ACCEPTED

## Created

2026-08-10

## Modified

2026-08-10

## Silent Failure Domain(s)

*The five domains below are the DekSpec host-project (Dektora) domains — none applies to ClassCam. ClassCam's real silent-failure risks for **this** spec are (a) **gamma-space gains** — applying the correction to gamma-encoded sRGB scales encoded values rather than light and leaves a residual cast with no error signal (council R2, the "sampled a pixel and hoped" trap), and (b) **sampling an untagged pixel** — a pixel of unknown colour encoding yields uninterpretable gains. Both are pinned in Business Rules BR1–BR4 and Failure Behavior below.*

- [ ] Transformer internals (position IDs, injection layer, KV cache)
- [ ] Numerical precision (quantization, tiered compression, serialization round-trips)
- [ ] GPU multi-process isolation (device assignment, process crash recovery)
- [ ] Graph consistency (shadow graph / Neo4j flush, phantom nodes)
- [ ] Timeline coherence (topic segmentation, tier assignment, decay, shadow timeline / PostgreSQL)

## Expertise Audit Record

*Native Swift/Core Image colour work; no Dektora role triggers. AE-001 is **Core** (the app's competitive correctness lives here — a wrong colour correction is a silently-wrong product), so the audit is recorded for completeness. The relevant expertise (Core Image colour management, linear-vs-gamma light) is supplied by the ClassCam persona council, not the Dektora role set.*

| Role | Triggered | Trigger rule | Rationale |
|------|-----------|-------------|-----------|
| ML / Model Behavior Expert | No | injection layer, position IDs, KV cache | No model; ClassCam runs no LLM. |
| Quantization / Precision Expert | No | tensor dtype / bit depth / precision threshold | No tensors; colour channels are Core Image float pixels, not a quantized tensor format. |
| CUDA Multi-Process Expert | No | more than one device or process boundary | Single-process on-device app (ADR-003); the only boundary is the `ImagingService` actor hop (ADR-005). |
| Graph / Multi-Store Expert | No | shadow graph / Neo4j / shadow timeline / PostgreSQL | No datastore; the correction lives in in-memory `ProcessingState`. |
| Embedding Space Geometer | No | similarity / distance / centroid | No embeddings. |
| Pipeline Sequencing Analyst | No | pipeline-stage ordering change | The white-balance node fills the slot INT-003 already reserved in the render graph; nothing is reordered. |

## Related Architecture Elements

- AE-001: ClassCam App — this spec measures the white-point correction stage: the linear-light gain math, the 1×1 tagged-colour-space sample, the non-destructive `whitePoint` flow, and the white-balance control on the processing screen.

## Governing ADRs

- ADR-009: Compute tap-to-white-balance in linear light and store it non-destructively — this spec's central decision; the gain math, `extendedLinearSRGB` application, and non-destructive storage all derive from it.
- ADR-002: Apple-native Vision + Core Image — the correction is a Core Image `CIColorMatrix` node rendered through the shared `CIContext`; the 1×1 sample also goes through that context.
- ADR-005: Swift 6 strict-concurrency actor-isolated services — `sampleGains` and `render` are separate `async` calls on the `ImagingService` actor; the tap gesture is `@MainActor`; only value types cross the actor line.
- ADR-006: ClassCamCore UI-free pure package — the per-channel gain math lives in `ClassCamCore/Color`, unit-testable with zero simulator.

## Interface Contracts

**Consumed contracts:** IC-001 (`CapturedImage`) — supplies the tagged colour space the 1×1 sample is interpreted in (council R2). The `WhitePoint`/`WhiteBalanceGains`/`ProcessingState` value types (WS-001) and the `ImagingService.render(_:quad:whitePoint:aspect:)` signature (INT-003) already exist; this spec fills the `whitePoint` slot.
**Defined contracts:** none (no new cross-component boundary; the shared value types and the `ImagingService` interface are already contracted).

## What This Does

This spec defines ClassCam's **tap-to-white-balance correction**: the behaviour by which a user taps a should-be-neutral spot and the whole image re-colours so that spot reads neutral, non-destructively. It owns three cooperating pieces: (1) the **pure per-channel gain math** in `ClassCamCore/Color` — compute gains from a sampled pixel, normalized to green, in linear light; (2) the **`ImagingService` colour work** — `sampleGains(at:in:)` samples the tapped pixel by rendering a 1×1 region through the shared `CIContext` in a tagged colour space, and a white-balance `CIColorMatrix` node applies the gains in `extendedLinearSRGB` and encodes back to sRGB inside the existing render graph; and (3) the **non-destructive processing-screen flow** — arm the control, tap, store `WhitePoint` in `ProcessingState`, recolour; Reset drops it, re-tap overwrites it.

**Mechanism:** on the `@MainActor` the tap yields a normalized point; the app `await`s `imaging.sampleGains(at:in:)` (a separate actor call), stores the resulting `WhitePoint{tappedPoint, gains}` in `ProcessingState`, then `await`s `imaging.render(...)` with that `whitePoint` — sampling and rendering are two calls sharing the one `CIContext`, and no pixel work ever runs on main. The gains are applied in linear light because multiplying gamma-encoded sRGB values does not scale light and leaves a residual cast (ADR-009).

## What This Does NOT Do

- **Colour scope:** does not perform capture, corner detection, perspective correction, output sizing, or export (INT-001/002/003) — it only adds the white-balance node into the render graph INT-003 authored and fills the `whitePoint` slot.
- **Storage:** does not bake the corrected pixels into a stored bitmap — the correction lives as `{ tappedPoint, gains }` in `ProcessingState`; Reset/re-tap are graph rebuilds, not bitmap undos.
- **Sampling:** does not read a framework image pixel of unknown/untagged encoding — the sample is rendered through the shared `CIContext` in a tagged colour space (council R2).
- **Concurrency:** does not create a second `CIContext` and does not run any pixel work on the main actor.

## Interfaces

### Data Interfaces

| Interface | Direction | Type / Shape | Source or Consumer | Guarantees |
|-----------|-----------|--------------|--------------------|-----------|
| `sampleGains(at:in:) async` | out | `(CGPoint, CapturedImage) -> WhiteBalanceGains` | AppModel/processing flow → ImagingService | Samples the tapped pixel via a 1×1 render through the shared `CIContext` in the image's tagged colour space; returns green-normalized gains (BR2, BR3). |
| white-balance gain math | in/out | `(sampledRGB) -> WhiteBalanceGains` | `ClassCamCore/Color` (pure) | `g_R = G/R`, `g_G = 1`, `g_B = G/B` in linear light; unit-testable off-device (BR1). |
| `render(_:quad:whitePoint:aspect:) async throws` | out | `whitePoint: WhitePoint?` slot | ImagingService | When `whitePoint` is non-nil, a diagonal `CIColorMatrix` applies the gains in `extendedLinearSRGB`, then encodes to sRGB (BR4); when nil, the graph omits the node (BR5). |
| `ProcessingState.whitePoint` | in/out | `WhitePoint?` = `{ tappedPoint: CGPoint; gains: WhiteBalanceGains }` | processing flow | nil until a tap; set on tap, overwritten on re-tap, set to nil on Reset (BR6, BR7). |
| tap gesture | in | normalized `CGPoint` (0–1) | processing screen (`@MainActor`) | A tap on the armed control yields a normalized point; no pixels touched on main (BR8). |

### Process Interfaces

*Omitted — single-process, in-memory only. The only boundary is the `ImagingService` actor hop (ADR-005), across which `CGPoint`, `CapturedImage`, `WhitePoint`, `WhiteBalanceGains`, and `Slide` are all `Sendable`.*

### Dependencies

| Dependency | Interface | Failure behavior |
|------------|-----------|-----------------|
| `CapturedImage` colour-space tag (IC-001) | supplies the space the 1×1 sample is interpreted in | An untagged/unknown space makes the sample uninterpretable → the sampling contract requires a tagged space (BR3, council R2). |
| Shared `CIContext` (ImagingService, INT-003) | one app-lifetime context for both sample and render | A second context would violate ADR-005/INT-003; sampling and rendering reuse the one context (BR2, BR4). |
| `ImagingService.render` signature (INT-003) | `whitePoint: WhitePoint?` slot already present | This spec fills the slot; INT-003 authored the empty slot. |

## Domain Constraints

| Constraint | Value | Scope | Rationale |
|------------|-------|-------|-----------|
| Colour space (application) | Gains applied as a diagonal `CIColorMatrix` in `extendedLinearSRGB` (linear light), then encoded to sRGB | IB-010, IB-011 | Multiplying gamma-encoded sRGB scales encoded values, not light — leaves a residual cast (ADR-009, council R2). |
| Sampling space | 1×1 region rendered through the shared `CIContext` in the image's **tagged** colour space | IB-011 | A pixel of unknown encoding yields uninterpretable gains (council R2); never read a framework `UIImage` pixel. |
| Gain normalization | `g_R = G/R`, `g_G = 1`, `g_B = G/B` | IB-010 | Green-normalized per-channel gains neutralize the tapped pixel; matches the `WhiteBalanceGains` `g == 1` convention (WS-001 BR9). |
| Non-destructive storage | `{ tappedPoint, gains }` in `ProcessingState.whitePoint`; no baked bitmap | IB-012 | Reset (`nil`) and re-tap (overwrite) are cheap graph rebuilds (ADR-009). |
| Single `CIContext` | Sample + render share the one app-lifetime context | IB-011 | ADR-005 / INT-003; a per-call context is a performance cliff (council R4). |
| Actor isolation | Tap on `@MainActor`; `sampleGains`/`render` are `async` `ImagingService` calls; no pixel work on main | IB-011, IB-012 | ADR-005; only value types cross the actor line. |
| Compute device / dtype | n/a — Core Image float pixels, no tensors or device pinning | all-IBs | ClassCam has no tensor/device surface; the Dektora device/dtype rows do not apply. |

## Governing Formulas

| Formula | Expression | Variables | Units / Scale | Valid range | Validated by |
|---------|-----------|-----------|---------------|-------------|-------------|
| Per-channel gain | `g_R = G/R`, `g_G = 1`, `g_B = G/B` | `R,G,B` = linear-light channel values of the sampled should-be-neutral pixel | unitless multipliers | `R,G,B > 0` | `ClassCamCore/Color` unit tests (BR1) |

## Business Rules

1. **general** The per-channel gains are computed from the sampled pixel's **linear-light** RGB as `g_R = G/R`, `g_G = 1`, `g_B = G/B` — green-normalized; a `ClassCamCore/Color` unit test asserts the gains applied to a known cast pixel neutralize it to the expected neutral, **plus an explicit gamma-vs-linear divergence test** asserts the gamma-space computation leaves a measurable residual while the linear path does not (council §7). Pure, off-device.
2. **general** `sampleGains(at:in:)` samples the tapped pixel by rendering a **1×1 region through the shared `CIContext`** (not a second context), returning green-normalized `WhiteBalanceGains` — verified by on-device attestation (the tapped spot reads neutral after correction) with the gain arithmetic covered by the BR1 unit test.
3. **general (R2)** The 1×1 sample is interpreted in the image's **tagged** colour space (from `CapturedImage`, IC-001); a pixel of unknown/untagged encoding is never read — verified by code review of the sampling call (tagged-space render) + the IC-001 tag guarantee.
4. **general (R2)** When `whitePoint` is non-nil, `render` applies the gains as a diagonal `CIColorMatrix` in **`extendedLinearSRGB`** (linear light) and encodes back to **sRGB** — verified by on-device attestation (tapped spot reads neutral, no residual cast) + code review that the matrix node sits in the linear-light stage, not gamma sRGB.
5. **general** When `whitePoint` is `nil`, the render graph omits the white-balance node entirely (the image renders with its original colours) — verified by on-device attestation (Reset restores original colours).
6. **general** A tap on the armed white-balance control stores `WhitePoint{tappedPoint, gains}` in `ProcessingState`; a re-tap on a different spot **overwrites** it and re-applies the correction — verified by on-device attestation (re-tap re-colours to the new neutral).
7. **general** **Reset** sets `ProcessingState.whitePoint` to `nil`, dropping the correction; there is no baked bitmap to undo — verified by on-device attestation (Reset restores the original colours instantly).
8. **general** The tap gesture runs on the `@MainActor` and yields a **normalized** `CGPoint` (0–1); `sampleGains` and `render` are separate `await` calls on `ImagingService` sharing the one `CIContext`; no pixel work runs on main — verified by code review of the actor call sites + on-device main-thread-checker cleanliness.

## Failure Behavior

*ClassCam has no server/exception-telemetry surface; the observable signals here are (a) a pure-function unit assertion for the gain math and the gamma-vs-linear divergence, and (b) on-device attestation for the sampling/rendering/flow behaviors, per the Intent's Verification. The dominant failure mode — a residual cast — is a **silently-wrong** output, which the divergence test (off-device) and the tapped-neutral attestation (on-device) exist to catch.*

| Failure | Detection | Assertion type | Behavior | Recovery |
|---------|-----------|---------------|----------|----------|
| Gains applied in gamma-encoded sRGB leave a residual cast | Gamma-vs-linear divergence unit test (off-device) | assert | Test fails the build if the correction is computed/applied in gamma space | Apply the matrix in `extendedLinearSRGB` (BR1, BR4) |
| Tapped pixel sampled in an unknown/untagged colour space | Code review of the sampling call + IC-001 tag guarantee | assert | Gains would be uninterpretable; the sample must render in the tagged space | Sample through the shared `CIContext` in the tagged space (BR3) |
| Correction baked into a stored bitmap (Reset/re-tap become destructive) | Code review of the storage path | assert | The correction must live as `{ tappedPoint, gains }` in state, not pixels | Store `WhitePoint`; Reset = nil, re-tap = overwrite (BR6, BR7) |
| Pixel work runs on the main actor | Code review + on-device main-thread checker | assert | Sampling/rendering must be `ImagingService` `async` calls | Move the work behind the actor hop (BR8) |
| Second `CIContext` created for sampling or rendering | Code review of the imaging path | assert | A per-call context is a performance cliff (council R4) | Reuse the one app-lifetime context (BR2, BR4) |

## Open Issues

- *None. The correction is fully specified for INT-004's scope; it fills the `whitePoint` slot INT-003 reserved and reuses the shared value types without changing them.*

## Amendment Log

| Date | Type | Change | Author |
|------|------|--------|--------|
| 2026-08-10 | Substantive | Initial authoring — tap-to-white-balance behavioral contract: linear-light green-normalized gains, 1×1 tagged-space sampling, white-balance `CIColorMatrix` render node, non-destructive `whitePoint` flow (INT-004 IU1/IU2/IU3; council §3.4/§7, R2). | noreply@anthropic.com |
| 2026-08-10 | Substantive | WS-007 authored: linear-light gains + 1x1 tagged sampling + non-destructive flow | jeffhaskin1@gmail.com |
| 2026-08-10 | Substantive | Accepted as INT-004 child spec | noreply@anthropic.com |
