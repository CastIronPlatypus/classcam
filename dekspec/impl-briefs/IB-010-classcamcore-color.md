# Implementation Brief: ClassCamCore Color (linear-light white-balance gain math)

**Spec:** `dekspec/working-specs/WS-007-tap-white-balance.md`
**Intent:** `dekspec/intents/INT-004-tap-white-balance.md`
**Source AEs:** AE-001
**Depends on:** IB-001 (the `WhiteBalanceGains` value type + `ClassCamCore` package)
**Production gate:** none (pure package; unit-tested on the host toolchain with zero simulator)
**Status:** ACCEPTED

## Precedence

Reviewed for conflicts before writing. Resolve residual ambiguity by: (1) Constraints & Decisions (this IB), (2) Domain Constraints (this IB), (3) Quality Checklists. Do not implement from Spec Context. Stop and ask on any unresolved conflict.

## Goal

`ClassCamCore/Color` exports the pure per-channel white-balance gain math — green-normalized gains computed in linear light (`g_R = G/R`, `g_G = 1`, `g_B = G/B`) — with a unit suite that neutralizes a known cast pixel **and** an explicit **gamma-vs-linear divergence test** proving the linear path is correct, all green with **zero simulator**.

## Out of Scope

- `ImagingService.sampleGains` and the white-balance `CIColorMatrix` render node (IB-011) — this IB owns only the pure arithmetic, not Core Image sampling or the render graph.
- The white-balance control UI and non-destructive `ProcessingState` flow (IB-012).
- Any colour-space conversion in Core Image — this IB computes gains from already-linear channel values handed to it; the `extendedLinearSRGB` application lives in IB-011.

## Escalation Protocol

Stop and ask when a decision needs information not in this IB, when a file outside Files to Modify must change, or when a Done When criterion cannot be met without touching out-of-scope work. Do not guess.

## Spec Context

**Do not implement from this section — traceability only.** WS-007 §Governing Formulas (per-channel gain), §Business Rules BR1, §Domain Constraints (gain normalization, colour space). ADR-009 §Decision (linear-light gains). WS-001 BR9 (`WhiteBalanceGains` `g == 1` convention).

## Files to Modify

*Xcode-side paths (built on the Mac). Structure per council plan §4.*

