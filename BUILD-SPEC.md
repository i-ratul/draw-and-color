# BUILD-SPEC.md — "Draw & Colour" (Tool: a calm drawing canvas for a 3-year-old)

> Build target: a single static web page (vanilla HTML/CSS/JavaScript, **no frameworks, no libraries, no build step**), hosted on GitHub Pages / Cloudflare Pages.
> Device: **Samsung S22 Ultra**, Android Chrome, **portrait only**, used with the **S Pen**.
> This document is the complete specification. Build it in the order given in Section 10. Do not add features not listed here.

---

## 0. One-paragraph summary

A full-screen drawing canvas with a small fixed toolbar. The child draws **only with the S Pen** (finger touches on the canvas do nothing — this gives free palm rejection). The **buttons** are operated **only with the finger**. There are four drawing tools (Pen, Brush, Highlighter, Eraser), five always-visible colours plus a popup palette, a popup of preset line-art pictures to colour over, an Undo button, and two arrows to move between **10 blank pages**. No saving, no text, no scores, no timers, no sounds. Drawings persist while the app is open and are discarded on exit.

---

## 1. Design philosophy (read before building — non-negotiable)

This tool follows the same calm-design rules as the rest of this project:

- **One task only:** drawing/colouring. Nothing else.
- **No gamification:** no points, stars, timers, win/lose sounds, background music, confetti, or "good job!" feedback. The mark appearing under the pen *is* the reward (contingent feedback).
- **No text anywhere.** All controls are icons or colour swatches. A non-reading 3-year-old must be able to operate everything.
- **Large, forgiving targets.** Every button is big enough for an imprecise toddler finger.
- **Calm visuals.** Muted, low-arousal UI. No animation on the buttons beyond a simple "this is selected" state. No flashing.
- **Nothing punishes the child.** There is no "wrong". There is no destructive action that can't be recovered from by simply turning the page.

If a build decision is ambiguous, choose the **simpler, calmer** option.

---

## 2. Device & rendering targets

- **Orientation:** Portrait only. Do **not** lock orientation via the Screen Orientation API (not needed); just design for portrait and let the layout fill whatever the viewport is. The phone will be held upright.
- **Fullscreen:** The page must enter the browser Fullscreen API on the first user tap (see Section 9). This hides the Chrome address bar and tabs, preventing accidental navigation.
- **High-DPI rendering:** The S22 Ultra has a high `devicePixelRatio` (~3.5). **All canvases must be sized for device pixels**, or lines will look blurry. For every canvas:
  - Set the canvas **CSS size** to the layout size (e.g. `width: 100%; height: 100%` of its container).
  - Set the canvas **drawing-buffer size** to `cssWidth * devicePixelRatio` × `cssHeight * devicePixelRatio`.
  - Call `ctx.scale(devicePixelRatio, devicePixelRatio)` once after sizing, so all drawing code can work in CSS pixels.
  - Recompute this if the viewport size ever changes (e.g. after entering fullscreen — the available height changes when the address bar disappears, so **(re)size the canvases *after* fullscreen is entered**, not before).
- **Coordinate mapping:** Convert pointer events to canvas coordinates using `getBoundingClientRect()` on the canvas element, subtracting the rect's left/top from the event's `clientX/clientY`. Do not assume the canvas starts at (0,0) — the toolbar offsets it.

---

## 3. Layout

Portrait screen, divided top-to-bottom:

```
┌─────────────────────────────────────┐
│  TOOLBAR  (top, ~14% of viewport)   │  ← finger-only zone
│  Row 1:  [Pen][Brush][Highlt][Erase]│
│          [Undo]   [Pictures]        │
│  Row 2:  [⬛][🟥][🟦][🟨][🟩][＋palette]│
├─────────────────────────────────────┤
│                                     │
│                                     │
│         CANVAS  (~86%)              │  ← pen-only zone
│      (two stacked layers)           │
│                                     │
│   [◀ left arrow]      [right ▶]     │  ← arrows overlaid on canvas
│                                     │     bottom corners
└─────────────────────────────────────┘
```

