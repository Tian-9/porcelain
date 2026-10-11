---
name: retouch-lab
description: Tune or extend the Porcelain face-retouch app — adjust slider strengths, move/resize the smile-line mask, add new sliders or filters, verify changes visually in a local browser. Use whenever the user asks to change how any beauty effect looks or behaves.
---

# Retouch Lab

You are tuning a single-file WebGL beauty app. **Read `CLAUDE.md` first** — it has the architecture map, the tuning table (every constant with location and safe range), and the gotchas. This skill adds the workflow.

## Workflow for any tuning request

1. **Translate the ask into a constant.** "磨皮太狠 / smoothing too strong" → lower detail re-add or lower the mix cap. "法令纹没盖住 / fold band misses the crease" → band position/width in `buildMaskCanvas()`. Check the tuning table before hunting through code.
2. **Edit conservatively.** Move one constant at a time, stay inside the safe range, keep defaults subtle — a beauty app that looks obviously filtered at default settings is a failed beauty app.
3. **Verify visually, never blind-ship.**
   - `python3 -m http.server 8788` from the repo root, open `http://localhost:8788/index.html`
   - Load a **synthetic/AI-generated test portrait** (never a real person's photo without their explicit request). If the user wants tuning for their own face, ask them to test and report back rather than handling their photo yourself, unless they explicitly hand it to you.
   - For automated A/B checks use the debug hooks: set `__ketuParams.<key>`, call `__ketuRender()` (synchronous — background-tab rAF silently stalls and fakes a "no effect" result), screenshot, compare at slider 0 vs 100.
   - Check extremes: slider at 100 must not tear background/hair edges or warp mouth/nostrils.
4. **Keep both languages.** New UI strings go in Chinese in the markup **and** in the English i18n override block.
5. **Bump `CACHE` in `sw.js`** if you changed any cached file, or returning users keep the old version.

## Adding features

- **New slider**: follow the checklist in CLAUDE.md (markup → params → buildOps/uniform → presets → disable list → i18n).
- **New filter**: append to `FILTERS` (`[exposure, contrast, saturation, temp, tint, fade]`) with `name` (zh) + `en`, and add a swatch gradient in the chip builder.
- **New mask-based effect** (e.g. teeth whitening): the mask canvas has a free alpha channel; build the polygon from FaceMesh landmark indices, feather with canvas blur, consume as `m.a` in `FS_MAIN`.

## Hard rules

- Photos never leave the device: no fetch/upload of image data, ever. The privacy promise is the product.
- No frameworks, no build step, no new dependencies. It's one HTML file plus the bundled MediaPipe model — that's the point.
- Don't rename `vendor/face_landmarker.task.wasm` and don't remove the `DEV-SKELETON` markers.
