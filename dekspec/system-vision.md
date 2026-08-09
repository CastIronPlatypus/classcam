# System Vision: ClassCam

ClassCam is a native iPadOS app that turns an angled, colour-cast photo of a classroom presentation — shown on a TV, monitor, or whiteboard — into a clean, straightened, correctly-white-balanced slide image. Someone sitting off to the side points their iPad at the screen, captures a full-resolution still, nudges the auto-detected corners, taps a should-be-white spot to fix the display's blue cast, and copies or shares the flattened slide straight into a note-taking app like GoodNotes or Notability. It replaces a drawer of skewed, blue-tinted screen photos with square, readable slides.

## Status

DRAFT

## Created

2026-08-09

## Modified

2026-08-09

## What This Is

ClassCam is a single-purpose iPadOS capture-and-correct tool. It consumes one full-resolution photo of a presentation surface taken with the iPad's rear camera and produces one rectilinear slide image at 1920 pixels on the long edge (1920×1080 for 16:9, 1920×1440 for 4:3). Its user-observable surface is three screens: a **Capture** screen (live camera preview with a digital-zoom slider and a shutter button), a **Processing** screen (the captured still with an auto-detected four-corner quad the user drags to refine, a tap-to-white-balance control, a 16:9/4:3 aspect toggle, and Retake/Done), and a **Result** screen (the finished slide with Share, Edit, and Start Over). On Done it perspective-corrects to the chosen aspect ratio, applies the white-balance gains, and places the PNG on the clipboard automatically. Everything runs on-device using Apple frameworks — AVFoundation for capture, Vision for corner detection, Core Image for the warp and colour correction.

## Who This Is For

Anyone attending a class, lecture, or presentation who wants an image of a slide — for notes or any other reason — and who keeps those images in an iPad note-taking app (GoodNotes, Notability, and similar). They are typically **not** seated dead-centre in front of the screen, so their raw photos come out trapezoidal and colour-shifted. The single operator is the person holding the iPad; there is no second audience, no administrator, and no other system consuming ClassCam's output beyond the note app the user pastes into.

## Why This Exists

Photographing a slide off a screen from a classroom seat produces an image that is useless as a note: it is keystoned by the viewing angle, and its whites read blue because the iPad's camera white-balances for room light while the display emits its own cooler light. Existing options don't solve this in the moment — the built-in camera gives a raw skewed photo, general document scanners are tuned for paper on a desk (not a glowing screen viewed at an angle) and don't fix the display colour cast, and asking the presenter for the deck isn't always possible. ClassCam exists to make "grab that slide into my notes, straight and clean" a ten-second gesture. A browser-based version was explicitly considered and rejected: mobile Safari cannot deliver a live preview together with a full-resolution still, caps camera capture well below the sensor, and imposes fragile permission, clipboard, and canvas-memory constraints — going native removes every one of those ceilings.

## What Success Looks Like

- A user sitting off to the side of a room can point, capture, adjust corners, fix the colour, and have a clean slide on their clipboard.
- The exported image is rectilinear (no visible keystone) and its whites read white, not blue, after the tap-to-balance step.
- The exported image is 1920 px on its long edge in the selected aspect ratio and pastes cleanly into GoodNotes or Notability.
- Auto corner-detection lands close enough on a typical TV/whiteboard that the user only nudges corners rather than placing all four by hand; when detection finds nothing, a draggable default box is always present.
- The user can re-tap a different white point, re-edit corners, or start over on a fresh photo without losing their place.

## What We Are Not Building

- **Not a web or PWA app** — mobile Safari can't provide a live preview plus full-resolution capture, and its permission/clipboard/canvas limits make the core flow fragile; native iPadOS removes those ceilings.
- **Not a multi-frame image enhancer (in V1)** — burst capture, super-resolution, frame-stacking, and denoising are deferred to a later version so V1 can ship the core capture-and-correct flow first.
- **Not an OCR or text-extraction tool** — ClassCam outputs a picture of the slide, not searchable or editable text; note apps consume the image.
- **Not a cloud service** — no backend, accounts, sync, or upload; the app is fully on-device to stay fast and private.
- **Not a general photo editor** — no filters, drawing, or freeform adjustment beyond the perspective crop and the single tap-to-white-balance correction; scope is deliberately one slide, one straighten, one colour fix.

## Amendment Log

| Date | Type | Change | Author |
|------|------|--------|--------|
| 2026-08-09 | Substantive | Initial authoring — System Vision for ClassCam, the native iPadOS slide capture-and-correct app. | noreply@anthropic.com |
| 2026-08-09 | Substantive | Removed author-assumed content not supplied by the engineer: dropped the invented Operating Principles and made-up time targets, and broadened the audience to anyone at a class/lecture/presentation per the engineer's own framing. | noreply@anthropic.com |