### 3.1 Toolbar (the finger-only control strip)
- Occupies the **top ~14% of the viewport height** (tunable constant `TOOLBAR_HEIGHT_PCT`, default `14`). Two rows.
- **Background:** a distinct, calm solid colour (e.g. a soft warm grey `#ECE9E4`) so the toolbar is *visibly a different surface* from the white canvas. This visual separation is deliberate — it teaches the child "buttons live up here, drawing happens down there."
- The toolbar must have a **subtle bottom border / shadow** to reinforce the boundary.
- **Row 1 — tools & actions:** Pen, Brush, Highlighter, Eraser, then a gap, then Undo, then Pictures. Seven targets.
- **Row 2 — colours:** five fixed swatches (black, red, blue, yellow, green) then a "more colours" swatch (the palette opener — render it as a small multi-colour/rainbow swatch icon, **not text**).
- **Button sizing:** each button/swatch should be at least **64×64 CSS px** of tappable area with generous spacing. If seven items don't fit comfortably across the width in row 1, reduce the gap but never shrink targets below 64px. Compute sizes from viewport width so it adapts; do not hard-code for one width only.

### 3.2 Canvas area (the pen-only drawing zone)
- Occupies the remaining **~86%** of the viewport, full width.
- **Two stacked canvases**, identical size, absolutely positioned on top of each other:
  - **Layer 1 (back) — "artLayer":** holds the preset line-art picture, if one is loaded. Blank (transparent over white page background) otherwise. **The child never draws on this layer.** The eraser never touches it. This is what makes the outline indestructible.
  - **Layer 2 (front) — "drawLayer":** transparent. **All the child's strokes go here.** The eraser only clears pixels on this layer.
  - Behind both, the page background is plain **white** (`#FFFFFF`).
- Only `drawLayer` listens for pointer events. `artLayer` has `pointer-events: none`.
- **Stacking order (bottom → top):** white background → artLayer → drawLayer → arrow buttons.

