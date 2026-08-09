# ADR-005: Structure the app as one @Observable AppModel plus actor-isolated capture and imaging services, Swift 6 strict-concurrency from commit one

## Status

ACCEPTED

## Supersession

*Supersedes:* none
*Superseded by:* none

## Related Architecture Elements

- AE-001: ClassCam App — fixes the app's internal concurrency structure: one main-actor state owner plus two isolated service actors, with only value types crossing the actor boundaries.

## Created

2026-08-09

## Modified

2026-08-10

## Date

2026-08-09

## Deciders

Jeff Haskin; ClassCam Swift persona council (Swift/SwiftUI App Architect · AVFoundation Capture Engineer · Core Image/Vision Imaging Engineer) — unanimous consensus.

## Context and Decision Drivers

ClassCam is a native iPadOS app (ADR-001) built in SwiftUI (ADR-004) whose work spans three isolation-sensitive worlds: the main thread (UI, gesture math), the AVFoundation capture session (which demands a dedicated serial thread and blocks on `startRunning`), and Core Image / Vision processing (heavy, off-main, GPU-backed). Left unstructured, image work leaks onto the UI thread and janks the app, mutable state is shared across threads without discipline, and data races surface as intermittent, hard-to-reproduce corruption. The Swift language offers a first-class way to make these boundaries compiler-enforced rather than convention-enforced.

**Decision drivers:**
- The app has three genuinely distinct isolation domains (UI / capture session / image pipeline) that must not share mutable state carelessly.
- Data races in a capture-and-correct pipeline manifest as silent, intermittent wrong output — the compiler catching them at build time is far cheaper than debugging them on device.
- The app is small (three screens, one linear flow); it does not need a heavyweight application architecture, and adding one would be pure liability.
- Retrofitting strict concurrency onto an app written loosely is expensive; adopting it from the first commit is nearly free.

*Technical story:* Persona-council synthesis, `dekspec/historical-artifacts/classcam-swift-council-plan.md` §1–2.

## Decision

ClassCam is structured as **one `@MainActor`, `@Observable` application-state owner** holding a single state-machine value, plus **two isolated service actors** — one owning the AVFoundation capture session, one owning the Core Image / Vision pipeline — reached through narrow `async` interfaces. **Only `Sendable` value types cross an actor boundary**; framework reference types (the capture session, the Core Image context, framework image objects) never leave the actor that owns them. The project builds under **Swift 6 language mode with complete strict concurrency and warnings-as-errors from the first commit**. The capture actor additionally owns a private serial queue and marshals every capture-session call onto it, because an actor's executor is a cooperative-pool thread, not the dedicated serial thread the capture session requires.

Heavier application patterns — a view-model class per screen, a coordinator/router, a dependency-injection container, or a reactive-stream framework — are rejected: at three screens they add structure the app does not need, and the Observation framework plus `async`/`await` already covers every state and stream the app has. When the compiler's strict-concurrency checker objects, the boundary is fixed rather than silenced; blanket main-actor annotation or unchecked-`Sendable` escape hatches are not used to quiet it, because doing so would drag image work onto the UI thread or reintroduce the races this decision exists to prevent.

## Options Considered (if applicable)

### Option A: One @Observable main-actor state owner + isolated service actors, strict concurrency on from day one

**Pros:** the three isolation domains become compiler-enforced; data races are build-time errors, not device-only heisenbugs; the structure is the minimum the app actually needs; no retrofit cost later.
**Cons:** requires discipline about what crosses actor lines from the start; the capture actor needs an explicit private serial queue (an actor executor alone is insufficient).

### Option B: Loose MVVM with reference-type view models and ad-hoc threading, strict concurrency deferred

**Pros:** familiar pattern; fewer up-front constraints on where state lives.
**Cons:** for a three-screen app the per-screen view models are ceremony with no payoff; ad-hoc threading invites the exact silent-wrong-output races the app is most vulnerable to; turning on strict concurrency later is a large, disruptive retrofit.

## Consequences

**Positive:**
- The compiler enforces the capture / imaging / UI isolation boundaries; a data race is a build failure.
- The app carries the smallest structure that expresses it cleanly — one state owner, two services.
- Image work provably never runs on the main thread, so the UI stays responsive.

**Negative:**
- Every value that crosses an actor boundary must be `Sendable`, constraining the type design (see ADR-006 and IC-001).
- The capture actor must manage an explicit private serial queue in addition to its actor isolation.

## Validation

**Observable confirmation:**
The project compiles under Swift 6 strict concurrency with warnings-as-errors and no `@unchecked Sendable` escapes on the capture/imaging path; on device, capture and image processing run without blocking the UI, and no data-race instrumentation fires during a capture→process→result run.

**Reconsideration triggers:**
Strict concurrency forces an `@unchecked Sendable` or a main-actor-everywhere workaround on the core path that cannot be resolved by fixing the boundary, indicating the isolation model is wrong for some subsystem; or the app grows enough screens that the single-state-owner model no longer scales.

## Links

- ADR-004: Build the UI in SwiftUI — the UI layer this concurrency model serves.
- ADR-006: Isolate correctness-critical pure logic in a UI-free ClassCamCore package — the `Sendable` value types that cross the actor lines live there.
- IC-001: CapturedImage contract — the first value type to cross the capture → imaging boundary.

## Open Issues

- *None currently.*

## Amendment Log

| Date | Type | Change | Author |
|------|------|--------|--------|
| 2026-08-09 | Substantive | Initial authoring — records the Swift 6 strict-concurrency, one-AppModel-plus-two-service-actors architecture (INT-001 coverage finding; council plan §1–2). | noreply@anthropic.com |
| 2026-08-10 | Substantive | Accepted with INT-001 (council-ratified) | noreply@anthropic.com |
