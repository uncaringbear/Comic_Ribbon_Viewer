# Seamless Comic & Image Ribbon Viewer — Architecture & Functionality Spec

## 1. Overview & Purpose
The **Seamless Comic & Image Ribbon Viewer** (`comic-ribbon-viewer-v4.html`) is a single-file, zero-dependency browser application designed to batch-load local image files and render them as a continuous, single-row horizontal strip. It is specifically optimized for reading `3:4` portrait comic and manga pages at full screen height (`100vh`) on `16:9` 4K displays, featuring real-time GPU upscaling and sharpening.

---

## 2. Core Architecture

### 2.1 Self-Contained Single-File Design
* **Zero External Dependencies:** HTML, CSS, JavaScript, and GLSL shaders reside in one `.html` file that runs locally via `file://` without web servers, build steps, or plugins.
* **Two-State UI:**
  1. **Setup / Configuration Overlay (`#drop-zone`):** Handles drag-and-drop or file picker ingestion, configuration controls, and `localStorage` persistence.
  2. **Fullscreen WebGL Viewport (`#gl-canvas`):** Renders the interactive image strip across `100vw × 100vh` with a minimal auto-hiding HUD (`#hud`) and temporary status notifications (`#toast`).

### 2.2 File Ingestion & Texture Pipeline
1. **Batch Loading:** Accepts multiple files via drag-and-drop or `<input type="file" multiple accept="image/*">`.
2. **Ordering:**
   * **Default:** Natural alphanumeric sorting via `Intl.Collator(undefined, { numeric: true, sensitivity: 'base' })` so `page_2.jpg` precedes `page_10.jpg`.
   * **Manga Mode:** Reverses the sorted array so Page 1 starts at the far right edge of the strip.
   * **Randomize Mode:** Applies an in-place Fisher-Yates shuffle.
3. **GPU Texture Upload:**
   * Decodes images asynchronously using `createImageBitmap(file)` (with `HTMLImageElement` fallback).
   * Uploads each image to a WebGL2 `TEXTURE_2D` (`RGBA8`, `CLAMP_TO_EDGE`, `LINEAR` filtering, `UNPACK_PREMULTIPLY_ALPHA_WEBGL = false`).
   * Immediately calls `bitmap.close()` after upload to keep system RAM usage minimal even with hundreds of pages loaded.

### 2.3 Virtual Strip Geometry & Frustum Culling
* **Aspect-Ratio Scaling:** Every image scales proportionally to fill `100%` of the viewport height (`renderWidth = viewportHeight * (imgWidth / imgHeight)`).
* **Virtual Layout Precomputation:** Calculates each item's `xOffset` and `width` along a virtual horizontal axis (`totalStripWidth`), inserting `gutterWidth` spacing between images when enabled.
* **Frustum Culling:** On every animation frame, only images whose `[xOffset, xOffset + width]` intersect the visible camera window `[viewLeft, viewRight]` are drawn.
* **Seamless Loop Wrapping:** When **Continuous Loop** is enabled, the renderer evaluates virtual strip copies at `-totalStripWidth`, `0`, and `+totalStripWidth`, drawing boundary images seamlessly without duplicating GPU textures.

### 2.4 WebGL2 Rendering & GLSL Shaders
* **High-DPI Canvas:** Sizes the WebGL2 backing buffer to physical screen pixels (`window.innerWidth * devicePixelRatio`, `window.innerHeight * devicePixelRatio`).
* **Vertex Shader:** Maps unit quad coordinates (`[0, 1]`) into clip space (`[-1, 1]`) using `u_rect` (`x, y, w, h` in physical pixels) and `u_resolution`.
* **Fragment Shader (`#version 300 es`):** Supports three selectable upscaling modes (`u_mode`):
  * **`0` — Catmull-Rom + CAS (Default):** Uses an optimized 9-tap Catmull-Rom bicubic filter (via 4 hardware bilinear lookups) to reconstruct smooth curves, followed by AMD's **Contrast-Adaptive Sharpening (CAS)** (`u_sharpness` from `0.0` to `1.0`) to sharpen fine details while suppressing ringing/pixelation on high-contrast ink lines.
  * **`1` — Catmull-Rom Bicubic Only:** 9-tap Catmull-Rom resampling without post-sharpening.
  * **`2` — Standard Hardware Bilinear:** Direct `texture(u_tex, v_uv)` sampling.

