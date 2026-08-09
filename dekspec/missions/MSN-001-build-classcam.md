# Mission MSN-001: Build the ClassCam slide capture-and-correct app

**Mission ID:** MSN-001
**Status:** TODO
**Owner:** Jeff Haskin
**Created:** 2026-08-09
**Modified:** 2026-08-09
**Autonomy ceiling:** manual

*Valid statuses:* `TODO` → `ACTIVE` → `COMPLETING` → `COMPLETE` | any non-terminal stage → `KILLED`

---

## Near-immutable section

### Outcome

Someone attending a class, lecture, or presentation can point their iPad at a presentation shown on a TV, monitor, or whiteboard from an off-centre seat, capture it, adjust the detected corners, correct the display's colour cast by tapping a neutral spot, and obtain a clean, straightened, colour-corrected slide image — in a chosen 16:9 or 4:3 aspect ratio — on the clipboard and share sheet, ready to paste into a note-taking app.

### Mission Verification

```yaml
- name: _legacy_prose_end-to-end-slide-capture
  cmd: echo SKIP_LEGACY_VERIFY
```

*ClassCam is built and run locally in Xcode on a Mac; this specification repository has no executable app surface, so the end-to-end outcome is verified by manual on-device attestation rather than an automatable command here.*

### Out-of-scope

- Server-side or networked processing — ClassCam runs entirely on-device (ADR-003).
- Text extraction / OCR — the app produces an image, not text.
- General photo editing — no filters, drawing, or freeform adjustment beyond the perspective crop and the single white-point correction.
- Web or browser delivery — ClassCam is a native iPadOS app (ADR-001).

### Flag strategy

- **Flag name:** `none`
- **Default state during Mission:** n/a
- **Who flips it on:** n/a
- **Removal plan:** n/a

*ClassCam is a single-user on-device app with no server or cohort rollout; there is no partial user-visible state to guard, so no feature flag is needed.*

### Rollback plan

**Trigger:** ClassCam is a greenfield app with no existing production surface to regress; there is no deployed state to roll back to during the build.

```yaml
- name: _legacy_prose_no-rollback-surface
  cmd: echo SKIP_LEGACY_ROLLBACK
```

### Kill criteria

```yaml
- name: _legacy_prose_owner-declares-abandonment
  cmd: echo SKIP_LEGACY_KILL
```

### First Intent

INT-001 — Capture a presentation still with live preview and digital zoom

### Autonomy ceiling

`manual`

---

## Live section

### Intent queue

| INT | Title | Type | Status | Notes |
|---|---|---|---|---|
| INT-001 | Capture a presentation still with live preview and digital zoom | feature | IMPLEMENTING | first slice: capture screen; decomposed → ADR-005/006, WS-001/002, IC-001, IB-001/002/003, 7 beads |
| INT-002 | Detect and adjust the presentation's four corners | feature | IMPLEMENTING | decomposed → ADR-007, WS-003/004, IB-004/005/006, 5 beads |
| INT-003 | Perspective-correct, crop to aspect ratio, and export | feature | IMPLEMENTING | decomposed → ADR-008, WS-005/006, IB-007/008/009, 5 beads |
| INT-004 | Correct the display colour cast by tapping a neutral spot | feature | IMPLEMENTING | decomposed → ADR-009, WS-007, IB-010/011/012, 5 beads |

### Discovered prerequisites

- none yet

### Burndown

LOCKED: 0 / Estimated total: 4 / IMPLEMENTING: 4 (all decomposed to beads; 22 beads total across the Mission) / Sketches: 0

### Flag transitions

| Date | Action | Effect observed |
|---|---|---|
| — | — | — |

### Notes

- ClassCam is developed locally in Xcode; this repository holds the specification only. Child-Intent Verification predicates that require an executable surface use the manual/legacy-prose sentinel accordingly.

---

## Amendment Log

| Date | Type | Change | Author |
|------|------|--------|--------|
| 2026-08-09 | Activate | Initial authoring — MSN-001 framing the ClassCam build as a Mission with four ordered child-Intent slices. | noreply@anthropic.com |
