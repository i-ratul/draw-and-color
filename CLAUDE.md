# CLAUDE.md — guardrails for building "Draw & Colour"

You are helping build a **single static web page**: a calm drawing canvas for a **3-year-old**, used with an **S Pen on a Samsung S22 Ultra in portrait, Android Chrome**. The full specification is in `BUILD-SPEC.md`. Read it before writing code. This file is the short list of rules you must never break.

## I am not a programmer
Explain each step in plain English. Keep it simple. Tell me what to test on the phone after each step. Build in the numbered order in BUILD-SPEC.md Section 10, one step at a time, and let me verify before moving on.

## Hard technical rules
- **Vanilla HTML/CSS/JavaScript only.** No frameworks, no libraries, no npm, no build step, no bundler. One `index.html` (CSS and JS may be inline or in plain `.css`/`.js` files — your call, keep it simple).
- **Pointer Events API only.** Not mouse events, not touch events.
- **The canvas only accepts the pen.** In canvas pointer handlers, ignore any event whose `pointerType` is not `'pen'`. This is our palm rejection — a resting hand must never draw.
- **Buttons are finger-operated** and must not require the pen.
- **High-DPI is mandatory.** Size every canvas for `devicePixelRatio` (~3.5 on this device) or lines look blurry. Re-size canvases *after* entering fullscreen, because the viewport height changes.
- **Two stacked canvases:** a back **artLayer** (line-art outline, `pointer-events:none`, never drawn on, never erased) and a front **drawLayer** (all child strokes + eraser). The outline must be impossible to erase.
- `touch-action: none` on the canvas; `user-scalable=no` in the viewport meta; `preventDefault()` in canvas handlers — no scrolling or pinch-zoom.

## Calm-design rules (the point of this whole project)
- **No text anywhere.** Icons and colour swatches only. The child cannot read.
- **No gamification:** no points, stars, timers, badges, win/lose sounds, music, confetti, or praise. The mark under the pen is the only reward.
- **No sounds at all** in this tool.
- **Large, forgiving targets** (buttons ≥64px, arrows ≥72px).
- **Calm, muted visuals.** The toolbar is a visibly different surface colour from the white canvas, on purpose — it teaches "buttons up here, drawing down there." No flashing, no bouncy animation beyond a simple selected-state.
- **Nothing destructive in one tap.** No clear-everything button. Recovery is: turn the page, use the eraser, or undo (last 3 strokes).

## Scope discipline
Build **only** what is in BUILD-SPEC.md. The non-goals list in Section 13 is binding. If you think something needs an extra feature, ask me first — do not add it. When a choice is ambiguous, pick the **simpler, calmer** option.

## When something breaks on mobile
Expect a couple of iterations on: pen input not registering, blurry lines (DPR), fullscreen resizing the canvas, the highlighter darkening on overlap. These are known and covered in BUILD-SPEC.md. Flag them, don't paper over them.
