# ADR-006: Isolate correctness-critical pure logic in a UI-free ClassCamCore Swift package

## Status

ACCEPTED

## Supersession

*Supersedes:* none
*Superseded by:* none

## Related Architecture Elements

- AE-001: ClassCam App — separates the app's correctness-critical pure logic (geometry, colour math, output sizing, the state-machine value types) from its UI, capture, and imaging shells.

## Created

2026-08-09

## Modified

2026-08-10

## Date

2026-08-09

## Deciders

Jeff Haskin; ClassCam Swift persona council — unanimous consensus.

## Context and Decision Drivers

ClassCam's most dangerous failures are not crashes — they are subtly wrong images that ship without any error signal: a quad warped from the wrong coordinate space, a white balance computed in the wrong colour space, an export off by a scale factor. This correctness-critical logic (coordinate conversions and their origin flips, quad ordering and validation, per-channel white-balance gains, 1920-long-edge output sizing, and the value types the state machine is built from) is pure — it is arithmetic and data, not camera or GPU I/O. Pure logic can be exhaustively unit-tested, but only if it is reachable without a running app, a camera, or a simulator. Bundling it inside the app target chains its tests to a device/simulator boot and buries the arithmetic among UIKit and AVFoundation imports.

**Decision drivers:**
- The app's highest-risk failure mode is silent wrong output from pure math (coordinates, colour, sizing).
- Pure logic is fully testable off-device — but only if it does not import UI or capture frameworks.
- The build environment for specification and CI review is not always an Apple machine with a simulator; logic that tests with zero simulator is verifiable anywhere.
- Keeping the math in one import-free place stops the actors and views from re-implementing it inconsistently.

*Technical story:* Persona-council synthesis, `dekspec/historical-artifacts/classcam-swift-council-plan.md` §4, §7.

## Decision

ClassCam's correctness-critical pure logic lives in a separate **`ClassCamCore` Swift package that imports no UI, capture, or app framework** (no SwiftUI, no UIKit, no AVFoundation) — depending only on the low-level geometry and graphics value primitives it needs. It owns the coordinate-space conversions, quad ordering and validation, the white-balance gain math, the output-sizing and crop-rect math, and the `Sendable` value types plus the `Stage` state machine (per ADR-005). The app target depends on `ClassCamCore`; the service actors and views **call these functions and never re-implement the math**. Because the package imports nothing platform-visual, it builds and unit-tests with **zero simulator**, so the correctness-critical arithmetic is verified on any machine, in isolation from capture and rendering.

## Options Considered (if applicable)

### Option A: Separate UI-free ClassCamCore package for all pure logic + model types

**Pros:** the highest-risk math is exhaustively unit-testable with no simulator; one authoritative home for conversions and colour math prevents drift; the boundary is compiler-enforced (the package cannot accidentally import UIKit).
**Cons:** an extra package/module to set up and keep the app target pointed at.

### Option B: Keep all logic inside the single app target, organized by folder

**Pros:** one target, no package wiring.
**Cons:** pure-logic tests require a simulator/device boot; nothing stops the math from taking a UIKit dependency and becoming untestable in isolation; the correctness-critical arithmetic is harder to find and easier to duplicate.

## Consequences

**Positive:**
- The riskiest logic (coordinates, colour, sizing) is unit-tested off-device, deterministically.
- One import-free module is the single source of truth for the math; actors and views reuse it.
- The no-UI-imports rule is structurally enforced by the package boundary.

**Negative:**
- Requires maintaining a separate package target and its dependency edge from the app.
- Value types that must cross actor lines have to satisfy `Sendable` within the package (intended, per ADR-005).

## Validation

**Observable confirmation:**
`ClassCamCore` builds with no UIKit/AVFoundation/SwiftUI import, its unit-test suite runs to green with no simulator, and the app's actors/views reference its functions rather than re-declaring coordinate or colour math.

**Reconsideration triggers:**
A piece of correctness-critical logic genuinely cannot be expressed without a platform framework (forcing it back into the app target), or the package boundary imposes cost disproportionate to the testability it buys.

## Links

- ADR-005: Swift 6 strict-concurrency actor-isolated services — the `Sendable` value types housed here are what cross the actor boundaries.
- IC-001: CapturedImage contract — a ClassCamCore value type defining the capture → imaging boundary.

## Open Issues

- *None currently.*

## Amendment Log

| Date | Type | Change | Author |
|------|------|--------|--------|
| 2026-08-09 | Substantive | Initial authoring — records the UI-free ClassCamCore pure-package decision (INT-001 coverage finding; council plan §4). | noreply@anthropic.com |
| 2026-08-10 | Substantive | Accepted with INT-001 (council-ratified) | noreply@anthropic.com |
