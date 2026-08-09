# Implementation Brief: App shell (AppModel + three screens + stub services)

**Spec:** `dekspec/working-specs/WS-001-classcamcore-model-state-machine.md`
**Intent:** `dekspec/intents/INT-001-capture-presentation-still.md`
**Source AEs:** AE-001
**Depends on:** IB-001
**Production gate:** none (spec-only repo; built in Xcode on a Mac)
**Status:** ACCEPTED

## Precedence

Reviewed for conflicts before writing. Resolve residual ambiguity by: (1) Constraints & Decisions, (2) Domain Constraints, (3) Quality Checklists. Do not implement from Spec Context. Stop and ask on any unresolved conflict.

## Goal

The `ClassCam` app target launches to a live-navigable **walking skeleton**: an `@MainActor @Observable AppModel` owning a `Stage` value, three SwiftUI screens that switch on `stage`, and **stub** capture/imaging services returning canned values — compiling strict-concurrency-clean and navigating Capture → Processing → Result on fake data.

## Out of Scope

- Real AVFoundation capture (IB-003 replaces the stub `CaptureService`).
- Real Vision/Core Image imaging (later Intents replace the stub `ImagingService`).
- Any correctness math — the shell consumes `ClassCamCore` (IB-001) and reimplements none of it.

## Escalation Protocol

Stop and ask when a decision needs information not in this IB, when a file outside Files to Modify must change, or when a Done When criterion cannot be met without out-of-scope work. Do not guess.

## Spec Context

**Traceability only.** WS-001 §What This Does (AppModel transition mechanism), §Business Rules BR1–BR5 (transition rules), §Interfaces (the `Stage`/`ProcessingState` shapes the shell renders).

## Files to Modify

*Xcode-side paths (built on the Mac). Structure per council plan §4.*

| File | Change |
|------|--------|
| `ClassCam.xcodeproj` (build settings) | Swift 6 language mode, complete strict concurrency, warnings-as-errors; add `ClassCamCore` package dependency. |
| `ClassCam/Info.plist` | Add `NSCameraUsageDescription` (string shown at the permission prompt). |
| `ClassCam/App/ClassCamApp.swift` | `@main` entry; constructs `AppModel`, injects via `@Environment`. |
| `ClassCam/App/AppModel.swift` | `@MainActor @Observable final class AppModel` owning `var stage: Stage`; named transition methods (`beginProcessing(_:)`, `finish(_:)`, `retake()`/`startOver()`) enforcing BR2–BR4. Holds the two services. |
| `ClassCam/App/ContentView.swift` | `switch appModel.stage` → the three screens. |
| `ClassCam/Capture/CaptureScreen.swift` | Placeholder capture screen (a shutter button that calls the stub and transitions). |
| `ClassCam/Processing/ProcessingScreen.swift` | Placeholder processing screen (shows the fake image; Done → `finish`). |
| `ClassCam/Result/ResultScreen.swift` | Placeholder result screen (Share/Start-Over). |
| `ClassCam/Capture/CaptureService.swift` | **Stub** actor conforming to the capture interface, returning a canned `CapturedImage`. |
| `ClassCam/Imaging/ImagingService.swift` | **Stub** actor returning canned quad/gains/`Slide`. |
| `ClassCam/App/AppModelTests.swift` (test target) | State-transition tests with faked services (BR2–BR5). |

## Reuse Inventory

| Capability | Location | Use instead of reimplementing |
|------------|----------|-------------------------------|
| `Stage`, `ProcessingState`, `CapturedImage`, `Quad`, `AspectRatio`, `Slide` | `ClassCamCore` (IB-001) | Do not redeclare the model types in the app target; import them. |
| Observation (`@Observable`) + `@Environment` | SwiftUI / Observation framework | Use for state ownership + injection; no Combine/`ObservableObject` (ADR-005). |
| SwiftUI navigation (`switch` on state) | SwiftUI | No coordinator/router; the `Stage` switch is the navigation. |

## Domain Constraints

