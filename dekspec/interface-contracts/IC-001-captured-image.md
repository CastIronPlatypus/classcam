# Interface Contract: CapturedImage — the Capture → Imaging boundary value

## Status

ACCEPTED

## Created

2026-08-09

## Modified

2026-08-10

## Version

1.0.0

## Silent Failure Domain(s)

*The five domains below are the DekSpec host-project (Dektora) domains; none applies to ClassCam, a native Swift/Apple image app. ClassCam's own silent-failure risks at **this** boundary — a recycled pixel buffer and an untagged colour space — are captured as first-class rows in Domain Constraints and Error Semantics below.*

- [ ] Transformer internals (position IDs, injection layer, KV cache)
- [ ] Numerical precision (quantization, wave compression, serialization round-trips)
- [ ] GPU multi-process isolation (device assignment, process crash recovery)
- [ ] Graph consistency (shadow graph / Neo4j flush, phantom nodes)
- [ ] Timeline coherence (topic segmentation, tier assignment, decay, shadow timeline / PostgreSQL)

## Governing ADRs

- ADR-005: Swift 6 strict-concurrency actor-isolated services — this value is exactly what crosses the capture → imaging actor boundary, so it must be `Sendable` and self-owned.
- ADR-006: ClassCamCore UI-free pure package — `CapturedImage` is a ClassCamCore value type, importing no capture or UI framework.
- ADR-002: Apple-native Vision + Core Image — the consumer applies orientation and colour management through Core Image, which is why orientation and colour space travel as metadata rather than being baked into pixels.

## Purpose

`CapturedImage` is the single immutable value that the capture stage hands to the imaging stage. It exists as a separate contract — not as prose inside a Working Spec — because it is the seam that lets the two actors be built, tested, and reasoned about independently: the capture side promises exactly these fields with exactly these guarantees, and the imaging side may assume them without knowing anything about AVFoundation. Pinning it here also fixes the two boundary hazards the council flagged (a pixel buffer recycled out from under a later consumer; a colour space the consumer has to guess), turning both into stated, testable guarantees rather than latent bugs.

It is a **thin envelope**: it carries the pixels plus just enough metadata to interpret them correctly, and deliberately does *not* carry an eagerly-decoded full-resolution image.

## Parties

- **CaptureService (producer)** — the AVFoundation capture actor (ADR-005), running the capture session on its private serial queue. Authoritative for the pixels, the capture orientation, the pixel dimensions, and the colour-space tag. Constructs the `CapturedImage` at the shutter and hands it back through its `async` capture call.
- **ImagingService (consumer)** — the Core Image / Vision actor (ADR-005), owning the single Core Image context. Consumes the `CapturedImage` for corner detection (INT-002), perspective correction and export (INT-003), and tap-to-white-balance sampling (INT-004). Treats the value as read-only.

### Provider AE

AE-001: ClassCam App

### Consumer AEs

- AE-001: ClassCam App — the imaging stages (corner detection, correction, white balance, export) of the same Container consume this internal stage boundary.

## Relationship Pattern

**Pattern:** Shared Kernel

**Change impact:** When this interface changes, both the capture actor and the imaging actor adapt together — the value type is a shared kernel owned by neither side and lives in `ClassCamCore`, so a change is a bilateral, coordinated edit to the package both depend on. (It is close to a Published Language in spirit; Shared Kernel is chosen because both parties compile against the same in-process Swift type rather than a serialized wire format.)

## Shared Conventions

- The value is an **immutable Swift value type** conforming to `Sendable`; it is safe to pass across the actor boundary by the language's rules, with no locking.
- **Pixels are carried, not a decoded image.** The envelope holds the pixel data plus metadata; neither side decodes an eager full-resolution bitmap to construct or read the envelope.
- **Metadata, not mutation.** Orientation and colour space describe how to interpret the pixels; the producer does not pre-rotate or re-encode the pixels to "normalize" them — the consumer applies orientation as the first node of its pipeline (per ADR-002 / INT-003).
- All coordinates and dimensions are in **pixel units** at capture resolution (the imaging stage converts to and from other spaces via `ClassCamCore` geometry).

---

## Interface Definition

`CapturedImage` is a `ClassCamCore` value type with the following fields and guarantees. Field names are indicative; the guarantees are the contract.

