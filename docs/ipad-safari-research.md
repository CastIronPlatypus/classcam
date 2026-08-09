# iPad Safari Quirks — Research Findings

Reference notes for building the presentation perspective-corrector web app. Everything
below is WebKit-wide, so it also applies to Chrome/Firefox/Edge on iOS (all forced onto
WebKit).

---

## 1. Camera access (getUserMedia)

### HTTPS / secure context
- `getUserMedia` **requires a secure context**. On an insecure origin `navigator.mediaDevices`
  is `undefined` — no error, the whole object is just missing. Feature-detect it.
- Secure origins: `https://`, `http://localhost`, `127.0.0.1`, `file:///`.
- **LAN trap:** hitting a dev box from a physical iPad over `http://192.168.x.x` is **not**
  secure → camera fails silently. You must serve **trusted HTTPS** to the iPad.
- Self-signed certs work **only if the CA is installed + trusted** on the iPad (install profile,
  then enable full trust). "Visit anyway" on the warning is **not** enough. `mkcert` + its root
  CA on the iPad is the common path.

### User gesture & autoplay
- Start the camera **from a button tap** (more reliable prompts). Note iOS does **not** count
  accepting the permission dialog as a user gesture.
- The preview element needs `<video autoplay playsinline muted>` — set these as attributes
  **and** in JS if created dynamically.
  - `playsinline` — without it iPhone hijacks to fullscreen; include always.
  - `muted` — keeps you on the gesture-free autoplay path.
- Request `{ video: {...}, audio: false }` to stay on the autoplay-allowed path.
- WebKit pauses autoplay video that's scrolled off-screen or `display:none` — keep preview
  visible.

### Permissions / PWA
- Safari camera permission is effectively **per-session / transient** — expect re-prompts;
  don't assume persistence.
- **Avoid standalone home-screen PWA mode for camera** — it's been repeatedly broken
  (18.0 broke it, 18.1.1 fixed it) and permission doesn't persist. Run in a **Safari tab**.
- `SFSafariViewController` can't do camera. `WKWebView` only since iOS 14.3 and needs
  native entitlements.

### Rear camera / resolution
- Use `facingMode: { ideal: 'environment' }` — `exact` can hard-fail (OverconstrainedError).
- `enumerateDevices()` returns blank labels/IDs until a stream is granted; deviceIds rotate
  each page load — match on `label` if you persist a lens choice.
- **getUserMedia resolution ceiling ≈ 1080p** in practice (sometimes 720p on older iPads,
  up to 4K on some newer hardware). Never assume — read back
  `track.getSettings()` / `video.videoWidth`.

---

## 2. Capture pipeline & image quality

- **Two paths, big quality gap:**
  - **getUserMedia frame-grab** (draw `<video>` → canvas): only the video track's resolution,
    ~0.9MP (720p) to at best ~8MP (4K on some devices), with video-grade processing.
    **Gives us the live preview + zoom slider + corner overlay.**
  - **`<input type="file" accept="image/*" capture="environment">`**: launches the native
    camera, returns a **full-res still (~8–12MP)**, Apple's computational photography applied.
    **No in-page preview/overlay** and an extra "Use Photo" tap.
- **ImageCapture API (`takePhoto`/`grabFrame`) is NOT supported on iOS Safari through 2026.**
  Feature-detect and fall back.
- **HEIC:** files read in-browser from the file input are frequently `image/heic`, which Safari
  **can't decode to canvas**. Sniff the `ftyp` box (bytes 4–12: `heic`/`heix`/`mif1`/`heim`);
  convert with `heic2any` or `libheif-js`. **Conversion drops EXIF.**
- **EXIF orientation:** iOS stores rotation in EXIF. Use
  `createImageBitmap(blob, { imageOrientation: 'from-image' })` to bake correct orientation
  before drawing — otherwise corner coordinates won't match displayed pixels. If converting
  HEIC (strips EXIF), read the orientation value **first**.

### Design tension for THIS app
We want a **live preview with a digital-zoom slider and draggable corner overlay** → that
mandates the **getUserMedia path** (~1080p). The full-res file-input path can't host the live
overlay. Options: (a) getUserMedia only, accept 1080p; (b) offer both — live-aim preview plus a
"high-res photo" button using file input; correct corners on the returned still.

---

## 3. Canvas, clipboard, warp

### Canvas memory (silent failure)
- Per-canvas area cap: `w×h ≤ 16,777,216` (4096²) on iOS ≤17; raised to 8192² on iOS 18 —
  **treat 16.7M px as the safe ceiling.**
- Total canvas-memory budget across all live canvases ≈ **224–384 MB** (device-dependent);
  each pixel = 4 bytes, so one 4096² canvas = 64 MB.
- **On overflow Safari doesn't throw — it silently draws blank/transparent canvases** and
  `getContext` may return null. Mitigate: **downscale working images to ~1500–2000px long
  edge**; release intermediates by setting `canvas.width = canvas.height = 1`.

### Clipboard image copy (the async gotcha)
- `navigator.clipboard.write()` must run inside a user gesture, and **Safari invalidates the
  gesture across `await`**. Construct the `ClipboardItem` **synchronously** with a **Promise**
  value — don't await the blob first:
  ```js
  button.addEventListener('click', () => {
    const item = new ClipboardItem({
      'image/png': new Promise(resolve => canvas.toBlob(resolve, 'image/png'))
    });
    navigator.clipboard.write([item]);
  });
  ```
- WebKit clipboard image support is **PNG only** — not JPEG.

### Perspective warp / corner detection
- **OpenCV.js:** works (WASM fine on iPad) but ~6MB download and **every `cv.Mat` must be
  `.delete()`'d** or the tab OOM-crashes; `warpPerspective` on huge mats can bus-error.
  Downscale hard, free Mats in finally blocks, lazy-load.
- **jscanify** (MIT, wraps OpenCV.js 4.7.0): `findPaperContour` (blur→Canny→Otsu→findContours,
  largest area) + `getCornerPoints` (quadrant/max-distance-from-center heuristic) +
  `extractPaper` (getPerspectiveTransform + warpPerspective). Fastest path to auto-detect.
- **Lightweight alt (no WASM):** since we let the user drag 4 corners anyway, we can skip
  OpenCV for the warp — use `perspective-transform` (tiny pure-JS, 4 pts → 3×3 homography)
  and warp on canvas (per-pixel, or split quad into 2 triangles with `setTransform`).
  Reserve OpenCV/jscanify only for the **initial auto-guess** of corners.

### Performance
- Downscale to ~1500px before any CV/canvas work; throttle any live detection to ~10fps;
  free memory aggressively; lazy-load OpenCV; consider Worker + OffscreenCanvas (spotty on
  older iOS — feature-detect).

---

## Three things that will bite us if ignored
1. Silent **blank canvas** on memory overflow → downscale + release canvases.
2. Clipboard **`await` kills the gesture** → synchronous `ClipboardItem` with a Promise, PNG only.
3. Un-freed **OpenCV Mats** crash the tab → `.delete()` everything, or avoid OpenCV for the warp.