| Constraint | Value |
|------------|-------|
| Isolation | `AppModel` + all views are `@MainActor` |
| Reference types | `AppModel` is the only new reference type; services are actors |
| Strict concurrency | Swift 6 complete, warnings-as-errors; no `@unchecked Sendable` |
| Compute device / dtype | n/a |

## Environment Prerequisites

| Prerequisite | Probe command | Required |
|--------------|---------------|----------|
| n/a (walking skeleton uses stubs) | — | no |

## Do Not Touch

| Function/File | Reason |
|---------------|--------|
| `ClassCamCore` model math | Owned by IB-001; consume, don't edit. |
| Real capture APIs | Belong to IB-003. |

## Governing ADRs

| ADR | Title |
|-----|-------|
| ADR-005 | Swift 6 strict-concurrency actor-isolated services |
| ADR-004 | Build the UI in SwiftUI |
| ADR-006 | ClassCamCore UI-free pure package |

## Constraints & Decisions

- **One state owner:** `AppModel` (`@MainActor @Observable`) is the single source of navigation truth via `var stage: Stage`. No per-screen view-model classes, no coordinator, no DI container (ADR-005 rejects them).
- **Transitions are typed:** entering `processing` requires a `CapturedImage`; entering `result` requires a `Slide`. Encode as method parameters so illegal states don't compile (WS-001 BR2–BR3).
- **Retake/Start-Over resets:** returns `stage = .capture`, dropping any `ProcessingState`/`Slide` (WS-001 BR4).
- **Services are actors behind narrow async interfaces, injected as stubs here:** the shell talks to `CaptureService`/`ImagingService` protocols/actors and never touches AVFoundation or Core Image directly. IB-003 swaps the real capture actor in with no shell change.
- **Ephemeral view state is local `@State`:** drag offsets, toggles live in the view; only `stage` lives in `AppModel`.
- **Strict concurrency from commit one:** turn the build settings on now; do not defer.

## Interface Contracts

- `dekspec/interface-contracts/IC-001-captured-image.md` — the `CapturedImage` the stub returns and the real service (IB-003) produces.

## Quality Checklists

- SwiftUI state-ownership hygiene; strict-concurrency cleanliness.

## Test Promotion Criteria

Promotion refs: WS-001 BR2, BR3, BR4, BR5.

## Test Layout

- `ClassCam/App/AppModelTests.swift` (app test target — state-transition tests with faked services)

## Done When

- [ ] The app launches to the Capture screen and navigates Capture → Processing → Result on stub data — verified by manual on-device/simulator run (walking-skeleton milestone).
- [ ] Build settings are Swift 6 language mode + complete strict concurrency + warnings-as-errors, and the app compiles clean — verified by build.
- [ ] `Info.plist` contains `NSCameraUsageDescription` — verified by config check (WS-002 BR4, satisfied here at shell time).
- [ ] `AppModel` enters `processing` only with a `CapturedImage` and `result` only with a `Slide`; Retake resets to `capture` — verified by state-transition unit tests with faked services (WS-001 BR2–BR4).
- [ ] The app target reuses `ClassCamCore` types and redeclares none — verified by code review.
- [ ] All new tests pass; no pre-existing tests break — verified by test run.

**Golden State Transitions**

| Input | Expected Output | Verified by |
|-------|----------------|-------------|
| `AppModel` in `.capture`, `beginProcessing(cannedImage)` | `stage == .processing` with that image | unit test |
| `AppModel` in `.processing`, `finish(cannedSlide)` | `stage == .result` with that slide | unit test |
| `AppModel` in `.result`, `startOver()` | `stage == .capture`, no residual state | unit test |

## Open Issues

- *None.*

## Amendment Log

| Date | Type | Change | Author |
|------|------|--------|--------|
| 2026-08-10 | Substantive | Initial authoring — IB-002 app shell / walking skeleton (INT-001 IU2, from WS-001). | noreply@anthropic.com |
| 2026-08-10 | Substantive | Accepted as INT-001 IB | noreply@anthropic.com |