| File | Change |
|------|--------|
| `ClassCamCore/Sources/ClassCamCore/Color/WhiteBalance.swift` | Pure gain math: from a sampled linear-light `(r,g,b)` compute green-normalized `WhiteBalanceGains` (`g_R = G/R`, `g_G = 1`, `g_B = G/B`); guard non-positive channels. |
| `ClassCamCore/Sources/ClassCamCore/Color/GammaConversion.swift` | Pure sRGB⇄linear transfer-function helpers used **only** by the divergence test to demonstrate the two paths diverge (the production correction applies gains in Core Image's `extendedLinearSRGB`, IB-011). |
| `ClassCamCore/Tests/ClassCamCoreTests/WhiteBalanceTests.swift` | Neutralization test (gains applied to a known cast pixel yield the expected neutral) + the **gamma-vs-linear divergence test** (gamma-space computation leaves a measurable residual; linear path does not). |

## Reuse Inventory

| Capability | Location | Use instead of reimplementing |
|------------|----------|-------------------------------|
| `WhiteBalanceGains` (`(r,g,b)`, `g == 1`) | `ClassCamCore` (IB-001) | Construct the existing value type; do not define a new gains struct. |
| `Swift Testing` (`@Test`/`#expect`) | Swift toolchain | Use for the unit suite; don't build a bespoke assertion harness. |
| Standard `pow`/float math | Swift stdlib / Foundation-free | Use stdlib float math for the transfer function; no external colour library. |

## Domain Constraints

| Constraint | Value |
|------------|-------|
| Colour math space | Gains computed from **linear-light** channel values (`g_R=G/R, g_G=1, g_B=G/B`) |
| Gain normalization | Green-normalized (`g == 1`), per `WhiteBalanceGains` convention |
| Package imports | No UIKit / AVFoundation / SwiftUI / Core Image |
| Compute device / dtype | n/a — pure Swift float values |

## Environment Prerequisites

| Prerequisite | Probe command | Required |
|--------------|---------------|----------|
| n/a — no live services; host-toolchain unit tests only | — | no |

## Do Not Touch

| Function/File | Reason |
|---------------|--------|
| `ImagingService` / `ClassCam/Imaging` | Owned by IB-011; this IB is package-only. |
| `ClassCam/Processing` (UI + state flow) | Owned by IB-012. |
| Existing `ClassCamCore/Model` types | Reused as-is; do not restructure. |

## Governing ADRs

| ADR | Title |
|-----|-------|
| ADR-009 | Compute tap-to-white-balance in linear light and store it non-destructively |
| ADR-006 | ClassCamCore UI-free pure package |
| ADR-002 | Apple-native Vision + Core Image |

## Constraints & Decisions

- **Linear light is the hill:** gains are computed from linear-light channel values, not gamma-encoded sRGB. Multiplying gamma-encoded values scales encoded numbers, not light, and leaves a residual cast (ADR-009 / council R2).
- **Green-normalized:** `g_R = G/R`, `g_G = 1`, `g_B = G/B`; the result satisfies the `WhiteBalanceGains` `g == 1` convention (WS-001 BR9).
- **The divergence test is load-bearing:** the suite includes an explicit gamma-vs-linear divergence test that would fail if the correction were computed in gamma space — this is the off-device guard for the app's dominant silent-failure mode (council §7).
- **Guard degenerate channels:** a zero or non-positive sampled channel must not produce a `NaN`/infinite gain; guard it (clamp or reject) so the pure function is total over valid pixel input.
- **Package purity:** `ClassCamCore/Color` imports no UI/capture/Core Image framework; the actual `extendedLinearSRGB` application is IB-011's Core Image concern. This IB's gamma helpers exist only to prove divergence in a test.
- **Green with zero simulator:** the test target runs on the host toolchain; no test imports a platform-visual framework.

## Interface Contracts

- `dekspec/interface-contracts/IC-001-captured-image.md` — the tagged colour space the sample is interpreted in (traceability only; sampling itself is IB-011).

## Quality Checklists

- Swift API design + strict-concurrency cleanliness (no `@unchecked Sendable`); numerical-correctness review of the transfer function and gain formula.

## Test Promotion Criteria

Promotion refs: WS-007 BR1 (neutralization + gamma-vs-linear divergence); Governing Formula (per-channel gain).

## Test Layout

- `ClassCamCore/Tests/ClassCamCoreTests/WhiteBalanceTests.swift` (pure gain math + divergence — host toolchain).

## Done When

- [ ] Gains applied to a known cast pixel neutralize it to the expected neutral within tolerance — verified by unit test (WS-007 BR1).
- [ ] The gamma-vs-linear divergence test shows the gamma-space computation leaves a measurable residual while the linear path does not — verified by unit test (WS-007 BR1, council §7).
- [ ] Non-positive/zero sampled channels produce no `NaN`/infinite gain — verified by unit test.
- [ ] Computed gains satisfy the green-normalized (`g == 1`) convention — verified by unit test (WS-001 BR9).
- [ ] The suite runs green with **no simulator** — verified by running the package tests on the host toolchain.
- [ ] The outcome test landed first (red), implementation made it green, no other test files modified to make it pass — strong-TDD per ADR-029; verified by git-blame.
- [ ] All new tests pass; no pre-existing tests break — verified by test run.

**Golden State Transitions**

| Input | Expected Output | Verified by |
|-------|----------------|-------------|
| linear sample `(R=0.8, G=1.0, B=1.2)` | gains `(g_R=1.25, g_G=1.0, g_B≈0.833)` | unit test |
| gains applied to the sampling pixel | neutral `(R'=G, G'=G, B'=G)` | unit test |
| same cast neutralized in gamma space vs linear space | gamma path residual ≠ 0; linear path residual ≈ 0 | divergence unit test |
| linear sample with `R=0` | guarded (no `NaN`/inf gain) | unit test |

## Open Issues

- *None.*

## Amendment Log

| Date | Type | Change | Author |
|------|------|--------|--------|
| 2026-08-10 | Substantive | Initial authoring — IB-010 ClassCamCore Color: linear-light gain math + gamma-vs-linear divergence test (INT-004 IU1, from WS-007). | noreply@anthropic.com |
| 2026-08-10 | Substantive | Accepted as INT-004 IB | noreply@anthropic.com |
