# Working Spec: Export — clipboard PNG, share sheet, Result screen, remembered aspect

## Status

ACCEPTED

## Created

2026-08-10

## Modified

2026-08-10

## Silent Failure Domain(s)

*None of the five Dektora domains applies. ClassCam's real silent-failure risks for **this** spec are (a) placing a **wrong-colour-space or non-PNG** payload on the clipboard so it pastes into a note app with a cast or as a broken image, and (b) the **remembered aspect** silently failing to persist so the toggle resets every photo. Both are re-encoded here as Business Rules BR1, BR5 and Failure Behavior. This spec's payload is the sRGB `CGImage` WS-005 already verified; export does not re-render.*

- [ ] Transformer internals (position IDs, injection layer, KV cache)
- [ ] Numerical precision (quantization, tiered compression, serialization round-trips)
- [ ] GPU multi-process isolation (device assignment, process crash recovery)
- [ ] Graph consistency (shadow graph / Neo4j flush, phantom nodes)
- [ ] Timeline coherence (topic segmentation, tier assignment, decay, shadow timeline / PostgreSQL)

## Expertise Audit Record

*Native Swift/SwiftUI + UIKit-interop export; no Dektora role triggers. AE-001 is **Core**, so the audit is recorded for completeness. The relevant expertise (UIKit interop for `UIPasteboard`/`UIActivityViewController`, SwiftUI `@AppStorage`) is supplied by the ClassCam persona council, not the Dektora role set.*

| Role | Triggered | Trigger rule | Rationale |
|------|-----------|-------------|-----------|
| ML / Model Behavior Expert | No | injection / position IDs / KV cache | No model. |
| Quantization / Precision Expert | No | tensor dtype / precision threshold | The payload is a finished sRGB image, PNG-encoded; no tensors or quantization. |
| CUDA Multi-Process Expert | No | multiple devices / process boundaries | Single process; the only boundary is the SwiftUI↔UIKit interop bridge, handled by ADR-004, not CUDA. |
| Graph / Multi-Store Expert | No | shadow graph / Neo4j / timeline stores | The one persisted value (the aspect default) is a single `@AppStorage` key, not a datastore. |
| Embedding Space Geometer | No | similarity / distance | No embeddings. |
| Pipeline Sequencing Analyst | No | pipeline-stage reordering | Export is the terminal stage; nothing reordered. |

## Related Architecture Elements

- AE-001: ClassCam App — this spec measures the export/result stage: getting the finished slide onto the clipboard and into the share sheet, the Result screen's actions, and the remembered aspect toggle.

## Governing ADRs

- ADR-008: one lazy `CIImage` graph rendered once to a verified sRGB slide — this spec exports that slide; it does not re-render.
- ADR-002: Apple-native stack — export uses `UIPasteboard` and `UIActivityViewController` (UIKit), bridged into SwiftUI.
- ADR-004: SwiftUI UI layer — the Result screen is a SwiftUI view; the share sheet is a `UIViewControllerRepresentable` bridge (SwiftUI has no native `UIActivityViewController`).

## Interface Contracts

**Consumed contracts:** none new — this spec consumes the `Slide` (WS-001) produced by WS-005.
**Defined contracts:** none.

## What This Does

This spec defines what happens after `render` produces a `Slide`: on **Done**, the finished sRGB slide is PNG-encoded and placed on `UIPasteboard` **automatically**, the app advances to the **Result** screen, and the user is offered **Share** (a `UIActivityViewController` share sheet), **Edit** (back to `processing` with the same corners/aspect), and **Start Over** (back to `capture`). It also owns the **remembered aspect toggle**: the 16:9/4:3 choice is persisted via `@AppStorage` so the next photo defaults to the last-used aspect. The clipboard and share payloads are the exact PNG encoding of the slide WS-005 already verified as sRGB at 1920 long edge — export **re-encodes to PNG but does not re-render** the image.

**Mechanism:** entering `result(Slide)` triggers an automatic `UIPasteboard.general` PNG set; the Result view offers Share via a SwiftUI-wrapped `UIActivityViewController`, and Edit / Start Over via the `AppModel` transitions (WS-001 BR4). The aspect toggle reads/writes a single `@AppStorage` key so it survives across captures and launches.

## What This Does NOT Do

- **Rendering:** does not perspective-correct, scale, crop, or otherwise touch pixels — it exports the already-rendered `Slide` (WS-005 owns the image).
- **Colour:** does not change the colour space — the slide is already sRGB (ADR-008); export only PNG-encodes it.
- **Persistence beyond the aspect default:** does not save slides, history, or session state — the only persisted value is the single remembered `AspectRatio` (WS-001 explicitly scopes this as the one default).
- **Cloud / accounts:** no upload, no sharing backend — `UIActivityViewController` hands off to the OS share sheet; the app itself stays on-device (ADR-003).

## Interfaces

### Data Interfaces

| Interface | Direction | Type / Shape | Source or Consumer | Guarantees |
|-----------|-----------|--------------|--------------------|-----------|
| clipboard write | out | PNG `Data` of the `Slide` on `UIPasteboard.general` | OS clipboard → note apps | Automatic on entering `result`; sRGB PNG (BR1). |
| share sheet | out | `UIActivityViewController` over the slide PNG | OS share sheet | Presented on Share tap; same PNG payload (BR2). |
| Result actions | in | Share / Edit / Start Over | Result screen → AppModel | Edit → `processing` (same quad/aspect); Start Over → `capture` (BR3–BR4). |
| remembered aspect | in/out | `AspectRatio` via `@AppStorage("classcam.aspect")` | processing toggle ↔ next capture | Last-used aspect defaults the next photo; survives relaunch (BR5). |