### 3.3 Page-navigation arrows
- Two arrows, **left** and **right**, overlaid on the **bottom-left and bottom-right corners** of the canvas area.
- Large circular icon buttons (≥72×72 CSS px), semi-translucent so they don't dominate, but clearly visible.
- **Finger-operated** (they sit above the canvas; their own pointer handling takes finger taps — see Section 8 for how this coexists with pen-only canvas).
- On page 1, the left arrow is **visually dimmed and inert** (no wrap-around). On page 10, the right arrow is dimmed and inert. (No numbers shown — the child won't read them; the dimming is the only cue, which is fine.)

---

## 4. Tools (Row 1)

Four drawing tools. **Exactly one is selected at a time.** The selected tool shows a clear "selected" state (e.g. inset/darker background or a ring). Default selected tool on launch: **Pen**.

| Tool | Line width (CSS px, tunable) | Opacity | Colour source | Notes |
|------|------|---------|---------------|-------|
| **Pen** | `PEN_WIDTH` = 4 | opaque (1.0) | current colour | thin, crisp line |
| **Brush** | `BRUSH_WIDTH` = 22 | opaque (1.0) | current colour | thick, solid line |
| **Highlighter** | `HILITE_WIDTH` = 28 | semi-transparent (0.35) | current colour | thick, see-through |
| **Eraser** | `ERASER_WIDTH` = 40 | n/a | n/a | clears `drawLayer` pixels only |

### 4.1 Drawing mechanics (applies to Pen, Brush, Highlighter)
- Use `lineCap = 'round'` and `lineJoin = 'round'` for smooth, child-friendly strokes.
- A stroke is drawn as a series of line segments between successive pointermove points (pen-down → moves → pen-up). For smoothness, draw straight segments point-to-point; that is sufficient at this width and age.
- **No colour mixing.** Each new stroke is painted normally on top; whatever was underneath on `drawLayer` is simply covered. Use `globalCompositeOperation = 'source-over'` for all three drawing tools. (The "latest tool/colour remains visible on top" behaviour is exactly source-over; do not implement any blending.)
- **Highlighter transparency caveat — important:** to get a *clean* semi-transparent highlighter that does **not** darken where a single stroke overlaps itself, draw the entire stroke into an **offscreen buffer at full opacity**, then composite that whole buffer onto `drawLayer` once at 0.35 alpha on pen-up. If that is too complex on the first pass, the acceptable simpler fallback is: set `ctx.globalAlpha = 0.35` and draw directly — accepting that self-overlapping parts of one highlighter stroke look slightly darker. **Pick the simpler fallback for v1**; note the offscreen-buffer upgrade as a known refinement.

### 4.2 Eraser mechanics
- The eraser acts **only on `drawLayer`**. Set `ctx.globalCompositeOperation = 'destination-out'` and stroke with the eraser width; this clears pixels back to transparent, revealing the artLayer outline and/or white page beneath. The outline is never affected because it lives on a different canvas.
- `ERASER_WIDTH` is intentionally large (40). This is tunable; expect to revisit after testing — the child may rarely use it.
- The eraser is a *tool*, selected like the others. Selecting the eraser and dragging the pen erases. Selecting a drawing tool returns to drawing.

---

## 5. Colours (Row 2)

- **Five fixed swatches**, always visible: black `#000000`, red `#E23B2E`, blue `#2E6BE2`, yellow `#F2C53D`, green `#3DA35D`. (These are calm, slightly muted versions — tunable.)
- **Exactly one colour is selected at a time**, shown with a clear selected ring/border. Default on launch: **black**.
- **Palette opener swatch:** the sixth item in row 2. Tapping it opens the **palette popup**.

### 5.1 Palette popup
- A simple modal panel that appears centred over the canvas with a dimmed backdrop.
- Shows a **grid of 12–15 colour swatches** (tunable list `PALETTE_COLOURS`), each a large tappable square (≥64×64).
- Tapping a swatch: selects that colour as the current colour, closes the popup. The newly chosen colour also becomes the "active colour" — reflect selection state sensibly (the five fixed swatches lose their selected ring since the active colour is now a palette colour; that's fine).
- Tapping the dimmed backdrop (outside the grid) closes the popup **without** changing colour.
- **No text, no close button needed** — backdrop tap and swatch tap are the only two exits. (If a close affordance feels necessary, use a simple ✕ icon, but prefer backdrop-tap-to-close.)
- The popup is **finger-operated** (it's part of the control surface).

---

## 6. Pictures popup (preset line-art to colour over)

- The **Pictures button** (row 1) opens a popup panel over the canvas with a dimmed backdrop.
- The panel shows **exactly 6 thumbnails** in a grid (e.g. 2 columns × 3 rows, or 3×2 — choose what fits portrait comfortably). Each thumbnail is large and tappable.
- The 6 are chosen as follows: the project contains a folder of line-art PNGs (`/art/`). If there are **more than 6**, pick **6 at random** each time the popup opens. If there are exactly 6 or fewer, show what exists.
- **Tapping a thumbnail:**
  1. Loads that PNG onto the **artLayer** of the **current page** (drawn to fill the canvas area — see 6.2 for fit rules).
  2. Closes the popup.
  3. The child can now colour over it on `drawLayer`.
- Loading a picture onto a page **replaces** any picture previously on that page's artLayer. It does **not** clear the child's existing strokes on `drawLayer` (those stay on top). This is acceptable and simplest; do not over-engineer a confirm dialog.
- A **"blank" option:** include, as the **first** item in the popup, a plain blank thumbnail (just a white square with a thin border) that clears the artLayer back to empty. This lets the child return a page to a plain drawing page without a picture. (This is the only "clear" affordance in the app, and it only clears the *outline* layer, never the child's strokes.)
- The Pictures popup is **finger-operated**.

### 6.1 Art asset specification
- All line-art PNGs live in `/art/` with **transparent background** and **black outlines** only (no fill).
- **Author them at the canvas's physical pixel size** for crispness. Compute the target as: canvas CSS width × DPR by canvas CSS height × DPR. For planning, assume the canvas area is roughly the full phone width × 86% of height. **The PRD author (you) will measure the exact canvas pixel size on-device once during the first build and record it here:**
  - `ART_CANVAS_PX = 1080 × 1555`  ← measured on the S22 Ultra in fullscreen Chrome (CSS area 384×553, devicePixelRatio 2.8125). Author line-art at this size (portrait, transparent, black line) for crispness.
- Until measured, author art at a safe **1200 × 2000 px** (portrait, transparent, black line, centred subject with margin). This will scale cleanly.
- Keep file sizes modest (these are simple line drawings; each should be well under ~300 KB).
- Maintain a `/art/CREDITS.txt` noting the source/licence (CC0) of each image, per project hygiene.

### 6.2 Picture fit rules
- Draw the PNG onto the artLayer **scaled to "contain"** within the canvas area (fit entirely, preserving aspect ratio, centred), so no part of the outline is cut off. Compute scale = `min(canvasW / imgW, canvasH / imgH)`; centre the result.
- Do not stretch (no aspect-ratio distortion).

---

## 7. Pages (the notebook)

- **10 pages**, fixed. Each page has its **own artLayer content and its own drawLayer content**, both held **in memory for the session**.
- Implement each page's state as an offscreen store. Simplest robust approach:
  - Keep an array `pages[0..9]`, each holding two `ImageData` (or two offscreen canvases): one for artLayer, one for drawLayer.
  - On navigating away from a page, **snapshot** both visible canvases into that page's store.
  - On navigating to a page, **restore** both canvases from that page's store (or blank if never visited).
- Navigation is via the two arrows only. No wrap-around (Section 3.3).
- **No saving to disk.** Closing the tab / exiting fullscreen and leaving discards everything. That is intended. Do not add localStorage, downloads, or any persistence.
- Starting page on launch: **page 1**, blank, Pen + black selected.

---

## 8. Input model — pen vs finger (the core of palm rejection)

This is the most important technical section. Get it right.

- Use the **Pointer Events API** (`pointerdown`, `pointermove`, `pointerup`, `pointercancel`) everywhere. Do not use mouse or touch events.
- **The canvas (`drawLayer`) only accepts `event.pointerType === 'pen'`.** In the canvas's `pointerdown` handler, immediately `return` if `pointerType !== 'pen'`. This means:
  - The S Pen draws.
  - A resting palm, a stray finger, a second hand — **all ignored on the canvas.** Free palm rejection.
- **All buttons (toolbar, arrows, popups) accept finger (`'touch'`) and also mouse (for desktop testing).** They do **not** require pen. Practically: let the buttons be normal DOM buttons/divs with `pointerdown`/`click` handlers that don't filter by pointerType, OR filter to accept `touch`/`mouse` and ignore `pen` — but accepting all on buttons is fine and simpler. The deliberate *teaching* goal ("pen up here is for drawing, finger for buttons") is reinforced by **design language** (the toolbar's distinct surface colour), not by hard-blocking the pen on buttons. **Do not block finger on the canvas by accident** — only pen draws there, but that's because we filter *to* pen, not because we filter *out* finger elsewhere.
- **Prevent default touch behaviours on the canvas** to stop scrolling/zooming: set CSS `touch-action: none` on the canvas, and call `event.preventDefault()` in canvas pointer handlers. Set `overscroll-behavior: none` and disable pinch-zoom via the viewport meta tag (`user-scalable=no`).
- **Pointer capture:** on canvas `pointerdown` (pen), call `canvas.setPointerCapture(event.pointerId)` so the whole stroke is tracked even if the pen briefly leaves the element bounds. Release on `pointerup`.
- **Coalesced events (optional refinement):** for smoother lines at speed, you may use `event.getCoalescedEvents()` in `pointermove` to capture intermediate points. Nice-to-have, not required for v1.

---

## 9. Fullscreen & launch gesture

- The app loads showing a **Start screen**: a calm full-screen panel with a single large central icon button (e.g. a pencil/crayon icon — **no text**). Everything else is hidden behind it.
- On the **first tap** of that button (a genuine user gesture), do, in this order:
  1. Request **fullscreen** on the document root (`documentElement.requestFullscreen()`), with the webkit fallback if needed.
  2. Hide the Start panel, reveal the toolbar + canvas.
  3. **Then** run the high-DPI canvas sizing (Section 2) — because available viewport height changes once fullscreen hides the address bar, sizing must happen *after* fullscreen.
- **No orientation lock** is requested (portrait is assumed; not enforced).
- **No audio** in this tool, so there is no autoplay-unlock concern (unlike Wait for Green).
- If fullscreen is dismissed by the user (Android back gesture / swipe), that's acceptable — the app keeps working in normal view. Do not aggressively re-request fullscreen (annoying and can loop).

---

## 10. Build order (do these in sequence; verify each before moving on)

1. **Skeleton + portrait layout.** HTML with toolbar (placeholder buttons) and a canvas area. Toolbar distinct colour, ~14% height. Verify proportions on the S22 Ultra in Chrome.
2. **Fullscreen Start gesture.** Add the Start panel and the tap-to-fullscreen flow (Section 9). Verify the address bar disappears and the canvas resizes correctly afterwards.
3. **High-DPI single drawing canvas + Pen.** One canvas for now. Pen-only input (filter to `pointerType==='pen'`), black, source-over, round caps. Verify crisp non-blurry lines with the S Pen and that **finger does not draw**. **Log and record `canvas.width × canvas.height` here → fill into `ART_CANVAS_PX` in Section 6.1.**
4. **Tools + selected state.** Add Brush, Highlighter (use the simple `globalAlpha` fallback for v1), Eraser (destination-out). One-selected-at-a-time UI. Verify each behaves per Section 4.
5. **Two layers.** Split into artLayer (back, `pointer-events:none`) + drawLayer (front, receives input). Confirm eraser only clears drawLayer and cannot touch artLayer.
6. **Colours row + selected state.** Five fixed swatches + active-colour logic. Verify colour switching mid-drawing with no mixing.
7. **Palette popup.** 12–15 swatch grid, backdrop-tap-to-close, selects colour. Finger-operated.
8. **Pictures popup + art loading.** `/art/` folder, random-6 selection, blank option first, contain-fit onto artLayer of current page. Add 2–3 test PNGs to verify.
9. **Pages (1–10) with in-session memory.** Snapshot/restore both layers on navigation. Arrows with no-wrap dimming. Verify flipping away and back preserves both the outline and the strokes.
10. **Undo (last 3 strokes).** See Section 11. Verify it removes whole strokes (pen-down→pen-up units), up to 3 back, and no further.
11. **Polish pass.** Target sizes ≥ spec, selected states clear, calm colours, no text anywhere, no console errors. Test the whole flow with the actual S Pen on the actual device.

---

## 11. Undo specification

- Undo keeps the **last 3 strokes** on the **current page's drawLayer**. A "stroke" = one pen-down → pen-up unit (a continuous squiggle counts as one).
- Implementation (simple and robust): before each new stroke begins (on pen-down), **push a snapshot** of the current drawLayer (`ImageData` or a cloned offscreen canvas) onto a per-page `undoStack`. Cap the stack at **3** entries (drop the oldest when a 4th is pushed).
- **Undo button tap:** pop the latest snapshot and restore it to drawLayer. If the stack is empty, do nothing (button may dim, but no error).
- Undo affects **only drawLayer**, never artLayer (loading a picture is not undoable — that's fine; the blank option in the Pictures popup covers "remove the outline").
- The undo stack is **per page** and lives only for the session.
- Keep memory sane: snapshots are large at device resolution × 3 per page × 10 pages. To bound it, **only the current page needs a live undo stack**; you may reset/discard a page's undo stack when navigating away (acceptable — undo is a short-term "oops" fix, not cross-page history). Document this so it's a deliberate choice, not a bug.

---

## 12. Tunable constants (collect at top of the JS for easy adjustment)

```js
const CONFIG = {
  TOOLBAR_HEIGHT_PCT: 14,     // top control strip height as % of viewport
  PEN_WIDTH:   4,
  BRUSH_WIDTH: 22,
  HILITE_WIDTH:28,
  HILITE_ALPHA:0.35,
  ERASER_WIDTH:40,            // expect to revisit after testing
  FIXED_COLOURS: ['#000000','#E23B2E','#2E6BE2','#F2C53D','#3DA35D'],
  PALETTE_COLOURS: [ /* 12–15 hex values */ ],
  PAGE_COUNT: 10,
  PICTURES_SHOWN: 6,
  UNDO_DEPTH: 3,
  MIN_BUTTON_PX: 64,
  MIN_ARROW_PX: 72,
  TOOLBAR_BG: '#ECE9E4',
  PAGE_BG: '#FFFFFF'
};
```

## 13. Explicit non-goals (do NOT build these)

- No save / load / export / download / localStorage.
- No undo beyond 3 strokes; no redo.
- No clear-page button (turning the page is the reset; the blank picture option clears the outline layer only).
- No thickness/opacity sliders; no colour mixing/blending; no smudge.
- No scrolling or zooming of the canvas.
- No sounds, music, points, stars, timers, badges, animations beyond simple selected-states.
- No text labels anywhere — icons and swatches only.
- No landscape layout; no orientation lock.
- No second-child accounts, settings screens, or menus.
