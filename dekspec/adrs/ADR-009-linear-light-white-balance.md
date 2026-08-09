# ADR-009: Compute tap-to-white-balance in linear light and store it non-destructively

## Status

ACCEPTED

## Supersession

*Supersedes:* none
*Superseded by:* none

## Related Architecture Elements

- AE-001: ClassCam App — fixes how the white-point correction stage computes and stores its colour correction: gains in linear light, applied as a diagonal colour matrix, kept in state rather than baked into pixels.

## Created

2026-08-10

## Modified

2026-08-10

## Date

2026-08-10

## Deciders

Jeff Haskin; ClassCam Swift persona council (Swift/SwiftUI App Architect · AVFoundation Capture Engineer · Core Image/Vision Imaging Engineer) — unanimous consensus.

## Context and Decision Drivers

A photograph of a lit presentation display carries a colour cast: the camera balances for room light while the screen emits its own cooler light, so whites read blue even after the slide is straightened (INT-004). ClassCam corrects this by letting the user tap a should-be-neutral spot; the app must then re-colour the whole image so that spot reads neutral. Two decisions inside that flow are load-bearing and, left implicit, silently ship a plausible-but-wrong image.

First, **where in the colour pipeline the correction is computed.** sRGB is a nonlinear (gamma-encoded) curve: multiplying encoded channel values does not scale the actual light, so per-channel gains applied to gamma-encoded sRGB leave a residual cast — the correction looks *almost* right and no error ever fires. The council's imaging seat flags this as the hill it will die on (council plan §3.4, risk R2). Second, **how the correction is stored.** Baking the corrected pixels into a new bitmap makes Reset and re-tap into destructive, lossy, compounding operations; the app instead needs an instant, non-destructive, resettable correction (INT-004 Desired Outcome).

**Decision drivers:**
- Gamma-space channel multiplication is the classic "sampled a pixel and hoped" trap — it produces a wrong-but-believable result with no failure signal.
- The tapped pixel must be sampled in a *known* colour space, or the gains are computed from uninterpretable numbers (council R2; depends on the tagged `CapturedImage` from IC-001).
- The user must be able to re-tap a different spot or Reset to the original colours instantly (INT-004), which rules out a baked bitmap.
- The correction shares the one app-lifetime `CIContext` owned by `ImagingService` (ADR-005 / INT-003); sampling and rendering must both go through it, not a second context.

*Technical story:* Persona-council synthesis, `dekspec/historical-artifacts/classcam-swift-council-plan.md` §3.4 (tap-to-white-balance), §1.2 (locked surface), §6 (risk R2).

## Decision

ClassCam computes the tap-to-white-balance correction **in linear light and stores it non-destructively**.

The tapped should-be-neutral pixel is sampled by rendering a **1×1 region through the shared `CIContext` in a tagged colour space** (never a framework image pixel of unknown encoding). From that sample the app computes **per-channel gains normalized to green** — `g_R = G/R`, `g_G = 1`, `g_B = G/B` — and applies them as a **diagonal colour matrix in `extendedLinearSRGB` (linear light), then encodes back to sRGB**. The correction is applied in linear light specifically because multiplying gamma-encoded sRGB values does not scale the underlying light and leaves a residual cast.

The correction is stored as `{ tappedPoint, gains }` in `ProcessingState.whitePoint` (the `WhitePoint`/`WhiteBalanceGains` value types authored by INT-001). It is **non-destructive**: **Reset** sets `whitePoint` to `nil` and **re-tap** overwrites it, and each is a cheap rebuild of the lazy render graph rather than an undo of a baked bitmap. The pure gain math lives in the UI-free `ClassCamCore` `Color/` module (ADR-006) so it is unit-testable with zero simulator, including an explicit gamma-vs-linear divergence test proving the linear path is the correct one (council plan §7).

Applying the gains in gamma-encoded sRGB, sampling a pixel of unknown/untagged encoding, or baking the corrected pixels into a stored bitmap are all rejected: the first leaves a residual cast, the second computes gains from uninterpretable numbers, and the third makes Reset/re-tap destructive and lossy.

## Options Considered (if applicable)

### Option A: Gains in linear light (`extendedLinearSRGB`) via a diagonal colour matrix, stored non-destructively in state

**Pros:** the gains scale actual light, so a tapped neutral truly reads neutral with no residual cast; sampling through the shared `CIContext` in a tagged space gives interpretable numbers; storing `{ tappedPoint, gains }` makes Reset and re-tap instant, lossless graph rebuilds; the pure math is unit-testable off-device with a gamma-vs-linear divergence proof.
**Cons:** requires an explicit colour-space conversion into linear light and back to sRGB in the render graph; the team must resist the simpler-looking gamma-space multiply.

### Option B: Gains applied directly in gamma-encoded sRGB, correction baked into a new bitmap

**Pros:** fewer colour-space conversions; the multiply looks correct at a glance.
**Cons:** leaves a residual cast because gamma-encoded values are not proportional to light (council R2) — a silent, believable, wrong result; a baked bitmap makes Reset/re-tap destructive, lossy, and compounding; sampling an untagged pixel makes the gains uninterpretable.

## Consequences

**Positive:**
- A tapped should-be-neutral spot reads genuinely neutral — no residual cast — because the correction scales light, not gamma-encoded values.
- Reset and re-tap are instant, lossless graph rebuilds; there is no baked bitmap to undo.
- The correctness-critical gain math is pure and off-device-testable (ADR-006), with a divergence test that would fail the build if the gamma path were used.

**Negative:**
- The render graph must convert into `extendedLinearSRGB` and back to sRGB around the colour matrix — an extra, deliberate colour-management step.
- Correct sampling depends on the upstream `CapturedImage` carrying a known, tagged colour space (IC-001 / council R2); an untagged buffer would break the computation.

## Validation

**Observable confirmation:**
Off-device, the `ClassCamCore` `Color/` unit suite shows the linear-light gains neutralize a known cast pixel to the expected neutral, and the gamma-vs-linear divergence test shows the gamma path leaves a measurable residual while the linear path does not. On device, tapping a should-be-neutral spot makes it read neutral, Reset restores the original colours, and re-tapping a different spot re-applies the correction — all instantly.

**Reconsideration triggers:**
The tapped-neutral result still shows a visible cast after applying linear-light gains (indicating the colour-space handling or sampling is wrong), or the product decides to store a baked corrected bitmap for a reason that outweighs non-destructive Reset/re-tap.

## Links

- ADR-005: Swift 6 strict-concurrency actor-isolated services — `sampleGains` and `render` are separate `async` calls on `ImagingService`, sharing its one `CIContext`; no pixel work runs on main.
- ADR-006: Isolate correctness-critical pure logic in a UI-free ClassCamCore package — the linear-light gain math lives in `ClassCamCore/Color`, unit-tested with zero simulator.
- ADR-002: Apple-native Vision + Core Image — the correction is a Core Image `CIColorMatrix` node rendered through the shared `CIContext`.
- IC-001: CapturedImage contract — supplies the tagged colour space the 1×1 sample is interpreted in (council R2).

## Open Issues

- *None currently.*

## Amendment Log

| Date | Type | Change | Author |
|------|------|--------|--------|
| 2026-08-10 | Substantive | Initial authoring — records the linear-light tap-to-white-balance colour-math + non-destructive-storage decision (INT-004 coverage finding; council plan §3.4, R2). | noreply@anthropic.com |
| 2026-08-10 | Substantive | Accepted with INT-004 (council-ratified linear-light white-balance decision) | noreply@anthropic.com |