### Process Interfaces

*Omitted — single-process, on-device. The only boundaries are the SwiftUI↔UIKit interop bridges (`UIViewControllerRepresentable` for the share sheet; `UIPasteboard` call) and the `@AppStorage`→`UserDefaults` read/write, all intra-process (ADR-004).*

### Dependencies

| Dependency | Interface | Failure behavior |
|------------|-----------|-----------------|
| `Slide` (WS-005) | finished sRGB `CGImage` + `AspectRatio` | Present on entry to `result`; export PNG-encodes it. A PNG-encode failure surfaces a non-fatal "couldn't copy" state, not a crash. |
| `UIPasteboard.general` | UIKit clipboard | Set the PNG automatically; if unavailable, the copy silently no-ops and Share still works. |
| `UIActivityViewController` | UIKit share sheet | Wrapped in a `UIViewControllerRepresentable`; presented on Share. |
| `@AppStorage` / `UserDefaults` | SwiftUI persistence | Single `AspectRatio` key; a read miss defaults to `.sixteenNine` (BR6). |

## Domain Constraints

| Constraint | Value | Scope | Rationale |
|------------|-------|-------|-----------|
| Clipboard payload | sRGB **PNG** of the slide on `UIPasteboard.general`, set **automatically** on Done | all-IBs | The council flow: the finished slide is on the clipboard the instant the user is done, ready to paste into GoodNotes/Notability (§3.3). |
| No re-render | Export encodes the existing verified `Slide`; it does not re-run the `CIImage` graph | all-IBs | The slide is already sRGB at 1920 long edge (WS-005/ADR-008); re-rendering would risk a second, unverified output. |
| Share | `UIActivityViewController` via a `UIViewControllerRepresentable` bridge | all-IBs | SwiftUI has no native share sheet; UIKit interop is the sanctioned bridge (ADR-004). |
| Aspect persistence | One `@AppStorage("classcam.aspect")` key holding the `AspectRatio` | all-IBs | The single remembered default WS-001 scopes; the toggle is a `@Binding` on that value (council §3.3). |
| Default aspect | `.sixteenNine` when no stored value exists | all-IBs | 16:9 by default (INT-003 Desired Outcome). |
| Result actions | Share / Edit / Start Over | all-IBs | Edit re-enters `processing` with the same corners; Start Over resets to `capture` (WS-001 BR4). |
| Compute device / dtype | n/a — UI + clipboard, no tensors or device pinning | all-IBs | The Dektora device/dtype rows do not apply. |

## Governing Formulas

*None. Export, share, and the aspect toggle are UI/persistence behaviors, not configurable formulas; the sizing/colour math lives in WS-005.*

## Business Rules

1. **general** On entering `result(Slide)` (Done), the slide is PNG-encoded and placed on `UIPasteboard.general` **automatically**, without a separate Copy tap; the PNG is sRGB (the slide's colour space) — verified by on-device attestation (paste into a note app yields the slide with no cast).
2. **general** The Result screen offers a **Share** action that presents a `UIActivityViewController` over the same slide PNG (via a `UIViewControllerRepresentable` bridge) — verified by on-device attestation (Share sheet appears with the image).
3. **general** The Result screen offers **Edit** (returns to `processing` with the same placed corners + aspect) and **Start Over** (returns to `capture`, discarding the slide) — verified by on-device attestation + the AppModel transition tests (WS-001 BR4).
4. **general** Export does **not** re-render the image — it PNG-encodes the existing `Slide`'s verified sRGB `CGImage`; no `CIImage` graph runs in this stage — verified by code review.
5. **general** The 16:9/4:3 toggle writes its value to `@AppStorage("classcam.aspect")`; the next capture's `processing` state reads that key so the last-used aspect is the default — verified by on-device attestation (choose 4:3, take a new photo, it defaults to 4:3; persists across relaunch).
6. **general** When no stored aspect exists (first launch), the default is `.sixteenNine` — verified by on-device attestation (fresh install defaults to 16:9).

## Failure Behavior

*Observable signals are on-device UI behaviors attested manually (no automatable app surface in this repo) plus the AppModel transition unit tests (WS-001) for Edit/Start Over. Export is deliberately non-fatal: a copy or share hiccup never crashes the finished-slide screen.*

| Failure | Detection | Assertion type | Behavior | Recovery |
|---------|-----------|---------------|----------|----------|
| PNG encode of the slide fails | encode returns nil | assert (state) | Show a non-fatal "couldn't copy" state; keep the Result screen + Share available | User re-taps Share / re-does Done |
| Clipboard unavailable | `UIPasteboard` set no-ops | assert (state) | Automatic copy silently no-ops; Share still works | User uses Share |
| Aspect default missing on first launch | `@AppStorage` read miss | assert (default) | Fall back to `.sixteenNine` | n/a — expected first-run path |
| Edit / Start Over leaves stale slide | AppModel transition test | assert | Start Over resets to `capture` with no residual `Slide` (WS-001 BR4) | Test fails the build if residue observable |

## Open Issues

- *None. Export/result/persistence is fully specified for INT-003's scope.*

## Amendment Log

| Date | Type | Change | Author |
|------|------|--------|--------|
| 2026-08-10 | Substantive | Initial authoring — export (clipboard PNG + share sheet) + Result screen + remembered aspect toggle behavioral contract (INT-003 IU3). | noreply@anthropic.com |
| 2026-08-10 | Substantive | authored under INT-003 --decompose | jeffhaskin1@gmail.com |
| 2026-08-10 | Substantive | Accepted as INT-003 child spec | noreply@anthropic.com |
