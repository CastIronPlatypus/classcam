# Implementation Brief: Export — clipboard PNG + share sheet + aspect @AppStorage + Result screen

**Spec:** `dekspec/working-specs/WS-006-export-result-aspect-persistence.md`
**Intent:** `dekspec/intents/INT-003-correct-crop-export.md`
**Source AEs:** AE-001
**Depends on:** IB-008 (produces the `Slide`), IB-001 (`AspectRatio`), IB-002 (AppModel transitions + screen placeholders)
**Production gate:** on-device manual attestation (INT-003 Verification — export/result on a real iPad)
**Status:** ACCEPTED

## Precedence

Reviewed for conflicts before writing. Resolve residual ambiguity by: (1) Constraints & Decisions, (2) Domain Constraints, (3) Quality Checklists. Do not implement from Spec Context. Stop and ask on any unresolved conflict.

## Goal

On **Done**, the finished sRGB `Slide` is PNG-encoded and placed on `UIPasteboard.general` **automatically**; the app advances to a **Result** screen offering **Share** (a `UIActivityViewController` via a UIKit-interop bridge), **Edit**, and **Start Over**; and the 16:9/4:3 aspect toggle is remembered across photos and launches via `@AppStorage`.

## Out of Scope

- Rendering, sizing, cropping, colour space (IB-008 / IB-007) — this IB **exports the already-verified `Slide`**; it does not re-render or touch pixels beyond PNG-encoding.
- White balance (INT-004).
- Capture / detection / drag (INT-001/002).

## Escalation Protocol

Stop and ask when a decision needs information not in this IB, when a file outside Files to Modify must change, or when a Done When criterion cannot be met without out-of-scope work. Do not guess. In particular: do **not** re-run the `CIImage` graph in this stage.

## Spec Context

**Traceability only.** WS-006 §What This Does, §Business Rules BR1–BR6, §Domain Constraints, §Failure Behavior. WS-001 BR4 (Edit/Start-Over transitions). ADR-004 (SwiftUI↔UIKit interop bridges).

## Files to Modify

*Xcode-side paths (built on the Mac). Structure per council plan §4.*

| File | Change |
|------|--------|
| `ClassCam/Result/ResultScreen.swift` | Replace the IB-002 placeholder: show the slide; on appear, auto-copy the PNG to the clipboard; offer Share / Edit / Start Over. |
| `ClassCam/Result/ShareSheet.swift` | `UIViewControllerRepresentable` wrapping `UIActivityViewController` over the slide PNG (SwiftUI has no native share sheet). |
| `ClassCam/Result/SlideExport.swift` | PNG-encode the `Slide`'s sRGB `CGImage`; set `UIPasteboard.general`; a non-fatal "couldn't copy" path on encode failure. |
| `ClassCam/Processing/AspectToggle.swift` | The 16:9/4:3 toggle bound to `@AppStorage("classcam.aspect")`; the next capture's `processing` reads that default. |
| `ClassCam/Result/UIKitInterop.swift` | The single UIKit-interop file collecting the `UIPasteboard` call + share-sheet bridge (keeps UIKit interop in one place). |

## Reuse Inventory

| Capability | Location | Use instead of reimplementing |
|------------|----------|-------------------------------|
| `UIPasteboard.general` | UIKit | Set the PNG on the system clipboard; don't invent a copy mechanism. |
| `UIActivityViewController` | UIKit | The system share sheet; wrap in `UIViewControllerRepresentable`, don't build a custom share UI. |
| `UIViewControllerRepresentable` | SwiftUI | The sanctioned SwiftUI↔UIKit bridge (ADR-004); don't reach outside it. |
| `@AppStorage` / `UserDefaults` | SwiftUI | Persist the single `AspectRatio` default; don't hand-roll persistence. |
| `AspectRatio` | `ClassCamCore` (IB-001) | Store/restore the existing enum; don't redefine the aspect set. |
| `AppModel` transitions (Edit/Start Over) | app target (IB-002) | Reuse the existing `processing`/`capture` transitions (WS-001 BR4); don't add new stages. |
| PNG encoding (`CGImageDestination` / `UIImage.pngData`) | ImageIO / UIKit | Encode the existing sRGB `CGImage`; don't re-render via Core Image. |

## Domain Constraints

| Constraint | Value |
|------------|-------|
| Clipboard payload | sRGB PNG of the slide, set automatically on Done |
| No re-render | encode the existing `Slide`; do not re-run the CIImage graph |
| Share | `UIActivityViewController` via `UIViewControllerRepresentable` |
| Aspect persistence | one `@AppStorage("classcam.aspect")` key |
| Default aspect | `.sixteenNine` when no stored value |
| Result actions | Share / Edit / Start Over |
| Compute device / dtype | n/a — UI + clipboard, no tensors |

## Environment Prerequisites

| Prerequisite | Probe command | Required |
|--------------|---------------|----------|
| A real iPad (clipboard/share/persistence attestation; paste into a note app) | manual on-device run | yes |

