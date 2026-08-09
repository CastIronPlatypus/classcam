# Constitution: ClassCam

This Constitution captures the standing operational commitments for ClassCam — a native iPadOS app that turns an angled photo of a classroom presentation (TV, monitor, or whiteboard) into a clean, straightened, colour-corrected slide image ready to drop into a note-taking app. It applies to every DekSpec artifact authored in this repository and to every agent session that reads the spec graph. This repository is the **specification** for ClassCam; the app itself is built locally in Xcode (see Article 5).

## Status

DRAFT

*Valid statuses:* `TODO` → `DRAFT` → `PROPOSED` → `ACCEPTED` → `LOCKED` | any stage → `DEPRECATED`

## Created

2026-08-09

## Modified

2026-08-09

## Article 1: Project Identity

**Summary:** ClassCam is a native iPadOS app for photographing a presentation shown on a TV, monitor, or whiteboard from an off-centre seat, then perspective-correcting, cropping, and white-balancing it into a clean 16:9 (or 4:3) slide image the user pastes or shares into note-taking apps like GoodNotes and Notability. It serves anyone attending a class, lecture, or presentation who wants an image of a slide. It is a single-purpose capture-and-correct tool, fully on-device, not a general photo editor.

**See Also:** dekspec/system-vision.md

## Article 2: Technology Stack

ClassCam is a **native iPadOS application** written in **Swift** with a **SwiftUI** UI layer. Its capture and image pipeline is built on Apple frameworks only:

- **AVFoundation** — custom camera session: `AVCaptureVideoPreviewLayer` live preview, digital zoom via `videoZoomFactor`, full-resolution stills via `AVCapturePhotoOutput`.
- **Vision** — automatic screen/whiteboard quad detection via `VNDetectRectanglesRequest`.
- **Core Image** — `CIPerspectiveCorrection` for the warp; per-channel gain (`CIColorMatrix` or equivalent) for the tap-to-white-balance correction.
- **UIKit interop** — `UIPasteboard` (clipboard, PNG) and `UIActivityViewController` (share sheet) for output.

Build toolchain: **Xcode on macOS**. No third-party runtime dependencies, no backend, no network calls — the app is fully on-device.

Spec toolchain (this repo): **DekSpec** (currently v0.122.0) governs all specification artifacts.

## Article 3: Quality Standards

Because this repository holds the specification, the standing gate here is spec integrity, not app builds:

- `dekspec doctor` returns CLEAN with **zero `P0` / `P1` findings** before any artifact is considered settled.
- Every authored artifact passes `dekspec validate` for its kind.
- The spec graph is fully linked — no dangling forward references at ACCEPTED or later (informational broken refs are tolerated only while an upstream artifact is still being authored in the same foundational pass).

For the ClassCam app itself (enforced in the local Xcode project, not here): the image pipeline produces a 1920-long-edge output for both aspect ratios.

## Article 4: Architecture Principles

- ADR-001 — ClassCam is a native iPadOS app, not a web app.
- ADR-002 — Detection and correction use Apple-native Vision and Core Image.
- ADR-003 — All processing is on-device; no backend, network, or accounts.
- ADR-004 — The UI is built in SwiftUI.

## Article 5: Development Workflow

Work splits across two surfaces:

- **Specification (this repo):** authored with DekSpec on the branch `claude/presentation-perspective-corrector-1rheho`. All design decisions are captured as DekSpec artifacts (System Vision → Constitution → Architecture Elements → ADRs → Intents → Working Specs → Implementation Briefs) before implementation. Commits are descriptive; the spec graph is relinked (`dekspec relink`) after every substantive authoring pass.
- **Implementation (local):** the ClassCam app is developed, compiled, run, and deployed by the engineer in **Xcode on a Mac**. This environment (Linux, no Apple toolchain) never builds or runs the app; it produces specification only.

## Article 6: Model Configuration

Spec-authoring sessions default to the **highest-capability model tier available** (Opus tier). Lower-capability tiers may be used for mechanical, regenerable tasks (index reconciliation, link relinking, validation runs) but never for authoring or revising load-bearing artifacts (System Vision, Constitution, Architecture Elements, ADRs, Intents).

## Article 7: Boundaries

**Boundary ADRs:**

- ADR-003 — On-device processing; captured images never leave the device.

**Boundary AEs:**

- AE-001 — The ClassCam App names its non-goals (no server-side processing, no OCR, no general photo editing).

**Standing non-goals:**

- **Not a web app.** The project pivoted from a browser implementation to native iPadOS; browser/PWA delivery is out of scope.
- **No OCR or text extraction.** ClassCam produces an image, not searchable text.
- **No cloud, backend, or account system.** Fully on-device; no network dependency.
- **Not a general photo editor.** Scope is limited to capture → perspective-correct → crop → white-balance → export of a presentation surface.

## Article 8: Amendments

| Date       | Type        | Change                                                                 | Author               |
|------------|-------------|------------------------------------------------------------------------|----------------------|
| 2026-08-09 | Substantive | Initial authoring — L0 Constitution bootstrapped for ClassCam (native iPadOS slide-capture app), encoding the settled scope, tech stack, workflow split, and boundaries. | noreply@anthropic.com |
| 2026-08-09 | Substantive | Removed author-assumed content not supplied by the engineer: dropped the invented operating mantra, removed the minimum-iPadOS-version floor and the XCTest/no-warnings gates, and broadened the audience to anyone at a class/lecture/presentation. Confirmed direct with engineer: name ClassCam, SwiftUI, Apple-native Vision + Core Image, on-device requirement, both non-goals retained. | noreply@anthropic.com |
| 2026-08-09 | Substantive | Removed a deferred-feature non-goal per engineer direction to describe only what is being built now, not what is postponed. | noreply@anthropic.com |

## Class Lanes

| intent_type | risk_tier | lane | budget_cap_tokens | budget_cap_dollars | max_attempts_per_attempt | max_attempts_per_bead | promotion_threshold_clean_runs | demotion_threshold_reverts | effective_model_snapshot | effective_corpus_volume |
|---|---|---|---|---|---|---|---|---|---|---|
| feature | low | dark | 50000 | 0.50 | 3 | 5 | 10 | 2 | opus-tier | small-N<100 |
| feature | high | gated | 200000 | 5.00 | 3 | 5 | 25 | 1 | opus-tier | small-N<100 |