| Field | Meaning | Producer guarantee |
|-------|---------|--------------------|
| `pixels` (deep-copied pixel buffer) | The full-resolution captured frame's pixel data, owned by this value. | **A deep copy** taken at capture time — backing memory that the AVFoundation capture pool cannot reclaim. Never a live/pooled buffer. Valid for the full lifetime of the value. |
| `orientation` (EXIF orientation tag) | How the pixels must be rotated to display upright. | Sealed at the shutter from the capture connection's rotation; carried as a metadata tag. Pixels are **not** pre-rotated. |
| `pixelWidth`, `pixelHeight` | Pixel dimensions of `pixels`, before orientation is applied. | Match the buffer exactly; describe the native full-resolution frame (see zoom guarantee below). |
| `colorSpace` / pixel-format tag | The colour space and pixel format the pixel values are in. | Always populated with a **known, tagged** colour space (sRGB unless a wide-gamut capture is deliberately configured). Never absent/unknown. |

**Cross-field guarantees:**

1. **Full native resolution regardless of zoom.** Digital zoom (`videoZoomFactor`, INT-001) changes framing only; the delivered pixels are the full active-format still, so `pixelWidth`/`pixelHeight` reflect native resolution, not a zoom-cropped subframe.
2. **Self-owned lifetime.** The value can be held across any number of `async` imaging calls (detect → sample → render) without the pixels being reclaimed or mutated by the capture subsystem.
3. **Interpretable in isolation.** Given only a `CapturedImage`, the imaging side can produce an upright, correctly colour-managed `CIImage` using only `orientation` and `colorSpace` — no back-channel to the capture actor is required.

---

## Domain Constraints

| Constraint | Value | Rationale |
|------------|-------|-----------|
| Pixel-buffer ownership | Deep copy taken at the shutter; value owns its backing memory | AVFoundation stills come from a finite pool; a pooled buffer held across detect→sample→render can be recycled out from under the consumer, producing torn or blank pixels with no error (council risk **R1**). |
| Colour-space tag | Always present and known (sRGB unless deliberately wide-gamut) | Tap-to-white-balance (INT-004) samples a pixel and converts to linear light; without a known source colour space the sample is uninterpretable and the balance is silently wrong (council risk **R2**). |
| Orientation handling | Carried as an EXIF metadata tag; pixels never pre-rotated | Lets the imaging stage apply orientation once as the first pipeline node (INT-003), keeping every downstream coordinate in one consistent frame. |
| Compute device / dtype | n/a — this is a CPU/GPU-agnostic in-process Swift value, not a tensor-on-the-wire boundary | ClassCam has no multi-device tensor transport; the Dektora-style device/dtype rows do not apply. |

## Error Semantics

| Error condition | Producing party | Detection | Behavior | Consumer responsibility |
|-----------------|----------------|-----------|----------|-------------------------|
| Capture produced no usable pixel buffer | CaptureService | Capture callback yields no buffer / an error | The capture `async` call **throws** a typed capture error; **no** `CapturedImage` is constructed | Consumer never receives a half-built value; the app surfaces a capture-failure state (INT-001), not a crash |
| Deep copy of the pixel buffer fails (allocation failure) | CaptureService | Copy returns failure at the shutter | The capture call **throws**; a pooled buffer is **never** substituted to "succeed" | Consumer receives nothing; capture-failure state is surfaced |
| Colour space cannot be determined | CaptureService | No tagged colour space available at capture | The capture call **throws** rather than emitting an untagged value | Consumer may assume `colorSpace` is always valid on any value it receives |

*No "silent degradation" path exists at this boundary: every failure is surfaced as a thrown typed error on the producer side, and a `CapturedImage` that exists is fully valid by construction.*

## Consistency Guarantees

**Holds:**
- A `CapturedImage` the consumer receives is complete and immutable: all four fields are populated and mutually consistent, and the pixels are stable for the value's lifetime.
- The value is `Sendable`; concurrent reads across imaging calls are safe with no external synchronization.

**Does NOT hold:**
- No promise that the pixels are already upright — the consumer **must** apply `orientation` (they are intentionally un-rotated).
- No promise of a decoded `CGImage`/`UIImage` — the consumer builds its own `CIImage` from the envelope; the envelope is not a ready-to-display bitmap.
- No promise about zoom framing beyond "full native resolution" — the visible-preview crop is not encoded in the value.

## Open Issues

- [ ] Whether a deliberately wide-gamut (Display P3) capture path is ever enabled is deferred to the imaging Intents (INT-003/004); until then the tag is sRGB. — **Source:** initial draft — **Severity:** `P3`

## Amendment Log

| Date | Type | Change | Author |
|------|------|--------|--------|
| 2026-08-09 | Substantive | Initial authoring — CapturedImage capture→imaging boundary value; pins deep-copy (R1) and colour-space-tag (R2) guarantees. INT-001 decomposition. | noreply@anthropic.com |
| 2026-08-10 | Substantive | Accepted as INT-001 child spec | noreply@anthropic.com |