### 2.5 Camera, Motion & Auto-Pan Engine
* **Smooth Interpolation (Lerp):** Uses a `requestAnimationFrame` loop with exponential smoothing (`0.14` factor for X/Y panning, `0.16` factor for zoom scale).
* **Pixel-Aligned Auto-Pan (Optional):** When enabled, locks auto-scroll movement to exact integer physical pixel steps (`1 / devicePixelRatio` CSS pixels per frame) and bypasses fractional lerp smoothing during auto-pan to prevent sub-pixel shimmer on text and line art.

---

## 3. Configuration Settings & Persistence
Settings are automatically saved to `localStorage` (`comicRibbonSettings_v4`) on change and can be reset via the **Restore Defaults** button:

| Setting | Type | Default | Behavior |
| :--- | :--- | :--- | :--- |
| **Manga Mode** | Boolean | `false` | Reverses page order, aligns initial view to far right, sets default auto-pan direction right-to-left. |
| **Continuous Loop** | Boolean | `false` | Wraps final image seamlessly back into the first image in both directions. |
| **Randomize Images** | Boolean | `false` | Shuffles image order on load (overrides alphanumeric/Manga sorting). |
| **Vertical Gutter** | Boolean | `false` | Adds solid spacing between images (`0px` gap when disabled). |
| **Gutter Width / Color** | Number / Enum | `12px` / `#000000` | Sets gutter width in CSS pixels and background color (`#000000` or `#ffffff`). |
| **Upscale Shader** | Enum (`0`, `1`, `2`) | `0` (Catmull-Rom + CAS) | Selects the GPU resampling and sharpening algorithm. |
| **CAS Sharpness** | Slider (`0–100%`) | `55%` | Controls Contrast-Adaptive Sharpening intensity (`u_sharpness`). |
| **Pixel-Aligned Auto-Pan** | Boolean | `false` | Snaps auto-pan steps to exact device pixels instead of fractional sub-pixel lerp. |

---

## 4. Controls & Keyboard Shortcuts

| Input / Key | Action |
| :--- | :--- |
| **Mouse Wheel** | Smooth horizontal pan (maps vertical or horizontal wheel delta to X-axis). |
| **Click & Drag** | Grab-and-pan horizontally (and vertically when zoomed in). |
| **Double-Click** | Toggles `2.2x` cursor-centered **Zoom-to-Panel** (`1.0x` ↔ `2.2x`). |
| **`ArrowLeft` / `ArrowRight` / `A` / `D`** | Smooth manual horizontal pan (pauses auto-pan). |
| **`ArrowUp` / `ArrowDown` / `W` / `S`** | Vertical pan when zoomed in (`scale > 1.0`). |
| **`Space`** | Toggle hands-free **Auto-Pan** play/pause. |
| **`R`** | Reverse auto-pan scroll direction (`←` / `→`). |
| **`+` (`=`) / `-` (`_`)** | Increase or decrease auto-pan speed (`0.25` to `30.0` px/frame). |
| **`[` / `]`** | Decrease or increase live **CAS Sharpness** in `5%` increments. |
| **`K`** | Cycle live between Upscale Shader modes (`Catmull-Rom + CAS`, `Catmull-Rom Only`, `Standard Bilinear`). |
| **`1` – `9`, `0`** | Jump directly to image positions (`1` = first image, `2`–`9` = `20%`–`90%` marks, `0` = last image). |
| **`F`** | Toggle browser fullscreen mode. |
| **`Esc`** | Exit viewer and return to the Setup / Configuration panel (when not in fullscreen). |