## Do Not Touch

| Function/File | Reason |
|---------------|--------|
| `ImagingService` / render pipeline | Owned by IB-008; this IB consumes its `Slide`. |
| `ClassCamCore` Output/Geometry | Owned by IB-007 / INT-002. |
| `AppModel` stage enum / transition definitions | Owned by IB-002; reuse, don't restructure. |

## Governing ADRs

| ADR | Title |
|-----|-------|
| ADR-008 | One lazy CIImage graph, render once through the single CIContext, sRGB at 1920 long edge |
| ADR-002 | Apple-native Vision + Core Image (UIKit interop for UIPasteboard/UIActivityViewController) |
| ADR-004 | Build the UI in SwiftUI (SwiftUI↔UIKit interop bridges) |

## Constraints & Decisions

- **Automatic clipboard copy:** entering `result(Slide)` PNG-encodes the slide's sRGB `CGImage` and sets `UIPasteboard.general` **without** a separate Copy tap (WS-006 BR1). The PNG is sRGB (the slide already is).
- **No re-render:** export encodes the existing verified `Slide`; it does **not** run the `CIImage` graph again (WS-006 BR4 / ADR-008) — re-rendering would risk a second, unverified output.
- **Share sheet via interop:** Share presents `UIActivityViewController` over the slide PNG through a `UIViewControllerRepresentable` bridge (WS-006 BR2 / ADR-004).
- **Result actions:** Edit returns to `processing` with the same corners/aspect; Start Over returns to `capture`, discarding the slide — reusing the existing `AppModel` transitions (WS-006 BR3 / WS-001 BR4).
- **Remembered aspect:** the toggle is a `@Binding` on `@AppStorage("classcam.aspect")`; the next capture's `processing` reads that key so the last-used aspect is the default; missing key → `.sixteenNine` (WS-006 BR5–BR6).
- **Non-fatal export:** a PNG-encode failure or unavailable clipboard shows a non-fatal "couldn't copy" state and keeps Share available — never a crash (WS-006 Failure Behavior).

## Interface Contracts

- none (consumes the `Slide` value type defined by WS-001, produced by IB-008).

## Quality Checklists

- SwiftUI↔UIKit interop correctness; `@AppStorage` persistence; non-fatal export paths; on-device attestation checklist (paste into GoodNotes/Notability clean).

## Test Promotion Criteria

Promotion refs: WS-006 BR3 (Edit/Start-Over) reuses the WS-001 AppModel transition unit tests; BR1, BR2, BR5, BR6 are on-device attestation (clipboard/share/persistence have no automatable app surface in this repo).

## Test Layout

- Edit/Start-Over transitions are covered by the existing AppModel transition unit tests (WS-001, IB-002).
- On-device attestation checklist for clipboard/share/persistence (INT-003 Verification).

## Done When

- [ ] On Done, the slide PNG is on `UIPasteboard.general` automatically and pastes into a note app with no colour cast (sRGB) — verified by on-device attestation (WS-006 BR1).
- [ ] Share presents a `UIActivityViewController` over the slide PNG — verified by on-device attestation (WS-006 BR2).
- [ ] Edit returns to `processing` (same corners/aspect); Start Over returns to `capture` with no residual slide — verified by AppModel transition tests + on-device attestation (WS-006 BR3 / WS-001 BR4).
- [ ] The aspect toggle persists via `@AppStorage`: choosing 4:3 then taking a new photo defaults to 4:3, and it survives relaunch; a fresh install defaults to 16:9 — verified by on-device attestation (WS-006 BR5–BR6).
- [ ] Export runs no `CIImage` graph (encode-only) — verified by code review (WS-006 BR4).
- [ ] A PNG-encode failure surfaces a non-fatal "couldn't copy" state, not a crash — verified by on-device attestation.
- [ ] All new/affected tests pass; no pre-existing tests break — verified by test run.

**Golden State Transitions**

| Input | Expected Output | Verified by |
|-------|----------------|-------------|
| enter `result(Slide)` | slide PNG on `UIPasteboard.general` automatically | on-device attestation |
| tap Share | `UIActivityViewController` over the slide PNG | on-device attestation |
| tap Start Over | AppModel returns to `capture`, no residual `Slide` | AppModel transition test |
| choose 4:3, new photo | next `processing` defaults to `.fourThree` | on-device attestation |
| fresh install, no stored aspect | default `.sixteenNine` | on-device attestation |

## Open Issues

- *None.*

## Amendment Log

| Date | Type | Change | Author |
|------|------|--------|--------|
| 2026-08-10 | Substantive | Initial authoring — IB-009 export (clipboard PNG + share sheet) + aspect @AppStorage + Result screen (INT-003 IU3, from WS-006). | noreply@anthropic.com |
| 2026-08-10 | Substantive | Accepted as INT-003 IB | noreply@anthropic.com |
