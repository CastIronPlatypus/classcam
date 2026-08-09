# ADR-004: Build the UI in SwiftUI rather than UIKit

## Status

ACCEPTED

## Supersession

*Supersedes:* none
*Superseded by:* none

## Related Architecture Elements

- AE-001: ClassCam App — shapes how the app's capture, processing, and result screens are constructed.

## Created

2026-08-09

## Modified

2026-08-10

## Date

2026-08-09

## Deciders

Jeff Haskin

## Context and Decision Drivers

ClassCam is a native iPadOS app (ADR-001) with three screens — capture, processing, and result — carrying interactive controls such as a zoom slider, draggable corner handles, an aspect toggle, and buttons. The platform offers two UI frameworks for building these screens.

**Decision drivers:**
- The app is a small, screen-driven native app with interactive controls.
- The platform's declarative UI framework fits this shape well.

*Technical story:* none

## Decision

ClassCam's user interface is built in SwiftUI. UIKit is rejected for the app's UI layer in favour of the declarative framework, which suits the app's small, screen-and-control-driven surface.

## Options Considered (if applicable)

### Option A: SwiftUI

Declarative UI framework.

**Pros:** concise, state-driven UI well-suited to the app's screens and controls.
**Cons:** some low-level camera-preview integration is bridged from the underlying imperative framework.

### Option B: UIKit

Imperative UI framework.

**Pros:** mature, direct control of view lifecycle.
**Cons:** more boilerplate for this app's small, state-driven surface.

## Consequences

**Positive:**
- The screens and controls are built with a concise, state-driven UI.

**Negative:**
- Camera-preview integration is bridged from the underlying imperative framework where needed.

## Validation

**Observable confirmation:**
The three screens and their interactive controls are implemented in SwiftUI and behave correctly on device.

**Reconsideration triggers:**
A UI need arises that SwiftUI cannot serve without disproportionate effort.

## Links

- none

## Open Issues

- *None currently.*

## Amendment Log

| Date | Type | Change | Author |
|------|------|--------|--------|
| 2026-08-09 | Substantive | Initial authoring — records the SwiftUI-over-UIKit decision. | noreply@anthropic.com |
| 2026-08-10 | Substantive | Status PROPOSED to ACCEPTED — auto-transition by `dekspec audit linkage --fix` (T-STATUS status-maturity coherence, ADR-020). | dekspec-audit-fix |
